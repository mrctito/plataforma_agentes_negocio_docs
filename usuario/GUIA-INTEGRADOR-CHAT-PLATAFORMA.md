# Guia do Integrador: construindo uma UI própria de chat sobre a API da Plataforma

Este guia é para o **integrador parceiro** que vai construir uma **interface própria**
(uma tela web sua, em React, Vue, JS puro, ou até um app móvel/desktop com um browser
embutido) e conversar **direto com a API** da Plataforma de Agentes de IA, **sem usar o
componente de chat embutível pronto** (`PrometeuEmbeddableChatRuntime`).

Tudo aqui foi escrito a partir da leitura do código real do repositório. Onde um detalhe
não pôde ser confirmado no código, isso está declarado explicitamente — nada foi inventado.

> Leitura nível 101: este documento parte do zero. Cada termo técnico inevitável é
> explicado na primeira vez que aparece. Você não precisa conhecer o backend para seguir.

---

## 0. Antes de tudo: você precisa mesmo construir do zero?

A plataforma já oferece um **componente de chat pronto e reutilizável**, o
`PrometeuEmbeddableChatRuntime`. Ele renderiza a conversa (lista de mensagens, campo de
texto, botão enviar, linha de status com o identificador de rastreamento), monta o pedido,
cifra o YAML, chama o endpoint certo de cada modo e faz o acompanhamento assíncrono — tudo
isso seguindo o contrato oficial, sem você reimplementar nada.

**Caminho recomendado (na esmagadora maioria dos casos):** embutir esse componente. Ele é o
padrão oficial da própria plataforma. Veja como em
[GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md](GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md). Ele existe
justamente para evitar que cada tela reinvente o chat e espalhe contratos divergentes.

**Quando este guia (UI própria) faz sentido:**

- você precisa de uma experiência de conversa com **visual e comportamento totalmente seus**,
  que o componente não cobre;
- você está integrando **fora de um browser** (um backend seu, um app nativo, um bot);
- você quer entender **exatamente** o que vai no fio HTTP para auditar ou portar para outra
  linguagem.

Se a sua dúvida for "como monto o corpo HTTP exato?", leia também o lado técnico da montagem
do payload em
[README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md](../tecnico/README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md), e os
exemplos prontos em [README-EXEMPLOS-INTEGRACAO-API.md](README-EXEMPLOS-INTEGRACAO-API.md).
O catálogo completo de endpoints está em [API-ENDPOINTS-SWAGGER.md](../tecnico/API-ENDPOINTS-SWAGGER.md).

Uma observação honesta: construir do zero significa que **você** passa a ser responsável por
cifrar o YAML corretamente, montar o envelope, escolher o endpoint por modo, ler o
identificador de rastreamento e fazer o polling assíncrono. O componente já faz tudo isso. Não
existe "promessa" de que sua UI própria seja melhor — ela só é necessária quando o requisito
visual ou de plataforma realmente exige.

---

## 1. Os 4 modos de conversa e seus endpoints

A plataforma atende quatro modos de conversa. Cada modo tem **um endpoint próprio** e um
**formato de corpo próprio**. Isso está codificado no cliente oficial em
`app/ui/static/js/shared/plataforma-agentes-ia-crypto.js` (função `buildRequestPayload`) e em
`app/ui/static/js/shared/layout-mestre-api.js` (`LAYOUT_API_CONFIG.endpoints` e os métodos
`enviarQA`/`enviarAgente`/`enviarWorkflow`), e confirmado nos routers do backend.

| Modo (conceito)        | Valor de `mode` na UI | Endpoint HTTP         | Onde os campos ficam no corpo |
|------------------------|-----------------------|-----------------------|-------------------------------|
| Q&A (RAG)              | `qa`                  | `POST /rag/execute`   | Dentro de `{ operation: "ask", payload: { ... } }` |
| Agente                 | `agent`               | `POST /agent/execute` | No **topo** do corpo          |
| DeepAgent (supervisor) | `deepagent`           | `POST /agent/execute` | No topo do corpo, com `mode: "deepagent"` |
| Workflow (grafo)       | `workflow`            | `POST /workflow/execute` | No topo do corpo            |

Glossário rápido:

- **RAG** (Retrieval-Augmented Generation): o modo de "perguntas e respostas" sobre uma base
  de conhecimento. O sistema busca trechos relevantes e gera a resposta a partir deles.
- **Agente / DeepAgent**: executam uma "tarefa" usando ferramentas. DeepAgent é o caminho
  oficial de supervisão; no contrato público de `/agent/execute` o único `mode` aceito é
  `deepagent` (o valor `agent` é legado e o boundary rejeita explicitamente — confirmado em
  `agent_router.py`, validador `validate_mode`).
- **Workflow**: um grafo determinístico de passos. Recebe uma "mensagem".

Os endpoints auxiliares que você também vai usar:

| Função                         | Endpoint HTTP                          |
|--------------------------------|----------------------------------------|
| Handshake de criptografia      | `POST /crypto/session-key`             |
| Status de execução assíncrona  | `GET /api/v1/status/{task_id}`         |
| Stream (SSE) de status         | `GET /api/v1/status/stream/{task_id}`  |
| Retomada de pausa humana (HIL) | `POST /agent/continue` (DeepAgent) · `POST /workflow/continue` (Workflow) |

> Confirmado lendo: `LAYOUT_API_CONFIG.endpoints` (layout-mestre-api.js:30-35);
> `buildRequestPayload` (plataforma-agentes-ia-crypto.js:972-1041); `RagExecuteEnvelope`
> (rag_router.py:1179-1194); `AgentRequest` (agent_router.py:120-173); `WorkflowRequest`
> (workflow_router.py:89-133); `streaming_router.py:155-156` (`prefix="/status"` e
> `prefix="/api/v1/status"`).

---

## 2. Pré-requisitos: o que você precisa ter em mãos

Para fazer qualquer chamada de conversa, você precisa de **três coisas**, sendo que duas
delas são **alternativas** (basta uma):

1. **E-mail do usuário** — sempre **obrigatório**. É usado para auditoria. Sem ele, o backend
   recusa (campos `user_email` são `Field(...)` obrigatórios em `AgentRequest` e
   `WorkflowRequest`; no Q&A o e-mail é resolvido a partir do `payload.user_email`).

2. **UMA credencial** — você escolhe **uma** das duas formas abaixo. Elas são **alternativas,
   não cumulativas**: nunca envie as duas juntas como se ambas fossem obrigatórias.
   - **Opção A — header `X-API-Key`**: você manda a chave de API no header HTTP `X-API-Key`.
   - **Opção B — `authentication.access_key` dentro do YAML**: a chave de API vai escrita no
     próprio YAML, no caminho `authentication.access_key`. Nesse caso você **não** precisa do
     header.
   > Confirmado em `src/api/security/user_auth.py`: a chave é resolvida do header `X-API-Key`
   > **ou** do YAML `authentication.access_key` (função `_extract_access_key_from_auth_dict`,
   > linhas ~379-397; mensagem de diagnóstico `_build_missing_api_key_detail`, linhas
   > ~285-323: "Origens aceitas: header X-API-Key ou YAML authentication.access_key").

3. **O YAML do agente / workflow / Q&A** — o arquivo de configuração que define o que será
   executado (qual base de conhecimento, quais agentes, qual workflow, qual modelo). A
   plataforma é 100% configurada por YAML. **Você não inventa esse YAML**: ele é fornecido
   pela plataforma para o seu caso de uso.

> Reforço de campo crítico (regra do repositório, `app/ui/CLAUDE.md`): e-mail e chave de API
> são **campos críticos** e **falham fechado** — nunca use um valor padrão silencioso para
> "destravar". Se faltar o e-mail, ou faltar **as duas** fontes de credencial, a chamada deve
> falhar com erro claro, não com um default arbitrário.

### 2.1. Cuidado decisivo: NÃO envie o header de sessão da UI interna

Há um detalhe que separa "UI própria de integrador" de "tela interna da plataforma" e que
**não está no Swagger** — só no código. O cliente interno (`layout-mestre-api.js`) envia, em
todas as chamadas, o header:

```
X-Prometeu-UI-Session-Required: 1
```

Esse header **liga uma exigência extra**: quando ele chega com valor verdadeiro, o backend
passa a **exigir uma sessão web federada** (um cookie de login do navegador na própria
plataforma), além da chave de API. Isso existe para proteger ações disparadas pela UI oficial.

**Para um integrador externo que autentica com `X-API-Key` (ou com `access_key` no YAML), você
NÃO deve enviar esse header.** Se você enviá-lo sem ter a sessão federada, a requisição é
bloqueada com **HTTP 401** ("Sessão web autenticada é obrigatória para esta ação da UI.").

> Confirmado em `src/api/security/permission_registry.py`:
> `_request_marks_official_ui_action` (linhas 132-134) só considera "ação oficial da UI" se o
> header `X-Prometeu-UI-Session-Required` vier com valor verdadeiro (`1`, `true`, `yes`,
> `sim`, `on`); `_should_require_federated_session` (141-149) então liga
> `_enforce_required_federated_session` (191-204), que lança 401 quando não há sessão
> federada. Os endpoints `/rag/execute`, `/agent/execute` e `/workflow/execute` são marcados
> `ui_session_required=True` (rag_router.py:2285; agent_router.py:2706; workflow_router.py:496),
> mas essa marcação **só passa a exigir a sessão federada quando o header acima é enviado** —
> que é exatamente o que a UI interna faz e o integrador externo **não** deve fazer.

Em uma frase: **integrador externo usa só `X-API-Key` (ou `access_key` no YAML) + e-mail, e
omite o header `X-Prometeu-UI-Session-Required`.**

---

## 3. O fluxo ponta a ponta, explicado em 101

O YAML traz a configuração do seu caso de uso e, às vezes, a própria chave de API. Por isso
ele **nunca trafega em texto puro**: a plataforma usa **criptografia híbrida** para protegê-lo.
Antes de explicar o código, entenda a sequência:

```
[1] Handshake          POST /crypto/session-key
                       → o servidor devolve uma chave pública RSA temporária e um session_id

[2] Cifrar o YAML      no seu cliente:
                       - gera uma chave simétrica Fernet aleatória (rápida, para o YAML grande)
                       - cifra o YAML com essa chave Fernet
                       - "embrulha" a chave Fernet com a chave pública RSA do servidor
                       - monta o envelope encrypted_data = { session_id, wrapped_key, encrypted_yaml, ... }

[3] Executar           POST no endpoint do modo (/rag/execute, /agent/execute, /workflow/execute)
                       enviando o envelope encrypted_data + question/task/message + user_email
                       + header X-API-Key (se você escolheu a Opção A)

[4] Ler o id           a resposta devolve o X-Correlation-Id (header e/ou corpo).
                       Esse é o identificador oficial de rastreamento. Você só LÊ e EXIBE; nunca cria.

[5] Polling (só async) se você pediu execução assíncrona, o corpo traz um task_id e uma polling_url;
                       você consulta GET /api/v1/status/{task_id} até o status virar completed/failed.
```

Por que esse desenho? Glossário:

- **Handshake**: o "aperto de mãos" inicial. O servidor cria um **par de chaves RSA** só para
  esta operação: guarda a chave **privada** e te entrega a **pública**. A pública só serve
  para *cifrar*; só quem tem a privada (o servidor) consegue *decifrar*.
- **RSA** (criptografia assimétrica): segura para proteger coisas **pequenas** (como uma chave
  de 32 bytes). É lenta para dados grandes.
- **Fernet / AES** (criptografia simétrica): rápida para dados **grandes** (o YAML inteiro).
  Usa a mesma chave para cifrar e decifrar.
- **Criptografia híbrida**: combina os dois. Cifra o YAML com Fernet (rápido) e protege só a
  chave Fernet com RSA (seguro). É o melhor dos dois mundos.
- **`session_id`**: identifica, no servidor, qual par de chaves RSA usar para decifrar. A
  sessão **expira** (TTL — "time to live", tempo de vida); por isso você refaz o handshake
  para cada operação.
- **`X-Correlation-Id`** (identificador de correlação): o "número do protocolo" que segue a
  execução do início ao fim no backend. Serve para você rastrear/depurar uma chamada
  específica nos logs do servidor.

> Confirmado em `src/security/payload_crypto.py` (esquema `FERNET+RSA-OAEP`, `wrap_symmetric_key`
> com `RSA-OAEP` + `SHA256`, `build_encrypted_payload`); `plataforma-agentes-ia-crypto.js`
> (`buildEncryptedData`, `acquireSessionKey`, `generateFernetKey`, `fernetEncrypt`,
> `wrapKeyWithPublic`); `crypto_router.py` (`SessionKeyResponse` com `session_id`,
> `public_key_pem`, `expires_at`, `ttl_seconds`).

### 3.1. O formato exato do handshake e do envelope

`POST /crypto/session-key` (sem corpo) devolve:

```json
{
  "session_id": "uuid-da-sessao",
  "public_key_pem": "-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----",
  "expires_at": "2026-06-07T15:30:00Z",
  "ttl_seconds": 300
}
```

O envelope `encrypted_data` que você monta e envia tem este formato (confirmado em
`build_encrypted_payload` no backend e em `buildEncryptedData` no JS):

```json
{
  "session_id": "uuid-da-sessao",
  "wrapped_key": "<chave Fernet protegida com RSA, em Base64>",
  "encrypted_yaml": "<YAML cifrado com Fernet, em Base64>",
  "original_filename": "config.yaml",
  "encryption_scheme": "FERNET+RSA-OAEP",
  "yaml_operational_contract": "root_user_session_only_v1"
}
```

> Nota de fidelidade: o helper Python oficial (`build_encrypted_payload`) **não** inclui o
> campo `yaml_operational_contract` no dicionário retornado; o helper JS (`buildEncryptedData`)
> **inclui**. O backend aceita o envelope nos dois caminhos. Os campos imprescindíveis para a
> decriptação são `session_id`, `wrapped_key` e `encrypted_yaml`; `original_filename` e
> `encryption_scheme` são metadados.

---

## 4. Exemplo completo em JavaScript puro (sem o componente)

Este exemplo é **autossuficiente e funcional no browser**. Ele faz o handshake, cifra o YAML,
chama o endpoint do modo, lê o `X-Correlation-Id` e faz polling se a execução for assíncrona.

Você tem duas formas de cifrar no browser:

- **Forma simples (recomendada se a sua UI roda na mesma origem da plataforma):** reusar o
  helper oficial `window.PayloadCrypto.buildEncryptedData(...)` (arquivo
  `plataforma-agentes-ia-crypto.js`). Você não reimplementa criptografia nenhuma.
- **Forma "do zero" (se você não pode carregar aquele script):** replicar o algoritmo com a
  Web Crypto API. A seção 4.3 mostra exatamente o que replicar e os pontos onde é fácil errar.

### 4.1. Usando o helper oficial `PayloadCrypto` (caminho mais curto)

```html
<!-- Carregue o helper oficial ANTES do seu script -->
<script src="/ui/static/js/plataforma-agentes-ia-crypto.js"></script>
<script>
/**
 * Cliente mínimo de chat sobre a API da Plataforma, sem o componente embutível.
 * Usa o helper oficial window.PayloadCrypto para cifrar o YAML.
 */
class ChatPlataforma {
  /**
   * @param {Object} cfg
   * @param {string} cfg.apiBaseUrl   - Ex.: "https://api.suaempresa.com" (sem barra no fim)
   * @param {string} cfg.userEmail    - E-mail do usuário (OBRIGATÓRIO)
   * @param {string} cfg.yamlContent  - YAML do agente/workflow/Q&A em texto puro
   * @param {string} [cfg.apiKey]     - X-API-Key. OPCIONAL se o YAML já trouxer authentication.access_key
   * @param {string} cfg.mode         - "qa" | "agent" | "deepagent" | "workflow"
   * @param {string} [cfg.executionMode] - "auto" | "direct_sync" | "direct_async" (default "auto")
   */
  constructor(cfg) {
    if (!cfg.userEmail) {
      // Campo crítico: falha fechada, sem default silencioso.
      throw new Error("userEmail é obrigatório.");
    }
    if (!cfg.yamlContent && !cfg.apiKey) {
      // Precisa de pelo menos UMA fonte de credencial.
      throw new Error("Informe o YAML (com authentication.access_key) ou um apiKey (X-API-Key).");
    }
    this.apiBaseUrl = (cfg.apiBaseUrl || window.location.origin).replace(/\/$/, "");
    this.userEmail = cfg.userEmail;
    this.yamlContent = cfg.yamlContent;
    this.apiKey = (cfg.apiKey || "").trim();
    this.mode = (cfg.mode || "qa").toLowerCase();
    this.executionMode = cfg.executionMode || "auto";
    // O id oficial vem SEMPRE da resposta; nunca é criado aqui.
    this.lastCorrelationId = null;
  }

  /** Monta os headers. Note: NÃO enviamos X-Prometeu-UI-Session-Required. */
  _headers() {
    const headers = { "Content-Type": "application/json" };
    if (this.apiKey) {
      headers["X-API-Key"] = this.apiKey;  // Opção A. Opção B é deixar a chave no YAML.
    }
    return headers;
  }

  /** Lê o correlation_id oficial da resposta (header tem prioridade sobre o corpo). */
  _lerCorrelationId(response, body) {
    const fromHeader = (response.headers.get("X-Correlation-Id") || "").trim();
    if (fromHeader) return fromHeader;
    const b = body || {};
    return (
      b.correlation_id || b.correlationId ||
      (b.detail && (b.detail.correlation_id || b.detail.correlationId)) || null
    );
  }

  /** Cifra o YAML usando o helper oficial. Ele faz o handshake internamente. */
  async _cifrarYaml() {
    if (typeof window.PayloadCrypto?.buildEncryptedData !== "function") {
      throw new Error("plataforma-agentes-ia-crypto.js não foi carregado.");
    }
    return window.PayloadCrypto.buildEncryptedData({
      yamlContent: this.yamlContent,
      filename: "config.yaml",
      baseUrl: this.apiBaseUrl,
    });
  }

  /** Constrói endpoint + corpo conforme o modo (mesma lógica de buildRequestPayload). */
  _montarRequisicao(texto, encryptedData) {
    if (this.mode === "qa") {
      return {
        url: `${this.apiBaseUrl}/rag/execute`,
        body: {
          operation: "ask",
          payload: {
            question: texto,
            user_email: this.userEmail,
            format: "json",
            execution_mode: this.executionMode,
            encrypted_data: encryptedData,
          },
        },
      };
    }
    if (this.mode === "agent" || this.mode === "deepagent") {
      const body = {
        task: texto,
        user_email: this.userEmail,
        format: "json",
        execution_mode: this.executionMode,
        encrypted_data: encryptedData,
      };
      if (this.mode === "deepagent") body.mode = "deepagent";
      return { url: `${this.apiBaseUrl}/agent/execute`, body };
    }
    // workflow
    return {
      url: `${this.apiBaseUrl}/workflow/execute`,
      body: {
        message: texto,
        user_email: this.userEmail,
        thread_id: null,
        format: "json",
        execution_mode: this.executionMode,
        encrypted_data: encryptedData,
      },
    };
  }

  /**
   * Envia uma pergunta/tarefa/mensagem e devolve { correlationId, payload }.
   * Se a execução for assíncrona, faz polling até concluir.
   */
  async enviar(texto, { onStatus } = {}) {
    const dizer = (m) => onStatus && onStatus(m);
    dizer("Cifrando configuração...");
    const encryptedData = await this._cifrarYaml();

    const { url, body } = this._montarRequisicao(texto, encryptedData);
    dizer("Enviando...");
    const response = await fetch(url, {
      method: "POST",
      headers: this._headers(),
      body: JSON.stringify(body),
    });

    const raw = await response.text();
    let payload = {};
    try { payload = raw ? JSON.parse(raw) : {}; } catch { payload = { raw }; }

    // O correlation_id oficial vem SEMPRE da resposta (inclusive em erro).
    this.lastCorrelationId = this._lerCorrelationId(response, payload);

    if (!response.ok) {
      const msg =
        (payload.detail && (payload.detail.message || payload.detail)) ||
        payload.error || payload.message || `Erro HTTP ${response.status}`;
      const err = new Error(typeof msg === "string" ? msg : JSON.stringify(msg));
      err.status = response.status;
      err.correlationId = this.lastCorrelationId;  // pode ser null se o backend não devolveu
      throw err;
    }

    // Detecta execução assíncrona: HTTP 202 OU corpo com task_id + polling_url.
    const taskId = payload.task_id || payload.taskId ||
      (payload.detail && (payload.detail.task_id || payload.detail.taskId));
    const pollingUrl = payload.polling_url || payload.status_url ||
      (payload.detail && (payload.detail.polling_url || payload.detail.status_url));

    if (response.status === 202 || (taskId && pollingUrl)) {
      dizer("Execução assíncrona iniciada. Acompanhando...");
      const finalData = await this._aguardarConclusao(pollingUrl, onStatus);
      // O status final pode trazer um correlation_id mais recente.
      this.lastCorrelationId = this._lerCorrelationId(
        { headers: { get: () => null } }, finalData
      ) || this.lastCorrelationId;
      return { correlationId: this.lastCorrelationId, payload: finalData };
    }

    return { correlationId: this.lastCorrelationId, payload };
  }

  /** Faz polling em GET /api/v1/status/{task_id} até completed/failed/paused/cancelled. */
  async _aguardarConclusao(pollingUrl, onStatus) {
    const url = new URL(pollingUrl, this.apiBaseUrl).toString();
    const intervaloMs = 1500;
    const maxErrosSeguidos = 10;
    let errosSeguidos = 0;

    for (;;) {
      let resp;
      try {
        const headers = { Accept: "application/json" };
        if (this.apiKey) headers["X-API-Key"] = this.apiKey;
        resp = await fetch(url, { method: "GET", headers, cache: "no-store" });
      } catch (e) {
        errosSeguidos += 1;
        if (errosSeguidos >= maxErrosSeguidos) throw e;
        await new Promise((r) => setTimeout(r, intervaloMs));
        continue;
      }

      // 404 = a task ainda não apareceu no monitor; aguarde e tente de novo.
      if (resp.status === 404) {
        await new Promise((r) => setTimeout(r, 2000));
        continue;
      }
      if (!resp.ok) {
        errosSeguidos += 1;
        if (errosSeguidos >= maxErrosSeguidos) {
          throw new Error(`Status HTTP ${resp.status} ao consultar a task.`);
        }
        await new Promise((r) => setTimeout(r, intervaloMs));
        continue;
      }

      errosSeguidos = 0;
      const data = await resp.json();
      const status = String(data.status || "pending").toLowerCase();
      if (onStatus && data.message) onStatus(data.message);

      if (status === "completed") return data;
      if (status === "paused" || status === "awaiting_human_decision") return data;
      if (status === "failed") {
        const err = new Error(data.error || "Execução assíncrona falhou.");
        err.data = data;
        throw err;
      }
      if (status === "cancelled") {
        const err = new Error(data.message || "Execução cancelada.");
        err.data = data;
        throw err;
      }
      await new Promise((r) => setTimeout(r, intervaloMs));
    }
  }
}

// --- Uso ---
(async () => {
  const chat = new ChatPlataforma({
    apiBaseUrl: "https://api.suaempresa.com",
    userEmail: "usuario@empresa.com",
    yamlContent: document.getElementById("yaml").value, // YAML fornecido pela plataforma
    apiKey: "SUA_CHAVE",       // omita se o YAML já tiver authentication.access_key
    mode: "qa",                // qa | agent | deepagent | workflow
    executionMode: "auto",     // auto | direct_sync | direct_async
  });

  try {
    const { correlationId, payload } = await chat.enviar("Qual é a política de férias?", {
      onStatus: (m) => console.log("status:", m),
    });
    // Extrai o texto da resposta conforme o modo (campos possíveis no corpo).
    const texto = payload.answer || payload.response || payload.final_response ||
                  payload.message || "Resposta recebida.";
    console.log("Resposta:", texto);
    console.log("Correlation ID (exibir ao usuário):", correlationId);
  } catch (err) {
    console.error("Erro:", err.message, "| correlation_id:", err.correlationId || "(não veio)");
  }
})();
</script>
```

### 4.2. Por que cada decisão do exemplo é fiel ao código

- O **endpoint e o formato do corpo por modo** replicam exatamente `buildRequestPayload`
  (plataforma-agentes-ia-crypto.js:972-1041): `qa` usa o envelope `{operation:"ask", payload:{...}}`;
  `agent`/`deepagent` põem os campos no topo (com `mode:"deepagent"` quando for DeepAgent);
  `workflow` usa `message` + `thread_id`.
- O **`format: "json"`** é o que a UI oficial envia (layout-mestre-api.js: `format: 'json'` em
  `enviarQA`/`enviarAgente`/`enviarWorkflow`). O backend aceita `text` ou `json`
  (`Literal["text","json"]` em `AgentRequest`/`WorkflowRequest`); `json` facilita o parsing.
- A **leitura do `X-Correlation-Id`** (header primeiro, corpo depois) replica
  `extrairCorrelationIdResposta` (layout-mestre-api.js:64-114) e o `resolveCorrelationId` do
  componente embutível (embeddable-chat-runtime.js:357-369).
- A **detecção de assíncrono** (HTTP 202 ou `task_id` + `polling_url` no corpo) e o **polling**
  replicam o `_executarRequisicao` (layout-mestre-api.js, trecho `response.status === 202`) e o
  `WebchatAsyncRuntime.waitForTaskCompletion` (ui-webchat-async-runtime.js): mesmos estados
  terminais (`completed`, `failed`, `cancelled`, `paused`/`awaiting_human_decision`), mesmo
  tratamento de **404 como "task ainda não pronta"** (espera e repete) e a mesma política de
  erros consecutivos.
- **Não enviar** `X-Prometeu-UI-Session-Required` é o ponto da seção 2.1.

### 4.3. Se você NÃO puder usar o helper `PayloadCrypto` (replicando a cifra)

Se a sua UI roda em outra origem e você não pode carregar o `plataforma-agentes-ia-crypto.js`,
você precisa replicar o que `buildEncryptedData` faz. O algoritmo exato está em
`plataforma-agentes-ia-crypto.js` (funções `generateFernetKey`, `fernetEncrypt`,
`wrapKeyWithPublic`) e deve ser seguido **ao pé da letra**, porque o servidor decifra com a
biblioteca Fernet padrão do Python. Os pontos onde é fácil errar:

1. **Chave Fernet = 32 bytes aleatórios**, codificados em **Base64URL**. Dentro do Fernet,
   esses 32 bytes se dividem em: **primeiros 16 bytes** = chave de HMAC-SHA256 (autenticação);
   **últimos 16 bytes** = chave AES-128-CBC (cifra). (`generateFernetKey`, `fernetEncrypt`.)

2. **O token Fernet** é montado como:
   `versão(1 byte = 0x80) | timestamp(8 bytes, big-endian, Unix epoch em segundos) | IV(16 bytes) | ciphertext | HMAC-SHA256(32 bytes)`,
   onde o HMAC cobre `versão|timestamp|IV|ciphertext`. O ciphertext é AES-128-CBC com **padding
   PKCS#7**. (`fernetEncrypt`, etapas 3.1 a 3.8.)

3. **Atenção ao duplo encode do `encrypted_yaml`** (ponto sutil e fácil de errar): o JS gera o
   token Fernet, o codifica em **Base64URL** e depois **codifica esse texto novamente em
   Base64 padrão**. Em código: `return arrayBufferToBase64(stringToUint8Array(fernetToken))`,
   onde `fernetToken` já é Base64URL. Isso existe "para manter compatibilidade histórica com
   `crypto_manager.py`". Se você gerar o token Fernet e mandar só uma camada de Base64, o
   servidor pode rejeitar. (`fernetEncrypt`, linhas 405-409.)

4. **`wrapped_key` = a chave Fernet (a string Base64URL de 44 chars, em bytes) cifrada com
   RSA-OAEP** usando a `public_key_pem` da sessão, hash **SHA-256**, e o resultado em **Base64
   padrão** (não URL-safe). Importante: o que vai para o RSA são os **bytes da string Base64URL
   da chave**, não os 32 bytes crus — `wrapKeyWithPublic` faz
   `stringToUint8Array(symmetricKeyB64)`. (`wrapKeyWithPublic`, etapas 4.1 a 4.5.)

5. **Importação da chave pública**: PEM → DER (remove os cabeçalhos `BEGIN/END PUBLIC KEY` e os
   espaços, decodifica Base64), importada como `spki` com algoritmo `RSA-OAEP` + `SHA-256`.
   (`pemToArrayBuffer`, `wrapKeyWithPublic`.)

> Recomendação honesta: replicar criptografia à mão é a parte mais arriscada da integração. Se
> a sua integração **não** for no browser (por exemplo, um backend seu em Python), o caminho
> mais seguro e curto é usar a biblioteca `cryptography` (Fernet) diretamente, exatamente como
> faz o cliente de referência `examples/rag_api_client.py` (que usa
> `encrypt_yaml_and_keys_from_file` de `src/security/payload_crypto.py`). Os exemplos prontos
> em outras linguagens estão em [README-EXEMPLOS-INTEGRACAO-API.md](README-EXEMPLOS-INTEGRACAO-API.md).

---

## 5. Exemplo em React (componente de chat mínimo)

Componente React funcional que reusa a classe `ChatPlataforma` da seção 4.1. Em React, a regra
de ouro é a mesma: **o `correlation_id` é estado lido da resposta, nunca gerado no cliente.**

```jsx
import { useState, useRef, useCallback } from "react";

// Reuse a classe ChatPlataforma da seção 4.1 (ela depende de window.PayloadCrypto carregado).
// import { ChatPlataforma } from "./chat-plataforma";

/**
 * Chat mínimo consumindo a API da Plataforma direto, sem o componente embutível.
 *
 * Props:
 *  - apiBaseUrl, userEmail, yamlContent, apiKey, mode, executionMode
 */
export function ChatMinimo({ apiBaseUrl, userEmail, yamlContent, apiKey, mode = "qa", executionMode = "auto" }) {
  const [mensagens, setMensagens] = useState([]); // [{ autor, texto, correlationId }]
  const [pergunta, setPergunta] = useState("");
  const [status, setStatus] = useState("");
  const [enviando, setEnviando] = useState(false);

  // O cliente é criado uma vez e reaproveitado (ele refaz o handshake a cada envio).
  const clienteRef = useRef(null);
  if (!clienteRef.current) {
    clienteRef.current = new ChatPlataforma({
      apiBaseUrl, userEmail, yamlContent, apiKey, mode, executionMode,
    });
  }

  const enviar = useCallback(async () => {
    const texto = pergunta.trim();
    if (!texto || enviando) return;

    setMensagens((m) => [...m, { autor: "user", texto }]);
    setPergunta("");
    setEnviando(true);
    setStatus("Processando...");

    try {
      const { correlationId, payload } = await clienteRef.current.enviar(texto, {
        onStatus: setStatus,
      });
      const resposta =
        payload.answer || payload.response || payload.final_response ||
        payload.message || "Resposta recebida.";
      setMensagens((m) => [...m, { autor: "assistant", texto: resposta, correlationId }]);
    } catch (err) {
      setMensagens((m) => [
        ...m,
        { autor: "error", texto: err.message, correlationId: err.correlationId || null },
      ]);
    } finally {
      setEnviando(false);
      setStatus("");
    }
  }, [pergunta, enviando]);

  const onKeyDown = (e) => {
    // Enter envia; Shift+Enter quebra linha — mesmo comportamento do componente oficial.
    if (e.key === "Enter" && !e.shiftKey) {
      e.preventDefault();
      enviar();
    }
  };

  return (
    <div className="chat-minimo">
      <div className="chat-minimo__mensagens">
        {mensagens.map((msg, i) => (
          <div key={i} className={`chat-minimo__msg chat-minimo__msg--${msg.autor}`}>
            <div className="chat-minimo__texto">{msg.texto}</div>
            {msg.correlationId && (
              <small className="chat-minimo__corr">Correlation ID: {msg.correlationId}</small>
            )}
          </div>
        ))}
      </div>

      {status && <div className="chat-minimo__status">{status}</div>}

      <div className="chat-minimo__entrada">
        <textarea
          value={pergunta}
          onChange={(e) => setPergunta(e.target.value)}
          onKeyDown={onKeyDown}
          placeholder="Digite sua pergunta..."
          disabled={enviando}
        />
        <button onClick={enviar} disabled={enviando || !pergunta.trim()}>
          {enviando ? "Enviando..." : "Enviar"}
        </button>
      </div>
    </div>
  );
}
```

Notas fiéis ao componente oficial: **Enter envia / Shift+Enter quebra linha** e **o botão fica
desabilitado durante o envio** (previne envio duplicado) são exatamente o comportamento do
`PrometeuEmbeddableChatRuntime` descrito em
componente-chat-embutivel.md (seção 3) — você
está reproduzindo, na sua UI, o que o componente já entrega de graça.

---

## 6. Tratamento de erro e o `correlation_id`

Regra inegociável (vale para qualquer cliente, próprio ou pronto): **o cliente nunca cria,
deriva nem inventa um `correlation_id`.** Ele apenas **captura, exibe e propaga** o
identificador oficial devolvido pelo endpoint. Isso está no contrato do repositório
(`.claude/rules/log-instructions.md` e `app/ui/CLAUDE.md`) e é o que o componente embutível faz.

O que o backend garante (e o que você deve fazer com isso):

1. **Sucesso**: a resposta traz o `X-Correlation-Id` no **header** e, quando o corpo é JSON, o
   id também aparece no corpo (`correlation_id`). Confirmado: um middleware faz
   `response.headers.setdefault("X-Correlation-Id", correlation_id)` (service_api.py:1535) e
   injeta o id no corpo JSON; o CORS expõe o header `X-Correlation-Id`
   (`expose_headers=["X-Correlation-Id"]`, service_api.py:1305) — sem isso, JavaScript em outra
   origem nem conseguiria **ler** o header.
   > Detalhe de contrato: quando o corpo de sucesso é um **array**, o backend o envolve em
   > `{ correlationId, data }` (regra do `CLAUDE.md §9`). Então, se você esperava uma lista e
   > recebeu um objeto com `data`, procure a lista em `body.data` e o id em `body.correlationId`.

2. **Erro**: o `X-Correlation-Id` também é obrigatório nas respostas de erro. O backend tem um
   tratamento específico para devolver o id mesmo em **HTTP 500** (service_api.py:1440-1451:
   "não devolver um traceback cru SEM X-Correlation-Id"). Então, no seu cliente, **leia o id
   também no caminho de erro** — como o exemplo faz (`err.correlationId = ...`).

3. **Se o id não veio** (header e corpo vazios): isso é **falha de observabilidade do backend**,
   não algo para você "consertar" inventando um id. O comportamento correto da sua UI é exibir
   algo como "Correlation ID: aguardando backend" e registrar a ausência. Esse exato cenário já
   foi observado em produção do projeto (resposta 500 sem `X-Correlation-Id` em modo debug) e
   está registrado como defeito de backend — nunca como permissão para a UI gerar o id.

Como classificar um erro (didático):

- **HTTP 401** "Sessão web autenticada é obrigatória...": você provavelmente enviou o header
  `X-Prometeu-UI-Session-Required` sem ter sessão federada. **Remova esse header** (seção 2.1).
- **HTTP 401 / 403** sobre permissão/credencial: a chave de API está ausente, vazia ou sem
  permissão. Verifique o header `X-API-Key` **ou** o `authentication.access_key` do YAML —
  lembrando que basta **uma** das duas fontes.
- **HTTP 422**: o corpo não bateu com o schema (campo obrigatório faltando, valor inválido).
  Reveja o formato do modo (seção 1) — por exemplo, mandar `task` num corpo de Q&A ou esquecer
  o envelope `operation/payload`.
- **HTTP 500** com mensagem sobre vetor/dataset: normalmente é dado/ambiente (a base de
  conhecimento do YAML não está disponível naquele ambiente), não erro do seu cliente. Use o
  `correlation_id` para que a equipe da plataforma diagnostique pelo log oficial.

---

## 7. Pausa humana (HIL) — confirmado para DeepAgent

**HIL** (Human-in-the-Loop, "humano no circuito") é quando a execução **pausa** para pedir uma
decisão de uma pessoa antes de continuar (por exemplo, aprovar a chamada de uma ferramenta).
No código, o HIL é um recurso **confirmado e oficial do modo DeepAgent** em `/agent/execute`.

Como funciona, ponta a ponta (confirmado em `agent_router.py`):

1. Você chama `POST /agent/execute` com `mode: "deepagent"`. Para receber a pausa **na mesma
   resposta HTTP**, use `execution_mode: "direct_sync"`.

2. Se a execução pausar, a resposta vem com **HTTP 200** e um envelope `hil` no corpo. Os
   campos relevantes (confirmados no exemplo `hil_pending` em agent_router.py:309-356 e no
   `AgentResponse.hil`):

   ```json
   {
     "task": "...",
     "response": "DeepAgent pausado aguardando aprovação humana",
     "correlation_id": "<id oficial>",
     "thread_id": "agent_0f4d...",
     "success": true,
     "hil": {
       "pending": true,
       "protocol_version": "hil-http-v1",
       "message": "DeepAgent pausado aguardando aprovação humana",
       "allowed_decisions": ["approve", "reject", "edit"],
       "action_requests": [
         { "name": "calculator", "args": { "expression": "241 * 17" }, "description": "..." }
       ],
       "interrupt_id": "interrupt-calculator-1",
       "interrupt_ids": ["interrupt-calculator-1"],
       "resume_endpoint": "/agent/continue"
     }
   }
   ```

   - `allowed_decisions` diz **quais decisões a sua UI pode oferecer** ao usuário. Só ofereça
     `edit` se ele estiver nessa lista.
   - `action_requests` é a lista do que está aguardando aprovação (a ordem importa para o
     `edit`).
   - `thread_id` e `correlation_id`: você vai **reaproveitar exatamente esses dois valores** na
     retomada.

3. Para retomar, chame `POST /agent/continue` reaproveitando o mesmo `correlation_id` e
   `thread_id`. O contrato exato é o modelo `AgentContinueRequest`
   (`src/api/schemas/agent_hil_models.py:98-133`), com `model_config = ConfigDict(extra="forbid")`
   — ou seja, **campo fora do modelo faz a request falhar com HTTP 422**. Os campos são:

   | Campo            | Obrigatório? | O que é                                                                                  |
   | ---------------- | ------------ | ---------------------------------------------------------------------------------------- |
   | `resume`         | **Sim**      | Objeto `{ "decisions": [ ... ] }` com a(s) decisão(ões) humana(s); a lista não pode ser vazia. |
   | `user_email`     | **Sim**      | E-mail do usuário (usado em auditoria).                                                   |
   | `correlation_id` | **Sim**      | O **mesmo** `correlation_id` devolvido pela pausa.                                        |
   | `thread_id`      | **Sim**      | O **mesmo** `thread_id` devolvido pela pausa (string não vazia).                          |
   | `encrypted_data` | Opcional     | Envelope do YAML cifrado (o **mesmo** formato que você já usa no `/agent/execute`).       |
   | `yaml_config`    | Opcional     | Caminho de um arquivo YAML **no servidor** (ex.: `app/yaml/hil-deepagent-minimo.yaml`).   |
   | `mode`           | Opcional     | `"agent"` (padrão) ou `"deepagent"`. Para retomar pausa de DeepAgent, use `"deepagent"`.  |
   | `format`         | Opcional     | `"text"` (padrão) ou `"json"`.                                                            |
   | `interrupt_ids`  | Opcional     | Lista de IDs de interrupção resolvidos, quando aplicável.                                 |

   **Importante:** `/agent/continue` **aceita `encrypted_data`** (YAML cifrado) **e/ou**
   `yaml_config` (caminho de arquivo no servidor). Os dois são opcionais e alternativos: para um
   integrador externo que trabalha só com YAML cifrado, **envie `encrypted_data`** — não é
   preciso ter arquivo no servidor. Isso está confirmado no boundary
   `resolve_agent_continue_context` (agent_router.py:2277-2327), que chama
   `resolve_yaml_configuration(encrypted_data=request.encrypted_data, yaml_config_path=request.yaml_config or None, ...)`,
   e em `resolve_yaml_configuration` (config_resolution.py:997-1071), cuja **primeira ramificação**
   é justamente decifrar `encrypted_data` (YAML + chaves).

   Corpo de exemplo para um integrador externo, com **YAML cifrado** (o caminho recomendado):

   ```json
   {
     "resume": { "decisions": [ { "type": "approve" } ] },
     "user_email": "usuario@empresa.com",
     "correlation_id": "<o mesmo do passo 2>",
     "thread_id": "<o mesmo do passo 2>",
     "mode": "deepagent",
     "encrypted_data": { "session_id": "...", "wrapped_key": "...", "encrypted_yaml": "..." }
   }
   ```

   - `resume.decisions` segue a ordem de `hil.action_requests`. Tipos aceitos pelo modelo
     `AgentResumeDecision` (agent_hil_models.py:64-84): `approve`, `edit`, `reject`, `respond`.
     Para `edit`, o objeto **exige** `edited_action: { name, args }` (e `edited_action` só pode
     ser enviado quando `type=edit`).
   - A resposta de `/agent/continue` vem com **HTTP 200** tanto para aprovado quanto para
     rejeitado; o resultado final está em `success`, `error` e `metrics.status` (confirmado em
     `AGENT_CONTINUE_RESPONSE_EXAMPLES` e `AGENT_CONTINUE_OPENAPI_RESPONSES`).

Exemplo JS de retomada:

```js
// `encryptedData` é o MESMO envelope que você já monta para o /agent/execute
// (POST /crypto/session-key -> cifra o YAML). Reaproveite-o aqui.
async function retomarHil(chat, { correlationId, threadId, encryptedData, tipoDecisao /* "approve"|"reject"|"edit"|"respond" */ }) {
  const decisao = { type: tipoDecisao };
  // Para edit, o modelo EXIGE edited_action junto da decisão:
  // const decisao = { type: "edit", edited_action: { name: "calculator", args: { expression: "241 * 17" } } };
  const resp = await fetch(`${chat.apiBaseUrl}/agent/continue`, {
    method: "POST",
    headers: chat._headers(),  // X-API-Key se houver; NÃO o header de sessão da UI
    body: JSON.stringify({
      resume: { decisions: [decisao] },
      user_email: chat.userEmail,
      correlation_id: correlationId,  // o mesmo da pausa
      thread_id: threadId,            // o mesmo da pausa
      mode: "deepagent",
      encrypted_data: encryptedData,  // YAML cifrado: caminho recomendado p/ integrador externo
      // alternativa: em vez de encrypted_data, use yaml_config: "app/yaml/arquivo.yaml"
      //              (só funciona se o arquivo existir NO SERVIDOR)
    }),
  });
  const data = await resp.json();
  return data; // { success, error, metrics: { status }, response, ... }
}
```

> Como resolver o YAML na retomada — confirmado no código: `/agent/continue` aceita as **duas**
> fontes de YAML, ambas opcionais e alternativas:
>
> - **`encrypted_data`** (envelope do YAML cifrado): use esta opção se você é integrador externo
>   e trabalha com YAML cifrado. É o **mesmo** envelope do `/agent/execute`, e **não** exige
>   nenhum arquivo no servidor. **É o caminho recomendado.**
> - **`yaml_config`** (caminho de arquivo): aponta para um YAML que **precisa existir no
>   servidor** (ex.: `"app/yaml/hil-deepagent-minimo.yaml"`). É o que aparece no exemplo do
>   Swagger, mas serve principalmente para fluxos internos que já têm o arquivo publicado.
>
> Confirmado no boundary `resolve_agent_continue_context` (agent_router.py:2277-2327) e em
> `resolve_yaml_configuration` (config_resolution.py:997-1071). O modelo `AgentContinueRequest`
> (agent_hil_models.py:98-133) declara `yaml_config` e `encrypted_data` como `Optional`, e a
> docstring do campo `encrypted_data` diz literalmente "opcional no continue".

Sobre o polling assíncrono e HIL: no modo `direct_async`, o status pode parar em `paused` ou
`awaiting_human_decision` — o exemplo da seção 4.1 já trata esses estados como terminais do
polling (igual ao `WebchatAsyncRuntime`, ui-webchat-async-runtime.js:230-233), devolvendo o
payload para que a sua UI mostre a pausa e ofereça os botões de decisão.

**Workflow também tem retomada**, mas por um endpoint e um modelo **diferentes**:
`POST /workflow/continue` com `WorkflowContinueRequest` (workflow_router.py:191-227). Diferenças
em relação ao agente: o workflow também aceita `encrypted_data` **e/ou** `yaml_config` (ambos
opcionais), `correlation_id` e `thread_id` são obrigatórios, mas a decisão humana pode vir como
`resume` (tipado, equivalente a approve/reject/edit) **ou** como `human_response` (texto/objeto
livre) — você informa **um** dos dois, nunca os dois juntos, e pelo menos um é obrigatório.
O modo Q&A (`/rag/execute`) **não** tem HIL — é uma pergunta-resposta direta.

---

## 8. Resumo operacional (checklist do integrador)

1. Tenha: **e-mail** + **uma** credencial (`X-API-Key` **ou** `authentication.access_key` no
   YAML) + o **YAML** do seu caso de uso.
2. **NÃO** envie o header `X-Prometeu-UI-Session-Required` (ele é só para a UI interna com
   sessão federada).
3. Para cada envio: `POST /crypto/session-key` → cifre o YAML (helper oficial, ou replique
   Fernet+RSA-OAEP com cuidado no duplo-encode) → monte o corpo **do modo certo** → `POST` no
   endpoint do modo.
4. **Leia** o `X-Correlation-Id` da resposta (header e corpo), **inclusive em erro**, e exiba.
   Nunca crie um id local.
5. Se vier `task_id` + `polling_url` (ou HTTP 202): faça polling em
   `GET /api/v1/status/{task_id}` até `completed`/`failed`/`paused`/`cancelled`.
6. HIL (só DeepAgent/Workflow): se a resposta trouxer `hil.pending`, ofereça
   `hil.allowed_decisions` e retome reaproveitando `correlation_id` e `thread_id` —
   `POST /agent/continue` para DeepAgent, `POST /workflow/continue` para Workflow. Em ambos,
   o YAML pode vir como `encrypted_data` (cifrado, recomendado) ou `yaml_config` (arquivo no
   servidor).
7. **Antes de tudo isto**, reconsidere usar o componente embutível pronto
   ([GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md](GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md)): é o caminho
   recomendado e dispensa reimplementar handshake, cifra, montagem por modo, leitura de id e
   polling.

---

## 9. Documentos relacionados

- [GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md](GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md) — o componente
  pronto (caminho recomendado).
- [README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md](../tecnico/README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md) — o
  detalhe técnico de como o corpo HTTP é montado.
- [README-CONCEITUAL-WEBCHAT-MONTAGEM-PAYLOAD.md](../conceitual/README-CONCEITUAL-WEBCHAT-MONTAGEM-PAYLOAD.md)
  — a visão conceitual da montagem do payload.
- [README-EXEMPLOS-INTEGRACAO-API.md](README-EXEMPLOS-INTEGRACAO-API.md) — clientes de exemplo
  prontos em várias linguagens.
- [API-ENDPOINTS-SWAGGER.md](../tecnico/API-ENDPOINTS-SWAGGER.md) — catálogo de endpoints e o contrato
  OpenAPI publicado.
- .claude/rules/componente-chat-embutivel.md —
  contrato de uso do componente embutível.
