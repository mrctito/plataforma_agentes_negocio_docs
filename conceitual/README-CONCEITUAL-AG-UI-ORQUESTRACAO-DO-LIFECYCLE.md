# Manual detalhado da etapa: Orquestracao do lifecycle do AG-UI

## 1. O que esta etapa faz

Esta etapa controla o lifecycle do run AG-UI. Ela existe para garantir ordem, terminalidade, persistencia e tratamento de erro sem acoplar o core a um dominio especifico como PDV, ERP ou dashboard.

Em linguagem simples: e a camada que decide quando um run comeca, como os eventos saem em sequencia e como ele termina.

## 2. Onde ela entra no fluxo

No codigo lido, essa camada mora no AgUiRunOrchestrator. O router monta AgUiRunContext, entrega esse contexto ao orchestrator e o orchestrator resolve o adapter correto para emitir o stream.

## 3. O que entra e o que sai

Entradas confirmadas:

- AgUiRunContext imutavel com correlation_id, thread_id, run_id, user, tenant, execution_kind, input e resume
- mapping explicito de adapters por execution_kind
- event store opcional para persistencia append-only

Saidas confirmadas:

- RUN_STARTED emitido pelo proprio orchestrator
- eventos do adapter validados antes de sair
- RUN_FINISHED sintetizado quando o adapter nao fecha o ciclo sozinho
- RUN_ERROR com codigos explicitos quando algo falha

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. o orchestrator cria logger com correlation_id;
2. emite RUN_STARTED antes de chamar qualquer adapter;
3. persiste o evento com sequence monotonica quando ha event store;
4. resolve o adapter pelo execution_kind;
5. itera os eventos emitidos pelo adapter;
6. valida que o adapter nao emitiu RUN_STARTED e que o runId do evento nao diverge do contexto;
7. persiste cada evento na ordem;
8. encerra cedo se o adapter emitir RUN_FINISHED ou RUN_ERROR;
9. se nenhum terminal vier, sintetiza RUN_FINISHED com outcome success;
10. em falhas controladas ou inesperadas, transforma o problema em RUN_ERROR.

O detalhe mais importante e que o orchestrator protege o lifecycle contra adapters mal comportados. Ele nao assume que cada adapter vai respeitar sozinho todas as regras do protocolo.

## 5. Decisoes tecnicas importantes

### 5.1. RUN_STARTED pertence ao core, nao ao adapter

O adapter nao pode emitir RUN_STARTED. Isso evita duplicidade de inicio e deixa a responsabilidade do lifecycle no lugar certo.

### 5.2. Terminalidade e tratada como regra de protocolo

RUN_FINISHED e RUN_ERROR sao tratados como eventos terminais. Se o adapter nao emitir nenhum deles, o orchestrator fecha o ciclo com sucesso explicito.

### 5.3. Persistencia acompanha a ordem do stream

Cada evento recebe sequence monotonica e pode ser gravado no event store. Na pratica, replay e auditoria contam a mesma historia que o frontend viu.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- execution_kind sem adapter registrado vira RUN_ERROR AG_UI_ADAPTER_NOT_FOUND;
- adapter que emite RUN_STARTED quebra a regra e vira AG_UI_DUPLICATE_RUN_STARTED;
- evento com runId divergente quebra o stream com AG_UI_RUN_ID_MISMATCH;
- falha de persistencia no event store vira AG_UI_EVENT_STORE_WRITE_FAILED;
- excecao inesperada vira AG_UI_RUN_FAILED.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- logs com marcadores de inicio, fim e erro do orchestrator;
- presence ou ausencia de RUN_STARTED no inicio do replay;
- sequencia monotonica interrompida ou inconsistente;
- RUN_ERROR com code explicito;
- stream sem evento terminal emitido pelo adapter, mas fechado pelo success sintetico do orchestrator.

Em linguagem simples: quando o comportamento parece estranho entre router e runtime, o ponto de arbitragem e o lifecycle.

## 8. Exemplo pratico guiado

Cenario: um adapter retail_demo so emite eventos de passo, tool e texto, mas nao emite RUN_FINISHED.

1. o orchestrator inicia o run com RUN_STARTED;
2. o adapter envia os eventos do dominio;
3. nenhum terminal chega ao fim da iteracao;
4. o orchestrator sintetiza RUN_FINISHED com success;
5. o frontend recebe um run completo sem depender de regra especial no adapter.

O valor desta etapa e padronizar o lifecycle mesmo quando os runtimes internos diferem bastante.

## 9. Evidencias no codigo

- src/api/services/ag_ui_run_orchestrator.py
  - Simbolo relevante: AgUiRunContext
  - Comportamento confirmado: contexto imutavel de execucao para o slice AG-UI.
- src/api/services/ag_ui_run_orchestrator.py
  - Simbolo relevante: AgUiRunOrchestrator.run
  - Comportamento confirmado: emissao de RUN_STARTED, iteracao dos eventos do adapter e fechamento do run.
- src/api/services/ag_ui_run_orchestrator.py
  - Simbolo relevante: _validate_adapter_event
  - Comportamento confirmado: protecao contra RUN_STARTED duplicado e runId divergente.
- src/api/services/ag_ui_run_orchestrator.py
  - Simbolo relevante: blocos de excecao com AgUiRunErrorEvent
  - Comportamento confirmado: tratamento controlado de adapter ausente, falha de execucao, falha de event store e erro inesperado.
