# Log Analyzer — Documentação Técnica, Operacional e de Diagnóstico

## 1. Entrypoints e formas de uso

O módulo possui três pontos de entrada:

### CLI
```bash
python -m src.log_analyzer query \
    --question-type has_error \
    --correlation-id abc123 \
    --logs-dir /app/logs

python -m src.log_analyzer query \
    --question-type last_event \
    --correlation-id abc123 \
    --since 2026-06-01T10:00:03+00:00 \
    --logs-dir /app/logs

python -m src.log_analyzer                  # contrato JSON raiz da CLI inteira (modo agente)
python -m src.log_analyzer --help           # ajuda textual para humano
python -m src.log_analyzer query --list-types     # lista todos os tipos disponíveis
python -m src.log_analyzer query --schema         # contrato detalhado do subcomando query
python -m src.log_analyzer analyze --correlation-id abc123 --logs-dir /app/logs
python -m src.log_analyzer stats --logs-dir /app/logs
```

### Python — análise profunda
```python
from pathlib import Path
from src.log_analyzer import LogAnalyzerService, IndividualAnalysisRequest

service = LogAnalyzerService()
result = service.analyze_individual(
    IndividualAnalysisRequest(
        correlation_id="abc123",
        logs_dir=Path("/app/logs"),
    ),
    correlation_id="diagnostico-interno",
)
```

### Python — consulta rápida
```python
from pathlib import Path
from src.log_analyzer import LogQueryService, FastLogQueryRequest, QuestionType

service = LogQueryService()
result = service.query(
    FastLogQueryRequest(
        question_type=QuestionType.HAS_ERROR,
        correlation_id="abc123",
        logs_dir=Path("/app/logs"),
    )
)
print(result.success, result.answer, result.code)
print(result.metadata.get("answer_timestamp"))

incremental = service.query(
    FastLogQueryRequest(
        question_type=QuestionType.LAST_EVENT,
        correlation_id="abc123",
        logs_dir=Path("/app/logs"),
        since=datetime.fromisoformat("2026-06-01T10:00:03+00:00"),
    )
)
print(incremental.answer)
print(incremental.metadata["since_timestamp"])
print(incremental.metadata["answer_timestamp"])
print(incremental.metadata["last_record_timestamp"])
```

### Import público completo
```python
from src.log_analyzer import (
    # Serviços
    LogAnalyzerService,
    LogQueryService,
    # Contratos de entrada — análise profunda
    IndividualAnalysisRequest,
    AggregatedAnalysisRequest,
    # Contratos de entrada/saída — consulta rápida
    FastLogQueryRequest,
    FastLogQueryResult,
    # Contratos de saída — análise profunda
    AnalysisResult,
    SerializedAnalysisResult,
    AnalysisError,
    # Catálogos
    QuestionType,
    QueryCode,
    # Exceções
    LogAnalyzerError,
    CorrelationIdMissingError,
    LogFileNotFoundError,
    LogRecordLimitExceededError,
    LogsDirNotFoundError,
    AnalysisEngineError,
)
```

---

## 2. Arquitetura do módulo

O módulo segue arquitetura hexagonal: domínio isolado da infraestrutura, todos os
contratos são dataclasses Python puras, nenhuma dependência de settings globais.

```
src/log_analyzer/
├── __init__.py                  ← API pública do módulo
├── __main__.py                  ← CLI via python -m src.log_analyzer
├── cli_contract.py              ← contrato JSON raiz/subcomandos para descoberta por agente
├── service.py                   ← LogAnalyzerService (análise profunda)
├── query_service.py             ← LogQueryService (consulta rápida)
├── filter_service.py            ← LogFilterService (subcomando filter — filtro objetivo)
├── contracts.py                 ← Todos os contratos, enums e dataclasses
├── errors.py                    ← Hierarquia de exceções de domínio
├── schema_registry.py           ← catálogo conhecido de campos/eventos (schema fail-close)
├── source_field_contract.py     ← campos canônicos consumidos e aliases proibidos
├── analysis/
│   ├── engine.py                ← ABCs: AbstractLogEngine, AbstractAnalysisEngine,
│   │                               AbstractOperationalQueryEngine, AnalysisDialect, AnalysisPlugin
│   ├── plugins.py               ← 7 plugins padrão do pipeline V1.0 (DEFAULT_PLUGINS)
│   ├── request_timeline.py      ← janela HTTP, duração autoritativa e lanes concorrentes
│   ├── query_engine.py          ← FastQueryEngine + 9 handlers
│   ├── filter_engine.py         ← engine do subcomando filter
│   ├── integrity.py             ← detecção de foreign_correlation_id (campo problems)
│   ├── pandas_engine.py         ← PandasAnalysisEngine + PandasAnalysisDialect
│   ├── registry.py              ← AnalysisRegistry
│   └── insights/                ← handlers de insight de ingestão e RAG
│       ├── handlers_ingestion.py       ← IngestionPerformanceQueryHandler
│       ├── handlers_rag_diagnosis.py   ← RagDiagnosisQueryHandler
│       ├── extractors_performance.py, base.py, intervals.py, stats.py
├── io/
│   ├── locator.py               ← LogFileLocator
│   └── loader.py                ← LogRecordLoader (iterador JSONL)
├── parsing/field_extractor.py   ← extração canônica de campos do registro
├── report/serializer.py         ← serialização JSON-safe dos resultados
├── utils/time_utils.py          ← helpers de timestamp
└── validation/                  ← verdade manual e snapshot para validar a CLI
    ├── manual_truth.py, snapshot.py, cli_runtime.py, contracts.py
```

---

## 3. Pipeline de análise profunda

**Classe:** `LogAnalyzerService` (`service.py`)

Pipeline completo de execução para `analyze_individual`:

```
IndividualAnalysisRequest
    │
    ├── LogFileLocator.locate()       → resolve (arquivo primário, família completa)
    │                                   Estratégia: manifest → CanonicalLogReader
    ├── LogRecordLoader.load()        → list[dict]  (carrega tudo em memória)
    │                                   Limite: max_records (default 50.000)
    ├── PandasAnalysisEngine.load()   → constrói DataFrame
    │
    ├── AnalysisRegistry.plugins()    → instancia os 7 plugins (DEFAULT_PLUGINS)
    │
    ├── Plugin 1: ErrorWarningPlugin      → erros, warnings explicáveis e gaps de schema
    ├── Plugin 2: PerformancePlugin       → dict com duration_stats, slow_operations
    ├── Plugin 3: SlowestSegmentsPlugin   → dict com maiores gaps entre eventos consecutivos
    ├── Plugin 4: StageTimingPlugin       → dict com tempo entre etapas (stage) do pipeline
    ├── Plugin 5: EventFlowPlugin         → dict com unique_events, top_events
    ├── Plugin 6: ComponentActivityPlugin → dict com active_components, component_activity
    └── Plugin 7: VectorDocumentIntegrityPlugin
                                           → integridade vetorial por documento
                                       │
                                       ▼
                                 AnalysisResult
```

O método `analyze_aggregated` percorre todos os arquivos `.json`/`.log` do `logs_dir`,
aplica filtro de janela temporal (`from_dt`, `to_dt`) e executa o mesmo pipeline de
plugins em cada arquivo, agregando os resultados.

### Construção padrão (sem injeção de dependências)
```python
service = LogAnalyzerService()
```

### Construção com injeção (para testes)
```python
from src.log_analyzer.io.locator import LogFileLocator
from src.log_analyzer.analysis.pandas_engine import PandasAnalysisEngine
from src.log_analyzer.analysis.registry import AnalysisRegistry

service = LogAnalyzerService(
    locator=LogFileLocator(),
    engine=PandasAnalysisEngine(),
    registry=AnalysisRegistry.default(),
)
```

---

## 4. Pipeline de consulta rápida

**Classe:** `LogQueryService` (`query_service.py`)

Pipeline com parada antecipada via iterador:

```
FastLogQueryRequest
    │
    ├── LogFileLocator.locate()          → resolve (arquivo primário, família)
    │
    ├── LogRecordLoader.iter_records()   → Iterator[dict]  (streaming, sem carregar tudo)
    │
    └── FastQueryEngine.execute_query()  → despacha por question_type para o Handler certo
            │
            ├── ErrorQueryHandler        → has_error, has_exception, last_error,
            │                              last_exception, error_count, has_critical_error
            ├── FlowStatusQueryHandler   → flow_started, flow_finished, flow_failed,
            │                              flow_succeeded, flow_status, last_event
            ├── StuckQueryHandler        → is_stuck, has_start_without_end
            ├── TimeoutRetryQueryHandler → has_timeout, has_retry
            ├── ComponentQueryHandler    → component_failed
            ├── ParallelismQueryHandler  → had_real_parallelism,
            │                              max_real_concurrency,
            │                              parallelism_evidence,
            │                              had_fanout_without_parallelism,
            │                              parallelism_blocker_reason
            ├── IngestionPdfProgressQueryHandler   → ingestion_pdf_progress
            ├── IngestionPerformanceQueryHandler   → ingestion_performance
            └── RagDiagnosisQueryHandler           → rag_diagnosis
                                           │
                                           ▼
                                     FastLogQueryResult
```

Os dois últimos handlers vivem em `analysis/insights/` (`handlers_ingestion.py` e
`handlers_rag_diagnosis.py`) e são importados tardiamente pelo `FastQueryEngine` para
evitar ciclo de import. `rag_diagnosis` auto-detecta se a execução foi agentic (DeepAgent)
ou Q&A/retrieval clássico e responde com o `QueryCode` correspondente.

A consulta rápida **nunca levanta exceção**. Falhas são encapsuladas em
`FastLogQueryResult(success=False, error=...)`.

### Parada antecipada por tipo de pergunta

Perguntas booleanas simples param no primeiro match encontrado:
- `has_error` → para ao encontrar o primeiro registro ERROR/CRITICAL
- `flow_started` → para ao encontrar o primeiro evento de início
- `flow_failed` → para ao encontrar o primeiro status de falha
- `has_timeout` → para ao encontrar a primeira menção de timeout
- `has_retry` → para ao encontrar o primeiro evento de retry

Perguntas de contagem (`error_count`) e de estado completo (`flow_status`) precisam varrer
o log inteiro para garantir a resposta correta.

---

## 5. Catálogo de QuestionType — 25 tipos em 8 grupos

```
Grupo: erros
  has_error              → bool: houve qualquer erro no log?
  has_exception          → bool: houve exceção registrada?
  last_error             → str: mensagem do último erro encontrado
  last_exception         → str: mensagem da última exceção encontrada
  error_count            → int: total de registros ERROR + CRITICAL
  has_critical_error     → bool: houve erro de nível CRITICAL?

Grupo: status_do_fluxo
  flow_started           → bool: fluxo foi iniciado?
  flow_finished          → bool: fluxo terminou (qualquer status)?
  flow_failed            → bool: fluxo falhou?
  flow_succeeded         → bool: fluxo teve sucesso?
  flow_status            → str: "running" | "succeeded" | "failed" | "finished" | "unknown"
  last_event             → str: nome do último event_name registrado

Grupo: travamento
  is_stuck               → bool: há início sem fim correspondente?
  has_start_without_end  → bool: equivalente a is_stuck

Grupo: timeout_retry
  has_timeout            → bool: houve timeout ou mensagem de timeout?
  has_retry              → bool: houve tentativa de retry?

Grupo: componente
    component_failed       → bool: o componente especificado falhou?

Grupo: paralelismo_real
    had_real_parallelism            → bool: houve sobreposição temporal real entre execuções?
    max_real_concurrency            → int: qual foi o pico real de concorrência observado?
    parallelism_evidence            → list[dict]: quais evidências estruturadas provam a sobreposição?
    had_fanout_without_parallelism  → bool: houve fanout configurado sem paralelismo real?
    parallelism_blocker_reason      → str: qual blocker formal explica a ausência de paralelismo?

Grupo: ingestao
    ingestion_pdf_progress          → dict: progresso por PDF (concluídos, abortados, em andamento)
    ingestion_performance           → dict: latência real, paralelismo, balanceamento e throughput da ingestão

Grupo: diagnostico_rag
    rag_diagnosis                   → dict: diagnóstico do RAG (auto-detecta agentic DeepAgent vs Q&A/retrieval clássico)
```

Obtendo o catálogo programaticamente:
```python
from src.log_analyzer.contracts import QuestionType
all_types = sorted(QuestionType.all_types())
```

---

## 6. Catálogo de QueryCode — 25 códigos de resposta estáveis

Os códigos são strings constantes retornados em `FastLogQueryResult.code`. São estáveis
entre versões — podem ser comparados programaticamente.

```python
QueryCode.FOUND                  # evidência encontrada (genérico)
QueryCode.NOT_FOUND              # evidência não encontrada
QueryCode.FLOW_STARTED           # fluxo foi iniciado
QueryCode.FLOW_RUNNING           # fluxo em andamento
QueryCode.FLOW_FINISHED          # fluxo finalizou
QueryCode.FLOW_SUCCEEDED         # fluxo teve sucesso
QueryCode.FLOW_FAILED            # fluxo falhou
QueryCode.FLOW_UNKNOWN           # status não determinável
QueryCode.STUCK                  # início sem fim detectado
QueryCode.NOT_STUCK              # nenhum início sem fim
QueryCode.TIMEOUT_FOUND          # timeout detectado
QueryCode.RETRY_FOUND            # retry detectado
QueryCode.COMPONENT_FAILED       # componente com falha detectada
QueryCode.PARALLELISM_FOUND      # paralelismo real comprovado por janela temporal
QueryCode.FANOUT_WITHOUT_PARALLELISM  # fanout observado sem sobreposição temporal real
QueryCode.INGESTION_PROGRESS          # resposta de progresso de ingestão por PDF
QueryCode.INGESTION_PERFORMANCE       # resposta de performance de ingestão
QueryCode.RAG_DIAGNOSIS_AGENTIC       # execução RAG detectada como agentic (DeepAgent)
QueryCode.RAG_DIAGNOSIS_CLASSIC       # execução RAG detectada como Q&A/retrieval clássico
QueryCode.RAG_DIAGNOSIS_UNKNOWN       # tipo de execução RAG não determinável
QueryCode.UNSUPPORTED_QUESTION_TYPE   # tipo de pergunta não suportado
QueryCode.MISSING_CORRELATION_ID      # correlation_id ausente
QueryCode.LOG_NOT_FOUND               # arquivo de log não encontrado
QueryCode.SCHEMA_CONTRACT_VIOLATION   # violação do contrato estrutural do log
QueryCode.QUERY_ERROR                 # erro interno do analisador durante a consulta
```

---

## 7. Contratos de entrada e saída

### FastLogQueryRequest
```python
@dataclass
class FastLogQueryRequest:
    question_type: str        # Obrigatório. Use QuestionType.* — ex: QuestionType.HAS_ERROR
    correlation_id: str       # Obrigatório. ID da execução a consultar.
    logs_dir: Path            # Obrigatório. Diretório onde os logs residem.
    component: str | None     # Opcional. Filtra por campo "component" nos registros.
    phase: str | None         # Opcional. Filtra por campo "stage".
    max_records: int | None   # Omitido = lê todo o log disponível.
    max_evidence: int         # Default: 3. Máximo de itens de evidência no resultado.
    timeout_ms: float         # Default: 9000. Timeout da consulta em milissegundos.
    since: datetime | None    # Opcional. Reconsulta incremental a partir desta data/hora.
    include_evidence: bool    # Default: True. Inclui registros de evidência no resultado.
```

### FastLogQueryResult
```python
@dataclass
class FastLogQueryResult:
    success: bool             # True = consulta executou sem erro interno
    question_type: str        # Tipo de pergunta respondida
    answer: bool | int | str | list[dict[str, object]] | None  # Resposta principal
    code: str                 # Código estável programático (use QueryCode.*)
    summary: str              # Texto legível da resposta
    correlation_id: str       # correlation_id consultado
    evidence: list[dict]      # Até max_evidence registros de evidência
    metadata: dict            # records_scanned, files_scanned, matches, duration_ms,
                              # truncated, since_timestamp, answer_timestamp,
                              # last_record_timestamp
    error: dict | None        # code, message, details quando success=False
```

Método auxiliar:
```python
result.to_dict()  # Retorna dict completamente JSON-serializável
```

### IndividualAnalysisRequest
```python
@dataclass(frozen=True)
class IndividualAnalysisRequest:
    correlation_id: str     # Obrigatório
    logs_dir: Path          # Obrigatório
    include_rotated: bool   # Default: True — inclui arquivos rotacionados
    max_records: int        # Default: 50.000
```

### AggregatedAnalysisRequest
```python
@dataclass(frozen=True)
class AggregatedAnalysisRequest:
    logs_dir: Path              # Obrigatório
    from_dt: datetime | None    # Início da janela temporal (None = sem limite)
    to_dt: datetime | None      # Fim da janela temporal (None = sem limite)
    max_files: int              # Default: 100
    max_records_per_file: int   # Default: 10.000
```

### AnalysisResult
```python
@dataclass(frozen=True)
class AnalysisResult:
    success: bool
    correlation_id: str | None         # None no modo aggregated
    mode: str                          # "individual" ou "aggregated"
    analyses: dict[str, Any]           # Indexado por analysis_id de cada plugin
    files_analyzed: list[str]
    records_processed: int
    duration_ms: float
    truncated: bool                    # True quando max_records foi atingido
    error: AnalysisError | None
    schema_contract_findings: list[dict[str, Any]]   # violações de schema toleradas (strict=False)
    problems: list[dict[str, Any]]     # problemas de integridade (a parte que acusa problemas)
```

O campo `problems` é a **parte que acusa problemas**: cada item é um dict com
`problem_type`, `severity` e `message`. Hoje cobre `foreign_correlation_id` —
quando o log do alvo contém registros de um `correlation_id` diferente
(contaminação por append concorrente, vazamento entre execuções ou reúso do id),
o problema traz `expected_correlation_id`, `distinct_foreign_ids`,
`affected_records` e amostras em `foreign_correlation_ids`. Lista vazia = log íntegro.

Convertendo para JSON:
```python
serialized = LogAnalyzerService.serialize(result)   # → SerializedAnalysisResult
d = serialized.to_dict()                            # → dict JSON-safe
```

---

## 8. Camada IO: localização e carregamento de arquivos

### LogFileLocator (`io/locator.py`)

Resolve quais arquivos pertencem a um `correlation_id` seguindo 3 estratégias em cascata:

1. **Manifest persistente** (`_meta/correlation_manifest.jsonl`) — leitura reversa eficiente
2. **CanonicalLogReader.resolve_correlation_log()** — busca por padrão de nome e sidecar `.meta.json`
3. **CanonicalLogReader.resolve_correlated_log_family_files()** — expande família (API + worker + scheduler)

```python
locator = LogFileLocator()
primary_file, family_files = locator.locate(
    correlation_id="abc123",
    logs_dir=Path("/app/logs"),
    include_rotated=True,        # inclui .1, .2, ... da família
)
```

**Exceções levantadas:**
- `CorrelationIdMissingError` — `correlation_id` vazio ou ausente
- `LogsDirNotFoundError` — `logs_dir` não existe
- `LogFileNotFoundError` — nenhum arquivo encontrado para o `correlation_id`

### LogRecordLoader (`io/loader.py`)

Itera registros JSONL linha a linha sem carregar o arquivo inteiro em memória:

```python
# Modo streaming (recomendado para consulta rápida)
for record in LogRecordLoader.iter_records(family_files, max_records=5000):
    process(record)

# Modo carregamento total (análise profunda)
records: list[dict] = LogRecordLoader.load(family_files, max_records=50_000)
```

**Comportamento:**
- Linhas não-JSON são silenciosamente ignoradas (contadas em `skip_count`)
- Arquivos inacessíveis (OSError) são pulados silenciosamente e a iteração continua
- Quando `max_records` é atingido, `LogRecordLimitExceededError` é levantado

---

## 9. Engines e plugins

### Hierarquia de ABCs (`analysis/engine.py`)

```
AbstractLogEngine
  └── AbstractAnalysisEngine          ← para PandasAnalysisEngine
        Properties: engine_name, dialect, record_count
        Methods: load(records), execute_analysis(plugin)

  └── AbstractOperationalQueryEngine  ← para FastQueryEngine
        Methods: execute_query(request, record_iter)

AnalysisDialect                       ← interface agnóstica ao framework de dados
  Methods: get_state(), count_total(), filter_by_level(), filter_by_field_value(),
           count_by_field(), top_values(), numeric_stats()

AnalysisPlugin                        ← interface de plugin analítico
  Property: analysis_id
  Method: run(dialect) → dict[str, Any]
```

### PandasAnalysisEngine (`analysis/pandas_engine.py`)

Implementação concreta de `AbstractAnalysisEngine` usando Pandas como engine de dados.
O `PandasAnalysisDialect` encapsula todas as operações sobre o DataFrame — plugins nunca
importam Pandas diretamente.

```python
engine = PandasAnalysisEngine()
engine.load(records)                          # carrega list[dict] → DataFrame interno
result = engine.execute_analysis(plugin)      # executa um plugin via dialeto
```

**Isolamento proposital:** `import pandas` existe apenas dentro de `PandasAnalysisDialect.numeric_stats()`
(importação local) e no método `load()`. Isso permite trocar por Polars sem tocar nos plugins.

### FastQueryEngine (`analysis/query_engine.py`)

Engine de consulta rápida que despacha por `question_type` para o handler registrado:

| Handler                    | question_types atendidos                                                                                                     |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `ErrorQueryHandler`        | has_error, has_exception, last_error, last_exception, error_count, has_critical_error                                        |
| `FlowStatusQueryHandler`   | flow_started, flow_finished, flow_failed, flow_succeeded, flow_status, last_event                                            |
| `StuckQueryHandler`        | is_stuck, has_start_without_end                                                                                              |
| `TimeoutRetryQueryHandler` | has_timeout, has_retry                                                                                                       |
| `ComponentQueryHandler`    | component_failed                                                                                                             |
| `ParallelismQueryHandler`  | had_real_parallelism, max_real_concurrency, parallelism_evidence, had_fanout_without_parallelism, parallelism_blocker_reason |
| `IngestionPdfProgressQueryHandler` | ingestion_pdf_progress                                                                                              |
| `IngestionPerformanceQueryHandler` (`analysis/insights/handlers_ingestion.py`) | ingestion_performance                                                          |
| `RagDiagnosisQueryHandler` (`analysis/insights/handlers_rag_diagnosis.py`) | rag_diagnosis                                                                      |

Cada handler recebe o `Iterator[dict]` do loader — controla a iteração e pode parar
antecipadamente a qualquer momento com `break`.

Filtros transversais aplicados por todos os handlers:
- `_matches_component_filter(record, request.component)` — filtra pelo campo `component`
- `_matches_phase_filter(record, request.phase)` — filtra pelo campo `stage` ou `phase`

### Plugins analíticos padrão (`analysis/plugins.py`)

| Plugin | `analysis_id` | Métricas produzidas |
| --- | --- | --- |
| `ErrorWarningPlugin` | `error_warning_summary` | erros, warnings explicáveis, origem da mensagem e lacunas de schema |
| `PerformancePlugin` | `performance_summary` | estatísticas dos `duration_ms` publicados pelos produtores |
| `SlowestSegmentsPlugin` | `slowest_segments` | gaps entre eventos consecutivos; não substitui duração direta |
| `StageTimingPlugin` | `stage_timing_summary` | janela HTTP, duração total, sequência de stages e lanes |
| `EventFlowPlugin` | `event_flow_summary` | eventos distintos e distribuição |
| `ComponentActivityPlugin` | `component_activity_summary` | componentes ativos e contagens |
| `VectorDocumentIntegrityPlugin` | `ingestion_vector_integrity_summary` | recibos de integridade vetorial por documento |

```python
# Acessando métricas de um plugin no AnalysisResult:
error_data = result.analyses["error_warning_summary"]
perf_data = result.analyses["performance_summary"]
```

### 9.1. Janela request-scoped e duração autoritativa

`RequestTimeline` é a fonte única usada pelo `StageTimingPlugin` e pelo
`rag_diagnosis` para separar a requisição do restante da família de logs.

O recorte segue estas regras:

1. aceita como início `api.http.request.entered` ou `rag.http.operation.received`;
2. usa o primeiro terminal HTTP compatível depois do início;
3. rejeita registros com `correlation_id` externo divergente e o identificador
   aninhado confirmado `token_billing_event.session_id`;
4. prefere o `duration_ms` do terminal como total;
5. só calcula o total por diferença de timestamps quando início e terminal existem;
6. sem limites completos, devolve `total_duration_ms=null` e uma entrada em
   `observability_gaps`.

`stage_sequence` e `per_stage` descrevem deltas entre stages **dentro da lane HTTP
recortada**. Esses deltas ajudam a localizar intervalos, mas não substituem
`duration_ms` medido no produtor para atribuir custo a uma operação específica.

### 9.2. Lanes: request HTTP e worker de telemetria

O analyzer mantém duas visões no mesmo resultado:

- `http_request`: caminho crítico da resposta;
- `telemetry_worker`: persistência e termos de consulta executados de forma
  assíncrona depois do enqueue.

Os eventos do worker continuam em `async_lanes`, com sequência, stage e soma apenas
dos `duration_ms` publicados nos terminais `interaction.telemetry.task.completed`
ou `interaction.telemetry.task.failed`. Eles não aumentam `total_duration_ms` do
HTTP.

A classificação exige o contrato explícito:

- enqueue: `lane=http_request`, `task_id` e `span_id`;
- worker: `lane=telemetry_worker`, o mesmo `task_id`, `span_id` próprio e
  `parent_span_id` igual ao `span_id` do enqueue.

O resultado materializa essa relação em `causal_links`. Log histórico sem esses
campos não recebe fallback pelo nome `interaction.telemetry.*`; ele fica fora da
lane assíncrona e gera `observability_gaps`.

### 9.3. Cache semântico: documentos, não resposta pronta

No RAG clássico, `rag.retrieval.semantic_cache.hit` representa reuso de
documentos. O terminal de retrieval publica:

```json
{
  "cache_hit": true,
  "cache_payload_kind": "documents",
  "generation_required": true
}
```

Portanto, uma chamada posterior à LLM é comportamento esperado. O hit fecha a
tentativa de retrieval, mas não entrega a resposta final. O analyzer também
reconhece logs históricos em que o hit de documentos existe sem o antigo marco
`retrieval.completed`, sem confundir esse caso com cache de resposta.

### 9.4. Warnings explicáveis e lacunas de schema

Para cada registro `WARNING`, `ErrorWarningPlugin` escolhe o primeiro texto
significativo nesta ordem:

1. `error_message`;
2. `reason`;
3. `event_name`;
4. `message`.

`top_warning_messages` devolve `display_message`, `source_field`, `count`,
`event_name`, `component`, `stage` e `timestamp`, sem copiar payload arbitrário
ou sensível. Ocorrências equivalentes são agrupadas.

Se nenhum dos quatro campos textuais existir, o warning continua contado e pode
aparecer nas até cinco amostras de `warning_schema_gaps`, com os campos ausentes
e a orientação de corrigir o emissor pelo builder canônico. O analyzer não
inventa uma explicação.

### 9.5. O que o diagnóstico prova e o que não prova

Logs podem provar o fluxo executado, status, fontes registradas, contagens,
permissões, cache, durações e lacunas observáveis. Eles não provam sozinhos:

- factualidade completa da resposta;
- frescor ou versão universal do corpus;
- correção semântica de todo trecho gerado pela LLM;
- ausência de contenção indireta de CPU, Redis ou rede sem métrica própria.

A bateria DNIT validou resposta não vazia, fontes, documentos permitidos,
terminais, cache e isolamento entre correlações. Uma avaliação factual completa
exige um oracle documental/de negócio separado.

---

## 10. AnalysisRegistry (`analysis/registry.py`)

Registro imutável de plugins. Instancia cada plugin por chamada para garantir isolamento
entre execuções paralelas.

```python
registry = AnalysisRegistry.default()      # carrega DEFAULT_PLUGINS (os 7 padrão)
plugins = registry.plugins()               # list[AnalysisPlugin] — novas instâncias
ids = registry.analysis_ids()             # list[str] — lista os analysis_id

# Registro customizado (ex.: pipeline de testes com apenas 1 plugin)
from src.log_analyzer.analysis.plugins import ErrorWarningPlugin
registry = AnalysisRegistry((ErrorWarningPlugin,))
```

---

## 11. Hierarquia de exceções (`errors.py`)

```
Exception
  └── LogAnalyzerError                          ← raiz de todas as exceções do módulo
        ├── CorrelationIdMissingError            ← correlation_id vazio ou ausente
        ├── LogFileNotFoundError                 ← arquivo não encontrado para o correlation_id
        │     Atributos: correlation_id, logs_dir
        ├── LogRecordLimitExceededError          ← max_records atingido antes do fim
        │     Atributos: limit, files_read
        ├── AnalysisEngineError                  ← falha interna da engine analítica
        └── LogsDirNotFoundError                 ← diretório de logs não existe
              Atributos: logs_dir
```

**Regra:** `LogQueryService.query()` e `LogAnalyzerService.analyze_*()` nunca propagam
exceções para o chamador — capturam e retornam como resultado estruturado com `success=False`.

---

## 12. CLI completa (`__main__.py`)

### Subcomandos disponíveis

```
python -m src.log_analyzer [SUBCOMANDO] [OPÇÕES]

Subcomandos:
  analyze      análise profunda (Pandas, 7 plugins)
  stats        resumo estatístico de múltiplos logs (análise agregada)
  query        consulta rápida por question_type
  filter       filtro objetivo de registros reais
  list-types   lista o catálogo de tipos de pergunta
```

### Chamada sem argumentos

```bash
python -m src.log_analyzer
```

Quando a CLI é chamada sem parâmetros, ela devolve **JSON machine-readable**
com o contrato raiz da ferramenta inteira. Esse é o entrypoint preferencial
para agentes de IA e automações que ainda não sabem qual subcomando usar.

Resumo do payload raiz:

- `default_invocation`: explica o comportamento default sem argumentos
- `recommended_machine_entrypoints`: atalhos recomendados para descoberta por IA
- `recommended_human_entrypoints`: atalhos recomendados para humanos
- `execution_commands`: comandos que executam trabalho real
- `help_commands`: comandos informacionais
- `commands`: resumo leve de cada subcomando com `prog`, `description`, argumentos principais e `discovery_hint`

Para ajuda textual de terminal humano, o caminho continua sendo:

```bash
python -m src.log_analyzer --help
```

### query — todas as opções

```
--question-type {has_error,has_exception,...}  OBRIGATÓRIO. Tipo da pergunta
--correlation-id CORRELATION_ID               OBRIGATÓRIO. ID da execução a consultar
--logs-dir LOGS_DIR                           Dir de logs. Default: ./logs
--component COMPONENT                         Filtra por campo "component"
--phase PHASE                                 Filtra por campo "stage"
--since DATETIME                              Filtra registros a partir desta data/hora
--max-records MAX_RECORDS                     Limite opcional de registros. Omitido = lê o log completo
--max-evidence MAX_EVIDENCE                   Máx. itens de evidência. Default: 3
--timeout-ms TIMEOUT_MS                       Timeout em ms. Default: 9000
--no-evidence                                 Remove evidências do resultado
--list-types                                  Imprime catálogo e sai
--schema                                      Imprime schema JSON para IA e sai
```

Exemplo de consulta incremental com data/hora:

```bash
python -m src.log_analyzer query \
        --question-type last_event \
        --correlation-id abc123 \
        --since 2026-06-01T10:00:03+00:00
```

Exemplo de retorno temporal útil:

```json
{
    "answer": "rag.ask.pipeline.completed",
    "metadata": {
        "since_timestamp": "2026-06-01T10:00:03+00:00",
        "answer_timestamp": "2026-06-01T10:00:05+00:00",
        "last_record_timestamp": "2026-06-01T10:00:05+00:00"
    }
}
```

### --schema: saída machine-readable

```bash
python -m src.log_analyzer query --schema
```

Retorna JSON estruturado com:
- `command`, `prog`, `description`
- `arguments`: contrato detalhado de cada argumento, incluindo `required`, `default`, `type` e `choices`
- `output`: formato de saída e exit codes
- `record_contract`: campos canônicos consumidos, aliases proibidos e política fail-close
- `known_schema`: catálogo conhecido de campos e eventos do analyzer
- `question_type_catalog`: agrupamento semântico dos valores válidos de `--question-type`

Ideal para agentes de IA que precisam do catálogo sem parsear texto de `--help`.

### analyze — todas as opções

```
--correlation-id CORRELATION_ID   OBRIGATÓRIO
--logs-dir LOGS_DIR               Default: ./logs
--output {json,text}              Default: json
--no-rotated                      Exclui arquivos rotacionados (padrão inclui)
--max-records MAX_RECORDS         Default: 50000
--strict                          Aborta na primeira violação de contrato de schema
                                  (event_name fora do catálogo ou alias proibido).
                                  Padrão: tolerante — reporta em schema_contract_findings.
```

### stats — todas as opções

```
--logs-dir LOGS_DIR   Default: ./logs
--output {json,text}  Default: json
--max-files MAX_FILES Default: 100
--strict              Aborta na primeira violação de contrato de schema
```

---

## 13. Observabilidade e diagnóstico

### Campos úteis em `FastLogQueryResult.metadata`

```json
{
  "records_scanned": 1247,
  "files_scanned": 1,
    "since_timestamp": "2026-06-01T10:00:03+00:00",
    "answer_timestamp": "2026-06-01T10:00:05+00:00",
    "last_record_timestamp": "2026-06-01T10:00:05+00:00",
  "matches": 3,
  "duration_ms": 45.2,
  "truncated": false
}
```

Se `truncated=true`, o `max_records` foi atingido antes do fim do log. Aumente
`--max-records` para varredura completa.

Se `since_timestamp` estiver preenchido, a resposta foi calculada apenas com a
janela temporal incremental pedida.

Se `answer_timestamp` estiver preenchido, ele marca exatamente a data/hora do
registro que sustentou a resposta principal.

Se `last_record_timestamp` estiver preenchido, ele marca o último registro lido
na consulta atual e ajuda a reexecutar a próxima pergunta sem cache interno.

### Como diagnosticar "log não encontrado"

1. partir do `correlation_id` exato devolvido pelo boundary;
2. quando a origem for remota, chamar
   `BaseLogProvider.prepare_logs_for_correlation(correlation_id)`;
3. passar a origem materializada para `LogFileLocator.locate`;
4. conferir o resultado estruturado `LOG_NOT_FOUND` e o `logs_dir` efetivo;
5. se o provider materializou a família mas o locator não a resolveu, inspecionar
   `CanonicalLogReader` e o sidecar daquela correlação.

Não liste nem varra `/logs` para procurar por tentativa e erro.

### Como diagnosticar resultado com `success=False`

```python
if not result.success:
    print(result.error)          # dict | None com "code", "message", "details"
    # ou
    print(result.code)           # ex.: QueryCode.QUERY_ERROR, LOG_NOT_FOUND, MISSING_CORRELATION_ID
```

Para `AnalysisResult`:
```python
if not result.success:
    print(result.error.error_type)   # Nome da exceção
    print(result.error.message)      # Mensagem legível
    print(result.error.correlation_id_missing)  # True se foi CorrelationIdMissingError
```

### Identificando amostras parciais

```python
# Consulta rápida:
if result.metadata.get("truncated"):
    print("AVISO: varredura parcial — aumente max_records")

# Análise profunda:
if result.truncated:
    print(f"AVISO: apenas {result.records_processed} de N registros processados")
```

---

## 14. Testes

A estrutura de contratos pure-Python facilita testes sem mocks pesados:

```python
from pathlib import Path
from src.log_analyzer import LogQueryService, FastLogQueryRequest, QuestionType

def test_has_error_encontra_erro(tmp_path):
    # Criar log JSONL mínimo de teste
    log_file = tmp_path / "abc123.json"
    log_file.write_text('{"level": "ERROR", "message": "falha", "event_name": "error_event"}\n')

    service = LogQueryService()
    result = service.query(
        FastLogQueryRequest(
            question_type=QuestionType.HAS_ERROR,
            correlation_id="abc123",
            logs_dir=tmp_path,
        )
    )

    assert result.success is True
    assert result.answer is True
    assert result.code == "FOUND"
```

Para testar análise profunda com injeção:
```python
from src.log_analyzer.io.locator import LogFileLocator
from src.log_analyzer.analysis.pandas_engine import PandasAnalysisEngine
from src.log_analyzer.analysis.registry import AnalysisRegistry
from src.log_analyzer.analysis.plugins import ErrorWarningPlugin

service = LogAnalyzerService(
    locator=LogFileLocator(),
    engine=PandasAnalysisEngine(),
    registry=AnalysisRegistry((ErrorWarningPlugin,)),  # apenas 1 plugin para velocidade
)
```

---

## 15. Referência rápida de `AnalysisResult.analyses`

Após `analyze_individual`, o dict `result.analyses` terá:

```python
# error_warning_summary
{
    "analysis_id": "error_warning_summary",
    "total_records": 1247,
    "error_count": 3,
    "warning_count": 12,
    "error_rate_pct": 0.24,
    "top_error_messages": [{"value": "Connection refused", "count": 2}, ...],
    "top_warning_messages": [
        {
            "display_message": "qdrant_timeout",
            "source_field": "event_name",
            "count": 2,
            "event_name": "qdrant_timeout",
            "component": "semantic_cache",
            "stage": "semantic_cache:qdrant:search_points",
            "timestamp": "..."
        }
    ],
    "warning_schema_gaps": []
}

# performance_summary
{
    "analysis_id": "performance_summary",
    "duration_stats": {"min": 5.0, "max": 1230.0, "mean": 145.3, "p95": 980.0, "p99": 1180.0, "count": 89},
    "top_slow_event_names": [{"value": "pdf_extraction", "count": 3}, ...]
}

# stage_timing_summary — tempo entre as etapas (stage) do pipeline já calculado
{
    "analysis_id": "stage_timing_summary",
    "analyzed_stage_events": 18,
    "distinct_stages": 6,
    "total_window_seconds": 10.0,
    "total_duration_source": "rag.http.request.completed.duration_ms",
    "request_window": {
        "complete": true,
        "start_event": "rag.http.operation.received",
        "terminal_event": "rag.http.request.completed"
    },
    "stage_sequence": [
        {"stage": "retrieval", "from_timestamp": "...", "to_timestamp": "...", "duration_seconds": 7.0}, ...
    ],
    "per_stage": [{"stage": "retrieval", "total_seconds": 7.0, "occurrences": 1, "avg_seconds": 7.0}, ...],
    "slowest_stage": {"stage": "retrieval", "total_seconds": 7.0},
    "lanes": [
        {"lane": "http_request", "kind": "critical_path", "total_duration_ms": 10000.0},
        {"lane": "telemetry_worker", "kind": "asynchronous", "measured_duration_ms": 220.0}
    ],
    "async_lanes": [...],
    "observability_gaps": []
}

# event_flow_summary
{
    "analysis_id": "event_flow_summary",
    "unique_events": 23,
    "event_distribution": [{"event_name": "ingestion_started", "count": 1}, ...],
    "top_events": [...]  # top 5
}

# component_activity_summary
{
    "analysis_id": "component_activity_summary",
    "active_components": 7,
    "component_activity": [{"component": "PDFExtractor", "count": 45}, ...]
}
```

---

## 16. Troubleshooting

### Sintoma: `LogFileNotFoundError`
- **Causa provável:** `correlation_id` correto mas `logs_dir` apontando para diretório errado, ou arquivo ainda não foi gerado pelo processo
- **Como confirmar:** preparar a correlação pelo provider oficial e consultar pelo
  `LogFileLocator`, sem listar o diretório
- **Ação:** verificar o `PreparedLogSource`, o `logs_dir` materializado e o sidecar
  da correlação exata

### Sintoma: `result.truncated=True` com `answer=False` suspeito
- **Causa:** `max_records` muito baixo, o erro pode estar além dos primeiros N registros
- **Ação:** aumentar `--max-records` ou `max_records` no request; verificar `metadata.records_scanned`

### Sintoma: `result.success=False` com `code=QUERY_ERROR`
- **Causa:** exceção não esperada dentro da engine ou handler
- **Como investigar:** `result.error` (dict com `code`/`message`/`details` no FastLogQueryResult) ou `result.error.message + result.error.error_type` (AnalysisResult)

### Sintoma: `answer=False` para `flow_status` mesmo com log existente
- **Causa:** os campos `event_name` e `status` do log usam vocabulário que não está nos conjuntos `_START_EVENTS`, `_END_EVENTS`, `_SUCCESS_STATUSES` do `FlowStatusQueryHandler`
- **Como confirmar:** usar `analyze_individual` + `event_flow_summary` para ver os `event_name` reais no log
- **Ação:** comparar os event_names do log com os conjuntos constantes no topo de `query_engine.py`

### Sintoma: consulta lenta em log grande
- **Causa:** perguntas que não têm parada antecipada (ex.: `error_count`, `flow_status`) precisam varrer todo o log
- **Ação:** aumentar `--timeout-ms`; ou usar `--max-records` menor com ciência de que o resultado será parcial
