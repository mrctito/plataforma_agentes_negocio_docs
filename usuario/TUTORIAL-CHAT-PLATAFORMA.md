# Tutorial 101: chat da plataforma com resposta em streaming

Este tutorial ensina a embutir o componente global `PrometeuEmbeddableChatRuntime` em uma tela web, enviar uma mensagem para a API da plataforma e exibir a resposta enquanto ela chega.

Ao terminar, voce sabera:

- quando usar streaming SSE e quando manter o fluxo classico;
- como carregar os scripts na ordem correta;
- como montar um chat Q&A/RAG com uma unica bolha crescendo durante a resposta;
- como usar DeepAgent com Human-in-the-Loop (HIL);
- como usar `projectKey` sem entregar YAML ou API key ao navegador;
- como acompanhar estado, fontes e `correlation_id` sem criar um protocolo paralelo;
- como diagnosticar os erros mais comuns.

> **Regra principal:** se a tela esta dentro da plataforma e ja possui o YAML ou o `projectKey`, use o componente global. Nao monte `fetch`, parser SSE, payload, criptografia ou bolhas de mensagem por conta propria.

## 1. O que significa streaming neste chat

No fluxo com SSE, a tela faz um `POST /ag-ui/runs`. A conexao HTTP permanece aberta e a API envia eventos AG-UI progressivamente.

O componente global cuida do trabalho dificil:

1. monta o payload oficial;
2. abre o `POST` com `Accept: text/event-stream`;
3. interpreta os eventos AG-UI;
4. cria uma unica mensagem provisoria do assistente;
5. atualiza essa mesma mensagem conforme novos fragmentos chegam;
6. substitui a mensagem provisoria pela mensagem final;
7. preserva fontes, HIL, A2UI, `thread_id` e o `correlation_id` oficial.

Portanto, streaming nao significa criar uma bolha para cada token. A experiencia correta e uma unica bolha crescendo.

### 1.1. Streaming SSE e fluxo classico nao sao a mesma coisa

| Necessidade | Configuracao | Transporte usado |
|---|---|---|
| Q&A/RAG com texto progressivo | `mode: "qa"`, `chatRenderer: "jspuro"` e `agUiSseTransport: true` | `POST /ag-ui/runs`, SSE |
| DeepAgent com texto progressivo e HIL | `mode: "deepagent"`, `chatRenderer: "jspuro"` e `agUiSseTransport: true` | `POST /ag-ui/runs`, SSE |
| Projeto autorizado pelo servidor | `projectKey` e `agUiSseTransport: true` | `POST /ag-ui/runs`, SSE |
| Streaming desativado ou modo nao suportado | flag ausente/falsa, outro renderer ou modo como `workflow` | endpoints classicos, resposta completa |

O componente falha fechado. Ativar apenas `agUiSseTransport: true` nao obriga qualquer modo a usar SSE. O renderer tambem precisa ser `jspuro`, e o fluxo precisa ser Q&A, DeepAgent ou baseado em `projectKey`.

## 2. Antes de comecar

Voce precisa de:

- uma pagina servida pela aplicacao;
- um elemento HTML que recebera o chat;
- o e-mail real do usuario autenticado;
- exatamente uma fonte de configuracao autorizada:
  - `yamlContent` ja carregado pelo host; ou
  - `encryptedPayload`; ou
  - `projectKey` resolvido pelo servidor;
- API key somente quando o fluxo autorizado realmente exigir e o host ja a possuir.

Nunca:

- exponha YAML, token ou API key no console;
- grave segredo em HTML;
- invente `correlation_id`;
- envie `projectKey` junto com YAML explicito;
- use `EventSource`, pois ele faz `GET` e o endpoint oficial exige `POST` com JSON;
- concatene tokens manualmente no DOM.

## 3. Carregue os scripts na ordem correta

Exemplo de HTML do host:

```html
<div id="chat-container" aria-label="Conversa com o assistente"></div>

<!-- Opcionais quando o host recebe YAML criptografado. -->
<script src="/ui/static/js/plataforma-agentes-ia-crypto.js"></script>
<script src="/ui/static/js/shared/yaml-access-key-extractor.js"></script>

<!-- Fundacao compartilhada do chat. -->
<script src="/ui/static/js/shared/layout-mestre-api.js"></script>
<script src="/ui/static/js/shared/ui-webchat-runtime-utils.js"></script>

<!-- Necessarios quando a tela usa DeepAgent com HIL. -->
<script src="/ui/static/js/shared/ui-webchat-hil-contract.js"></script>
<script src="/ui/static/js/shared/hil-review-panel.js"></script>

<!-- Transporte SSE oficial e ponte para o componente global. -->
<script src="/ui/static/js/shared/embeddable-chat-ag-ui-transport.js"></script>
<script type="module" src="/ui/static/js/shared/ag-ui-embeddable-transport-bridge.js"></script>

<!-- Componente global. -->
<script src="/ui/static/js/shared/embeddable-chat-runtime.js"></script>

<!-- Codigo especifico da sua tela. Em producao, mantenha-o em arquivo externo. -->
<script src="/ui/static/js/minha-tela-chat.js"></script>
```

O host pode omitir os scripts opcionais que nao usa. Nao altere a ordem relativa entre transporte, bridge e componente.

## 4. Exemplo A: Q&A/RAG com texto em streaming

Este e o exemplo recomendado quando a tela ja carregou um YAML RAG pelo resolvedor oficial da plataforma.

### 4.1. HTML

```html
<main>
  <h1>Assistente de documentos</h1>
  <p id="chat-status" role="status">Preparando o chat...</p>
  <div id="chat-container"></div>
</main>
```

Inclua tambem os scripts da secao anterior.

### 4.2. JavaScript do host

Crie `minha-tela-chat.js`:

```javascript
document.addEventListener("DOMContentLoaded", () => {
  // Valores entregues pelos boundaries oficiais da tela.
  const yamlTextAlreadyLoaded = window.minhaTela.yamlContent;
  const authenticatedUserEmail = window.minhaTela.userEmail;
  const apiKeyFromAuthorizedHost = window.minhaTela.apiKey;
  const statusElement = document.getElementById("chat-status");

  const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
    yamlContent: yamlTextAlreadyLoaded,
    yamlFilename: "configuracao-rag.yaml",
    userEmail: authenticatedUserEmail,
    apiKey: apiKeyFromAuthorizedHost,
    mode: "qa",
    chatRenderer: "jspuro",
    agUiSseTransport: true,

    onChange(state) {
      if (statusElement) {
        statusElement.textContent = `Estado do chat: ${state.status}`;
      }

      // A observacao publica e por estado. Nao existe callback publico onToken.
      const lastMessage = state.messages.at(-1);
      if (
        state.status === "streaming" &&
        lastMessage?.role === "assistant"
      ) {
        console.debug("Resposta em andamento", {
          receivedCharacters: lastMessage.content.length,
        });
      }
    },
  });

  chat.mount(document.getElementById("chat-container"));
});
```

O componente renderiza textarea, botao de envio, mensagens, estado e `correlation_id`. O host nao precisa desenhar nem atualizar as bolhas.

### 4.3. O payload enviado pelo componente

Para `mode: "qa"`, o componente mapeia o modo para `execution_kind: "rag_qa"` e envia o contrato oficial para `/ag-ui/runs`. O payload inclui, conforme o caso:

- `thread_id`;
- `run_id`;
- `user_email`;
- `input` com a mensagem do usuario;
- uma unica fonte de configuracao autorizada.

Nao monte esse JSON no host. A lista serve para entender o contrato, nao para duplicar o componente.

### 4.4. Como ler a resposta final e as fontes

Durante o stream, observe `state.status === "streaming"`. Quando a rodada termina, consulte o estado atual:

```javascript
const state = chat.getState();
const finalResponse = state.lastResponse;
const sources = finalResponse?.sources ?? [];
```

No Q&A, as fontes chegam pelo estado AG-UI e sao preservadas no payload final. Nao tente deduzir fonte a partir do texto exibido.

## 5. Exemplo B: DeepAgent com streaming e HIL

O HIL pausa a execucao e pede uma decisao humana. A decisao deve ser enviada pelo metodo proprio; escrever "aprovar" no chat nao resolve a interrupcao.

```javascript
const pendingHilElement = document.getElementById("hil-pending");

const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
  yamlContent: window.minhaTela.yamlContent,
  yamlFilename: "agente.yaml",
  userEmail: window.minhaTela.userEmail,
  apiKey: window.minhaTela.apiKey,
  mode: "deepagent",
  chatRenderer: "jspuro",
  agUiSseTransport: true,

  onEvent(event) {
    if (event.type === "hil-pending" && pendingHilElement) {
      pendingHilElement.hidden = false;
    }
  },
});

chat.mount(document.getElementById("chat-container"));

document.getElementById("approve-button").addEventListener("click", async () => {
  await chat.responderHil("approve");
});

document.getElementById("reject-button").addEventListener("click", async () => {
  await chat.responderHil("reject");
});
```

Decisoes aceitas:

- `approve`: aprova e continua;
- `reject`: rejeita;
- `edit`: envia uma edicao estruturada no segundo argumento, quando o contrato da interrupcao permitir.

Depois da decisao, o componente usa o fluxo de retomada existente. Ele preserva `thread_id` e o contexto HIL recebido do backend.

> Generative UI/A2UI e uma capacidade adicional do DeepAgent. O mesmo transporte SSE tambem pode entregar apenas texto. Para renderizacao A2UI, carregue ainda os scripts e os renderers descritos no [Tutorial 101 de Generative UI](./TUTORIAL-101-GENERATIVE-UI.md).

## 6. Exemplo C: `projectKey` sem YAML nem API key no navegador

Use esta forma quando um projeto autorizado pelo servidor e a fonte de configuracao.

```javascript
const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
  projectKey: "projeto-autorizado-pelo-servidor",
  userEmail: window.sessao.email,
  mode: "agent",
  chatRenderer: "jspuro",
  agUiSseTransport: true,
});

chat.mount(document.getElementById("chat-container"));
```

Neste fluxo, o navegador nao deve enviar:

- `yamlContent`;
- `encryptedPayload`;
- `apiKey`;
- `execution_kind` inventado pelo host.

O servidor resolve a configuracao e o tipo de execucao autorizados para aquele projeto. Se `projectKey` competir com uma fonte explicita de YAML, o componente falha antes de iniciar a chamada.

## 7. Exemplo D: fallback consciente para o fluxo classico

Nem todo chat precisa de streaming. Para manter o fluxo classico e receber a resposta completa:

```javascript
const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
  yamlContent: window.minhaTela.yamlContent,
  userEmail: window.minhaTela.userEmail,
  apiKey: window.minhaTela.apiKey,
  mode: "workflow",
  chatRenderer: "jspuro",
  agUiSseTransport: false,
});

chat.mount(document.getElementById("chat-container"));
```

O componente usa o endpoint classico aplicavel e so exibe a resposta do assistente quando ela termina. No componente embutivel atual, esse ramo e `direct_sync`; configurar `executionMode: "direct_async"` ou `"auto"` nao transforma o componente em cliente de polling.

Se um integrador construir um cliente classico proprio, polling e cancelamento assincrono pertencem ao contrato descrito no [Guia de integracao do chat](./GUIA-INTEGRADOR-CHAT-PLATAFORMA.md), e nao ao componente desta pagina.

## 8. Identificadores: nao confunda os tres

| Identificador | Quem fornece | Para que serve |
|---|---|---|
| `thread_id` | cliente/componente, preservado na conversa | agrupa as rodadas de uma conversa |
| `run_id` | cliente/componente, novo por rodada | identifica a rodada no protocolo AG-UI |
| `correlation_id` | boundary oficial da API | rastreia a execucao nos logs e volta no header `X-Correlation-Id` |

O componente pode criar `thread_id` e `run_id` locais para o protocolo. Ele nunca cria `correlation_id`; apenas captura e exibe o valor devolvido pela API.

## 9. Estados que o host pode observar

Use `onChange(state)` ou `chat.getState()`:

| `state.status` | Significado pratico |
|---|---|
| `idle` | componente ainda sem configuracao pronta |
| `ready` | pronto para receber mensagem |
| `sending` | requisicao iniciada e ainda sem o primeiro fragmento |
| `streaming` | texto progressivo sendo recebido |
| `error` | a rodada falhou |

`completed` e `cancelled` podem aparecer como resultado/status de mensagem ou interacao. Eles nao devem ser tratados como novos estados globais do componente.

## 10. Cancelamento: limite atual importante

O botao/metodo de cancelamento do componente consegue interromper a rodada enquanto o estado global ainda e `sending`. Depois que o primeiro fragmento muda o estado para `streaming`, o contrato publico atual nao garante cancelamento de toda a conexao em andamento.

Nao anuncie cancelamento durante todo o stream para o usuario final sem uma evolucao comprovada do componente e testes correspondentes.

O guard de reentrada e o estado desabilitado do composer tambem cobrem apenas `sending`.
Portanto, a host nao deve iniciar outra chamada programatica nem orientar uma segunda
pergunta enquanto `state.status === "streaming"`. O runtime atual nao garante serializacao
de duas rodadas sobrepostas depois do primeiro fragmento.

## 11. Erros comuns e diagnostico

Antes de alterar codigo, responda estas perguntas:

1. `agUiSseTransport` esta exatamente como `true`?
2. `chatRenderer` esta como `jspuro`?
3. O modo e `qa`, `deepagent`, ou existe um `projectKey` valido?
4. Os scripts do cliente AG-UI e da bridge carregaram antes do primeiro envio?
5. O `userEmail` real foi informado?
6. Existe exatamente uma fonte de configuracao?
7. A requisicao e `POST /ag-ui/runs` com `Accept: text/event-stream`?
8. O response devolveu `X-Correlation-Id`?
9. A tela esta tentando usar `EventSource` ou parser SSE paralelo?
10. A tela criou varias bolhas em vez de deixar o componente atualizar uma unica mensagem?

Use a aba Network do navegador para confirmar metodo, endpoint, status e headers. Nao exponha corpo com segredo em print ou console.

### 11.1. A resposta veio inteira, sem streaming

Isso normalmente significa que o componente escolheu conscientemente o ramo classico. Confira flag, renderer e modo. `workflow` e `copilotkit`, por exemplo, nao entram no ramo SSE apenas porque a flag esta ligada.

### 11.2. A chamada falhou antes de chegar ao backend

Confirme se `projectKey` foi combinado indevidamente com YAML, payload criptografado ou API key. Esse conflito deve falhar fechado.

### 11.3. Recebi texto, mas nao recebi fontes

Confirme se a execucao Q&A produziu `sources` no snapshot de estado AG-UI. O componente preserva as fontes recebidas; ele nao as inventa.

### 11.4. O HIL apareceu, mas digitar "sim" nao continuou

Use `chat.responderHil("approve")`, `"reject"` ou `"edit"`. Mensagem de chat e decisao HIL sao canais diferentes.

### 11.5. Posso usar `EventSource`?

Nao. `EventSource` abre `GET`; `/ag-ui/runs` exige `POST` com JSON. Use o componente global dentro da plataforma ou o cliente AG-UI oficial atras de um BFF, conforme o guia de parceiros.

### 11.6. Posso atualizar o DOM a cada token pelo host?

Nao e necessario. Observe o estado apenas quando a tela precisar de telemetria ou controles externos. A bolha e responsabilidade do componente.

## 12. Guia rapido para validar o exemplo

1. Abra a pagina autenticada.
2. Confirme que o chat chegou ao estado `ready`.
3. Envie uma pergunta.
4. Na aba Network, confirme `POST /ag-ui/runs` e `text/event-stream`.
5. Observe uma unica bolha do assistente crescer.
6. Ao final, confirme a resposta consolidada, as fontes quando existirem e o `correlation_id` exibido.
7. No DeepAgent, force um caso HIL e confirme que os botoes chamam `responderHil`.
8. Revise console e requests para garantir que nenhum segredo foi exposto.

## 13. Evidencia e limite desta documentacao

O contrato descrito aqui foi confirmado no componente global, no transporte AG-UI, na bridge, no endpoint `/ag-ui/runs`, nos hosts oficiais e nos testes focados existentes. A investigacao documental que originou este tutorial encontrou 53 testes frontend focados verdes para o transporte e o componente.

Essa evidencia nao equivale a uma prova E2E com backend real e navegador real para todos os exemplos desta pagina. A rodada backend focada observada na investigacao nao encerrou normalmente, e nao houve validacao API-live do SSE nessa mesma rodada. Por isso, o tutorial separa o contrato executavel comprovado daquilo que ainda requer uma execucao integrada no ambiente de destino.

## 14. Proximos guias

- [Guia completo do componente WebChat embutivel](./GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md)
- [Guia para integradores do chat da plataforma](./GUIA-INTEGRADOR-CHAT-PLATAFORMA.md)
- [Guia AG-UI para SDKs e parceiros externos](./GUIA-AG-UI-SDK-TERCEIROS.md)
- [Tutorial 101 de Generative UI](./TUTORIAL-101-GENERATIVE-UI.md)
- [FAQ de onboarding](./faq-onboarding.md)
