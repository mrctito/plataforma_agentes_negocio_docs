# Manual técnico, operacional e de integração: API usada pelo WebChat v3 para conversar com o backend

## 1. Escopo deste guia

Este é o guia técnico principal para quem quer construir uma interface própria compatível com o comportamento real de [app/ui/static/ui-webchat-v3.html](../app/ui/static/ui-webchat-v3.html).

Ele cobre:

- endpoint usado por modo;
- montagem real do payload;
- lógica de escolha desse payload;
- ciclo de resposta síncrona e assíncrona;
- leitura de `correlation_id`;
- exemplo mínimo porém funcional em React.

## 2. Fonte de verdade lida no código

Os contratos e a jornada abaixo foram confirmados nestes pontos do repositório:

- app/ui/static/ui-webchat-v3.html;
- app/ui/static/js/ui-webchat-v3.js;
- app/ui/static/js/shared/ui-webchat-runtime-utils.js;
- app/ui/static/js/shared/yaml-access-key-extractor.js;
- app/ui/static/js/plataforma-agentes-ia-crypto.js;
- src/api/routers/rag_router.py;
- src/api/schemas/rag_models.py;
- src/api/routers/agent_router.py;
- src/api/routers/workflow_router.py;
- src/api/routers/streaming_router.py.

## 3. Mapa dos endpoints usados pela tela

### 3.1 Endpoint principal por modo

| Modo na UI  | Endpoint            | Corpo principal                |
| ----------- | ------------------- | ------------------------------ |
| `rag`       | `/rag/execute`      | `operation + payload.question` |
| `agent`     | `/agent/execute`    | `task`                         |
| `deepagent` | `/agent/execute`    | `task + mode='deepagent'`      |
| `workflow`  | `/workflow/execute` | `message`                      |

### 3.2 Endpoints auxiliares da jornada

| Função                       | Endpoint                          | Papel                                                           |
| ---------------------------- | --------------------------------- | --------------------------------------------------------------- |
| Sessão do usuário            | `/api/auth/federated/session`     | resolve o usuário autenticado usado pela UI                     |
| Sessão criptográfica efêmera | `/crypto/session-key`             | usado indiretamente por `PayloadCrypto.buildEncryptedData(...)` |
| Polling de execução          | `/api/v1/status/{task_id}`        | acompanha respostas em modo assíncrono                          |
| Streaming assíncrono         | `/api/v1/status/stream/{task_id}` | acompanhamento contínuo por SSE                                 |

## 4. Jornada técnica completa da pergunta até a resposta

### Etapa 1: a UI coleta a pergunta

A tela registra a pergunta do usuário no estado local e só então começa a montagem do contexto.

### Etapa 2: a UI resolve o modo de operação

O modo pode vir de:

- seleção manual na interface;
- leitura do YAML atual por `extractPreferredModeFromYaml(...)`.

Esse resolvedor segue esta lógica prática:

- `selected_supervisor` ou `multi_agents` sugerem `deepagent`;
- `selected_workflow` ou `workflows` sugerem `workflow`;
- `mode` ou `execution.type` explícitos também podem influenciar;
- se nada disso existir, o comportamento tende a permanecer no modo padrão da tela.

### Etapa 3: a UI resolve a API key

A função `_resolverApiKey()` usa esta prioridade:

1. valor digitado no campo da tela;
2. fallback extraído do YAML;
3. se houver apenas payload criptografado e nenhum campo preenchido, a UI não consegue recuperar a key do payload.

Isso é importante porque o payload criptografado não foi desenhado para devolver a `access_key` ao frontend.

### Etapa 4: a UI resolve `encrypted_data`

O método `_resolverEncryptedData()` segue esta ordem:

1. se `payloadConteudo` existir, ele é parseado e usado diretamente;
2. senão, a UI tenta usar `yamlConteudo`;
3. se houver YAML, a UI busca o e-mail autenticado via `/api/auth/federated/session`;
4. injeta esse e-mail no YAML com `PayloadCrypto.injectUserEmailInYaml(...)`;
5. chama `PayloadCrypto.buildEncryptedData(...)`.

O objeto final inclui campos como:

- `session_id`;
- `wrapped_key`;
- `encrypted_yaml`;
- `original_filename`;
- `encryption_scheme`.

## 5. Contrato real do payload por modo

### 5.1 Q&A / RAG

No WebChat v3, o modo Q&A usa o dispatcher RAG com envelope externo e payload interno.

```json
{
  "operation": "ask",
  "payload": {
    "question": "Qual é o SLA de suporte premium?",
    "user_email": "analista.suporte@cliente.com",
    "format": "json",
    "execution_mode": "auto",
    "encrypted_data": {
      "session_id": "...",
      "wrapped_key": "...",
      "encrypted_yaml": "...",
      "original_filename": "config-webchat.yaml",
      "encryption_scheme": "FERNET+RSA-OAEP"
    }
  }
}
```

Campos opcionais relevantes no modo RAG:

- `payload.metadata_filters`;
- `payload.correlation_id`;
- `payload.image_base64` em cenários multimodais.

### 5.2 Agent / DeepAgent

No contrato público de `/agent/execute`, a UI manda:

```json
{
  "task": "Analise este cenário e responda com plano de ação.",
  "user_email": "analista.suporte@cliente.com",
  "format": "json",
  "execution_mode": "auto",
  "encrypted_data": {"...": "..."},
  "mode": "deepagent"
}
```

O campo `mode` só deve ser usado no contrato público para forçar `deepagent`.

### 5.3 Workflow

No contrato público de `/workflow/execute`, a UI manda:

```json
{
  "message": "Quero continuar o processo anterior.",
  "user_email": "analista.suporte@cliente.com",
  "thread_id": null,
  "format": "json",
  "execution_mode": "auto",
  "encrypted_data": {"...": "..."}
}
```

## 6. Como a lógica sabe montar o payload certo

Ela decide a estrutura a partir de quatro perguntas simples:

1. qual modo está ativo?
2. existe YAML ou payload já pronto?
3. qual é o usuário operacional?
4. existem filtros ativos?

Em pseudo-lógica:

```text
se modo == rag:
    endpoint = /rag/execute
    body = { operation: ask, payload: {...} }

se modo == agent:
    endpoint = /agent/execute
    body = { task: pergunta, ... }

se modo == deepagent:
    endpoint = /agent/execute
    body = { task: pergunta, mode: deepagent, ... }

se modo == workflow:
    endpoint = /workflow/execute
    body = { message: pergunta, thread_id, ... }
```

O ponto prático é este: a UI não deduz a estrutura olhando a resposta. Ela deduz pela combinação de modo atual + contexto carregado.

## 7. Como a resposta é lida pela UI

Depois do `fetch`, a UI:

1. lê o corpo como texto;
2. passa esse texto por `parseResponsePayload(...)`;
3. extrai o `correlation_id` com prioridade para o payload, e depois para headers HTTP como `x-correlation-id`;
4. registra esse correlation_id no estado da interface.

Essa ordem existe para preservar o valor oficial do backend mesmo quando ele vem em formatos ligeiramente diferentes.

## 8. Resposta síncrona vs assíncrona

### 8.1 Resposta síncrona

Quando a API responde com sucesso imediato, a UI consome o JSON final e renderiza a resposta.

No caso de `/rag/execute`, o modelo público inclui campos como:

- `answer`;
- `correlation_id`;
- `timestamp`;
- `sources`;
- `source_documents`;
- `analysis`;
- `metrics`;
- `task_id`, `polling_url`, `stream_url` quando aplicável.

### 8.2 Resposta assíncrona

Quando a API responde com `202`, a UI trata isso como início de execução em background. Ela extrai:

- `task_id`;
- `polling_url`;
- `stream_url`;
- `status_url` como alias legado quando aparecer;
- `correlation_id`.

Depois disso, ela acompanha a tarefa até status terminal, usando o slice de status.

## 9. Jornada técnica do estagiário para construir a própria interface

### Fase 1: fazer o menor caso real funcionar

Comece com RAG. Não comece com DeepAgent e nem com Workflow.

Objetivo da primeira entrega:

- um campo para YAML;
- um campo para pergunta;
- um campo para e-mail ou uso da sessão oficial;
- geração de `encrypted_data`;
- `POST /rag/execute`;
- renderização de `answer`.

### Fase 2: adicionar robustez operacional

Depois que o caso mínimo funcionar, adicione:

- leitura de `correlation_id`;
- tratamento de `202`;
- polling em `/api/v1/status/{task_id}`;
- exibição de erro amigável.

### Fase 3: só então expandir para outros modos

Quando o caso RAG estiver sólido, aí sim implemente Agent/DeepAgent e Workflow.

## 10. Exemplo mínimo, porém funcional, em React

O exemplo abaixo é pequeno, mas funcional. Ele:

- usa `window.PayloadCrypto` real;
- injeta e-mail no YAML quando o helper existir;
- chama `/rag/execute`;
- trata resposta síncrona e `202` com polling;
- preserva `correlation_id`.

Pré-requisito operacional:

- a página que hospeda o React precisa carregar antes o script browser de criptografia da plataforma;
- o backend precisa estar acessível no mesmo host ou em `apiBaseUrl` compatível.

```jsx
import React, { useState } from 'react';

function getCorrelationId(response, payload) {
  const payloadId = [
    payload?.correlation_id,
    payload?.correlationId,
    payload?.detail?.correlation_id,
    payload?.detail?.correlationId,
  ].find(Boolean);

  if (payloadId) {
    return String(payloadId).trim();
  }

  return String(
    response.headers.get('x-correlation-id')
      || response.headers.get('X-Correlation-Id')
      || ''
  ).trim();
}

async function parseJsonOrThrow(response) {
  const rawText = await response.text();
  let payload = {};

  if (rawText.trim()) {
    try {
      payload = JSON.parse(rawText);
    } catch {
      payload = { detail: rawText };
    }
  }

  const correlationId = getCorrelationId(response, payload);
  if (correlationId && !payload.correlation_id) {
    payload.correlation_id = correlationId;
  }

  if (!response.ok) {
    const message = payload?.detail || payload?.error || `Erro HTTP ${response.status}`;
    const error = new Error(String(message));
    error.payload = payload;
    throw error;
  }

  return payload;
}

async function maybeResolveUserEmail(apiBaseUrl, fallbackEmail) {
  try {
    const response = await fetch(`${apiBaseUrl}/api/auth/federated/session`, {
      credentials: 'include',
    });
    if (!response.ok) {
      return fallbackEmail;
    }
    const session = await response.json();
    return session?.email || fallbackEmail;
  } catch {
    return fallbackEmail;
  }
}

async function buildEncryptedData(apiBaseUrl, yamlText, userEmail) {
  if (!window.PayloadCrypto || typeof window.PayloadCrypto.buildEncryptedData !== 'function') {
    throw new Error('PayloadCrypto não foi carregado.');
  }

  const yamlWithEmail = typeof window.PayloadCrypto.injectUserEmailInYaml === 'function'
    ? window.PayloadCrypto.injectUserEmailInYaml(yamlText, userEmail)
    : yamlText;

  return window.PayloadCrypto.buildEncryptedData({
    yamlContent: yamlWithEmail,
    filename: 'rag-config-webchat.yaml',
    baseUrl: apiBaseUrl,
  });
}

async function pollUntilDone(apiBaseUrl, pollingUrl) {
  const endpoint = pollingUrl.startsWith('http')
    ? pollingUrl
    : `${apiBaseUrl}${pollingUrl}`;

  for (;;) {
    const response = await fetch(endpoint, {
      credentials: 'include',
    });
    const payload = await parseJsonOrThrow(response);
    const status = String(payload?.status || '').toLowerCase();

    if (status === 'completed') {
      return payload?.result || payload;
    }
    if (status === 'failed' || status === 'cancelled') {
      throw new Error(payload?.error || payload?.message || 'Execução assíncrona falhou.');
    }

    await new Promise((resolve) => setTimeout(resolve, 1000));
  }
}

export default function WebchatRagMinimalReact() {
  const apiBaseUrl = window.location.origin;
  const [yamlText, setYamlText] = useState('');
  const [question, setQuestion] = useState('Qual é o objetivo deste YAML?');
  const [email, setEmail] = useState('dev@example.com');
  const [apiKey, setApiKey] = useState('');
  const [answer, setAnswer] = useState('');
  const [correlationId, setCorrelationId] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  async function handleSend() {
    setLoading(true);
    setError('');
    setAnswer('');
    setCorrelationId('');

    try {
      const resolvedEmail = await maybeResolveUserEmail(apiBaseUrl, email);
      const encryptedData = await buildEncryptedData(apiBaseUrl, yamlText, resolvedEmail);

      const headers = {
        'Content-Type': 'application/json',
      };
      if (apiKey.trim()) {
        headers['X-API-Key'] = apiKey.trim();
      }

      const response = await fetch(`${apiBaseUrl}/rag/execute`, {
        method: 'POST',
        headers,
        credentials: 'include',
        body: JSON.stringify({
          operation: 'ask',
          payload: {
            question,
            user_email: resolvedEmail,
            format: 'json',
            execution_mode: 'auto',
            encrypted_data: encryptedData,
          },
        }),
      });

      const payload = await parseJsonOrThrow(response);
      setCorrelationId(payload?.correlation_id || '');

      if (response.status === 202) {
        const finalPayload = await pollUntilDone(
          apiBaseUrl,
          payload?.polling_url || payload?.status_url
        );
        setAnswer(finalPayload?.answer || finalPayload?.response || finalPayload?.message || 'Sem resposta final.');
        setCorrelationId(finalPayload?.correlation_id || payload?.correlation_id || '');
        return;
      }

      setAnswer(payload?.answer || payload?.response || payload?.message || 'Sem resposta.');
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Erro desconhecido.');
    } finally {
      setLoading(false);
    }
  }

  return (
    <section style={{ maxWidth: 960, margin: '0 auto', display: 'grid', gap: 12 }}>
      <h2>React mínimo para o contrato real do WebChat/RAG</h2>

      <label>
        YAML
        <textarea
          rows={12}
          value={yamlText}
          onChange={(event) => setYamlText(event.target.value)}
          style={{ width: '100%' }}
          placeholder="Cole aqui o YAML usado pelo runtime"
        />
      </label>

      <label>
        Pergunta
        <input
          value={question}
          onChange={(event) => setQuestion(event.target.value)}
          style={{ width: '100%' }}
        />
      </label>

      <label>
        E-mail fallback
        <input
          value={email}
          onChange={(event) => setEmail(event.target.value)}
          style={{ width: '100%' }}
        />
      </label>

      <label>
        API Key opcional
        <input
          value={apiKey}
          onChange={(event) => setApiKey(event.target.value)}
          style={{ width: '100%' }}
        />
      </label>

      <button onClick={handleSend} disabled={loading || !yamlText.trim() || !question.trim()}>
        {loading ? 'Enviando...' : 'Enviar pergunta'}
      </button>

      {correlationId ? <pre>correlation_id: {correlationId}</pre> : null}
      {error ? <pre style={{ color: 'crimson' }}>{error}</pre> : null}
      {answer ? <pre style={{ whiteSpace: 'pre-wrap' }}>{answer}</pre> : null}
    </section>
  );
}
```

## 11. Por que esse exemplo é funcional de verdade

Ele não simplifica as partes críticas do contrato:

- usa `PayloadCrypto` oficial;
- gera `encrypted_data` no browser;
- chama `/rag/execute` com o envelope certo;
- envia `credentials: 'include'`;
- aceita `X-API-Key` opcional;
- trata resposta síncrona e `202`;
- preserva `correlation_id`.

## 12. Explicação detalhada do exemplo React

### 12.1 `maybeResolveUserEmail(...)`

Tenta obter o e-mail pela mesma sessão federada da plataforma. Se não conseguir, usa o e-mail preenchido manualmente.

### 12.2 `buildEncryptedData(...)`

Usa o helper browser oficial para:

- opcionalmente injetar e-mail no YAML;
- solicitar a sessão criptográfica efêmera ao backend;
- gerar o envelope `encrypted_data`.

### 12.3 `parseJsonOrThrow(...)`

Lê a resposta como texto e tenta parsear JSON. Isso evita perder a mensagem real do backend quando a resposta vier como texto simples.

### 12.4 `pollUntilDone(...)`

Quando a resposta inicial for `202`, essa função consulta `polling_url` até o status ficar terminal.

### 12.5 `handleSend()`

É o orquestrador do componente. Ele faz a mesma jornada conceitual da UI oficial, mas em versão compacta.

## 13. Troubleshooting para quem está integrando

### Sintoma: “PayloadCrypto não foi carregado”

Causa provável:

- o script browser de criptografia não foi incluído na página.

Ação:

- carregar `plataforma-agentes-ia-crypto.js` antes do bundle React.

### Sintoma: erro de autenticação

Causa provável:

- faltou `credentials: 'include'`;
- faltou `X-API-Key` quando o ambiente exige chave;
- a UI estava usando payload criptografado sem API key explícita no campo.

### Sintoma: backend respondeu 202 e nada apareceu na tela

Causa provável:

- o frontend não implementou polling;
- o código tratou 202 como sucesso final sem acompanhar a task.

### Sintoma: o backend rejeitou a pergunta mesmo com JSON válido

Causa provável:

- o corpo foi montado no formato errado para o modo escolhido;
- faltou `encrypted_data`;
- o YAML criptografado não representa um runtime válido.

## 14. Regras de manutenção

- não invente um cliente HTTP paralelo incompatível com a UI oficial;
- não troque o formato do corpo por conveniência;
- não esconda `correlation_id` quando ele vier do backend;
- não reduza a integração a “só mandar um texto”.

## 15. Checklist final para o estagiário

- Consigo enviar YAML ou payload criptografado.
- Consigo gerar `encrypted_data` no browser.
- Entendi qual endpoint usar por modo.
- Entendi como a UI decide o formato do corpo.
- Consigo tratar resposta `200` e `202`.
- Consigo exibir `correlation_id`.
- Consigo explicar por que a pergunta não vai sozinha para a API.

## 16. Evidências no código

- app/ui/static/js/shared/layout-mestre-api.js (fonte única) e app/ui/static/js/shared/embeddable-chat-runtime.js (componente)
  - Motivo da leitura: confirmar a lógica real de modo, API key, payload e parse da resposta. Desde 2026-06-10 a `ui-webchat-v3.html` é host do componente embutível: o dispatch por modo vive no `enviar()` da fonte única e o componente o consome; o host (`ui-webchat-v3.js`) só repassa contexto/selects (`_aplicarModoPreferencialDaEntradaAtual`, `_resolverApiKeyHost`).
  - Comportamento confirmado: a fonte única troca endpoint e formato do corpo conforme o modo.

- app/ui/static/js/shared/ui-webchat-runtime-utils.js
  - Motivo da leitura: confirmar parse e leitura de `correlation_id`.
  - Símbolos relevantes: `resolveResponseCorrelationId`, `parseResponsePayload`.
  - Comportamento confirmado: o runtime prioriza o payload e depois headers HTTP.

- app/ui/static/js/shared/yaml-access-key-extractor.js
  - Motivo da leitura: confirmar inferência de modo a partir do YAML.
  - Símbolo relevante: `extractPreferredModeFromYaml`.
  - Comportamento confirmado: o YAML pode sugerir `deepagent` ou `workflow`.

- app/ui/static/js/plataforma-agentes-ia-crypto.js
  - Motivo da leitura: confirmar geração de `encrypted_data`.
  - Símbolos relevantes: `buildEncryptedData`, `injectUserEmailInYaml`.
  - Comportamento confirmado: o envelope é gerado no browser via sessão criptográfica efêmera.

- src/api/routers/rag_router.py e src/api/schemas/rag_models.py
  - Motivo da leitura: confirmar o contrato público do dispatcher RAG.
  - Comportamento confirmado: `operation='ask'` usa envelope com `payload.question`, `user_email`, `execution_mode` e `encrypted_data`.

- src/api/routers/agent_router.py
  - Motivo da leitura: confirmar contrato público do modo Agent/DeepAgent.
  - Comportamento confirmado: o corpo usa `task` e pode usar `mode='deepagent'`.

- src/api/routers/workflow_router.py
  - Motivo da leitura: confirmar contrato público do modo Workflow.
  - Comportamento confirmado: o corpo usa `message`, `thread_id`, `execution_mode` e `encrypted_data`.

- src/api/routers/streaming_router.py
  - Motivo da leitura: confirmar polling assíncrono.
  - Comportamento confirmado: `/api/v1/status/{task_id}` devolve `status`, `result`, `error` e metadados operacionais.
