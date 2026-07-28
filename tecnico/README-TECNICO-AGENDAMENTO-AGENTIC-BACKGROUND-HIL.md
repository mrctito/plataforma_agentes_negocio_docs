# Manual técnico de agendamento agentic, background e HIL

## 1. Escopo

Esta capacidade executa DeepAgents e WorkflowAgents fora do request HTTP, com
agenda, lifecycle durável, resultados factuais, aprovação humana e comunicação
assíncrona.

O desenho atual possui três donos claros:

- Scheduler: agenda e comando;
- Job Core: lifecycle, lease, tentativas, cancelamento e eventos;
- `agent_background`: alvos e fatos funcionais da execução.

Não existe ledger paralelo de lifecycle no domínio agentic.

## 2. Arquitetura

```text
Tool/serviço de agendamento
  -> scheduler.scheduled_jobs
  -> ocorrência publicada no Job Core
  -> JobCoreExecutor
  -> BackgroundExecutionSchedulerProcess
  -> BackgroundExecutionProcessService
  -> DeepAgent ou WorkflowAgent
  -> fato completed ou approval_requested
```

Se houver HIL:

```text
approval_requested
  -> agent_hil_approval_requests
  -> notificação/UI segura
  -> decisão atômica
  -> segundo job no Job Core
  -> continuação da mesma thread
```

## 3. Entrada de agendamento

A Tool canônica é `schedule_background_execution_request`, do catálogo interno
via `ToolsFactory`. Ela recebe:

- `target_type`: `deepagent` ou `workflow`;
- `target_ref`: id do supervisor/workflow cadastrado;
- `requested_command`: comando funcional;
- agenda tipada;
- payload, snapshots e metadados opcionais conforme o boundary interno.

Exemplo em node `tool` de WorkflowAgent:

```yaml
- id: "agendar_conciliacao"
  mode: "tool"
  params:
    tool_id: "schedule_background_execution_request"
    arguments:
      target_type: "workflow"
      target_ref: "workflow_conciliacao_financeira"
      requested_command:
        expr: "input_text"
      schedule_type: "cron"
      cron_expression: "0 2 * * *"
      timezone: "America/Sao_Paulo"
```

`target_type=agent` não é aceito pelo contrato atual.

Antes de criar a agenda, `BackgroundExecutionService`:

1. valida o ator/tenant;
2. localiza um alvo habilitado com o mesmo tenant, tipo e referência;
3. cria `request_id`;
4. monta `BackgroundExecutionScheduledRequest`;
5. grava uma única agenda no Scheduler, com o pedido completo em `payload`;
6. devolve `request_id`, `schedule_id`, `target_id`, `next_run_at` e a
   correlação da solicitação.

Não há escrita adicional em uma tabela `background_execution_requests`.

## 4. Modelo do pedido no Scheduler

`BackgroundExecutionScheduledRequest` carrega:

- `request_id`, `tenant_id`, `client_code`;
- `user_email`, `user_code`;
- `source_channel`, `source_conversation_id`;
- `target_id`, `target_type`, `target_ref`;
- `requested_command`, `normalized_command`;
- `input_payload`;
- `yaml_snapshot_hash`, `yaml_snapshot`;
- `context_snapshot`;
- `created_correlation_id`;
- `request_metadata`, `target_metadata`.

Campos obrigatórios ausentes falham fechado em `from_payload`; o worker não
completa silenciosamente uma agenda corrompida.

## 5. Publicação e execução pelo Job Core

A ocorrência do Scheduler usa:

```text
route_kind=background_execution
dispatch_mode=scheduler_execution
```

O worker reserva o job, materializa o `JobProcess` registrado e executa
`BackgroundExecutionSchedulerProcess`. Só o `JobCoreExecutor` decide estados
como queued, running, succeeded, failed ou cancelled.

`BackgroundExecutionProcessService` apenas:

1. registra o contexto factual;
2. chama o runtime agentic;
3. registra resultado ou erro funcional;
4. despacha HIL quando necessário;
5. enfileira fatos de comunicação;
6. devolve `ProcessOutcome` ou propaga a exceção.

Os handlers antigos que decidiam `running`, `waiting_hil`, `completed` e
`failed` fora do Core não fazem parte do caminho atual.

## 6. Runtime agentic

`AgenticBackgroundExecutionRuntime` seleciona a espinha pelo alvo:

- `deepagent`: usa o supervisor DeepAgent oficial;
- `workflow`: usa `WorkflowOrchestrator`.

O runtime reidrata o snapshot do YAML, valida identidade/hash, resolve chaves
pelos mecanismos canônicos e preserva o contexto de tenant/usuário. Segredo
resolvido não deve ser gravado no ledger nem no log.

O resultado funcional é normalizado como:

```python
BackgroundExecutionResult(
    outcome_kind="completed" | "approval_requested",
    final_response=...,
    thread_id=...,
    result_payload=...,
    telemetry=...,
)
```

`approval_requested` é um fato do domínio, não um status do Job Core.

## 7. Modelo físico atual

O schema `agent_background` possui quatro tabelas.

### 7.1. `background_execution_targets`

Cadastro de alvos permitidos por ambiente/tenant:

- PK `(environment, target_id)`;
- unique `(environment, tenant_id, target_type, target_ref)`;
- `target_type` limitado a `deepagent|workflow`;
- `enabled`, timezone default e metadata.

### 7.2. `background_execution_runs`

Fato consultável da execução:

- PK `(environment, tenant_id, run_id)`;
- vínculos com request, agenda, alvo e correlação;
- contexto e thread;
- `outcome_kind` limitado a `completed|approval_requested`;
- resposta, payload, telemetria ou erro funcional;
- `recorded_at` indica materialização do fato.

Não possui colunas de lease, retry, heartbeat ou status do job.

### 7.3. `agent_hil_approval_requests`

Pedido HIL durável:

- PK `(environment, approval_request_id)`;
- `run_id` opcional com FK para o fato background;
- correlação, thread, tenant, supervisor e modo;
- ações, configurações de revisão e decisões permitidas;
- hash/hint do token;
- identidade esperada e dados de notificação;
- decisão, decisor, expiração e metadata.

`decision_type` aceita `approve|edit|reject|respond`. Não existe `task_id`.

### 7.4. `background_execution_outbox`

Fatos de comunicação para `webchat`, `whatsapp`, `email`, `teams`, `slack` ou
`instagram`, com status `pending|published|failed|dead_letter`, contador de
tentativas e próxima tentativa.

Detalhes de colunas e constraints estão no
[README de schema](README-SCHEMA-BANCO.md).

## 8. Ownership entre schemas

| Pergunta | Fonte correta |
| --- | --- |
| Quando deve rodar de novo? | `scheduler.scheduled_jobs` |
| O job está queued/running/succeeded/failed? | `job_core.job_runs` |
| Quais transições e tentativas ocorreram? | `job_core.job_run_events` |
| Qual era o comando/target? | payload da agenda + contexto factual |
| Qual foi o resultado agentic? | `agent_background.background_execution_runs` |
| Há aprovação humana pendente? | `agent_background.agent_hil_approval_requests` |
| Há mensagem aguardando materialização? | `agent_background.background_execution_outbox` |

Não consulte uma projeção por proxy para responder outra pergunta.

## 9. HIL em background

Quando o resultado indica pausa humana:

1. o processo grava `outcome_kind=approval_requested`;
2. `BackgroundExecutionHilApprovalDispatcher` cria o pedido durável;
3. `HilBackgroundApprovalService` aplica TTL, política, aprovadores e canais;
4. o serviço de notificação entrega botões ou link seguro;
5. a decisão entra por `/agent/hil/decisions` ou
   `HilApprovalChannelBridge`;
6. `BackgroundHilContinuationSubmissionService` resolve a decisão e publica o
   segundo job;
7. `BackgroundHilContinuationExecutionService` retoma DeepAgent ou
   WorkflowAgent.

O segundo job possui novo `continuation_correlation_id` e referencia
`previous_execution_correlation_id`. Uma nova pausa gera novo pedido HIL.

Não existe a limitação antiga de bloquear WorkflowAgent em HIL background.

## 10. Decisão e resposta HTTP

Request:

```json
{
  "approval_token": "<token>",
  "approval_request_id": "hil_01J...",
  "decision_type": "approve",
  "approver_email": "aprovador@empresa.com",
  "decided_channel": "webchat"
}
```

Para background, a resposta é HTTP `202`:

```json
{
  "status": "continuation_submission_confirmed",
  "approval_request_id": "hil_01J...",
  "decision_type": "approve",
  "correlation_id": "20260728_130000-...",
  "thread_id": "thread_01J...",
  "continuation": null,
  "continuation_job_id": "job_01J...",
  "continuation_correlation_id": "20260728_131000-...",
  "previous_execution_correlation_id": "20260728_130000-..."
}
```

O operador acompanha `continuation_job_id` no Job Core. Repetir
`/agent/continue` ou `/workflow/continue` em paralelo criaria competição com o
fluxo durável e é incorreto.

## 11. Comunicação e webchat

O outbox separa resultado funcional de entrega. Uma execução pode ter
terminado corretamente mesmo que o canal esteja temporariamente indisponível.

Superfícies administrativas:

- `GET /admin/background-executions/communications/summary`;
- `GET /admin/background-executions/communications`;
- `POST /admin/background-executions/communications/{communication_id}/ack`.

O ack confirma materialização idempotente. Ele não altera lifecycle nem
reexecuta o agente.

## 12. API administrativa atual

| Método e rota | Uso |
| --- | --- |
| `GET /admin/background-executions/schedules` | listar agendas do Scheduler |
| `GET /admin/background-executions/runs/recent` | listar fatos recentes |
| `GET /admin/background-executions/runs/last` | último fato por request/agenda |
| `GET /admin/background-executions/runs/{run_id}` | detalhe factual |
| `GET /admin/background-executions/hil` | pedidos HIL |
| `GET /admin/background-executions/communications/summary` | resumo do outbox webchat |
| `GET /admin/background-executions/communications` | itens pendentes |
| `POST /admin/background-executions/communications/{id}/ack` | confirmar consumo |
| `DELETE /admin/background-executions/schedules/{schedule_id}` | cancelar agenda |

Não existem nesse router rotas `/requests`, `/runs/active` ou `/events`.
Lifecycle e eventos devem ser consultados no Job Core.

Todas as consultas são segregadas pelo tenant da `access_key` e pelas
permission keys administrativas de leitura/escrita.

## 13. Cancelamento e reagendamento

Cancelar uma agenda delega ao Scheduler e preserva histórico e runs já
produzidos. Não apaga fatos.

Reagendar atualiza apenas a definição da agenda e preserva o payload factual.
A Tool `reschedule_background_execution_request` cobre esse caso de uso.

Cancelamento de um job já publicado é responsabilidade do Job Core e segue o
contrato cooperativo; cancelar a agenda não implica cancelar automaticamente
uma ocorrência já em execução.

## 14. Observabilidade

Identidades que devem permanecer distinguíveis:

- `request_id`: intenção factual carregada pela agenda;
- `schedule_id`: agenda no Scheduler;
- `run_id`: fato agentic daquela ocorrência;
- `job_id`: lifecycle no Job Core;
- `correlation_id`: execução correlacionada;
- `thread_id`: estado retomável;
- `approval_request_id`: rodada HIL;
- `continuation_job_id`: segundo job.

O log deve registrar entrada, alvo, publicação, início/fim de processo,
outcome, persistência de fatos, criação/decisão HIL, publicação da continuação,
comunicação e erros. Tokens, YAML resolvido e credenciais não entram no log.

## 15. Troubleshooting

### Agenda não aparece

Confirme alvo habilitado para o tenant, `target_type`, `target_ref` e o retorno
da Tool. Consulte o Scheduler, não uma tabela agentic removida.

### Agenda existe, mas não há job

Confirme `next_run_at`, processo Scheduler/worker e publicação da ocorrência.
Depois consulte o Job Core por rota/dispatch.

### Job falhou, mas run factual está vazio

Leia eventos e log do Job Core pela correlação. Falha antes do registro de
contexto pode não produzir um fato terminal agentic.

### Fato mostra `approval_requested`

Consulte `/admin/background-executions/hil` e o pedido correspondente. Não
procure status `waiting_hil` em `background_execution_runs`.

### Decisão retornou `202`

É sucesso de submissão. Acompanhe o `continuation_job_id` até o estado
terminal.

### Resultado terminou, mas não apareceu no webchat

Consulte o outbox e suas tentativas. Não reexecute o agente antes de separar
falha funcional de falha de entrega.

## 16. Validação ponta a ponta

Uma prova completa deve demonstrar:

1. alvo cadastrado e habilitado;
2. agenda criada com payload completo;
3. ocorrência publicada no Job Core;
4. execução pela espinha correta;
5. fato agentic gravado;
6. se houver HIL, pedido durável e notificação;
7. decisão `202`, segundo job e mesma thread;
8. resultado final e comunicação materializada;
9. log sem erro ou violação de contrato.

## 17. Fontes executáveis

- `src/agentic_layer/background_execution/models.py`
- `src/agentic_layer/background_execution/services.py`
- `src/agentic_layer/background_execution/runtime.py`
- `src/agentic_layer/tools/system_tools/background_execution.py`
- `src/scheduler_layer/job_processes.py`
- `src/api/services/background_hil_continuation_submission_service.py`
- `src/api/services/background_hil_continuation_execution_service.py`
- `src/api/services/background_execution_communication_service.py`
- `src/api/routers/admin/background_execution_router.py`
- `scripts/sql/20260502_create_agent_background_schema.sql`
