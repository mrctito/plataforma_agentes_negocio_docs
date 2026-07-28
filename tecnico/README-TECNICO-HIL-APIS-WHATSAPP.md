# Manual técnico de HIL: APIs, aprovação durável e canais

## 1. Escopo

Human-in-the-Loop (HIL) é o contrato de pausa, revisão e retomada de uma
execução agentic. Este manual separa três caminhos que não devem ser
confundidos:

1. pausa e continuação foreground;
2. aprovação durável de execução background;
3. entrega e captura da decisão por WhatsApp, e-mail ou outro canal integrado.

As decisões públicas atuais são `approve`, `edit`, `reject` e `respond`.

## 2. Superfícies HTTP

| Endpoint | Uso |
| --- | --- |
| `POST /agent/continue` | retomar DeepAgent/agent pausado no foreground |
| `POST /workflow/continue` | retomar WorkflowAgent pausado no foreground |
| `POST /agent/hil/review-context` | carregar por token o contrato pendente para a UI segura |
| `POST /agent/hil/decisions` | decidir aprovação durável; continua inline ou publica o segundo job |
| `POST /channels/{channel_id}/messages` | receber webhook/mensagem; a bridge intercepta payload HIL antes do fluxo comum |
| `GET /admin/background-executions/hil` | listar aprovações duráveis sem expor token/hash |

As rotas por token usam `POST`. Aprovação por `GET` é proibida porque scanners
de links e previews poderiam executar a ação sem intenção humana.

## 3. Envelope público de pausa

O runtime normaliza a pausa como `AgentHilResponse`:

```json
{
  "pending": true,
  "protocol_version": "hil-http-v1",
  "message": "Revise a ação antes de continuar.",
  "allowed_decisions": ["approve", "edit", "reject", "respond"],
  "action_requests": [
    {
      "name": "enviar_pagamento",
      "args": {
        "fornecedor_id": "fornecedor-42",
        "valor": 1500.0
      }
    }
  ],
  "review_configs": [
    {
      "action_name": "enviar_pagamento",
      "allowed_decisions": ["approve", "edit", "reject"]
    }
  ],
  "interrupt_id": "interrupt_01J...",
  "interrupt_ids": ["interrupt_01J..."],
  "resume_endpoint": "/agent/continue"
}
```

O cliente renderiza somente as decisões publicadas. A ordem de
`resume.decisions` deve acompanhar a ordem de `action_requests`.

## 4. Continuação foreground

### 4.1. DeepAgent/agent

```http
POST /agent/continue
X-API-Key: <chave-agent-continue>
Content-Type: application/json

{
  "resume": {
    "decisions": [
      {"type": "approve"}
    ]
  },
  "user_email": "aprovador@empresa.com",
  "correlation_id": "20260728_120000-...",
  "thread_id": "thread_01J...",
  "mode": "deepagent",
  "encrypted_data": {
    "session_id": "<sessao>",
    "wrapped_key": "<chave-protegida>",
    "encrypted_yaml": "<yaml-cifrado>",
    "original_filename": "deepagent.yaml",
    "encryption_scheme": "FERNET+RSA-OAEP",
    "yaml_operational_contract": "root_user_session_only_v1"
  }
}
```

### 4.2. WorkflowAgent

`/workflow/continue` aceita exatamente um entre `human_response` e `resume`:

```json
{
  "thread_id": "thread_01J...",
  "correlation_id": "20260728_120000-...",
  "resume": {
    "decisions": [
      {"type": "edit", "edited_action": {
        "name": "enviar_pagamento",
        "args": {"fornecedor_id": "fornecedor-42", "valor": 1200.0}
      }}
    ]
  },
  "interrupt_ids": ["interrupt_01J..."],
  "user_email": "aprovador@empresa.com",
  "encrypted_data": {
    "session_id": "<sessao>",
    "wrapped_key": "<chave-protegida>",
    "encrypted_yaml": "<yaml-cifrado>",
    "original_filename": "workflow.yaml",
    "encryption_scheme": "FERNET+RSA-OAEP",
    "yaml_operational_contract": "root_user_session_only_v1"
  }
}
```

Foreground preserva a mesma `thread_id` e a correlação da pausa. Isso é uma
continuação, não uma execução nova.

## 5. Configuração da aprovação assíncrona

DeepAgent usa
`middlewares.human_in_the_loop.async_approval`. WorkflowAgent usa
`settings.human_in_the_loop.async_approval`.

### 5.1. DeepAgent

```yaml
multi_agents:
  - id: "supervisor-operacao"
    execution:
      type: "deepagent"
    middlewares:
      human_in_the_loop:
        enabled: true
        async_approval:
          enabled: true
          ttl_seconds: 3600
          expiration_policy: "expire"
          require_approver_match: true
          channels:
            - type: "whatsapp"
              enabled: true
              channel_id: "canal-operacao"
              template_id: "hil_aprovacao_operacao"
          approvers:
            - user_email: "aprovador@empresa.com"
              user_code: "aprovador-1"
              channel_user_ids:
                whatsapp: "5511999999999"
```

### 5.2. WorkflowAgent

```yaml
workflows:
  - id: "workflow-conciliacao"
    enabled: true
    settings:
      human_in_the_loop:
        enabled: true
        async_approval:
          enabled: true
          ttl_seconds: 3600
          expiration_policy: "fail_run"
          require_approver_match: true
          channels:
            - type: "email"
              enabled: true
              template_id: "hil_aprovacao_email"
          approvers:
            - user_email: "aprovador@empresa.com"
              user_code: "aprovador-1"
```

Contrato compartilhado:

- canais declarativos aceitos: `whatsapp` e `email`;
- `ttl_seconds`: 60 a 604800; default 3600;
- `expiration_policy`: `expire` ou `fail_run`;
- canal habilitado exige `template_id`;
- WhatsApp exige `channel_user_ids.whatsapp` de ao menos um aprovador;
- e-mail exige `user_email`;
- `async_approval.enabled=true` exige HIL principal habilitado.

## 6. Persistência

O pedido atual vive em
`agent_background.agent_hil_approval_requests`. Campos relevantes:

- identidade: `environment`, `approval_request_id`, `tenant_id`,
  `client_code`, `supervisor_id`, `agent_mode`;
- vínculo: `run_id` opcional, `correlation_id`, `thread_id`;
- usuário: `user_email`, `user_code`;
- contrato: `protocol_version`, `action_requests`, `review_configs`,
  `allowed_decisions`;
- lifecycle HIL: `status`, `notification_status`, `expires_at`;
- token: `approval_token_hash` e `approval_token_hint`;
- aprovador esperado: `expected_approver_email`, `expected_channel`,
  `expected_channel_user_id`;
- entrega: `notification_channel`, `notification_provider`,
  `provider_message_id`;
- decisão: `decision_type`, `decision_payload`, identidades do decisor,
  `decided_channel` e `decided_at`;
- auditoria adicional: `metadata`, `created_at`, `updated_at`.

Não há coluna `task_id` nessa tabela. O token bruto também não é persistido;
somente hash e hint. `run_id` é opcional porque o mesmo contrato também atende
pausas não ligadas a um run factual background.

A tabela legada `public.agent_hil_approval_requests` pode existir fisicamente,
mas não é a tabela do runtime atual.

## 7. Carregar a revisão segura

```http
POST /agent/hil/review-context
Content-Type: application/json

{
  "approval_token": "<token-recebido>",
  "approval_request_id": "hil_01J..."
}
```

Resposta:

```json
{
  "status": "pending",
  "approval_request_id": "hil_01J...",
  "correlation_id": "20260728_120000-...",
  "thread_id": "thread_01J...",
  "mode": "workflow",
  "decision_endpoint": "/agent/hil/decisions",
  "notification_channel": "whatsapp",
  "notification_provider": "meta_whatsapp",
  "hil": {
    "pending": true,
    "protocol_version": "hil-http-v1",
    "message": "Revise a ação.",
    "allowed_decisions": ["approve", "edit", "reject"],
    "action_requests": [],
    "review_configs": [],
    "interrupt_id": null,
    "interrupt_ids": [],
    "resume_endpoint": "/agent/continue"
  }
}
```

O token pode estar no fragmento `#` de uma URL de revisão, pois fragmentos não
são enviados ao servidor no request inicial. O JavaScript da página lê o
fragmento e envia o segredo no corpo do `POST`; ele não deve migrar o token
para query string.

## 8. Registrar a decisão

### 8.1. Aprovar, rejeitar ou responder

```http
POST /agent/hil/decisions
Content-Type: application/json

{
  "approval_token": "<token-recebido>",
  "approval_request_id": "hil_01J...",
  "decision_type": "approve",
  "approver_email": "aprovador@empresa.com",
  "decided_channel": "webchat"
}
```

### 8.2. Editar uma ação

Para uma única ação:

```json
{
  "approval_token": "<token-recebido>",
  "approval_request_id": "hil_01J...",
  "decision_type": "edit",
  "approver_email": "aprovador@empresa.com",
  "decided_channel": "webchat",
  "edited_action": {
    "name": "enviar_pagamento",
    "args": {"fornecedor_id": "fornecedor-42", "valor": 1200.0}
  }
}
```

Para múltiplas ações, envie `resume.decisions` completo na mesma ordem de
`action_requests`. Workflow HIL assíncrono aceita exatamente uma decisão/ação
por retomada.

## 9. Branch `200`: continuação inline

Quando o pedido não pertence a uma execução background, a decisão resolve e
continua na mesma chamada:

```json
{
  "status": "resolved",
  "approval_request_id": "hil_01J...",
  "decision_type": "approve",
  "correlation_id": "20260728_120000-...",
  "thread_id": "thread_01J...",
  "continuation": {
    "resume": {"decisions": [{"type": "approve"}]},
    "response": "Execução retomada.",
    "steps": [],
    "tools_used": [],
    "metrics": {},
    "processing_time_ms": 340.2,
    "correlation_id": "20260728_120000-...",
    "thread_id": "thread_01J...",
    "success": true,
    "error": null,
    "hil": null
  },
  "continuation_job_id": null,
  "continuation_correlation_id": null,
  "previous_execution_correlation_id": null
}
```

## 10. Branch `202`: segundo job

Quando o pedido pertence a background, a decisão é resolvida atomicamente e
um segundo job é publicado:

```json
{
  "status": "continuation_submission_confirmed",
  "approval_request_id": "hil_01J...",
  "decision_type": "approve",
  "correlation_id": "20260728_120000-...",
  "thread_id": "thread_01J...",
  "continuation": null,
  "continuation_job_id": "job_01J...",
  "continuation_correlation_id": "20260728_121500-...",
  "previous_execution_correlation_id": "20260728_120000-..."
}
```

O segundo job suporta DeepAgent e WorkflowAgent. Ele pode concluir, falhar ou
gerar outra rodada HIL. `202` não autoriza a UI a exibir “execução concluída”;
ela deve acompanhar `continuation_job_id`.

## 11. WhatsApp e outros canais

O serviço de notificação monta botões diretos apenas para `approve` e
`reject`. Quando `edit` é permitido, ele acrescenta um link para a UI segura
de revisão. Um canal simples não edita argumentos dentro do botão.

Na volta:

1. o webhook entra em `/channels/{channel_id}/messages`;
2. `HilApprovalChannelBridge` procura o payload HIL nos botões ou no texto;
3. a bridge valida token, pedido, status, expiração, decisão e principal;
4. chama o mesmo `HilApprovalDecisionService` usado pela API;
5. o fluxo comum do canal é pulado para impedir que a decisão também vire uma
   mensagem normal ao agente.

O payload do botão é construído por `HilApprovalDecisionPayloadCodec`; não
monte strings próprias para imitar esse contrato.

Os channels reconhecidos pelo modelo de decisão incluem `whatsapp`, `email`,
`teams`, `slack`, `webchat` e `instagram`, mas o bloco declarativo
`async_approval.channels` atualmente configura somente WhatsApp e e-mail.

## 12. Segurança e idempotência

O serviço valida:

- token não vazio e hash correspondente;
- `approval_request_id`, quando enviado, coerente com o token;
- status e expiração;
- decisão presente em `allowed_decisions`;
- identidade/canal esperados;
- shape de `edit` e ordem das decisões;
- modo `agent`, `deepagent` ou `workflow`;
- separação entre correlação anterior e correlação do segundo job.

Erros importantes:

| HTTP | Significado |
| --- | --- |
| `400` | decisão/shape inválido |
| `403` | pedido, canal ou aprovador não corresponde ao esperado |
| `404` | token/pedido não localizado |
| `409` | conflito de resolução ou replay incompatível |
| `410` | pedido expirado |
| `503` | submissão do segundo job indisponível |

Replay idêntico de uma submissão background confirmada pode recuperar a mesma
confirmação; replay divergente falha. O cliente nunca deve gerar uma nova
decisão para “forçar” progresso sem consultar o estado real.

## 13. Observabilidade

Eventos centrais:

- `hil.decision.received`;
- `hil.decision.accepted` ou rejeição estruturada;
- `hil.continuation.submission_confirmed`;
- `hil.continuation.publish_failed`;
- `hil.continuation.finished`;
- `hil.decision.channel.detected`;
- `hil.decision.channel.rejected`.

Registre somente ids, presença de identidade, contagens e status. Token,
argumentos sensíveis, YAML e credenciais não devem aparecer no log.

Em background, use três identidades juntas:

- `previous_execution_correlation_id`: execução que pausou;
- `continuation_correlation_id`: segundo job;
- `continuation_job_id`: lifecycle no Job Core.

## 14. Troubleshooting

### A notificação não foi enviada

Confirme `async_approval.enabled`, canal habilitado, `template_id`, aprovador e
identificador de destino. Depois leia `notification_status` e os eventos da
correlação.

### O botão foi recebido como mensagem normal

Confirme que o payload foi produzido pelo codec oficial e que a definição do
canal entra pela bridge antes do runtime conversacional.

### `edit` não aparece como botão

É o comportamento esperado. A edição usa a UI segura; botões diretos cobrem
aprovar/rejeitar.

### A decisão respondeu `202`

Consulte o segundo job. Não chame `/workflow/continue` nem `/agent/continue`
em paralelo para a mesma aprovação.

### Workflow foi rejeitado por múltiplas decisões

Workflow HIL assíncrono aceita exatamente uma ação/decisão por retomada.
Modele uma nova rodada para a próxima ação.

## 15. Fontes executáveis

- `src/api/schemas/agent_hil_models.py`
- `src/api/routers/agent_router.py`
- `src/api/services/hil_approval_decision_service.py`
- `src/api/services/hil_approval_review_query_service.py`
- `src/api/services/hil_background_approval_service.py`
- `src/api/services/hil_approval_notification_service.py`
- `src/api/services/hil_approval_channel_bridge.py`
- `src/api/services/background_hil_continuation_submission_service.py`
- `src/api/services/background_hil_continuation_execution_service.py`
- `src/api/services/workflow_hil_continuation_service.py`
- `src/api/repositories/agent_hil_approval_requests_repository.py`
- `scripts/sql/20260502_create_agent_background_schema.sql`

Tutorial prático: [WorkflowAgent com Skills e HIL pelas APIs](../usuario/TUTORIAL-101-WORKFLOW-SKILLS-HIL.md).
