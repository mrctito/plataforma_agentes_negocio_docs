# Guia AG-UI SDK para terceiros

Este guia e a porta de entrada para integrar uma interface externa ao AG-UI da plataforma Plataforma de Agentes de IA.

Ele nao substitui o manual tecnico. Ele explica o fluxo em linguagem direta, mostra o caminho seguro para o primeiro run e aponta onde aprofundar.

Importante: o pacote `@prometeu/ag-ui-runtime` continua `private: true` dentro do monorepo. Para terceiros, ele nao e um SDK publico instalado por npm. O caminho suportado para fora do repositório e o template de referencia com `@ag-ui/client` e um backend-for-frontend proprio.

Referencias tecnicas principais:

1. [README-TECNICO-AG-UI.md](../tecnico/README-TECNICO-AG-UI.md)
2. [README-TECNICO-AG-UI-BORDA-HTTP-DEDICADA.md](../tecnico/README-TECNICO-AG-UI-BORDA-HTTP-DEDICADA.md)
3. [README-TECNICO-AG-UI-RUNTIME-COMPARTILHADO-DO-FRONTEND.md](../tecnico/README-TECNICO-AG-UI-RUNTIME-COMPARTILHADO-DO-FRONTEND.md)
4. [README-TECNICO-AG-UI-REPLAY-E-AUDITORIA.md](../tecnico/README-TECNICO-AG-UI-REPLAY-E-AUDITORIA.md)
5. [README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md](../tecnico/README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md)
6. [templates/ag-ui-official-third-party](../../templates/ag-ui-official-third-party)
7. [README-TECNICO-CHAT-QA-FONTES-NA-UI.md](../tecnico/README-TECNICO-CHAT-QA-FONTES-NA-UI.md)
   — como capturar as fontes do Q&A/RAG e exibi-las no chat. Leitura obrigatoria para quem monta
   chat em modo Q&A: a resposta do modelo vem SEM citacao no corpo do texto, e relacionar as
   fontes no final e responsabilidade do cliente.

## 0. Tres caminhos para colocar um chat conversando com um agente da plataforma

Antes de escolher entre "protocolo AG-UI cru" e "CopilotKit", vale situar as **tres**
superficies reais que este repositorio oferece para conversar com um agente. Elas nao
sao concorrentes entre si; servem publicos e restricoes tecnicas diferentes.

**Status de validacao (leia antes de escolher):** os caminhos **1** e **2** possuem wiring
executavel e testes focados de contrato. O componente e usado por hosts reais da
plataforma, e 53 testes frontend focados do transporte/componente ficaram verdes na
investigacao que sincronizou este guia. Essa evidencia nao equivale a um E2E de parceiro
contra backend real: a mesma rodada nao executou uma prova browser + API-live do SSE. O
caminho **3** (CopilotKit) esta implementado, mas continua sem validacao ponta a ponta
registrada — aviso completo na secao 2.1.

1. **Componente pronto `PrometeuEmbeddableChatRuntime` (JS puro, caminho mais rapido).**
   Use quando sua tela e servida pela propria origem da plataforma (ou pode carregar os
   scripts estaticos dela) e voce quer um chat funcional em minutos — sem escrever
   `fetch`, criptografia de YAML nem parsing de evento. Ele resolve handshake, payload,
   modo de execucao, leitura de `correlation_id` e HIL sozinho. Guia completo:
   [GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md](GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md). Esse
   componente conversa com os endpoints classicos de execucao
   (`/rag/execute`, `/agent/execute`, `/workflow/execute` — documentados no
   [GUIA-INTEGRADOR-CHAT-PLATAFORMA.md](GUIA-INTEGRADOR-CHAT-PLATAFORMA.md), que e o
   irmao deste guia para quem quer construir uma UI 100% propria sobre **esses** mesmos
   endpoints classicos, sem AG-UI). Ele abre `POST /ag-ui/runs` de forma **opt-in** para
   Q&A/RAG, DeepAgent ou `projectKey`, quando a host usa `chatRenderer: "jspuro"` e liga
   `agUiSseTransport`. A2UI e uma capacidade adicional do DeepAgent, nao a unica razao
   para usar SSE. Tutorial copiavel:
   [TUTORIAL-CHAT-PLATAFORMA.md](TUTORIAL-CHAT-PLATAFORMA.md).

2. **Protocolo AG-UI — `POST /ag-ui/runs` + SSE com `@ag-ui/client`.** Use quando
   voce quer uma UI 100% propria (React, Vue, mobile, outro backend) falando
   diretamente o protocolo aberto AG-UI, com replay, HIL e discovery de capabilities.
   **Este e o assunto central deste guia** — secoes 1 a 4 abaixo, com o exemplo real de
   consumo de stream na secao 3.5.1. O template tem teste de contrato automatizado
   (`tests/unit/test_02-01-52_ag_ui_third_party_template_contract.py`), sem registro de
   execucao manual por um integrador externo real nesta rodada.

3. **Cliente CopilotKit (React) — `POST /ag-ui/copilotkit/runs`.** Use quando voce ja
   tem (ou vai construir) um app React usando o SDK CopilotKit e quer que ele converse
   com os DeepAgents/Workflows da plataforma sem reimplementar o protocolo do zero.
   Coberto nas secoes 2.1 (contrato da requisicao) e 2.2 (o que dos recursos do
   CopilotKit realmente funciona com os DeepAgents desta plataforma). **Status:
   implementado, porem NAO testado** ⚠️ — use com verificacao propria e reporte
   problemas; detalhe na nota de maturidade no inicio da secao 2.1.

Os caminhos 2 e 3 falam o **mesmo** protocolo AG-UI e o **mesmo** boundary de execucao
(`AgUiRunOrchestrator`) — a diferenca e so o formato do envelope HTTP que cada cliente
ja fala nativamente (`AgUiRunRequest` proprio da plataforma vs. `RunAgentInput` puro do
CopilotKit, traduzido por um servico de compatibilidade). O caminho 1 encapsula tanto o
ramo classico quanto o ramo SSE; a host nao implementa nenhum dos dois. Nos caminhos 2 e
3, o parceiro controla a interface e precisa manter o protocolo e a fronteira de
seguranca.

Se a sua duvida for "quero so um chat funcionando na minha tela o mais rapido possivel,
e minha tela roda dentro da propria plataforma (ou pode carregar os scripts dela)",
pare aqui e va para o
[GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md](GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md). O
restante deste guia e para quem precisa falar o protocolo AG-UI diretamente — com UI
propria (secoes 1-4) ou com CopilotKit (secoes 2.1-2.2).

## 1. O que e AG-UI na plataforma

AG-UI e o contrato que permite que uma tela acompanhe uma execucao de IA como um processo observavel.

Em vez de receber apenas uma resposta final em texto, a interface recebe eventos. Esses eventos mostram quando a execucao comecou, qual mensagem chegou, qual estado mudou, se alguma ferramenta de interface foi declarada, se houve pausa para revisao humana e como o run terminou.

Em linguagem simples: AG-UI e a trilha entre a tela e o runtime agentic. A tela nao precisa conhecer YAML interno, banco, DSN, catalogo de tools LangChain ou segredos. O boundary produtivo recebe um `AgUiRunRequest` em `POST /ag-ui/runs`, deriva o runtime pelo YAML e devolve um stream de eventos para desenhar a experiencia. Quando um integrador usa um backend-for-frontend proprio, o browser ainda pode montar um `RunAgentInput` local para conversar com esse backend, mas a chamada produtiva para a plataforma continua sendo `/ag-ui/runs`.

### 1.1. AG-UI nao e a spec visual

AG-UI e o protocolo de eventos. Ele informa quando o run comecou, qual mensagem chegou, qual estado mudou, se houve pausa humana e como a execucao terminou.

Generative UI e outra camada: e o formato visual que viaja dentro de alguns desses eventos para a tela montar um painel, um canvas ou um componente dinamico.

Na Plataforma de Agentes de IA, o perfil visual oficial do chat generativo e o **A2UI**: um envelope declarativo (`{a2ui_operations: [createSurface, updateComponents]}`) que o supervisor DeepAgent produz via tool `generate_a2ui` quando o YAML declara o bloco `ag_ui.generative`.

Isso significa duas coisas praticas:

1. o frontend Plataforma de Agentes de IA nao renderiza JSON visual arbitrario vindo de fora;
2. mesmo usando o formato aberto A2UI, o que de fato chega a tela e restrito a um **catalogo fechado e governado pela plataforma** (8 componentes hoje: `Card`, `Column`, `Row`, `Text`, `Divider`, `BarChart`, `LineChart`, `DataTable`) — declarado no YAML do supervisor, nunca escolhido livremente pelo integrador.

Essa regra existe para manter bloqueio de HTML, script, SQL livre, DSN, segredos e `correlation_id` no payload visual.

### 1.2. Contrato oficial para adapter externo

Se um integrador usar um formato visual proprio diferente do A2UI (Open JSON UI, MCP-UI ou equivalente), essa spec nao entra direto no renderer Plataforma de Agentes de IA.

O contrato oficial e este:

1. O adapter seguro recebe a spec externa no backend, nunca no renderer final.
2. O adapter converte a spec externa para o formato A2UI, restrito ao catalogo de componentes que o YAML do supervisor declarar, antes da materializacao AG-UI.
3. O payload convertido continua sujeito ao mesmo renderer fail-closed do produto (componente fora do catalogo, ou dado malformado, derruba a superficie inteira para texto).
4. HTML, JavaScript, SQL livre, DSN, segredo e `correlation_id` continuam proibidos, mesmo que existam na spec de origem.
5. Sem adapter explicito, a spec externa deve falhar fechada.

Em linguagem simples: o Plataforma de Agentes de IA nao aceita desenho arbitrario vindo de fora. Ele so aceita o desenho final no catalogo governado do proprio produto — ainda que o formato de transporte (A2UI) seja um padrao aberto.

> Um caminho alternativo existe para quem já integra com o SDK CopilotKit: declarando `chat_renderer: copilotkit` no bloco `ag_ui.generative`, a rota `POST /ag-ui/copilotkit/runs` repassa o envelope A2UI sem filtro para um cliente React que usa `@ag-ui/a2ui-middleware`. Nesse caminho, o catalogo continua vindo do YAML da plataforma — o cliente terceiro não escolhe componentes livremente.

## 2. Contrato publico recomendado

Para integracao nova de terceiros, use este endpoint:

```text
POST /ag-ui/runs
```

O body produtivo deve seguir `AgUiRunRequest`.

A plataforma nao expõe mais rota publica por `agent_id`, nem para execucao nem para capabilities. Se o template third-party ainda usar `RunAgentInput` no browser do integrador, o backend desse integrador deve converter esse envelope para `AgUiRunRequest` antes de chamar a plataforma.

Replay publico:

```text
GET /ag-ui/runs/{run_id}/events
GET /ag-ui/threads/{thread_id}/events
```

Discovery publico:

```text
GET /ag-ui/capabilities
```

Use `GET /ag-ui/capabilities` para descobrir o catalogo de negocio Plataforma de Agentes de IA, ou seja, quais capabilities, exemplos, permissoes e `UISpecs` governadas estao disponiveis para o tenant.

Se o seu cliente precisar de um shape equivalente a `AgentCapabilities`, faca esse mapeamento no backend do integrador a partir do proprio payload de `GET /ag-ui/capabilities`. O produto nao mantem um segundo endpoint publico so para projetar esse contrato por `agent_id`.

## 2.1. Cliente CopilotKit (React) via `/ag-ui/copilotkit/runs`

### Por que existe (nivel 101)

O contrato da secao 2 (`POST /ag-ui/runs`) espera o envelope `AgUiRunRequest`, com campos proprios da plataforma (`user_email`, fonte YAML). Um cliente React com **CopilotKit**, porem, nao fala esse envelope: o runtime do CopilotKit aponta o `runtimeUrl` de cada agente para uma URL e faz `POST` de um `RunAgentInput` **AG-UI puro** (a conversa nativa: `threadId`, `runId`, `messages`, `state`, `tools`, `context`, `forwardedProps`). Os dois formatos sao incompativeis.

O endpoint **`POST /ag-ui/copilotkit/runs`** fecha exatamente essa lacuna: ele aceita o `RunAgentInput` puro do CopilotKit, traduz para o `AgUiRunRequest` canonico no **lado da plataforma** e delega ao mesmo boundary de execucao (mesma resolucao de YAML, mesma autenticacao, mesmo stream SSE). Em vez de o parceiro escrever a conversao `RunAgentInput -> AgUiRunRequest` no proprio backend (secao 2), a plataforma faz isso por ele.

**Quando usar:** o parceiro ja tem (ou quer ter) um cliente React sofisticado com CopilotKit e quer que ele consuma os DeepAgents/Workflows da plataforma com chat + Generative UI (dashboards), sem reimplementar o protocolo. Para clientes que nao usam CopilotKit, o caminho continua sendo o `/ag-ui/runs` da secao 2.

### Contrato da requisicao

```text
POST /ag-ui/copilotkit/runs
Header:  X-API-Key: <chave-do-servidor>
Body:    RunAgentInput AG-UI puro (camelCase)
```

O **corpo** e o `RunAgentInput` que o CopilotKit ja monta. A configuracao da plataforma viaja no sub-objeto **`forwardedProps.platform`** — o escape hatch padrao do CopilotKit para metadados de aplicacao:

| Campo em `forwardedProps.platform` | Obrigatorio | Significado |
| --- | --- | --- |
| `userEmail` | Sim | Identidade do usuario final (sem fallback; falha fechada se ausente). |
| `yamlConfig` \| `yamlInlineContent` \| `encryptedData` | Sim (exatamente uma) | Fonte de configuracao governada, igual ao `/ag-ui/runs`. Mais de uma → erro 400. |
| `executionKind` | Nao | Cross-check opcional; o runtime real (`deepagent`/`workflow`) e **derivado do YAML**. Se divergir, 400. |
| `metadata` | Nao | Metadados livres repassados ao contexto canonico. |

Exemplo de corpo (montado pelo backend confiavel do parceiro, **nunca** no browser cru):

```json
{
  "threadId": "thread-demo-001",
  "runId": "run-demo-001",
  "state": {},
  "messages": [
    { "id": "m1", "role": "user", "content": "Mostra o dashboard de vendas do periodo" }
  ],
  "tools": [],
  "context": [],
  "forwardedProps": {
    "platform": {
      "userEmail": "operacao@cliente.exemplo",
      "yamlInlineContent": "<yaml-governado-no-servidor>"
    }
  }
}
```

### Como a plataforma trata o payload (e por que e seguro)

1. O sub-objeto `forwardedProps.platform` e **lido e removido** antes de a conversa seguir para o agente. A `yamlConfig`/`encryptedData` entra pela porta confiavel `AgUiRunRequest.yaml_config`/`encrypted_data` (a mesma do `/ag-ui/runs`), **nunca** vaza para dentro do `forwardedProps` que o agente recebe.
2. O restante do `RunAgentInput` (a conversa: `messages`, `state`, `tools` e os demais `forwardedProps` do parceiro) e propagado integralmente ao `LangGraphAgent` oficial, sem recorte — confirmado lendo `AgUiLangGraphExecutionAdapter._build_run_input` (`src/api/services/ag_ui_langgraph_agent_factory.py`): quando `context.protocol_input` existe, ele e revalidado como `RunAgentInput` inteiro e entregue ao `LangGraphAgent.run(...)` do pacote oficial `ag-ui-langgraph`, que propaga sozinho para o grafo. Isso e o que sustenta **frontend tools/actions e shared state do CopilotKit** (detalhe e limites reais na secao 2.2 — inclusive um caso em que o HIL **nativo da plataforma** ainda nao tem retomada ligada por este endpoint).
3. A resposta e o **mesmo stream AG-UI por SSE** da secao 2 (mesma matriz de eventos da secao 7), com o header `X-Correlation-Id` oficial.

A regra de seguranca da secao 4 continua valendo integralmente: `userEmail`, `yamlConfig`/`encryptedData` e a `X-API-Key` sao injetados pelo **servidor confiavel do parceiro** (o backend-for-frontend que faz a ponte com o CopilotKit), nunca no JavaScript publico. O threat model da secao 4.3 nao muda: campos sensiveis escondidos em `state`/`messages`/`forwardedProps` continuam barrados em falha fechada.

### Onde o CopilotKit aponta

No cliente React do parceiro, o `runtimeUrl` de cada agente aponta para o backend-for-frontend do parceiro, que reescreve a chamada para `POST /ag-ui/copilotkit/runs` adicionando a `X-API-Key` e o `forwardedProps.platform`. O CopilotKit segue falando `RunAgentInput` puro de ponta a ponta; a traducao para o contrato governado acontece nesse salto servidor-a-servidor.

### Wiring do cliente React (referencia oficial do SDK CopilotKit)

Esta subsecao documenta como o **SDK CopilotKit** (pacotes `@copilotkit/react-core`, `@copilotkit/runtime`, `@ag-ui/client`) espera ser configurado para apontar a um agente AG-UI proprio — isto e comportamento **oficial do CopilotKit**, nao do nosso backend; o contrato do nosso servidor continua sendo exatamente o descrito acima (`RunAgentInput` + `forwardedProps.platform`).

O SDK CopilotKit conecta um agente por HTTP com a classe `HttpAgent` do pacote `@ag-ui/client`, que aceita `url` e `headers` fixos:

```typescript
// backend-for-frontend do parceiro (Node.js, ex.: rota de API do Next.js)
// server-side — nunca no browser.
import { CopilotRuntime, copilotRuntimeNextJSAppRouterEndpoint } from "@copilotkit/runtime";
import { HttpAgent } from "@ag-ui/client";

const runtime = new CopilotRuntime({
  agents: {
    // "default" (ou o nome que o front escolher) aponta para o endpoint
    // de compatibilidade da Plataforma de Agentes de IA.
    default: new HttpAgent({
      url: "https://prometeu.exemplo.local/ag-ui/copilotkit/runs",
      headers: { "X-API-Key": process.env.PROMETEU_AG_UI_API_KEY! },
    }),
  },
});

export const POST = async (req: NextRequest) => {
  const { handleRequest } = copilotRuntimeNextJSAppRouterEndpoint({
    runtime,
    endpoint: "/api/copilotkit",
  });
  return handleRequest(req);
};
```

No React do parceiro, o provider `<CopilotKit>` so precisa apontar para **o proprio backend do parceiro**, nunca direto para a plataforma:

```tsx
import { CopilotKit } from "@copilotkit/react-core";
import { CopilotChat } from "@copilotkit/react-ui";

export default function App() {
  return (
    <CopilotKit runtimeUrl="/api/copilotkit">
      <CopilotChat />
    </CopilotKit>
  );
}
```

**Ponto critico de seguranca, que o SDK CopilotKit nao resolve sozinho:** o `HttpAgent.headers` acima e **estatico** (a mesma `X-API-Key` do servidor para todo mundo) — ele resolve bem a chave de API, que e uma credencial do servidor do parceiro, nao do usuario final. Mas `forwardedProps.platform.userEmail` e a fonte de YAML **variam por usuario/tenant** e nao tem, hoje, um hook documentado do CopilotKit para injecao dinamica por request dentro de `CopilotRuntime`/`HttpAgent`. O caminho seguro e o mesmo da secao 3 deste guia: a rota `/api/copilotkit` do backend-for-frontend do parceiro (o handler acima) le a sessao autenticada do usuario **antes** de `handleRequest(req)`, e monta a chamada final para `/ag-ui/copilotkit/runs` com `forwardedProps.platform` resolvido a partir dessa sessao — nunca a partir de um valor que o browser escolheu.

> Nao confundir com a prop `properties` do `<CopilotKit>` (`<CopilotKit properties={{...}}>`, documentada no SDK para repassar dados como `tenantId` para `forwardedProps`): essa prop roda no **browser** e o valor vai exposto no payload publico. Ela serve para metadado de aplicacao não sensível — nunca para `userEmail`/`yamlInlineContent`/`encryptedData`, que precisam nascer no servidor confiavel (secao 4 deste guia).
>
> Referencia oficial consultada para este wiring: documentacao do pacote `@copilotkit/runtime` (`CopilotRuntime`, `copilotRuntimeNextJSAppRouterEndpoint`) e `@ag-ui/client` (`HttpAgent`). Comportamento de terceiros — nao e codigo deste repositorio; o contrato que o nosso servidor aceita continua sendo o documentado nas subsecoes anteriores desta secao 2.1.

## 2.2. Recursos do CopilotKit com os DeepAgents da plataforma (o que funciona de fato)

O CopilotKit anuncia um conjunto de recursos de alto nivel (streaming, frontend
actions, human-in-the-loop, generative UI, shared state). Esta secao mapeia, recurso a
recurso, o que **realmente** chega a um DeepAgent desta plataforma quando o cliente usa
`/ag-ui/copilotkit/runs` — com honestidade sobre o que nao esta ligado hoje.

### Streaming de mensagens — funciona

O CopilotKit consome o mesmo stream SSE AG-UI da secao 7 (matriz de eventos). Como o
`RunAgentInput` do CopilotKit e revalidado e entregue integralmente ao `LangGraphAgent`
oficial (sem recorte, confirmado no codigo), os eventos `TEXT_MESSAGE_START/CONTENT/END`
chegam exatamente como em qualquer outro cliente AG-UI. `useCopilotChat`/`CopilotChat`
do lado React consomem isso nativamente.

### Frontend actions/tools (`useCopilotAction`) — funciona, com a mesma allowlist da secao 4.2

Uma `useCopilotAction` declarada no React vira, no protocolo, uma entrada em
`RunAgentInput.tools`. Isso chega ao DeepAgent do mesmo jeito que qualquer frontend
tool do caminho cru (secao 4.2): **nao** e uma Tool LangChain da plataforma, e vale a
mesma regra pratica — se a capability nao publicou aquele nome em `frontendTools` no
discovery (`GET /ag-ui/capabilities`), a tool nao deveria ser usada para autorizar nada
alem de uma acao visual da interface.

### Human-in-the-loop — dois mecanismos diferentes, so um funciona ponta a ponta hoje

O CopilotKit documenta HIL como `useCopilotAction({ renderAndWaitForResponse })`: uma
acao de frontend que o agente chama, a interface renderiza um painel de aprovacao e
**a propria conversa continua** com o resultado da decisao como resposta da tool — sem
precisar de um mecanismo de pausa/retomada dedicado no protocolo. Esse padrao, sendo
apenas troca de mensagens (`messages`/`tools`), **funciona** com o DeepAgent da mesma
forma que qualquer frontend tool.

Ja o **HIL nativo desta plataforma** e outro mecanismo: uma pausa real do grafo
LangGraph (`interrupt()`), que termina o run com `RUN_FINISHED` e
`outcome.type: "interrupt"` (secao 7, secao 4.6), e so continua com um **novo run**
carregando `AgUiRunRequest.resume` (lista de `AgUiResumeInput`: `interruptId`,
`status`, `payload.decisions`). **Esse campo `resume` nao e populado pelo
`CopilotKitAgUiCompatService`** (`src/api/services/copilotkit_ag_ui_compat_service.py`,
metodo `build_run`, lido por completo): ele nunca le nem `forwardedProps.platform` nem
nenhum outro lugar do `RunAgentInput` para preencher `AgUiRunRequest.resume` — o valor
fica sempre `None`. Na pratica: um DeepAgent que pausa (`RUN_FINISHED`/`interrupt`)
durante uma execucao iniciada por `/ag-ui/copilotkit/runs` **emite a pausa
corretamente** para o cliente CopilotKit (o evento chega), mas **nao ha, hoje, um
caminho comprovado no codigo para retomar essa pausa especifica atraves do mesmo
endpoint `/ag-ui/copilotkit/runs`** — a retomada testada e documentada (secao 4.6, e o
teste `tests/unit/test_02-01-100_ag_ui_copilotkit_compat.py`) so cobre a traducao da
requisicao inicial, nao um segundo round-trip de `resume`. Isso e uma lacuna real, nao
uma limitacao de design explicada em algum lugar do codigo — registre-a como tal ao
planejar um caso de uso que dependa de HIL nativo via CopilotKit.

### Generative UI — funciona, via A2UI, sem passar pelo Component Catalog do CopilotKit

O gatilho e um unico campo no YAML do supervisor DeepAgent:

```yaml
multi_agents:
  - id: ag_ui_pdv_vendas_supervisor
    execution:
      type: deepagent
    ag_ui:
      generative:
        chat_renderer: copilotkit   # em vez de "jspuro" — sinaliza cliente CopilotKit
        a2ui_schema:
          catalog_id: "https://plataforma.local/catalogs/pdv-vendas-a2ui.json"
          components: [Card, Column, Row, Text, BarChart, LineChart, DataTable, Divider]
```

- `chat_renderer: copilotkit` (obrigatorio dentro do bloco `generative`): so muda **quem
  consome** o envelope A2UI — o supervisor continua gerando o mesmo conteudo com
  `jspuro` ou `copilotkit` (proximo paragrafo comprova isso lendo o codigo).
- `a2ui_schema.catalog_id`/`a2ui_schema.components`: identicos ao caminho `jspuro`,
  contrato completo e exemplo passo a passo em
  [TUTORIAL-101-GENERATIVE-UI.md](TUTORIAL-101-GENERATIVE-UI.md).

Quando o YAML do supervisor declara `ag_ui.generative.chat_renderer: copilotkit`, o
supervisor gera o **mesmo** envelope A2UI (`generate_a2ui`, `{a2ui_operations:
[createSurface, updateComponents]}`) que geraria com `chat_renderer: jspuro` — a
injecao do catalogo no estado do grafo nao muda de comportamento por causa do
`chat_renderer` (confirmado lendo `AgUiDeepAgentAdapter._build_a2ui_schema_context_entries`,
onde o valor so aparece em log, nunca em um `if`/branch). O que muda e **quem renderiza**:
com `copilotkit`, o `TOOL_CALL_RESULT` carregando o envelope A2UI chega sem filtro ao
cliente React, e quem o transforma em componentes visuais e o middleware oficial
`@ag-ui/a2ui-middleware` (pacote npm mantido fora deste repositorio, ja embutido no
runtime do CopilotKit quando ele fala AG-UI) — nao o renderer `jspuro` desta
plataforma. O catalogo de 8 componentes continua sendo o mesmo, governado pelo YAML;
o integrador nao escolhe componentes livremente.

### Shared state (`useCoAgent`) — passthrough comprovado no codigo, sem teste automatizado ate a ponta do grafo

O campo `state` do `RunAgentInput` do CopilotKit atravessa intacto ate
`RunAgentInput.model_validate(dict(context.protocol_input))`, dentro de
`AgUiLangGraphExecutionAdapter._build_run_input`, e e entregue ao `LangGraphAgent.run(...)`
do pacote oficial `ag-ui-langgraph` sem recorte prévio deste repositorio. Isso significa
que a sincronizacao de estado que `useCoAgent` espera (eventos `STATE_SNAPSHOT`/
`STATE_DELTA`, presentes na matriz de eventos da secao 7) depende, na pratica, do
comportamento padrao do SDK `ag-ui-langgraph` ao redor do grafo do DeepAgent — este
projeto nao adiciona nem recorta esse comportamento. **Nao encontramos, na leitura
realizada para este guia, um teste automatizado do repositorio que exercite `useCoAgent`
sincronizando estado real de um DeepAgent ponta a ponta**; a afirmacao acima e baseada em
leitura direta do wiring do adapter, nao em um teste que prove o comportamento no
frontend React. Trate como comportamento esperado do SDK oficial, nao como garantia
testada deste repositorio.

## 3. Primeiro agent em 10 minutos

Este fluxo parte do template oficial criado para integradores externos.

### 3.1. Copie o template operacional

Use [templates/ag-ui-official-third-party](../../templates/ag-ui-official-third-party).

O template tem duas partes:

1. Backend-for-frontend em FastAPI.
2. Frontend vanilla com Vite e `@ag-ui/client`.

O backend-for-frontend e necessario porque a credencial de acesso a Plataforma de Agentes de IA e a fonte de configuracao precisam ficar no servidor do integrador, nunca no browser.

### 3.2. Configure somente o servidor

No backend do integrador, configure:

```text
PROMETEU_AG_UI_BASE_URL=https://prometeu.exemplo.local
PROMETEU_AG_UI_API_KEY=<chave-servidor>
PROMETEU_AG_UI_USER_EMAIL=operacao@cliente.exemplo
PROMETEU_AG_UI_YAML_INLINE_CONTENT=<yaml-governado-no-servidor>
PUBLIC_TENANT_LABEL=ambiente-demo
```

`PROMETEU_AG_UI_API_KEY` e exemplo de nome de variavel, nao valor real. O valor deve ficar fora do frontend e fora de arquivos versionados.

Se o integrador ja tiver um envelope criptografado emitido pela plataforma, pode usar `PROMETEU_AG_UI_ENCRYPTED_DATA_JSON` no servidor em vez de `PROMETEU_AG_UI_YAML_INLINE_CONTENT`.

### 3.3. Descubra a capability

Chame:

```text
GET /ag-ui/capabilities
```

Voce pode filtrar por `executionKind` (`deepagent`, `workflow` ou `capability_pack`) e, para capability packs, tambem por `capabilityPackId` (`retail_demo`, `erp_backoffice_demo`):

```text
GET /ag-ui/capabilities?executionKind=capability_pack&capabilityPackId=retail_demo
```

Olhe principalmente para estes campos:

1. `executionKind`: identifica o runtime ou o executionKind compartilhado `capability_pack`.
2. `capabilityPackId`: quando presente, identifica qual pack de negocio (`retail_demo`, `erp_backoffice_demo`) responde por aquela capability dentro do executionKind `capability_pack`.
3. `capability`: identifica a intencao de negocio liberada.
4. `frontendTools`: lista de ferramentas de interface que o browser pode declarar.
5. `supportedEvents`: lista de eventos que a tela deve entender.
6. `supportsHil` e `supportsResume`: indicam pausa humana e retomada.

**Nao existe rota publica por `agent_id`** neste boundary — nem para execucao nem para capabilities (secao 2 acima). Existe, porem, um metodo de servico interno (`AgUiCapabilitiesService.get_canonical_agent_capabilities`, com cobertura de teste unitario) que projeta uma capability isolada no shape oficial `AgentCapabilities` do SDK (`identity`, `transport`, `tools`, `state`, `humanInTheLoop`, `multiAgent`); ele **nao esta ligado a nenhum endpoint HTTP hoje** — e implementacao sem rota exercitada, nao um caminho publico disponivel. Se seu cliente precisa desse shape especifico, monte-o no seu proprio backend a partir do payload de `GET /ag-ui/capabilities`.

### 3.4. Monte um AgUiRunRequest minimo

Exemplo seguro do servidor confiavel para a plataforma:

```json
{
  "threadId": "thread-demo-001",
  "runId": "run-demo-001",
  "user_email": "operacao@cliente.exemplo",
  "input": {
    "capability": "sales_summary",
    "parameters": {}
  },
  "metadata": {
    "surface": "third-party-demo"
  },
  "yaml_inline_content": "<yaml-governado-no-servidor>"
}
```

Esse JSON representa a chamada do backend confiavel para `POST /ag-ui/runs`, nao o payload exposto no browser.

Se a integracao usar um backend-for-frontend proprio, o browser pode continuar montando um `RunAgentInput` local para falar com esse backend. O ponto importante e que o servidor do integrador converta esse envelope para `AgUiRunRequest` antes de chamar a plataforma.

O browser nao deve enviar:

1. YAML bruto.
2. `executionKind`.
3. `tenantId`.
4. `security_keys` (o boundary bloqueia tambem a grafia `securityKeys`).
5. `tools_library` (o boundary bloqueia tambem a grafia `toolsLibrary`).
6. DSN ou connection string.
7. SQL livre.
8. API key.

Esses dados pertencem ao backend da Plataforma de Agentes de IA ou ao backend-for-frontend do integrador.

### 3.5. Consuma o stream com o SDK oficial

No frontend, use `@ag-ui/client`.

Para terceiros, nao consuma `@prometeu/ag-ui-runtime` diretamente. Esse pacote organiza o runtime interno da Plataforma de Agentes de IA e permanece privado. A integracao externa suportada e: template de referencia + `@ag-ui/client` + backend-for-frontend mantendo a chave no servidor.

O template demonstra o fluxo com:

1. `runHttpRequest` para abrir o POST streaming.
2. `transformHttpEventStream` para transformar o stream em eventos oficiais.
3. Um store local simples para desenhar status, mensagens e eventos.

O frontend pode trocar completamente a camada visual. O protocolo nao muda.

### 3.5.1. Exemplo E: browser + BFF + SSE + HIL

O trecho abaixo reproduz, quase literalmente, o codigo real do template
(`templates/ag-ui-official-third-party/frontend/src/ag-ui-client.js` e `main.js`) e
acrescenta o tratamento de pausa humana (HIL) — que **o template de referencia hoje NAO
implementa no frontend** (confirmado lendo `frontend/src/main.js`: o handler de
`RUN_FINISHED` so muda o status visual para "completed", sem checar `event.outcome`).
Se a sua integracao usa DeepAgent com HIL (secao 4.6), o bloco `if (event.outcome?.type
=== 'interrupt')` abaixo e uma extensao que voce precisa escrever — nao vem pronta.

O endpoint usado pelo browser e `/api/ag-ui/runs`, pertencente ao BFF do parceiro. API
key, e-mail confiavel e YAML ficam no BFF. Somente o BFF chama o `/ag-ui/runs` da
plataforma.

HTML minimo da interface do parceiro:

```html
<p id="status" role="status">Pronto</p>
<p>Rastreamento: <code id="correlation-id">aguardando</code></p>
<div id="messages" aria-live="polite"></div>
<div id="hil-actions" hidden></div>
<div id="sources"></div>
```

```javascript
import { runHttpRequest, transformHttpEventStream } from '@ag-ui/client';
import { Observable } from 'rxjs';

function createHeaderAwareEvents(httpEvents, onCorrelationId) {
  return new Observable((subscriber) => {
    const subscription = httpEvents.subscribe({
      next(httpEvent) {
        if (httpEvent?.type === 'headers') {
          const correlationId = httpEvent.headers?.get?.('X-Correlation-Id')
            || httpEvent.headers?.get?.('x-correlation-id')
            || '';
          if (correlationId) onCorrelationId?.(correlationId);
        }
        subscriber.next(httpEvent);
      },
      error(error) { subscriber.error(error); },
      complete() { subscriber.complete(); },
    });
    return () => subscription.unsubscribe();
  });
}

/** Abre o POST streaming e devolve uma Promise que resolve quando o run termina. */
function runOfficialAgUiStream({ endpoint, payload, onEvent, onCorrelationId, signal }) {
  const requestInit = {
    method: 'POST',
    headers: { Accept: 'text/event-stream', 'Content-Type': 'application/json' },
    credentials: 'include',
    body: JSON.stringify(payload),
    signal,
  };
  // runHttpRequest/transformHttpEventStream sao do SDK oficial @ag-ui/client;
  // este projeto nao reimplementa parsing de protocolo (README-TECNICO-AG-UI.md, secao 14.1, item 7).
  const httpEvents = createHeaderAwareEvents(
    runHttpRequest(endpoint, requestInit),
    onCorrelationId,
  );
  const agUiEvents = transformHttpEventStream(httpEvents);

  return new Promise((resolve, reject) => {
    const subscription = agUiEvents.subscribe({
      next(event) { onEvent?.(event); },
      error(error) { reject(error); },
      complete() { resolve(); },
    });
    signal?.addEventListener('abort', () => subscription.unsubscribe(), { once: true });
  });
}

// Uma rodada usa uma unica bolha; cada delta atualiza o mesmo elemento.
let currentAssistantBubble = null;
let ultimoRunPausado = null;

function setStatus(status) {
  document.getElementById('status').textContent = status;
}

function showCorrelationId(correlationId) {
  document.getElementById('correlation-id').textContent = correlationId;
}

function appendMessage(delta) {
  if (!currentAssistantBubble) {
    currentAssistantBubble = document.createElement('div');
    currentAssistantBubble.className = 'message message--assistant';
    document.getElementById('messages').appendChild(currentAssistantBubble);
  }
  currentAssistantBubble.textContent += delta;
}

function renderSources(sources) {
  const container = document.getElementById('sources');
  container.replaceChildren();
  if (!Array.isArray(sources)) return;
  for (const source of sources) {
    const item = document.createElement('p');
    item.textContent = typeof source === 'string'
      ? source
      : String(source.reference || source.title || source.document_title || 'Documento');
    container.appendChild(item);
  }
}

function mostrarPainelDeAprovacao({ mensagem, decisoesPermitidas, onDecidir }) {
  const container = document.getElementById('hil-actions');
  container.replaceChildren();
  container.hidden = false;

  const description = document.createElement('p');
  description.textContent = mensagem || 'A execucao aguarda uma decisao.';
  container.appendChild(description);

  for (const decisao of decisoesPermitidas) {
    const button = document.createElement('button');
    button.type = 'button';
    button.textContent = decisao;
    button.addEventListener('click', () => onDecidir(decisao));
    container.appendChild(button);
  }
}

/** Trata cada evento recebido — igual ao template, com a extensao de HIL adicionada. */
function renderEvent(event) {
  if (event.type === 'RUN_STARTED') {
    currentAssistantBubble = null;
    setStatus('running');
  }
  if (event.type === 'TEXT_MESSAGE_CONTENT' || event.type === 'TEXT_MESSAGE_CHUNK') {
    appendMessage(event.delta || event.content || '');
  }
  if (event.type === 'STATE_SNAPSHOT') {
    renderSources(event.snapshot?.sources);
  }
  if (event.type === 'RUN_FINISHED') {
    // Extensao de HIL (nao existe no template original): checar outcome.type
    // antes de tratar o run como concluido de verdade.
    if (event.outcome?.type === 'interrupt') {
      ultimoRunPausado = { threadId: event.threadId, runId: event.runId };
      handleHilInterrupt(event.outcome.interrupts); // AgUiInterrupt[] — ver abaixo
      return;
    }
    setStatus('completed');
  }
  if (event.type === 'RUN_ERROR') {
    setStatus('failed');
    appendMessage(event.message || 'Falha no run AG-UI.');
  }
}

/**
 * `interrupts` e uma lista de AgUiInterrupt (src/api/schemas/ag_ui_models.py):
 * { id, reason, message, toolCallId, responseSchema, expiresAt, metadata }.
 * `responseSchema` e um JSON Schema com a lista de decisoes permitidas em
 * `properties.decisions.items.enum` (contrato montado em
 * src/api/services/ag_ui_hil_protocol_mapper.py, funcao _build_response_schema).
 */
function handleHilInterrupt(interrupts) {
  for (const interrupt of interrupts) {
    const decisoesPermitidas =
      interrupt.responseSchema?.properties?.decisions?.items?.enum;
    if (!Array.isArray(decisoesPermitidas) || decisoesPermitidas.length === 0) {
      throw new Error('Interrupcao HIL sem decisoes permitidas no responseSchema.');
    }
    mostrarPainelDeAprovacao({
      mensagem: interrupt.message,
      decisoesPermitidas,
      onDecidir: (tipoDecisao) => enviarResume({
        // Falha fechado se o RUN_FINISHED nao trouxe a identidade da rodada pausada.
        threadId: ultimoRunPausado.threadId,
        parentRunId: ultimoRunPausado.runId,
        interruptId: interrupt.id,
        tipoDecisao,
      }),
    });
  }
}

/**
 * Retoma um run pausado. Contrato confirmado em AgUiRunRequest.resume
 * (lista de AgUiResumeInput: interruptId, status, payload) — a retomada e um
 * NOVO run (novo runId), referenciando o run pausado via parentRunId, exemplo
 * real do shape em README-TECNICO-AG-UI.md, secao 3.2.2.
 */
async function enviarResume({ threadId, parentRunId, interruptId, tipoDecisao }) {
  const payload = {
    threadId,
    runId: `${threadId}-resume-${Date.now()}`,
    parentRunId,
    input: {},
    resume: [
      {
        interruptId,
        status: 'resolved', // ou 'cancelled' para desistir da pausa
        // tipoDecisao: 'approve' | 'reject' | 'edit' | 'respond' (AgentResumeDecision)
        payload: { decisions: [{ type: tipoDecisao }] },
      },
    ],
  };
  return runOfficialAgUiStream({
    endpoint: '/api/ag-ui/runs',
    payload,
    onEvent: renderEvent,
    onCorrelationId: showCorrelationId,
  });
}

// --- Disparo do primeiro run ---
const controller = new AbortController();
await runOfficialAgUiStream({
  endpoint: '/api/ag-ui/runs',
  payload: {
    threadId: 'portal-terceiro',
    runId: `run-${Date.now()}`,
    state: {},
    messages: [
      {
        id: 'mensagem-usuario-1',
        role: 'user',
        content: 'Resuma vendas do dia.',
      },
    ],
    tools: [],
    context: [],
    forwardedProps: {
      capability: 'sales_summary',
      parameters: {},
      metadata: { surface: 'portal-parceiro' },
    },
  },
  onEvent: renderEvent,
  onCorrelationId: showCorrelationId,
  signal: controller.signal,
});
```

O browser envia um `RunAgentInput` publico somente ao BFF. O BFF valida chaves proibidas,
injeta `user_email`, `yaml_inline_content` ou `encrypted_data` e a `X-API-Key`, converte o
payload para `AgUiRunRequest` e entao chama a plataforma. `POST /ag-ui/runs` da plataforma
aceita o `AgUiRunRequest`; nao envie `RunAgentInput` puro diretamente para ele.

O BFF copiavel ja esta em
[`templates/ag-ui-official-third-party/backend/main.py`](../../templates/ag-ui-official-third-party/backend/main.py).
O endpoint `POST /api/ag-ui/runs` chama `PublicPayloadGuard.validate_run_payload(...)` e
`PrometeuAgUiProxy.open_run_stream(...)`; o proxy preserva o stream e o
`X-Correlation-Id`. Em producao, adapte a configuracao estatica do template para resolver
usuario e tenant a partir da sessao autenticada do parceiro, nunca de campos escolhidos
pelo browser.

O `input: {}` na retomada HIL permite ao BFF validar o envelope publico; a decisao real
viaja em `resume`. YAML e e-mail continuam sendo reinjetados no servidor.

### 3.6. Reconstrua com replay

Quando precisar reconstruir a tela depois de refresh, suporte ou auditoria, consulte:

```text
GET /ag-ui/runs/{run_id}/events
```

O replay e sanitizado no event store. Ele existe para reconstruir a experiencia sem reexpor segredos internos.

## 4. Guia de seguranca

### 4.1. Segredos

Segredo fica no servidor.

O browser pode receber uma configuracao publica, como lista de agentes permitidos e rotas do backend-for-frontend. Ele nao pode receber API key, DSN, YAML interno, catalogo de tools LangChain, credenciais de banco ou security keys.

### 4.2. Tools LangChain e frontendTools nao sao a mesma coisa

No boundary publico AG-UI, `RunAgentInput.tools` significa somente frontend tools.

Frontend tool e uma capacidade visual da interface, por exemplo abrir um painel aprovado. Ela nao e uma Tool LangChain da plataforma, nao passa pela ToolsFactory e nao autoriza banco, Redis, Qdrant, APIs internas ou SQL.

A regra pratica e simples:

1. Se a capability nao publicou allowlist em `frontendTools`, envie `tools: []`.
2. Se publicou allowlist, envie apenas nomes e schemas exatamente iguais ao discovery.
3. Se o frontend inventar uma tool, o backend deve rejeitar.
4. Se a tool trouxer SQL, DSN, connection string ou security keys, o backend deve rejeitar.

### 4.3. Threat model do RunAgentInput publico

Threat model significa mapa de ameacas. Em linguagem simples, e a lista do que pode dar errado se um cliente externo enviar campos perigosos no payload.

O endpoint publico aplica falha fechada para estes riscos:

| Campo do `RunAgentInput` | Risco principal                                                      | Regra pratica                                                                                    |
| ------------------------ | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `messages`               | texto muito grande ou tentativa de esconder segredo no prompt        | limite de quantidade e tamanho; DSN e chave privada sao bloqueados quando aparecem como valor    |
| `tools`                  | confundir frontend tool com Tool LangChain, banco ou SQL             | somente tools publicadas em `frontendTools` podem ser enviadas, com schema identico ao discovery |
| `context`                | transportar tenant, DSN, YAML ou segredo como contexto auxiliar      | chaves internas e sensiveis sao recusadas em qualquer profundidade                               |
| `forwardedProps`         | burlar governanca com `executionKind`, YAML, tenant ou security keys | apenas chaves publicas permitidas entram; qualquer chave fora da allowlist falha com HTTP 400    |
| `state`                  | esconder configuracao interna em estado inicial                      | chaves como `yamlPath`, `securityKeys`, `dsn`, `sql` e equivalentes sao recusadas                |
| `resume`                 | retomar uma pausa humana com contrato divergente                     | `resume` top-level e `forwardedProps.resume`, quando ambos existirem, precisam ser iguais        |

Tambem existem limites de tamanho e quantidade para reduzir abuso operacional. Isso evita que uma integracao use AG-UI como canal para payload gigante, segredo, SQL livre ou configuracao bruta da Plataforma de Agentes de IA.

Quando uma rejeicao acontece, o backend registra uma decisao estruturada em log com `ag_ui.public_payload.decision`, motivo da rejeicao, `agent_id` e contagens nao sensiveis. O log nao grava o segredo, o SQL nem o conteudo completo do payload.

O mesmo principio vale para UI generativa: uma spec visual externa nao entra direto no renderer Plataforma de Agentes de IA. Primeiro ela precisa ser convertida para o formato A2UI governado, restrito ao catalogo de componentes declarado no YAML (seção 1.1).

### 4.4. SQL e dados de varejo

SQL livre nao entra pelo browser.

Em cenarios de varejo e ERP, a tela envia uma intencao de negocio. O backend resolve isso usando capability governada, catalogo permitido e credencial segura no servidor.

Isso evita dois problemas práticos:

1. Expor estrutura interna do banco para o usuario final.
2. Permitir que uma interface externa execute consulta arbitraria.

### 4.5. Replay

Replay e ferramenta de reconstrucao e auditoria, nao canal de vazamento.

O event store sanitiza payloads para reduzir risco de segredo, SQL livre e identificadores internos aparecerem em evento publico. Mesmo assim, o integrador deve tratar replay como dado operacional sensivel: autentique, autorize e mostre apenas para usuarios com permissao.

### 4.6. HIL e resume

HIL significa human in the loop. Em linguagem simples, e quando a IA pausa e pede decisao humana.

Quando a capability indicar suporte, a pausa aparece como `RUN_FINISHED` com `outcome` de interrupcao. A retomada volta pelo mesmo endpoint de run, usando `resume` no `RunAgentInput`.

## 5. Varejo e vendas

O dominio de varejo e a vitrine mais madura do AG-UI atual.

O caso tipico e este:

1. A tela pede uma capability de negocio, como resumo de vendas ou analise operacional.
2. O backend aplica regras governadas.
3. A execucao publica eventos AG-UI.
4. A interface mostra status, mensagens, ferramentas visuais e estado.
5. O replay permite reconstruir o que aconteceu.

Use o discovery antes de hardcodar qualquer capability. O que existe em um ambiente pode variar conforme configuracao, permissao e catalogo ativo.

Documentos complementares:

1. [README-CONCEITUAL-AG-UI-DOMINIO-VAREJO-DEMO.md](../conceitual/README-CONCEITUAL-AG-UI-DOMINIO-VAREJO-DEMO.md)
2. [README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md](../tecnico/README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md)

## 6. Migracao e adaptacao de ERP

Para adaptar AG-UI a um ERP ou PDV existente, evite comecar pela tela. Comece pelo contrato de negocio.

Sequencia recomendada:

1. Liste as telas em que IA deve ajudar o usuario.
2. Para cada tela, escreva a intencao de negocio em linguagem simples.
3. Converta cada intencao em capability governada no backend.
4. Mantenha DSN, SQL, catalogos e credenciais no servidor.
5. Exponha somente `agent_id`, capability, parametros permitidos e frontendTools aprovadas.
6. Consuma eventos AG-UI e desenhe a experiencia no design system do ERP.
7. Use replay para suporte e auditoria.

Exemplo pratico: uma tela de fechamento de caixa nao deve enviar SQL ao AG-UI. Ela deve enviar algo como "analisar divergencias do turno" com parametros permitidos. O backend decide quais consultas aprovadas ou workflows internos sao usados.

## 7. Matriz de eventos suportados no boundary Plataforma de Agentes de IA

Esta matriz lista os eventos que o schema Plataforma de Agentes de IA envolve diretamente hoje em [src/api/schemas/ag_ui_models.py](../../src/api/schemas/ag_ui_models.py).

| Grupo               | Eventos                                                                                     | Uso pratico na UI                                                                                                              |
| ------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Lifecycle           | `RUN_STARTED`, `RUN_FINISHED`, `RUN_ERROR`                                                  | Mostrar inicio, sucesso, pausa humana ou erro terminal.                                                                        |
| Etapas              | `STEP_STARTED`, `STEP_FINISHED`                                                             | Mostrar progresso por etapa quando o adapter emitir etapas.                                                                    |
| Texto               | `TEXT_MESSAGE_START`, `TEXT_MESSAGE_CONTENT`, `TEXT_MESSAGE_END`, `TEXT_MESSAGE_CHUNK`      | Renderizar mensagens incrementais.                                                                                             |
| Tool call           | `TOOL_CALL_START`, `TOOL_CALL_ARGS`, `TOOL_CALL_END`, `TOOL_CALL_RESULT`, `TOOL_CALL_CHUNK` | Mostrar chamada, argumentos e resultado de tool quando o runtime publicar esse detalhe.                                        |
| Estado              | `STATE_SNAPSHOT`, `STATE_DELTA`                                                             | Substituir ou alterar estado visual local. `STATE_DELTA` usa operacoes de patch.                                               |
| Mensagens           | `MESSAGES_SNAPSHOT`                                                                         | Reconstituir lista de mensagens quando houver snapshot completo.                                                               |
| Atividade           | `ACTIVITY_SNAPSHOT`, `ACTIVITY_DELTA`                                                       | Renderizar timeline e progresso operacional.                                                                                   |
| Extensao controlada | `CUSTOM`, `RAW`                                                                             | Transportar eventos especificos quando o adapter precisar, com sanitizacao e validacao do lado da Plataforma de Agentes de IA. |

Eventos adicionais existem no pacote oficial AG-UI instalado, mas nao devem ser prometidos como contrato Plataforma de Agentes de IA sem confirmacao no schema, adapter e testes.

## 8. Checklist antes de publicar uma integracao

Antes de colocar uma integracao AG-UI em producao, confirme:

1. O browser nao recebe API key, DSN, YAML, tenant interno, security keys ou SQL livre.
2. O endpoint usado pela plataforma para execucao e `POST /ag-ui/runs`.
3. O payload produtivo segue `AgUiRunRequest`.
4. `RunAgentInput.tools` esta vazio ou bate exatamente com `frontendTools` do discovery.
5. O backend-for-frontend injeta a fonte de configuracao confiavel no servidor.
6. O frontend consome SSE por POST, nao por GET `EventSource` simples.
7. O replay exige autenticacao e autorizacao.
8. Erros exibem mensagem operacional clara, sem fallback silencioso.
9. O design system renderiza eventos e estado sem interpretar HTML livre.
10. O fluxo foi validado com o template oficial e com o replay.

### 8.1. Checklist de novo componente visual

Se a integracao vai introduzir um componente novo na UI generativa da Plataforma de Agentes de IA, confirme tambem:

1. O componente foi registrado no Component Catalog da Plataforma de Agentes de IA, nao em JSON arbitrario solto.
2. Props, actions e bindings ficaram em allowlist explicita.
3. O cadastro novo continua bloqueando HTML, JavaScript, SQL livre, DSN, segredo e `correlation_id`.
4. Existe teste backend/frontend provando que apenas payload validado pelo Component Catalog (ou, no caso de A2UI, componente do catálogo fechado de 8 declarado no YAML) chega na materializacao.
5. Existe teste frontend provando que o Component Catalog e o renderer rejeitam payload fora da allowlist.

Esse checklist evita que um componente novo vire uma excecao escondida no contrato visual.

## 9. Arquivos que provam este contrato

Este guia foi escrito com base no fluxo executavel atual, nao apenas em intencao de produto.

Evidencias principais (paths a partir da raiz do repositorio; corrigidos nesta rodada — apontavam para `../src/...`/`../templates/...`/`../tests/...`, um nivel raso demais a partir de `docs/usuario/`):

1. [src/api/routers/ag_ui_router.py](../../src/api/routers/ag_ui_router.py) — as 5 rotas reais (`capabilities`, `runs`, `copilotkit/runs`, `runs/{run_id}/events`, `threads/{thread_id}/events`)
2. [src/api/schemas/ag_ui_models.py](../../src/api/schemas/ag_ui_models.py) — `AgUiRunRequest`/`AgUiStrictModel` bloqueiam campo desconhecido no topo do payload; `input`/`metadata` não têm varredura recursiva comprovada (lacuna registrada em [README-TECNICO-AG-UI-BORDA-HTTP-DEDICADA.md, seção 9](../tecnico/README-TECNICO-AG-UI-BORDA-HTTP-DEDICADA.md)); `AgUiResumeInput`, `AgUiInterrupt`, `AgUiRunInterruptOutcome` (contrato de HIL usado nas seções 3.5.1 e 2.2)
3. [src/api/services/ag_ui_yaml_first_run_preparation_service.py](../../src/api/services/ag_ui_yaml_first_run_preparation_service.py)
4. [src/api/services/ag_ui_frontend_tool_policy.py](../../src/api/services/ag_ui_frontend_tool_policy.py)
5. [src/api/services/ag_ui_event_store.py](../../src/api/services/ag_ui_event_store.py)
6. [src/api/services/ag_ui_adapter_registry.py](../../src/api/services/ag_ui_adapter_registry.py)
7. [src/api/services/ag_ui_langgraph_agent_factory.py](../../src/api/services/ag_ui_langgraph_agent_factory.py) — `AgUiLangGraphExecutionAdapter._build_run_input`: decide entre sintetizar `RunAgentInput` a partir de `context.input` ou revalidar `context.protocol_input` inteiro (caso CopilotKit), sem recorte antes de `LangGraphAgent.run(...)` (base da seção 2.2)
8. [src/api/services/copilotkit_ag_ui_compat_service.py](../../src/api/services/copilotkit_ag_ui_compat_service.py) — traducao `RunAgentInput` (CopilotKit) -> `AgUiRunRequest` da secao 2.1; `build_run` nunca popula `AgUiRunRequest.resume` (lacuna de HIL documentada na seção 2.2)
9. [src/api/services/ag_ui_deepagent_adapter.py](../../src/api/services/ag_ui_deepagent_adapter.py) — `_build_a2ui_schema_context_entries`: injeção do catálogo A2UI independe do valor de `chat_renderer` (seção 2.2, "Generative UI")
10. [src/api/services/ag_ui_hil_protocol_mapper.py](../../src/api/services/ag_ui_hil_protocol_mapper.py) e [src/api/services/ag_ui_runtime_adapter_support.py](../../src/api/services/ag_ui_runtime_adapter_support.py) — mapeamento de interrupção LangGraph para `AgUiInterrupt`/`AgUiRunInterruptOutcome` e `execute_agentic_resume` (contrato de `resume` usado na seção 3.5.1)
11. [templates/ag-ui-official-third-party](../../templates/ag-ui-official-third-party) — `frontend/src/ag-ui-client.js` (`runOfficialAgUiStream`, `buildRunAgentInput`) e `frontend/src/main.js` (`renderEvent`); o frontend do template **não** trata `RUN_FINISHED.outcome`/interrupt (base do exemplo estendido na seção 3.5.1)
12. [tests/unit/test_02-01-52_ag_ui_third_party_template_contract.py](../../tests/unit/test_02-01-52_ag_ui_third_party_template_contract.py)
13. [tests/unit/test_02-01-48_ag_ui_router.py](../../tests/unit/test_02-01-48_ag_ui_router.py)
14. [tests/unit/test_02-01-100_ag_ui_copilotkit_compat.py](../../tests/unit/test_02-01-100_ag_ui_copilotkit_compat.py) — prova a secao 2.1 (traducao + wiring da rota) até o `AgUiRunContext` entregue ao orchestrator; não prova a cadeia até o grafo compilado do DeepAgent (limite registrado na seção 2.2, "Shared state")
15. [tests/unit/test_02-01-38_ag_ui_event_store.py](../../tests/unit/test_02-01-38_ag_ui_event_store.py)

## 10. Proximos passos de leitura

1. Para implementar rapido: templates/ag-ui-official-third-party/README.md
2. Para entender o boundary: [README-TECNICO-AG-UI-BORDA-HTTP-DEDICADA.md](../tecnico/README-TECNICO-AG-UI-BORDA-HTTP-DEDICADA.md)
3. Para entender replay: [README-TECNICO-AG-UI-REPLAY-E-AUDITORIA.md](../tecnico/README-TECNICO-AG-UI-REPLAY-E-AUDITORIA.md)
4. Para runtime frontend: [README-TECNICO-AG-UI-RUNTIME-COMPARTILHADO-DO-FRONTEND.md](../tecnico/README-TECNICO-AG-UI-RUNTIME-COMPARTILHADO-DO-FRONTEND.md)
5. Para varejo/vendas: [README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md](../tecnico/README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md)
