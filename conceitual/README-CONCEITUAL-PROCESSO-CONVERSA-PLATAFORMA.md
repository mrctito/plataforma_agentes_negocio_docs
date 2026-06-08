# Manual conceitual, executivo e de onboarding: API usada pela tela de conversa da plataforma

## 1. O que é esta feature

Este documento explica como a página [app/ui/static/ui-webchat-v3.html](../app/ui/static/ui-webchat-v3.html) conversa com o backend para enviar uma pergunta e receber uma resposta.

O ponto mais importante é este: a tela não manda só um texto livre para a API. Ela monta um envelope operacional que combina:

- a intenção da conversa;
- o modo de runtime escolhido;
- o contexto do usuário;
- um bloco criptografado chamado `encrypted_data`;
- e, dependendo do caso, filtros ou dados de continuidade.

Em linguagem 101: a pergunta digitada é só uma parte da história. O backend precisa de muito mais contexto para conseguir responder de forma real.

## 2. Que problema esta feature resolve

Se um desenvolvedor olhar a tela de conversa por fora, ele pode achar que o sistema funciona assim:

1. o usuário digita uma pergunta;
2. o frontend faz `POST` com esse texto;
3. o backend responde.

Isso está incompleto.

No projeto real, a conversa depende de um YAML ou payload criptografado que descreve o runtime. Sem isso, o backend não sabe qual configuração agentic, qual vetor, qual recuperação e qual política usar.

Então esta feature resolve um problema estrutural: transformar uma pergunta solta em uma requisição completa, segura, auditável e compatível com o runtime oficial da plataforma.

## 3. Visão executiva

Para liderança, o valor não está em “uma tela de chat”. O valor está em ter uma borda de conversa governada e reaproveitável.

Na prática, isso traz:

- menos risco de integrações paralelas erradas;
- mais previsibilidade do comportamento da conversa;
- rastreabilidade com `correlation_id`;
- possibilidade de criar novas interfaces sem reinventar o protocolo.

## 4. Visão comercial

Comercialmente, esta capacidade pode ser explicada assim:

> A plataforma permite criar frontends próprios de conversa usando o mesmo backend oficial do WebChat real, sem precisar duplicar a lógica de segurança, criptografia e observabilidade.

Isso ajuda muito em cenários de:

- portais do cliente;
- aplicações React customizadas;
- experiências embutidas em sistemas existentes;
- provas de conceito que precisam conversar com o runtime real.

## 5. Visão estratégica

Estratégicamente, esta feature reforça quatro pilares do produto:

- frontend desacoplado da regra de negócio;
- YAML-first com transporte seguro;
- fronteiras HTTP oficiais por modo de execução;
- observabilidade com `correlation_id` e execução assíncrona quando necessário.

## 6. Conceitos necessários para entender

### 6.1 Pergunta

É o texto digitado pelo usuário. Sozinha, ela não basta.

### 6.2 YAML ou payload carregado

É o contexto operacional que define como o runtime deve responder.

### 6.3 `encrypted_data`

É o envelope criptografado que leva esse contexto para o backend sem trafegar YAML em claro.

### 6.4 `execution_mode`

É o modo de execução pedido pela tela, como `auto`, `direct_sync` ou `direct_async`, dependendo do endpoint.

### 6.5 `correlation_id`

É o identificador oficial da execução devolvido pelo backend. A tela exibe, guarda e propaga esse valor. Ela não cria esse identificador.

### 6.6 Resposta assíncrona

Quando a API responde com `202`, ela está dizendo: “recebi a solicitação, mas a resposta final vai chegar depois”. Nesse caso, o frontend precisa acompanhar `polling_url` ou `stream_url`.

## 7. Qual API a tela usa de fato

A tela [app/ui/static/ui-webchat-v3.html](../app/ui/static/ui-webchat-v3.html) suporta mais de um modo de conversa. Isso significa que ela não usa um único endpoint em todos os casos.

### 7.1 Quando o modo é Q&A / RAG

Ela usa:

- `POST /rag/execute`

Esse é o caso clássico de “enviar pergunta e receber resposta”.

### 7.2 Quando o modo é Agent ou DeepAgent

Ela usa:

- `POST /agent/execute`

Se o modo selecionado for `deepagent`, o payload inclui `mode: 'deepagent'`.

### 7.3 Quando o modo é Workflow

Ela usa:

- `POST /workflow/execute`

### 7.4 Endpoints de apoio que também participam da jornada

Além do endpoint principal de conversa, a jornada real da UI usa também:

- `GET /api/auth/federated/session` para resolver o usuário autenticado;
- `POST /crypto/session-key` de forma indireta, via `PayloadCrypto.buildEncryptedData(...)`, para gerar a sessão criptográfica efêmera;
- `GET /api/v1/status/{task_id}` quando a resposta vem em modo assíncrono.

## 8. Como a tela sabe qual payload montar

Essa é uma das partes mais importantes do guia.

A tela não monta o payload por chute. Ela decide o formato a partir de algumas fontes reais de verdade.

### 8.1 Fonte 1: modo de operação

O modo pode vir de duas origens:

- seleção manual do usuário na própria tela;
- inferência a partir do YAML carregado.

Quando há YAML, a tela usa o resolvedor `extractPreferredModeFromYaml(...)` para tentar identificar automaticamente se aquele documento representa:

- `deepagent`;
- `workflow`;
- ou outro modo explícito.

Em linguagem simples: se o YAML tem cara de DeepAgent, a tela tenta se comportar como DeepAgent. Se tem cara de Workflow, ela muda para Workflow.

### 8.2 Fonte 2: origem do contexto seguro

A tela verifica se já existe um `payloadConteudo` carregado. Se existir, ela usa esse payload já criptografado.

Se não existir, ela procura `yamlConteudo` e então gera `encrypted_data` no browser.

### 8.3 Fonte 3: e-mail do usuário

Antes de criptografar o YAML, a tela consulta a sessão federada e tenta injetar o e-mail operacional em `user_session.user_email`.

Isso importa porque o backend usa esse dado para auditoria e rastreamento.

### 8.4 Fonte 4: API Key

A tela resolve a API key com esta prioridade:

1. campo preenchido na própria interface;
2. fallback extraído do YAML, quando disponível;
3. se só houver payload criptografado e o campo estiver vazio, a tela não consegue recuperar a key do payload, então a requisição pode falhar por autenticação.

### 8.5 Fonte 5: filtros de metadados

No modo RAG, se houver filtros ativos na UI, eles são convertidos para `metadata_filters` e enviados junto com a pergunta.

## 9. Jornada do programador estagiário para construir sua própria interface

Esta seção foi escrita de propósito em linguagem de onboarding.

### Passo 1: entenda que você não está construindo “um chat genérico”

Você está construindo um cliente para o runtime oficial da plataforma. Isso muda tudo.

Seu trabalho não é inventar um protocolo. Seu trabalho é respeitar o protocolo que a UI real já usa.

### Passo 2: escolha o recorte inicial mais simples

Se você está começando, implemente primeiro só o modo RAG.

Motivo prático: ele é o fluxo mais simples para aprender:

- pergunta;
- `encrypted_data`;
- `POST /rag/execute`;
- resposta ou polling.

### Passo 3: garanta que o backend está acessível

Você precisa conseguir acessar:

- a página de UI real para comparar comportamento;
- `POST /rag/execute`;
- `GET /api/auth/federated/session` se quiser reaproveitar a sessão oficial;
- a jornada de criptografia client-side via `PayloadCrypto`.

### Passo 4: carregue o YAML ou payload seguro

Você precisa de uma destas duas entradas:

- um YAML carregado pelo usuário;
- ou um payload criptografado já pronto.

Sem um dos dois, a tela não consegue montar a requisição oficial.

### Passo 5: resolva o e-mail operacional

Se você quiser espelhar a tela oficial, use a sessão federada para descobrir o e-mail do usuário autenticado.

Se isso não estiver disponível no seu ambiente, pelo menos peça o e-mail explicitamente no formulário.

### Passo 6: gere `encrypted_data`

Se você tiver YAML, use `PayloadCrypto.buildEncryptedData(...)`.

Esse passo é obrigatório no caminho oficial.

### Passo 7: monte o payload certo para o modo certo

Não misture os formatos.

- RAG usa `operation + payload.question`;
- Agent usa `task`;
- Workflow usa `message`.

### Passo 8: envie a requisição com sessão e headers corretos

Na prática, isso significa:

- `Content-Type: application/json`;
- `credentials: 'include'` quando a sessão precisa ir junto;
- `X-API-Key` quando o ambiente exigir autenticação por chave.

### Passo 9: trate `200` e `202` como casos normais

Se vier `202`, não é erro. É execução assíncrona.

### Passo 10: leia e preserve o `correlation_id`

Esse valor é a sua referência oficial de suporte, log e diagnóstico.

### Passo 11: só depois melhore a UX

Primeiro faça funcionar com o contrato real. Depois pense em layout, histórico bonito, streaming refinado ou badges visuais.

## 10. Como o payload é montado

### 10.1 Payload no modo RAG

No modo Q&A, a UI monta um envelope com dois níveis:

1. envelope externo com `operation: 'ask'`;
2. envelope interno em `payload`.

O resultado lógico é:

```json
{
  "operation": "ask",
  "payload": {
     "question": "Qual é o SLA do plano premium?",
     "user_email": "analista@cliente.com",
     "format": "json",
     "execution_mode": "auto",
     "encrypted_data": {
        "session_id": "...",
        "wrapped_key": "...",
        "encrypted_yaml": "...",
        "original_filename": "config.yaml",
        "encryption_scheme": "FERNET+RSA-OAEP"
     }
  }
}
```

### 10.2 Payload no modo Agent / DeepAgent

Nesse caso, a intenção não fica em `payload.question`. Ela vira `task` na raiz do corpo:

```json
{
  "task": "Analise este cenário e proponha um plano",
  "user_email": "analista@cliente.com",
  "format": "json",
  "execution_mode": "auto",
  "encrypted_data": {"...": "..."},
  "mode": "deepagent"
}
```

### 10.3 Payload no modo Workflow

No workflow, a mensagem usa outro nome e ainda pode levar `thread_id`:

```json
{
  "message": "Quero continuar o processo da thread atual",
  "user_email": "analista@cliente.com",
  "thread_id": null,
  "format": "json",
  "execution_mode": "auto",
  "encrypted_data": {"...": "..."}
}
```

## 11. O que a resposta devolve para a tela

No caso síncrono, o backend costuma devolver o conteúdo final da resposta e o `correlation_id`.

No caso assíncrono, a tela espera algo como:

- `task_id`;
- `polling_url`;
- `stream_url`;
- `status_url` como alias legado em alguns fluxos;
- `correlation_id`.

## 12. FAQ com 15 perguntas e respostas

1. A tela usa sempre `/rag/execute`?
    Não. Ela usa `/rag/execute`, `/agent/execute` ou `/workflow/execute` conforme o modo.

2. Então por que as pessoas falam tanto de `/rag/execute`?
    Porque esse é o caminho padrão da pergunta e resposta no modo Q&A, que é o caso mais comum de chat.

3. A pergunta sozinha já basta para a API responder?
    Não. O backend também precisa do contexto transportado em `encrypted_data`.

4. O frontend envia YAML puro para o backend?
    Não no caminho oficial. O YAML é criptografado no browser antes do envio.

5. Quem decide o endpoint?
    A lógica de modo da UI, baseada na seleção do usuário e/ou no YAML carregado.

6. Como a tela sabe que um YAML é DeepAgent ou Workflow?
    Ela usa o resolvedor de modo preferencial que procura sinais como `selected_supervisor`, `multi_agents`, `selected_workflow` e `workflows`.

7. O `correlation_id` nasce no frontend?
    Não. Ele nasce no backend e a UI apenas o captura e preserva.

8. Para que serve `credentials: 'include'`?
    Para enviar a sessão do navegador quando a autenticação depende de cookie/sessão federada.

9. O `X-API-Key` sempre é obrigatório?
    Não necessariamente. Depende do ambiente e da política da borda. Mas a UI sabe enviá-lo quando estiver disponível.

10. Por que a API key não vem de volta do payload criptografado?
    Porque ela é usada no processo de criptografia e não fica armazenada ali em aberto.

11. Quando eu devo tratar `202` como erro?
    Nunca por reflexo. Primeiro trate como execução assíncrona. Só é erro se o acompanhamento posterior falhar.

12. Se eu quiser criar uma interface própria, preciso copiar a UI inteira?
    Não. Você precisa respeitar o contrato HTTP e a jornada de criptografia, não copiar o layout.

13. O jeito certo de começar é por Agent ou Workflow?
    Não. O jeito mais seguro para onboarding é começar por RAG.

14. O payload muda quando há filtros?
    Sim. No modo RAG, filtros ativos viram `metadata_filters` dentro de `payload`.

15. Qual é o erro conceitual mais comum?
    Tratar esse chat como se fosse uma API de texto livre qualquer. Ele é um runtime configurado por YAML e protegido por envelope criptografado.

## 13. Explicação 101

Imagine que a pergunta do usuário é uma carta, mas o backend não responde olhando só a carta. Ele também precisa do dossiê que diz quem é o usuário, qual configuração de runtime está ativa e como aquela conversa deve ser interpretada. O `encrypted_data` é esse dossiê fechado. O endpoint é o balcão certo para o tipo de pedido. O `correlation_id` é o protocolo do atendimento.

## 14. Limites e pegadinhas

- não use um endpoint único para todos os modos sem respeitar o formato correto;
- não tente montar `encrypted_data` manualmente;
- não invente `correlation_id` no navegador;
- não trate `202` como falha automática;
- não esqueça que payload criptografado não guarda a API key em aberto.

## 15. Evidências no código

- app/ui/static/ui-webchat-v3.html
  - Motivo da leitura: confirmar a superfície real usada pelo usuário.
  - Comportamento confirmado: a página documenta os modos RAG, Agent, DeepAgent e Workflow.

- app/ui/static/js/ui-webchat-v3.js
  - Motivo da leitura: confirmar montagem de payload, escolha de endpoint, API key, filtros e tratamento da resposta.
  - Comportamento confirmado: a UI monta o corpo de forma diferente por modo e usa `PayloadCrypto` quando o contexto vem de YAML.

- app/ui/static/js/shared/yaml-access-key-extractor.js
  - Motivo da leitura: confirmar inferência de modo preferencial a partir do YAML.
  - Comportamento confirmado: a UI consegue inferir `deepagent` e `workflow` do documento carregado.

- app/ui/static/js/plataforma-agentes-ia-crypto.js
  - Motivo da leitura: confirmar a geração de `encrypted_data`.
  - Comportamento confirmado: a criptografia client-side passa por sessão efêmera e envelope oficial.

- src/api/routers/rag_router.py
  - Motivo da leitura: confirmar o contrato do dispatcher `/rag/execute`.
  - Comportamento confirmado: o modo Q&A passa por envelope com `operation` e `payload`.

- src/api/routers/agent_router.py e src/api/routers/workflow_router.py
  - Motivo da leitura: confirmar contratos mínimos de `task` e `message` para os outros modos de conversa.
  - Comportamento confirmado: Agent e Workflow têm contratos públicos diferentes do RAG.
