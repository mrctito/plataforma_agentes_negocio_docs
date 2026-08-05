# Tutorial 101: Generative UI (AG-UI) — Como fazer o DeepAgent desenhar gráficos e painéis no chat

> Leitura recomendada antes: [GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md](GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md) para entender o componente de chat.
> Referência técnica: [README-TECNICO-AG-UI.md](../tecnico/README-TECNICO-AG-UI.md).
> Visão de produto: [README-CONCEITUAL-AG-UI.md](../conceitual/README-CONCEITUAL-AG-UI.md).

---

## O que é Generative UI nesta plataforma

"Generative UI" (interface gerada pelo agente) significa que o DeepAgent, em vez de responder só com texto, pode desenhar de verdade um card, uma tabela ou um gráfico dentro da própria bolha do chat — quando o usuário pedir uma visualização.

O nome "AG-UI" (Agent-Generated UI) é a sigla usada internamente. O mecanismo atual, provado em produção, chama-se **A2UI**: o supervisor DeepAgent ganha uma ferramenta (`generate_a2ui`) que devolve um envelope declarativo descrevendo os componentes a desenhar (`{a2ui_operations: [createSurface, updateComponents]}`), e o frontend transforma esse envelope em DOM real usando um catálogo fechado de componentes seguros.

**Por que isso importa?** Porque a alternativa seria o agente devolver texto e cada integrador ter que "adivinhar" quando havia um número para exibir, ou pior, o agente devolver HTML/JavaScript livre — um risco de segurança sério. Com A2UI, o agente nunca manda marcação ou script: ele só escolhe, entre um catálogo pré-aprovado, quais componentes usar e com quais dados (que ele já obteve de tools reais, nunca inventados).

**A característica mais importante do mecanismo: é aditivo e condicional.**

- **Aditivo:** se o YAML do supervisor não declarar o bloco `ag_ui.generative`, nada muda — o DeepAgent responde exatamente como sempre respondeu, texto puro, byte a byte.
- **Condicional:** mesmo com o bloco declarado, o gráfico só aparece **quando o usuário pede uma visualização**. A mesma pergunta feita sem pedir gráfico continua vindo em texto normal. Não existe prompt fixo forçando desenho — a decisão de desenhar ou não é do modelo, orientada por regras explícitas no prompt do supervisor (ver Passo 3).

---

## As duas superfícies AG-UI desta plataforma

Existem dois caminhos para usar AG-UI nesta plataforma. Confundi-los é a fonte de dúvida mais comum.

| | Superfície A — Chat embutível (`generate_a2ui`) | Superfície B — Telas operacionais dedicadas |
|---|---|---|
| **Como funciona** | O componente `PrometeuEmbeddableChatRuntime` abre, de forma **opt-in**, uma conexão SSE com `POST /ag-ui/runs` quando o modo é `deepagent`. O envelope A2UI chega no resultado da tool `generate_a2ui` e é desenhado dentro da bolha do assistente. | Uma tela dedicada (ex.: hub de varejo) chama `POST /ag-ui/runs` diretamente para um **capability pack** governado (ex.: `retail_demo`) que expõe consultas pré-aprovadas como eventos SSE, com painel lateral e timeline própria. |
| **Para quem é** | Quem já usa (ou vai usar) o WebChat v3 / componente `PrometeuEmbeddableChatRuntime` e quer que o próprio chat desenhe visualizações. | Quem precisa de uma tela operacional dedicada, fora do chat, com múltiplas capabilities de negócio e navegação própria. |
| **Configuração** | Só YAML: bloco `multi_agents[].ag_ui.generative` no supervisor DeepAgent. | Capability pack no backend (código) + tela frontend dedicada. |
| **Formato de resposta** | Stream SSE de `/ag-ui/runs`, consumido internamente pelo transporte do componente de chat; o resultado final aparece como uma bolha normal com a superfície A2UI desenhada dentro. | Stream SSE de `/ag-ui/runs` consumido diretamente pela tela (eventos `RUN_STARTED`, `TOOL_CALL_*`, `STATE_SNAPSHOT`, `RUN_FINISHED`...). |
| **Este tutorial cobre** | **Sim — é o assunto central deste documento.** | Não em profundidade; ver [README-TECNICO-AG-UI.md](../tecnico/README-TECNICO-AG-UI.md) para o boundary de capability packs e discovery. |

A maior parte dos casos de uso de consultor e integrador cai na **Superfície A**. Este tutorial foca nela.

---

## Parte 1 — Tutorial passo a passo: "quero que meu DeepAgent responda com gráficos quando o usuário pedir"

### Os 3 papéis envolvidos

Gerar uma visualização no chat envolve três papéis diferentes. Entender a fronteira entre eles evita a maior parte das dúvidas de configuração:

| Papel | Quem faz | O que faz |
|---|---|---|
| **1. Quem gera** | O próprio **supervisor DeepAgent** — não um subagente separado. Quando o bloco `ag_ui.generative` existe no YAML, a plataforma binda a ferramenta `generate_a2ui` diretamente no supervisor. | Depois que um especialista já trouxe os dados (via `task` + `dyn_sql`), o supervisor decide criar a superfície visual e chama `generate_a2ui` compondo componentes do catálogo com os números que já estão na conversa. |
| **2. Quem entrega** | A camada de wiring do servidor (adapter + factory AG-UI). | Injeta o catálogo de componentes declarado no YAML (`a2ui_schema`) no estado do grafo (`state["ag-ui"]["a2ui_schema"]`), para a tool saber quais componentes tem à disposição; depois, transporta o envelope produzido pela tool como resultado da chamada (`TOOL_CALL_RESULT`) no stream SSE de `/ag-ui/runs`. |
| **3. Quem desenha** | O **frontend** — o transporte SSE do componente de chat + o renderer A2UI. | Detecta o envelope `{a2ui_operations}` no stream, valida contra um catálogo fechado de componentes seguros e monta o DOM real (cards, tabelas, gráficos). Componente fora do catálogo ou envelope malformado → cai em texto (falha fechada, nunca quebra a tela). |

Em uma frase: **o supervisor decide e compõe; o servidor entrega o catálogo e transporta o resultado; o frontend valida e desenha.** Em nenhum desses três papéis a IA manda HTML, script ou SQL livre — ela só escolhe nomes de componentes de uma lista fechada e reaproveita dados que tools já buscaram.

> **De onde vêm os dados?** Nunca da tool `generate_a2ui` em si — ela **lê o histórico da conversa** (resultados de tools já chamadas pelos especialistas via `task`). Por isso a ordem importa: primeiro o supervisor obtém os dados delegando para um especialista, só depois chama `generate_a2ui`. O prompt do supervisor reforça essa ordem e proíbe explicitamente inventar números.

### Passo 1 — Ter um DeepAgent com especialistas que buscam dado real

O bloco `ag_ui.generative` vive dentro de um supervisor `execution.type: deepagent`. Para o gráfico ter o que desenhar, o supervisor precisa de pelo menos um subagente especialista com tools `dyn_sql` (ou equivalente) capaz de trazer números reais — a mesma estrutura de qualquer DeepAgent funcional, sem nada especial. O demo `app/yaml/rag-config-pdv-vendas-demo.yaml` tem vários (`subdominio_vendas`, `subdominio_catalogo` etc.).

### Passo 2 — Declarar o bloco `ag_ui.generative` no supervisor

Exemplo real, tirado de `app/yaml/rag-config-pdv-vendas-demo.yaml`:

```yaml
multi_agents:
  - id: ag_ui_pdv_vendas_supervisor
    execution:
      type: deepagent          # espinha dorsal DeepAgent (obrigatório)
    ag_ui:
      generative:
        chat_renderer: jspuro  # jspuro (chat embutível JS) ou copilotkit
        a2ui_schema:
          catalog_id: "https://plataforma.local/catalogs/pdv-vendas-a2ui.json"
          components:
            - Card
            - Column
            - Row
            - Text
            - BarChart
            - LineChart
            - DataTable
            - Divider
      ui_specs: []              # conceito distinto (UISpec governada), não usado aqui
    agents:
      - id: subdominio_vendas
        # ... especialistas com tools dyn_sql ...
```

Campos do bloco (contrato completo lido em `src/agentic_layer/supervisor/ag_ui_generative_config.py`):

- **`chat_renderer`** (obrigatório): `jspuro` para o renderer JS nativo do chat embutível, ou `copilotkit` para repassar o envelope A2UI, sem filtro, a um cliente React que usa o SDK CopilotKit. Ver "Eixo — trocar de renderer" mais abaixo.
- **`a2ui_schema.catalog_id`** (obrigatório): um rótulo/URI estável do catálogo. **Ninguém faz fetch dele** — é só um identificador que o frontend casa com o mesmo id; ele não precisa resolver para nada.
- **`a2ui_schema.components`** (obrigatório, lista não vazia): os nomes de componente que o supervisor tem autorização de usar ao montar a superfície. Cada item pode ser um nome simples (string) ou uma definição mais rica (dict) — o parser de runtime aceita as duas formas.

> **Ausência do bloco = comportamento intocado.** Se `ag_ui` nem existir no supervisor, ou existir sem a chave `generative`, o DeepAgent responde exatamente como respondia antes — nenhuma tool nova é bindada, nenhum prompt extra é anexado.

### Passo 3 — Por que o gráfico só aparece quando pedido (o contrato de saída do prompt)

Quando o bloco existe, a plataforma anexa ao prompt do supervisor um bloco de regras curto e direto (pensado para funcionar até com modelos pequenos), resumido assim:

- Se o usuário pediu gráfico/dashboard/visualização, a resposta só é válida **depois** de chamar `generate_a2ui` — responder só texto nesse caso é tratado como erro.
- Ordem obrigatória: primeiro obter os dados (o mínimo de delegações possível), só depois chamar `generate_a2ui` **uma única vez**, e encerrar com uma frase curta.
- Desenhar é responsabilidade exclusiva do supervisor via `generate_a2ui` — os especialistas (chamados via `task`) **nunca** têm acesso a essa ferramenta e nunca devem receber a instrução de "desenhar".
- Se o usuário **não** pediu visualização, o supervisor responde em texto normal e não chama `generate_a2ui`.
- Nunca inventar números: a superfície só pode usar dados que já apareceram no histórico da conversa.

É esse contrato — não uma heurística de UI — que garante a ortogonalidade: mesma pergunta, sem pedir gráfico, sempre em texto; pedindo gráfico, sempre com `generate_a2ui` chamada.

### Passo 4 — Testar

Com o YAML configurado e o agente selecionado no WebChat v3 (ou na bancada `ui-embeddable-chat-test.html`) com o transporte SSE ligado (ver "Como habilitar no host" abaixo):

1. Faça uma pergunta **sem** pedir visualização (ex.: "qual o faturamento do mês?"). Deve vir texto normal.
2. Faça uma pergunta pedindo visualização (ex.: "monte um gráfico de vendas por loja"). Deve aparecer um card com o gráfico desenhado na própria bolha.
3. Pelo `correlation_id` da execução, a ordem esperada no log é: `task` (especialista busca dado) → `dyn_sql` (query real) → `generate_a2ui` (supervisor desenha) → `render_a2ui` (frontend consome o resultado).

### Como habilitar o transporte no host (Superfície A)

O bloco `ag_ui.generative` no YAML **não é suficiente sozinho** para o chat embutível desenhar — a tela host também precisa ligar o transporte SSE opt-in na configuração do componente:

```javascript
const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
  mode: 'deepagent',          // obrigatório para gerar e transportar A2UI
  chatRenderer: 'jspuro',     // precisa bater com o chat_renderer do YAML
  agUiSseTransport: true,     // opt-in explícito do transporte SSE
  // ... demais configs (yaml, email, apiKey) ...
});
```

Para **A2UI**, os três valores acima são obrigatórios porque a geração visual pertence ao
supervisor DeepAgent. Isso não significa que o transporte SSE seja exclusivo do A2UI:
Q&A/RAG também usa SSE com `mode: 'qa'`, e um host autorizado pode usá-lo com
`projectKey`. `workflow`, agente clássico e `chatRenderer: 'copilotkit'` no componente
continuam fora desse gate. Sem DeepAgent + `jspuro` + flag, o A2UI não ativa mesmo com o
YAML correto.

Para texto progressivo sem visualização, veja o
[Tutorial 101 de chat com streaming](TUTORIAL-CHAT-PLATAFORMA.md).

Os scripts que a página host precisa carregar estão na seção "Como carregar os scripts" mais abaixo.

---

## Parte 2 — Como o painel de capacidades funciona (CapabilitiesSpec)

Quando o usuário pergunta "o que você faz?" ou "sobre o que posso te perguntar?", um supervisor DeepAgent responde automaticamente com um painel visual de capacidades — cards com grupos de assuntos e chips clicáveis de perguntas-exemplo. **Este mecanismo é independente do A2UI** — não depende do bloco `ag_ui.generative` nem do transporte SSE.

Isso acontece porque a tool builtin `descrever_capacidades` é auto-injetada em todo supervisor DeepAgent. O painel lê os subagentes declarados em `multi_agents[].agents[]` do YAML e monta os cards a partir das descrições amigáveis de cada subagente — por isso a qualidade do painel depende da qualidade dos campos `name` e `description` dos seus subagentes.

O painel nunca expõe nome interno de tool, id de subagente, SQL ou qualquer dado técnico; um validador no backend barra vazamento antes de emitir o spec.

---

## Parte 3 — O catálogo de componentes A2UI (renderer `jspuro`)

O renderer nativo do chat embutível (`ag-ui-a2ui-surface-renderer.js`) é **fail-closed com um catálogo fechado de 8 componentes**. A superfície inteira só é desenhada quando toda a árvore de componentes é reconhecida; qualquer componente fora da lista, ou envelope malformado, faz o componente inteiro cair em texto — nunca desenha "pela metade".

| Componente | Categoria | O que é |
|---|---|---|
| `Card` | layout | contêiner visual com borda/agrupamento |
| `Column` | layout | empilha filhos verticalmente |
| `Row` | layout | organiza filhos lado a lado |
| `Text` | conteúdo | texto simples |
| `Divider` | conteúdo | linha separadora |
| `BarChart` | gráfico | gráfico de barras (via ChartAdapter + ApexCharts) |
| `LineChart` | gráfico | gráfico de linha (via ChartAdapter + ApexCharts) |
| `DataTable` | conteúdo | tabela com linhas e colunas |

Os gráficos (`BarChart`/`LineChart`) reaproveitam a mesma **porta neutra de gráfico** (`ChartAdapter`, `ag-ui-chart-adapter.js`) usada por outras superfícies AG-UI da plataforma — o renderer A2UI não depende diretamente do ApexCharts; se o adapter concreto não estiver carregado, o gráfico simplesmente não valida, e a superfície cai em texto (nunca quebra a tela).

O array `components` do bloco `ag_ui.generative.a2ui_schema` no YAML declara **quais desses componentes o supervisor pode usar** — normalmente você declara todos os 8, como no demo, mas pode restringir a um subconjunto se quiser limitar o vocabulário visual daquele agente.

> **Diferença entre parser e renderer:** o parser de runtime (backend) que lê o YAML é **permissivo** — aceita qualquer nome de componente não vazio, porque quem decide o vocabulário renderizável de verdade é o **frontend**. Declarar no YAML um nome fora dos 8 acima não quebra o parse, mas a IA nunca vai conseguir desenhá-lo de fato (o renderer recusa e cai em texto).

---

## Parte 4 — Como dar bind em campos da tela e mandar junto com a pergunta

"Bind" aqui significa: quando o usuário digitar uma pergunta e apertar Enviar, a plataforma envia ao backend não só a pergunta digitada, mas também o contexto da tela — por exemplo, o projeto aberto, os arquivos marcados, ou o período selecionado. O usuário vê a pergunta limpa na bolha; o agente recebe a pergunta + contexto enriquecido. Este mecanismo é independente do A2UI e funciona em qualquer modo do componente.

### O hook `buildPayloadText`

O componente `PrometeuEmbeddableChatRuntime` expõe um hook chamado `buildPayloadText`. Você passa esse hook na criação do componente, na sua página host. A cada envio feito pelo input embutido (botão Enviar ou tecla Enter), o componente chama sua função passando o texto digitado e usa o texto que você devolver como o payload enviado ao backend.

```javascript
// Exemplo conceitual — na SUA página host
const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
  // ... outras configs (yaml, email, apiKey, modo) ...
  buildPayloadText: (perguntaDigitada) => {
    const contexto = obterContextoAtualDaTela();  // função da SUA página
    if (!contexto) return perguntaDigitada;       // sem contexto: envia normal
    return contexto + '\n\nPergunta do usuário:\n' + perguntaDigitada;
  },
});
```

A bolha exibe `perguntaDigitada`. O backend recebe o texto completo com o contexto.

### Exemplo real: tela de projetos DNIT

A tela `app/ui/static/js/gesdoc-project-detail.js` usa exatamente esse hook. Quando o usuário faz uma pergunta na aba de chat de um projeto DNIT, o componente enriquece automaticamente o payload com o título do projeto aberto, os arquivos marcados pelo usuário na lista de documentos e trechos de contexto selecionados:

```javascript
// registro do hook na criação do componente (gesdoc-project-detail.js ~L1483)
buildPayloadText: (pergunta) => this.chatComporPayloadText(pergunta),

// composição do texto enriquecido (~L1608-1614)
chatComporPayloadText(pergunta) {
  const contexto = (this._chatContextoSessao || '').trim();
  const limpa = (pergunta || '').trim();
  if (!contexto) return limpa;
  return contexto + '\n\nPergunta do usuário:\n' + limpa;
}
```

O `_chatContextoSessao` é montado previamente pela tela. O usuário não vê esse contexto na bolha — ele aparece apenas no log interno com o `correlation_id`, para auditoria.

### Alternativa por chamada: `payloadText` direto

Se você chama `perguntar()` programaticamente (não pelo input embutido), pode passar o payload enriquecido diretamente:

```javascript
await chat.perguntar('Qual o status do contrato?', {
  payloadText: 'Projeto: Duplicação BR-050\nArquivos: contrato_2024.pdf\n\nQual o status do contrato?'
});
```

A bolha exibe "Qual o status do contrato?". O backend recebe o texto completo.

---

## Parte 5 — Como expandir

### Eixo 1 — Trocar de renderer: `jspuro` × `copilotkit`

`chat_renderer` no bloco `ag_ui.generative` escolhe **quem desenha** o envelope A2UI:

- **`jspuro`**: o renderer nativo descrito na Parte 3, embutido no componente `PrometeuEmbeddableChatRuntime`. Catálogo fechado de 8 componentes, fail-closed.
- **`copilotkit`**: a rota de compatibilidade `POST /ag-ui/copilotkit/runs` repassa o envelope A2UI **sem filtro** para um cliente React que usa o SDK oficial CopilotKit (`@ag-ui/a2ui-middleware`, já embutido no runtime deles). Use quando o consumidor final é uma aplicação React que já integra CopilotKit, em vez do chat embutível JS puro desta plataforma.

O valor de `chat_renderer` não muda o que o supervisor gera (o envelope A2UI é o mesmo); ele documenta e sinaliza qual frontend é o consumidor esperado.

### Eixo 2 — Restringir ou ampliar o vocabulário visual

Edite a lista `a2ui_schema.components` do YAML para conter só o subconjunto dos 8 componentes que faz sentido para aquele agente. Isso não muda o catálogo do renderer (que continua fixo), apenas o que você **autoriza** o supervisor a escolher para aquele domínio.

### Eixo 3 — Adicionar um componente novo ao catálogo (requer código)

O catálogo de 8 componentes é fechado por decisão de segurança — não é configurável só por YAML. Ampliá-lo é evolução de código: é preciso estender `SUPPORTED_COMPONENTS` e a lógica de render em `app/ui/static/js/shared/ag-ui-a2ui-surface-renderer.js` (frontend), e — se o novo componente for um gráfico — também o mapeamento de tipo em `app/ui/static/js/shared/ag-ui-chart-adapter.js`. O parser de runtime (`ag_ui_generative_config.py`) já é permissivo e não precisa mudar.

### Eixo 4 — Superfície B: telas operacionais dedicadas

Para uma experiência fora do chat — painel lateral, timeline de eventos, múltiplas capabilities de negócio encadeadas — o caminho é um capability pack dedicado consumindo `POST /ag-ui/runs` diretamente (Superfície B da tabela acima), não o mecanismo `ag_ui.generative` deste tutorial. Documentação técnica completa: [README-TECNICO-AG-UI.md](../tecnico/README-TECNICO-AG-UI.md).

---

## Como carregar os scripts numa tela host nova (Superfície A)

Se você vai criar uma página HTML nova e quer que ela suporte A2UI no chat embutível, carregue os scripts **nesta ordem** (ordem real comprovada em `ui-embeddable-chat-test.html`, a bancada de testes do componente):

```html
<!-- 1. ApexCharts (vendor) + porta neutra de gráfico. Dependência OPCIONAL: sem
     ela, gráficos não validam e a superfície cai em texto. -->
<script src="/ui/static/js/vendor/apexcharts.min.js?v=5.14.0"></script>
<script src="/ui/static/js/shared/ag-ui-chart-adapter.js"></script>
<script src="/ui/static/js/shared/ag-ui-chart-adapter-apexcharts.js"></script>

<!-- 2. Renderer A2UI (fail-closed, catálogo fechado de 8 componentes). -->
<script src="/ui/static/js/shared/ag-ui-a2ui-surface-renderer.js"></script>

<!-- 3. Detecção de spec (CapabilitiesSpec + A2UI) + registry de renderizadores. -->
<script src="/ui/static/js/shared/embeddable-chat-spec-runtime.js"></script>
<!-- 4. Bridge ESM: liga os renderizadores (inclusive A2UI) e publica em window. -->
<script type="module" src="/ui/static/js/shared/ag-ui-spec-render-bridge.js"></script>

<!-- 5. Transporte SSE opt-in. Para A2UI: deepagent + jspuro + flag.
     O mesmo transporte também atende texto em qa ou por projectKey. -->
<script src="/ui/static/js/shared/embeddable-chat-ag-ui-transport.js"></script>
<script type="module" src="/ui/static/js/shared/ag-ui-embeddable-transport-bridge.js"></script>

<!-- 6. Por fim, o componente de chat. -->
<script src="/ui/static/js/shared/embeddable-chat-runtime.js"></script>
```

A ordem importa porque cada script depende do anterior (o bridge de passo 4 lê `window.AgUiA2uiSurfaceRenderer` e `window.AgUiChartAdapter`, publicados pelos passos 1-2). O `renderStructured` do componente vem **ligado por padrão**; para forçar texto puro numa host específica, passe `renderStructured: false` na configuração.

O WebChat v3 (`ui-webchat-v3.html`) já inclui tudo isso — se você parte dele, não precisa repetir.

---

## Por que às vezes aparece texto em vez do gráfico

O componente tem um **fallback duro**: se qualquer coisa der errada no render estruturado, ele mostra o texto/JSON bruto da resposta em vez de travar. Isso é por design.

Razões reais para isso acontecer:

1. **O usuário não pediu visualização.** Comportamento correto e esperado — o contrato de saída do supervisor (Passo 3) só chama `generate_a2ui` quando o pedido é de visualização.
2. **YAML sem o bloco `ag_ui.generative`.** Sem o bloco, a tool nem existe para o supervisor — sempre texto.
3. **Transporte SSE não habilitado no host.** Sem `agUiSseTransport: true` + `mode: 'deepagent'` + `chatRenderer: 'jspuro'` na configuração do componente, o chat segue no caminho síncrono de sempre e nunca vê o envelope A2UI.
4. **Componente fora do catálogo dos 8, ou envelope malformado.** O renderer é fail-closed: qualquer parte não reconhecida derruba a superfície inteira para texto.
5. **Dados que a LLM não tinha no histórico.** O prompt proíbe inventar números; se o especialista não trouxe dado suficiente, o supervisor tende a responder em texto em vez de desenhar com dados incompletos.
6. **Scripts fora de ordem ou ausentes.** Se `ag-ui-spec-render-bridge.js` ou `ag-ui-embeddable-transport-bridge.js` não carregaram (ou vieram fora de ordem), o runtime de spec ou o transporte não existem em `window` e o componente cai em texto. Verifique o console do navegador.
7. **`renderStructured: false`** foi passado explicitamente na configuração do host.

Para confirmar pelo log: a ordem esperada de eventos numa execução com gráfico é `task` → `dyn_sql` → `generate_a2ui` → `render_a2ui`, pelo `correlation_id` daquela pergunta.

---

## FAQ 101

**P: Preciso mexer no código da plataforma para usar A2UI?**
R: Não. É um bloco YAML (`ag_ui.generative`) no supervisor DeepAgent + três flags na configuração do componente de chat (`mode`, `chatRenderer`, `agUiSseTransport`). Nenhuma alteração no core.

**P: O A2UI substitui algum recurso antigo de dashboard?**
R: Sim — este é o único mecanismo de generative UI no chat hoje. Um mecanismo anterior, baseado num subagente de saída dedicado injetado por uma flag de middleware e num contrato de spec de dashboard versionado, existiu no passado e foi removido do código; qualquer documentação ou exemplo que ainda descreva esse fluxo está desatualizado.

**P: O supervisor precisa de um subagente especial para desenhar?**
R: Não. `generate_a2ui` é bindada **diretamente no supervisor**, não num subagente separado. Os subagentes especialistas continuam só buscando dado (via `dyn_sql`); quem desenha é sempre o supervisor.

**P: Por que a mesma pergunta às vezes vem com gráfico e às vezes só texto?**
R: Porque a decisão de desenhar depende do **pedido do usuário**, nunca de um prompt fixo. Peça explicitamente "monte um gráfico"/"mostre num dashboard" para acionar `generate_a2ui`; pergunta neutra vem em texto.

**P: Quais componentes visuais existem hoje?**
R: 8, fixos: `Card`, `Column`, `Row`, `Text`, `Divider`, `BarChart`, `LineChart`, `DataTable` (Parte 3). A lista `a2ui_schema.components` do YAML restringe quais desses o supervisor pode usar.

**P: `catalog_id` precisa apontar para uma URL real que eu publico?**
R: Não. É só um rótulo/identificador estável — ninguém faz fetch dele. O frontend só usa esse valor para casar com o mesmo catálogo do lado do YAML.

**P: O parser do YAML valida os nomes dos componentes contra os 8 do renderer?**
R: Não — o parser de runtime (`ag_ui_generative_config.py`) é permissivo (aceita qualquer nome não vazio). Quem de fato restringe ao catálogo de 8 é o **renderer no frontend**; declarar um nome fora da lista no YAML não quebra o parse, mas a IA nunca vai conseguir desenhá-lo.

**P: `chat_renderer: jspuro` e `chat_renderer: copilotkit` mudam o que a IA gera?**
R: Não. O envelope A2UI que o supervisor produz é o mesmo nos dois casos. O campo só sinaliza/documenta qual frontend deve consumir — `jspuro` para o renderer nativo desta plataforma, `copilotkit` para um cliente React com o SDK CopilotKit consumindo `/ag-ui/copilotkit/runs`.

**P: Preciso ligar `middlewares.subagents.enabled: true` para o A2UI funcionar?**
R: Indiretamente sim, mas não por causa do A2UI em si: sem subagentes especialistas habilitados, o supervisor não tem como delegar a busca de dados via `task`, e sem dado real no histórico o `generate_a2ui` não tem o que desenhar. O bind da tool `generate_a2ui` em si só depende do bloco `ag_ui.generative` existir.

**P: Como dar "bind" nos campos da minha tela para mandar contexto junto com a pergunta?**
R: Via hook `buildPayloadText` (Parte 4). Independente do A2UI — funciona em qualquer modo do componente.

**P: Como testo se está funcionando?**
R: Abra a bancada `ui-embeddable-chat-test.html` (ou o WebChat v3) com um YAML que tenha o bloco `ag_ui.generative`, `mode: 'deepagent'`, `chatRenderer: 'jspuro'` e `agUiSseTransport: true`. Peça uma pergunta sem visualização (deve vir texto) e uma pedindo gráfico (deve desenhar). Se algo falhar, veja "Por que às vezes aparece texto" acima e confira o console do navegador.

**P: O A2UI funciona com Q&A (RAG) ou Workflow?**
R: Não. A geração A2UI pertence ao supervisor DeepAgent. Isso não impede Q&A/RAG de usar
o mesmo transporte SSE para texto progressivo com `mode: 'qa'`; apenas não transforma o
Q&A em gerador de A2UI. Workflow e agente clássico continuam fora desse gate no componente.

**P: Qual a diferença entre a Superfície A e a Superfície B na prática?**
R: Superfície A = chat embutível que também sabe desenhar (este tutorial). Superfície B = tela dedicada fora do chat, consumindo capability packs próprios (ex.: `retail_demo`) diretamente em `/ag-ui/runs`. São mecanismos independentes; um YAML pode ter os dois, mas eles não se misturam.

**P: O que é `correlation_id` e por que aparece nos logs?**
R: É o identificador único gerado pelo backend para cada envio, usado para reconstruir o que aconteceu numa execução (`python -m src.log_analyzer analyze --correlation-id <id>`). Pegue-o em `chat.obterEstadoAtual().correlationId` quando precisar investigar por que um gráfico não apareceu.

**P: Existe algum limite de quantas vezes o supervisor chama `generate_a2ui` numa resposta?**
R: O prompt instrui explicitamente **uma única chamada** por resposta, para evitar redesenhos repetidos (thrashing) — não é um limite técnico rígido da tool, é uma regra de contrato reforçada no prompt.

---

## Erros a evitar (pegadinhas)

**Confundir "sem bloco YAML" com "bug".** Se `ag_ui.generative` não está no supervisor, o comportamento correto é texto puro sempre — não é falha, é o contrato aditivo por design.

**Esquecer de ligar as 3 flags no host.** O YAML sozinho não é suficiente: sem `mode: 'deepagent'` + `chatRenderer: 'jspuro'` + `agUiSseTransport: true` na configuração do componente, o transporte SSE nunca ativa e o chat nunca vê o envelope A2UI, mesmo com o YAML perfeito.

**Pedir gráfico e esperar dado que nenhum especialista busca.** Se não existe subagente/tool `dyn_sql` capaz de trazer aquele número, o supervisor não tem histórico suficiente para desenhar — o prompt proíbe inventar dado, então tende a responder em texto.

**Declarar componente fora dos 8 em `a2ui_schema.components`.** O parser aceita, mas o renderer nunca vai conseguir desenhá-lo — a superfície cai em texto na hora H. Use só os 8 nomes da Parte 3.

**Misturar `chat_renderer` do YAML com a configuração do host.** Se o YAML diz `chat_renderer: jspuro` mas a tela host configura `chatRenderer: 'copilotkit'` (ou vice-versa), o comportamento não é o esperado — mantenha os dois alinhados.

**Confundir as duas superfícies.** `generate_a2ui` (Superfície A, chat embutível) e capability packs de telas dedicadas (Superfície B) são mecanismos independentes que compartilham o mesmo endpoint (`/ag-ui/runs`), mas não a mesma configuração nem o mesmo propósito.

**Documentação ou exemplo antigo citando `DashboardSpec`, `output_subagent` ou `middlewares.generative_ui_dashboard`.** Esses três nomes descrevem um mecanismo anterior que foi removido do código. Se você encontrar um desses termos em algum material, trate como desatualizado — o mecanismo vigente é o `ag_ui.generative` descrito neste tutorial.

---

## Referências cruzadas

- [GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md](GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md) — detalhe completo do componente de chat, API pública, eventos, HIL, payloadText, messageActions.
- [README-TECNICO-AG-UI.md](../tecnico/README-TECNICO-AG-UI.md) — referência técnica completa: endpoints, eventos SSE, adapters, capability packs, wiring backend do A2UI.
- [README-CONCEITUAL-AG-UI.md](../conceitual/README-CONCEITUAL-AG-UI.md) — visão de produto, conceitos e posicionamento.
- [GUIA-AG-UI-SDK-TERCEIROS.md](GUIA-AG-UI-SDK-TERCEIROS.md) — integração via SDK para terceiros (inclusive o caminho `copilotkit`).
