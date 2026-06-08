# Manual conceitual: integrar um chat React com o WebChat real

## 1. O que é esta feature

Esta documentação explica como um cliente React pode usar o mesmo caminho do WebChat real da plataforma para conversar com o pipeline RAG, sem reimplementar a lógica de criptografia e sem inventar um segundo protocolo de comunicação.

O ponto importante é que o fluxo não começa em React. Ele começa no backend e no helper browser oficial da plataforma:

1. o navegador pede uma sessão criptográfica efêmera em /crypto/session-key;
2. o helper browser gera um payload `encrypted_data` com Fernet + RSA-OAEP;
3. o React envia esse payload para /rag/execute;
4. o backend resolve o YAML, executa a pergunta e devolve a resposta.

Esse desenho é o mesmo usado pelo **componente global de chat embutível** (`PrometeuEmbeddableChatRuntime`) e pelo cliente HTTP canônico (`layout-mestre-api.js`), que já fazem chamadas reais para o backend.

Antes de escrever um cliente React do zero, vale lembrar: se o seu React puder carregar os scripts compartilhados da plataforma, o caminho recomendado é **embutir o componente** (ver GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md). Este exemplo React do zero é para quem precisa de uma interface 100% própria; o guia completo desse caso é o GUIA-INTEGRADOR-CHAT-PLATAFORMA.md.

## 2. Que problema ela resolve

Sem essa orientação, um desenvolvedor novo tende a tentar chamar o endpoint RAG como se fosse uma API simples de texto livre. Na prática, o backend exige uma estrutura de segurança e de envelope que já está codificada no WebChat real.

O valor desta documentação é mostrar por que esse fluxo existe:

- o YAML não pode trafegar em claro;
- o backend precisa receber `encrypted_data` com sessão efêmera;
- o cliente precisa manter o `correlation_id` e o `execution_mode` no envelope certo;
- o browser precisa usar o mesmo helper oficial que o WebChat já usa.

## 3. Visão executiva

Para liderança e equipe de produto, esta integração é importante porque reduz risco operacional. Em vez de criar uma implementação paralela de criptografia ou um contrato customizado, o frontend reaproveita o mesmo caminho do produto já validado no WebChat.

Isso melhora:

- previsibilidade do runtime;
- segurança do envelope de dados;
- compatibilidade com o backend real;
- velocidade de onboarding para desenvolvedores externos.

## 4. Visão comercial

Para clientes e times de integração, essa capacidade pode ser explicada como “um chat embutido no front-end, mas conectado ao motor RAG oficial da plataforma”.

A promessa comercial correta é esta:

- o cliente ganha uma experiência React nativa;
- o backend continua sendo o mesmo engine de RAG já usado pela plataforma;
- a integração se aproxima do comportamento real do WebChat, sem depender de um fluxo experimental paralelo.

## 5. Visão estratégica

Do ponto de vista arquitetural, essa abordagem mantém o sistema alinhado com o modelo YAML-first e com o runtime oficial do WebChat. Ela evita duplicar lógica de criptografia e evita construir uma camada de front-end que não reflita o comportamento real do produto.

## 6. Conceitos necessários para entender

- `encrypted_data`: envelope criptografado que o backend usa para recuperar YAML e chaves.
- `PayloadCrypto.buildEncryptedData(...)`: helper browser oficial da plataforma para produzir esse envelope.
- `/crypto/session-key`: endpoint que entrega a chave pública efêmera do servidor.
- `/rag/execute`: endpoint oficial que recebe `operation='ask'` e devolve a resposta RAG.
- `correlation_id`: identificador de rastreabilidade retornado pelo backend e propagado no frontend.

## 7. Como a feature funciona por dentro

O fluxo React não precisa reescrever a regra de criptografia. Ele usa o helper que o WebChat real já usa. A parte importante para o desenvolvedor é entender a ordem correta:

1. ler o YAML que será usado como configuração;
2. chamar `PayloadCrypto.buildEncryptedData(...)` no browser;
3. montar o envelope `{ operation: 'ask', payload: { question, user_email, format: 'json', encrypted_data } }`;
4. chamar `/rag/execute` com `credentials: 'include'`;
5. ler a resposta e renderizar `answer`, `sources`, `metrics` e `correlation_id`.

## 8. Fluxo principal

O caminho feliz é simples, mas depende de uma sequência correta:

- o front recebe o YAML do usuário;
- o helper gera `encrypted_data`;
- a chamada HTTP vai para o backend oficial;
- o backend retorna a resposta final ou uma resposta assíncrona com URLs de polling.

## 9. Decisões técnicas e trade-offs

A decisão correta aqui é reutilizar o helper oficial do browser, e não reimplementá-lo em cada tela. Isso reduz risco de divergência entre o WebChat real e um cliente React criado por fora.

O custo é que o desenvolvedor precisa respeitar a estrutura de payload exigida pelo backend. Essa é uma dependência real, não uma convenção opcional.

## 10. Limites e pegadinhas

- o fluxo depende de um backend rodando com o endpoint `/rag/execute` e `/crypto/session-key` disponíveis;
- se o YAML estiver mal formado, o erro real vem do backend, não do React;
- o `X-API-Key` só é necessário quando o ambiente exige autenticação por chave;
- se o browser não carregar `PayloadCrypto`, a integração não pode funcionar como o WebChat oficial.

## 11. Checklist de entendimento

- Entendi que o React precisa usar o helper oficial do browser.
- Entendi que `/rag/execute` exige `encrypted_data` e não apenas um texto simples.
- Entendi que o backend controla a resposta real, inclusão de `correlation_id` e modo de execução.
- Entendi que o exemplo React deve ser mantido pequeno e baseado no fluxo real do WebChat.
