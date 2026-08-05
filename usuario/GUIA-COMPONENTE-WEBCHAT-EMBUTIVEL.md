# Guia do Componente WebChat Embutível

## 1. Objetivo

Este documento define a v1 do componente PrometeuEmbeddableChatRuntime.

O objetivo é ter um componente de WebChat embutível, reutilizável em várias telas da aplicação, responsável por encapsular a experiência de conversa com o backend.

Em termos simples:

- a tela host configura e observa;
- o componente renderiza e conversa.

A tela hospedeira não deve redesenhar a conversa, recriar a chamada HTTP, montar payload por conta própria ou duplicar lógica de tratamento de resposta.

O componente deve entregar:

- área de mensagens;
- campo de digitação;
- botão de envio;
- controle de estado;
- histórico da sessão;
- envio da pergunta para o backend;
- tratamento da resposta;
- tratamento de erro;
- publicação de eventos;
- estado exportado para a tela host;
- adaptação ao espaço disponível no container onde foi montado.

## 2. Regra principal da v1

A v1 deve resolver o essencial.

Ela deve permitir que qualquer tela da aplicação consiga embutir um chat funcional sem virar dona da lógica do chat.

Não é objetivo da v1 criar:

- novo backend;
- novo endpoint;
- novo contrato de payload;
- cliente HTTP paralelo;
- runtime alternativo;
- framework genérico demais;
- duplicação da página ui-webchat-v3.html.

A v1 deve ser simples, clara, reutilizável e compatível com o fluxo que já funciona hoje.

## 3. Página host oficial (a antiga "página base de referência")

Histórico em uma frase: a `ui-webchat-v3.html` foi a referência funcional usada para
construir o componente; em 2026-06-10 a migração da Fase B se completou e **ela própria
virou a página host oficial do componente** — o motor de chat duplicado que ela tinha
(montagem de payload, fetch, criptografia e HIL próprios) foi removido.

http://127.0.0.1:5555/ui/static/ui-webchat-v3.html

O que isso significa na prática:

- a v3 é hoje o melhor exemplo REAL de host completo: injeta YAML/payload/e-mail, repassa
  os selects de modo/execução e os filtros de metadados, e re-liga as funções satélites
  (histórico localStorage, export, análise de log, alerta de background) sobre as APIs e
  eventos do componente;
- a fonte de verdade do ramo HTTP clássico (payload, criptografia, headers e endpoints)
  é `app/ui/static/js/shared/layout-mestre-api.js`; o ramo SSE usa o transporte AG-UI
  oficial descrito na seção 18.1;
- qualquer dúvida sobre contrato, payload, criptografia, headers, endpoint ou tratamento
  de resposta deve ser resolvida lendo a fonte única e a seção 35.2 deste guia.

A regra prática atual é:

se a sua tela host injetar contexto e delegar a conversa inteira ao componente como a
v3 faz, ela está no padrão oficial.

## 4. O que é o componente

O PrometeuEmbeddableChatRuntime é o chat em si.

Ele é um bloco reutilizável que pode ser montado dentro de uma página, painel, card, modal, drawer ou área específica da aplicação.

A tela host não precisa implementar a conversa. Ela apenas reserva um container, informa a configuração necessária e observa o estado ou eventos publicados pelo componente.

## 5. O que ele não é

Este componente não é:

- uma página administrativa completa;
- um substituto do Layout Mestre;
- um adapter do runtime DNIT legado;
- um cliente HTTP independente;
- um novo webchat concorrente;
- uma cópia da ui-webchat-v3.html;
- um lugar para regras específicas de uma única tela.

A função dele é ser um componente oficial de chat embutível, baseado no fluxo que já funciona no WebChat atual.

## 6. Onde está o código

Runtime do componente:

app/ui/static/js/shared/embeddable-chat-runtime.js

Cliente canônico usado pelo componente:

app/ui/static/js/shared/layout-mestre-api.js

Utilitários compartilhados de resposta e correlação:

app/ui/static/js/shared/ui-webchat-runtime-utils.js

Runtime de polling assíncrono (histórico — ver nota da seção 16):

app/ui/static/js/shared/ui-webchat-async-runtime.js

Transporte SSE do componente e bridge para o cliente AG-UI oficial:

app/ui/static/js/shared/embeddable-chat-ag-ui-transport.js
app/ui/static/js/shared/ag-ui-embeddable-transport-bridge.js

Cliente e store de estado AG-UI usados pela bridge:

app/ui/static/js/shared/ag-ui-client.js
app/ui/static/js/shared/ag-ui-state-store.js

Página de teste isolada (bancada) do componente:

app/ui/static/ui-embeddable-chat-test.html

Script da bancada de teste isolada:

app/ui/static/js/ui-embeddable-chat-test.js

Teste E2E Playwright Python do componente isolado:

tests/playwright/test_08-01-10_embeddable_chat_isolated.py

Página host oficial de produto (WebChat v3, host fino do componente desde 2026-06-10):

app/ui/static/ui-webchat-v3.html

Script do host v3 (contexto, selects, histórico localStorage, export, análise de log):

app/ui/static/js/ui-webchat-v3.js

Teste estrutural anti-regressão do single-source (componente + v3 protegidos):

tests/frontend/webchat_single_source_regression_contract.test.js

## 7. Ideia arquitetural correta

A separação correta é:

- o componente cuida do chat;
- a tela host cuida do contexto externo.

Na prática:

- o componente renderiza a conversa;
- o componente controla input, envio, loading, resposta, erro e histórico;
- o componente chama o backend usando o caminho canônico;
- a host page decide qual YAML usar;
- a host page decide qual userEmail usar;
- a host page decide qual apiKey usar;
- a host page decide qual modo expor ao usuário;
- a host page pode mostrar painéis externos, filtros, resumos, debug, status, histórico ou ações administrativas.

Esse desenho evita que cada tela recrie o runtime do chat do seu próprio jeito.

## 8. Dependências obrigatórias

O componente deve reutilizar dependências canônicas já existentes no projeto.

Dependências obrigatórias no browser (o construtor lança erro explícito se faltar
qualquer uma — comprovado em `embeddable-chat-runtime.js`, função
`createGenericEmbeddableChat`):

- window.prometeuLayoutMestreApi
- window.WebchatRuntimeUtils

Isso é intencional.

A v1 não deve mascarar contrato quebrado com fallback escondido.

`window.WebchatAsyncRuntime` **não** é mais checado pelo componente (ver seção 16 —
desde a decisão "Slice A", o ramo clássico é `direct_sync` e não faz polling). O parâmetro
`asyncRuntime` ainda é aceito no construtor só para não quebrar hosts/testes antigos que
o injetam, mas nunca é usado internamente. A bancada de teste isolada
(`ui-embeddable-chat-test.js`) ainda faz sua própria checagem fail-closed desse script por
herança histórica — isso é uma característica da bancada, não do componente.

Arquivos normalmente carregados antes do componente:

```html
<script src="/ui/static/js/shared/layout-mestre-api.js"></script>
<script src="/ui/static/js/shared/ui-webchat-runtime-utils.js"></script>
<script src="/ui/static/js/shared/embeddable-chat-ag-ui-transport.js"></script>
<script type="module" src="/ui/static/js/shared/ag-ui-embeddable-transport-bridge.js"></script>
<script src="/ui/static/js/shared/embeddable-chat-runtime.js"></script>
```

Os dois scripts de transporte são necessários somente quando a host habilita SSE. A
bridge importa o cliente e o store AG-UI oficiais; a host não cria parser SSE próprio.

## 9. Fluxo mental do componente

O fluxo esperado é:

1. A host page carrega as dependências obrigatórias.
2. A host page cria a instância do componente.
3. A host page monta o componente em um container HTML.
4. A host page injeta contexto, como YAML, payload, e-mail, chave e modo.
5. O componente valida se possui contexto mínimo para enviar.
6. O usuário digita uma pergunta.
7. O componente monta o payload seguindo o padrão da ui-webchat-v3.html.
8. O componente escolhe um dos dois caminhos oficiais: cliente clássico, ou
   `POST /ag-ui/runs` com SSE quando o gate opt-in está satisfeito.
9. No SSE, o componente atualiza uma única mensagem provisória conforme os fragmentos
   chegam; no clássico, recebe a resposta completa.
10. O componente atualiza mensagens, estado, correlation_id, última resposta e histórico.
11. A host page pode reagir por eventos ou lendo o estado exportado.

## 10. Responsabilidade da tela host

A tela host continua existindo, mas muda de responsabilidade.

Ela deve cuidar de:

- reservar o container onde o chat será montado;
- carregar ou resolver o YAML;
- informar yamlContent ou encryptedPayload;
- informar userEmail;
- informar apiKey;
- escolher mode;
- escolher conscientemente entre o ramo clássico e o SSE;
- informar filtros de metadados, se houver;
- ouvir eventos do componente;
- ler estado exportado quando necessário;
- exibir informações externas ao chat, se fizer sentido.

A tela host não deve:

- renderizar a conversa principal;
- montar payload manualmente fora do caminho canônico;
- criar cliente HTTP paralelo;
- interpretar resposta da API com lógica duplicada;
- competir com o DOM interno do componente;
- criar outro contrato de envio.

## 11. Responsabilidade do componente

O componente deve cuidar de:

- renderizar área de mensagens;
- renderizar campo de digitação;
- renderizar botão de envio;
- controlar estado de envio;
- impedir reentrada enquanto o estado ainda é `sending`; durante `streaming`, a host deve
  evitar iniciar outra rodada (limite da seção 18.3);
- montar o payload conforme a ui-webchat-v3.html;
- enviar a pergunta para a API;
- tratar resposta;
- tratar erro;
- preservar correlation_id;
- manter histórico da sessão;
- publicar eventos;
- exportar estado;
- adaptar-se ao container recebido.

## 12. Configuração suportada

A criação do componente e o método definirConfiguracao(config) devem aceitar os principais campos abaixo:

- apiBaseUrl: base da API. Se não for informada, usa a origem da página.
- yamlContent: conteúdo YAML em texto puro.
- encryptedPayload: payload já preparado pela host page, usado quando a host não trabalhar com yamlContent.
- projectKey: chave de projeto autorizada pelo servidor. Quando usada, não pode competir
  com YAML, payload criptografado nem API key no navegador.
- yamlFilename: nome lógico do YAML.
- userEmail: e-mail obrigatório da sessão.
- apiKey: chave explícita enviada no padrão da aplicação quando o boundary exigir;
  `setApiKey` vence a credencial resolvida do YAML, e o fluxo `projectKey` não envia essa
  chave pelo navegador.
- mode: modo de operação.
- executionMode: aceito por compatibilidade, mas **não seleciona o transporte** — no ramo
  clássico ele é normalizado para `direct_sync`; o SSE é ativado somente pelo gate
  `agUiSseTransport` + renderer + modo/projeto descrito na seção 16.
- chatRenderer: renderer declarado do chat (`jspuro` | `copilotkit`). Default `jspuro`.
  Junto com `mode` e `agUiSseTransport`, decide se o transporte AG-UI por SSE é ativado
  (ver seção 18.1).
- agUiSseTransport: liga o transporte AG-UI por SSE opt-in (default `false`). Só ativa de
  fato com `chatRenderer === 'jspuro'` e uma destas rotas: `mode === 'qa'`,
  `mode === 'deepagent'` ou `projectKey` presente (ver seção 18.1).
- scopeRef: bucket lógico da conversa (ex.: `projectId` de um projeto DNIT). Quando
  informado, viaja no envio como `scope_ref` e o backend grava/lista o turno naquele
  bucket; vazio = bucket geral.
- metadataFilters: filtros de metadados, usados principalmente em Q&A/RAG.
- threadId: identificador de thread, útil em fluxo de workflow.
- disabled: bloqueia interação do componente.
- placeholder: texto do campo de pergunta.
- submitLabel: texto do botão.
- emptyTitle / emptyMessage: título e texto do estado vazio quando o componente **ainda
  não** tem contexto mínimo para enviar.
- emptyReadyTitle / emptyReadyMessage: título e texto do estado vazio quando o contexto
  mínimo **já** está presente (`state.ready === true`) mas o usuário ainda não perguntou
  nada. Default: "Tudo pronto para conversar" / "Contexto carregado. Digite sua primeira
  pergunta no campo abaixo e clique em Enviar."
- autoFocus: indica se o campo deve receber foco automaticamente.
- minHeightPx: altura mínima quando o container ainda não tiver altura útil.
- logAnalysisUrl: URL da página de análise de log usada pelo link de correlação exibido em
  cada mensagem com `correlationId` (ver seção 19). Default:
  `/ui/static/ui-admin-plataforma-analise-logs-central-v3.html`.
- renderStructured, welcomeCapabilities, welcomeCapabilitiesSpec: controlam a renderização
  estruturada AG-UI (Capacidades/A2UI) e o onboarding de boas-vindas — detalhados na
  seção 18.1.

## 13. Contexto mínimo para envio

O componente só deve permitir envio quando existir contexto mínimo válido.

Contexto mínimo (regra canônica de `app/ui/CLAUDE.md`: a configuração deve vir de uma
fonte autorizada e o e-mail é obrigatório):

- no fluxo explícito, **uma** fonte de credencial/configuração: `yamlContent` **ou**
  `encryptedPayload` **ou** `apiKey`;
- no fluxo de projeto, `projectKey` sozinho como fonte de configuração; o servidor resolve
  a configuração autorizada;
- `userEmail` (sempre obrigatório).

O `apiKey` separado **não** é exigido quando o YAML já carrega a credencial em
`authentication.access_key` — o backend aceita a chave do header `X-API-Key` **ou** do YAML
(`src/api/security/user_auth.py`). Exigir os três ao mesmo tempo seria violar a regra.

Como o componente resolve isso de verdade (comprovado no código):

- se o host chamou `setApiKey`, essa chave **vence** e é usada como `X-API-Key`;
- sem `setApiKey`, o componente extrai `authentication.access_key` do YAML injetado,
  reusando o helper compartilhado `PrometeuYamlExtractor.extractAccessKeyFromYaml`
  (por isso o script `yaml-access-key-extractor.js` deve ser carregado antes do
  componente);
- sem nenhuma das duas fontes, o envio **falha fechado** com mensagem clara, exibida na
  linha de status — o erro de pré-condição nunca é engolido.

Se faltar e-mail, ou se não houver nenhuma fonte de credencial, o componente não envia
a pergunta e mostra o erro (inclusive quando o envio é chamado programaticamente).

## 14. Tratamento de YAML e payload

O componente deve receber o YAML já carregado pela tela host.

A host page decide qual YAML usar.

O componente consome esse YAML para montar a requisição.

A transformação de YAML em payload, o tratamento de payload criptografado e a chamada à
API acontecem na fonte única `app/ui/static/js/shared/layout-mestre-api.js` (detalhe do
fluxo HTTP real na seção 35.2).

O componente embutível não inventa outro formato.

### 14.1 Comportamento HTTP alinhado ao v3 (offline-store)

O comportamento HTTP do componente no envio do chat é idêntico ao da `ui-webchat-v3.html`: o componente NÃO faz o POST extra em `/crypto/offline-store`. A fonte única de montagem de payload/chamada de API é `app/ui/static/js/shared/layout-mestre-api.js`, que expõe a opção `registrarPayloadOffline` (default `true`). O componente embutível instancia o cliente com `registrarPayloadOffline: false`, replicando o v3. Os demais consumidores de `prometeuLayoutMestreApi` permanecem no default `true`, sem mudança de comportamento. Esse alinhamento foi confirmado por teste de contrato e por validação em runtime (o envio real pelo componente passa por `/crypto/session-key` e `/<modo>/execute`, sem `/crypto/offline-store`).

## 15. Modes suportados

Valores oficiais de mode na v1:

- qa
- agent
- deepagent
- workflow

Alias aceito por compatibilidade:

- rag

Quando rag for informado, ele deve ser tratado internamente como equivalente ao fluxo de Q&A/RAG já usado pelo WebChat atual.

O modo `agent` é puro: vai para `POST /agent/execute` sem `mode: "deepagent"` (a antiga coerção `agent→qa` foi removida em 2026-06-10).

Quem decide como cada modo monta payload e chama backend é a fonte única `layout-mestre-api.js` (dispatch do `enviar()`), consumida pelo componente.

## 16. Modos de execução: clássico síncrono ou SSE

O componente tem dois transportes, escolhidos por contrato:

1. **Ramo clássico:** qualquer `executionMode` recebido (`auto`, `direct_sync`,
   `direct_async` ou vazio) é normalizado para `direct_sync`. O componente não faz
   polling.
2. **Ramo SSE:** quando `agUiSseTransport === true`, `chatRenderer === 'jspuro'` e o modo
   é `qa`/`deepagent` ou existe `projectKey`, a rodada usa `POST /ag-ui/runs` e recebe
   eventos AG-UI progressivos.

Na prática:

- o campo `executionMode` e o método `setExecutionMode(mode)` continuam existindo na API
  (compatibilidade), mas **não mudam** o comportamento de envio;
- no clássico, a resposta final chega no mesmo request HTTP;
- no SSE, `run_id` e `thread_id` pertencem ao protocolo AG-UI e não substituem o
  `correlation_id` criado pelo backend;
- `cancelar()` tem o limite descrito na seção 18.3;
- `window.WebchatAsyncRuntime` (`ui-webchat-async-runtime.js`, método
  `waitForTaskCompletion`) deixou de ser usado pelo componente — o arquivo continua
  existindo e ainda é carregado por algumas páginas (herança histórica), mas nenhuma
  chamada real acontece a partir do componente embutível.

Se você leu uma versão anterior deste guia dizendo que o componente inteiro era
"sync-only", leia como "o ramo clássico é `direct_sync`". O streaming SSE é outro
transporte e está ativo apenas pelo gate opt-in acima.

## 17. API pública principal

A API pública principal da v1 deve priorizar nomes claros e estáveis.

### Ciclo de vida

- mount(container)
- update(nextOptions)
- destroy()

### Configuração

- definirConfiguracao(config)
- definirYaml(yaml)
- definirPayload(payload)
- setUserEmail(email)
- setApiKey(apiKey)
- setMode(mode)
- setExecutionMode(mode) — aceito por compatibilidade, sem efeito real (ver seção 16)
- setMetadataFilters(filters)
- setThreadId(threadId)
- definirScopeRef(scopeRef) — define o bucket de conversa (alias técnico: setScopeRef)
- setRenderStructured(enabled) — liga/desliga a renderização AG-UI estruturada (ver seção 18.1)
- definirCapacidadesBoasVindas(spec, enabled) — painel de capacidades como onboarding no estado vazio (alias técnico: setWelcomeCapabilities)

### Interação

- preencherPergunta(texto)
- limparCampo()
- enviarPergunta(options)
- perguntar(texto, options) — `options.payloadText` envia texto diferente do exibido (ver seção 18.5)
- cancelar() — aborta o envio síncrono em andamento via AbortController (ver seção 18.3)
- responderHil(tipoDecisao, edicoes) — decide uma pendência HIL programaticamente (ver seção 18.2)
- temHilPendente() — indica se há uma pendência HIL bloqueando novos envios (alias técnico: hasPendingHil)
- restaurarConversa(messages) — re-hidrata uma conversa completa (ver seção 18.4)
- inserirMensagemExterna(item) — injeta mensagem externa do assistente (ver seção 18.4)
- limparHistorico()
- focarCampo()

### Leitura de estado

- obterHistorico()
- obterUltimaInteracao()
- obterEstadoAtual()
- obterYamlAtual()

## 18. Aliases técnicos opcionais

Aliases podem existir por compatibilidade, mas não devem ser a interface principal do guia.

Aliases possíveis:

- setConfig(config)
- setYamlContent(yaml)
- setEncryptedPayload(payload)
- setScopeRef(scopeRef)
- getMessages()
- getLastInteraction()
- getState()
- focusInput()
- cancel()
- respondHil(...)
- hasPendingHil()
- restoreConversation(...)
- insertExternalAssistantMessage(...)

A recomendação é que novas telas usem a API pública principal.

## 18.1 Renderização AG-UI estruturada (Capacidades e A2UI)

> Para um tutorial passo a passo — como configurar o YAML, quais widgets existem, como testar e as 10 dúvidas de quem está começando — veja [TUTORIAL-101-GENERATIVE-UI.md](TUTORIAL-101-GENERATIVE-UI.md). Esta seção foca no contrato do componente; o tutorial foca em como usar.

### O que é, em uma frase

Quando a resposta do backend traz um **spec AG-UI conhecido** (um bloco de dados estruturado que descreve uma interface, e não só texto), o componente **desenha esse spec como UI visual** dentro da bolha do assistente — cards de capacidades, gráficos, KPIs, tabelas — em vez de mostrar um JSON cru ou um texto seco. Se a resposta **não** traz spec reconhecido, o componente continua mostrando texto, exatamente como antes. Esse é o ponto central: a feature é **aditiva e invisível para quem só usa texto**.

"Spec AG-UI" aqui significa: um objeto que o agente devolve descrevendo *o que mostrar* (ex.: "estes são meus assuntos", "este é o dashboard de vendas"), seguindo um contrato fixo. O componente reconhece o contrato, valida que ele é seguro e o transforma em DOM. AG-UI = *Agent-Generated UI*, interface gerada pelo agente.

> Importante não confundir transporte com detecção: o **CapabilitiesSpec** pode chegar no
> corpo da resposta clássica. O transporte SSE de `/ag-ui/runs` também atende texto
> progressivo de Q&A/RAG, DeepAgent e projetos autorizados por `projectKey`; ele não existe
> somente para A2UI. O **A2UI** (item 2 abaixo) é uma capacidade visual adicional do
> DeepAgent. Nos dois transportes, a detecção do spec acontece sobre uma mensagem já
> normalizada.
>
> Se em vez deste componente pronto você precisa falar o protocolo AG-UI diretamente — UI própria consumindo `POST /ag-ui/runs` por SSE, ou um cliente React com **CopilotKit** via `POST /ag-ui/copilotkit/runs` — o guia dedicado é [GUIA-AG-UI-SDK-TERCEIROS.md](GUIA-AG-UI-SDK-TERCEIROS.md) (seção 0 explica quando cada caminho faz sentido; seção 2.2 mapeia, recurso a recurso, o que dos recursos do CopilotKit funciona de fato com os DeepAgents desta plataforma).

### Os dois specs que o componente reconhece

1. **CapabilitiesSpec — painel "o que você faz / sobre o que falo".** Renderiza um título, uma introdução, **cards de grupos de assuntos** (cada card com um rótulo e uma descrição amigável) e **chips clicáveis de perguntas-exemplo**. Clicar num chip envia aquela pergunta **uma vez**, pelo mesmo caminho oficial de envio do componente (mesmo guard anti-duplo-envio do botão Enviar) — o usuário não precisa digitar. Serve para responder, de forma visual, perguntas como "o que você faz?", "sobre o que posso te perguntar?". O painel **nunca** mostra nomes internos de ferramenta, subdomínio ou parâmetro técnico: o backend monta o painel a partir das descrições amigáveis dos especialistas do agente e um validador barra qualquer vazamento. Não exige nenhuma configuração de YAML — a tool que o gera é auto-injetada em todo supervisor DeepAgent.

2. **A2UI — visualização condicional gerada pelo supervisor.** Quando o YAML do supervisor declara o bloco `multi_agents[].ag_ui.generative` e o usuário pede uma visualização, o supervisor DeepAgent chama a tool `generate_a2ui` e o componente desenha o resultado: cards, tabela e **gráficos reais** (barra e linha) com dados que o agente já obteve na conversa. O catálogo de componentes é fechado em 8 nomes (`Card`, `Column`, `Row`, `Text`, `Divider`, `BarChart`, `LineChart`, `DataTable`); qualquer coisa fora disso cai em texto. Sem o bloco no YAML, ou sem o transporte SSE ligado na host (ver "Como ATIVAR"), a resposta continua sempre em texto. Passo a passo completo: [TUTORIAL-101-GENERATIVE-UI.md](TUTORIAL-101-GENERATIVE-UI.md).

> Um terceiro spec (interface genérica governada, `UISpec`) chegou a ser reconhecido nessa mesma detecção do chat embutível; hoje não é mais — `ag_ui.ui_specs` continua existindo como conceito governado na plataforma, mas fora deste caminho de detecção do componente.

### Como o componente decide entre desenhar e mostrar texto

O fluxo, passo a passo, para cada resposta de assistente (sem erro):

1. o componente pega a mensagem já normalizada (do corpo da resposta síncrona, ou do envelope extraído do stream SSE opt-in) e procura um spec conhecido nela (na raiz e em contêineres convencionais como `ag_ui`, `structured`, `data`, `result`, ou o discriminador `a2ui_operations` para A2UI);
2. achou um spec? ele passa por um **validador fail-closed** (descrito abaixo). Spec inválido = tratado como se não existisse;
3. spec válido = o renderizador correspondente desenha a UI dentro da bolha;
4. **qualquer** uma destas condições cai em texto puro: não há spec reconhecido; o spec é inválido ou tem componente fora do catálogo; `renderStructured` está desligado; o runtime de renderização não foi carregado na página; a lib de gráfico está ausente (só o componente de gráfico não é desenhado — pela regra fail-closed do A2UI, isso derruba a superfície inteira para texto). Esse é o **fallback duro**: o pior caso é o comportamento de antes (texto), nunca um erro na tela.

Em uma frase: **o componente nunca fica pior do que era**. Quem não ativar nada, ou cujo agente só devolve texto, não percebe diferença.

### Como ATIVAR (ligar a feature numa host page)

A feature tem duas partes: carregar os scripts certos (uma vez, na host) e, opcionalmente, ligar o onboarding.

**1. Carregar os scripts, NESTA ordem, antes do `embeddable-chat-runtime.js`:**

```html
<!-- vendor da lib de gráfico (opcional, mas necessário para desenhar gráficos A2UI) -->
<script src="/ui/static/js/vendor/apexcharts.min.js?v=5.14.0"></script>
<!-- porta neutra de gráfico + adapter ApexCharts (o adapter se auto-registra como ativo) -->
<script src="/ui/static/js/shared/ag-ui-chart-adapter.js"></script>
<script src="/ui/static/js/shared/ag-ui-chart-adapter-apexcharts.js"></script>
<!-- renderer A2UI fail-closed (catálogo fechado de 8 componentes) -->
<script src="/ui/static/js/shared/ag-ui-a2ui-surface-renderer.js"></script>
<!-- detecção de spec + registry de renderizadores + renderer de Capacidades -->
<script src="/ui/static/js/shared/embeddable-chat-spec-runtime.js"></script>
<!-- bridge ESM: liga os renderizadores oficiais e publica o runtime montado em window -->
<script type="module" src="/ui/static/js/shared/ag-ui-spec-render-bridge.js"></script>
<!-- transporte SSE opt-in: qa, deepagent ou projectKey + chatRenderer:'jspuro' +
     agUiSseTransport:true; demais combinações seguem no ramo clássico -->
<script src="/ui/static/js/shared/embeddable-chat-ag-ui-transport.js"></script>
<script type="module" src="/ui/static/js/shared/ag-ui-embeddable-transport-bridge.js"></script>
<!-- por fim, o componente -->
<script src="/ui/static/js/shared/embeddable-chat-runtime.js"></script>
```

A ordem importa porque cada peça depende da anterior: o adapter ApexCharts só se registra se a porta `ag-ui-chart-adapter.js` já existir; o renderer A2UI espera essa porta publicada em `window`; os bridges ESM (`type="module"`, executam deferidos) exigem os módulos UMD anteriores já carregados para montar o runtime e publicá-lo em `window`. O componente resolve esses runtimes de forma **lazy** no momento de renderizar/enviar — por isso os bridges podem terminar depois da criação do componente sem quebrar nada.

**2. CapabilitiesSpec não precisa de mais nada:** `renderStructured` já vem **ligado por padrão**. Para forçar o modo 100% texto (ex.: um parceiro que só quer texto), passe `renderStructured: false` na configuração, ou chame `componente.setRenderStructured(false)` em runtime.

**3. A2UI precisa de YAML + 3 flags na config do componente.** No backend: declare o bloco `multi_agents[].ag_ui.generative` no supervisor DeepAgent (catálogo de componentes + `chat_renderer`). Na host: ligue `mode: 'deepagent'`, `chatRenderer: 'jspuro'` e `agUiSseTransport: true` na configuração do componente — as três juntas ativam o transporte SSE opt-in que pode carregar o envelope A2UI. Essas flags são requisitos do A2UI, não de todo streaming textual: Q&A e `projectKey` também podem usar SSE sem declarar visualização generativa. Passo a passo: [TUTORIAL-101-GENERATIVE-UI.md](TUTORIAL-101-GENERATIVE-UI.md). O CapabilitiesSpec continua auto-injetado, sem depender dessas flags.

**4. Onboarding (opcional):** para mostrar o painel de capacidades já no **estado vazio** do chat (antes da primeira pergunta), como boas-vindas:

```js
// liga o onboarding e injeta um CapabilitiesSpec curado pela própria host
componente.definirCapacidadesBoasVindas(specDeBoasVindas, true);
// alias técnico equivalente: componente.setWelcomeCapabilities(specDeBoasVindas, true);
```

Ou na configuração inicial: `welcomeCapabilities: true` + `welcomeCapabilitiesSpec: <spec>`. Esse opt-in vem **desligado por padrão** (sem ele, o estado vazio é idêntico ao de antes). O componente **nunca busca o spec sozinho** — a host injeta, coerente com o contrato do componente embutível (a host carrega o contexto, o componente não vai atrás).

### Segurança: por que isso não vira porta de injeção

Todo spec — de Capacidades ou A2UI — passa por **validação fail-closed** antes de virar DOM, espelhando no frontend o mesmo rigor de segurança que o backend já aplica. Na prática:

- HTML, JavaScript, SQL livre e segredos são **bloqueados** (chave proibida ou string insegura = spec inválido = cai em texto);
- a renderização usa **primitivas DOM seguras**: o conteúdo do agente entra como `textContent`, nunca como `innerHTML`; não há `onclick` inline;
- os chips de pergunta são `<button>` semânticos, sem `onclick` inline — o clique é capturado por **delegação de evento** no container de mensagens, que lê a pergunta de um atributo `data-`;
- os gráficos do ApexCharts desenham **SVG a partir de números e rótulos de texto já validados**, com HTML desabilitado em tooltip, dataLabels, legenda e labels — o gráfico não pode virar vetor de injeção.

O CapabilitiesSpec ainda carrega quatro flags de segurança (`htmlAllowed`, `scriptAllowed`, `sqlAllowed`, `secretsAllowed`), todas **sempre `false`** — se vierem diferentes, o spec é recusado.

### Erros a evitar (pegadinhas)

- **Esquecer o bridge ESM ou a ordem dos scripts.** Sem `ag-ui-spec-render-bridge.js` carregado, o componente não encontra o runtime de spec em `window` e renderiza texto — você vai achar que a feature "não funciona", quando na verdade ela degradou para o fallback. Confira a ordem da seção "Como ATIVAR".
- **Esperar gráfico sem o vendor.** Sem `apexcharts.min.js`, o componente de gráfico não é desenhado — e, diferente de um placeholder isolado, isso derruba a **superfície A2UI inteira** para texto (o renderer é fail-closed por árvore inteira, não por componente). Isso é por design, não bug.
- **Achar que toda resposta vira UI.** Só vira UI a resposta que carrega um spec reconhecido e válido. Resposta de texto continua texto. Render estruturado vale **só** para mensagem de assistente sem erro.
- **Tentar montar o payload/spec na host.** A host não inventa spec nem mexe no contrato — quem emite o spec é o backend (a tool de capacidades ou o supervisor DeepAgent via `generate_a2ui`). A host, no máximo, injeta um CapabilitiesSpec **curado** para o onboarding de boas-vindas.

### Estado real por host (honesto)

A renderização estruturada está ativa apenas onde o wiring visual completo foi carregado.
Isso é diferente do transporte textual SSE, que pode existir sem os renderizadores A2UI:

- **`ui-webchat-v3.html`** (a página oficial de WebChat, host do componente desde 2026-06-10): carrega a cadeia AG-UI completa (ApexCharts → adapter → adapter-apexcharts → spec-runtime) **antes** do componente, com gate falha-fechada que exige o spec runtime resolvido → **ATIVA pelo wiring e pelos testes focados**. A rodada documental de 2026-08-04 não repetiu prova browser + API-live.
- **`ui-embeddable-chat-test.html`** (bancada de teste isolada): wiring completo → **ATIVA**.

## 18.2 HIL (Human-in-the-loop) no componente

HIL é o fluxo em que o agente pausa e pede aprovação humana antes de executar uma ação
(por exemplo, rodar uma tool sensível). Desde a Fase A da migração, esse fluxo vive
**dentro do componente** — a tela host não implementa nada de HIL.

O que o componente faz sozinho quando a resposta do backend traz uma pendência HIL:

- valida o contrato da resposta com `HilContract.normalizeResponse` (fail-closed: contrato
  inválido vira erro explícito, nunca aprovação silenciosa);
- monta o painel compartilhado `HilReviewPanel` na própria bolha da mensagem, com as ações
  de aprovar, rejeitar e editar;
- **bloqueia novos envios** enquanto a pendência existir (o usuário precisa decidir antes
  de continuar a conversa);
- executa a decisão pelos 2 caminhos oficiais, sempre via fonte única: com `approvalToken`
  → `enviarDecisaoHil` (POST `/agent/hil/decisions`); sem token → `enviarResumeHil`
  (POST no `resumeEndpoint` do contrato);
- se a decisão falhar no backend, a pendência **permanece** (dá para tentar de novo);
- emite os eventos `hil-pending`, `hil-decision-sent`, `hil-decision-completed` e
  `hil-decision-failed` para a host reagir (ver seção 21).

Pré-requisito de scripts: `ui-webchat-hil-contract.js` e `hil-review-panel.js` carregados
antes do componente. A checagem é falha-fechada apenas quando uma resposta HIL chega —
hosts sem HIL no fluxo não precisam dos scripts, mas a recomendação é sempre carregá-los.

A host pode decidir programaticamente (por exemplo, num fluxo automatizado de teste):

```javascript
await chat.responderHil('approve');            // aprovar
await chat.responderHil('reject');             // rejeitar
await chat.responderHil('edit', edicoes);      // aprovar com argumentos editados
```

## 18.3 Cancelamento de envio

`cancelar()` (alias `cancel()`) aborta o envio enquanto o estado global ainda é `sending`.
No ramo clássico, o `AbortController` é repassado à fonte única. Desde a decisão "Slice A"
(seção 16) o componente não faz polling, então não existe um job assíncrono separado para
cancelar.

O componente materializa o estado "cancelado" na conversa (não vira erro genérico), emite
o evento `send-cancelled` e fica pronto para o próximo envio. No SSE, depois que o primeiro
fragmento muda o estado para `streaming`, a API pública atual não garante o cancelamento da
conexão inteira. Não prometa cancelamento durante todo o stream sem evolução e teste do
contrato.

O mesmo recorte vale para reentrada: o guard de `enviarPergunta` e o composer desabilitado
checando `sending` não serializam uma segunda rodada iniciada durante `streaming`. Hosts não
devem disparar `perguntar(...)` enquanto o estado público estiver em qualquer uma dessas
duas fases.

## 18.4 Hidratação de conversa (restaurar histórico e mensagens externas)

Duas APIs permitem à host injetar conteúdo sem existir caminho de render paralelo — tudo
passa pela mesma normalização interna de mensagem do componente:

- `restaurarConversa(messages)` — substitui a conversa inteira por uma lista de mensagens
  `{role, content, correlationId?, hil?}`. Uso típico: a v3 restaura conversas salvas no
  localStorage (`webchat_history`). Se uma mensagem restaurada tiver pendência HIL, ela é
  re-hidratada de verdade: o painel remonta e o envio volta a ficar bloqueado até a
  decisão. Emite `conversation-restored`.
- `inserirMensagemExterna(item)` — injeta uma única mensagem do assistente vinda de fora
  do fluxo de envio. Uso típico: materializar pendências de background (`hil_pending`,
  `final_result_pending`) vindas do ledger, com `correlationId` e contrato HIL
  normalizado.

```javascript
// exemplo: restaurar uma conversa salva pela host
chat.restaurarConversa([
  { role: 'user', content: 'Qual o faturamento de 2023?' },
  { role: 'assistant', content: 'R$ 1,2 mi', correlationId: '20260610_..-abc' },
]);
```

## 18.5 payloadText: exibir uma coisa, enviar outra

`perguntar(textoExibido, { payloadText })` mostra `textoExibido` na bolha do usuário e
envia `payloadText` ao backend. Serve para hosts que enriquecem a pergunta com contexto
que não deve poluir a conversa (caso DNIT: contexto de projeto e arquivos anexado à
pergunta). O histórico e os eventos carregam **os dois** textos, para auditoria. Sem a
opção, o comportamento é o padrão (envia o que exibe).

## 18.5A buildPayloadText: bind permanente de campos da tela

Enquanto o `payloadText` de §18.5 é passado por chamada (você enriquece manualmente cada
invocação de `perguntar()`), o hook `buildPayloadText` é um bind **permanente** registrado
uma única vez na criação do componente. A partir daí, todo envio feito pelo input embutido
(botão Enviar, tecla Enter) passa automaticamente pelo hook antes de sair.

**Quando usar:** quando a tela host tem contexto dinâmico que deve acompanhar toda pergunta
digitada pelo usuário — projeto aberto, filtros ativos, arquivos selecionados, período de
análise. O usuário não precisa repassar esse contexto manualmente; ele está sempre presente.

**Contrato:**

```javascript
const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
  // ... outras configs ...
  buildPayloadText: (perguntaDigitada) => {
    const contexto = obterContextoAtualDaTela(); // função da SUA página host
    if (!contexto) return perguntaDigitada;       // sem contexto: envia normal
    return contexto + '\n\nPergunta do usuário:\n' + perguntaDigitada;
  },
});
```

- A função recebe o texto digitado no input.
- Deve devolver uma string: o texto que será enviado ao backend.
- A bolha do chat exibe o texto original digitado (`perguntaDigitada`), não o enriquecido.
- O log interno (rastreável pelo `correlation_id`) carrega os dois, para auditoria.
- Se o hook lançar exceção, o componente captura silenciosamente e envia o texto original — o fallback garante que o envio não quebra.

**Precedência:** se `perguntar()` for chamado com `{ payloadText }` explícito (§18.5), esse
valor tem precedência sobre o hook. O hook só é acionado quando não há `payloadText` por
chamada.

**Exemplo real: tela de projetos DNIT**
(`app/ui/static/js/gesdoc-project-detail.js`, linhas ~1483 e ~1608-1614)

A tela registra o hook na criação do componente:
```javascript
buildPayloadText: (pergunta) => this.chatComporPayloadText(pergunta),
```

E a função de composição monta o payload enriquecido com o contexto da sessão:
```javascript
chatComporPayloadText(pergunta) {
  const contexto = (this._chatContextoSessao || '').trim();
  if (!contexto) return pergunta;
  return contexto + '\n\nPergunta do usuário:\n' + pergunta;
}
```

O `_chatContextoSessao` é atualizado pela tela à medida que o usuário abre projetos e
seleciona arquivos — o componente não conhece esse dado, só chama o hook a cada envio.

`gesdoc-project-detail.js` (host da tela `ui-dnit-project-detail.html`) monta o componente
oficial normalmente (`embeddableChatRuntime.createGenericEmbeddableChat`, mesma classe usada
pela v3) — não existe mais runtime paralelo aqui. O antigo
`dnit-project-chat-runtime.js` foi removido do repositório; testes de contrato
(`tests/frontend/ui_gesdoc_project_detail_runtime_contract.test.js`) protegem contra a volta
dele.

## 18.6 messageActions: ações da host por mensagem

A opção `messageActions` (lista passada na criação do componente) adiciona botões de ação
nas bolhas de mensagem, executados pela host. É como a v3 liga "copiar", "análise de log"
e "download do log" por mensagem sem tocar no render do componente. Cada ação exige
`label` (texto do botão) e `onSelect` (função que recebe uma CÓPIA da mensagem — a host
nunca muta o estado interno); o predicado opcional `quando(mensagem)` filtra em quais
mensagens o botão aparece; `title` (opcional) vira o atributo `title` HTML do botão
(tooltip ao passar o mouse):

```javascript
const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
  // ...config...
  messageActions: [
    {
      label: '📜',
      title: 'Analisar log',
      quando: (mensagem) => Boolean(mensagem.correlationId),
      onSelect: (mensagem) => abrirAnaliseDeLog(mensagem.correlationId),
    },
  ],
});
```

## 19. Estado exportado

O estado público deve incluir, no mínimo:

- input
- messages
- lastInteraction
- lastResponse
- lastError
- status
- statusMessage
- correlationId
- ready
- mounted
- config

Significado prático:

- input: texto atual no campo de digitação;
- messages: histórico renderizado pelo componente;
- lastInteraction: última pergunta e última resposta associadas;
- lastResponse: último payload bruto útil recebido do backend;
- lastError: erro normalizado, quando existir;
- status: estado técnico do runtime;
- statusMessage: mensagem pronta para a host page exibir;
- correlationId: correlation_id oficial recebido do backend;
- ready: indica se o componente tem contexto mínimo para envio;
- mounted: indica se o componente está montado no DOM;
- config: snapshot seguro da configuração aplicada.

## 20. Status esperados

Status técnicos esperados na v1:

- idle
- ready
- sending
- streaming
- error

`completed` e `cancelled` podem aparecer como resultado/status de mensagem ou interação;
não são novos estados globais do componente.

## 21. Eventos emitidos

O namespace dos eventos do componente é:

- prometeu-embeddable-chat:*

Eventos mínimos emitidos:

- prometeu-embeddable-chat:state-change
- prometeu-embeddable-chat:question-sent
- prometeu-embeddable-chat:response-received
- prometeu-embeddable-chat:error
- prometeu-embeddable-chat:history-cleared

Eventos do HIL (seção 18.2):

- prometeu-embeddable-chat:hil-pending — pendência de aprovação detectada (envio bloqueado)
- prometeu-embeddable-chat:hil-decision-sent — decisão despachada à fonte única
- prometeu-embeddable-chat:hil-decision-completed — decisão aceita pelo backend
- prometeu-embeddable-chat:hil-decision-failed — decisão falhou (pendência mantida)

Eventos de cancelamento e hidratação (seções 18.3 e 18.4):

- prometeu-embeddable-chat:send-cancelled — envio abortado pelo usuário/host
- prometeu-embeddable-chat:conversation-restored — conversa re-hidratada via restaurarConversa

Cada evento deve carregar detail com dados serializáveis do runtime.

Exemplo:

```javascript
container.addEventListener('prometeu-embeddable-chat:state-change', (event) => {
  const state = event.detail;
  console.log('Novo estado do chat:', state);
});
```

## 22. Histórico da sessão

O componente deve manter histórico da sessão em memória.

Cada interação deve preservar, no mínimo:

- pergunta do usuário;
- resposta do assistente, quando existir;
- erro, quando existir;
- status;
- horário de envio;
- horário de resposta;
- correlation_id, quando existir;
- payload bruto útil da resposta, quando existir.

A v1 não precisa persistir histórico em banco, Redis ou localStorage.

Persistência externa pode ser feita futuramente pela host page usando eventos e estado exportado.

## 22.1 Sessões de conversa (várias conversas salvas) — headless

O componente entende o conceito de **sessão de conversa** (cada "thread" salva, como as
conversas separadas na barra lateral do ChatGPT), mas continua **headless**: ele **não grava
nada** e **não conhece a lista de sessões nem os títulos**. Ele sabe apenas qual é a **sessão
ATIVA** e **avisa a tela host** quando a conversa muda, para o host gravar onde quiser.

**Quem faz o quê (101):**

- **Componente:** guarda a sessão ativa (id + mensagens em memória), expõe a API de sessão e
  dispara o aviso de persistência. Nada de localStorage/banco aqui.
- **Host (a tela):** é dono da **lista** de conversas, da **interface** (sidebar/seletor), do
  **título**, e da **gravação**. Renomear e excluir são do host.

**API de sessão do componente:**

- `definirSessaoAtiva(id)` / `setActiveSessionId(id)` e `obterSessaoAtiva()` / `getActiveSessionId()`.
- `carregarSessao({ sessionId, messages })` / `loadSession(...)`: o host escolheu uma conversa e
  injeta as mensagens dela; o componente define a sessão ativa e re-hidrata num passo. **Não**
  dispara o aviso de persistência (o host acabou de fornecer esses dados).
- `novaSessao(id)` / `newSession(id)`: inicia uma conversa nova (limpa e zera/define a sessão ativa).
- `obterEstadoAtual().sessionId` traz a sessão ativa; **todo evento** carrega `sessionId`.
- Eventos novos: `session-loaded`, `session-started`.

**O aviso de persistência (callback do host):**

```js
const chat = EmbeddableChatRuntime.createGenericEmbeddableChat({
  /* ...deps, yaml, email... */
  onConversationChanged: ({ sessionId, messages, lastInteraction, reason }) => {
    // o HOST grava aqui (ex.: localStorage). reason: 'response-received' |
    // 'send-cancelled' | 'error' | 'hil-decision-completed' | 'external-message' |
    // 'history-cleared'. Uma exceção neste hook NÃO derruba o componente.
  },
});
```

**O store pronto do lado HOST: `PrometeuChatSessionStore`**

Para não reimplementar o CRUD de sessões em cada tela, use o helper compartilhado
`window.PrometeuChatSessionStore` (carregue `shared/chat-session-store.js` **antes** do script
do host). Ele expõe **dois backends sob o mesmo contrato de instância**
(`create({ mode })` — comprovado em `chat-session-store.js`, funções `createStore` e
`createBackendStore`):

- **`mode: 'local'` (default):** CRUD em `localStorage`, por namespace (`storageKey` +
  `scopeRef` opcional). Síncrono.

  ```js
  const store = PrometeuChatSessionStore.create({
    storageKey: 'webchat_admin_history', // namespace no localStorage
    scopeRef: projectId,                  // opcional: isola conversas por escopo (ex.: projeto)
  });
  // listar() / obter(id) / obterMensagensComponente(id) / criar({title,mode})
  // salvarConversa(id, messages, meta) / renomear(id, title) / excluir(id)
  ```

- **`mode: 'backend'`:** CRUD nas tabelas físicas via `GET/PATCH/DELETE /chat/conversations`
  (`src/api/routers/chat_conversations_router.py`), usando o cliente HTTP oficial
  `PrometeuAdminApiClient`. Todos os métodos devolvem `Promise` (assíncrono). `userEmail` e
  `apiKey` aceitam string fixa **ou** função getter (para hosts com valor vivo que muda em
  runtime); campo crítico vazio no momento da chamada falha fechado. `salvarConversa()` é
  **no-op** nesse modo — a persistência do turno é feita pelo próprio backend, de forma lazy,
  no primeiro envio que carregar `conversation_id`.

  ```js
  const store = PrometeuChatSessionStore.create({
    mode: 'backend',
    userEmail: () => this.userEmail || '',
    apiKey: () => this._resolverApiKeyHost(),
    scopeRef: projectId, // opcional: filtra a lista por bucket (ex.: projeto DNIT)
  });
  ```

Padrão de wiring no host: registrar `onConversationChanged` → `store.salvarConversa(...)`;
ao escolher uma conversa → `chat.carregarSessao({ sessionId, messages: store.obterMensagensComponente(id) })`;
"nova conversa" → `chat.novaSessao(null)`; renomear/excluir → `store.renomear/excluir` + re-render.
**Ordem de init importa:** monte o componente (que cria/segura o store) **antes** de ler o
histórico — inverter quebra o load.

**Estado de adoção confirmado nos hosts atuais:**

- **WebChat v3** (`ui-webchat-v3.js`, método `_montarChatEmbutivel`) — `mode: 'backend'`, sem
  `scopeRef` (bucket geral). As conversas vêm do banco via `/chat/conversations`, não mais de
  `localStorage`.
- **Detalhe de projeto DNIT** (`gesdoc-project-detail.js`, método `chatInicializarSessoes`) —
  `mode: 'backend'`, `scopeRef: projectId` (cada projeto só vê as próprias conversas).
A persistência em **banco** deixou de ser backlog: já existe nos dois hosts listados. O
endpoint físico é `src/api/routers/chat_conversations_router.py` (`GET`/`PATCH`/
`DELETE /chat/conversations`); o `POST` de "nova conversa" não existe como chamada de rede —
é um gesto local do front (gera o id, o backend persiste o turno lazy no primeiro envio com
`conversation_id`).

## 23. Diretriz obrigatória: componente testável isoladamente

O componente embutível deve ser testável sozinho, fora da página host final.

Essa é uma diretriz obrigatória da v1.

O objetivo é separar dois problemas diferentes:

1. validar se o componente funciona corretamente por conta própria;
2. validar depois se uma tela host específica conseguiu embutir o componente corretamente.

A implementação não deve depender exclusivamente de uma página host de produto para provar
que o componente funciona.

Deve existir uma página de teste isolada, simples e dedicada ao componente, capaz de montar o PrometeuEmbeddableChatRuntime diretamente em um container controlado.

Essa página de teste deve permitir validar:

- carregamento das dependências obrigatórias;
- criação da instância do componente;
- montagem do componente no DOM;
- injeção de yamlContent ou encryptedPayload;
- injeção de userEmail;
- injeção de apiKey;
- habilitação do envio quando o contexto mínimo estiver presente;
- preenchimento manual de pergunta;
- envio de pergunta para a API;
- renderização da mensagem do usuário;
- renderização da resposta do assistente ou erro útil do backend;
- preservação de correlation_id;
- atualização do histórico;
- leitura de obterEstadoAtual();
- leitura de obterHistorico();
- leitura de obterUltimaInteracao();
- limpeza do histórico;
- preenchimento externo com preencherPergunta(texto);
- envio programático com perguntar(texto);
- comportamento básico de redimensionamento.

Existe teste Playwright para a página isolada. Na cobertura atual, ele valida o ramo
clássico com a chamada interceptada; não deve ser apresentado como prova API-live do SSE.
O contrato incremental SSE é protegido por testes frontend focados do transporte e do
componente.

O teste E2E deve abrir a página de teste do componente, aplicar uma configuração válida, enviar uma pergunta e validar o comportamento real do componente.

O teste deve validar, no mínimo:

- o componente foi montado;
- o componente ficou pronto para envio;
- a pergunta enviada apareceu na conversa;
- a resposta ou erro útil do backend apareceu na conversa;
- o estado exportado foi atualizado;
- o histórico interno recebeu a interação;
- o correlation_id foi preservado quando retornado pelo backend.

Cada tela host real também pode ter testes próprios, mas ela não deve ser o único lugar
onde o componente é validado.

A regra é:

primeiro o componente precisa funcionar sozinho;
depois ele deve funcionar embutido em uma tela host real.

Isso reduz o risco de misturar problema do componente com problema de layout, contexto, shell, YAML, autenticação ou integração específica da host page.

## 24. Redimensionamento e encaixe no host

O componente deve ocupar 100% da largura e da altura do container onde for montado.

Isso significa:

- o componente não decide sozinho o tamanho da página;
- quem define o espaço é a host page;
- o componente se adapta ao container recebido;
- a área de mensagens deve usar scroll interno;
- o campo de digitação deve permanecer acessível;
- em telas menores, o composer pode quebrar para coluna automaticamente;
- o componente não deve estourar o layout da host page.

Regra prática para quem embute:

<div id="meu-chat-host" style="width: 100%; height: 100%; min-height: 480px;"></div>

Se a host não reservar altura útil, o chat não terá onde crescer.

## 25. Exemplo mínimo de host page

HTML:

```html
<div id="chat-host" style="width:100%;height:480px"></div>
<div id="chat-summary"></div>

<script src="/ui/static/js/shared/layout-mestre-api.js"></script>
<script src="/ui/static/js/shared/ui-webchat-runtime-utils.js"></script>
<script src="/ui/static/js/shared/embeddable-chat-ag-ui-transport.js"></script>
<script type="module" src="/ui/static/js/shared/ag-ui-embeddable-transport-bridge.js"></script>
<script src="/ui/static/js/shared/embeddable-chat-runtime.js"></script>
<script src="/ui/static/js/minha-host-chat.js"></script>
```

JavaScript da host, mantido em arquivo externo:

```javascript
document.addEventListener('DOMContentLoaded', () => {
  const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
    yamlContent: window.minhaTela.yamlContent,
    yamlFilename: 'config.yaml',
    userEmail: window.minhaTela.userEmail,
    apiKey: window.minhaTela.apiKey,
    mode: 'qa',
    chatRenderer: 'jspuro',
    agUiSseTransport: true,
    onChange(state) {
      document.getElementById('chat-summary').textContent = state.statusMessage || '';
    }
  });

  chat.mount(document.getElementById('chat-host'));
});
```

Veja explicação passo a passo, exemplos de DeepAgent/HIL, `projectKey` e fallback clássico
no [Tutorial 101 de chat com streaming](TUTORIAL-CHAT-PLATAFORMA.md).

## 26. Exemplo com host reagindo ao estado exportado

```javascript
const host = document.getElementById('chat-host');
const correlation = document.getElementById('correlation-label');
const payloadViewer = document.getElementById('payload-viewer');

const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
  yamlContent: yamlText,
  userEmail: currentUserEmail,
  apiKey: currentApiKey,
  mode: 'agent',
  onChange(state) {
    correlation.textContent = state.correlationId || 'aguardando backend';
    payloadViewer.textContent = state.lastResponse
      ? JSON.stringify(state.lastResponse, null, 2)
      : 'sem resposta ainda';
  }
});

chat.mount(host);
```

## 27. Exemplo usando eventos DOM

```javascript
const host = document.getElementById('chat-host');

const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
  yamlContent: yamlText,
  userEmail: currentUserEmail,
  apiKey: currentApiKey,
  mode: 'workflow'
});

const root = chat.mount(host);

root.addEventListener('prometeu-embeddable-chat:state-change', (event) => {
  const state = event.detail;
  console.log('Novo estado do chat:', state);
});

root.addEventListener('prometeu-embeddable-chat:error', (event) => {
  console.error('Falha visível do componente:', event.detail);
});
```

## 28. Exemplo de integração com host que recebe YAML externamente

```javascript
function syncHostIntoComponent(snapshot) {
  chat.definirConfiguracao({
    yamlContent: snapshot.yamlContent,
    yamlFilename: snapshot.yamlFilename || 'config.yaml',
    userEmail: snapshot.userEmail,
    apiKey: snapshot.apiKey,
    mode: modeSelect.value,
    chatRenderer: 'jspuro',
    agUiSseTransport: true,
    metadataFilters: parseMetadataFilters(),
  });
}
```

O exemplo escolhe `yamlContent`. Se a host usar `encryptedPayload` ou `projectKey`, ela
deve omitir as outras fontes; não envie duas configurações concorrentes.

A ideia importante:

- o shell ou a tela host resolve o contexto;
- o componente consome o contexto;
- os dois continuam desacoplados.

## 29. Hosts reais para estudar

Os melhores exemplos atuais são:

- `app/ui/static/ui-webchat-v3.html` + `app/ui/static/js/ui-webchat-v3.js`: host geral;
- `app/ui/static/ui-dnit-project-detail.html` +
  `app/ui/static/js/gesdoc-project-detail.js`: Q&A/RAG com SSE e sessões por projeto;
- `app/ui/static/ui-saas-project.html` + `app/ui/static/js/saas-project-public.js`: host
  que passa `projectKey`, sem YAML nem API key no navegador.

Em todos, procure o mesmo desenho: a host carrega contexto e monta o componente; o
componente é dono do envio e da conversa.

## 30. Validação isolada do componente

Antes de tratar a integração em telas mais complexas, a v1 deve permitir validação isolada do componente.

Deve existir uma tela ou página de teste dedicada, capaz de montar o componente fora de uma host page complexa.

A bancada oficial dessa validação isolada é `app/ui/static/ui-embeddable-chat-test.html` (com `app/ui/static/js/ui-embeddable-chat-test.js`). Ela só configura e observa: monta o componente em um container controlado, resolve as dependências canônicas com falha fechada e observável, injeta contexto por campos simples, expõe botões para todos os métodos públicos e permite redimensionar o container. Ela não monta payload próprio, não faz fetch de execução e não desenha a conversa (a conversa é desenhada pelo componente). O caminho primário da bancada é `encryptedPayload` (objeto stub), para provar o componente sem arrastar a criptografia real.

Essa tela deve permitir validar:

- carregamento ou injeção de YAML;
- preenchimento manual de pergunta;
- envio ao boundary do navegador, real ou interceptado conforme o teste;
- montagem correta do payload;
- recebimento e exibição da resposta;
- preservação de correlation_id;
- histórico da sessão;
- consulta da última interação;
- consulta do histórico completo;
- limpeza do histórico;
- preenchimento externo do campo;
- envio programático com perguntar(texto);
- tratamento de erro;
- redimensionamento básico.

O objetivo é separar dois problemas:

1. fazer o componente funcionar sozinho;
2. depois embutir o componente em telas reais.

## 31. Teste E2E

A v1 deve possuir teste automatizado E2E, preferencialmente com Playwright se esse for o padrão já existente no projeto.

O teste E2E oficial é `tests/playwright/test_08-01-10_embeddable_chat_isolated.py` (famílias `browser` + `e2e` + `asyncio`). Ele serve a bancada por HTTP, intercepta de forma determinística o boundary HTTP do navegador (`/crypto/offline-store` e `/<modo>/execute`) e valida o componente em desktop e mobile. Usa o caminho `encryptedPayload` com `executionMode=direct_sync` (sem polling), de modo a provar o componente sem backend vivo nem criptografia real.

O teste deve abrir a tela de teste isolada do componente e validar, no mínimo:

- componente montado;
- contexto mínimo aplicado;
- botão de envio habilitado quando YAML, e-mail e API key estiverem presentes;
- envio de pergunta;
- mensagem do usuário renderizada;
- resposta ou erro útil do backend renderizado;
- correlation_id preservado quando retornado;
- histórico interno atualizado.

O teste valida o componente isolado de verdade, mas com boundary clássico interceptado.
Ele não prova uma sessão SSE contra uma API viva.

## 31.1 Definição de 100% verde: componente embutível **E** backend, ambos verdes

Regra obrigatória do roteiro de teste deste componente (decisão registrada do usuário):

- **"100% verde" só é válido quando o componente embutível E o backend estiverem rodando 100% verdes.** Não basta o componente passar nos testes automáticos e se comportar corretamente na UI; a rodada só é aprovada quando uma pergunta enviada pela UI real **recebe resposta final visível no chat** e o **log da correlação fecha limpo**, sem erro, traceback, fallback indevido nem `correlation_id` ausente.
- **Todo erro de backend encontrado durante o teste faz parte do roteiro de teste deste componente** — não é "fora de escopo". Isso inclui, por exemplo: `500` sem `X-Correlation-Id` na resposta; `logging.contract.violation` / `missing_event_name`; falha de inicialização de supervisor DeepAgent (`CompositeBackend não encontrado`); resolução de alvo vetorial (`VectorTargetResolutionError`) e qualquer exceção, warning bloqueante ou fallback indevido no caminho exercitado.
- Cada erro de backend deve ser tratado pelo **loop de auto-correção por log** (`.claude/rules/loops-estrategicos.md`): capturar o `correlation_id`, abrir o log oficial, provar a causa raiz no código, corrigir na origem, proteger com teste e repetir a rodada pela UI real.
- O teste só pode ser declarado concluído com sucesso quando **ambos** os lados — componente e backend — estiverem verdes no caminho oficial de runtime, com prova por log.

## 32. Regras práticas de uso

### 32.1 Não criar cliente HTTP paralelo

O componente deve usar o caminho canônico do projeto.

Se uma página criar outro cliente HTTP só para facilitar, ela reabre o problema que este componente existe para resolver.

### 32.2 Não esconder erro real do backend

Se o backend devolver erro com mensagem útil, o componente deve mostrar essa mensagem e preservar o correlation_id.

Isso ajuda o usuário e permite rastreabilidade no log.

### 32.3 A host page não deve desenhar a conversa

A host page pode desenhar:

- filtros;
- painéis auxiliares;
- resumo de estado;
- botões externos;
- telemetria;
- histórico externo;
- ações administrativas.

Mas ela não deve competir com o DOM interno do componente para renderizar a conversa principal.

### 32.4 Resize correto depende do container

O componente ocupa o espaço recebido.

A host page precisa fornecer altura útil.

Se o container pai não tiver altura, o componente pode aplicar uma altura mínima de segurança, mas o comportamento ideal depende de um layout bem definido pela host.

### 32.5 Não alterar a página base

A página ui-webchat-v3.html é referência funcional.

Ela não deve ser alterada para fazer o componente funcionar.

Se houver dúvida, consulte a página para copiar o comportamento correto de contrato, não para modificar a referência.

## 33. Critérios de aceite da v1

A v1 está pronta quando:

- o componente monta dentro de um container;
- o componente recebe yamlContent, userEmail e apiKey;
- o componente também aceita encryptedPayload quando esse for o fluxo da host;
- o envio só fica habilitado quando o contexto mínimo estiver presente;
- o componente envia pergunta usando o cliente canônico;
- o payload enviado é compatível com a ui-webchat-v3.html;
- o tratamento de YAML segue o comportamento da ui-webchat-v3.html;
- o tratamento de payload criptografado segue o comportamento da ui-webchat-v3.html;
- a resposta da API é interpretada como na ui-webchat-v3.html;
- a mensagem do usuário aparece na conversa;
- a resposta do assistente aparece na conversa;
- erros úteis do backend aparecem na conversa;
- o correlation_id oficial do backend é preservado;
- messages é atualizado;
- lastInteraction é atualizado;
- lastResponse é atualizado;
- lastError é atualizado quando houver erro;
- limparHistorico() funciona;
- preencherPergunta(texto) funciona;
- perguntar(texto) funciona;
- obterEstadoAtual() retorna snapshot coerente;
- obterHistorico() retorna histórico da sessão;
- obterUltimaInteracao() retorna a última interação;
- o componente funciona em tela de teste isolada;
- existe teste E2E validando o componente isoladamente;
- não existe cliente HTTP paralelo;
- não existe renderização duplicada da conversa fora do componente;
- não existe código legado morto relacionado ao componente;
- não existem duas versões concorrentes do componente.

## 34. FAQ

### O componente pode ser usado fora do shell administrativo?

Sim, desde que a página carregue as dependências canônicas necessárias e entregue a configuração mínima.

Ele não depende da sidebar administrativa nem da página de exemplo.

### Preciso passar YAML sempre em texto puro?

Não.

A host page pode passar yamlContent ou encryptedPayload, dependendo do fluxo.

### O componente decide sozinho qual YAML usar?

Não.

O correto é a host page decidir isso e repassar ao componente.

### Posso usar esse componente para Q&A, Agent, DeepAgent e Workflow?

Sim.

O campo mode existe para isso.

### Posso forçar síncrono ou assíncrono?

Não pelo campo `executionMode`.

Desde a decisão "Slice A" (seção 16), o ramo clássico é sempre `direct_sync`. Para receber
texto progressivo, use o gate explícito `agUiSseTransport: true` +
`chatRenderer: 'jspuro'` com Q&A, DeepAgent ou `projectKey`.

### Existe callback `onToken` para eu montar o texto?

Não. O componente atualiza a própria bolha e publica snapshots por `onChange(state)` ou
`prometeu-embeddable-chat:state-change`. Durante a resposta, observe
`state.status === 'streaming'` e a última mensagem sem recriar o DOM.

### Posso usar `EventSource` para chamar `/ag-ui/runs`?

Não. `EventSource` usa `GET`, enquanto o endpoint oficial exige `POST` com JSON. Carregue o
transporte e a bridge oficiais.

### Q&A também pode usar SSE ou isso é só para A2UI?

Q&A usa SSE com `mode: 'qa'`, `chatRenderer: 'jspuro'` e
`agUiSseTransport: true`. A2UI é uma capacidade visual adicional do DeepAgent, não o nome
do transporte.

### Como pego a última resposta sem ler o DOM?

Use obterEstadoAtual().

O campo lastResponse foi pensado para isso.

### Como limpo a conversa programaticamente?

Use limparHistorico().

### Como coloco uma pergunta por código e envio sem o usuário digitar?

Use perguntar(texto).

### Como coloco uma pergunta no campo sem enviar?

Use preencherPergunta(texto).

### O componente gera correlation_id no browser?

Não.

Ele só preserva e exibe o correlation_id recebido do backend.

### O que acontece se a X-API-Key for inválida?

O backend responde com erro.

O componente deve preservar a mensagem real, preservar o correlation_id quando existir, atualizar lastError e publicar evento de erro.

### Posso adicionar botões externos, filtros, tabs e painéis em volta dele?

Sim.

Esse é o desenho recomendado.

O componente cuida do chat. A host page cuida do resto.

## 35. Checklist de integração de uma nova host page

Antes de considerar uma nova host page pronta, confirme:

- existe um container com altura útil;
- as dependências canônicas foram carregadas;
- o componente foi montado no container correto;
- a host page injeta uma fonte autorizada: yamlContent, encryptedPayload ou projectKey;
- a host page injeta userEmail;
- a host page injeta apiKey somente quando o fluxo autorizado exige;
- a host page define mode;
- a host page decide se habilita o gate SSE; `executionMode` não seleciona transporte;
- se habilitar SSE, carrega transporte e bridge antes do componente;
- a host page observa correlationId ou lastResponse se precisar de rastreabilidade;
- a host page não criou cliente HTTP paralelo;
- a host page não recriou a renderização da conversa;
- a host page não alterou a ui-webchat-v3.html;
- a host page não duplicou lógica de payload;
- a host page não duplicou lógica de tratamento de resposta.

## 35.1 Estado de convergência e itens abertos

Estado da convergência após a migração da Fase B (2026-06-10):

- O componente e a página oficial `ui-webchat-v3.html` usam `layout-mestre-api.js` como
  ponto único do ramo clássico de payload,
  criptografia e HTTP — incluindo o HTTP das decisões HIL (`enviarDecisaoHil`/
  `enviarResumeHil`). O motor próprio da v3 (~2.100 linhas de payload/fetch/criptografia/
  HIL/polling/markdown) foi removido; a v3 é host fino.
- O teste estrutural `webchat_single_source_regression_contract.test.js` protege componente
  e v3: se qualquer um voltar a ter fetch de execução, criptografia ou payload próprio, o
  teste falha (provado por regressão simulada).

A antiga pendência de migração do detalhe de projeto DNIT foi **fechada**: o arquivo
`app/ui/static/js/shared/dnit-project-chat-runtime.js` não existe mais no repositório —
`gesdoc-project-detail.js` monta o componente oficial (ver seção 18.5A), e testes de
contrato (`tests/frontend/ui_gesdoc_project_detail_runtime_contract.test.js`,
`ui_gesdoc_project_detail_layout_contract.test.js`) garantem que o runtime paralelo não
volta.

## 35.2 Comunicação com a API: os dois fluxos reais

Esta seção consolida em um único lugar a sequência HTTP que o componente executa por
baixo. Ela não substitui a montagem de payload (detalhada em
[README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md](../tecnico/README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md)),
mas explica a ordem real das chamadas — útil para quem precisa diagnosticar a rede ou
construir uma interface própria (ver [GUIA-INTEGRADOR-CHAT-PLATAFORMA.md](GUIA-INTEGRADOR-CHAT-PLATAFORMA.md)).

### 35.2.1 Ramo clássico (`direct_sync`)

1. **Handshake criptográfico** — quando há YAML, `PayloadCrypto.buildEncryptedData(...)`
   chama `POST /crypto/session-key` e recebe `{ session_id, public_key_pem, expires_at,
   ttl_seconds }`. A chave pública protege a chave Fernet usada para cifrar o YAML.
2. **Cifra do YAML** — o helper gera o envelope `encrypted_data` com `session_id`,
   `wrapped_key`, `encrypted_yaml`, `original_filename`, `encryption_scheme` e
   `yaml_operational_contract`. Quando o usuário carrega um payload já cifrado, esse
   passo é pulado (o componente usa o objeto direto e, no chat, **não** registra em
   `/crypto/offline-store`).
3. **Envio por modo** — `POST` para o endpoint do modo selecionado:
   - `qa` → `/rag/execute` (envelope `{ operation: "ask", payload: { ... } }`);
   - `agent`/`deepagent` → `/agent/execute` (campos no topo do corpo; `deepagent` força
     `mode: "deepagent"`);
   - `workflow` → `/workflow/execute`.
   Headers reais: `Content-Type: application/json`, `X-API-Key: <chave>` (ou a chave em
   `authentication.access_key` no YAML), `X-Prometeu-UI-Session-Required: 1`, e
   `credentials: 'include'`.
4. **Leitura do correlation_id** — o componente lê `X-Correlation-Id` (header) ou os
   campos de correlação do corpo, exibe na barra de status e propaga. Nunca cria esse
   identificador localmente.
5. **Sem polling (Slice A)** — o componente sempre manda `execution_mode: "direct_sync"`
   (seção 16) e lê o resultado final direto de `result.dados` da própria resposta HTTP; não
   há passo 5 de acompanhamento assíncrono. `layout-mestre-api.js` ainda sabe interpretar um
   eventual `HTTP 202` (função `_extrairInfoAssincrona`, usada por outros consumidores da
   fonte única), mas o componente embutível não aciona polling algum a partir disso — o
   `WebchatAsyncRuntime.waitForTaskCompletion` que fazia esse papel não é mais chamado por
   ele (ver seção 16).

### 35.2.2 Ramo SSE (`POST /ag-ui/runs`)

Quando o gate opt-in da seção 16 está satisfeito, o componente não passa pelos endpoints
clássicos acima. Ele:

1. monta `thread_id`, `run_id`, `user_email`, `input` e exatamente uma fonte de
   configuração;
2. mapeia `qa` para `execution_kind: "rag_qa"`, mantém `deepagent`, ou envia somente
   `project_key` para o servidor resolver a execução autorizada;
3. faz `POST /ag-ui/runs` com `Accept: text/event-stream` pelo cliente AG-UI oficial;
4. captura `X-Correlation-Id` sem inventar o identificador;
5. acumula os eventos de texto em uma única mensagem provisória e a substitui pela
   mensagem final;
6. preserva `sources` do Q&A, envelope A2UI e pendência HIL quando presentes.

O host observa esse processo por `onChange`; não cria `fetch`, parser SSE ou bolhas
paralelas. O exemplo copiável está no
[Tutorial 101 de chat com streaming](TUTORIAL-CHAT-PLATAFORMA.md).

![35.2 Comunicação com a API: o fluxo HTTP real, ponta a ponta](../assets/diagrams/docs-guia-componente-webchat-embutivel-diagrama-01.svg)

## 36. Resumo final

Se precisar lembrar de uma regra só, lembre desta:

a tela host configura e observa; o componente renderiza e conversa.

A página ui-webchat-v3.html é hoje o exemplo real de host completo do componente — para
ver "como integrar", leia o `ui-webchat-v3.js`.

Para dúvidas do ramo clássico, a fonte de verdade é `layout-mestre-api.js`. Para o SSE,
consulte o transporte, a bridge e a seção 35.2 deste guia.

O componente embutível é o único motor de conversa das telas migradas: sem lógica
duplicada, sem cliente paralelo e sem legado morto para trás.
