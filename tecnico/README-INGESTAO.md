# Manual de ingestão documental

## 1. O que este documento explica

Este documento é o ponto de entrada da ingestão documental no repositório. Ele existe para explicar, em linguagem simples, como a plataforma recebe fontes, prepara o runtime, decide paralelismo, publica fan-out e acompanha o lote até o fechamento operacional.

Em termos práticos, este é o documento dono da ingestão. Os manuais conceituais e técnicos por tipo, como PDF, HTML, Excel e JSON, aprofundam partes específicas. Este arquivo fica responsável pelo contrato comum que atravessa toda a esteira.

## 2. Como pensar a ingestão sem confusão

O erro mais comum é tratar ingestão como upload de arquivo. No código real, ingestão é uma esteira assíncrona com quatro responsabilidades separadas:

- receber a intenção do YAML e transformar isso em um pedido executável;
- resolver as fontes reais do lote;
- preparar o runtime comum antes do processamento pesado;
- executar o processamento por tipo sem perder rastreabilidade do run.

Isso importa porque OCR, parsing, chunking, persistência e indexação não devem ficar misturados no endpoint HTTP.

### 2.1. O que separa esta esteira de uma ingestão ingênua

Uma ingestão ingênua faz três coisas: lê o arquivo, corta o texto em pedaços, gera os vetores. O
problema não é que ela seja simples — é que ela **não sabe quando deu errado**. Um arquivo lido pela
metade, uma geração antiga não removida, um lote interrompido no meio: nos três casos ela termina sem
exceção e reporta sucesso.

As quatro decisões estruturais desta esteira atacam exatamente isso:

| Decisão | O que ela compra |
|---|---|
| **Execução como job durável, não como script** | o lote sobrevive a reinício de servidor, é cancelável de verdade e deixa histórico auditável (§5, §6) |
| **Desfecho derivado, não presumido** | "terminou sem exceção" não é sucesso: cada documento recebe um veredito que separa perda real de descarte deliberado |
| **Identidade e versão explícitas por documento** | reingerir substitui a geração anterior em vez de acumular duas versões do mesmo documento no acervo |
| **Um pipeline especializado por formato, atrás de contrato comum** | PDF, planilha, HTML e JSON têm problemas diferentes, e tratá-los com a mesma leitura genérica perde estrutura em todos |

Quem quiser a versão detalhada dessas garantias no formato mais exigente — PDF — deve ler
[README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md),
§0.4.

## 3. Componentes que governam o fluxo

Os componentes abaixo são os pontos mais importantes para entender o comportamento real da ingestão.

- `IngestionService`: fachada principal que recebe o pedido, resolve política de execução e decide se haverá fan-out.
- `IngestionRuntimePreparationService`: preparador compartilhado que organiza o runtime comum antes do processamento pesado. Ele reduz duplicação e evita que cada processor monte seu próprio mundo operacional.
- `ContentIngestionOrchestrator`: orquestrador que coordena a esteira quando o lote segue pelo caminho direto.
- `DocumentFanoutCoordinator`: coordenador responsável pelo fan-out documental quando o lote precisa ser quebrado em filhos por documento.
- `IngestionParentProcess` e `DocumentFanoutChildProcess`: processos nominais registrados no Job
  Core. Eles recebem somente input de domínio e `ProcessContext`, sem acesso ao ledger.

Em linguagem simples: a fachada organiza, o preparador monta o contexto, o orquestrador conduz o fluxo direto e o `DocumentFanoutCoordinator` cuida da versão paralela por documento.

### 3.1. Fontes aceitas pelo pedido executável

O request interno consolidado cobre conteúdo em markdown, texto, JSON, Excel, PDF, DOCX, PPT,
imagens e URLs web, além de referências de Confluence, YouTube, Google Drive, Azure Blob e S3. A
presença no request não garante fan-out: cada família ainda precisa provar inventário e replay
seguros (§5.1).

No Google Drive, `ingestion.google_drive.sources` aceita três modos distintos:

| `type` | Campos decisivos | Efeito |
|---|---|---|
| `folder` | `folder_id`, `recursive`, `file_types`, `file_patterns`, `max_results` | lista a pasta e, quando solicitado, seus descendentes |
| `documents` | `document_ids` | usa IDs explícitos; `recursive` é falso |
| `search` | `query`, `file_types`, `file_patterns`, `max_results` | lista por consulta do Drive |

`ingestion.google_drive.max_files` aplica o limite global depois da resolução das fontes. A
enumeração percorre a paginação usando `nextPageToken`; receber menos itens que `pageSize` não é
prova de que a listagem terminou. Recursão é escolha explícita do modo `folder`, não comportamento
que deve ser presumido para toda fonte.

### 3.1 Filtro por nome de arquivo (`file_patterns`)

`file_patterns` restringe, por nome de arquivo, o que a varredura entrega ao pipeline. Arquivo que
não casa nenhum padrão é descartado ainda na listagem: não vira job filho, não é baixado e não é
processado. É a mesma chave já usada por S3, MinIO e Azure Blob, com a semântica de glob do
`fnmatch` — `eng*.pdf`, `*RELATORIO*` ou o nome exato `IPR-740.pdf`.

```yaml
ingestion:
  google_drive:
    enabled: true
    max_files: 600
    file_patterns:            # vale para todas as fontes abaixo
      - "eng*.pdf"
      - "IPR-740.pdf"
    sources:
      - type: "folder"
        folder_id: "1NiMpZ..."
        file_types: ["pdf", "application/pdf"]
        file_patterns: ["rel*.pdf"]   # opcional: sobrepõe o global nesta fonte
        max_results: 600
```

Regras que valem no runtime:

- **Precedência:** `sources[].file_patterns` vence `google_drive.file_patterns`, simétrico a
  `sources[].max_results` sobre `google_drive.max_files`. Ausência nos dois níveis (ou lista vazia)
  mantém o comportamento padrão de entregar tudo o que a varredura encontrar.
- **Caixa ignorada:** `eng*.pdf` casa `ENG-123.PDF`. Só no Google Drive — S3, MinIO e Azure Blob
  seguem sensíveis à caixa, como sempre foram.
- **Ordem em relação ao limite:** o filtro roda antes do corte por `max_results`/`max_files`, então
  o teto conta apenas arquivos que passaram; um arquivo descartado não consome vaga do lote.
- **Escrita única, dois consumidores:** o filtro é aplicado dentro do datasource
  (`GoogleDriveDatasourceClient.list_available_documents`), de onde tanto o planejamento do fan-out
  quanto o processamento remoto herdam a mesma decisão. O predicado é o helper canônico
  `src/ingestion_layer/core/file_pattern_matching.py::matches_file_patterns`.
- **Observabilidade:** `ingestion.request_source_resolvers.google_drive.file_patterns.resolved`
  registra o filtro aplicado e sua origem; o resumo da varredura ganha `skipped_by_file_pattern`; e
  `ingestion.content_family.remote.google_drive.file_pattern_filter_empty` alerta em nível `warning`
  quando o filtro descarta 100% dos arquivos elegíveis — o cenário típico de padrão digitado errado.

## 4. Contrato operacional mínimo do run

Aceitação assíncrona não significa sucesso final. Na maioria das vezes, significa só que o pedido foi aceito e entregue ao runtime assíncrono.

Por isso a leitura operacional do lote depende de duas rotas de consulta:

- `/ingestion-runs/query`: lista runs, estados agregados e visão operacional resumida.
- `/ingestion-runs/detail`: mostra o detalhe operacional do run, incluindo filhos, progresso e snapshots relevantes.

Quando existe fan-out por documento, um dos contratos mais úteis do detalhe operacional é `fanout_overview`. Esse campo resume o estado agregado do pai e dos filhos, incluindo volume de documentos, distribuição de status, sinais de cancelamento e informações usadas pela UI administrativa.

## 5. Fan-out documental sem job fantasma

Fan-out documental existe para throughput e isolamento. Ele permite que um lote com vários documentos seja dividido em filhos menores, sem perder a ideia de um run pai agregado.

Mas existe uma trava anterior ao primeiro filho: para o mesmo `tenant_code + vectorstore_id`, só pode existir um run pai ativo por vez. Em termos simples, pode haver paralelismo dentro do lote, mas não pode haver dois lotes pais diferentes brigando pelo mesmo acervo vivo ao mesmo tempo.

Essa regra ignora `vector_store.if_exists`. O campo ainda decide como o dataset vivo será tratado quando o lote for válido (`overwrite`/`update`/`skip` — ver `src/config/vector_store_contract.py::VectorStoreContract` e `src/ingestion_layer/vector_stores/dataset_lifecycle_orchestrator.py::apply_if_exists_policy`), mas ele não serve para liberar concorrência entre dois pais. Se já existe um pai ativo, o comportamento correto é rejeitar a nova admissão até o run anterior terminar ou ser reconciliado.

**Não existe mais RabbitMQ nem Dramatiq no transporte de jobs** (limpeza anterior a 13/07/2026). Não há fila física separada para pai e filho, não há broker e não há redelivery de mensagem: pai e filho são a mesma tabela `job_core.job_runs`, e cada linha só pode ser reivindicada uma única vez por `claim_next_run` (`FOR UPDATE SKIP LOCKED`, `src/core/job_core/postgres_store.py`). Também não existe mais `DocumentFanoutExecutionGate`: essa classe própria da ingestão foi removida do produto (plano `investigacao-simplificacao-job-core`, tarefa T10, 2026-07-17).

Quem decide hoje, em dois pontos diferentes, nenhum deles vivendo na ingestão:

- **Admissão de um filho novo:** o próprio Job Core, no momento do `claim_next_run`. O envelope do filho carrega `dispatch_mode=document_fanout_child` e o pai carrega `max_active_children` (`JobEnvelope.max_active_children`); o `claim_next_run` só libera um filho novo se o número de filhos ativos daquele pai estiver abaixo do limite, na mesma consulta atômica que reivindica a linha.
- **Cancelamento de um filho já em execução:** o processo consulta cooperativamente apenas
  `ProcessContext.host.is_cancellation_requested()`/`raise_if_cancelled()`. O token e o store
  concretos ficam encapsulados no Job Core; `ProcessHostReporter` adapta o progresso funcional da
  ingestão ao port sem entregar lifecycle ao domínio. Se o pai não terminar de forma limpa (worker
  morto, heartbeat expirado), quem encerra a run órfã é o reconciliador cancel-only do Job Core
  (`cancel_orphaned_run`), nunca uma reconciliação local da ingestão.

Em linguagem simples: antes, a ingestão perguntava "o pai ainda deixa eu trabalhar?" a cada filho, consultando uma gate própria. Hoje, se o filho foi reivindicado é porque o Job Core já decidiu que ele pode rodar; e se o pai for cancelado no meio do caminho, o mesmo Job Core avisa o filho para parar. Detalhe completo, com símbolos e eventos de log: `README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md`, seções 2.3, 2.4 e 3.4.

## 5.1 Elegibilidade real do fan-out

Nem toda fonte que parece remota está pronta para fan-out. O coordinator só deve publicar filhos quando consegue inventariar e reconstruir cada documento com segurança.

O contrato atual é este:

- PDF e filesystem local continuam fora do fan-out. O reason code operacional esperado é `local_filesystem_not_shared`.
- Fontes remotas com inventário seguro e replay explícito entram no fan-out com o sinal positivo `remote_reference_replayable`.
- Fontes que ainda não têm inventário seguro no coordinator caem para o modo sequencial com `fanout_inventory_not_supported:<fonte>`.

Em linguagem simples: remoto não basta. O sistema precisa saber listar o documento, reconstruir a referência e provar que o filho consegue ser reexecutado sem adivinhação.

## 5.2 Cancelamento cooperativo sem promessa falsa

Quando o operador cancela um lote com fan-out por documento, o efeito correto não é “matar processo fisicamente no mesmo milissegundo”. O efeito correto é registrar o cancelamento de forma durável, impedir trabalho novo e drenar com segurança o que já tinha começado.

Em termos práticos, o contrato operacional é este:

- filho ainda `queued` no ledger do Job Core é cancelado diretamente, sem nunca ser reivindicado e sem iniciar OCR, parsing ou republicação (não existe broker nem redelivery neste transporte — a linha do filho só é reivindicada uma vez);
- filho já dentro de OCR, download ou parsing pesado só para quando o host do `ProcessContext`
  observa `cancel_requested`/`cancelled` num checkpoint cooperativo, ou quando o worker morre e o
  reconciliador cancel-only do Job Core (`cancel_orphaned_run`) encerra a run órfã diretamente como
  `cancelled`;
- não existe mais `auto_recovery` nem `auto_promotion` tentando salvar ou reencaminhar um filho — o cancelamento nunca vira retry, replay ou reenfileiramento.

Isso importa porque evita duas leituras erradas ao mesmo tempo: achar que o botão falhou quando o lote ainda está drenando, ou achar que o sistema promete kill físico imediato quando o modelo real é cooperativo.

## 5.3 Flag operacional e caminho estável

O caminho estável do produto continua sendo a ingestão sequencial. O fan-out documental fica protegido pela flag `INGESTION_DOCUMENT_FANOUT_ENABLED`.

Na prática, isso significa:

- se a flag estiver desligada, o sistema permanece sequencial e registra `feature_flag_disabled` nos sinais operacionais;
- se a flag estiver ligada, o coordinator ainda pode recuar para sequencial quando a fonte não for elegível ou quando o plano de controle obrigatório não estiver íntegro;
- esse recuo não é fallback escondido para mascarar erro de banco ou de contrato. Quando a decisão crítica não pode ser provada, o comportamento correto continua sendo falhar fechado.

## 6. O que acontece por baixo do capô

O fluxo comum da ingestão segue esta lógica:

1. `IngestionService` recebe a intenção de execução.
2. O sistema resolve as fontes reais do lote.
3. `IngestionRuntimePreparationService` monta o runtime compartilhado.
4. O serviço decide entre caminho direto e fan-out.
5. Se for caminho direto, `ContentIngestionOrchestrator` coordena o processamento.
6. Se for fan-out, `DocumentFanoutCoordinator` publica os filhos e preserva o run pai como unidade lógica.
7. O operador acompanha o lote pelas rotas `/ingestion-runs/query` e `/ingestion-runs/detail`.

## 6.1. Reingestão e identidade do documento

Um acervo vivo é um acervo reprocessado, e é aí que uma esteira ingênua acumula lixo. Três garantias
sustentam o reprocesso nesta plataforma, todas detalhadas no manual de PDF:

- **Identidade lógica estável.** O documento é reconhecido pela origem (sistema de origem e
  identificador externo), não pelo caminho do arquivo. Renomear não cria um documento novo.
- **Geração identificada por conteúdo *e* política.** A chave que separa uma ingestão da seguinte
  inclui tanto os bytes do arquivo quanto a configuração de extração usada. É o que faz "mesmo
  arquivo, configuração melhor" ser reconhecido como uma geração nova.
- **Substituição, não acumulação.** Depois da publicação confirmada, as gerações anteriores daquele
  documento são removidas do vector store, com contagem antes e depois como prova. Resíduo é tratado
  como falha, não como sucesso silencioso.

O efeito prático para quem opera: sob `vector_store.if_exists: update`, uma reingestão pula o que não
mudou, reprocessa o que mudou de conteúdo **ou** de configuração, e reprocessa também o que ficou
incompleto na rodada anterior.

## 7. Como estudar o resto da ingestão

Depois deste documento, a leitura mais produtiva é esta:

1. `README-INGESTAO-INDICE.md`, para navegar pelos documentos do domínio.
2. `README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO.md`, para visão funcional do PDF.
3. `README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md`, para pipeline técnico do PDF e a admissão/cancelamento canônicos do fan-out pelo Job Core.
4. `README-CONCEITUAL-INGESTAO-HTML-PIPELINE-COMPLETO.md` e `README-TECNICO-INGESTAO-HTML-PIPELINE-COMPLETO.md`, para HTML e web.
5. `README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO.md` e `README-TECNICO-INGESTAO-EXCEL-PIPELINE-COMPLETO.md`, para planilhas.
6. `README-CONCEITUAL-INGESTAO-JSON-PIPELINE-COMPLETO.md` e `README-TECNICO-INGESTAO-JSON-PIPELINE-COMPLETO.md`, para JSON estruturado.

## 8. Explicação 101

Pense na ingestão como um centro de triagem.

- Uma equipe recebe o pedido.
- Outra separa os itens por tipo.
- Outra prepara o ambiente certo para cada item.
- Outra acompanha se o lote ainda está autorizado a seguir.

Cada caixa (documento) só é retirada da prateleira uma única vez — não existe fila com reentrega. Antes de retirar, o sistema confere no banco (Job Core) se aquele lote continua válido e se ainda há vaga para mais uma caixa em processamento. Esse é o papel do contrato operacional da ingestão.

## 9. Evidências no código

- `src/api/services/ingestion_job_processes.py`: processos nominais pai e filho da ingestão.
- `src/core/job_core/job_process.py`: `ProcessContext.host` e o port mínimo exposto ao domínio.
- `src/api/services/process_host_reporter.py`: adaptação de progresso funcional sem lifecycle
  paralelo.
- `src/services/ingestion_request_builder.py` e
  `src/services/ingestion_request_source_resolvers.py`: request consolidado e contratos das fontes.
- `src/ingestion_layer/core/file_pattern_matching.py`: predicado único do filtro por nome
  (`file_patterns`) usado por Google Drive, S3 e Azure Blob.
- `src/ingestion_layer/clients/google_drive_client.py`: paginação por `nextPageToken` e limite
  explícito da listagem.
- `src/services/document_fanout_coordinator.py`: inventário, replay e limites globais antes da
  publicação dos filhos.
