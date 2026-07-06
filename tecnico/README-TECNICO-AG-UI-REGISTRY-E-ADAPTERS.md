# Manual técnico por etapa: registry e adapters do AG-UI

## 1. O que esta etapa cobre

Esta etapa cobre o catálogo explícito de adapters e o papel técnico de cada executionKind suportado no slice AG-UI.

O objetivo do registry é simples: impedir fallback implícito e concentrar o mapeamento entre executionKind e runtime real em um único lugar.

## 2. Registry padrão confirmado

O registry padrão registra exatamente três executionKinds:

- deepagent
- workflow
- capability_pack

O registro falha cedo se o nome vier vazio ou se houver duplicidade. Isso evita dois tipos de fragilidade: wiring hardcoded espalhado e sobrescrita silenciosa de adapter.

`retail_demo` e `erp_backoffice_demo` **não** são executionKinds do registry — são `capability_pack_id` de um segundo catálogo interno, o `AgUiCapabilityPackRegistry`, plugado como o único adapter do executionKind `capability_pack` (`AgUiCapabilityPackRegistryAdapter`). Quando um run chega com `execution_kind="capability_pack"`, esse adapter lê `context.metadata["capabilityPackId"]` e roteia para o pack correspondente; sem esse metadado, ou com um id não registrado, ele falha explícito (`AG_UI_CAPABILITY_PACK_ID_MISSING` / `AG_UI_CAPABILITY_PACK_NOT_REGISTERED`). Em linguagem simples: há um registry por dentro do outro — um resolve o executionKind, o segundo resolve qual pack de negócio responde por esse executionKind compartilhado.

## 3. Papel técnico de cada adapter

### 3.1. deepagent

Usa o runtime de DeepAgent, expõe `CompiledStateGraph` e delega a execução padrão para `LangGraphAgent` oficial do pacote `ag-ui-langgraph`. O resume continua usando a continuidade agentic canônica.

### 3.2. workflow

Inicializa `Workflowagent`, obtém o `CompiledStateGraph` e delega a execução padrão para `LangGraphAgent`. O resume usa executor dedicado que valida a interrupção persistida, traduz `approve`, `reject`, `cancel` e `edit` com `edited_payload` para o contrato canônico do workflow e chama o serviço canônico de continuação sem criar runtime paralelo fora de LangGraph.

### 3.3. capability_pack (retail_demo e erp_backoffice_demo)

`capability_pack` é o único executionKind do registry que não é um runtime agentic oficial. O adapter (`AgUiCapabilityPackRegistryAdapter`) delega, pelo `capabilityPackId` do metadata, a um pack concreto:

- **retail_demo** é o pack de domínio governado que entrega capabilities fechadas de varejo (query governada em `dyn_sql`, sem dashboard dinâmico — ver [README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md](README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md)), sem liberar SQL livre no browser.
- **erp_backoffice_demo** é o pack governado de backoffice, atualmente focado em `fechar_caixa` e `conferir_turno_caixa`, sem expor procedure, DSN ou segredo no discovery público.

Nenhum dos dois é um executionKind próprio nem aparece isolado em `AgUiAdapterRegistry`; ambos vivem dentro do `AgUiCapabilityPackRegistry` resolvido por `get_default_ag_ui_capability_pack_registry()`.

## 4. Por que isso importa tecnicamente

Sem o registry, o router ou o orquestrador precisariam saber detalhes de cada runtime. Com o registry explícito:

- o core só resolve executionKind
- cada adapter controla a tradução do seu domínio para AG-UI
- capacidades novas podem ser plugadas sem ifs espalhados
- executionKind inexistente falha de forma observável

## 5. Erros e limites importantes

Os comportamentos mais importantes desta etapa são:

- adapter não registrado não tem fallback
- adapter duplicado não sobrescreve registro anterior
- resume permanece dependente da capacidade real do adapter
- modos legados não aparecem no registry público AG-UI
- Workflow e DeepAgent usam o adapter oficial LangGraph quando o runtime expõe `CompiledStateGraph`

## 6. Diagnóstico recomendado

Quando um executionKind não funciona:

1. confira se ele está no registry padrão
2. valide se o adapter suporta resume ou apenas execução simples
3. verifique se a falha veio do domínio do adapter ou do catálogo de registro

## 6.1. Porta de gráfico `ChartAdapter` (registry + adapter no frontend)

O mesmo padrão registry+adapter do backend reaparece no frontend, na renderização de gráficos do envelope A2UI do chat embutível (componentes `BarChart`/`LineChart`, ver [README-TECNICO-AG-UI-RUNTIME-COMPARTILHADO-DO-FRONTEND.md](README-TECNICO-AG-UI-RUNTIME-COMPARTILHADO-DO-FRONTEND.md), seção 7.2). É uma fronteira hexagonal pequena, mas com contrato real e teste de regressão.

**Modelo neutro.** `ChartModel` descreve um gráfico independente de lib: `kind` (`bar` | `line` | `pie` | `donut`, conjunto fechado), `series` (`number[]` para série única ou `[{name, data:number[]}]` multi-série), `categories` (`string[]`, sempre texto) e `options` (`{title?, stacked?, height?, colors?}`).

**Porta + registry.** `ag-ui-chart-adapter.js` (UMD, `window.AgUiChartAdapter`) expõe:

- `WIDGET_TYPE_TO_CHART_KIND` — mapa fechado de tipo para `kind` neutro: `bar_chart→bar`, `line_chart→line`, `donut_chart→donut`. O renderer A2UI passa `'bar_chart'`/`'line_chart'` como `widgetType` ao chamar esta porta para os componentes `BarChart`/`LineChart`;
- `buildChartModel(widget, theme)` — converte dados já resolvidos (números em `series`/`categories`, diretos ou no bloco `data`) no `ChartModel`. Sem dados numéricos válidos, devolve `null` — e, no renderer A2UI, isso derruba a superfície inteira para texto (fail-closed por árvore, não por widget isolado);
- `setActiveChartAdapter` / `registerChartAdapter` / `getActiveChartAdapter` — registry de slot único do adapter ativo. `registerChartAdapter` só aceita objeto que cumpra a interface mínima (`isAvailable`, `supports`, `render`, `destroy`).

A porta **não importa nenhuma lib de gráfico**. Esse é o invariante que o teste de acoplamento protege.

**Adapter concreto.** `ag-ui-chart-adapter-apexcharts.js` (UMD) é o **único** arquivo que conhece a API do ApexCharts. Mapeia `ChartModel` → opções ApexCharts (`bar/line/pie/donut`) e, ao carregar no browser, **se auto-registra** como adapter ativo via `registerChartAdapter`. Dependência opcional: se `window.ApexCharts` (vendor em `js/vendor/apexcharts.min.js`, v5.14.0) não existir, `isAvailable()` é `false` e o gráfico não é desenhado (o que, no renderer A2UI, cai a superfície inteira em texto).

**Segurança.** O adapter desliga todo caminho de HTML: `dataLabels`/`tooltip`/`legend`/`labels`/`xaxis` usam texto puro, `legend.formatter` devolve `String(...)` sem markup, `tooltip.custom` é `undefined`, `toolbar` off. O gráfico desenha SVG só de número e texto já validados — não vira vetor de injeção.

**Trocar de lib** = escrever outro adapter implementando a mesma interface e registrá-lo; zero mudança no renderer (`ag-ui-a2ui-surface-renderer.js`, que fala só com `window.AgUiChartAdapter`). A renderização desses gráficos no componente global de chat embutível depende dessa porta — ver o [guia do componente](../usuario/GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md), seção 18.1.

## 7. Evidências no código

- app/ui/static/js/shared/ag-ui-chart-adapter.js
  - Motivo: porta neutra + registry de gráfico no frontend.
  - Comportamento confirmado: `ChartModel`, `WIDGET_TYPE_TO_CHART_KIND`, `buildChartModel` e o registry de adapter ativo; a porta não importa nenhuma lib.
- app/ui/static/js/shared/ag-ui-chart-adapter-apexcharts.js
  - Motivo: único adapter concreto.
  - Comportamento confirmado: mapeia `ChartModel` para ApexCharts, auto-registra-se e desabilita HTML em tooltip/labels/legend.
- tests/frontend/ag_ui_chart_adapter_contract.test.js
  - Motivo: proteção do desacoplamento renderer↔lib.
  - Comportamento confirmado: regressão falha se o renderer voltar a depender da lib de gráfico direto.
- src/api/services/ag_ui_adapter_registry.py
  - Motivo: catálogo explícito do slice.
  - Comportamento confirmado: o registry padrão registra apenas três executionKinds — deepagent, workflow e capability_pack.
- src/api/services/ag_ui_adapter_registry.py
  - Motivo: proteção contra wiring frágil.
  - Comportamento confirmado: nomes vazios e duplicados são rejeitados explicitamente.
- src/api/services/ag_ui_capability_pack.py
  - Motivo: segundo registry, interno ao executionKind capability_pack.
  - Comportamento confirmado: `AgUiCapabilityPackRegistry` guarda os packs por `capability_pack_id` (`retail_demo`, `erp_backoffice_demo`); `AgUiCapabilityPackRegistryAdapter` roteia por `context.metadata["capabilityPackId"]` e falha explícito sem esse metadado ou com id desconhecido.
- src/api/services/ag_ui_run_orchestrator.py
  - Motivo: acoplamento entre registry e execução.
  - Comportamento confirmado: executionKind inexistente é convertido em erro estruturado, sem adapter implícito.
