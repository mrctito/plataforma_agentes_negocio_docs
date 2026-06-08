# Manual detalhado da etapa: Fronteira de protocolo do AG-UI

## 1. O que esta etapa faz

Esta etapa define o contrato tipado que todo cliente AG-UI precisa respeitar para falar com o backend. Ela existe para impedir que cada tela, sidecar ou dominio invente seu proprio payload de request, seus proprios eventos de stream e seu proprio formato de replay.

Em linguagem simples: e a parte que transforma AG-UI em protocolo de produto, e nao em combinacao ad hoc de JSONs.

## 2. Onde ela entra no fluxo

No codigo lido, essa fronteira aparece antes do router e antes do orchestrator. O request HTTP entra como AgUiRunRequest, o stream trafega eventos tipados como RUN_STARTED, STEP_STARTED, TOOL_CALL_RESULT, STATE_DELTA e RUN_FINISHED, e o replay devolve AgUiStoredEventItem.

## 3. O que entra e o que sai

Entradas confirmadas:

- threadId e runId obrigatorios
- executionKind obrigatorio
- user_email obrigatorio
- input opcional
- resume opcional com interruptId e status
- fonte explicita de configuracao via yaml_config, yaml_inline_content ou encrypted_data

Saidas confirmadas:

- eventos AG-UI tipados para stream
- outcome terminal de sucesso ou interrupt dentro de RUN_FINISHED
- itens de replay com correlationId, sequence, eventType, payload e createdAt

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. AgUiStrictModel usa validacao estrita com extra proibido e aliases oficiais do protocolo;
2. AgUiRunRequest centraliza o contrato HTTP do run;
3. has_config_source garante que AG-UI nao execute sem fonte explicita de configuracao;
4. AgUiResumeInput define o formato canônico de retomada de interrupcao;
5. AgUiRunFinishedOutcome discrimina entre success e interrupt;
6. os eventos oficiais modelam lifecycle, mensagens, tools, snapshots, deltas e atividades;
7. AgUiStoredEventItem e AgUiReplayResponse padronizam o contrato de auditoria e replay.

O detalhe mais importante e que o contrato usa aliases camelCase de ponta a ponta. Isso reduz atrito no frontend, mas sem perder rigor porque os modelos continuam estritos.

## 5. Decisoes tecnicas importantes

### 5.1. Validacao estrita evita deriva de contrato

Os modelos baseiam-se em extra proibido. Na pratica, isso bloqueia campos inesperados e reduz a chance de a UI depender silenciosamente de payloads improvisados.

### 5.2. Interrupcao humana nao e um endpoint paralelo no protocolo

O contrato coloca interrupcoes dentro do outcome de RUN_FINISHED. Isso significa que pausa humana continua fazendo parte do lifecycle do run, e nao uma experiencia lateral desconectada.

### 5.3. Estado incremental usa JSON Patch tipado

STATE_DELTA e ACTIVITY_DELTA usam operacoes de JSON Patch. Isso permite que o frontend reconstrua estado sem exigir snapshot completo a cada evento.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- campo extra enviado pela UI quebra validacao do modelo;
- run sem threadId, runId, executionKind ou user_email nao satisfaz o contrato;
- resume sem interruptId ou status invalido nao representa uma retomada valida;
- frontend que ignore aliases oficiais pode produzir payload incompatível;
- usar AG-UI como chat generico sem fonte explicita de configuracao viola o contrato real do slice.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- erro de validacao imediatamente na borda HTTP;
- ausencia de qualquer uma das tres fontes explicitas de configuracao;
- mismatch entre o nome do campo esperado pela UI e o alias oficial do protocolo;
- outcome interrupt sem interrupts preenchidas.

Em linguagem simples: se backend e frontend parecem falar linguas diferentes, o primeiro lugar para auditar e esta fronteira.

## 8. Exemplo pratico guiado

Cenario: uma pagina quer iniciar um run deepagent com possibilidade futura de HIL.

1. a UI envia threadId, runId, executionKind, user_email e input;
2. o request inclui yaml_config ou yaml_inline_content;
3. o stream devolve RUN_STARTED;
4. o runtime pode emitir mensagens, tools, steps e deltas de estado;
5. no final, RUN_FINISHED chega com outcome success ou interrupt.

O valor desta etapa e manter o mesmo idioma de integracao para PDV, ERP, dashboard e runtimes agentic.

## 9. Evidencias no codigo

- src/api/schemas/ag_ui_models.py
  - Simbolo relevante: AgUiStrictModel
  - Comportamento confirmado: validacao estrita com aliases oficiais do protocolo.
- src/api/schemas/ag_ui_models.py
  - Simbolo relevante: AgUiRunRequest e has_config_source
  - Comportamento confirmado: contrato HTTP do run e exigencia de fonte explicita de configuracao.
- src/api/schemas/ag_ui_models.py
  - Simbolo relevante: AgUiRunFinishedOutcome e AgUiInterrupt
  - Comportamento confirmado: termino de run com sucesso ou pausa humana dentro do mesmo contrato.
- src/api/schemas/ag_ui_models.py
  - Simbolo relevante: AgUiStateDeltaEvent e AgUiStoredEventItem
  - Comportamento confirmado: deltas tipados de estado e replay auditavel.
