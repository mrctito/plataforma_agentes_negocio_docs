# Job Core: contrato, worker e paralelismo

Este documento descreve o Job Core que está ligado no runtime atual. O núcleo recebe um
`JobEnvelope`, resolve um `JobProcessDescriptor`, cria um processo nominal por invocação e é a
única autoridade de claim, status, heartbeat, cancelamento, árvore e terminalização. Ingestão,
ETL, scheduler, background/HIL, agent, workflow, RAG e manutenções são clientes desse mesmo
contrato; nenhum deles mantém executor ou lifecycle paralelo.

As fontes executáveis principais são:

- [`job_process.py`](../../src/core/job_core/job_process.py): contrato nominal do processo;
- [`registry.py`](../../src/core/job_core/registry.py): catálogo único de descritores;
- [`executor.py`](../../src/core/job_core/executor.py): runner e tradução única para lifecycle;
- [`store.py`](../../src/core/job_core/store.py): port, estados, árvore, liveness e store InMemory;
- [`postgres_store.py`](../../src/core/job_core/postgres_store.py): admissão e ledger duráveis;
- [`async_job_process_catalog.py`](../../src/api/services/async_job_process_catalog.py): catálogo
  único dos processos do produto;
- [`async_job_worker_runtime_factory.py`](../../src/api/services/async_job_worker_runtime_factory.py):
  composition root oficial do worker;
- [`job_core_worker_runtime.py`](../../src/api/services/job_core_worker_runtime.py): polling e
  slots locais;
- [`reconciliation.py`](../../src/core/job_core/reconciliation.py): reconciliação cancel-only.

O modelo físico está em
[`README-SCHEMA-BANCO.md`](README-SCHEMA-BANCO.md#domínio-job-core).

## Responsabilidades e limites

O Core é responsável por:

- validar envelope, descritor, input, output e plano de filhos;
- publicar e reivindicar trabalho no ledger PostgreSQL;
- controlar ownership, heartbeat, estados, cancelamento e terminalização;
- materializar árvores, limitar filhos diretos, liberar o slot do pai e agregar resultados;
- serializar admissão compartilhada entre processos;
- expor leitura tipada, paginação e gestão escopada;
- emitir eventos duráveis e logs canônicos correlacionados.

O processo de domínio recebe input tipado e um contexto mínimo. Ele não recebe `JobEnvelope`,
store, conexão, fila, worker ID, token PostgreSQL nem API de status. Também não publica/polla
filhos, não escolhe terminal e não implementa retry do job.

## Contrato público do processo

O contrato real vive em `src/core/job_core/job_process.py`:

```python
class ProcessHostPort(Protocol):
    def emit(self, fact: DomainFact) -> None: ...
    def is_cancellation_requested(self) -> bool: ...
    def raise_if_cancelled(self) -> None: ...

@dataclass(frozen=True, slots=True)
class ProcessContext:
    correlation_id: str
    host: ProcessHostPort

class JobProcess(ABC, Generic[ProcessInputT, ProcessOutputT]):
    @abstractmethod
    async def execute(
        self,
        process_input: ProcessInputT,
        context: ProcessContext,
    ) -> ProcessOutcome[ProcessOutputT]: ...
```

`JobProcessDescriptor` liga `process_key`, `route_kind`, `dispatch_mode`, factory, decoder de
entrada e encoder de saída. A factory cria uma instância nova por execução. `ProcessOutcome`
carrega o valor de domínio e, opcionalmente, um `ChildWorkPlan`; ele nunca carrega `JobStatus`.

`JobCoreExecutor` cria o `ProcessContext`, chama `raise_if_cancelled()` antes e depois do processo,
executa `await process.execute(...)` e traduz exatamente uma vez:

- retorno `ProcessOutcome` válido → sucesso ou espera pelos filhos;
- `TaskCancelledError` → `cancelled`;
- outra exceção → `failed` com erro estruturado.

Não existe `None => succeeded`, template `run()`, host guardado no construtor nem callback de
lifecycle. Os testes de contrato e host-transparência estão em
[`test_02-22-28_job_process_contract.py`](../../tests/unit/core/test_02-22-28_job_process_contract.py).

## Fatos e progresso

`DomainFact` é a saída lateral permitida. `ProgressFact` é apenas um subtipo com `stage`,
`current` e `total`, emitido pelo mesmo `ProcessHostPort.emit(...)`. O executor registra o fato
com `job_core.execution.domain_fact_emitted`, mas ele:

- não altera status;
- não persiste percentual no ledger;
- não autoriza retry, claim ou terminalização;
- não cria callback paralelo de progresso.

Um adapter de domínio pode traduzir seu reporter local para `ProgressFact`; isso não amplia o port
do Core nem transforma progresso em verdade operacional.

## Registry, catálogo e wiring oficial

`JobProcessRegistry` registra lotes de descritores atomicamente, rejeita chaves duplicadas e
resolve por `route_kind + dispatch_mode` ou por `process_key`. Ele não executa processos e não
possui fallback.

`build_async_job_process_registry(...)` registra no mesmo catálogo ingestão, ETL, scheduler,
agent/workflow/RAG assíncronos, continuação HIL, checkpoints e manutenções. Em seguida,
`build_async_job_worker_runtime(...)` monta um único `PostgresJobRunStore`, `JobCoreExecutor`,
`JobCoreCancelOnlyReconciler` e `JobCoreWorkerPollingRuntime`.

O consumer runtime aceito é `job_core_polling`. Não existem catálogo ou executor de
payload, registry/dispatcher scheduler ou worker de lifecycle background como caminho alternativo.

## Envelope, publicação e polling

`JobEnvelope` preserva os campos físicos `job_id`, `route_kind`, `dispatch_mode`, `job_type`,
`handler_key`, `correlation_id`, payload e metadata, além de árvore, due time e concorrência. O nome
histórico `handler_key` continua no formato persistido, mas não representa um contrato de handler:
o runtime resolve um `JobProcessDescriptor` pelo routing key.

O caminho durável é:

1. o boundary cria a correlação, resolve segurança/YAML quando aplicável e monta
   `QueuedJobEnvelope`;
2. `JobCoreJobQueue` persiste a run `queued`;
3. o poller informa as routing keys existentes no registry e chama `claim_next_run(...)`;
4. o PostgreSQL escolhe um job elegível com `FOR UPDATE ... SKIP LOCKED`;
5. claim, owner, timestamps e evento `job_core.execution.claimed` são gravados na mesma transação;
6. o worker entrega a run já reivindicada a `JobCoreExecutor.execute_claimed(...)`;
7. o executor cria o processo, mantém heartbeat e conclui ou materializa a árvore.

`not_before_at` é um campo tipado do envelope. Job futuro permanece `queued`; jobs vencidos são
ordenados por due time efetivo, criação e `job_id`. Isso implementa due time e FIFO determinístico,
não prioridade ponderada.

## Quatro níveis de concorrência

Estes conceitos não são equivalentes:

| Nível | Dono | Efeito real |
|---|---|---|
| Slot físico local | `JobCoreWorkerPollingRuntime.max_concurrency` | limita futures/threads somente naquele processo de worker |
| Admissão global durável | `PostgresJobRunStore.claim_next_run` | coordena réplicas pelo ledger compartilhado, routing key, due time, ownership, exclusividade e cap parental |
| Cap de filhos diretos | `ChildWorkPlan.desired_max_active_children` → `JobEnvelope.max_active_children` do pai | limita quantos filhos diretos do mesmo pai ocupam estados ativos |
| Concorrência interna | processo/engine | controla threads, OMP ou I/O dentro de um job já admitido; não concede vaga no Core |

`concurrency_key` acrescenta exclusividade global para uma chave opaca com ambiente/tenant; é cap
1 por chave, não quota comercial nem pool genérico de recursos.

## Árvore e atomicidade parental

O processo pai devolve um `ChildWorkPlan` completo. Cada `ChildWorkItem` informa `process_key`,
`work_key` e input. Antes de persistir qualquer filho, o executor resolve todos os descritores,
decodifica todos os inputs, deriva IDs determinísticos e monta `JobRunChildMaterialization`.

O store persiste o plano uma vez. Se ainda houver filho não terminal, o pai fica
`waiting_children`; esse estado não ocupa slot do worker nem vaga no cap. O último filho consolida
os ancestrais prontos na mesma operação de conclusão.

No PostgreSQL, a admissão de filho com cap não é “uma contagem solta” nem uma única query mágica:

1. o poller trava o candidato com `FOR UPDATE ... SKIP LOCKED`;
2. tenta um `pg_try_advisory_xact_lock` namespaced pelo pai;
3. relê `max_active_children` e conta irmãos em estados ativos;
4. se houver vaga, promove o filho e grava o evento de claim na mesma transação curta.

Os estados que ocupam vaga direta são `claimed`, `running`, `cancelling` e
`cancel_requested`. `queued` e `waiting_children` não contam. Contenção ou cap cheio deixam o
trabalho durável e retornam decisão explícita; não criam lease ou promoção no domínio.

A paridade InMemory/PostgreSQL e a prova multiprocesso estão em
[`test_02-22-14_job_core_store.py`](../../tests/unit/core/test_02-22-14_job_core_store.py),
[`test_02-22-12_job_core_postgres_store.py`](../../tests/unit/core/test_02-22-12_job_core_postgres_store.py)
e [`test_03-01-42_job_core_multiprocess_soak_n11.py`](../../tests/integration/test_03-01-42_job_core_multiprocess_soak_n11.py).

## Estados, cancelamento e reconciliação

Estados terminais e imutáveis: `cancelled`, `succeeded`, `partial_success`, `failed` e
`reconciled_failed`. Estados não terminais: `queued`, `claimed`, `running`, `waiting_children`,
`cancelling`, `cancel_requested`, `stale` e `orphaned`.

`stale`, `orphaned` e `reconciled_failed` permanecem no modelo e no DDL para leitura histórica,
mas o runtime atual não os produz. Liveness ativa considera `claimed`, `running`, `cancelling` e
`cancel_requested`. `waiting_children` só vira candidato quando a árvore prova que não há filho
ativo e ainda existe descendente não terminal preso.

Cancelamento é hierárquico e cooperativo:

- `queued` pode ir diretamente a `cancelled`;
- job ativo recebe `cancel_requested`;
- o processo observa somente `ProcessHostPort.is_cancellation_requested()` ou
  `raise_if_cancelled()`;
- o Core cancela descendentes não terminais e consolida ancestrais;
- terminal nunca é reaberto.

`JobCoreCancelOnlyReconciler` roda na cadência do próprio poller. Ele usa
`list_stale_runs(...)`/`evaluate_job_run_liveness(...)` e chama `cancel_orphaned_run(...)` por CAS.
O único terminal que pode produzir é `cancelled`; nunca recupera, reexecuta, reencaminha ou
reenfileira. O token concreto e o store ficam encapsulados no host criado pelo executor e não são
expostos ao processo.

## HIL: dois jobs, uma continuidade factual

HIL não mantém a thread ocupada nem reabre o mesmo lifecycle. O primeiro job conclui com o fato
de pedido de aprovação. A decisão humana submete um segundo job nominal de continuação. A ligação
durável usa `approval_request_id`, `thread_id`/checkpoint e fatos do domínio; cada execução recebe
seu próprio `correlation_id`.

As implementações atuais estão em
[`background_hil_continuation_process.py`](../../src/api/services/background_hil_continuation_process.py)
e [`hil_approval_decision_service.py`](../../src/api/services/hil_approval_decision_service.py).
Não existe estado `waiting_hil` no Job Core nem reutilização de correlação para fingir a mesma
execução.

## Retry: uma execução por padrão

O Job Core não possui retry automático de job, requeue, dead-letter ou ressurreição de terminal.
`not_before_at` agenda a primeira execução; não representa nova tentativa.

Retries estreitos de HTTP/OCR/I/O idempotente podem existir no colaborador de domínio. O retry
transiente do executor SQL protege a operação do ledger e pode repetir a transação segura; ele não
chama `JobProcess.execute(...)` novamente.

## Leitura, escopo e SQL

`JobRunStore` é o port único. `JobRunSnapshotQuery`, `JobRunSnapshot` e
`JobRunSnapshotPage` fornecem bulk read, filtros, liveness e paginação sem expor SQL ou envelope
bruto aos consumidores. Todo SQL sobre `job_core.job_runs` e `job_core.job_run_events` fica em
`PostgresJobRunStore`; o InMemory oferece a mesma semântica observável para testes.

`JobRunAccessScope` exige ambiente, tenant e solicitante para gestão escopada. Exclusão aceita
somente jobs terminais no mesmo escopo. Projeções de domínio combinam os snapshots do Core com
seus fatos de negócio na leitura; não fazem dual-write, overlay SQL ou predicado próprio de
liveness.

## Schema e operação

As tabelas canônicas são `job_core.job_runs` e `job_core.job_run_events`. O DDL é aplicado
manualmente. Startup/worker apenas chama validação read-only de compatibilidade; DDL em runtime é
proibido.

`parent_job_id`, `root_job_id`, `max_active_children` e `not_before_at` são persistidos pelo
contrato existente; nenhuma coluna nova foi necessária para árvore, cap, due time ou HIL em dois
jobs. As migrações e restrições aplicadas estão detalhadas no README de schema.

## Observabilidade

O snapshot, o ledger de eventos e o log canônico são evidências complementares. O fluxo registra
recebimento, validação, claim, running, heartbeat, fato de domínio, materialização/espera de filhos,
admissão deferida, cancelamento, agregação e terminal. `correlation_id` é recebido do boundary e
propagado; filhos da mesma árvore preservam a correlação lógica.

`job_core.execution.running` é log operacional; o evento durável correspondente é
`job_core.execution.executed` com status `running`. Não invente evento para espelhar nome de log.

## Testes e guardas

A suíte oficial usa marker `job_core`, opção `--with-job-core` e target `backend.job_core`:

```bash
source .venv/bin/activate
python suite_de_testes_padrao.py --backend --with-job-core --run-id <uuid>
```

[`test_02-22-07_job_architecture_boundaries.py`](../../tests/unit/core/test_02-22-07_job_architecture_boundaries.py)
impede segundo protocolo/registry/publisher, lifecycle nos domínios e qualquer SQL do ledger fora
do store. Testes de executor, contrato nominal, stores e multiprocesso protegem o wiring oficial.
Uma alteração não está concluída se só o componente isolado passa: o boundary real precisa usar o
registry, executor e store canônicos.

## Capacidades não implementadas

O contrato atual não promete:

- retry automático, requeue, dead-letter ou reabertura de job terminal;
- quotas comerciais por tenant;
- weighted fairness ou prioridade ponderada;
- streaming/paginação de `ChildWorkPlan`;
- progresso percentual persistido no Core;
- cap genérico N por recurso — `concurrency_key` oferece somente exclusividade;
- fan-out de arquivo local não compartilhado.

Se uma dessas capacidades virar requisito real, ela deve nascer como evolução genérica do Core,
com contrato, persistência, observabilidade e prova concorrente próprios. Não deve ser simulada no
domínio.
