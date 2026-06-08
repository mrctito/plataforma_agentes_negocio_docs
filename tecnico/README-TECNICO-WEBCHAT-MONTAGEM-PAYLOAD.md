# Manual técnico: montagem do payload real do WebChat para o endpoint

Este documento descreve, com fidelidade ao código real, **como o browser transforma
um YAML (ou um payload já criptografado) no corpo HTTP que vai para os endpoints de
execução** (`/rag/execute`, `/agent/execute`, `/workflow/execute`). É o "momento de
montagem do payload".

O caminho oficial hoje passa pelo **componente global de chat embutível**
(`PrometeuEmbeddableChatRuntime`) e pelo **cliente HTTP canônico** `layout-mestre-api.js`.
Quem precisa entender a sequência ponta a ponta (handshake → cifra → envio → polling)
deve ler junto o GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md
e, para uma interface 100% própria sem o componente, o
GUIA-INTEGRADOR-CHAT-PLATAFORMA.md.

## 1. Onde o fluxo real está confirmado no código

A jornada abaixo foi confirmada lendo os pontos reais do projeto:

- `app/ui/static/js/shared/embeddable-chat-runtime.js` — componente de chat. Em
  `enviarPergunta()` chama `api.enviar({ modo, conteudo, extras })` e decide se faz
  polling assíncrono (modo `direct_async` ou resposta com URLs de status).
- `app/ui/static/js/shared/layout-mestre-api.js` — cliente HTTP canônico. Resolve o
  `encrypted_data` em `resolverPayloadCriptografado()`, monta o `requestBody` por modo
  (`enviarQA`, `enviarAgente`, `enviarWorkflow`) e executa o `fetch` em
  `_executarRequisicao()`. Define os endpoints em `LAYOUT_API_CONFIG.endpoints`.
- `app/ui/static/js/plataforma-agentes-ia-crypto.js` — helper browser oficial. Gera o
  envelope `encrypted_data` com `PayloadCrypto.buildEncryptedData(...)`.
- `src/api/routers/crypto_router.py` — entrega a sessão efêmera (`POST /crypto/session-key`)
  usada na criptografia, e o registro offline (`POST /crypto/offline-store`).
- `src/api/routers/rag_router.py` — contrato do `/rag/execute`: envelope
  `{ operation: "ask", payload: { question, user_email, format, execution_mode, encrypted_data, ... } }`.
- `src/api/routers/agent_router.py` — contrato do `/agent/execute`: campos no topo do
  corpo (`task`, `user_email`, `format`, `execution_mode`, `encrypted_data`, `mode`).

## 2. Passo a passo da montagem real

### Passo 1: resolver o `encrypted_data`

No cliente canônico, a função `resolverPayloadCriptografado()` faz duas coisas:

1. se o usuário já carregou um **payload pré-criptografado** (`credenciais.tipo === 'payload'`),
   valida e usa esse objeto diretamente (e, no default, registra a chave em
   `/crypto/offline-store` para evitar expiração da sessão);
2. se há **YAML** (`credenciais.tipo === 'yaml'`), então:
   - resolve o e-mail operacional (campo do contexto ou sessão federada);
   - injeta esse e-mail no YAML com `PayloadCrypto.injectUserEmailInYaml(...)`;
   - chama `PayloadCrypto.buildEncryptedData(...)`, que internamente pede a sessão
     efêmera em `/crypto/session-key`.

```js
const encryptedData = await this.resolverPayloadCriptografado();
```

### Passo 2: gerar o envelope com `PayloadCrypto`

O helper browser oficial usa esta assinatura:

```js
const payload = await window.PayloadCrypto.buildEncryptedData({
  yamlContent: yamlComEmail,
  filename: this.credenciais.yamlFilename || 'config.yaml',
  baseUrl: this.apiBaseUrl,
});
```

O retorno é exatamente este objeto (confirmado em `buildEncryptedData`):

```js
{
  session_id,                 // UUID da sessão efêmera
  wrapped_key,                // chave Fernet protegida via RSA-OAEP
  encrypted_yaml,             // YAML cifrado com Fernet
  original_filename,          // nome do arquivo informado
  encryption_scheme,          // "FERNET+RSA-OAEP"
  yaml_operational_contract,  // contrato operacional do YAML
}
```

> Importante (divergência corrigida): o helper **não** gera mais o campo `encrypted_keys`.
> As credenciais passaram a ser injetadas pelo backend; se você passar `keysContent`,
> o helper ignora e emite um aviso no console. Exemplos antigos do Swagger que mostram
> `encrypted_keys` no envelope refletem um contrato anterior — o envelope real é o de cima.

Esse objeto é o que o backend recebe no campo `encrypted_data`.

### Passo 3: montar o corpo final pelo modo

Depois de resolver o `encrypted_data`, o cliente monta o corpo conforme o modo. Os
corpos abaixo são os reais, copiados de `layout-mestre-api.js`.

#### 3.1 RAG (`/rag/execute`) — envelope `{ operation, payload }`

```js
const requestBody = {
  operation: 'ask',
  payload: {
    question: pergunta.trim(),
    user_email: userEmail,
    format: 'json',
    execution_mode: this.executionMode,   // 'auto' | 'direct_sync' | 'direct_async'
    encrypted_data: encryptedData,
  },
};
// Filtros opcionais de busca:
if (filtrosMetadata && Object.keys(filtrosMetadata).length > 0) {
  requestBody.payload.metadata_filters = filtrosMetadata;
}
```

#### 3.2 Agente / DeepAgent (`/agent/execute`) — campos no topo do corpo

```js
const requestBody = {
  task: tarefa.trim(),
  user_email: userEmail,
  format: 'json',
  execution_mode: this.executionMode,
  encrypted_data: encryptedData,
};
// Para DeepAgent, força o runtime moderno:
if (mode === 'deepagent') {
  requestBody.mode = 'deepagent';
}
```

> No contrato público do backend, `mode` aceita apenas `deepagent`. `agent` é legado e
> é bloqueado explicitamente (ver `AgentRequest.validate_mode` em `agent_router.py`).

#### 3.3 Workflow (`/workflow/execute`)

```js
const requestBody = {
  message: mensagem.trim(),
  user_email: userEmail,
  thread_id: threadId || null,
  format: 'json',
  execution_mode: this.executionMode,
  encrypted_data: encryptedData,
};
```

### Passo 4: enviar com os headers corretos

A chamada real (`_executarRequisicao`) usa `fetch` com `credentials: 'include'` **e**
os headers de autenticação. Não é só `credentials: 'include'`:

```js
const response = await fetch(url, {
  method: 'POST',
  credentials: 'include',
  headers: {
    'Content-Type': 'application/json',
    'X-API-Key': apiKey,                       // LAYOUT_API_CONFIG.accessKeyHeader
    'X-Prometeu-UI-Session-Required': '1',     // exige sessão de UI
  },
  body: JSON.stringify(requestBody),
});
```

A chave de API pode vir do header `X-API-Key` **ou** estar embutida no YAML em
`authentication.access_key` (o backend aceita as duas fontes). Quando o YAML já carrega
a chave, o `X-API-Key` pode ir vazio no header sem quebrar o fluxo.

### Passo 5: ler o correlation_id e tratar resposta assíncrona

O cliente extrai o `correlation_id` em `extrairCorrelationIdResposta()`, priorizando o
header `X-Correlation-Id` e caindo para o corpo (`correlation_id`, `correlationId`,
`log_id`, `interaction_id`, `detail.correlation_id`). **O browser nunca cria esse valor.**

Se a resposta for `HTTP 202` (ou síncrona mas com URLs de status no corpo), o componente
entra em polling via `WebchatAsyncRuntime.waitForTaskCompletion`, consultando
`/api/v1/status/{task_id}` com o header `X-API-Key`, até o status virar `completed`,
`failed`, `cancelled`, `paused` ou `awaiting_human_decision`.

## 3. Regras importantes para quem implementa em React/JavaScript

### 3.1 Não envie YAML em claro como se fosse o corpo final

O contrato real do backend exige `encrypted_data`. O browser gera esse envelope; não
tente "fingir" que o YAML bruto é suficiente.

### 3.2 Injete o e-mail operacional antes da criptografia

`injectUserEmailInYaml(...)` faz parte do caminho real: o e-mail operacional entra em
`user_session.user_email` **antes** da cifra, não depois.

### 3.3 Preserve o modo de execução

`execution_mode` é parte do contrato. Mantenha exatamente o valor (`auto`, `direct_sync`
ou `direct_async`) que a tela seleciona.

### 3.4 Não gere `correlation_id` no browser

O `correlation_id` vem do backend e é lido da resposta (header e/ou corpo), nunca
inventado no frontend.

## 4. Exemplo mínimo pronto para copiar (vanilla, com handshake + cifra + envio)

```js
async function enviarPerguntaRag({ baseUrl, yamlText, question, userEmail, apiKey }) {
  // 1) Gera o envelope cifrado. buildEncryptedData faz o handshake em /crypto/session-key.
  const encryptedData = await window.PayloadCrypto.buildEncryptedData({
    yamlContent: window.PayloadCrypto.injectUserEmailInYaml(yamlText, userEmail),
    filename: 'config.yaml',
    baseUrl,
  });

  // 2) Envia o envelope para /rag/execute.
  const response = await fetch(`${baseUrl}/rag/execute`, {
    method: 'POST',
    credentials: 'include',
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': apiKey || '',                  // pode ir vazio se o YAML tiver authentication.access_key
      'X-Prometeu-UI-Session-Required': '1',
    },
    body: JSON.stringify({
      operation: 'ask',
      payload: {
        question,
        user_email: userEmail,
        format: 'json',
        execution_mode: 'auto',
        encrypted_data: encryptedData,
      },
    }),
  });

  // 3) Lê o correlation_id (header ou corpo) e a resposta.
  const correlationId =
    response.headers.get('X-Correlation-Id') || '';
  const body = await response.json();
  if (!response.ok) {
    throw new Error(
      `${body?.detail || 'Falha ao chamar /rag/execute'} (correlation_id: ${correlationId || body?.correlation_id || 'ausente'})`
    );
  }
  return { ...body, correlation_id: body.correlation_id || correlationId };
}
```

> Para o caminho assíncrono completo (HTTP 202 + polling em `/api/v1/status`), veja o
> exemplo vanilla no GUIA-INTEGRADOR-CHAT-PLATAFORMA.md.

## 5. Diagnóstico rápido

Se a chamada falhar, verifique nesta ordem:

1. `window.PayloadCrypto` está carregado (script `plataforma-agentes-ia-crypto.js`);
2. o YAML é válido e não está vazio;
3. `/crypto/session-key` responde 200 (o handshake acontece dentro de `buildEncryptedData`);
4. o corpo final envia `encrypted_data` como **objeto**, não string;
5. os headers `X-API-Key` (ou `authentication.access_key` no YAML) e
   `X-Prometeu-UI-Session-Required` estão presentes;
6. o endpoint escolhido é o certo para o modo atual;
7. capture o `correlation_id` da resposta (header/corpo) para investigar no log do backend
   com `python -m src.log_analyzer query --correlation-id <id>`.

## 6. Limites confirmados

- esta documentação descreve o caminho oficial do WebChat real, não uma API paralela;
- o frontend precisa respeitar o contrato do backend;
- o helper browser é a fonte certa para a criptografia; reimplementar isso manualmente
  aumenta risco de divergência;
- o envelope real **não** contém `encrypted_keys` (campo descontinuado no helper).
