# Manual técnico por etapa: runtime compartilhado do frontend AG-UI

## 1. O que esta etapa cobre

Esta etapa cobre o runtime JavaScript puro compartilhado entre as páginas AG-UI do repositório. Ele reúne o cliente SSE via POST, o store de estado local e o sidecar reutilizável de chat e HIL. No estado atual, o browser consome um bundle oficial versionado em `app/ui/static/js/vendor/prometeu-ag-ui-runtime.browser.js`, enquanto `app/ui/static/js/shared/ag-ui-client.js` ficou apenas como fachada fina para preservar o contrato de importação das páginas estáticas.

## 2. Cliente SSE via POST

O cliente compartilhado abre AG-UI com fetch POST, não com EventSource tradicional. Isso existe porque o contrato do run precisa enviar corpo JSON tipado.

No pacote interno `@prometeu/ag-ui-runtime`, o cliente público agora é `createPrometeuAgUiOfficialClient`, reexportado também como `createAgUiSseClient`. Ele usa o SDK oficial `@ag-ui/client` para transformar o stream HTTP em eventos AG-UI, evitando que o pacote da Plataforma de Agentes de IA mantenha parser de protocolo próprio.

O pacote Plataforma de Agentes de IA nao deve ser entendido como outro protocolo. Ele existe para facilitar a vida de quem integra com a plataforma: monta headers publicos, URLs canonicas de run e replay, contexto de diagnostico, normalizacao de tenant e mapeamento do catalogo de capabilities. Os eventos e tipos AG-UI continuam vindo dos pacotes oficiais `@ag-ui/client` e `@ag-ui/core`.

Comportamentos confirmados:

- resolve endpoint absoluto a partir do contexto atual
- monta headers Content-Type e Accept para text event stream usando helper compartilhado
- injeta X-API-Key quando a tela fornece essa credencial
- lê X-Correlation-Id do header de resposta
- usa o parser oficial do SDK JS para transformar o stream em eventos
- serve o browser por um bundle oficial versionado em `/ui/static/js/vendor`, sem expor `node_modules`
- suporta retentativa explícita controlada, mas não cria correlation id local por conta própria
- reexporta tipos oficiais de `@ag-ui/core`, quando úteis para consumidores JavaScript

## 2.1. Helpers Plataforma de Agentes de IA

Os helpers ficam em `packages/ag-ui-runtime/prometeu-helpers.js` e cobrem apenas integração da plataforma.

1. Auth: `buildPrometeuAgUiHeaders()` monta headers públicos e nunca gera `correlation_id` no browser.
2. Runs: `buildPrometeuRunsEndpoint()` monta `/ag-ui/runs` como endpoint canônico de execução.
3. Tenant: `normalizePrometeuTenantId()` normaliza texto para diagnóstico, sem injetar tenant em payload público.
4. Replay: `buildPrometeuRunReplayEndpoint()` e `buildPrometeuThreadReplayEndpoint()` montam URLs de replay canônico.
5. Diagnostics: `buildPrometeuAgUiDiagnostics()` cria um objeto enxuto só com campos presentes, sem inventar identificadores.
6. Catalog mapping: `mapPrometeuCapabilities()`, `resolvePrometeuCapability()` e `listPrometeuFrontendTools()` ajudam a consumir `/ag-ui/capabilities` sem duplicar regra em cada tela.

Na pratica, o runtime compartilhado agora enxerga uma unica superficie publica de discovery. `/ag-ui/capabilities` continua sendo o menu de negocio Plataforma de Agentes de IA, usado para listar capabilities, `frontendTools`, exemplos, `UISpecs` governadas e metadados AG-UI suficientes para o frontend e para integradores mapearem seus contratos sem uma segunda rota publica.

APIs antigas de parser manual, como `parseSseChunk` e `readSseResponse`, sao tratadas como removidas no pacote. Isso impede que o runtime da Plataforma de Agentes de IA volte a competir com o parser oficial do SDK.

## 3. Component Catalog da Plataforma de Agentes de IA

O Component Catalog da Plataforma de Agentes de IA e a lista permitida de componentes, actions, bindings e props que uma UI generativa pode usar. Ele fica em `app/ui/static/js/shared/ag-ui-component-catalog.js` e e reexportado por `@prometeu/ag-ui-runtime`.

O comportamento e fechado por padrão operacional: componente desconhecido falha, action desconhecida falha, binding desconhecido falha e prop obrigatoria ausente falha. Quando a spec passa, ela pode ser emitida como evento AG-UI `CUSTOM` com `name=prometeu.component.render` e `schemaVersion=1.0`.

A validação de conteudo perigoso foi centralizada em `app/ui/static/js/shared/ag-ui-safe-content.js`. Esse helper bloqueia HTML, script, SQL livre, segredos e `correlation_id` nas superfícies AG-UI que o consomem — inclusive o ComponentCatalog e o renderer A2UI do chat embutível (seção 7.2). Assim, a regra de segurança nao fica espalhada em varios validadores diferentes.

## 4. Store de estado compartilhado

O store local mantém um retrato completo da sessão AG-UI no browser:

- run
- messages
- tools
- state
- activities
- steps
- interrupts
- rawEvents
- customEvents
- lastEvent

Ele também aplica JSON Patch de forma compatível com o backend, inclusive move, copy e test. Isso permite que páginas estáticas complexas acompanhem snapshots e deltas sem framework reativo pesado.

## 5. Sidecar reutilizável

O sidecar encapsula um padrão pronto de interação:

- mostra status e correlation id
- renderiza mensagens e timeline de tools
- exibe contexto serializado da tela atual
- integra interrupções HIL usando o painel compartilhado
- monta payload de resume AG-UI no mesmo endpoint de run
- exige endpoint AG-UI explícito e falha fechado quando a tela não informa a rota de execução

Na prática, ele é a ponte entre o protocolo AG-UI e uma experiência administrativa reutilizável em páginas HTML estáticas.

## 6. Onde esta etapa costuma falhar

Os problemas mais comuns são:

- falha HTTP na abertura do stream
- falha de carregamento do bundle oficial do browser
- dependências oficiais JS ausentes no workspace do pacote runtime
- aplicação inválida de JSON Patch no state store
- ComponentCatalog com componente, action, binding ou prop fora da allowlist
- ausência de HilContract para montar resume
- UI sem apiKeyProvider quando a superfície precisa de X-API-Key explícito
- ausência de endpoint explícito no sidecar ou controller da página

## 7. Diagnóstico recomendado

Para investigar o runtime frontend:

1. confirme se o cliente recebeu X-Correlation-Id
2. confirme se a página carregou `/ui/static/js/vendor/prometeu-ag-ui-runtime.browser.js` com HTTP 200
3. valide se os eventos emitidos batem com o modelo AG-UI
4. confira se o store local conseguiu aplicar snapshots e deltas
5. confira se specs generativas passam por PrometeuComponentRegistry antes de renderizar
6. em HIL, garanta que o contrato de resume exista antes de clicar em aprovar ou rejeitar

## 7.1. Spec-runtime e bridge do componente de chat embutível

O runtime compartilhado tem duas formas de consumir AG-UI no frontend, e elas não se confundem:

- **sidecar (seções acima):** consome o stream SSE de `/ag-ui/runs`, reconstrói estado incremental e renderiza timeline/HIL;
- **spec-na-resposta/no-stream:** o componente global de chat embutível detecta um spec AG-UI numa resposta já normalizada — seja o corpo da resposta síncrona (CapabilitiesSpec), seja o envelope A2UI extraído do `TOOL_CALL_RESULT` de `generate_a2ui` pelo transporte SSE opt-in (ver [TUTORIAL-101-GENERATIVE-UI.md](../usuario/TUTORIAL-101-GENERATIVE-UI.md)) — e o renderiza dentro da bolha. É a superfície usada pela renderização estruturada do componente.

A camada de detecção+render é compartilhada para não duplicar lógica por tela:

**Spec-runtime** (`embeddable-chat-spec-runtime.js`, UMD): `detectAgUiSpec(payload)` procura um spec conhecido na raiz e em contêineres convencionais (`ag_ui`, `agUi`, `structured`, `ui_spec`, `spec`, `data`, `result`) e classifica em dois tipos hoje: `capabilities` (`specType === 'capabilities'`) ou `a2ui` (presença de um array `a2ui_operations`). `renderInto(detection, container)` delega: o A2UI vai para o renderer fail-closed **injetado** (`createA2uiSurfaceRenderer`, seção 7.2); o CapabilitiesSpec é validado por `validateCapabilitiesSpecPayload` (espelha o contrato Pydantic: `specType="capabilities"`, `version="1.0"`, `title`, `intro`, `groups[]`, `suggestions[]`, `safety` com 4 flags `false`) e renderizado por primitivas DOM seguras (`createTextElement`/`replaceChildrenSafe`, sem `innerHTML` de conteúdo do agente, chips `<button>` sem `onclick` inline). Conteúdo inseguro, ou componente fora do catálogo fechado, reprova o spec → o componente cai em texto.

> Um terceiro spec (`dashboard`, com renderer e validador próprios) existia nessa mesma detecção e foi removido do código junto com todo o mecanismo que o gerava no backend; hoje a única visualização estruturada gerada dinamicamente pelo DeepAgent é o A2UI.

**Bridge** (`ag-ui-spec-render-bridge.js`, ESM): é o único ponto que `import`a os renderizadores oficiais (ES modules) e as primitivas seguras, monta o spec-runtime por injeção de fábrica e publica `window.PrometeuEmbeddableChatSpecRuntime`. Existe porque o componente e o spec-runtime são UMD (carregados por `<script>` clássico) e não podem importar ES modules; o bridge (`type="module"`, deferido) faz a ponte. Fail-closed: bridge ausente = `window.PrometeuEmbeddableChatSpecRuntime` indefinido = componente renderiza texto. O componente resolve esse runtime de forma lazy no momento do render, tolerando o defer do bridge.

Ordem de carregamento, ativação por flag (`renderStructured`, `welcomeCapabilities`) e estado real por host: ver o [guia do componente embutível](../usuario/GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md), seção 18.1.

## 7.2. Renderer A2UI, o ChartAdapter e como adicionar um componente novo

Esta seção abre a caixa-preta do renderizador A2UI (`createA2uiSurfaceRenderer`, em `app/ui/static/js/shared/ag-ui-a2ui-surface-renderer.js`) — o componente que o spec-runtime invoca quando detecta o envelope `{a2ui_operations}` produzido pela tool `generate_a2ui` (wiring completo: [README-TECNICO-AG-UI.md, seção 1A](README-TECNICO-AG-UI.md)). É aqui que cada componente vira pixel.

### Despacho por `component` e fail-closed por árvore inteira

O renderer recebe as operações `createSurface`/`updateComponents`, monta um mapa de componentes e resolve as raízes. Cada componente é despachado pelo campo `component` (o nome, ex. `Card`, `BarChart`):

| `component` | Categoria | O que desenha |
|---|---|---|
| `Card`, `Column`, `Row` | layout (`LAYOUT_COMPONENTS`) | contêineres que recursivamente renderizam os filhos |
| `Text` | conteúdo | texto simples |
| `Divider` | conteúdo | linha separadora |
| `DataTable` | conteúdo | tabela (best-effort, sem fixture ainda) |
| `BarChart`, `LineChart` | gráfico | gráfico real via `ChartAdapter` + ApexCharts, ou `null` se a porta não estiver disponível |

**A validação é fail-closed pela árvore inteira, não por nó:** se `renderComponent` para qualquer nó devolver `null` (componente fora da lista acima, dado malformado, ou gráfico sem porta disponível), `renderInto` inteiro devolve `{ rendered: false }` e o chamador cai em texto — o renderer nunca desenha "pela metade".

### A porta neutra `ChartAdapter` (hexagonal no frontend, reaproveitada)

O renderer A2UI **não chama o ApexCharts direto**. Ele fala com a mesma **porta neutra** já usada por outras superfícies AG-UI, publicada em `window.AgUiChartAdapter` (`ag-ui-chart-adapter.js`); o adapter concreto que implementa essa porta com ApexCharts é `ag-ui-chart-adapter-apexcharts.js`. Em `renderChart(component, widgetType)` (`widgetType` é `'bar_chart'` ou `'line_chart'`, mapeado internamente pelo `ChartAdapter` para o `kind` neutro `bar`/`line`):

1. normaliza as séries do componente A2UI (`normalizeChartSeries` — a LLM produz o shape de dados de formas variáveis; a função reconhece as formas observadas em runtime e devolve cru quando não reconhece, para o passo seguinte falhar fechado);
2. se `chartAdapter` (injetado) não existir ou não tiver `buildChartModel`, devolve `null` direto;
3. `chartAdapter.buildChartModel(widget, { height })` transforma os dados num `ChartModel` neutro;
4. `chartAdapter.getActiveChartAdapter()` pega o adapter ativo; ApexCharts ausente → o componente de gráfico não é desenhado (e, pela regra acima, a superfície inteira cai em texto).

Dois princípios de arquitetura, idênticos aos do backend:

- **Inversão de dependência (hexagonal):** trocar a lib de gráfico não toca o renderer A2UI — basta um novo adapter que implemente a porta `ChartAdapter`. O renderer depende do contrato, não do provider.
- **Falha fechada (defense in depth):** qualquer problema no gráfico (script ausente, dado não reconhecível, tipo não suportado) derruba a superfície inteira para texto — nunca meio-desenhada.

### O catálogo fechado de 8 componentes

Ao contrário do parser de runtime do YAML (`ag_ui_generative_config.py`, deliberadamente permissivo — ver TUTORIAL-101), o **catálogo que de fato pode ser desenhado** é fechado aqui no frontend, na constante `SUPPORTED_COMPONENTS` do renderer: `Card`, `Column`, `Row`, `Text`, `BarChart`, `LineChart`, `DataTable`, `Divider`. Não há um segundo "validador" separado do renderer — a checagem de catálogo e a renderização acontecem no mesmo despacho, fail-closed.

### Como adicionar um componente novo (ex.: um `Gauge`)

Usar **mais** dos 8 componentes existentes é só listar no YAML (`a2ui_schema.components`) e deixar o supervisor pedir — zero código. Criar um componente **novo** é evolução de código, só no frontend:

1. Adicione o nome a `SUPPORTED_COMPONENTS` e trate o despacho em `renderComponent` (`ag-ui-a2ui-surface-renderer.js`) — reaproveite uma função existente ou crie uma nova.
2. Se for gráfico, o caminho limpo é **estender o `ChartAdapter`** (suportar um `kind` novo no adapter ApexCharts) em vez de desenhar direto no renderer — mantém a inversão de dependência.
3. Não é preciso mexer no parser de runtime do backend (`ag_ui_generative_config.py`) nem no AST — ambos já são permissivos quanto a nomes de componente por decisão de design (ver `docs/tecnico/README-AST-AGENTIC-DESIGNER.md`, seção 8.5.2bis).
4. Cubra com teste de contrato do renderer: `tests/frontend/ag_ui_a2ui_surface_renderer_contract.test.js`.

Regra de ouro: **nunca** desenhe HTML arbitrário do agente. O componente novo recebe só dados já validados pela fronteira `ag-ui-safe-content.js` e os monta com primitivas DOM seguras ou via `ChartAdapter`.

## 8. Evidências no código

- app/ui/static/js/shared/embeddable-chat-spec-runtime.js
  - Motivo: detecção + registry de renderizadores AG-UI para o chat embutível.
  - Comportamento confirmado: `detectAgUiSpec` classifica o spec em `capabilities` ou `a2ui` (array `a2ui_operations`); A2UI delega ao renderer fail-closed injetado; CapabilitiesSpec é validado fail-closed e renderizado com DOM seguro.
- app/ui/static/js/shared/ag-ui-spec-render-bridge.js
  - Motivo: ponte ESM↔UMD que liga os renderizadores oficiais ao spec-runtime.
  - Comportamento confirmado: importa os renderizadores ES module, injeta-os no spec-runtime e publica `window.PrometeuEmbeddableChatSpecRuntime`; ausência do bridge degrada para texto.
- packages/ag-ui-runtime/official-http-client.js
  - Motivo: wrapper oficial do cliente AG-UI no pacote Plataforma de Agentes de IA.
  - Comportamento confirmado: AG-UI usa POST com corpo JSON, preserva X-Correlation-Id e transforma o stream com `@ag-ui/client`.
- packages/ag-ui-runtime/prometeu-helpers.js
  - Motivo: helpers de integracao da Plataforma de Agentes de IA sem definir protocolo novo.
  - Comportamento confirmado: headers, agent id, replay, tenant, diagnostico, catalog mapping e frontend tools ficam centralizados.
- app/ui/static/js/shared/ag-ui-component-catalog.js
  - Motivo: catalogo seguro para UI generativa controlada.
  - Comportamento confirmado: componentes, actions, bindings e props desconhecidos falham fechado antes de renderizar.
- app/ui/static/js/shared/ag-ui-safe-content.js
  - Motivo: regra compartilhada de bloqueio de HTML, script, SQL livre, segredo e correlation_id.
  - Comportamento confirmado: ComponentCatalog e o renderer A2UI usam a mesma inspeção de conteúdo inseguro.
- app/ui/static/js/shared/ag-ui-client.js
  - Motivo: fachada estável de importação para as páginas estáticas já existentes.
  - Comportamento confirmado: o módulo reexporta o cliente oficial vindo do bundle versionado servido em `/ui/static/js/vendor`, sem manter parser SSE local.
- app/ui/static/js/vendor/prometeu-ag-ui-runtime.browser.js
  - Motivo: artefato oficial consumível pelo browser estático.
  - Comportamento confirmado: embute `@ag-ui/client` e `rxjs` para o browser, sem imports bare remanescentes.
- app/ui/static/js/shared/ag-ui-state-store.js
  - Motivo: reconstrução do estado local.
  - Comportamento confirmado: o store guarda mensagens, tools, interrupts, snapshots e deltas compatíveis com o backend.
- app/ui/static/js/shared/ag-ui-sidecar-chat.js
  - Motivo: UI compartilhada de chat e HIL.
  - Comportamento confirmado: o sidecar reutiliza o runtime comum, exige endpoint explícito e monta resume AG-UI no mesmo endpoint.
- app/ui/static/js/shared/ag-ui-a2ui-surface-renderer.js
  - Motivo: renderizador oficial do envelope A2UI (seção 7.2).
  - Comportamento confirmado: despacha por `component` contra o catálogo fechado `SUPPORTED_COMPONENTS` (8 nomes); gráficos vão pela porta neutra `ChartAdapter`; qualquer nó não reconhecido derruba a árvore inteira (`rendered: false`).
- app/ui/static/js/shared/ag-ui-chart-adapter.js
  - Motivo: porta neutra de gráfico (inversão de dependência no frontend), reutilizada pelo renderer A2UI.
  - Comportamento confirmado: expõe `buildChartModel`/`getActiveChartAdapter`/`WIDGET_TYPE_TO_CHART_KIND`; o renderer depende dela, não do ApexCharts direto.
- app/ui/static/js/shared/ag-ui-chart-adapter-apexcharts.js
  - Motivo: adapter concreto que implementa a porta com ApexCharts.
  - Comportamento confirmado: `isAvailable()`/`supports(kind)`/`render()`; ausente ou tipo não suportado → o componente de gráfico não é desenhado.
- src/agentic_layer/supervisor/ag_ui_generative_config.py
  - Motivo: fonte única do contrato de runtime do bloco `ag_ui.generative` (YAML).
  - Comportamento confirmado: parser permissivo para `a2ui_schema.components` (o catálogo fechado de verdade vive no renderer frontend, não aqui); `chat_renderer` fechado em `jspuro`/`copilotkit`. O contrato `DashboardSpec` e o módulo que o gerava não existem mais no código.
