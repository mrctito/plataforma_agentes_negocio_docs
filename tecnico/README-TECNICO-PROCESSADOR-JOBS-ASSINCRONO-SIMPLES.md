# Manual técnico: Processador de Jobs Assíncrono Simples (spec-101)

## Atenção antes de ler

Este documento cobre um mecanismo **distinto e separado** do Job Core genérico (RabbitMQ + Dramatiq + Worker + fan-out). Os dois coexistem no produto. Para o Job Core genérico, leia:

- [README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md](README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md)

Para o pipeline de ingestão PDF (extração, OCR, chunking, embeddings, indexação), leia:

- [README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md)

Este documento cobre o mecanismo de fila, wakeups RabbitMQ, dispatcher, Killer, dashboard mínimo e execução do job. Não duplica o pipeline de PDF em si.

**Nota sobre o Job Core genérico:** o mecanismo genérico (Dramatiq + Worker dedicado) está marcado como legado em depreciação futura. O processador simples (spec-101) é a direção de evolução para ingestão assíncrona de PDF.

---

## 1. Arquitetura em 9 etapas

### Etapa 1 — Submit via endpoint

O endpoint de ingestão chama `RedisRuntimeIngestionStreamPublisher.publish` em `src/api/services/ingestion_job_executor.py`.

O `publish()` é mínimo e direto: não usa maquinário legado do Job Core. O que acontece dentro dele:

1. Gera `run_id` (UUID) e `queued_at`.
2. Resolve o `dispatcher_wakeup_publisher` (instância de `RabbitMqJobQueue` ou equivalente).
3. Instancia `SimpleAsyncPdfJobStore.from_runtime_environment` (lê `INGESTION_TELEMETRY_DSN`).
4. Loga `job_submit_requested`.
5. Decripta o payload para extrair o snapshot de `vectorstore` (id e tipo), usado para preencher a coluna `vectorstore_id`.
6. Emite feedback `pending` no callback da UI (sinal imediato de "enfileirado").
7. Chama `job_store.create_job(...)` com `job_type="ingestion_pdf"` e `status="pending"` na tabela `job_core.async_jobs`.
8. Loga `job_row_created`.
9. Chama `dispatcher_wakeup_publisher.publish_dispatcher_wakeup(correlation_id, job_id)`. Em falha de wakeup, marca o job `error` antes de re-raise.
10. Loga `job_dispatch_signal_sent` e `job_submit_finished`.
11. Retorna `RuntimeIngestionEnqueueReservation` ao chamador imediatamente.

O payload JSON gravado em `job_params_json` inclui: `encrypted_data`, `output_format`, `document_parallelism`, `run_id`, `queued_at`. O YAML não é serializado aqui — é decriptado pelo processo do job a partir de `encrypted_data`.

O `schedule_runtime_ingestion` (camada acima, no `IngestionJobExecutor`) ainda chama `_register_runtime_progress_yaml_snapshot` antes de chamar `publish()`. Essa função registra um snapshot mínimo para overlays efêmeros da UI via `attach_ingestion_job_runtime_monitor`. Isso é infraestrutura de overlay da UI, não a lógica de fila: o `publish()` em si não depende dela.

### Etapa 2 — Sinal RabbitMQ de alarme

`publish_dispatcher_wakeup` publica uma mensagem JSON mínima na fila `prometeu.jobs.ingestion_dispatcher_wakeup` (namespaced). A mensagem carrega apenas `message_id`, `event_name`, `correlation_id`, `job_id`, `async_job_kind="ingestion"` e `async_job_queue_role="ingestion_dispatcher_wakeup"`. Não carrega o payload do job.

A fila de alarme é tratada como efêmera: sem dead-letter exchange nem `x-message-ttl`.

### Etapa 3 — Dispatcher acorda no worker

O dispatcher é implementado em `dispatch_pending_simple_async_pdf_jobs` (`simple_async_pdf_job_runtime.py`). Ele é chamado pelo runtime `SimpleAsyncPdfDispatcherRabbitMqRuntime`, que sobe dentro do processo worker oficial via `build_async_job_worker_runtime(...)` e `WorkerProcessRuntime.start()`.

Ao acordar:

1. Loga `dispatcher_wakeup_received`.
2. Chama `_reap_finished_job_processes()` — colhe (join) os processos de job já terminados chamando `multiprocessing.active_children()`. Isso evita processos zumbi acumulando entre rodadas.
3. Tenta adquirir `pg_try_advisory_lock(hashtext('simple_async_pdf_dispatcher_v1'))` via `store.dispatcher_lock()`. Se não adquire, loga `dispatcher_lock_not_acquired` e retorna. Garante que apenas um dispatcher processe a fila de pendentes por vez.
4. Reconcilia jobs presos em `processing` sem atividade além da janela configurada.
5. Conta jobs `processing` → `processing_count`.
6. Calcula `available_slots = config.max_jobs_parallel - processing_count`.
7. Lista jobs `pending` limitados a `available_slots` (ORDER BY `created_at ASC`).
8. Loga `dispatcher_pending_jobs_found`.
9. Se sem slots: loga `dispatcher_no_slots_available`.
10. Se sem pendentes: loga `dispatcher_no_jobs_started`.

### Etapa 4 — Processo do job

Para cada job pendente que caiba nos slots disponíveis, o dispatcher cria um processo separado:

```python
process = multiprocessing.Process(
    target=execute_simple_async_pdf_job_process,
    kwargs={...},
    daemon=False,   # não-daemon: precisa criar subprocessos runners
)
process.start()
```

O processo é criado com `daemon=False` porque, no Python, um processo daemon não pode criar subprocessos filhos. Os runners são subprocessos criados dentro do processo do job — portanto o processo do job obrigatoriamente não pode ser daemon. A colheita desses processos não-daemon ocorre em `_reap_finished_job_processes()` chamada a cada wakeup do dispatcher.

Os runners, que não criam subprocessos próprios, são criados com `daemon=True` (ver etapa 7).

O dispatcher loga `dispatcher_process_spawned` se o processo iniciou, ou `dispatcher_process_spawn_failed` em caso de exceção. Loga `dispatcher_round_completed` ao final.

### Etapa 5 — Início do processo do job

`execute_simple_async_pdf_job_process` (função de nível de módulo em `simple_async_pdf_job_runtime.py`):

1. Cria logger e store com `correlation_id` do job.
2. Chama `store.mark_processing(job_id)` — UPDATE WHERE `status='pending'` RETURNING. Se retornar `None`, loga `job_process_skipped_non_pending` e retorna (idempotência).
3. Persiste `runner_pgid = os.getpid()` para o Killer conseguir alcançar a árvore inteira depois.
4. Loga `job_process_started` e `job_status_changed_to_processing`.
5. Chama `_build_job_work_items(...)` — monta a lista de `SimpleAsyncPdfWorkItem`. Se vazia, loga `job_pdf_list_empty` e levanta `RuntimeError`.
6. Calcula `runner_count = clamp(record.parallelism, 1, max_runners_per_job)`.
7. Loga `job_runners_create_started`.
8. Cria `multiprocessing.Queue` para tasks e results.
9. Cria N processos `_runner_loop` com `daemon=True` (runners não criam filhos).
10. Loga `job_runner_started` para cada runner.
11. Loga `job_runners_create_finished`.
12. Enfileira os work items e N sentinels `None` na task queue.
13. Loga `job_no_more_pdfs_to_dispatch`.
14. Chama `_collect_runner_results` e depois `_finalize_job`.

### Etapa 6 — Lista de PDFs

`_build_job_work_items` resolve o YAML a partir de `job_params_json.encrypted_data` (via `resolve_yaml_configuration`) e monta dois tipos de PDF:

- **Locais:** extrai `pdf_file_paths` do request de ingestão. Para cada caminho, cria uma cópia profunda do YAML, desativa todas as fontes (`local_files`, `google_drive`, `remote_sources.web_scraping`, `remote_sources.s3`) e habilita apenas `local_files` com `discovery_patterns.include = [pdf_file_path]`.
- **Remotos:** chama `asyncio.run(service._build_document_fanout_plan(request))` para resolver o plano de fan-out. Cada item do plano vira um `SimpleAsyncPdfWorkItem`.

Em ambos os casos, `_enforce_upsert_batch_size` força `vector_store.upsert_strategy.batch_size = 250` independente do YAML.

Eventos: `job_yaml_config_resolution_started`, `job_yaml_config_resolution_completed` (ou `job_yaml_config_reused`), `job_pdf_list_started`, `job_pdf_list_finished` (com `pdf_count`, `local_pdf_count`, `remote_pdf_count`).

### Etapa 7 — Runners (pool de subprocessos)

Cada runner é criado como `multiprocessing.Process(target=_runner_loop, daemon=True)`.

`_runner_loop`:

1. Cria seu próprio logger e `SimpleAsyncPdfJobStore` com novo `ConnectionPool` (fork-safe: criado após o fork, não herdado do processo pai).
2. Loop: `task_queue.get()` — se `None` (sentinel), loga `job_runner_stopped` e retorna.
3. Para cada `SimpleAsyncPdfWorkItem`: loga `job_runner_received_pdf`, chama `_execute_simple_async_pdf_work_item`.
4. Coloca resultado em `result_queue`.

### Etapa 8 — Pipeline de ingestão por PDF

`_execute_simple_async_pdf_work_item`:

1. Loga `job_pdf_dispatched_to_runner`, `pdf_started`, `pdf_text_extraction_started`.
2. Cria `_SimpleAsyncPdfPipelineProgressCallback` — bridge mínimo que traduz cada chamada `callback.update()` do pipeline oficial em `pdf_pipeline_progress` no log canônico, com `pipeline_stage` normalizado por `_normalize_pipeline_progress_stage`.
3. Cria `IngestionService(yaml_config=work_item.yaml_config_data, requested_document_parallelism=1)`.
4. Chama `asyncio.run(service.execute_single_document(output_format, progress_callback, parent_task_id))`.
5. Em sucesso: `_raise_if_pdf_execution_result_failed(result)`, extrai métricas reais de `chunks_stored`/`chunks_created`, loga `pdf_finished_success` com `chunk_count` e `embedding_count` reais.
6. Em erro: loga `pdf_pipeline_failed` (com `pipeline_stage` do momento da falha e campos de progresso) e `pdf_finished_error`. Retorna dict de erro sem re-raise (isolamento por PDF).

Os eventos `pdf_pipeline_progress` são emitidos pelo callback real ligado ao progresso verdadeiro do `IngestionService.execute_single_document` — não são sintéticos. O campo `pipeline_stage` reflete o estágio real do pipeline no momento de cada atualização.

### Etapa 9 — Coleta de resultados e fechamento

`_collect_runner_results`:

- Aguarda `len(results) == expected_results` com `result_queue.get(timeout=1.0)`.
- Loga `job_runner_became_free` a cada resultado recebido.
- Se processo runner morrer antes de enviar resultado: loga `job_runner_missing_results_detected`, sintetiza resultado de erro com `error_type=RunnerProcessExitedUnexpectedly` e loga `pdf_finished_error` para cada PDF perdido.
- Chama `process.join(timeout=5.0)` para todos os processos.
- Loga `job_runner_result_collection_completed`.

`_finalize_job`:

- Loga `job_summary_computed` com `total_pdfs`, `success_count`, `error_count`.
- `final_status = "success" if error_count == 0 else "error"`.
- `final_message = f"Job concluido com {success_count} PDFs com sucesso e {error_count} PDFs com erro"`.
- Chama `store.finish_job(job_id, status, final_message, error_message)`.
- Loga `job_finished_success` ou `job_finished_error`.

---

## 2. Schema da tabela `job_core.async_jobs`

DDL/migrações do contrato atual:

- `scripts/sql/20260618_create_simple_async_pdf_jobs.sql`
- `scripts/sql/20260618_add_tenant_scope_to_simple_async_pdf_jobs.sql`
- `scripts/sql/20260621_add_cancellation_to_simple_async_pdf_jobs.sql`

| Coluna | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `job_id` | TEXT | Sim (PK) | Mesmo valor do `task_id` / `correlation_id` do submit |
| `job_type` | TEXT | Sim | `"ingestion_pdf"` |
| `job_params_json` | JSONB | Sim | Parâmetros do job: `encrypted_data`, `output_format`, `document_parallelism`, `run_id`, `queued_at` |
| `status` | TEXT | Sim | `pending`, `processing`, `success`, `error`, `cancelling`, `cancelled` |
| `created_at` | TIMESTAMPTZ | Sim | Momento da criação (default: NOW()) |
| `started_at` | TIMESTAMPTZ | Não | Quando o processo do job chamou `mark_processing` |
| `finished_at` | TIMESTAMPTZ | Não | Quando `finish_job` foi chamado |
| `last_activity_at` | TIMESTAMPTZ | Sim | Atualizado em todo log via `store.touch_activity` |
| `correlation_id` | TEXT | Sim | `correlation_id` do submit original |
| `requested_by` | TEXT | Não | Email do usuário que submeteu |
| `parallelism` | INTEGER | Sim | Número de runners paralelos (min 1) |
| `final_message` | TEXT | Não | Mensagem consolidada ao finalizar |
| `error_message` | TEXT | Não | Detalhe do erro se status=error |
| `tenant_code` | TEXT | Não | Escopo do tenant (NULL em linhas antigas) |
| `vectorstore_id` | TEXT | Não | Identificador do vector store destino |
| `runner_pgid` | INTEGER | Não | Process-group id do líder do job-runtime, usado pelo Killer |

**Constraints confirmadas no DDL:**

- `status` deve ser `pending`, `processing`, `success`, `error`, `cancelling` ou `cancelled`
- `job_params_json` deve ser objeto JSON (não array, não null)
- `parallelism >= 1`
- `started_at >= created_at` (quando não null)
- `finished_at >= started_at` (quando ambos não null)

**Índices:**

- `ix_async_jobs_status_created_at` — (status, created_at ASC) — usado na leitura do dispatcher
- `ix_async_jobs_correlation_id` — busca por correlation_id
- `ix_async_jobs_tenant_code_created_at` — (tenant_code, created_at DESC) — dashboard de tenant

---

## 3. Queue roles e runtimes oficiais

O simple job usa duas filas curtas próprias, além da tabela `job_core.async_jobs`:

- `ingestion_dispatcher_wakeup`
  - acorda o dispatcher da spec-101
  - payload mínimo: `message_id`, `event_name`, `correlation_id`, `job_id`, `async_job_kind`, `async_job_queue_role`
- `ingestion_killer_wakeup`
  - acorda o Killer da spec-101
  - payload mínimo: `message_id`, `event_name`, `correlation_id`, `job_id`, `async_job_kind`, `async_job_queue_role`

As duas filas são publicadas por `RabbitMqJobQueue` e consumidas no **worker oficial**, não na API.

O runtime do worker sobe um runtime composto com três blocos:

- runtime RabbitMQ genérico do worker;
- dispatcher da spec-101;
- Killer da spec-101.

Isso é montado em:

- `src/api/services/async_job_worker_runtime_factory.py`
- `src/api/services/worker_process_runtime.py`

---

## 4. Limites de concorrência

Dois parâmetros controlam o paralelismo, passados como `SimpleAsyncPdfDispatcherConfig`:

- `max_jobs_parallel` — número máximo de jobs com status `processing` ao mesmo tempo. O dispatcher não inicia novos jobs se `count_processing_jobs() >= max_jobs_parallel`.
- `max_runners_per_job` — número máximo de processos runners dentro de um único job. O runner count efetivo é `clamp(record.parallelism, 1, max_runners_per_job)`.

O campo `parallelism` em `job_core.async_jobs` vem do `document_parallelism` passado no submit. O dispatcher usa `max_runners_per_job` como teto absoluto.

---

## 5. Dashboard operacional da spec-101

O dashboard dos jobs spec-101 usa um reader específico:

- `src/api/services/simple_async_pdf_job_dashboard_reader.py`

Esse reader existe porque o caminho histórico do dashboard (`vector_*`) não enxerga esses jobs como fonte durável própria. O contrato atual do dashboard da spec-101 é deliberadamente mínimo:

- resumo por tenant com `task_id`, `run_id`, `tenant_code`, `vectorstore_id`, `status`, tempos e `correlation_id`;
- detalhe mínimo com:
  - `metadata.requested_by`
  - `metadata.parallelism`
  - `metadata.job_type`
  - `metadata.persisted_status`
- sem `documents_preview` real;
- sem `document_status_counts` reais;
- `allowed_actions` com:
  - `open_detail`
  - `open_central_logs`, quando houver `correlation_id`
- sem oferta de cancel pelo dashboard nesse reader; o cancelamento é um boundary separado.

Filtros operacionais atuais do store:

- ativos: `pending`, `processing`
- histórico durável: `success`, `error`

`cancelling` e `cancelled` existem no contrato do modelo/store, mas o reader atual foi mantido minimalista e conservador. A documentação precisa refletir o código atual, não um comportamento idealizado.

---

## 6. Como diagnosticar

### Via correlation_id

Todo log gerado pelo processador inclui o `correlation_id` do submit. Use o módulo oficial:

```bash
source .venv/bin/activate
python -m src.log_analyzer query --correlation-id <correlation_id>
```

O log com o `correlation_id` do job contém todos os eventos do ciclo de vida, em ordem de emissão real:

`job_submit_requested` → `job_row_created` → `job_dispatch_signal_sent` → `job_submit_finished` → `dispatcher_wakeup_received` → `dispatcher_lock_acquired` → `dispatcher_pending_jobs_found` → `dispatcher_process_spawned` → `job_process_started` → `job_status_changed_to_processing` → `job_pdf_list_started` → `job_pdf_list_finished` → `job_runners_create_started` → `job_runner_started` (×N) → `job_runners_create_finished` → `job_no_more_pdfs_to_dispatch` → `job_runner_result_collection_started` → `job_pdf_dispatched_to_runner` (×PDF) → `pdf_started` → `pdf_text_extraction_started` → `pdf_pipeline_progress` (×progresso) → `pdf_finished_success` ou `pdf_pipeline_failed`+`pdf_finished_error` → `job_runner_became_free` → `job_runner_result_collection_completed` → `job_summary_computed` → `job_finished_success` ou `job_finished_error`.

### Via banco PostgreSQL

```sql
-- Status atual dos jobs
SELECT job_id, status, created_at, started_at, finished_at, final_message, tenant_code
FROM job_core.async_jobs
ORDER BY created_at DESC
LIMIT 20;

-- Jobs por status
SELECT status, COUNT(*) FROM job_core.async_jobs GROUP BY status;

-- Job específico por correlation_id
SELECT * FROM job_core.async_jobs WHERE correlation_id = '<id>';

-- Jobs potencialmente presos, mesmo com reconciliacao basica
SELECT job_id, status, last_activity_at, runner_pgid
FROM job_core.async_jobs
WHERE status IN ('processing', 'cancelling')
ORDER BY last_activity_at ASC;
```

### Campos de diagnóstico nos logs

Campos canônicos relevantes no payload de log (`build_canonical_log_context` com grupos `async_runtime` e `error`):

- `job_id` — identificador do job na tabela
- `runner_id` — qual runner está processando
- `pdf_name` — nome do PDF sendo processado
- `pdf_status` — `success` ou `error`
- `chunk_count`, `embedding_count` — métricas reais de resultado por PDF (de `chunks_stored`/`chunks_created`)
- `success_count`, `error_count` — consolidados ao fechar o job
- `exception_scope` — onde a exceção foi capturada (`job_process`, `runner_loop`, `pdf_work_item`, `dispatcher`)
- `pipeline_stage` — stage normalizado do pipeline no momento do erro ou progresso
- `progress_current`, `progress_total`, `progress_percentage` — progresso real do pipeline
- `decision_code` — decisão tomada em branches relevantes

### `last_activity_at` como sinal de liveness

`store.touch_activity(job_id)` é chamado após **cada** evento de log via `_log_and_touch`. Se `last_activity_at` não for atualizado por mais de N minutos enquanto `status=processing`, o job pode estar travado (ver limitações).

---

## 7. Cancelamento simples via Killer

O cancelamento atual da spec-101 não usa mais o caminho cooperativo legado do Job Core genérico.

Boundary:

- `src/api/routers/rag_runtime_cancellation_compat.py`

Runtime operacional:

- `src/api/services/simple_async_pdf_killer_rabbitmq.py`

Fluxo:

1. o endpoint resolve o `job_id` na tabela `job_core.async_jobs`;
2. se o job já for terminal, devolve conflito;
3. se ainda estiver vivo, publica wakeup em `ingestion_killer_wakeup`;
4. o Killer no worker marca o job como `cancelling`;
5. se houver `runner_pgid`, envia `SIGTERM` ao líder para drenar o lote;
6. se o grupo não encerrar na carência, escala para `SIGKILL`;
7. fecha o job como `cancelled`.

Limites reais:

- o cancelamento é por lote, não por documento individual;
- o callback do pipeline ainda não interrompe um mesmo PDF em granularidade fina;
- a premissa operacional atual é um worker por host, porque o Killer age sobre `process-group` local.

---

## 8. Estado atual e limitações técnicas

### Processamento de PDFs funciona de ponta a ponta

O mecanismo está operacional. O bloqueador histórico de `daemon=True` foi corrigido: o processo do job é criado com `daemon=False`, o que permite a criação dos subprocessos runners. A colheita de processos terminados ocorre em `_reap_finished_job_processes()`, chamada a cada wakeup do dispatcher via `multiprocessing.active_children()`.

Existe teste de regressão em `tests/integration/test_03-01-34_simple_async_pdf_job_e2e.py::test_dispatcher_cria_processo_do_job_nao_daemon_e_runners_daemon` que falha se `daemon=True` for reintroduzido no processo do job.

**Ressalva de honestidade:** a validação foi realizada pela suíte de testes focada (e2e offline, 18/18 verde). Uma ingestão real end-to-end com instâncias de Postgres e Qdrant em produção não foi documentada como executada após essa correção. O código está pronto; a validação em infra real é o próximo grau de confiança.

### Pools PostgreSQL fork-safe

Cada runner cria seu próprio `ConnectionPool` após o fork, via `_ensure_pool()` em `SimpleAsyncPdfJobStore`. Isso garante que o pool do processo pai não seja compartilhado com os subprocessos (commit `4401577ff`). Essa parte está integrada e funcional.

### Reconciliação básica de jobs travados

Se o processo do job for morto externamente (OOM, kill, crash do container), o mecanismo atual já tenta liberar o slot com `reconcile_stale_processing_jobs(...)`. Essa reconciliação:

- roda quando o dispatcher acorda;
- procura jobs `processing` sem atividade acima do limite;
- fecha essas linhas como `error`.

O que ela não faz:

- não retoma o lote;
- não recompõe progresso parcial;
- não substitui reconciliação mais sofisticada de negócio.

### Cancelamento ainda é simples, não sofisticado

O cancelamento via Killer já existe, mas continua simples:

- não há cancelamento fino dentro do processamento de um mesmo PDF;
- o callback `_SimpleAsyncPdfPipelineProgressCallback.is_cancel_requested()` permanece sem cooperação sofisticada;
- a garantia forte é parar o lote e encerrar a árvore de processos, não preservar parcial elegante por documento.

### Submit registra snapshot de overlay para a UI

`schedule_runtime_ingestion` chama `_register_runtime_progress_yaml_snapshot` antes do `publish()`, que usa `attach_ingestion_job_runtime_monitor` para overlays efêmeros da UI. Isso é infraestrutura de display da UI, não lógica de fila. O `publish()` em si não depende disso.

---

## 9. Arquivos principais

| Arquivo | Responsabilidade |
| --- | --- |
| `src/api/services/simple_async_pdf_job_runtime.py` | Dispatcher, `_reap_finished_job_processes`, processo do job (não-daemon), runners (daemon), progress callback real, coleta de resultados, finalização |
| `src/api/services/simple_async_pdf_job_store.py` | Acesso PostgreSQL à tabela `job_core.async_jobs`, advisory lock, retry, pools fork-safe |
| `src/api/services/simple_async_pdf_job_models.py` | `SimpleAsyncPdfJobRecord`, `SimpleAsyncPdfJobStatus`, `SimpleAsyncPdfJobValidationError` |
| `src/api/services/async_job_rabbitmq.py` | `publish_dispatcher_wakeup`, `publish_killer_wakeup`, filas curtas de wakeup |
| `src/api/services/simple_async_pdf_dispatcher_rabbitmq.py` | Runtime RabbitMQ do dispatcher da spec-101 |
| `src/api/services/simple_async_pdf_killer_rabbitmq.py` | Runtime RabbitMQ do Killer da spec-101 |
| `src/api/services/simple_async_pdf_job_dashboard_reader.py` | Leitura mínima do dashboard para jobs da spec-101 por tenant |
| `src/api/routers/rag_runtime_cancellation_compat.py` | Boundary HTTP de cancelamento da ingestão spec-101 |
| `src/api/services/async_job_worker_runtime_factory.py` | Wiring do runtime composto do worker com dispatcher e Killer |
| `src/api/services/ingestion_job_executor.py` | `RedisRuntimeIngestionStreamPublisher.publish` (submit mínimo) e `IngestionJobExecutor.schedule_runtime_ingestion` |
| `scripts/sql/20260618_create_simple_async_pdf_jobs.sql` | DDL inicial da tabela |
| `scripts/sql/20260618_add_tenant_scope_to_simple_async_pdf_jobs.sql` | Adição de `tenant_code` e `vectorstore_id` |
| `scripts/sql/20260621_add_cancellation_to_simple_async_pdf_jobs.sql` | Adição de `runner_pgid` e estados de cancelamento |
| `tests/integration/test_03-01-34_simple_async_pdf_job_e2e.py` | Suíte e2e, incluindo teste de regressão do daemon |

---

## 10. Referências cruzadas

Este mecanismo é distinto do Job Core genérico. Para entender o Job Core (legado em depreciação futura):

- [README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md](README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md) — arquitetura genérica, Worker dedicado, Dramatiq, fan-out
- [README-CONCEITUAL-SISTEMA-JOBS-WORKER-PARALELISMO.md](../conceitual/README-CONCEITUAL-SISTEMA-JOBS-WORKER-PARALELISMO.md) — visão executiva do Job Core

Para o pipeline de ingestão PDF:

- [README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md) — extração, OCR, chunking, embeddings, indexação

Para a visão conceitual deste processador:

- [README-CONCEITUAL-PROCESSADOR-JOBS-ASSINCRONO-SIMPLES.md](../conceitual/README-CONCEITUAL-PROCESSADOR-JOBS-ASSINCRONO-SIMPLES.md)

Para o índice geral de ingestão:

- [README-INGESTAO-INDICE.md](README-INGESTAO-INDICE.md)
