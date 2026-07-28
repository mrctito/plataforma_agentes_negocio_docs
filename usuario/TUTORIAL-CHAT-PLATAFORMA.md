# Tutorial: montar um chat funcional na plataforma, ponta a ponta

Este é o tutorial guiado de chat da Plataforma de Agentes de IA. Diferente do
[GUIA-PRIMEIROS-PASSOS.md](GUIA-PRIMEIROS-PASSOS.md), que entrega **uma** resposta com o menor número de passos,
aqui você vai construir um **chat funcional completo**, reutilizável, cobrindo os 4 modos de conversa, o
tratamento de erro e a leitura de `correlation_id`.

O caminho **recomendado** é embutir o componente oficial da plataforma, o `PrometeuEmbeddableChatRuntime`. Ele já
resolve handshake criptográfico, cifra do YAML, envio por modo e `correlation_id`, sem reabrir contrato. Criar um
runtime de chat paralelo numa tela é violação de reuso — quando o componente atende, embuta o componente; quando
faltar algo, a decisão correta é **evoluí-lo**, não criar outro ao lado.

> Pré-requisito: faça o [GUIA-PRIMEIROS-PASSOS.md](GUIA-PRIMEIROS-PASSOS.md) primeiro. Lá você sobe a aplicação
> (`./run.sh +a`, host e porta obrigatórios de `FASTAPI_HOST`/`FASTAPI_PORT`), prepara credencial + e-mail + YAML e
> confirma que a API
> responde. Este tutorial assume que esse ambiente já está de pé e **não repete** essa preparação.

Leituras de aprofundamento que este tutorial cruza, sem duplicar:

- [GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md](GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md) — contrato completo do componente
  (API pública, eventos, estado exportado, configuração).
- [README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md](../tecnico/README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md) — como o corpo HTTP é
  montado por baixo, com fidelidade ao código.
- [GUIA-INTEGRADOR-CHAT-PLATAFORMA.md](GUIA-INTEGRADOR-CHAT-PLATAFORMA.md) — para quem precisa de uma interface
  100% própria, sem o componente.

## 1. O modelo mental: host configura, componente conversa

A separação de responsabilidades é o coração deste tutorial:

- **a tela host** (sua página) decide o contexto: qual YAML usar, qual e-mail, qual chave de acesso, qual modo de
  conversa e qual modo de execução. Ela reserva um espaço na página e observa o que acontece.
- **o componente** renderiza a conversa (lista de mensagens, campo de pergunta, botão, status), monta o payload
  pelo caminho canônico, chama a API, trata a resposta e o erro, preserva o `correlation_id` e mantém o histórico
  da sessão.

A tela host **não** desenha a conversa, **não** monta payload por conta própria, **não** cria cliente HTTP
paralelo e **não** inventa `correlation_id`. Se você se pegar fazendo qualquer uma dessas coisas, parou de usar o
componente e começou a reabrir o problema que ele existe para resolver.

## 2. Conceitos que este tutorial usa

Estes conceitos já foram introduzidos no quickstart; aqui vão ganhar profundidade prática:

- **Modos de conversa (`mode`)**: definem como a pergunta é processada e qual endpoint é chamado.
  - `qa` (RAG, pergunta e resposta sobre uma base de conhecimento) -> `POST /rag/execute`;
  - `agent` (agente que executa uma tarefa) -> `POST /agent/execute`;
  - `deepagent` (supervisão multi-agente, o caminho moderno de agentes) -> `POST /agent/execute` com
    `mode: "deepagent"`;
  - `workflow` (grafo determinístico de etapas) -> `POST /workflow/execute`.
  O alias `rag` é aceito e tratado internamente como `qa`.
- **`executionMode` (campo vestigial, sem efeito real):** existe na API do componente por
  compatibilidade, mas o envio é sempre síncrono — o componente não faz mais polling de
  status. Detalhe e evidência de código na seção 7.
- **`correlation_id`**: o identificador oficial da execução, lido do header `X-Correlation-Id` ou do corpo. O
  componente apenas captura, exibe e propaga; nunca cria.

## 3. Carregar as dependências obrigatórias

O componente reutiliza a infraestrutura canônica da plataforma e **falha fechado** se faltar qualquer
dependência. Carregue os scripts compartilhados **antes** do componente, nesta ordem:

```html
<script src="/ui/static/js/shared/layout-mestre-api.js"></script>
<script src="/ui/static/js/shared/ui-webchat-runtime-utils.js"></script>
<script src="/ui/static/js/shared/embeddable-chat-runtime.js"></script>
```

Essas dependências precisam existir no escopo global do browser: `window.prometeuLayoutMestreApi` e
`window.WebchatRuntimeUtils`. Se uma delas faltar, o construtor do componente lança erro explícito (por exemplo,
"prometeuLayoutMestreApi é obrigatório para o chat embutível."). Isso é intencional: a v1 não mascara contrato
quebrado com fallback escondido.

> `window.WebchatAsyncRuntime` (`ui-webchat-async-runtime.js`) não precisa mais ser carregado por esse motivo: o
> componente não faz polling de status desde a decisão "Slice A" (seção 7) e não checa mais essa dependência.
>
> O helper de criptografia (`plataforma-agentes-ia-crypto.js`, que expõe `window.PayloadCrypto`) também precisa
> estar disponível quando o fluxo usar YAML em texto, pois é ele quem faz o handshake e a cifra. Quando a host já
> entrega um payload pré-criptografado, esse passo é dispensado.

## 4. Reservar o container

O componente ocupa 100% da largura e da altura do container onde é montado. Quem define o tamanho é a host. Sem
altura útil, o chat não tem onde crescer:

```html
<div id="chat-host" style="width: 100%; height: 480px; min-height: 480px;"></div>
```

## 5. Montar o chat: exemplo funcional completo

O exemplo abaixo é um chat completo e funcional usando o componente oficial. Ele cria a instância, monta no
container, expõe um controle para trocar de modo, e reage aos eventos do componente (resposta, erro, mudança de
estado). Os nomes de método e de evento são fiéis a `app/ui/static/js/shared/embeddable-chat-runtime.js`.

```html
<!-- Controles da host (fora do componente) -->
<label>Modo:
  <select id="seletor-modo">
    <option value="qa">qa (RAG)</option>
    <option value="agent">agent</option>
    <option value="deepagent">deepagent</option>
    <option value="workflow">workflow</option>
  </select>
</label>

<!-- Container do chat -->
<div id="chat-host" style="width:100%;height:480px;min-height:480px"></div>

<!-- Painel observador da host -->
<div id="barra-status"></div>
<div id="rotulo-correlation"></div>

<!-- Dependências canônicas (ordem importa) -->
<script src="/ui/static/js/shared/layout-mestre-api.js"></script>
<script src="/ui/static/js/shared/ui-webchat-runtime-utils.js"></script>
<script src="/ui/static/js/shared/embeddable-chat-runtime.js"></script>

<script>
  // 1) Contexto resolvido pela host. Em produção, o YAML vem da carga same-origin
  //    (?yaml=<arquivo>.yaml) e o e-mail da sessão federada. Aqui ilustramos os campos.
  const contexto = {
    yamlContent: '/* conteúdo do seu YAML de app/yaml/ */',
    yamlFilename: 'config.yaml',
    userEmail: 'developer@empresa.com',
    apiKey: '',                 // vazio se o YAML já traz authentication.access_key
  };

  const barraStatus = document.getElementById('barra-status');
  const rotuloCorrelation = document.getElementById('rotulo-correlation');

  // 2) Cria o componente. onChange e onEvent são os canais de observação da host.
  const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
    yamlContent: contexto.yamlContent,
    yamlFilename: contexto.yamlFilename,
    userEmail: contexto.userEmail,
    apiKey: contexto.apiKey,
    mode: 'qa',                 // modo de conversa inicial
    // Observação reativa: estado exportado do componente
    onChange(state) {
      barraStatus.textContent = state.statusMessage || '';
      rotuloCorrelation.textContent = state.correlationId
        ? `correlation_id: ${state.correlationId}`
        : 'correlation_id: aguardando backend';
    },
    // Observação por evento tipado (question-sent, response-received, error, ...)
    onEvent(evento) {
      if (evento.type === 'error') {
        console.error('Falha visível do chat:', evento.payload);
      }
    },
  });

  // 3) Monta no container. mount(container) retorna o elemento raiz do componente,
  //    útil para ouvir eventos DOM no namespace prometeu-embeddable-chat:*.
  const root = chat.mount(document.getElementById('chat-host'));

  root.addEventListener('prometeu-embeddable-chat:response-received', (event) => {
    console.log('Resposta recebida:', event.detail);
  });

  // 4) Troca de modo de conversa, sem recriar o componente.
  document.getElementById('seletor-modo').addEventListener('change', (e) => {
    chat.setMode(e.target.value); // 'qa' | 'agent' | 'deepagent' | 'workflow'
  });
</script>
```

O usuário digita no campo do componente e envia por clique ou Enter (Shift+Enter quebra linha). Você também pode
disparar perguntas por código: `chat.perguntar('texto')` envia direto, e `chat.preencherPergunta('texto')` apenas
preenche o campo sem enviar.

## 6. Os 4 modos de conversa na prática

A única coisa que muda entre os modos, do ponto de vista da host, é o valor de `mode`. O componente cuida do
endpoint e do formato do corpo:

```javascript
// Q&A (RAG) sobre uma base de conhecimento — POST /rag/execute
chat.setMode('qa');
chat.perguntar('Quais são os requisitos descritos no manual?');

// Agente que executa uma tarefa — POST /agent/execute
chat.setMode('agent');
chat.perguntar('Gere um resumo executivo do contrato anexado.');

// DeepAgent (supervisão multi-agente, caminho moderno) — POST /agent/execute (mode=deepagent)
chat.setMode('deepagent');
chat.perguntar('Coordene a análise técnica e a checagem de conformidade do documento.');

// Workflow (grafo determinístico de etapas) — POST /workflow/execute
chat.setMode('workflow');
chat.perguntar('Inicie o fluxo de triagem com os dados informados.');
```

Para Q&A, você pode restringir a busca por metadados:

```javascript
chat.setMode('qa');
chat.setMetadataFilters({ categoria: 'engenharia', ano: 2026 });
chat.perguntar('O que mudou nas normas técnicas neste ano?');
```

Para Workflow, é comum manter uma thread de conversa:

```javascript
chat.setMode('workflow');
chat.setThreadId('thread-atendimento-001');
chat.perguntar('Continuar o atendimento do protocolo anterior.');
```

> Diferença entre `agent` e `deepagent`: ambos chamam `/agent/execute`, mas `deepagent` força o runtime moderno
> de supervisão multi-agente (`mode: "deepagent"` no corpo). No contrato público do backend, `mode` aceita apenas
> `deepagent`; o `agent` clássico no corpo é legado e bloqueado. Detalhe em
> [README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md](../tecnico/README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md).

## 7. Por que não existem mais "3 modos de execução" (componente é sync-only)

Versões anteriores deste tutorial descreviam `auto`/`direct_sync`/`direct_async` como três escolhas reais, com
polling automático em `direct_async`. Isso mudou: uma decisão do produto ("Slice A") tornou o componente
**sempre síncrono**. Evidência no código (`embeddable-chat-runtime.js`, função `normalizeExecutionMode`):

```javascript
function normalizeExecutionMode(_mode) {
    // Decisão do usuário (Slice A): TODO webchat opera SOMENTE em modo síncrono...
    return 'direct_sync';
}
```

Na prática, para quem usa o componente:

- `chat.setExecutionMode(...)` e a opção `executionMode` na configuração continuam aceitos (não quebram sua
  chamada), mas **não mudam** o comportamento — o valor é sempre normalizado para `direct_sync`;
- não existe mais polling: a resposta final chega na própria resposta HTTP do envio, sem passos extras;
- para tarefas longas de agente, o backend continua respondendo síncrono, protegido por um timeout guard interno
  (não é responsabilidade da host acompanhar progresso).

Se você precisa **mesmo assim** de acompanhamento assíncrono manual (por exemplo, construindo uma interface
própria fora do componente — seção 11), o mecanismo de polling ainda existe no backend e em
`layout-mestre-api.js`/`ui-webchat-async-runtime.js`, mas o componente embutível não o aciona mais.

## 8. Tratamento de erro e `correlation_id`

O componente **mostra o erro real do backend** e preserva o `correlation_id`, em vez de esconder a falha. Você
observa isso de três formas complementares:

1. **Estado exportado** (via `onChange` ou `chat.obterEstadoAtual()`): os campos `lastError`, `status` (`error`) e
   `correlationId`.
2. **Evento de erro** (`prometeu-embeddable-chat:error` no DOM, ou `onEvent` com `evento.type === 'error'`).
3. **Visualmente na conversa**: a mensagem de erro útil aparece na própria área de mensagens.

```javascript
const chat = window.EmbeddableChatRuntime.createGenericEmbeddableChat({
  yamlContent: contexto.yamlContent,
  userEmail: contexto.userEmail,
  apiKey: contexto.apiKey,
  mode: 'qa',
  onChange(state) {
    if (state.status === 'error' && state.lastError) {
      // lastError já vem normalizado pelo componente.
      console.error('Erro do backend:', state.lastError);
      console.log('correlation_id para investigar o log:', state.correlationId);
    }
  },
});
```

Pontos de disciplina, fiéis ao contrato do produto:

- **O `correlation_id` é sempre do backend.** O componente lê o header `X-Correlation-Id` ou o corpo; nunca
  inventa um valor local. Se a barra de status exibir "aguardando backend", é porque a resposta ainda não trouxe o
  id — o que, em erro, indica falha de observabilidade do backend, não algo para a UI contornar.
- **Erro com mensagem útil deve aparecer.** Não engula o erro do backend; o componente já o surfa para o usuário e
  preserva o id para rastreabilidade.
- **Para investigar a causa raiz**, use o `correlation_id` capturado:

  ```bash
  python -m src.log_analyzer query --correlation-id <correlation-id-capturado>
  ```

## 9. Histórico e leitura de estado

O componente mantém o histórico da sessão em memória. A host pode lê-lo sem tocar no DOM:

```javascript
const estado = chat.obterEstadoAtual();   // snapshot completo (input, messages, lastResponse, correlationId, ...)
const historico = chat.obterHistorico();  // lista de interações da sessão
const ultima = chat.obterUltimaInteracao(); // última pergunta + resposta associadas
chat.limparHistorico();                   // limpa a conversa programaticamente
```

A persistência externa (banco, Redis, localStorage) não é responsabilidade do componente; se você precisar dela,
faça pela host consumindo o estado exportado e os eventos.

## 10. Checklist de uma host bem feita

- O container tem altura útil (sem altura, o chat não cresce).
- As dependências canônicas (`layout-mestre-api.js`, `ui-webchat-runtime-utils.js` e, quando usar YAML em texto,
  `plataforma-agentes-ia-crypto.js`) foram carregadas, na ordem certa, antes do componente.
- A host injeta `yamlContent` (ou `encryptedPayload`) + `userEmail`; a chave vai por `apiKey` **ou** já está no
  YAML em `authentication.access_key` (alternativas, basta uma; e-mail sempre obrigatório).
- A host observa `correlationId`/`lastResponse` quando precisa de rastreabilidade.
- A host **não** criou cliente HTTP paralelo, **não** recriou a renderização da conversa e **não** duplicou
  montagem de payload.

## 11. Quando você não pode usar o componente

Se a integração é fora do navegador da plataforma (um backend Python/Node consumindo a API), o componente não se
aplica. Nesse caso, siga o mesmo contrato de payload e criptografia, mas implementando o cliente você mesmo:

- exemplos completos por linguagem em [README-EXEMPLOS-INTEGRACAO-API.md](README-EXEMPLOS-INTEGRACAO-API.md) e nos
  arquivos `examples/`;
- montagem exata do corpo em [README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md](../tecnico/README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md);
- interface própria ponta a ponta (handshake + cifra + envio + polling) no
  [GUIA-INTEGRADOR-CHAT-PLATAFORMA.md](GUIA-INTEGRADOR-CHAT-PLATAFORMA.md).

Mesmo nesses casos, a regra de reuso continua valendo: dentro da plataforma, embuta o componente; criar um chat
paralelo só se justifica com requisito comprovado que o componente não cobre — e a saída certa é evoluí-lo.

## 12. Evidências no código

- `app/ui/static/js/shared/embeddable-chat-runtime.js` — `createGenericEmbeddableChat`, `mount`, `setMode`,
  `setExecutionMode`, `perguntar`, `obterEstadoAtual`, eventos `prometeu-embeddable-chat:*`.
- `app/ui/static/js/shared/layout-mestre-api.js` — endpoints por modo, headers, polling (`_extrairInfoAssincrona`).
- `app/ui/static/js/shared/ui-webchat-async-runtime.js` — `waitForTaskCompletion` (loop de polling).
- `app/ui/static/js/plataforma-agentes-ia-crypto.js` — `PayloadCrypto.buildEncryptedData`, `injectUserEmailInYaml`.
- `src/api/routers/rag_router.py`, `agent_router.py`, `workflow_router.py` — boundaries por modo.
- `src/api/routers/streaming_router.py` — `/api/v1/status/{task_id}` (acompanhamento assíncrono).
- `.claude/rules/componente-chat-embutivel.md` — regra de reuso e estado de adoção do componente.
