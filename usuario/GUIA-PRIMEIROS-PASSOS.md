# Guia de Primeiros Passos: do zero à primeira pergunta respondida

Este é o **quickstart** da integração de chat/API da Plataforma de Agentes de IA. O objetivo é único e
estreito: sair do nada e conseguir **uma** resposta real da API, com o menor número de passos. Se você nunca
subiu este projeto nem chamou os endpoints, comece por aqui.

> Este guia é deliberadamente mínimo. Quando você quiser montar um chat funcional de verdade (com os 4 modos,
> os 3 modos de execução, tratamento de erro e histórico), siga o
> [TUTORIAL-CHAT-PLATAFORMA.md](TUTORIAL-CHAT-PLATAFORMA.md). Para uma interface 100% própria sem o componente
> oficial, veja o [GUIA-INTEGRADOR-CHAT-PLATAFORMA.md](GUIA-INTEGRADOR-CHAT-PLATAFORMA.md). Para a montagem
> exata do payload, o [README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md](../tecnico/README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md).

## O que você vai conseguir no fim

Ao terminar este guia, você terá:

- a aplicação rodando localmente na porta certa;
- uma credencial (chave de acesso) e um e-mail válidos;
- um YAML de configuração apontado;
- uma chamada que faz o handshake criptográfico, envia uma pergunta e lê a resposta;
- o `correlation_id` da execução em mãos para investigar o log quando precisar.

## Conceitos mínimos antes de começar (nível 101)

Você só precisa entender quatro ideias para o quickstart funcionar:

- **YAML de configuração**: é o arquivo que descreve o que o sistema deve fazer (qual modelo de IA, qual base
  de conhecimento, qual agente). O sistema é configurado por YAML; você não programa o comportamento, você o
  declara. Os YAMLs de exemplo ficam em `app/yaml/`.
- **Chave de acesso (`X-API-Key` / `access_key`)**: é a credencial que autoriza a chamada. Ela pode ir no
  header HTTP `X-API-Key` **ou** já estar embutida no YAML no campo `authentication.access_key`. São fontes
  **alternativas** — basta **uma**. O e-mail do usuário é sempre obrigatório à parte.
- **Handshake criptográfico**: antes de enviar o YAML, o cliente pede uma "sessão" ao backend
  (`POST /crypto/session-key`), recebe uma chave pública temporária e usa essa chave para cifrar o YAML. Isso
  protege configurações sensíveis no transporte. O cliente não envia o YAML em claro.
- **`correlation_id`**: é o identificador oficial da execução, devolvido pelo backend (no header
  `X-Correlation-Id` e/ou no corpo). Ele liga a resposta, os logs e o trabalho assíncrono. O cliente **nunca**
  inventa esse valor — apenas lê e propaga o que o backend devolveu.

## Passo 1: subir a aplicação

A aplicação roda em três processos independentes: **API** (núcleo HTTP), **Worker** (tarefas assíncronas) e
**Scheduler** (tarefas agendadas). Para o quickstart de chat síncrono, a **API** já basta.

Primeiro, ative o ambiente Python:

```bash
source .venv/bin/activate
```

Depois suba a API pelo launcher oficial. O `run.sh` exige flag explícita — sem flag, ele falha de propósito:

```bash
./run.sh +a
```

A flag `+a` sobe a API. Se quiser subir tudo de uma vez (útil quando o seu fluxo usar execução assíncrona ou
ingestão), use:

```bash
./run.sh +a +w +s
```

A porta da API segue estritamente a variável `FASTAPI_PORT` do arquivo `.env`. O valor padrão da aplicação é
**5555**. Não invente outra porta: confira o `.env` se tiver dúvida.

> Observação importante: os exemplos de cliente externo em `examples/` (Python/JavaScript) usam, por padrão,
> `http://localhost:8000` no texto histórico. A porta real do seu ambiente é a do `.env`. Para o ambiente
> local padrão deste repositório, use `http://localhost:5555`. Sempre prefira a porta do `.env` real.

Para confirmar que a API respondeu, faça o health check:

```bash
curl http://localhost:5555/health
```

Uma resposta com `"status": "healthy"` indica que o boundary HTTP está de pé.

> Pegadinha de ambiente: se a API parar de responder depois de ciclos de sobe/desce, trate primeiro como
> **porta presa** (defeito comum em VS Code + WSL), não como bug de endpoint:
> `sudo fuser -k 5555/tcp` e confirme com `sudo lsof -i :5555`.

## Passo 2: preparar credencial, e-mail e YAML

Você precisa de três coisas para a primeira chamada:

1. **Um YAML válido**: use um exemplo existente em `app/yaml/` (por exemplo `app/yaml/rag-config-auto.yaml`).
   Não invente chaves no YAML; use um arquivo já pronto do repositório.
2. **Uma chave de acesso (`access_key`) cadastrada**: ela costuma já estar no próprio YAML, em
   `authentication.access_key`. Se estiver lá, você não precisa de header `X-API-Key` separado.
3. **Um e-mail de usuário**: qualquer e-mail operacional válido para identificar quem faz a chamada (ex.:
   `developer@empresa.com`). O e-mail é sempre obrigatório.

Regra de ouro da credencial (para não travar logo no começo): **YAML e `X-API-Key` são alternativas, não
cumulativos; o e-mail é sempre obrigatório.** Se o YAML já carrega `authentication.access_key`, o header
`X-API-Key` pode ir vazio.

## Passo 3: a primeira chamada (handshake + pergunta + resposta)

O exemplo abaixo é o caminho mínimo no navegador, usando o helper de criptografia oficial da plataforma
(`window.PayloadCrypto`). Ele faz o handshake (`/crypto/session-key`) por dentro de `buildEncryptedData`, cifra
o YAML, envia a pergunta em `/rag/execute` e lê o `correlation_id`. É fiel ao caminho real do produto
(`app/ui/static/js/shared/layout-mestre-api.js` e `app/ui/static/js/plataforma-agentes-ia-crypto.js`).

```javascript
async function primeiraPergunta({ baseUrl, yamlText, question, userEmail, apiKey }) {
  // 1) Injeta o e-mail operacional no YAML ANTES de cifrar e gera o envelope cifrado.
  //    buildEncryptedData faz o handshake em /crypto/session-key internamente.
  const encryptedData = await window.PayloadCrypto.buildEncryptedData({
    yamlContent: window.PayloadCrypto.injectUserEmailInYaml(yamlText, userEmail),
    filename: 'config.yaml',
    baseUrl,
  });

  // 2) Envia a pergunta para /rag/execute (modo qa).
  const response = await fetch(`${baseUrl}/rag/execute`, {
    method: 'POST',
    credentials: 'include',
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': apiKey || '',                 // pode ir vazio se o YAML tiver authentication.access_key
      'X-Prometeu-UI-Session-Required': '1',
    },
    body: JSON.stringify({
      operation: 'ask',
      payload: {
        question,
        user_email: userEmail,
        format: 'json',
        execution_mode: 'auto',                  // 'auto' | 'direct_sync' | 'direct_async'
        encrypted_data: encryptedData,
      },
    }),
  });

  // 3) Lê o correlation_id (header tem prioridade; cai para o corpo). Nunca invente esse valor.
  const correlationId =
    response.headers.get('X-Correlation-Id') || '';
  const body = await response.json();

  if (!response.ok) {
    // O contrato exige correlation_id inclusive em erro: registre-o para investigar o log.
    throw new Error(
      `${body?.detail || 'Falha em /rag/execute'} ` +
      `(correlation_id: ${correlationId || body?.correlation_id || 'ausente'})`
    );
  }

  return {
    answer: body.answer,
    correlationId: body.correlation_id || correlationId,
    raw: body,
  };
}

// Uso:
primeiraPergunta({
  baseUrl: 'http://localhost:5555',
  yamlText: '/* conteúdo do seu YAML de app/yaml/ */',
  question: 'Qual é o procedimento descrito no documento?',
  userEmail: 'developer@empresa.com',
  apiKey: '',                                    // vazio se o YAML já traz authentication.access_key
}).then((r) => {
  console.log('Resposta:', r.answer);
  console.log('correlation_id:', r.correlationId);
});
```

> Este exemplo cobre o **caminho síncrono** (`execution_mode: 'auto'` com resposta direta). Se a resposta vier
> com `HTTP 202` ou com URLs de status no corpo, é execução assíncrona e exige polling — isso está detalhado no
> [TUTORIAL-CHAT-PLATAFORMA.md](TUTORIAL-CHAT-PLATAFORMA.md) e no
> [GUIA-INTEGRADOR-CHAT-PLATAFORMA.md](GUIA-INTEGRADOR-CHAT-PLATAFORMA.md).

> Integração fora do navegador (Python, Node, Ruby)? Os exemplos completos com criptografia FERNET+RSA-OAEP
> estão em [README-EXEMPLOS-INTEGRACAO-API.md](README-EXEMPLOS-INTEGRACAO-API.md) e nos arquivos `examples/`.

## Passo 4: ler o `correlation_id` e validar

Depois da chamada, você tem o `correlation_id` da execução. Ele é a chave para tudo:

- **Validar sucesso**: confira `answer` (a resposta) e, quando existir, `sources`/`source_documents` (as
  evidências usadas). A estrutura completa da resposta está documentada no
  [README-EXEMPLOS-INTEGRACAO-API.md](README-EXEMPLOS-INTEGRACAO-API.md).
- **Investigar problema**: com o `correlation_id` em mãos, abra o log oficial do backend sem varrer a pasta de
  logs às cegas:

  ```bash
  python -m src.log_analyzer query --correlation-id <seu-correlation-id>
  ```

Esse comando reconstrói a história da execução pelo log — é o jeito oficial de descobrir onde algo falhou.

## Erros comuns no primeiro contato

- **"API não está disponível"**: a API não subiu ou está em outra porta. Confira `./run.sh +a`, o `FASTAPI_PORT`
  do `.env` e o health check em `/health`. Cheque também porta presa (`sudo fuser -k 5555/tcp`).
- **HTTP 401 (não autenticado)**: a chave de acesso está ausente ou inválida. Confirme `authentication.access_key`
  no YAML, ou envie uma `X-API-Key` válida. Lembre: basta **uma** das duas fontes.
- **HTTP 422 (validação)**: o corpo da requisição está fora do contrato. Para `qa`, o envelope correto é
  `{ operation: "ask", payload: { ... } }` — não envie os campos soltos no topo.
- **YAML em claro no corpo**: o contrato exige `encrypted_data`. Não tente enviar o YAML bruto como corpo final;
  use o helper de criptografia.
- **`correlation_id` ausente na resposta**: leia tanto o header quanto o corpo. Se realmente faltar, é falha de
  observabilidade do backend — não gere um id local para "resolver".

## Próximo passo

Conseguiu uma resposta? Ótimo. Agora avance para o
[TUTORIAL-CHAT-PLATAFORMA.md](TUTORIAL-CHAT-PLATAFORMA.md), que mostra como montar um chat completo e reutilizável
usando o componente embutível oficial, passando pelos 4 modos (`qa`, `agent`, `deepagent`, `workflow`) e pelos 3
modos de execução, com tratamento de erro e histórico.

## Evidências no código

- `run.sh` — launcher oficial; flags `+a`, `+w`, `+s`.
- `.env` — `FASTAPI_PORT` (default da aplicação: 5555).
- `app/ui/static/js/shared/layout-mestre-api.js` — `LAYOUT_API_CONFIG.endpoints`, headers, `_executarRequisicao`.
- `app/ui/static/js/plataforma-agentes-ia-crypto.js` — `PayloadCrypto.buildEncryptedData`, `injectUserEmailInYaml`.
- `src/api/routers/crypto_router.py` — `POST /crypto/session-key`.
- `src/api/routers/rag_router.py` — `POST /rag/execute`.
- `src/api/security/user_auth.py` — chave aceita do header `X-API-Key` **ou** do YAML `authentication.access_key`.
