# Manual técnico por etapa: fronteira de protocolo do AG-UI

## 1. O que esta etapa cobre

Esta etapa cobre o contrato tipado que entra e sai do slice AG-UI. Ela define que campos o frontend pode enviar, quais eventos o backend pode emitir e como interrupções, mensagens, tools e snapshots de estado trafegam sem ambiguidade.

Em termos práticos, este arquivo responde a uma pergunta operacional simples: o que um cliente AG-UI precisa falar para ser aceito pelo backend e o que ele pode esperar do stream de volta.

## 2. Contrato de entrada do run

O contrato HTTP principal é AgUiRunRequest. Ele é estrito e usa extra igual a forbid, então o boundary rejeita payloads fora da superfície oficial.

Campos decisivos confirmados no código:

- threadId e runId para identidade do stream
- executionKind para escolha explícita do adapter
- user_email para contexto e auditoria
- parentRunId quando a execução é continuação
- input e metadata como envelope de domínio
- yaml_config, yaml_inline_content ou encrypted_data como fonte explícita de configuração
- resume como lista tipada de respostas a interrupções pendentes

O helper has_config_source impede que o cliente dispare AG-UI sem contexto YAML explícito. Isso evita comportamento escondido, principalmente em demos, resumes e execuções administrativas.

## 3. Contrato de eventos

O protocolo não se resume a texto incremental. O modelo canônico do repositório expõe famílias distintas de eventos:

- lifecycle: RUN_STARTED, RUN_FINISHED, RUN_ERROR
- etapas observáveis: STEP_STARTED e STEP_FINISHED
- mensagem textual em streaming: TEXT_MESSAGE_START, TEXT_MESSAGE_CONTENT e TEXT_MESSAGE_END
- tool streaming: TOOL_CALL_START, TOOL_CALL_ARGS, TOOL_CALL_END e TOOL_CALL_RESULT
- estado: STATE_SNAPSHOT e STATE_DELTA
- customização de domínio: CUSTOM

Essa separação técnica importa porque cada superfície da UI pode decidir o que renderizar. Uma tela pode ignorar tool timeline e focar em mensagens; outra pode priorizar snapshots de dashboard e interrupções humanas.

## 4. Interrupção humana e terminal único

O protocolo não inventa um canal paralelo de HIL. Quando a execução pausa por aprovação humana, o terminal continua sendo RUN_FINISHED, mas com outcome do tipo interrupt e uma lista tipada de interrupts.

Isso traz duas vantagens operacionais:

- o cliente continua tendo um único ponto terminal para fechar o stream
- replay, auditoria e suporte não precisam conciliar dois conceitos diferentes de término

## 5. JSON Patch aceito pelo runtime

Os modelos do backend aceitam operações add, remove, replace, move, copy e test em STATE_DELTA. O store compartilhado do frontend implementa exatamente o mesmo conjunto.

Isso é importante porque o produto não depende de snapshots gigantes o tempo todo. Ele pode transportar apenas diferenças incrementais, mantendo consistência entre backend e cliente.

## 6. Erros de contrato que aparecem cedo

Os pontos de falha mais úteis dessa etapa são estes:

- campo extra não previsto no payload
- ausência de threadId, runId ou executionKind
- resume malformado para uma interrupção
- ausência total de fonte de configuração
- JSON Patch inválido ou não navegável no cliente

Em termos práticos, quando o stream nem começa ou quando o estado quebra localmente, a causa costuma estar mais no contrato do protocolo do que no domínio do adapter.

## 7. Diagnóstico recomendado

Para investigar problemas nesta etapa, o caminho mais curto é:

1. validar o payload contra AgUiRunRequest
2. conferir se executionKind e resume estão tipados corretamente
3. confirmar se o stream recebeu X-Correlation-Id
4. inspecionar se os eventos emitidos respeitam nomes e aliases oficiais
5. verificar se o cliente aplicou somente operações JSON Patch suportadas

## 8. Evidências no código

- src/api/schemas/ag_ui_models.py
  - Motivo: contrato tipado de request, replay e eventos do protocolo.
  - Comportamento confirmado: AG-UI usa modelos estritos, aliases públicos e outcome interrupt tipado.
- src/api/schemas/ag_ui_models.py
  - Motivo: operação incremental de estado.
  - Comportamento confirmado: STATE_DELTA aceita add, remove, replace, move, copy e test.
- app/ui/static/js/shared/ag-ui-state-store.js
  - Motivo: compatibilidade do cliente com os deltas emitidos pelo backend.
  - Comportamento confirmado: o store local implementa o mesmo conjunto de operações JSON Patch.
