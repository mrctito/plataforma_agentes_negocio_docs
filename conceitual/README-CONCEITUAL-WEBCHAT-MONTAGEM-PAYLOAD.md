# Manual conceitual: como o WebChat monta o payload que vai para o endpoint

## 1. O que é esta feature

Esta documentação foca em um ponto real do WebChat da plataforma: a etapa em que o browser transforma o YAML ou o payload já carregado em um envelope pronto para ser enviado ao backend.

A ideia principal é simples:

1. o usuário carrega um YAML ou um payload criptografado;
2. a tela resolve o `encrypted_data` correto;
3. a tela monta o corpo HTTP final;
4. a tela chama o endpoint oficial do RAG, do agente ou do workflow.

Essa etapa é o “momento de montagem do payload”. Sem ela, o frontend pode até parecer que está chamando a API, mas na prática não está respeitando o contrato real do produto.

No produto atual, esse momento de montagem vive em dois lugares reais: o **componente global de chat embutível** (`PrometeuEmbeddableChatRuntime`, em `app/ui/static/js/shared/embeddable-chat-runtime.js`) e o **cliente HTTP canônico** (`layout-mestre-api.js`). O componente decide o que perguntar; o cliente canônico monta o envelope e fala com a API. Quem quer ver a sequência completa (handshake, cifra, envio e polling) deve ler o lado técnico em [README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md](../tecnico/README-TECNICO-WEBCHAT-MONTAGEM-PAYLOAD.md).

## 2. Que problema ela resolve

Sem essa explicação, é fácil confundir três coisas diferentes:

- o YAML que o usuário editou;
- o payload criptografado gerado pelo helper browser;
- o corpo final enviado ao endpoint.

O WebChat real não envia “texto puro” para a API. Ele envia um envelope estruturado com:

- `question`, `task` ou `message`;
- `user_email`;
- `format`;
- `execution_mode`;
- `encrypted_data`.

Esse envelope existe porque o backend precisa recuperar o YAML de forma segura e rastreável.

## 3. Visão executiva

Para produto, operação e integração, essa etapa é importante porque ela mostra como o frontend se conecta ao motor real da plataforma. O valor é direto:

- reduz risco de integração paralela;
- evita reimplementação insegura de criptografia;
- mantém o contrato alinhado com a tela oficial;
- facilita manutenção, suporte e onboarding.

## 4. Visão comercial

Em conversa com clientes e times de integração, esta parte pode ser explicada assim:

> O frontend não inventa um protocolo novo. Ele usa o mesmo envelope que a tela WebChat oficial já usa para chamar o backend RAG/agente/workflow.

Isso ajuda a responder objeções como: “o seu React usa a mesma lógica da plataforma?”

## 5. Visão estratégica

Do ponto de vista da arquitetura, esta etapa garante que o front-end continue acoplado ao runtime real, e não a uma interpretação improvisada. Isso preserva a lógica YAML-first, a segurança da criptografia e a rastreabilidade via `correlation_id`.

## 6. Conceitos necessários para entender

- `encrypted_data`: envelope criptografado que o backend usa para reconstruir o YAML. Seus campos reais são `session_id`, `wrapped_key`, `encrypted_yaml`, `original_filename`, `encryption_scheme` e `yaml_operational_contract`. Atenção: o campo `encrypted_keys`, que aparecia em exemplos antigos, **não é mais gerado** pelo helper browser — as credenciais passaram a ser injetadas pelo backend.
- `PayloadCrypto.buildEncryptedData(...)`: helper browser oficial para produzir esse envelope. Internamente, ele já faz o handshake com `/crypto/session-key`.
- `user_email`: valor operacional injetado no YAML (em `user_session.user_email`) antes da criptografia.
- `execution_mode`: modo de execução selecionado pela tela (`auto`, `direct_sync` ou `direct_async`).
- `metadata_filters`: filtros opcionais de busca usados no RAG.

## 7. Como a montagem acontece na prática

A jornada real da tela é esta:

1. a tela verifica se existe YAML carregado ou payload já criptografado;
2. se houver YAML, ela busca o e-mail do usuário na sessão OAuth;
3. injeta esse e-mail no YAML;
4. chama o helper browser para gerar `encrypted_data`;
5. monta o corpo final conforme o modo de operação;
6. envia para `/rag/execute`, `/agent/execute` ou `/workflow/execute`.

Esse é o núcleo da integração. O ponto mais importante é que o payload final nunca é apenas “uma pergunta”. Ele é um envelope com contexto de execução e configuração criptografada.

## 8. O que muda por modo

### 8.1 Modo RAG

O corpo final costuma ser:

```js
{
  operation: 'ask',
  payload: {
    question: pergunta,
    user_email: userEmail,
    format: 'json',
    execution_mode: 'auto',
    encrypted_data: encryptedData,
    metadata_filters: {...}
  }
}
```

### 8.2 Modo agente

O corpo final costuma ser:

```js
{
  task: pergunta,
  user_email: userEmail,
  format: 'json',
  execution_mode: 'auto',
  encrypted_data: encryptedData,
  mode: 'deepagent'
}
```

### 8.3 Modo workflow

O corpo final costuma ser:

```js
{
  message: pergunta,
  user_email: userEmail,
  thread_id: null,
  format: 'json',
  execution_mode: 'auto',
  encrypted_data: encryptedData
}
```

## 9. O detalhe que faz a diferença para React/JavaScript

Se você estiver implementando isso em React, o caminho mais seguro é:

1. usar o mesmo helper browser oficial (`PayloadCrypto`);
2. não tentar reescrever a criptografia do zero;
3. manter o `user_email` no mesmo ponto onde a tela já o conhece;
4. enviar `credentials: 'include'` **e** os headers `X-API-Key` (ou a chave em `authentication.access_key` no YAML) e `X-Prometeu-UI-Session-Required: 1`, como faz o cliente canônico;
5. tratar o `encrypted_data` como objeto, não como string livre.

## 10. Limites e pegadinhas

- o payload final não substitui a autenticação do backend;
- o `encrypted_data` não é opcional no caminho real da tela;
- se o YAML vier com `access_key`, a tela pode reutilizar esse valor como fallback, mas o payload criptografado ainda não carrega essa chave em aberto;
- o `correlation_id` não é gerado no browser; ele vem do backend.

## 11. Checklist de entendimento

- Entendi que o corpo final é um envelope, não apenas um texto.
- Entendi onde o `encrypted_data` entra na montagem.
- Entendi que o e-mail do usuário é injetado antes da criptografia.
- Entendi por que o backend exige esse formato.
- Entendi como adaptar esse fluxo para React ou JavaScript sem inventar protocolo paralelo.
