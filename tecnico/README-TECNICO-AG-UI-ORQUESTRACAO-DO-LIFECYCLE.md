# Manual técnico por etapa: orquestração do lifecycle do AG-UI

## 1. O que esta etapa cobre

Esta etapa cobre o AgUiRunOrchestrator, que é a peça que governa início, sequência, terminal, persistência e normalização de falhas do stream AG-UI.

Em termos simples, ele garante que adapters diferentes pareçam a mesma experiência para o cliente.

## 2. Regras de lifecycle confirmadas

O orquestrador aplica regras rígidas:

- ele mesmo emite RUN_STARTED
- o adapter não pode emitir RUN_STARTED de novo
- cada evento pode ser persistido por sequência monotônica
- um terminal válido encerra o stream
- se o adapter não emitir terminal, o orquestrador fecha com RUN_FINISHED de sucesso

Esse desenho impede dois problemas frequentes em runtimes event-driven: adapters que controlam o lifecycle por conta própria e streams que ficam sem fechamento terminal.

## 3. Como a sequência é formada

O fluxo interno observado no código é este:

1. loga início com correlation_id, thread_id e run_id
2. monta RUN_STARTED e persiste se houver event store
3. resolve o adapter a partir de execution_kind
4. itera sobre os eventos emitidos pelo adapter
5. valida runId e tipo de evento emitido
6. persiste cada evento com sequence incremental
7. encerra ao encontrar RUN_FINISHED ou RUN_ERROR
8. se não houver terminal, injeta RUN_FINISHED de sucesso
9. loga término do run

## 4. Como as falhas são normalizadas

Há quatro classes de erro com tratamento explícito:

- adapter não registrado vira RUN_ERROR com código AG_UI_ADAPTER_NOT_FOUND
- erro de execução controlado pelo adapter vira RUN_ERROR com código explícito
- falha do event store vira RUN_ERROR com código AG_UI_EVENT_STORE_WRITE_FAILED
- exceção inesperada vira RUN_ERROR genérico AG_UI_RUN_FAILED

Isso é valioso para suporte porque separa claramente problema de catálogo, problema de domínio, problema de persistência e falha inesperada do runtime.

## 5. Garantias que o cliente recebe

Do ponto de vista do frontend, esta etapa entrega três garantias úteis:

- sempre existe RUN_STARTED antes do restante do stream
- o stream sempre termina com evento terminal ou erro normalizado
- um adapter não consegue corromper o stream emitindo runId divergente ou duplicando lifecycle inicial

## 6. Diagnóstico recomendado

Quando o stream AG-UI se comportar mal, a ordem correta de investigação é:

1. confirmar se executionKind resolve um adapter registrado
2. verificar se algum evento do adapter veio com runId divergente
3. checar se houve RUN_FINISHED ou RUN_ERROR
4. validar se o event store falhou ao persistir alguma sequência

## 7. Evidências no código

- src/api/services/ag_ui_run_orchestrator.py
  - Motivo: regra central de lifecycle.
  - Comportamento confirmado: RUN_STARTED pertence ao orquestrador e terminal ausente é fechado com sucesso automático.
- src/api/services/ag_ui_run_orchestrator.py
  - Motivo: normalização de erro.
  - Comportamento confirmado: cada classe de falha é convertida em RUN_ERROR com código estruturado.
- src/api/services/ag_ui_run_orchestrator.py
  - Motivo: integridade do stream.
  - Comportamento confirmado: eventos com runId divergente ou RUN_STARTED duplicado são recusados.
