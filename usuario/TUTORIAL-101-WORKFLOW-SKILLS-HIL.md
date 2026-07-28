# Tutorial 101: WorkflowAgent com Skills e HIL pelas APIs

Este tutorial mostra uma jornada completa e atual do produto:

1. declarar uma biblioteca de Skills no YAML;
2. selecionar Skills em um node `agent` do WorkflowAgent;
3. validar o recorte governado pela API de Assembly;
4. executar o workflow em modo síncrono ou `direct_async`;
5. retomar uma pausa HIL no foreground;
6. decidir uma aprovação HIL durável criada em background.

O exemplo executável completo está em
[`app/yaml/rag-config-workflow-skills-demo.yaml`](../../app/yaml/rag-config-workflow-skills-demo.yaml).
Use-o como referência de estrutura. Não copie credenciais, tenants ou
placeholders para produção sem adaptar o contexto do cliente.

## 1. Mapa rápido

| Objetivo | Endpoint | Resultado esperado |
| --- | --- | --- |
| Validar AST de Workflow | `POST /config/assembly/validate` | relatório semântico e fragmento compilado |
| Executar e aguardar | `POST /workflow/execute` com `direct_sync` | resposta final ou pausa ligada a uma thread |
| Executar no Job Core | `POST /workflow/execute` com `direct_async` | `task_id`, `polling_url` e `stream_url` |
| Acompanhar job | `GET /api/v1/status/{task_id}` | estado e resultado do Job Core |
| Retomar pausa foreground | `POST /workflow/continue` | continuação na mesma thread |
| Abrir revisão por token | `POST /agent/hil/review-context` | envelope HIL pendente |
| Decidir HIL durável | `POST /agent/hil/decisions` | `200` inline ou `202` para o segundo job |

## 2. Pré-requisitos

- API iniciada pelo fluxo oficial `./run.sh +a`.
- Host e porta lidos de `FASTAPI_HOST` e `FASTAPI_PORT` no `.env`.
- `FEATURE_AGENTIC_AST_ENABLED=true` para usar `/config/assembly`.
- Credencial com `config.generate` para validar a AST.
- Credencial com `workflow.execute` para executar e continuar o workflow.
- Checkpointer/store PostgreSQL resolvido quando qualquer node declarar
  `skills`.
- YAML publicado como projeto SaaS ou payload YAML cifrado pelo fluxo oficial.

Exemplo de preparação local:

```bash
source .venv/bin/activate
source .env
./run.sh +a
```

Nos comandos seguintes, considere:

```text
BASE_URL=http://<FASTAPI_HOST>:<FASTAPI_PORT>
X-API-Key=<chave-com-a-permissao-da-rota>
```

O cliente não deve criar um `correlation_id`. Capture o valor devolvido no
header `X-Correlation-Id` e no corpo. Só o reapresente quando um endpoint de
continuação exigir a identidade da execução anterior.

## 3. Declarar Skills no YAML

`skills_library` pertence à raiz do documento. DeepAgent e WorkflowAgent
consomem a mesma biblioteca, mas cada runtime seleciona apenas as Skills que
seu contrato referencia.

```yaml
agent-instructions-md: >-
  Responda apenas com regras comprovadas nas skills selecionadas.

selected_entrypoint: "workflow_atendimento"

memory:
  checkpointer:
    enabled: true
    backend: "postgresql"
    postgres:
      connection_string: "${USER_MEMORY_DATABASE_DSN}"

skills_library:
  - name: "politica-troca"
    description: >-
      Use para perguntas sobre troca, devolução, prazo de arrependimento e
      abertura do pedido.
    content: |
      # Política de troca

      - Confirme a data de entrega.
      - O prazo de arrependimento é 7 dias corridos.
      - Antes de prometer reembolso, aguarde a conferência do item.

workflows:
  - id: "workflow_atendimento"
    enabled: true
    nodes:
      - id: "consultar_politica"
        mode: "agent"
        skills:
          - "politica-troca"
        prompt:
          system: |
            Escolha a skill aplicável, abra o SKILL.md indicado pelo catálogo
            com read_file e responda somente com a regra lida.
        tools: []

      - id: "revisar_resposta"
        mode: "agent"
        skills: []
        prompt:
          system: |
            Revise a resposta anterior sem criar regra nova.
        tools: []
```

Regras que falham fechado:

- `workflows[].nodes[].skills` só pode existir em node `mode: agent`;
- cada nome selecionado precisa existir em `skills_library`;
- nomes repetidos ou itens vazios são inválidos;
- node sem Skills não recebe middleware, store ou tool de leitura;
- o conteúdo completo não entra no prompt inicial: o catálogo publica nome e
  descrição, e o modelo abre `/<skill>/SKILL.md` por `read_file` somente quando
  precisar;
- não existe tabela operacional `agent_skills`: a fonte é a biblioteca do YAML
  resolvido e o runtime materializa o arquivo no store canônico.

## 4. Validar antes de executar

O endpoint de validação recebe quatro campos obrigatórios: `target`,
`base_yaml`, `ast_payload` e `strict`. Para Workflow, o recorte governado fica
em `ast_payload`; configurações comuns, como LLM, autenticação e memória,
permanecem em `base_yaml`.

```http
POST /config/assembly/validate
X-API-Key: <chave-config-generate>
Content-Type: application/json

{
  "target": "workflow",
  "base_yaml": {
    "memory": {
      "checkpointer": {
        "enabled": true,
        "backend": "postgresql",
        "postgres": {
          "connection_string": "${USER_MEMORY_DATABASE_DSN}"
        }
      }
    }
  },
  "ast_payload": {
    "target": "workflow",
    "agent-instructions-md": "Use apenas as skills selecionadas.",
    "selected_entrypoint": "workflow_atendimento",
    "skills_library": [
      {
        "name": "politica-troca",
        "description": "Use para perguntas sobre troca e devolução.",
        "content": "# Política de troca\n\n- Confirme a data de entrega."
      }
    ],
    "workflows": [
      {
        "id": "workflow_atendimento",
        "enabled": true,
        "nodes": [
          {
            "id": "consultar_politica",
            "mode": "agent",
            "skills": ["politica-troca"],
            "prompt": {"system": "Leia a skill aplicável antes de responder."},
            "tools": []
          }
        ]
      }
    ]
  },
  "strict": true
}
```

Não prossiga apenas porque o HTTP foi `200`. O gate real é:

```json
{
  "success": true,
  "validation_report": {
    "is_valid": true,
    "error_count": 0
  }
}
```

Se `success=false`, leia `diagnostics[].code`, `diagnostics[].path` e
`diagnostics[].message`. Exemplos importantes são
`WORKFLOW_NODE_SKILLS_MODO_INVALIDO` e
`WORKFLOW_SKILL_NAO_DECLARADA_NA_LIBRARY`.

## 5. Executar o WorkflowAgent

### 5.1. Projeto SaaS publicado

Quando o YAML já está publicado e ligado a um projeto, use `projectKey`. Ele
não pode competir com `encrypted_data` no mesmo request.

```http
POST /workflow/execute
X-API-Key: <chave-workflow-execute>
Content-Type: application/json

{
  "projectKey": "atendimento-cliente",
  "message": "Qual é o prazo para devolver o produto?",
  "user_email": "usuario@empresa.com",
  "format": "text",
  "execution_mode": "direct_sync"
}
```

Guarde da resposta:

- `thread_id`, para uma eventual retomada;
- `correlation_id` e `X-Correlation-Id`, para auditoria;
- `workflow_metadata`, que contém a trilha do workflow;
- `final_response`, quando a execução terminou no mesmo request.

### 5.2. YAML cifrado

Quando a integração ainda não usa um projeto publicado, envie o envelope
produzido pelo fluxo criptográfico oficial no campo `encrypted_data`:

```json
{
  "message": "Qual é o prazo para devolver o produto?",
  "user_email": "usuario@empresa.com",
  "format": "text",
  "execution_mode": "direct_sync",
  "encrypted_data": {
    "session_id": "<sessao-efemera>",
    "wrapped_key": "<chave-fernet-protegida-por-rsa-oaep>",
    "encrypted_yaml": "<yaml-cifrado-com-fernet>",
    "original_filename": "workflow-atendimento.yaml",
    "encryption_scheme": "FERNET+RSA-OAEP",
    "yaml_operational_contract": "root_user_session_only_v1"
  }
}
```

Não invente os campos criptográficos a partir deste exemplo. Gere o envelope
pelo fluxo descrito em
[`README-EXEMPLOS-INTEGRACAO-API.md`](README-EXEMPLOS-INTEGRACAO-API.md),
que é o documento dono da criptografia de requests.

## 6. Usar `direct_async` e acompanhar o Job Core

`direct_async` não usa uma tarefa efêmera do processo FastAPI. O router publica
um job no Job Core e responde imediatamente:

```http
POST /workflow/execute
X-API-Key: <chave-workflow-execute>
Content-Type: application/json

{
  "projectKey": "atendimento-cliente",
  "message": "Analise o lote completo e produza o relatório.",
  "user_email": "usuario@empresa.com",
  "execution_mode": "direct_async",
  "estimated_duration_seconds": 180
}
```

Shape relevante da resposta:

```json
{
  "final_response": "",
  "task_id": "job_01J...",
  "status_url": "/api/v1/status/job_01J...",
  "polling_url": "/api/v1/status/job_01J...",
  "stream_url": "/api/v1/status/stream/job_01J...",
  "execution_mode": "direct_async",
  "correlation_id": "20260728_103000-...",
  "success": true
}
```

Consulte `polling_url` até o estado terminal ou abra `stream_url` por SSE.
Receber `task_id` significa “publicação aceita”, não “workflow concluído”. O
valor `subprocess` não é modo HTTP público válido e responde `422`; use
`direct_async`.

## 7. Retomar HIL no foreground

Quando uma execução direta pausa por `interrupt`, reapresente a mesma
`thread_id` e a mesma `correlation_id`. Envie exatamente um dos campos
`human_response` ou `resume`.

O modelo atual de `/workflow/continue` não possui `projectKey`; portanto, em
uma integração HTTP direta, reidrate o mesmo YAML por `yaml_config` no ambiente
controlado ou por `encrypted_data`. Exemplo com resposta humana livre e payload
cifrado:

```json
{
  "thread_id": "thread_01J...",
  "correlation_id": "20260728_103500-...",
  "human_response": "Aprovado. Prossiga com a execução.",
  "user_email": "aprovador@empresa.com",
  "encrypted_data": {
    "session_id": "<sessao-efemera>",
    "wrapped_key": "<chave-fernet-protegida-por-rsa-oaep>",
    "encrypted_yaml": "<yaml-cifrado-com-fernet>",
    "original_filename": "workflow-atendimento.yaml",
    "encryption_scheme": "FERNET+RSA-OAEP",
    "yaml_operational_contract": "root_user_session_only_v1"
  }
}
```

Para uma decisão tipada:

```json
{
  "thread_id": "thread_01J...",
  "correlation_id": "20260728_103500-...",
  "resume": {
    "decisions": [
      {"type": "approve"}
    ]
  },
  "interrupt_ids": ["interrupt_01J..."],
  "user_email": "aprovador@empresa.com",
  "encrypted_data": {
    "session_id": "<sessao-efemera>",
    "wrapped_key": "<chave-fernet-protegida-por-rsa-oaep>",
    "encrypted_yaml": "<yaml-cifrado-com-fernet>",
    "original_filename": "workflow-atendimento.yaml",
    "encryption_scheme": "FERNET+RSA-OAEP",
    "yaml_operational_contract": "root_user_session_only_v1"
  }
}
```

As decisões aceitas são `approve`, `edit`, `reject` e `respond`. Em `edit`,
`edited_action` é obrigatório; nos outros tipos, ele é proibido.

## 8. Decidir HIL durável criado em background

Uma aprovação assíncrona usa o boundary público por token. Primeiro, a tela
carrega o contexto pendente:

```http
POST /agent/hil/review-context
Content-Type: application/json

{
  "approval_token": "<token-seguro>",
  "approval_request_id": "hil_01J..."
}
```

Depois, envia a decisão:

```http
POST /agent/hil/decisions
Content-Type: application/json

{
  "approval_token": "<token-seguro>",
  "approval_request_id": "hil_01J...",
  "decision_type": "approve",
  "approver_email": "aprovador@empresa.com",
  "decided_channel": "webchat"
}
```

Há dois resultados válidos:

- `200` + `status=resolved`: a continuação ocorreu inline e o campo
  `continuation` traz o resultado;
- `202` + `status=continuation_submission_confirmed`: a execução original era
  background e um segundo job foi publicado. Acompanhe
  `continuation_job_id`; a nova execução possui
  `continuation_correlation_id` próprio e referencia
  `previous_execution_correlation_id` apenas como vínculo.

O segundo job suporta WorkflowAgent e DeepAgent. Se ele pausar novamente, uma
nova aprovação durável pode ser criada; isso é uma nova rodada HIL, não a
reabertura do pedido já decidido.

## 9. Erros comuns

| Sintoma | Causa provável | Correção |
| --- | --- | --- |
| `404` em `/config/assembly/*` | feature AST desligada | ativar `FEATURE_AGENTIC_AST_ENABLED` no ambiente correto |
| `422` ao executar | `projectKey` competiu com `encrypted_data`, ou modo inválido | enviar uma única origem de YAML e usar `direct_sync`, `direct_async` ou `auto` |
| Skill “não declarada” | nome do node não existe na biblioteca raiz | alinhar `nodes[].skills[]` com `skills_library[].name` |
| Skills em node não `agent` | contrato não suporta middleware nesse modo | mover a seleção para um node `mode: agent` |
| Falha ao montar runtime com Skills | store/checkpointer PostgreSQL não resolvido | corrigir `memory.checkpointer` e o DSN do ambiente |
| `400`/`404` na continuação | `thread_id`, correlação ou checkpoint não correspondem à pausa | usar os valores devolvidos pela execução original |
| Decisão HIL retorna `409` | pedido já decidido, expirado ou em transição | recarregar o contexto; não reenviar cegamente |
| Cliente para após `202` | confirmação confundida com conclusão | acompanhar o `continuation_job_id` no Job Core |

## 10. Checklist de integração

- [ ] O YAML passou em Assembly com `is_valid=true` e `error_count=0`.
- [ ] Cada Skill selecionada existe na biblioteca raiz.
- [ ] Apenas nodes `agent` declaram Skills.
- [ ] O store PostgreSQL está disponível para nodes com Skills.
- [ ] O cliente captura `X-Correlation-Id`; não gera um ID paralelo.
- [ ] A continuação foreground reapresenta a mesma thread e correlação.
- [ ] A automação trata `200 resolved` e `202 continuation_submission_confirmed`
      como branches diferentes.
- [ ] O consumidor de `direct_async` e do segundo job acompanha o Job Core até
      estado terminal.
- [ ] Tokens, YAML e credenciais não aparecem em logs nem em query string.

## 11. Próximas leituras

- [Manual técnico do WorkflowAgent](../tecnico/README-TECNICO-AGENTE-WORKFLOW-COMPLETO.md)
- [Manual técnico de HIL e canais](../tecnico/README-TECNICO-HIL-APIS-WHATSAPP.md)
- [Agendamento agentic e HIL em background](../tecnico/README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md)
- [Catálogo de APIs e Swagger](../tecnico/API-ENDPOINTS-SWAGGER.md)
- [Schema físico do banco](../tecnico/README-SCHEMA-BANCO.md)

## 12. Fontes executáveis deste tutorial

- `src/config/agentic_assembly/ast/workflow.py`
- `src/config/agentic_assembly/validators/workflow_semantic_validator.py`
- `src/agentic_layer/workflow/agent_workflow.py`
- `src/agentic_layer/workflow/nodes/agent_node.py`
- `src/agentic_layer/skills/skills_store_materializer.py`
- `src/api/routers/config_assembly_router.py`
- `src/api/routers/workflow_router.py`
- `src/api/routers/agent_router.py`
- `src/api/services/background_hil_continuation_execution_service.py`
- `src/api/services/workflow_hil_continuation_service.py`
