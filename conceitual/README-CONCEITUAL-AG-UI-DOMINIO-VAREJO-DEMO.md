# Manual detalhado da etapa: Dominio varejo demo do AG-UI

## 1. O que esta etapa faz

Esta etapa prova AG-UI como superficie de negocio concreta. Ela transforma capabilities governadas de varejo e PDV em consultas aprovadas ou materializacao segura de dashboard, e devolve tudo isso como eventos de experiencia AG-UI.

Em linguagem simples: e a parte que mostra AG-UI deixando de ser so protocolo e virando produto utilizavel.

## 2. Onde ela entra no fluxo

No codigo lido, esse dominio aparece no RetailDemoAgUiAdapter, no catalogo fechado de queries PDV e no caminho alternativo de dashboard dinamico. Ele e acionado pelo endpoint publico oficial `POST /ag-ui/runs`, usando YAML governado e `metadata.capabilityPackId=retail_demo` para selecionar o acelerador.

Em linguagem simples: a tela nao envia DSN, SQL ou segredo. Ela envia um pedido oficial AG-UI para a borda canônica, e o backend decide como executar esse dominio governado.

## 3. O que entra e o que sai

Entradas confirmadas:

- `AgUiRunRequest` canônico no endpoint `/ag-ui/runs`
- capability de negocio dentro de `input`
- parameters escalares quando a capability e baseada em query
- dashboardSpec quando a capability e dashboard_dynamic
- `metadata.capabilityPackId` quando a tela fixa o pack governado
- configuracao de ambiente DATABASE_VAREJO_DSN e DATABASE_VAREJO_SCHEMA

Saidas confirmadas:

- eventos de passo, tool call, snapshot de estado e mensagem textual
- resultado da query aprovada ou do dashboard materializado
- bloqueio explicito quando o payload tenta sair do contrato governado

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. a UI chama `POST /ag-ui/runs` com `AgUiRunRequest` canônico;
2. o boundary resolve o dominio governado pelo YAML e pelo `capabilityPackId`, sem URL paralela por `agent_id`;
3. o adapter detecta se o payload pede materializacao de dashboard;
4. se for dashboard_dynamic, delega ao DashboardMaterializationService e encerra o fluxo;
5. se nao for, transforma o input em intent de varejo;
6. bloqueia SQL livre em qualquer parte do payload;
7. resolve DSN e schema a partir do ambiente;
8. monta um RetailDemoQueryCatalog fechado e validado como read-only;
9. escolhe a query aprovada pela capability;
10. valida parametros exatos e escalares;
11. cria dyn_sql pela factory canonica com secret_key governada;
12. executa a query e devolve eventos AG-UI com tool call, snapshot e mensagem final.

O detalhe mais importante e que o usuario final nunca recebe SQL livre nem DSN. A UI conversa com capabilities de negocio, enquanto o backend faz o binding seguro com dyn_sql.

## 5. Decisoes tecnicas importantes

### 5.1. Catalogo fechado de capabilities substitui SQL ad hoc

sales_summary, checkout_funnel, catalog_opportunities e customer_segments sao consultas aprovadas. Isso reduz risco operacional e facilita demonstracao comercial.

### 5.1.1. capabilityPackId do acelerador

O identificador publico do acelerador de varejo/vendas continua sendo `retail_demo`, mas ele viaja como `metadata.capabilityPackId` e nao como parte da URL.

Isso importa porque o frontend nao precisa conhecer `executionKind`, DSN nem query interna. Na pratica, isso reduz acoplamento: a tela pede uma capability de negocio, informa o pack governado e o backend resolve a implementacao segura.

Para adaptar o acelerador a outro ERP ou PDV, mantenha a mesma ideia:

1. defina um capability pack publico para o dominio;
2. registre esse capability pack no backend;
3. exponha apenas capabilities aprovadas;
4. resolva DSN, schema, secrets e query/procedure somente no backend;
5. envie pela UI apenas `AgUiRunRequest` canônico com capability, parametros de negocio e fonte explicita de configuracao.

### 5.2. Read-only e validado no catalogo

Cada query aprovada passa por SqlReadOnlyPolicy usando sqlparse. Na pratica, a etapa se protege contra evolucao indevida para escrita no banco.

### 5.3. Dashboard dinamico e um caminho governado separado

Quando a capability e dashboard_dynamic, o fluxo muda para materializacao de DashboardSpec segura. Isso evita forcar toda experiencia visual a passar por query textual.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- DATABASE_VAREJO_DSN ou DATABASE_VAREJO_SCHEMA ausentes geram AG_UI_RETAIL_CONFIG_MISSING;
- schema invalido gera AG_UI_RETAIL_INVALID_SCHEMA;
- capability fora do catalogo gera AG_UI_RETAIL_CAPABILITY_NOT_ALLOWED;
- SQL livre no payload gera AG_UI_RETAIL_FREE_SQL_BLOCKED;
- parametros faltantes, extras ou nao escalares geram AG_UI_RETAIL_INVALID_PARAMETERS;
- se a tool dyn_sql nao for materializada, o adapter falha com AG_UI_RETAIL_TOOL_NOT_CREATED.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- erro de configuracao PDV ja no inicio da execucao;
- snapshot retailDemo com capability, queryId, toolName e result;
- tool_call_id formado por run_id mais query_id;
- diferenca entre capability esperada e capability realmente enviada no input;
- resultado vazio ou inesperado vindo da query aprovada, nao de SQL livre.

Em linguagem simples: se a tela PDV parece poderosa demais ou permissiva demais, esta etapa mostra que o desenho real e justamente o oposto, governado e fechado.

## 8. Exemplo pratico guiado

Cenario: a lideranca quer KPIs de vendas do periodo.

1. a UI chama `/ag-ui/runs` com `AgUiRunRequest` canônico;
2. dentro de `input`, a UI envia capability sales_summary com p1 e p2;
3. o adapter valida que nao existe SQL livre no payload;
4. escolhe a query pdv_vendas_kpis_periodo no catalogo;
5. executa dyn_sql com DSN resolvido por segredo do ambiente;
6. emite STEP_STARTED, TOOL_CALL_START, TOOL_CALL_RESULT, STATE_SNAPSHOT e mensagem final;
7. o orchestrator fecha o run com success.

O valor desta etapa e permitir conversa com dados de negocio sem abrir mao de governanca.

## 9. Evidencias no codigo

- src/api/services/ag_ui_retail_demo_adapter.py
  - Simbolo relevante: RetailDemoAgUiAdapter.execute
  - Comportamento confirmado: bifurcacao entre dashboard dynamic e queries aprovadas de varejo.
- src/api/services/ag_ui_retail_demo_adapter.py
  - Simbolo relevante: RetailDemoQueryCatalog
  - Comportamento confirmado: catalogo fechado de capabilities PDV e queries read-only.
- src/api/services/ag_ui_retail_demo_adapter.py
  - Simbolo relevante: EnvironmentRetailDemoSettingsProvider
  - Comportamento confirmado: obrigatoriedade de DATABASE_VAREJO_DSN e DATABASE_VAREJO_SCHEMA.
- src/api/services/ag_ui_retail_demo_adapter.py
  - Simbolo relevante: DynamicSqlRetailDemoExecutor
  - Comportamento confirmado: execucao via factory canonica dyn_sql com secret_key governada.
- src/api/services/ag_ui_capability_pack.py e src/api/services/ag_ui_retail_demo_adapter.py
  - Simbolo relevante: assert_no_free_sql_payload e _validate_parameters
  - Comportamento confirmado: bloqueio de SQL livre e validacao estrita de parametros.
- app/ui/static/js/shared/ag-ui-retail-demo-page.js
  - Simbolo relevante: RETAIL_ACCELERATOR_AGENT_ID e buildRunPayload
  - Comportamento confirmado: tela usa `/ag-ui/runs` e monta `AgUiRunRequest` com `capabilityPackId`, fonte YAML explicita e sem segredo, SQL ou `correlation_id` vindo do browser.
