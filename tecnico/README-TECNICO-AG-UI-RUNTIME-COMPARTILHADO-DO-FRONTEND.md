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

A validação de conteudo perigoso foi centralizada em `app/ui/static/js/shared/ag-ui-safe-content.js`. Esse helper bloqueia HTML, script, SQL livre, segredos e `correlation_id` tanto no DashboardSpec quanto no ComponentCatalog. Assim, a regra de segurança nao fica espalhada em dois validadores diferentes.

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
- **spec-na-resposta:** o componente global de chat embutível **não** abre stream — detecta um spec AG-UI no corpo da resposta normalizada dos endpoints de chat (`/rag/execute`, `/agent/execute`) e o renderiza. É a superfície usada pela renderização estruturada do componente.

A camada de detecção+render é compartilhada para não duplicar lógica por tela:

**Spec-runtime** (`embeddable-chat-spec-runtime.js`, UMD): `detectAgUiSpec(payload)` procura um spec conhecido na raiz e em contêineres convencionais (`ag_ui`, `agUi`, `structured`, `ui_spec`, `spec`, `data`, `result`) e classifica como `capabilities`, `dashboard` ou `uiSpec`. `renderInto(detection, container)` delega: DashboardSpec e UISpec vão para os renderizadores **oficiais** injetados (`createAgUiDashboardRenderer` / `createAgUiUiSpecRenderer`), que rodam o próprio validador fail-closed; o CapabilitiesSpec é validado por `validateCapabilitiesSpecPayload` (espelha o contrato Pydantic: `specType="capabilities"`, `version="1.0"`, `title`, `intro`, `groups[]`, `suggestions[]`, `safety` com 4 flags `false`) e renderizado por primitivas DOM seguras (`createTextElement`/`replaceChildrenSafe`, sem `innerHTML` de conteúdo do agente, chips `<button>` sem `onclick` inline). Conteúdo inseguro (HTML/JS/SQL/segredo) reprova o spec → o componente cai em texto.

**Bridge** (`ag-ui-spec-render-bridge.js`, ESM): é o único ponto que `import`a os renderizadores oficiais (ES modules) e as primitivas seguras, monta o spec-runtime por injeção de fábrica e publica `window.PrometeuEmbeddableChatSpecRuntime`. Existe porque o componente e o spec-runtime são UMD (carregados por `<script>` clássico) e não podem importar ES modules; o bridge (`type="module"`, deferido) faz a ponte. Fail-closed: bridge ausente = `window.PrometeuEmbeddableChatSpecRuntime` indefinido = componente renderiza texto. O componente resolve esse runtime de forma lazy no momento do render, tolerando o defer do bridge.

Ordem de carregamento, ativação por flag (`renderStructured`, `welcomeCapabilities`) e estado real por host: ver o [guia do componente embutível](../usuario/GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md), seção 18.1.

## 8. Evidências no código

- app/ui/static/js/shared/embeddable-chat-spec-runtime.js
  - Motivo: detecção + registry de renderizadores AG-UI para o chat embutível.
  - Comportamento confirmado: `detectAgUiSpec` classifica o spec; DashboardSpec/UISpec delegam aos renderizadores oficiais injetados; CapabilitiesSpec é validado fail-closed e renderizado com DOM seguro.
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
  - Comportamento confirmado: DashboardSpec e ComponentCatalog usam a mesma inspeção de conteúdo inseguro.
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
