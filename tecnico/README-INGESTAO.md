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

## 3. Componentes que governam o fluxo

Os componentes abaixo são os pontos mais importantes para entender o comportamento real da ingestão.

- `IngestionService`: fachada principal que recebe o pedido, resolve política de execução e decide se haverá fan-out.
- `IngestionRuntimePreparationService`: preparador compartilhado que organiza o runtime comum antes do processamento pesado. Ele reduz duplicação e evita que cada processor monte seu próprio mundo operacional.
- `ContentIngestionOrchestrator`: orquestrador que coordena a esteira quando o lote segue pelo caminho direto.
- `DocumentFanoutCoordinator`: coordenador responsável pelo fan-out documental quando o lote precisa ser quebrado em filhos por documento.

Em linguagem simples: a fachada organiza, o preparador monta o contexto, o orquestrador conduz o fluxo direto e o `DocumentFanoutCoordinator` cuida da versão paralela por documento.

## 4. Contrato operacional mínimo do run

Aceitação assíncrona não significa sucesso final. Na maioria das vezes, significa só que o pedido foi aceito e entregue ao runtime assíncrono.

Por isso a leitura operacional do lote depende de duas rotas de consulta:

- `/ingestion-runs/query`: lista runs, estados agregados e visão operacional resumida.
- `/ingestion-runs/detail`: mostra o detalhe operacional do run, incluindo filhos, progresso e snapshots relevantes.

Quando existe fan-out por documento, um dos contratos mais úteis do detalhe operacional é `fanout_overview`. Esse campo resume o estado agregado do pai e dos filhos, incluindo volume de documentos, distribuição de status, sinais de cancelamento e informações usadas pela UI administrativa.

## 5. Fan-out documental sem job fantasma

Fan-out documental existe para throughput e isolamento. Ele permite que um lote com vários documentos seja dividido em filhos menores, sem perder a ideia de um run pai agregado.

Mas existe uma trava anterior ao primeiro filho: para o mesmo `tenant_code + vectorstore_id`, só pode existir um run pai ativo por vez. Em termos simples, pode haver paralelismo dentro do lote, mas não pode haver dois lotes pais diferentes brigando pelo mesmo acervo vivo ao mesmo tempo.

Essa regra ignora `vector_store.if_exists`. O campo ainda decide como o dataset vivo será tratado quando o lote for válido, mas ele não serve para liberar concorrência entre dois pais. Se já existe um pai ativo, o comportamento correto é rejeitar a nova admissão até o run anterior terminar ou ser reconciliado.

Mas a fila não decide regra de negócio. RabbitMQ e Dramatiq apenas transportam o envelope. Eles podem reentregar mensagem antiga ou acordar um filho atrasado. Isso é normal em sistemas assíncronos.

Quem decide se um filho ainda pode executar é o plano de controle durável no PostgreSQL. No código atual, essa decisão está centralizada em `DocumentFanoutExecutionGate`.

O contrato prático é este:

- a fila entrega a mensagem;
- a gate consulta o estado canônico do run pai e do fan-out;
- só então o filho pode executar, republicar ou promover outro `queued`.

Se o pai estiver `cancelled`, `cancelling`, `completed` ou `failed`, o filho deve ser bloqueado. Se o plano de controle estiver indisponível, o comportamento correto é falhar fechado. Em linguagem simples: sem confirmação do banco, não existe trabalho caro de PDF.

Isso torna o redelivery seguro. A mensagem velha pode até acordar, mas não consegue gerar OCR, parsing, chunking ou indexação sem autorização durável.

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

- filhos `queued` e `retrying` podem ser terminalizados no plano de controle sem iniciar trabalho caro;
- mensagens antigas do broker podem até acordar, mas a gate canônica bloqueia a continuação;
- filho já dentro de OCR, download ou parsing pesado depende de checkpoint de cancelamento e drenagem cooperativa;
- se o broker não expuser contagem por run, a UI deve mostrar essa limitação como falta de visibilidade física por lote, não como falha do cancelamento.

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

## 7. Como estudar o resto da ingestão

Depois deste documento, a leitura mais produtiva é esta:

1. `README-INGESTAO-INDICE.md`, para navegar pelos documentos do domínio.
2. `README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO.md`, para visão funcional do PDF.
3. `README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md`, para pipeline técnico do PDF e a gate canônica do fan-out.
4. `README-CONCEITUAL-INGESTAO-HTML-PIPELINE-COMPLETO.md` e `README-TECNICO-INGESTAO-HTML-PIPELINE-COMPLETO.md`, para HTML e web.
5. `README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO.md` e `README-TECNICO-INGESTAO-EXCEL-PIPELINE-COMPLETO.md`, para planilhas.
6. `README-CONCEITUAL-INGESTAO-JSON-PIPELINE-COMPLETO.md` e `README-TECNICO-INGESTAO-JSON-PIPELINE-COMPLETO.md`, para JSON estruturado.

## 8. Explicação 101

Pense na ingestão como um centro de triagem.

- Uma equipe recebe o pedido.
- Outra separa os itens por tipo.
- Outra prepara o ambiente certo para cada item.
- Outra acompanha se o lote ainda está autorizado a seguir.

Se a fila entregar uma caixa antiga, isso não significa que a caixa ainda pode entrar na esteira. Antes de trabalhar, o sistema confere no banco se aquele lote continua válido. Esse é o papel do contrato operacional da ingestão.
