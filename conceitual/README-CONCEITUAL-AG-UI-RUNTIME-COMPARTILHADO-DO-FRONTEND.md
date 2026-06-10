# Manual detalhado da etapa: Runtime compartilhado do frontend AG-UI

## 1. O que esta etapa faz

Esta etapa entrega o runtime reutilizavel de frontend do slice AG-UI. Ela existe para que paginas estaticas em JavaScript puro consumam o protocolo de forma consistente, sem reimplementar cliente SSE, store de estado, timeline de tools, sidecar e ponte de HIL toda vez.

Em linguagem simples: e o kit interno que transforma eventos AG-UI em experiencia de tela.

## 2. Onde ela entra no fluxo

No codigo lido, essa camada aparece na fachada packages/ag-ui-runtime/index.js e nos modulos compartilhados dentro de app/ui/static/js/shared. E ela que recebe o stream vindo do endpoint AG-UI informado pela pagina, normalmente /ag-ui/runs nas integracoes novas, reconstrui o estado e renderiza a experiencia.

## 3. O que entra e o que sai

Entradas confirmadas:

- endpoint AG-UI e payload do run
- stream SSE de eventos AG-UI
- correlation_id devolvido pelo backend no header
- contrato HIL global da pagina quando ha resume
- reviewPanelFactory opcional para painel compartilhado de aprovacao

Saidas confirmadas:

- cliente interno para POST SSE
- store com run, messages, tools, state, activities, steps, interrupts e raw events
- sidecar de chat reutilizavel para paginas estaticas
- payload de resume AG-UI montado a partir do contrato HIL comum

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. a fachada interna reexporta cliente, state store, sidecar, renderers e validadores canônicos;
2. createAgUiSseClient exige endpoint explicito e executa fetch POST com Accept text/event-stream;
3. o cliente extrai X-Correlation-Id dos headers e repassa ao consumidor;
4. no pacote @prometeu/ag-ui-runtime, a leitura do stream fica sobre o SDK oficial @ag-ui/client; nas paginas estaticas, essa leitura chega ao browser por um bundle oficial versionado em /ui/static/js/vendor/prometeu-ag-ui-runtime.browser.js, e `app/ui/static/js/shared/ag-ui-client.js` virou apenas uma fachada fina de compatibilidade de importacao;
5. createAgUiStateStore parte de um estado inicial com run, messages, tools, state, activities, steps, interrupts e eventos brutos;
6. o store aplica snapshots e JSON Patch para manter estado incremental consistente;
7. createAgUiSidecarChat combina client, store e renderers para montar uma experiencia reutilizavel;
8. interrupcoes AG-UI sao adaptadas ao contrato do painel HIL compartilhado;
9. o runtime usa HilContract global para construir o payload oficial de resume.

O detalhe mais importante e que tudo isso foi implementado em JavaScript puro. O slice nao depende de React para entregar um runtime AG-UI reaproveitavel.

## 5. Decisoes tecnicas importantes

### 5.1. O stream nasce por POST, nao por GET EventSource simples

O cliente usa fetch porque o run AG-UI precisa enviar payload completo no corpo do request. No pacote runtime, essa leitura passa pelo SDK oficial. Como o backend serve apenas `app/ui/static`, o repositório agora versiona um bundle oficial de browser para resolver `@ag-ui/client` e `rxjs` sem expor `node_modules` publicamente.

### 5.2. O browser nao inventa correlation_id

O cliente so extrai o header do backend. Isso preserva a regra de origem do correlation_id na borda servidora.

### 5.3. HIL visual e compartilhado entre superficies

O sidecar adapta a interrupcao AG-UI ao contrato do painel HIL comum. Na pratica, a camada visual de aprovacao e reutilizavel mesmo fora de uma unica pagina especifica.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- ambiente sem fetch nao consegue consumir o cliente;
- payload de resume falha se HilContract global nao estiver carregado;
- frontends que ignorem JSON Patch perdem coerencia de estado incremental;
- paginas que reimplementem parsing SSE fora do runtime oficial podem divergir do comportamento canônico;
- deixar o bundle versionado fora de sincronismo com a fonte quebraria as paginas HTML mesmo sem alterar o backend;
- como o runtime e interno, tratar o pacote como distribuicao publica seria conclusao indevida.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- correlation_id exibido no sidecar;
- estado run.status, isRunning e outcome dentro do store;
- fila de interrupts no state store;
- erro de leitura SSE ou falha HTTP reportada pelo client;
- excecao dizendo que HilContract global esta indisponivel.

Em linguagem simples: se o backend esta emitindo bem, mas a experiencia na tela esta incoerente, o problema tende a estar aqui.

## 8. Exemplo pratico guiado

Cenario: uma tela administrativa quer embutir um sidecar AG-UI sem mudar a stack para React.

1. importa-se a fachada interna do runtime;
2. monta-se o payload de run para /ag-ui/runs;
3. o cliente abre o POST SSE e captura o X-Correlation-Id;
4. o store reconstrui mensagens, tools e estado incrementalmente;
5. o sidecar exibe mensagens, timeline e interrupcoes;
6. se houver HIL, o painel compartilhado gera o resume canônico.

O valor desta etapa e industrializar o frontend AG-UI no contexto real do projeto, que hoje e HTML estatico mais JavaScript puro.

## 8.1. Spec-runtime e bridge do componente de chat embutivel

Alem do sidecar que consome o stream `/ag-ui/runs`, existe uma segunda forma de o frontend consumir AG-UI: **renderizar specs que chegam no corpo da resposta** dos endpoints de chat ja usados pelo componente global de chat embutivel. Vale explicar porque sao duas superficies diferentes do mesmo conceito.

- O **sidecar** (secoes anteriores) abre um stream SSE de eventos AG-UI e reconstroi estado incrementalmente. E o caminho de streaming.
- O **componente de chat embutivel** nao abre stream. Ele recebe a resposta normalizada do backend e **detecta** se ha um spec AG-UI conhecido nela. Se houver, desenha a UI; se nao, mostra texto. E o caminho de spec-na-resposta.

Para isso o componente reusa uma camada compartilhada de **deteccao + registry de renderizadores**, que existe justamente para nao espalhar essa logica por cada tela:

- um **spec-runtime** detecta o spec na resposta e roteia para o renderizador certo. Ele **nao reimplementa** os renderizadores oficiais de DashboardSpec e UISpec — recebe-os por injecao. O unico renderizador novo e o de Capacidades (CapabilitiesSpec), montado com primitivas DOM seguras;
- um **bridge** em modulo ES e o ponto que `import`a os renderizadores oficiais (que sao ES modules) e os entrega ao spec-runtime (que e UMD), publicando o runtime montado em `window` para o componente consumir. Esse padrao de ponte ESM↔UMD ja e usado em outras partes do projeto, e existe porque o componente, sendo UMD, nao pode importar ES modules diretamente.

A consequencia de reuso e a mesma do restante deste runtime compartilhado: a deteccao, a validacao de seguranca e a delegacao aos renderizadores oficiais ficam **num lugar so**, e qualquer host que carregue o bridge ganha a renderizacao estruturada sem reimplementar nada. Se o bridge nao for carregado, o componente nao encontra o runtime em `window` e renderiza texto — fail-closed identico ao do sidecar quando uma dependencia falta. O detalhe de ativacao, ordem de scripts e estado por host esta no [guia do componente embutivel](../usuario/GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md).

## 9. Evidencias no codigo

- packages/ag-ui-runtime/index.js
  - Simbolo relevante: fachada publica interna do runtime
  - Comportamento confirmado: reexportacao canônica dos modulos compartilhados.
- app/ui/static/js/shared/ag-ui-client.js
  - Simbolo relevante: createAgUiSseClient
  - Comportamento confirmado: fachada fina que reexporta o cliente oficial empacotado para browser, mantendo POST SSE com captura de X-Correlation-Id e reconexao opcional.
- app/ui/static/js/vendor/prometeu-ag-ui-runtime.browser.js
  - Simbolo relevante: bundle oficial versionado do runtime AG-UI para browser.
  - Comportamento confirmado: entrega ao browser o cliente oficial e helpers necessarios sem imports bare de `node_modules`.
- app/ui/static/js/shared/ag-ui-state-store.js
  - Simbolo relevante: createInitialState e aplicacao de JSON Patch
  - Comportamento confirmado: reconstrucao do estado AG-UI no frontend.
- app/ui/static/js/shared/ag-ui-sidecar-chat.js
  - Simbolo relevante: adaptInterruptToReviewContract e buildAgUiResumeInput
  - Comportamento confirmado: adaptacao de interrupcao AG-UI ao painel HIL compartilhado, exigencia de endpoint explicito e montagem do resume oficial.
- app/ui/static/js/shared/embeddable-chat-spec-runtime.js
  - Simbolo relevante: createEmbeddableChatSpecRuntime, detectAgUiSpec, validateCapabilitiesSpecPayload
  - Comportamento confirmado: detecta spec AG-UI na resposta, delega DashboardSpec/UISpec aos renderizadores oficiais injetados e renderiza o CapabilitiesSpec com DOM seguro; spec invalido/desconhecido cai em texto.
- app/ui/static/js/shared/ag-ui-spec-render-bridge.js
  - Simbolo relevante: bridge ESM que monta e publica o spec-runtime em window
  - Comportamento confirmado: importa os renderizadores oficiais ES module e os entrega por injecao ao spec-runtime UMD, publicando window.PrometeuEmbeddableChatSpecRuntime.
