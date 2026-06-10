# Manual técnico: exemplo React para o caminho WebChat/RAG

## 1. Onde o fluxo real está confirmado no código

A base real para esta implementação está nos seguintes pontos do repositório:

- `app/ui/static/js/shared/embeddable-chat-runtime.js` — o componente de chat embutível, caminho oficial atual. Em `enviarPergunta()` delega para o cliente canônico e decide o polling assíncrono.
- `app/ui/static/js/shared/layout-mestre-api.js` — o cliente HTTP canônico. Monta o `requestBody` com `operation='ask'`, `encrypted_data` e `execution_mode`, define `endpoint = '/rag/execute'` e envia os headers `X-API-Key` e `X-Prometeu-UI-Session-Required`.
- `app/ui/static/js/plataforma-agentes-ia-crypto.js` — o helper browser oficial gera `encrypted_data` com `PayloadCrypto.buildEncryptedData(...)` (handshake interno em `/crypto/session-key`).
- `src/api/routers/rag_router.py` — o backend expõe o endpoint `/rag/execute` e delega para o runtime oficial do RAG. O corpo é o envelope `{ operation: 'ask', payload: { ... } }`.
- `src/api/routers/crypto_router.py` — o endpoint `/crypto/session-key` gera a chave pública efêmera usada para proteger a chave Fernet.

> Se você precisa de uma interface React inteira própria (sem usar o componente embutível), o guia completo, com polling assíncrono e HIL, é o [GUIA-INTEGRADOR-CHAT-PLATAFORMA.md](../usuario/GUIA-INTEGRADOR-CHAT-PLATAFORMA.md).

## 2. Como montar um cliente React mínimo

O exemplo mais curto é este:

```jsx
const encryptedData = await window.PayloadCrypto.buildEncryptedData({
  yamlContent: window.PayloadCrypto.injectUserEmailInYaml(yamlText, userEmail),
  filename: 'rag-config-webchat.yaml',
  baseUrl: 'http://localhost:5555',
});

const response = await fetch('http://localhost:5555/rag/execute', {
  method: 'POST',
  credentials: 'include',
  headers: {
    'Content-Type': 'application/json',
    'X-API-Key': apiKey || '',               // ou deixe vazio se o YAML tiver authentication.access_key
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
```

Esse trecho é a versão mínima do que o WebChat real já faz. Os headers `X-API-Key` e `X-Prometeu-UI-Session-Required` são os mesmos que o cliente canônico envia em produção — `credentials: 'include'` sozinho não basta.

## 3. Como integrar no React

1. carregue `plataforma-agentes-ia-crypto.js` antes de usar `window.PayloadCrypto`;
2. cole o YAML do usuário em um `textarea` ou estado do componente;
3. gere `encrypted_data` com `PayloadCrypto.buildEncryptedData(...)`;
4. mande `operation='ask'` para `/rag/execute`;
5. trate a resposta JSON com `answer`, `sources`, `correlation_id` e `metrics`.

## 4. Exemplo completo

O arquivo de exemplo em `examples/react_webchat_client_example.jsx` mostra uma implementação mínima com `useState`, `fetch` e `window.PayloadCrypto`.

## 5. Observações de operação

- a autenticação tem duas fontes alternativas: o header `X-API-Key` **ou** a chave em `authentication.access_key` dentro do YAML; basta uma delas, mais o e-mail do usuário (sempre obrigatório);
- se o backend responder em `202` (ou retornar `status_url`/`polling_url`/`stream_url` no corpo), trate como execução assíncrona e faça polling em `/api/v1/status/{task_id}` com o header `X-API-Key`, até o status virar `completed`, `failed` ou `cancelled` (e `paused`/`awaiting_human_decision` quando houver HIL);
- o `correlation_id` vem do header `X-Correlation-Id` e/ou do corpo; mantenha esse valor para auditoria e diagnóstico.

## 6. Diagnóstico rápido

Se o cliente falhar:

1. confirme se `window.PayloadCrypto` está disponível;
2. confirme se `/crypto/session-key` responde;
3. confirme se o YAML é válido para o backend;
4. confirme se o `fetch` está enviando `credentials: 'include'` quando necessário;
5. leia o `correlation_id` da resposta para investigar no log do backend.

## 7. Limites confirmados

- a lógica de criptografia é responsabilidade do helper browser oficial da plataforma;
- esta documentação não inventa um contrato paralelo para o frontend;
- a integração depende do runtime real do WebChat e do endpoint oficial `/rag/execute`.
