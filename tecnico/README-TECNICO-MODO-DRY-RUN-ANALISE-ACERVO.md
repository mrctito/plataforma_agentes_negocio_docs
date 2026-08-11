# Manual técnico, executivo, comercial e estratégico: Modo dry-run de ingestão (análise de acervo sem ingerir)

> Recurso novo, implementado e validado em produção local em 2026-08-09/10 (fases A-D da
> campanha de ábacos DNIT). Documento nível 101: parte do zero, explica o conceito antes do
> código e termina em evidência de código. Toda cifra e todo número deste manual vêm de rodada
> real (10 PDFs de calibração e o acervo DNIT inteiro, 563 documentos), nunca de estimativa.

## 1. O que é esta feature

O modo dry-run de ingestão é uma chave YAML (`ingestion.analysis_only`) que faz a esteira real
de ingestão — a mesma que resolve credenciais, enumera documentos na origem, baixa arquivos,
aplica retry e roda dentro do Job Core — parar exatamente no momento em que ela abriria o PDF
para gravar conteúdo. Em vez de gravar, ela **mede** a estrutura de cada página e devolve um
relatório: quantas páginas o acervo tem, quantas parecem conter uma figura técnica (ábaco,
nomograma, gráfico, curva) e quanto custaria transcrever essas figuras com um modelo de visão.

Não é uma ferramenta separada, um script avulso ou uma segunda esteira de ingestão. É a **mesma**
classe que processa um documento em produção (`FileProcessingOrchestrator`, em
`src/ingestion_layer/file_pipeline_services.py`) com um único desvio condicional no topo do seu
método principal. Isso importa porque o dry-run herda de graça tudo que a ingestão real já sabe
fazer bem — autenticação com a origem (Google Drive, filesystem local, etc.), paralelismo por
documento, retry de rede, observabilidade por `correlation_id` — sem duplicar nenhuma dessas
responsabilidades.

O modo é genérico por construção: não conhece "DNIT" nem "ábaco" como conceitos de código. Ele
mede estrutura de PDF (segmentos vetoriais, vocabulário textual, cobertura de imagem embutida) e
delega o julgamento "isto é uma figura técnica que vale transcrever?" para um vocabulário
configurável por YAML. Qualquer acervo de PDFs — não só engenharia rodoviária — pode usar o mesmo
mecanismo para se perguntar "quanto vai custar processar isto com IA?" antes de gastar um
centavo.

## 2. Que problema ela resolve

Antes deste recurso, a única forma de saber quantas figuras técnicas um acervo continha e quanto
custaria transcrevê-las era **ingerir de verdade** (ou escrever um script avulso fora do produto,
como aconteceu na fase de investigação desta mesma campanha — `.sandbox/abacos/triagem.py`). As
duas opções tinham o mesmo defeito: gastar dinheiro ou tempo de engenharia para responder uma
pergunta que deveria ser respondida antes de gastar.

O caso real que originou o recurso: o acervo DNIT tem 563 documentos e 20.697 páginas. Transcrever
uma figura técnica com um modelo de visão como o `gpt-5.6-terra` custa, medido, cerca de
US$ 0,08 por figura (2.500 tokens de entrada + 6.200 de saída). Sem saber quantas páginas
realmente têm figura, a única forma de descobrir seria rodar a ingestão completa e ver a conta no
fim — arriscando um gasto de centenas de dólares para descobrir, depois, que boa parte das páginas
"prováveis" eram fotos de sinalização de trânsito, não ábacos de engenharia.

O dry-run resolve isso descolando a **pergunta de dimensionamento** ("quantas páginas têm figura,
quanto custa transcrever") da **decisão de gastar** ("agora vamos transcrever de verdade"). O
relatório sai do mesmo pipeline que rodaria a ingestão real, então o número que ele produz é
literalmente o número que a ingestão real produziria — não uma amostra, não uma extrapolação.
Isso importa porque a rodada real provou a diferença: a extrapolação inicial por amostra (10 PDFs
de 327 páginas médias) previa 24,5% de páginas selecionadas; a medição real do acervo inteiro
(563 documentos, média de 37 páginas por documento) bateu em 24,4% — quase idêntico só porque a
proporção se manteve, o que não era garantido de antemão.

## 3. Visão executiva

Para quem decide orçamento de IA, o dry-run transforma uma decisão às cegas em uma decisão
informada por dado real, sem custo residual: rodar o dry-run sobre os 563 documentos do acervo
DNIT custou aproximadamente US$ 0 (a etapa de medição estrutural não usa nenhum modelo de IA).
O resultado dessa rodada — 5.043 páginas candidatas de 20.697, custo de US$ 400,41 para
transcrever todas com o modelo principal — é o número que orienta a decisão de gastar ou não, e
quanto.

O risco que essa feature reduz é duplo: risco financeiro (gastar em transcrição de conteúdo que
não interessa) e risco operacional (descobrir, no meio de uma ingestão de produção paga, que o
acervo tem 20x mais páginas do que a amostra sugeria). Ambos os riscos já se manifestaram nesta
mesma campanha antes do dry-run existir — a segunda descoberta (563 vs. amostra de 10) só foi
possível medir com segurança porque o dry-run garante, por construção, que nada é gravado durante
a medição.

A capacidade que isso cria é reutilizável: qualquer novo acervo, qualquer novo cliente com
documentos ricos em conteúdo visual (manuais técnicos, laudos, plantas, catálogos) pode passar
pelo mesmo funil antes da primeira cobrança. Isso melhora previsibilidade de custo (o cliente
recebe um número antes de assinar), governança (a decisão de gastar fica documentada e
auditável por `correlation_id`) e velocidade de decisão (o relatório sai em horas, não em uma
rodada de ingestão completa que pode levar dias em acervos grandes).

## 4. Visão comercial

A dor do cliente que este recurso resolve é concreta: "eu tenho um acervo de X mil documentos e
não sei quanto vai custar processar isto com IA antes de assinar". Hoje a resposta correta e
honesta é "rode o dry-run e eu te devolvo o número exato em algumas horas, sem cobrar nada pela
medição" — isso é um diferencial real porque a alternativa do mercado costuma ser estimativa por
amostra (com a margem de erro que este próprio caso mostrou) ou cobrança por ingestão completa
"para descobrir".

O diferencial comercial defensável: o número devolvido não é uma projeção — é a contagem real do
acervo inteiro, com o mesmo pipeline que fará o trabalho de verdade depois. Isso é mensurável e
demonstrável em uma prova de conceito: mostrar o relatório de um acervo do próprio cliente antes
de qualquer contrato de transcrição.

O que **não** deve ser prometido comercialmente, porque o código não sustenta: (1) o dry-run não
identifica "isto é um ábaco" com certeza — a triagem estrutural (etapa 1, grátis) é generosa por
desenho e inclui falsos positivos (fotos, sinalização, tabelas densas); o número exato de figuras
verdadeiras só sai com o classificador visual (etapa 2, opcional, paga) ligado. (2) o dry-run não
é 100% gratuito quando o classificador está ligado — ele custa uma chamada de visão barata por
página aprovada na triagem (medido em ~US$ 3 para 5.043 páginas na rodada real). (3) o dry-run
não garante zero escrita em qualquer cenário sem a proteção específica contra `if_exists:
overwrite` (seção 14) — essa garantia existe porque foi implementada e testada, não por acidente.

## 5. Visão estratégica

Esta feature fortalece a plataforma porque generaliza uma capacidade que nasceu de um caso
específico (ábacos de engenharia rodoviária no acervo DNIT) para qualquer combinação de
acervo/formato/domínio, sem acoplamento de código ao vocabulário do cliente — o vocabulário de
figura, os limiares de triagem e a tabela de preço são todos parâmetros de YAML, não constantes
de código. Isso significa que o mesmo mecanismo serve o próximo cliente com um acervo de manuais
técnicos, catálogos de produto ou laudos de vistoria sem escrever uma linha de código nova.

Estrategicamente, o recurso também reforça um princípio de arquitetura da plataforma: capacidade
nova não vira fluxo paralelo. O dry-run não criou um segundo pipeline de ingestão, um segundo
mecanismo de fila ou uma segunda tabela de telemetria — ele estendeu o pipeline existente com um
desvio auditável (seção 9) e reaproveitou a telemetria, o Job Core e o catálogo de eventos já
existentes. Esse padrão de "estender sem duplicar" é o que mantém a plataforma evoluível sem
acumular dívida técnica a cada capacidade nova.

## 6. Conceitos necessários para entender

**Ingestão** é o processo que pega um documento de uma origem (Google Drive, filesystem local
etc.), extrai o conteúdo, transforma esse conteúdo em pedaços pesquisáveis (chunks) e grava tudo
em um vector store para consulta posterior por RAG. É um processo que **grava** dado permanente.

**Dry-run** ("corrida seca") é rodar um processo sem seu efeito colateral principal — aqui,
rodar a ingestão sem gravar. O termo é genérico em engenharia de software; nesta plataforma ele
tem uma implementação específica e comprovada por teste, não é apenas um modo de log verboso.

**Triagem estrutural** (etapa 1 do funil) é medir características objetivas de uma página de PDF
— sem nenhuma IA envolvida — que sugerem a presença de uma figura técnica: quantidade de
segmentos de desenho vetorial, presença de palavras como "ábaco" ou "gráfico" no texto da
página, e quanto da área da página é ocupada por uma imagem raster embutida. É rápida (poucos
milissegundos por página) e **generosa**: aprova qualquer página que *pareça* candidata, aceitando
falsos positivos porque o custo de errar para mais é zero (não gasta IA).

**Classificador visual** (etapa 2 do funil, opcional) é uma chamada real a um modelo de IA de
visão, uma por página já aprovada na etapa 1, perguntando "isto é mesmo um gráfico/ábaco
interpolável?". Ao contrário da etapa 1, esta custa dinheiro (a etapa 2 é, ela mesma, uma
transcrição em miniatura) e é a peneira que separa "parece figura" de "é figura confirmada".

**Job Core** é o mecanismo genérico de processamento assíncrono da plataforma (fila durável,
worker, lifecycle, ledger PostgreSQL). O dry-run não cria fila nem worker próprios — ele é um job
de ingestão comum, registrado e executado pelo mesmo Job Core que processa qualquer ingestão.
Detalhe completo: `README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md`.

**`vector_store.if_exists`** é a chave YAML que decide o que acontece com o alvo vetorial quando
a ingestão roda: `overwrite` apaga e recria do zero, `update` só insere/atualiza o que mudou,
`skip` recusa se o alvo já existir. Essa chave é central para entender a seção 14 (por que o
dry-run precisou de uma proteção específica).

**`correlation_id`** é o identificador único de uma execução ponta a ponta, usado para
reconstruir a história completa de uma rodada (o que aconteceu, quando, em que ordem) a partir do
log oficial. Todo achado numérico deste manual foi extraído de um `correlation_id` real.

## 7. Como a feature funciona por dentro

O fluxo começa exatamente como uma ingestão normal: o cliente dispara `POST /rag/ingest` com um
YAML cifrado contendo, entre outras configurações, o bloco `ingestion:` com `analysis_only: true`.
O boundary HTTP resolve o YAML pelo caminho oficial (o mesmo de qualquer ingestão), cria o
`correlation_id` e publica o job pai no Job Core.

O worker reivindica o job e o processo pai de ingestão inventaria os documentos da origem
(elegibilidade de fan-out, deduplicação de unidades) — essa etapa não sabe nem precisa saber que
está em modo de análise; ela só decide *quais* documentos existem, não *o que fazer* com eles.

Para cada documento, o Job Core materializa um job filho que executa a pipeline documental real.
É dentro dessa pipeline, no `FileProcessingOrchestrator.process_file`
(`src/ingestion_layer/file_pipeline_services.py:2045`), que o desvio acontece — um único `if` no
topo do método:

```python
async def process_file(self, request: FilePipelineRequest) -> FilePipelineExecutionResult:
    started_at = monotonic()

    # DESVIO ÚNICO do modo de análise de acervo (`ingestion.analysis_only`).
    # Com a flag ligada o arquivo é medido e o pipeline termina aqui: nada de
    # chunk, embedding, vector store ou registro documental. Com a flag desligada
    # (default), este `if` é a única diferença em relação ao caminho de sempre.
    if self._acervo_analysis_settings.analysis_only:
        return await self._analyze_without_ingesting(request)

    # ... caminho normal de ingestão (prepare -> build_chunks -> finalize) ...
```

`_analyze_without_ingesting` reaproveita a **mesma** aquisição do caminho oficial
(`DataSourceDocumentExecutor.acquire`, que só baixa e materializa o documento — sem nenhuma
decisão de ingestão) e chama o serviço de análise. O resultado vira uma entrada no coletor de
estatísticas do run (`request.stats_collector`) e o método devolve um
`FilePipelineExecutionResult(None, {...})` — o `None` no primeiro campo é o sinal estrutural de
"nada foi indexado"; sem chunks, o lote nunca chama `DocumentIndexingExecutor.finalize`.

Uma decisão de desenho importante está em **onde** o desvio acontece na ordem do pipeline: antes
da deduplicação e da política incremental (`if_exists: update`). Se o desvio acontecesse depois,
um documento já ingerido em uma rodada anterior seria pulado pela lógica de "não mudou" e sumiria
da contagem — o relatório mediria só o que falta ingerir, não o acervo inteiro. Medindo antes, o
relatório sempre reflete o acervo completo, independente do histórico de ingestões anteriores.

## 8. Divisão em etapas ou submodulos

### 8.1. Contrato YAML (`acervo_analysis_settings.py`)

Responsabilidade: ler as quatro chaves do bloco `ingestion:` do YAML, validar tipo e faixa de
cada valor, e devolver um objeto imutável (`AcervoAnalysisSettings`) com defaults calibrados
quando a chave está ausente. Recebe o YAML já resolvido pelo caminho oficial (nunca lê arquivo
sozinho). Entrega a configuração resolvida para todos os outros submódulos — é o único ponto do
sistema que sabe interpretar essas quatro chaves. Risco: chave com tipo errado (ex.: string onde
se espera número) — o contrato falha explícito citando o caminho completo da chave, nunca
silencia com um default. Diagnóstico: mensagem de erro traz o caminho YAML exato
(`ingestion.analysis_thresholds.min_vector_segments`, por exemplo).

### 8.2. Triagem estrutural (`acervo_structure_analyzer.py`)

Responsabilidade: medir, página a página, três sinais objetivos de um PDF já em memória
(segmentos vetoriais via PyMuPDF, vocabulário no texto extraído, cobertura de imagem embutida) e
aplicar a política de união de critérios. Recebe bytes crus do PDF e os limiares resolvidos pelo
contrato. Entrega um `DocumentStructureAnalysis` com a lista de páginas aprovadas e o motivo de
cada aprovação. Conecta-se ao serviço de documento (8.4) como sua primeira e obrigatória etapa.
Risco: PDF corrompido ou ilegível — vira `RuntimeError` tratado a montante (não derruba o run
inteiro); falha ao medir os retângulos de uma imagem específica não é engolida em silêncio — vira
contador `image_measurement_errors` no relatório. Diagnóstico: o relatório por documento traz
`total_pages`, `selected_pages`, `selection_reasons` e `image_measurement_errors` — dá para saber
exatamente por que um documento teve poucas ou muitas páginas selecionadas.

### 8.3. Classificador visual opcional (`acervo_page_classifier.py`)

Responsabilidade: para cada página aprovada na triagem, renderizar a página em imagem e fazer uma
chamada de visão barata pela porta canônica `ImageDescriptor` (`classify_image`, não
`describe_image` — resposta curta é custo curto). Recebe a lista de números de página aprovados
e os bytes do PDF. Entrega um `DocumentClassificationResult` com cada página marcada como
`confirmed`, `rejected` ou `unclassified`. Conecta-se depois da triagem (8.2) e antes do
agregado do relatório (8.5) — só roda quando `analysis_classifier.enabled: true`. Risco: erro de
render de página ou falha do provider de visão — **nunca** descarta a página (fail-closed): ela
vira `unclassified` com o erro registrado, e o relatório continua contando-a. Diagnóstico: eventos
`ingestion.acervo_analysis.classification.completed`/`.failed` no log, com `error_type` e
`error_message` estruturados quando há falha.

### 8.4. Serviço de análise de documento (`acervo_document_analysis_service.py`)

Responsabilidade: orquestrar as etapas 8.2 e 8.3 para **um** documento e decidir seu status final
(`analyzed`, `skipped` ou `failed`). Recebe o documento já adquirido (nome, tipo de conteúdo,
bytes) do ponto de desvio (seção 7). Entrega um `AcervoDocumentAnalysisOutcome` serializável.
Conecta-se ao orquestrador de arquivo (que o invoca) e ao relatório do acervo (que consome sua
saída). Risco: documento que não é PDF ou está vazio vira `skipped` com motivo literal — fato
operacional normal, não erro; documento cuja leitura falha vira `failed` com `logger.exception` e
não derruba a contagem do acervo inteiro. Diagnóstico: eventos
`ingestion.acervo_analysis.document.{analyzed,skipped,failed}`, um por documento, com o
`correlation_id` da execução.

### 8.5. Relatório e projeção de custo (`acervo_analysis_report.py`)

Responsabilidade: consolidar as saídas de todos os documentos processados **dentro de um mesmo
job** (via `stats_collector` compartilhado, o mesmo coletor que a ingestão normal já usa) em um
único relatório agregado, e calcular a projeção de custo por cenário e por modelo declarado na
tabela de preços. Recebe a lista de outcomes por documento e a configuração resolvida. Entrega um
dicionário JSON-serializável com totais, motivos de seleção agregados e `cost_projection`.
Conecta-se ao fechamento do job (8.6), que injeta esse relatório na telemetria. Risco/limitação
importante: este agregado só cobre os documentos processados **no mesmo processo/job** — em
acervos grandes que usam fan-out por documento (job filho por PDF), o agregado nativo cobre só um
documento por vez; a consolidação do acervo inteiro precisa somar os eventos
`document.analyzed` pelo `correlation_id` (detalhe completo na seção 9 e 15).

### 8.6. Corte do bootstrap do vector store (`document_persistence_manager.py`)

Responsabilidade: garantir que, em modo dry-run, **nenhuma** operação de bootstrap do alvo
vetorial aconteça — nem criação de coleção, nem aplicação da política `if_exists`, nem upsert do
registro de dataset. É o submódulo de segurança que impede que um "dry-run" apague dado real de
produção. Recebe a mesma configuração resolvida do contrato (8.1). Entrega um handle de vector
store **inerte** (sem `prepare_for_ingestion`, com `allow_destructive_vector_store_operations=
False`), que a esteira de conteúdo ainda exige como pré-condição estrutural, mas que nunca chega
a tocar o Qdrant/Postgres. Risco: esta é a peça mais crítica de todo o recurso — um defeito aqui
significa um "dry-run" que apaga dado real (ver seção 14 para o incidente real que motivou esta
proteção). Diagnóstico: evento
`ingestion.acervo_analysis.vector_store_bootstrap.skipped`, com o valor de `if_exists_policy`
que **estava declarado** no YAML e não foi executado — a prova de que a proteção funcionou.

## 9. Pipeline ou fluxo principal

1. **Cliente dispara `POST /rag/ingest`** com YAML cifrado contendo `ingestion.analysis_only:
   true`. O boundary resolve o YAML pelo caminho oficial, gera o `correlation_id` e devolve
   `202 Accepted` com `task_id` e o próprio `correlation_id` (corpo + header
   `X-Correlation-Id`). Nenhuma decisão de análise acontece ainda aqui — o boundary só publica o
   job pai no Job Core.

2. **Job Core materializa a árvore.** O processo pai de ingestão inventaria os documentos da
   origem e decide entre execução sequencial ou fan-out por documento (essa decisão não conhece
   `analysis_only` — é elegibilidade normal de origem). Cada documento vira um job filho.

3. **Bootstrap do alvo vetorial é interceptado.** Antes de processar qualquer arquivo, a esteira
   de conteúdo pede um handle de vector store
   (`DocumentPersistenceManagerMixin._prepare_ingestion_vector_store`). Em modo dry-run, este
   método **não** aplica `if_exists`, **não** cria coleção e **não** grava em
   `vector_dataset_master` — devolve um handle inerte. Evento:
   `ingestion.acervo_analysis.vector_store_bootstrap.skipped`.

4. **Cada documento entra em `FileProcessingOrchestrator.process_file`.** O desvio único (seção 7)
   redireciona para `_analyze_without_ingesting`, que reusa a aquisição normal (download,
   validação de tipo de conteúdo) e chama `AcervoDocumentAnalysisService.analyze`.

5. **Triagem estrutural roda para cada página** (etapa 1, grátis): segmentos vetoriais,
   vocabulário, cobertura de imagem embutida. Página aprovada por qualquer um dos três critérios
   entra na lista de selecionadas do documento.

6. **Classificador visual roda, se ligado** (etapa 2, opcional): uma chamada de visão por página
   aprovada, confirmando ou rejeitando. Fail-closed em caso de erro.

7. **Resultado do documento é registrado** no coletor de estatísticas do run e no log
   (`ingestion.acervo_analysis.document.analyzed/skipped/failed`). O `FilePipelineExecutionResult`
   volta com `inventory_status: "skipped"` e `reason: "analysis_only_no_ingestion"` — sinal
   estrutural de que nada foi indexado, mas o documento **foi processado com sucesso** pelo
   dry-run.

8. **Ao fechar o job (por documento, no caso de fan-out), o relatório agregado é construído**
   (`build_acervo_analysis_report`) e injetado em `metadata["acervo_analysis"]` do resultado
   da telemetria já existente — sem tabela nova. Evento:
   `ingestion.acervo_analysis.report.completed`, com `cost_projection`.

9. **O desfecho do job é decidido.** `documents_processed == 0` é o resultado correto do
   dry-run (nada foi ingerido de propósito). O sistema reconhece esse sinal
   (`result["acervo_analysis"]["analysis_only"] is True`) e fecha o job como `succeeded`, não
   como `failed` — essa correção é detalhada na seção 13.

10. **Consumo do relatório.** Para acervo pequeno (execução sequencial, um job só), o relatório
    do evento `report.completed` já é o agregado do acervo inteiro. Para acervo grande com
    fan-out por documento, o agregado do acervo precisa ser consolidado somando os eventos
    `document.analyzed` de todos os filhos pelo mesmo `correlation_id` — ver seção 15.

## 10. Decisões técnicas e trade-offs

**Flag YAML-first, não campo HTTP.** O request real de `POST /rag/ingest`
(`RagIngestionRequest`) não carrega configuração de ingestão — o YAML chega dentro de
`encrypted_data`, cifrado. Adicionar um campo HTTP separado criaria um segundo caminho de
configuração paralelo à resolução YAML oficial. Custo: a flag só é visível depois de decifrar o
YAML; ganho: zero caminho alternativo de configuração, YAML continua sendo a única fonte de
verdade.

**Desvio antes da deduplicação, não depois.** Já explicado na seção 7 — o trade-off é que o
dry-run sempre mede o acervo inteiro, mesmo que uma ingestão futura com `if_exists: update` só
processasse parte dele. Ganho: o relatório de custo é sempre uma medida honesta do total, nunca
subestimado por histórico de ingestões anteriores.

**Classificador usa `classify_image`, não `describe_image`.** A porta `ImageDescriptor` tem
exatamente duas operações; `describe_image` gera descrição exaustiva (caro), `classify_image`
devolve tags curtas (barato). O julgamento "isto é uma figura técnica?" fica em código puro
(`FigureTagPolicy`), não em um prompt escondido dentro de uma porta compartilhada com a ingestão
normal. Trade-off: a qualidade da classificação depende do vocabulário de tags configurado no
YAML (`figure_tags`), não de uma descrição rica — para a maioria dos acervos isso é suficiente e
mais barato; para casos ambíguos pode exigir ajuste fino do vocabulário.

**Preço nunca nasce no código.** A tabela `analysis_cost_table.models` nasce vazia por default;
se um modelo é declarado, `input_per_million`/`output_per_million` são obrigatórios — não existe
preço "padrão" a inventar. Trade-off: sem tabela declarada, o relatório mostra volume (páginas,
tokens) mas nenhuma cifra em dinheiro — o operador precisa saber o preço vigente do fabricante
para obter o número final. Ganho: nunca existe risco de uma decisão de orçamento nascer de um
preço desatualizado hardcoded.

**Store "inerte" em vez de `None`.** A primeira implementação da proteção contra bootstrap (T9)
devolvia `(None, None)` quando `analysis_only` estava ligado. Isso quebrou porque ~7 guardas nas
famílias de conteúdo (`local_content_family`, `remote_content_family`) exigem um handle de store
como pré-condição antes de despachar a pipeline — e não tinham nada a ver com o modo de análise.
A correção final devolve um vector store real, mas **nunca preparado** para ingestão
(`allow_destructive_vector_store_operations=False`, sem `prepare_for_ingestion`). Trade-off: um
objeto a mais é construído (custo desprezível, é operação puramente local); ganho: a proteção não
depende de nenhuma das sete guardas "lembrar" do modo de análise — o `BaseVectorStore` falha
fechado por construção se alguém tentar usá-lo para escrever.

## 11. Configurações que mudam o comportamento

Todas as quatro chaves vivem sob `ingestion:` no YAML e são resolvidas por
`resolve_acervo_analysis_settings` (`src/ingestion_layer/analysis/acervo_analysis_settings.py`).

- **`ingestion.analysis_only`** (booleano). Liga/desliga o modo inteiro. Ausente = `false`
  explícito (comportamento de ingestão normal, sem qualquer diferença). Presente com tipo
  diferente de booleano falha citando o caminho da chave.

- **`ingestion.analysis_thresholds`** (mapa opcional). Controla os limiares da triagem estrutural
  (etapa 1). Quatro subchaves, todas opcionais com default calibrado:
  - `min_vector_segments` (inteiro, default `400`): densidade vetorial que sozinha já aprova a
    página.
  - `min_vector_segments_with_vocabulary` (inteiro, default `130`): densidade vetorial menor,
    mas só conta se a página também tiver vocabulário de figura.
  - `min_embedded_image_coverage` (fração 0.0-1.0, default `0.05`): cobertura mínima de imagem
    raster embutida, também só conta com vocabulário presente.
  - `figure_vocabulary_terms` (lista de strings, default: ábaco, abaco, nomograma, gráfico,
    grafico, figura, curva): palavras que caracterizam vocabulário de figura técnica.
  Risco de errar: limiar muito baixo aprova páginas demais (relatório superestima custo); limiar
  muito alto perde páginas reais (relatório subestima). Os defaults foram calibrados contra um
  gabarito real (10 PDFs, 11 alvos conhecidos, 10 capturados).

- **`ingestion.analysis_classifier`** (mapa opcional). Liga a etapa 2 (paga). Subchaves:
  - `enabled` (booleano, default `false`).
  - `engine` (string, default `"openai"`): precisa estar no catálogo canônico de engines de
    descrição visual (`IMAGE_DESCRIPTION_ENGINES`); qualquer valor fora dele falha explícito.
  - `model` (string, **sem default**): obrigatório quando `enabled: true` — decisão de custo do
    tenant, nunca inventada pelo código.
  - `figure_tags` (lista de strings, default: ábaco, abaco, nomograma, gráfico, grafico,
    diagrama, curva, chart, graph, plot): vocabulário que confirma a página a partir das tags
    devolvidas pelo modelo.
  Risco se ausente: classificador desligado, funil roda só na etapa 1 (mais falsos positivos, mas
  zero custo). Risco se `enabled: true` sem `model`: configuração falha no boot do job — o
  operador é avisado antes de qualquer chamada de IA acontecer.

- **`ingestion.analysis_cost_table`** (mapa opcional). Controla o cálculo de custo do relatório.
  Subchaves:
  - `figure_input_tokens` (inteiro, default `2500`) e `figure_output_tokens` (inteiro, default
    `6200`): tokens médios gastos para transcrever **uma** figura — medidos numa bateria real,
    sobrescrevíveis se o próprio acervo tiver perfil diferente.
  - `models` (lista, default vazia): cada item precisa de `model`, `input_per_million` e
    `output_per_million` — todos obrigatórios quando o item existe. **Sem esta lista, o
    relatório entrega volume e nenhuma cifra.**
  Risco se ausente: relatório mostra apenas quantidade de páginas/tokens projetados, sem valor em
  dinheiro — o operador precisa declarar a tabela para obter o número final de orçamento.

## 12. Contratos, entradas e saídas

**Entrada do disparo** (boundary `POST /rag/ingest`, contrato `RagIngestionRequest`):
`encrypted_data` (YAML cifrado, obrigatório), `user_email` (obrigatório — auditoria),
`output_format` (`"json"`), `correlation_id` (enviado vazio de propósito — o boundary gera o
oficial), `execution_mode` (`"direct_async"` para rodar em background pelo Job Core),
`document_parallelism` (inteiro, paralelismo por documento da rodada). Nenhum campo específico de
`analysis_only` existe no contrato HTTP — a flag vive inteiramente dentro do YAML cifrado.

**Saída imediata do boundary** (HTTP `202`): `correlation_id` (corpo + header
`X-Correlation-Id`), `task_id`, `status`, `log_file_name`, `polling_url`. É o mesmo contrato de
qualquer disparo assíncrono de ingestão — o dry-run não tem resposta HTTP diferenciada.

**Saída do relatório por documento** (`AcervoDocumentAnalysisOutcome.as_report_entry()`):
`document_name`, `document_path`, `content_type`, `status` (`analyzed`/`skipped`/`failed`),
`reason` (quando aplicável), `total_pages`, `selected_pages`, `selected_page_numbers`,
`selection_reasons`, `image_measurement_errors`, e — se o classificador rodou —
`classification_engine`, `classification_model`, `classified_pages`, `confirmed_pages`,
`confirmed_page_numbers`, `rejected_pages`, `unclassified_pages`.

**Saída do relatório agregado** (`build_acervo_analysis_report`): `analysis_only`,
`analysis_thresholds`, `analyzed_documents`, `skipped_documents`, `failed_documents`,
`total_pages`, `selected_pages`, `selection_ratio`, `selection_reasons` (agregado por motivo),
`classification_enabled`, `documents` (lista completa por documento), `cost_projection`.

**Efeito colateral proibido por contrato:** nenhum chunk, embedding, ponto vetorial, registro em
`vector_active_documents`/`vector_active_document_chunks`/`vector_active_document_pages` e
nenhuma operação destrutiva/criadora de `if_exists` — provado por teste de arquitetura (seção
14). Efeito colateral aceito: uma linha nova em `vector_ingestion_runs` (telemetria do run,
propositalmente preservada) e, só para `vectorstore_id` inédito, uma linha "ficha-FK" em
`vector_dataset_master` (limite honesto explicado na seção 14).

## 13. O que acontece em caso de sucesso

O job fecha com status `succeeded` no Job Core, mesmo com `documents_processed == 0` — esse é o
resultado correto por contrato do dry-run, não uma falha disfarçada. A decisão está em
`ResultAnalysisMixin._analyze_result` (`src/services/mixins/result_analysis_mixin.py`):

```python
if documents_processed == 0:
    overall_status = "warning"
    warning_summary = "Nenhum documento processado"
    if self._is_acervo_analysis_dry_run(result):
        overall_status = "success"
        warning_summary = (
            "Modo de análise de acervo (ingestion.analysis_only): nenhum documento "
            "foi ingerido por contrato; o run entregou o relatório de análise."
        )
```

O sinal que aciona esse caminho (`_is_acervo_analysis_dry_run`) é o próprio relatório que o run
já anexa ao resultado (`result["acervo_analysis"]["analysis_only"] is True`) — não existe flag
paralela nem estado materializado novo para essa decisão.

Na rodada real do acervo DNIT inteiro (correlation `20260810_213056-01fef831...`, job
`ingest_d08ec90f`), o job pai levou cerca de 2h25 no worker local, com 564 jobs filhos, todos
`succeeded`, zero ingestão. O log da correlação tem 0 erros e 0 exceções.

O que o operador percebe: o job aparece como `succeeded` no Job Core (não em `failed`), o evento
`ingestion.acervo_analysis.report.completed` traz o relatório completo com `cost_projection`, e a
telemetria de run (`vector_ingestion_runs`) registra o fechamento sem tocar em nenhuma tabela de
conteúdo.

## 14. O que acontece em caso de erro

**Documento não é PDF ou está vazio** → `status: "skipped"`, `reason:
"analysis_only_unsupported_content_type"` ou `"analysis_only_empty_document"`. Registrado em
`info`, não é erro — fato operacional normal, o run continua.

**PDF ilegível (corrompido, binário inválido)** → `status: "failed"`, `error_type`,
`error_message` estruturados, `logger.exception` (não `logger.error`). O documento individual
falha mas **não derruba o run** — os demais documentos continuam sendo medidos.

**Classificador visual falha** (erro de render de página ou do provider de IA) →
**fail-closed, nunca descarta a página**: ela vira `unclassified` ("triada, não classificada")
com o erro no relatório. Perder a página silenciosamente subestimaria o acervo — o oposto do
propósito do modo.

**Configuração inválida no YAML** (tipo errado, `enabled: true` sem `model`, engine fora do
catálogo, modelo de preço sem `input_per_million`) → falha explícita **antes** de qualquer
chamada externa, citando o caminho completo da chave — o operador corrige o YAML sem precisar
adivinhar.

**Defeito real encontrado e corrigido em produção — dry-run perfeito fechava como `failed`.**
Antes da correção descrita na seção 13, um dry-run que mediu 10/10 documentos sem nenhum erro
ainda assim fechava o job como `failed`, com `AsyncDomainExecutionError: "Nenhum documento
processado"`. Causa: o `ResultAnalysisMixin` tratava `documents_processed == 0` sempre como
`warning`, e o boundary assíncrono convertia esse `warning` em exceção. Corrigido estendendo o
mesmo ponto de decisão (não criando um segundo). Coberto por teste de regressão com 7 casos,
incluindo prova de que a correção **não** é uma anistia geral — uma ingestão normal (não
dry-run) com zero documentos processados continua fechando como `warning`, como antes.

**Risco alto encontrado e corrigido — `if_exists: overwrite` apagaria dado real.** Este é o
achado mais crítico de toda a campanha e está detalhado na seção 22 (Limites e pegadinhas) e na
seção 15 (armadilha operacional). Resumo: antes da correção T9, o bootstrap do vector store
(`dataset_lifecycle_orchestrator.apply_if_exists_policy`) rodava **antes** do desvio por arquivo e
não consultava `analysis_only` — um dry-run com `if_exists: overwrite` no YAML apagaria o acervo
de produção inteiro antes de medir qualquer página. Foi corrigido cortando o funil único de
bootstrap (`_prepare_ingestion_vector_store`) e provado em produção local: rodada real com
`if_exists: overwrite` sobre um dataset **existente** resultou em zero coleções criadas ou
removidas, zero mutação no registro master (`updated_at` intacto — o discriminante mais forte de
que a política destrutiva não rodou) e medição idêntica à rodada de controle.

## 15. Observabilidade e diagnóstico

Todo achado numérico deste manual foi extraído do log oficial da correlação pelo módulo canônico
`src.log_analyzer` — nunca de leitura manual de arquivo. Oito eventos canônicos cobrem o ciclo de
vida completo do modo (catálogo fechado em `src/core/generated_log_event_catalog.py`, campos na
allowlist do grupo `acervo_analysis` em `src/ingestion_layer/telemetry/log_vocabulary.py`):

- `ingestion.acervo_analysis.mode.resolved` — a configuração resolvida no boot do orquestrador
  (limiares, flags), uma vez por processo.
- `ingestion.acervo_analysis.vector_store_bootstrap.skipped` — prova de que o bootstrap do vector
  store foi pulado, com o `if_exists_policy` que estava declarado e não foi executado.
- `ingestion.acervo_analysis.document.analyzed` — um por documento medido com sucesso, com
  `total_pages`, `selected_pages`, `selection_reasons`.
- `ingestion.acervo_analysis.document.skipped` — um por documento ignorado (não-PDF, vazio).
- `ingestion.acervo_analysis.document.failed` — um por documento cuja leitura falhou.
- `ingestion.acervo_analysis.classification.completed` / `.failed` — resultado da etapa 2, por
  documento, quando o classificador está ligado.
- `ingestion.acervo_analysis.report.completed` — o relatório agregado, uma vez por job (ver
  limitação de fan-out abaixo).

**Limitação real: o fan-out por documento fragmenta o agregado.** Em acervos grandes, o Job Core
distribui um job filho por documento — cada job filho fecha com **seu próprio**
`report.completed`, cobrindo só aquele documento (ou o pequeno lote que aquele job processou). Não
existe hoje um agregado único do acervo inteiro em `vector_ingestion_runs`. Na rodada real do
acervo DNIT (563 documentos), a telemetria de run **não** recebeu o agregado consolidado — a
consolidação foi feita somando, pelo `correlation_id` da execução inteira, todos os eventos
`document.analyzed` que qualquer job filho emitiu. Isso funciona porque `correlation_id` é
propagado por toda a árvore pai/filhos (o mesmo `correlation_id` do disparo original), e o log
oficial reconstrói a árvore inteira.

O snippet abaixo usa a API canônica do `log_analyzer` (`LogFileLocator` +
`LogRecordLoader` — nunca `glob`/`open` direto na pasta `logs/`, proibido por
`.claude/rules/log-instructions.md`) e é equivalente ao usado para consolidar a rodada real de
2026-08-10, que fechou em 563 eventos `document.analyzed`, 20.697 páginas totais e 5.043
selecionadas:

```python
from pathlib import Path

from src.log_analyzer.io.locator import LogFileLocator
from src.log_analyzer.io.loader import LogRecordLoader

correlation_id = "20260810_213056-01fef831-..."  # correlation_id do disparo POST /rag/ingest

locator = LogFileLocator()
primary_file, family_files = locator.locate(
    correlation_id=correlation_id,
    logs_dir=Path("logs"),
    include_rotated=True,
)

# max_records=None: não trunca (o run tinha centenas de milhares de registros, um
# acervo de 563 documentos não cabe no default de 50.000).
loader = LogRecordLoader(max_records=None, strict=False)
records = loader.load([primary_file, *family_files])

document_events = [
    record
    for record in records
    if record.get("event_name") == "ingestion.acervo_analysis.document.analyzed"
]

total_pages = sum(int(record.get("total_pages", 0)) for record in document_events)
selected_pages = sum(int(record.get("selected_pages", 0)) for record in document_events)

print(len(document_events), total_pages, selected_pages, selected_pages / total_pages)
# 563 20697 5043 0.24365...
```

Para investigação pontual (um documento, um erro específico), use o modo `query` do
`src.log_analyzer` (`python -m src.log_analyzer query --correlation-id <id> --question-type
last_error`, por exemplo) em vez de reler o log inteiro — ver
`docs/tecnico/README-TECNICO-LOG-ANALYZER.md`.

**Acompanhamento pelo Job Core**, sem depender do log, usando os scripts canônicos read-only
(`.claude/scripts/job-core/`):

```bash
source .venv/bin/activate
# Todos os jobs (pai + filhos) desta correlação:
python .claude/scripts/job-core/list_job_processing_jobs.py --correlation-id <correlation_id>
# Detalhe completo de um job específico (status, eventos, causa de falha se houver):
python .claude/scripts/job-core/print_job_processing_job_details.py --job-id <job_id>
```

**Distinguir causa raiz por camada:** erro de configuração YAML (falha imediata, antes de
qualquer chamada externa, mensagem cita o caminho da chave) vs. erro de origem/rede (retry
automático já embutido na aquisição, visível no log como tentativas sucessivas) vs. erro de
provider de IA (só na etapa 2, sempre fail-closed — página vira `unclassified`, nunca derruba o
run) vs. erro estrutural do PDF (documento vira `failed`, run continua).

## 16. Impacto técnico

Reduz o acoplamento entre "medir o custo de um acervo" e "processar um acervo": antes, a única
forma de medir era processar. Agora são a mesma pipeline com um parâmetro, o que elimina a
necessidade de manter um segundo código de estimativa (o script `.sandbox/abacos/triagem.py` da
fase de investigação virou código de produto testado, não ficou como script paralelo). A parte
que fica mais evoluível: qualquer melhoria na triagem estrutural ou no classificador beneficia
tanto o dry-run quanto — se um dia a mesma lógica for reutilizada — qualquer decisão de
qualidade dentro da ingestão real, porque vive no mesmo módulo `src/ingestion_layer/analysis/`,
isolado por responsabilidade única (SRP: settings, analyzer, classifier, report, service, cada um
em seu arquivo).

## 17. Impacto executivo

Reduz o risco de estouro de orçamento em contratos de transcrição de conteúdo visual, porque a
decisão de gastar passa a ser baseada em contagem real do acervo inteiro, não em amostra. Reduz o
gargalo de "preciso rodar a ingestão para saber quanto vai custar" — o dry-run do acervo DNIT
completo (563 documentos, 20.697 páginas) levou cerca de 2h25 e custou próximo de zero, contra o
custo e o tempo de uma ingestão real paga inteira. Aumenta previsibilidade porque o mesmo
mecanismo devolve o mesmo tipo de número (páginas, custo por modelo) para qualquer acervo novo,
sem depender de estimativa manual de engenharia.

## 18. Impacto comercial

Ajuda pré-venda ao permitir demonstrar, com o acervo real do prospect, um número de custo
concreto antes de qualquer assinatura — isso é mais forte que uma tabela de preço genérica por
página, porque mostra o funil real (quantas páginas o acervo tem, quantas são candidatas, quanto
custaria transcrever). O tipo de cliente que mais se beneficia é o que tem acervo grande e rico em
conteúdo visual não-textual (engenharia, manuais técnicos, laudos, catálogos ilustrados) — onde a
diferença entre "estimar por amostra" e "medir o acervo inteiro" é grande, como o próprio caso
DNIT mostrou (amostra de 10 PDFs projetava corretamente por sorte; a diferença de composição do
acervo real, se tivesse sido maior, teria produzido um número de orçamento errado).

## 19. Impacto estratégico

Fortalece a estratégia de produto porque transforma uma capacidade nascida de um problema
específico (custo de transcrição de ábacos de engenharia) em uma capacidade de plataforma
reutilizável para qualquer combinação de acervo e domínio, sem exigir código novo — só
configuração YAML. Isso reduz o custo de onboarding do próximo cliente com necessidade
semelhante e prepara a plataforma para oferecer "dimensionamento de custo antes de contratar" como
um passo padrão de qualquer proposta comercial que envolva processamento de conteúdo visual em
escala.

## 20. Exemplos práticos guiados

### 20.1. Exemplo de configuração YAML completa e comentada

Bloco real, documentado em `app/yaml/system/rag-config-modelo.yaml` (linhas 396-448), com os
quatro elementos do contrato e explicação campo a campo:

```yaml
ingestion:
  # Modo de ANÁLISE DE ACERVO (dry-run). Com `true`, a ingestão roda a esteira
  # normal (credenciais, enumeração, download, retry, Job Core), mede a estrutura
  # de cada página do documento e NÃO grava nada: sem chunk, sem embedding, sem
  # vector store, sem registro documental. Serve para dimensionar o acervo e o
  # custo antes de gastar. Ausente = false.
  analysis_only: true

  # Limiares da triagem estrutural (etapa 1, grátis). Uma página é selecionada
  # quando satisfaz QUALQUER um dos critérios (união): densidade vetorial pura,
  # densidade moderada + vocabulário de figura, ou imagem embutida cobrindo
  # parte relevante da página + vocabulário. Bloco inteiro é opcional; os
  # valores abaixo são os defaults calibrados contra um gabarito real.
  analysis_thresholds:
    min_vector_segments: 400
    min_vector_segments_with_vocabulary: 130
    min_embedded_image_coverage: 0.05
    figure_vocabulary_terms:
      - "ábaco"
      - "nomograma"
      - "gráfico"
      - "figura"
      - "curva"

  # Etapa 2 (opcional, paga): para cada página aprovada acima, uma chamada de
  # visão barata confirma se a página traz mesmo figura técnica. Desligado por
  # default. Com enabled:true, "model" é OBRIGATÓRIO (decisão de custo do
  # tenant) e "engine" precisa estar no catálogo de descrição visual.
  analysis_classifier:
    enabled: true
    engine: "openai"
    model: "gpt-5.6-luna"          # modelo barato de visão para a triagem
    figure_tags:
      - "ábaco"
      - "gráfico"
      - "nomograma"

  # Tabela de custo do relatório final: tokens médios por figura x preço por
  # modelo. PREÇO NUNCA TEM DEFAULT NO CÓDIGO — sem "models" declarado, o
  # relatório mostra só o volume (páginas/tokens), nenhuma cifra em dinheiro.
  analysis_cost_table:
    figure_input_tokens: 2500      # medição real: ~2,5k tokens de entrada/figura
    figure_output_tokens: 6200     # medição real: ~6,2k tokens de saída/figura
    models:
      - model: "gpt-5.6-terra"
        input_per_million: 2.00    # consultar preço oficial do fabricante
        output_per_million: 12.00
      - model: "claude-opus-5"
        input_per_million: 5.00
        output_per_million: 25.00
```

**Explicação 101 campo a campo:** `analysis_only` é o interruptor mestre — sem ele `true`, tudo
o resto do bloco é ignorado e a ingestão roda normal. `analysis_thresholds` ajusta a sensibilidade
da triagem grátis — subir os números aprova menos páginas (mais preciso, arrisca perder figura
real); descer aprova mais (mais seguro, gasta mais tempo/dinheiro na etapa 2 se ligada).
`analysis_classifier` é a decisão "quero pagar um pouco para saber o número exato de figuras, ou
aceito a estimativa generosa da triagem grátis?". `analysis_cost_table` é só matemática — sem
ela o sistema conta páginas e tokens, mas não converte em dinheiro.

### 20.2. Exemplo completo em JavaScript — disparar o dry-run ponta a ponta

O endpoint `POST /rag/ingest` exige o YAML cifrado (Fernet + RSA-OAEP, sessão efêmera por
`POST /crypto/session-key`). A plataforma já tem o módulo canônico de criptografia client-side —
`PayloadCrypto` (`app/ui/static/js/plataforma-agentes-ia-crypto.js`) — que **não deve ser
reimplementado**; o exemplo abaixo o reutiliza, no mesmo padrão usado pelas telas administrativas
reais da plataforma:

```html
<script src="/ui/static/js/plataforma-agentes-ia-crypto.js"></script>
<script>
async function dispararDryRun({ baseUrl, yamlContent, userEmail, documentParallelism }) {
  // 1. Criptografa o YAML client-side (Fernet+RSA-OAEP) via módulo canônico —
  //    a chave de sessão nasce em POST /crypto/session-key, dentro da própria função.
  const encryptedData = await PayloadCrypto.buildEncryptedData({
    yamlContent,
    filename: 'rag-config-dry-run.yaml',
    baseUrl,
  });

  // 2. Monta o corpo do request. Note: nenhum campo específico de dry-run aqui —
  //    o "ingestion.analysis_only: true" já está DENTRO do YAML cifrado.
  const body = {
    user_email: userEmail,
    output_format: 'json',
    correlation_id: '',            // vazio de propósito: o boundary gera o oficial
    execution_mode: 'direct_async', // roda em background pelo Job Core
    document_parallelism: documentParallelism,
    encrypted_data: encryptedData,
  };

  // 3. Dispara. A X-API-Key vem de authentication.access_key do próprio YAML alvo.
  const response = await fetch(`${baseUrl}/rag/ingest`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': window.__DRY_RUN_API_KEY__, // resolvido do YAML alvo, nunca hardcode
    },
    body: JSON.stringify(body),
  });

  // 4. O correlation_id oficial volta no corpo E no header — sempre capturar os dois.
  const correlationId = response.headers.get('X-Correlation-Id');
  const payload = await response.json();

  if (response.status !== 202) {
    throw new Error(
      `Falha ao disparar dry-run (HTTP ${response.status}, correlation_id: ${correlationId})`
    );
  }

  console.log('Dry-run disparado:', {
    correlationId,
    taskId: payload.task_id,
    pollingUrl: payload.polling_url,
  });
  return payload;
}
</script>
```

**Nível 101 do fluxo:** o browser nunca envia o YAML em texto claro — ele pede uma chave pública
efêmera ao servidor, criptografa o YAML localmente e só então envia o pacote cifrado. O servidor
devolve `202 Accepted` na hora (o trabalho roda em background) com um `correlation_id` que serve
para acompanhar o progresso depois, pelos scripts da seção 15 ou pelo `polling_url` devolvido.

Para automação server-side/CLI (sem browser), existe o script Python equivalente e canônico,
usado nas rodadas reais desta campanha:

```bash
source .venv/bin/activate
python .claude/scripts/dnit/dispatch_ingest_endpoint.py \
  --base-url http://localhost:5555 \
  --yaml-path .sandbox/tmp/dryrun-acervo-dnit/rag-config-dnit-600-dryrun.yaml \
  --user-email mrctito@gmail.com \
  --document-parallelism 5
```

## 21. Explicação 101

Imagine que você tem uma caixa com 500 livros e quer saber quantas páginas têm desenhos técnicos
antes de contratar alguém para redesenhar cada um digitalmente. Em vez de abrir livro por livro à
mão (lento) ou pagar para redesenhar tudo e só depois ver a conta (arriscado), você usa uma régua
rápida e grátis para folhear cada página e marcar "isso parece um desenho" — sem desenhar nada de
novo, só contando. Se quiser ter mais certeza antes de decidir, você paga um especialista barato
para dar uma olhada rápida só nas páginas marcadas e confirmar quais são desenhos de verdade. No
fim, você tem um número: "existem X páginas com desenho, custaria Y reais para redesenhar todas".
Só depois disso você decide se manda redesenhar. É exatamente isso que o modo dry-run faz com
PDFs e IA: mede antes de gastar, usando o mesmo processo que faria o trabalho de verdade depois.

## 22. Limites e pegadinhas

**A triagem da etapa 1 é generosa por desenho — não é a resposta final.** Sem o classificador
ligado, o número "páginas selecionadas" inclui falsos positivos reais (fotos, sinalização,
tabelas densas). Na rodada real, os documentos com mais páginas selecionadas eram manuais
IMAGE-HEAVY sem ábaco (sinalização de trânsito, projeto de pontes) — o classificador visual é o
que separa "parece figura" de "é ábaco/gráfico confirmado".

**`if_exists: overwrite` foi um risco real, corrigido — mas conhecer o histórico importa.** O
código atual protege contra isso (seção 14/15), comprovado por teste de arquitetura e por rodada
real. Ainda assim, ao configurar um dry-run, usar `if_exists: update` (nunca destrutivo por
natureza) é boa prática defensiva adicional, mesmo com a proteção do produto ativa.

**`vectorstore_id` inédito ainda cria uma linha em `vector_dataset_master`.** Não pelo bootstrap
(que não roda em dry-run), mas porque `vector_ingestion_runs.dataset_id` é `NOT NULL` com FK para
`vector_dataset_master(dataset_id)` — o próprio ledger de telemetria de run exige essa ficha para
existir. Para um dataset já existente (o caso normal de reanalisar um acervo em produção), nada
novo é criado ali. Eliminar essa linha residual para dataset inédito exigiria DDL em runtime, que
é proibido pela governança do projeto — por isso o limite fica documentado, não "corrigido".

**Fan-out fragmenta o agregado do acervo — a consolidação exige o log.** Não existe hoje um
relatório único de acervo em `vector_ingestion_runs` para runs com fan-out por documento. Quem
precisa do agregado do acervo inteiro precisa consolidar pelo log (seção 15), não só ler o último
evento `report.completed`.

**Documento pulado no dry-run nunca vira retentativa automática.** Igual à ingestão normal, skip
por qualquer motivo não é reenfileirado — o operador decide se quer rodar de novo.

## 23. Troubleshooting

**"O job fechou como `failed` mas o relatório parece completo."** Sintoma corrigido nesta
campanha (seção 14) — se reaparecer, é regressão: verificar se
`ResultAnalysisMixin._is_acervo_analysis_dry_run` ainda está sendo chamado e se
`result["acervo_analysis"]["analysis_only"]` está mesmo `True` no payload do run.

**"O relatório do acervo parece incompleto / só cobre alguns documentos."** Hipótese primária: o
run usou fan-out por documento e você está lendo só o `report.completed` do último job filho, não
o agregado. Confirmar pelo log (seção 15) somando `document.analyzed` por `correlation_id`.

**"O dry-run ficou parado / não sei se ainda está rodando."** Usar
`list_job_processing_jobs.py --correlation-id <id>` para ver o status de todos os jobs (pai +
filhos) da árvore, sem precisar reler o log inteiro.

**"O job falhou imediatamente com erro de `user_email`."** Configuração incompleta no YAML de
teste — YAMLs de produção esperam que a UI preencha `user_session.user_email` via OAuth; um
disparo direto pelo boundary precisa desse campo preenchido manualmente no YAML de teste.

**"O dry-run pode estar sendo executado por um worker errado."** Ver seção obrigatória abaixo
(armadilha operacional crítica) — verificar `ENVIRONMENT` antes de suspeitar de bug de produto.

### Armadilha operacional crítica — fila compartilhada por `ENVIRONMENT`

O Job Core usa uma fila **compartilhada** entre ambientes que rodam com o mesmo valor de
`ENVIRONMENT`; o `claim_next_run` (admissão de job por um worker) filtra por esse valor, não por
"local" vs. "cloud". Se o ambiente local e um worker remoto na nuvem estiverem configurados com o
**mesmo** `ENVIRONMENT` (ex.: ambos `prod`), o worker remoto pode reivindicar um job disparado
localmente — e se o código daquele worker remoto for **antigo** (anterior à existência do modo
dry-run), ele **ignora** `ingestion.analysis_only` inteiramente e executa uma ingestão real.

Isso não é hipotético: aconteceu na validação real desta campanha. Um dry-run disparado
localmente foi reivindicado, em 1 segundo, pelo worker de produção da nuvem — o job falhou por um
motivo benigno não relacionado (campo `user_email` vazio no YAML de teste), mas se não tivesse
falhado, a ingestão real teria rodado sobre dados de produção sem que ninguém pretendesse isso.

**Procedimento validado para evitar isso** (usado com sucesso na rodada real do acervo DNIT
inteiro):

1. Editar `ENVIRONMENT=development` no `.env` local (valor exclusivo, que nenhum worker remoto
   compartilha).
2. **Reiniciar API e worker locais** — `ENVIRONMENT` é lido e cacheado em memória no boot do
   processo; editar o `.env` sozinho, sem reiniciar, não tem efeito.
3. Confirmar que o processo novo está realmente no ar antes de disparar (comparar o horário de
   início do processo com o `mtime` de qualquer módulo alterado, se aplicável — ver
   `.claude/rules/ambiente-local.md`).
4. Disparar o dry-run normalmente. Com `ENVIRONMENT=development`, apenas o worker local pode
   reivindicar o job — nenhum worker de produção compartilha esse filtro.

Reverter `ENVIRONMENT` para o valor anterior é decisão do operador, quando a pilha local voltar a
precisar rodar contra o ambiente de produção normal (trocar `ENVIRONMENT` sem necessidade quebra
`X-API-Key` de chaves cadastradas só para `prod` — ver limite conhecido em memória operacional do
projeto).

## 24. Diagramas

Fluxo ponta a ponta (texto, porque o mecanismo é essencialmente sequencial e o desvio é o que
importa entender):

```
POST /rag/ingest (YAML cifrado, ingestion.analysis_only: true)
        |
        v
Boundary resolve YAML + gera correlation_id -> publica job pai no Job Core
        |
        v
Job Core materializa job(ns) filho(s) por documento (fan-out se elegível)
        |
        v
_prepare_ingestion_vector_store  --[analysis_only?]--> SIM: bootstrap pulado
        |                                                    (evento vector_store_bootstrap.skipped)
        | NÃO (ingestão normal)
        v
FileProcessingOrchestrator.process_file
        |
        +--[analysis_only?]--> SIM: _analyze_without_ingesting
        |                            |
        |                            v
        |                      acquire() [reusa aquisição normal]
        |                            |
        |                            v
        |                      AcervoDocumentAnalysisService.analyze
        |                            |
        |                    +-------+-------+
        |                    v               v
        |          Triagem estrutural   Classificador visual
        |          (etapa 1, grátis)    (etapa 2, opcional, pago)
        |                    |               |
        |                    +-------+-------+
        |                            v
        |                  outcome -> stats_collector
        |                            |
        |                            v
        |                  FilePipelineExecutionResult(None, {inventory_status: skipped})
        |
        | NÃO -> caminho normal (prepare -> build_chunks -> finalize -> grava)
        v
Fechamento do job: documents_processed == 0 + acervo_analysis.analysis_only=True -> succeeded
        |
        v
Relatório agregado em metadata["acervo_analysis"] + evento report.completed
```

## 25. Mapa de navegação conceitual

Leia este documento nesta ordem, conforme sua dúvida:

- **"O que é e por que existe?"** → seções 1-6.
- **"Como eu configuro isso no meu YAML?"** → seção 11 e o exemplo comentado da seção 20.1.
- **"Como eu disparo e acompanho uma rodada?"** → seção 20.2 (JavaScript/script) e seção 15
  (Job Core e log).
- **"O que garante que isso não vai apagar meus dados?"** → seções 9 (bootstrap cortado), 14
  (histórico do risco real) e 22 (limites honestos).
- **"Como eu leio o número final (páginas, custo)?"** → seção 15 (consolidação por
  `correlation_id`) e seção 26 (custos de referência).
- **"Algo deu errado, o que eu faço?"** → seção 23 (troubleshooting), com destaque para a
  armadilha de `ENVIRONMENT` compartilhado.
- **"Preciso explicar isso para alguém não-técnico"** → seções 3, 4, 17, 18 e 21.

## 26. Como colocar para funcionar

1. **Escreva ou copie o bloco `ingestion:` do YAML alvo** — use o exemplo comentado da seção
   20.1 como ponto de partida, sempre a partir de um YAML já operacional do acervo que você quer
   medir (nunca inventar estrutura fora dele).
2. **Decida se quer o classificador ligado.** Sem ele: relatório rápido, grátis, generoso (mais
   falsos positivos). Com ele: número mais preciso, custo pequeno mas real (ex.: ~US$ 3 para
   medir 5 mil páginas, na experiência real desta campanha).
3. **Declare a tabela de preço** (`analysis_cost_table.models`) com os valores oficiais vigentes
   do(s) fabricante(s) que você quer comparar — sem isso, o relatório só mostra volume.
4. **Confirme `ENVIRONMENT`** antes de disparar contra uma fila compartilhada com workers remotos
   — ver seção 23, armadilha operacional crítica. Este passo é obrigatório sempre que houver
   qualquer worker remoto rodando no mesmo `ENVIRONMENT` do disparo local.
5. **Dispare pelo boundary oficial** — pelo exemplo completo em JavaScript da seção 20.2 (browser)
   ou pelo script `dispatch_ingest_endpoint.py` (linha de comando/CI), nunca por um caminho
   paralelo.
6. **Acompanhe pelo Job Core** com os scripts da seção 15
   (`list_job_processing_jobs.py --correlation-id`, `print_job_processing_job_details.py
   --job-id`) até o job pai fechar `succeeded`.
7. **Leia o relatório.** Para run pequeno (sem fan-out), o último evento `report.completed` já é
   o agregado do acervo. Para acervo grande com fan-out, consolide pelo `correlation_id` com o
   snippet da seção 15.
8. **Decida.** O relatório entrega volume e (se a tabela de preço foi declarada) custo projetado
   por cenário e por modelo — a decisão de transcrever de verdade é sempre manual, do operador,
   nunca automática a partir do dry-run.

## 27. Exercícios guiados

1. Configure `ingestion.analysis_only: true` em uma cópia de um YAML de ingestão real que você já
   tenha, mantendo `vector_store.if_exists: "update"` (nunca `overwrite` num primeiro teste).
   Dispare contra um datasource pequeno (poucos PDFs locais) e confirme, pelos scripts da seção
   15, que o job fecha `succeeded` com `documents_processed == 0`.
2. Repita o exercício 1 ligando o classificador (`analysis_classifier.enabled: true`) com um
   modelo de visão barato. Compare `selected_pages` (etapa 1) com `confirmed_pages` (etapa 2) no
   relatório — a diferença é a taxa de falso positivo da triagem grátis para aquele acervo
   específico.
3. Some manualmente, pelo snippet da seção 15, os eventos `document.analyzed` de uma correlação
   que você disparou, e confirme que o total bate com o campo `total_pages` do último
   `report.completed` (em run sem fan-out, os dois números devem ser idênticos).

## 28. FAQ técnica 101

**1. O dry-run precisa de credencial diferente da ingestão normal?**
Não. Ele reusa exatamente a mesma resolução de credencial da origem (Google Drive, filesystem
local etc.) que a ingestão normal usa, porque é a mesma etapa de aquisição
(`DataSourceDocumentExecutor.acquire`).

**2. Preciso de um YAML separado só para o dry-run?**
Não é obrigatório, mas é boa prática: crie uma cópia do YAML de produção com
`ingestion.analysis_only: true` acrescentado, para não correr o risco de esquecer a flag ligada
numa rodada de ingestão real.

**3. O dry-run funciona com qualquer tipo de origem (Google Drive, filesystem local, web)?**
O desvio está no processamento por arquivo (`FileProcessingOrchestrator.process_file`), que roda
independente da origem — mas fan-out seguro por documento só é elegível para origens remotas
replayable (regra geral de paralelismo da ingestão, `src/CLAUDE.md` Parte 3), não específica do
dry-run.

**4. Como eu sei que o dry-run realmente não gravou nada, sem confiar só na minha leitura do
código?**
Consulte a fonte de verdade real (Qdrant + PostgreSQL) antes e depois da rodada — os scripts de
`.claude/scripts/qdrant/` e `.claude/scripts/postgresql/` fazem isso. Além disso, existe teste de
arquitetura dedicado (`test_02-28-58_acervo_analysis_invariant_gate.py`,
`test_02-28-60_acervo_analysis_bootstrap_invariant_gate.py`) que varre o código por AST e prova,
com discriminância verificada, que nenhum caminho de persistência é alcançável em modo dry-run.

**5. O que acontece se eu ligar o classificador sem declarar `model`?**
A configuração falha explícita no boot do job, antes de qualquer chamada de IA — mensagem cita
`ingestion.analysis_classifier.model é obrigatório quando ...enabled = true`.

**6. Os limiares default (400/130/0,05) servem para qualquer acervo?**
Foram calibrados contra um gabarito real de engenharia rodoviária (ábacos DNIT). Para um domínio
muito diferente (ex.: catálogo de produtos com fotos), pode valer a pena recalibrar
`analysis_thresholds` e `figure_vocabulary_terms` comparando o relatório contra uma amostra
conhecida manualmente antes de confiar no número final.

**7. Por que a triagem usa "OU" entre os três critérios, e não "E"?**
Porque os critérios capturam tipos diferentes de figura: densidade vetorial pura pega desenho
técnico vetorial; vocabulário + densidade moderada pega figura vetorial mais simples com contexto
textual; vocabulário + imagem embutida pega figura RASTER (escaneada/foto), que não tem nenhum
segmento vetorial. Na rodada real do acervo DNIT, o critério de imagem embutida sozinho respondeu
por 2.417 das páginas selecionadas — a maioria — porque a maior parte dos ábacos do acervo real
são imagens raster, não desenho vetorial.

**8. O que acontece com um documento protegido por senha ou corrompido?**
Vira `status: "failed"` no relatório daquele documento específico, com `error_type`/
`error_message`, sem derrubar a medição dos outros documentos do acervo.

**9. Dá para rodar o dry-run em paralelo com uma ingestão real do mesmo acervo?**
Tecnicamente sim (são jobs independentes no Job Core), mas não é recomendado testar assim: o
foco do dry-run é medir sem interferência; rodar os dois ao mesmo tempo sobre o mesmo alvo
vetorial mistura efeitos observados e dificulta o diagnóstico.

**10. O relatório do dry-run considera páginas de anexos ou só do PDF principal?**
O escopo do desvio é o processamento por arquivo — cada arquivo elegível na origem (inclusive
anexos, se a origem os trouxer como arquivos separados) passa pelo mesmo `process_file` e é
medido individualmente.

**11. Quanto tempo leva um dry-run?**
Depende do volume e da origem. A medição estrutural em si é rápida (milissegundos por página);
o tempo dominante é a aquisição (download) dos documentos na origem — no acervo DNIT completo
(563 documentos), o job levou cerca de 2h25, dominado pelo download, não pela medição.

**12. O que é `image_measurement_errors` no relatório?**
Um contador de falhas ao medir os retângulos de uma imagem específica dentro de uma página
(limitação da biblioteca PyMuPDF para certas imagens malformadas). Não é engolido em silêncio —
aparece no relatório, mas não impede a página de ser avaliada pelos outros critérios.

**13. Posso usar o dry-run para reprocessar um acervo já ingerido e ver o que mudou?**
O relatório do dry-run mede a estrutura, não compara com o que já foi ingerido — ele não é uma
ferramenta de diff. Ele sempre mede o acervo inteiro, independente de já ter sido ingerido antes
(seção 7 explica por que o desvio acontece antes da deduplicação).

**14. O evento `mode.resolved` aparece quantas vezes por run?**
Uma vez por processo/orquestrador (não por documento) — é a confirmação de que a configuração foi
lida corretamente no boot daquele job.

**15. Existe algum limite de tamanho de acervo para o dry-run?**
Não há limite de código específico — o limite prático é o mesmo da ingestão normal (tempo de
aquisição na origem, paralelismo configurado). O acervo real testado tinha 563 documentos e
20.697 páginas sem problema.

## 29. Checklist de entendimento

- [ ] Sei explicar, sem olhar o documento, a diferença entre a triagem estrutural (etapa 1) e o
      classificador visual (etapa 2), e por que a etapa 1 é generosa de propósito.
- [ ] Sei onde fica o desvio único no código (`FileProcessingOrchestrator.process_file`) e por
      que ele acontece antes da deduplicação.
- [ ] Sei explicar por que o bootstrap do vector store precisou de um corte separado do desvio
      por arquivo (seção 9, passo 3, e seção 14).
- [ ] Sei quais das quatro chaves YAML são obrigatórias e quais têm default calibrado.
- [ ] Sei por que o preço nunca tem default no código.
- [ ] Sei explicar a limitação do fan-out fragmentando o relatório e como consolidar pelo log.
- [ ] Sei descrever a armadilha operacional de `ENVIRONMENT` compartilhado e o procedimento para
      evitá-la.
- [ ] Sei onde consultar o status de um job pelo Job Core sem reler o log inteiro.

## 30. Evidências no código

- Contrato YAML e resolução: `src/ingestion_layer/analysis/acervo_analysis_settings.py`
  (`resolve_acervo_analysis_settings`, `AcervoAnalysisSettings`, `AcervoStructureThresholds`,
  `AcervoClassifierSettings`, `AcervoCostTable`, `AcervoModelPrice`).
- Triagem estrutural: `src/ingestion_layer/analysis/acervo_structure_analyzer.py`
  (`AcervoStructureAnalyzer`, `FigurePageSelectionPolicy`, `DocumentStructureAnalysis`).
- Classificador visual: `src/ingestion_layer/analysis/acervo_page_classifier.py`
  (`AcervoPageClassifier`, `FigureTagPolicy`, `create_acervo_page_classifier`).
- Serviço por documento: `src/ingestion_layer/analysis/acervo_document_analysis_service.py`
  (`AcervoDocumentAnalysisService`, `AcervoDocumentAnalysisOutcome`).
- Relatório e custo: `src/ingestion_layer/analysis/acervo_analysis_report.py`
  (`build_acervo_analysis_report`, `build_cost_projection`, `record_document_analysis`).
- Desvio no orquestrador de arquivo: `src/ingestion_layer/file_pipeline_services.py`
  (`FileProcessingOrchestrator.process_file`, `_analyze_without_ingesting`, linhas ~2045-2053).
- Corte do bootstrap do vector store: `src/ingestion_layer/document_persistence_manager.py`
  (`DocumentPersistenceManagerMixin._prepare_ingestion_vector_store`, linhas ~1248-1298).
- Correção do desfecho do job: `src/services/mixins/result_analysis_mixin.py`
  (`ResultAnalysisMixin._is_acervo_analysis_dry_run`, `_analyze_result`).
- Vocabulário de log e eventos: `src/ingestion_layer/telemetry/log_vocabulary.py` (grupo
  `acervo_analysis`, constantes `INGESTION_ACERVO_ANALYSIS_*_EVENT`).
- YAML de exemplo documentado: `app/yaml/system/rag-config-modelo.yaml` (bloco `ingestion:`,
  linhas 396-448).
- Testes: `tests/unit/ingestion_layer/analysis/test_02-28-53_acervo_analysis_settings.py` até
  `test_02-28-60_acervo_analysis_bootstrap_invariant_gate.py` (8 arquivos; gates de invariante
  arquitetural em `test_02-28-58_*` e `test_02-28-60_*`); regressão do desfecho do job em
  `test_02-28-59_acervo_analysis_run_outcome.py`.
- Script de disparo canônico: `.claude/scripts/dnit/dispatch_ingest_endpoint.py`.
- Scripts de acompanhamento pelo Job Core: `.claude/scripts/job-core/list_job_processing_jobs.py`,
  `.claude/scripts/job-core/print_job_processing_job_details.py`.
- Módulo canônico de criptografia client-side: `app/ui/static/js/plataforma-agentes-ia-crypto.js`
  (`PayloadCrypto.buildEncryptedData`, `PayloadCrypto.acquireSessionKey`).
- Consulta de log oficial: `src/log_analyzer/io/locator.py::LogFileLocator`,
  `src/log_analyzer/io/loader.py::LogRecordLoader`.
- Plano executado com evidência tarefa a tarefa (fases A-D):
  `docs/.interno/.planos/abacos-dnit-yaml-600/plano--2026-08-09--modo-dry-run-analise-acervo.md`.
- Diário da campanha com a execução real no acervo DNIT (números, custos, incidentes):
  `docs/.interno/.planos/abacos-dnit-yaml-600/campanha--2026-08-05--solucao-ingestao-abacos.md`
  (RODADA 5).
