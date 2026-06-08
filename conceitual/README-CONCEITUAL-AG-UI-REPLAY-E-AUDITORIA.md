# Manual detalhado da etapa: Replay e auditoria do AG-UI

## 1. O que esta etapa faz

Esta etapa persiste e reproduz a historia de execucao do AG-UI. Ela existe para suporte, troubleshooting, auditoria e governanca, sem reexpor segredos que tenham passado pelos payloads do runtime.

Em linguagem simples: e a caixa-preta legivel do run AG-UI.

## 2. Onde ela entra no fluxo

No codigo lido, o orchestrator persiste eventos no AgUiEventStore enquanto o run acontece. Depois, o router expõe replay por run e por thread usando o provider canônico do store.

## 3. O que entra e o que sai

Entradas confirmadas:

- correlation_id, thread_id, run_id, sequence e evento AG-UI
- provider canônico configurado por ambiente
- consultas de replay por run_id ou thread_id

Saidas confirmadas:

- append-only idempotente por run_id mais sequence
- replay ordenado por sequence no escopo de run
- replay ordenado por created_at, run_id e sequence no escopo de thread
- payload sanitizado com valores sensiveis redigidos

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. build_stored_event transforma AgUiBaseEvent em registro persistivel;
2. sanitize_ag_ui_event_payload limpa o payload e redige chaves sensiveis como dsn, token, password, secret, encrypted_data, correlation_id interno e SQL livre;
3. InMemoryAgUiEventStore grava eventos por run_id mais sequence de forma thread-safe e idempotente;
4. PostgresAgUiEventStore grava os mesmos eventos em store duravel com pool e retry canônicos;
5. duplicidade com mesmo run_id mais sequence e payload diferente e rejeitada explicitamente;
6. get_canonical_ag_ui_event_store resolve o provider pela configuracao ativa;
7. o router expõe replay_ag_ui_run_events e replay_ag_ui_thread_events para leitura de suporte.

O detalhe mais importante e que replay aqui nao e dump cru do runtime. E um historico sanitizado, estruturado e ordenado.

## 5. Decisoes tecnicas importantes

### 5.1. Event store e append-only por run_id mais sequence

O contrato de persistencia usa sequence monotonica por run. Isso simplifica replay, consistencia e comparacao com o stream visto pelo frontend.

### 5.2. Memory so em development e test

Fora desses ambientes, o provider precisa ser configurado explicitamente. Isso evita esconder fragilidade operacional com store volatil em producao.

### 5.3. Postgres usa retry e pool canônicos

O store duravel trabalha com psycopg pool, retry explicito e classificacao de erros transitorios. Isso o alinha ao padrao de resiliência exigido pelo repositorio.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- AG_UI_EVENT_STORE_PROVIDER ausente fora de development ou test gera erro explicito;
- AG_UI_EVENT_STORE_PROVIDER=memory fora desses ambientes e proibido;
- provider postgres sem AG_UI_EVENT_STORE_DSN falha fechado;
- sequence menor que 1 invalida o registro;
- duplicidade com payload divergente gera erro de integridade do replay;
- se o store falhar durante o run, o orchestrator devolve AG_UI_EVENT_STORE_WRITE_FAILED.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- correlationId, threadId, runId, sequence e createdAt no replay;
- erro de configuracao do provider no bootstrap do store canônico;
- lacuna de sequence ou ausencia de RUN_STARTED no historico do run;
- payload redigido onde antes havia segredo, o que confirma sanitizacao;
- payload redigido onde antes havia `sql`, `raw_sql`, `statement`, DSN ou correlation_id interno;
- falhas transitórias do postgres absorvidas ou esgotadas pela politica de retry.

Em linguagem simples: quando suporte precisa contar a historia do que aconteceu sem acessar segredos, esta etapa e o instrumento principal.

## 8. Exemplo pratico guiado

Cenario: um usuario diz que o sidecar mostrou resultado estranho e a equipe precisa auditar o run.

1. a UI ou o suporte guarda o X-Correlation-Id e o runId;
2. consulta /ag-ui/runs/{run_id}/events;
3. revisa a sequencia dos eventos, o payload sanitizado e o createdAt;
4. se precisar contexto maior, consulta /ag-ui/threads/{thread_id}/events;
5. cruza correlationId com logs do backend.

O valor desta etapa e transformar AG-UI em superficie observavel e auditavel, e nao apenas interativa.

## 9. Evidencias no codigo

- src/api/services/ag_ui_event_store.py
  - Simbolo relevante: AgUiStoredEvent e AgUiEventStore
  - Comportamento confirmado: contrato append-only com sequence monotonica por run.
- src/api/services/ag_ui_event_store.py
  - Simbolo relevante: InMemoryAgUiEventStore e PostgresAgUiEventStore
  - Comportamento confirmado: store volatil para development test e store duravel com pool e retry para postgres.
- src/api/services/ag_ui_event_store.py
  - Simbolo relevante: sanitize_ag_ui_event_payload
  - Comportamento confirmado: redacao de segredos antes do replay.
- src/api/services/ag_ui_event_store.py
  - Simbolo relevante: get_canonical_ag_ui_event_store
  - Comportamento confirmado: provider canônico sem fallback implicito fora de development test.
- src/api/routers/ag_ui_router.py
  - Simbolo relevante: replay_ag_ui_run_events e replay_ag_ui_thread_events
  - Comportamento confirmado: exposicao HTTP do replay de auditoria.
