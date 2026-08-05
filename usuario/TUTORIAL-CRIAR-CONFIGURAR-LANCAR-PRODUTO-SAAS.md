# Tutorial: criar, configurar e lançar um produto SaaS

Este tutorial descreve o caminho V1 da plataforma: um produto pertence a um tenant, publica uma
versão imutável de um YAML, oferece um plano simulado e concede ao assinante somente as operações
do plano. Não há provedor de pagamento nesta versão e nenhum cartão é cobrado.

## 1. Escolha o tipo de produto

Antes de abrir a tela administrativa, classifique o produto pelo que o cliente realmente usa:

| Tipo | Exemplo | Operações do plano | Acesso do assinante |
|---|---|---|---|
| Agente | PDV Vendas | `agent` e, se necessário, `rag` | Chat agent na página do produto. |
| Conhecimento | DNIT | `rag` e `ingest` | Chat RAG na página do produto. |
| Canal externo | Casa Moderna | `rag` e `ingest` | WhatsApp configurado para o projeto; a página não abre agente. |

O YAML pode conter agente, RAG, ingestão e ETL ao mesmo tempo. Isso não significa que todas essas
operações serão vendidas. A oferta pública é a interseção entre a versão ativa e as operações dos
planos ativos.

## 2. Prepare o YAML

1. Salve o arquivo em `app/yaml/`.
2. Use somente campos já reconhecidos pelo runtime.
3. Para um produto agentic, configure `selected_entrypoint` com um entrypoint habilitado e existente.
4. Para um produto somente RAG/ingestão, não invente um agente. Casa Moderna e DNIT seguem esse
   modelo comercial.
5. Mantenha `agent-instructions-md` no próprio YAML quando houver agente.
6. Confirme que os placeholders de credenciais correspondem a nomes cadastrados em
   `tenant_security_keys`; valores secretos nunca devem ser gravados no YAML.

O runtime publicado usa o conteúdo materializado em `tenant_yaml.yaml_content`. O arquivo local é a
origem administrativa, mas não é lido pelo runtime da release a cada requisição.

## 3. Cadastre as credenciais do tenant

Abra a administração de chaves do tenant e cadastre somente as integrações exigidas pelo YAML. A
chave de API entregue a um integrador é uma credencial de acesso; os segredos internos, como token
da Meta, DSN, chave de modelo e credencial do vector store, permanecem no diretório seguro.

Para WhatsApp, a configuração operacional precisa incluir as credenciais Meta exigidas pelo canal,
incluindo token, identificador do número e segredo de assinatura do webhook. Para Instagram, é
necessário provisionar a conta Meta e criar um canal Instagram ativo; suporte de código sem canal
provisionado não constitui validação ponta a ponta.

No exemplo Casa Moderna, o YAML habilita Google Drive como fonte, aceita `json`/`application/json`
na pasta configurada e mantém o processador JSON com preservação de estrutura. `local_files` está
desabilitado; portanto, colocar um JSON apenas no disco local não o torna fonte de produção.

## 4. Suba o ambiente completo

Use o ambiente Python do projeto e inicie as três partes juntas:

```bash
source .venv/bin/activate
./run.sh +a +w +s
```

A porta local vem de `FASTAPI_PORT` no `.env`. Não presuma outra porta.

Em `ENVIRONMENT=prod`, o replay AG-UI precisa de persistência PostgreSQL:

```dotenv
AG_UI_EVENT_STORE_PROVIDER=postgres
AG_UI_EVENT_STORE_DSN=${DATABASE_PROMETEU_GENERIC_RAG_DSN}
```

O DDL versionado correspondente é `scripts/sql/20260503_create_ag_ui_schema.sql`. Ele deve ser
aplicado manualmente em janela controlada; o runtime não executa DDL.

## 5. Crie o projeto pelo wizard

1. Abra `/ui/static/ui-admin-saas-projects.html`.
2. Escolha a organização correta.
3. Clique em **Novo projeto**.
4. Informe o nome e o endereço público (`project_key`).
5. Selecione o YAML válido do tenant.
6. Confira as operações detectadas.
7. Publique a versão imutável.
8. Crie o plano gratuito simulado com apenas as operações que serão oferecidas.

Depois da criação, use as abas do projeto:

- **Visão geral**: nome, status e versão ativa;
- **Versões**: publicar uma nova versão ou reativar uma versão publicada;
- **Capacidades**: operações da versão ativa;
- **Planos**: cria plano fora do wizard, edita nome/operações/preço/moeda e transiciona
  ativo/inativo/arquivado — o preço exibido é o real do plano, não mais um valor fixo;
- **Assinantes**: lista paginada (com filtro por status e busca no servidor), direitos e
  cancelamento;
- **Uso**: métricas agregadas por operação e versão (total, sucesso, falha, duração média);
- **Auditoria**: histórico paginado de eventos do projeto, com filtros por origem, operação, versão e
  correlação.

Trocar de YAML significa publicar outra versão. Não altere uma release publicada. As abas Uso e
Auditoria consomem endpoints que já existiam no backend; consulte
`docs/tecnico/README-TECNICO-GESTAO-SAAS-TENANT.md` para o contrato técnico completo dessas rotas e
da gestão de planos.

Antes de publicar/ativar, vale saber quem confere o resultado depois: o Painel do YAML mostra, para
qualquer cliente, se a cópia publicada bate com o arquivo e se a release ativa realmente corresponde
a ela — é o primeiro lugar a olhar se um cliente disser que não está vendo a mudança. Referência
completa de cada tela deste ciclo (publicar, criar/ativar release, religar chave) e do modelo de
chaves de API: [TUTORIAL-101-CICLO-DE-VIDA-YAML-POR-CLIENTE.md](TUTORIAL-101-CICLO-DE-VIDA-YAML-POR-CLIENTE.md).

## 6. Configure canais externos quando aplicável

Um canal precisa apontar para o mesmo tenant e projeto SaaS e declarar a operação executada. Para a
Casa Moderna, o canal ativo é WhatsApp com operação `rag`: a mensagem recebida resolve a release
ativa, executa o RAG e entrega a resposta pelo cliente Meta.

Valide em camadas:

1. configuração e chaves do canal;
2. verificação e assinatura do webhook;
3. parsing do evento recebido;
4. resolução projeto → release → YAML;
5. resposta RAG;
6. entrega Meta para um número ou conta de teste autorizado.

Não use um destinatário inventado para o último passo. Instagram também exige canal e conta reais;
o endpoint seguro de teste de webhook valida parsing, mas não substitui uma conversa externa real.

## 7. Teste a assinatura como cliente

A página pública segue este formato:

```text
/projects/{tenant_id}/{project_key}
```

Na conta de teste local:

1. escolha a demonstração;
2. confirme a assinatura;
3. ative a demonstração;
4. atualize a página e confirme que a assinatura continua administrável;
5. teste a experiência liberada.

O comportamento esperado é:

- projeto agentic: chat agent com transporte AG-UI;
- projeto RAG sem canal externo: chat Q&A usando `projectKey`, sem expor YAML ou API key;
- projeto RAG com canal externo: orientação para usar o canal integrado.

No transporte HTTP do projeto assinado, o header `X-API-Key` deve ser **omitido**, não enviado com
valor vazio ou `undefined`. O backend só dispensa a chave depois de validar sessão, assinatura,
release, operação e entitlement. Integrações sem essa identidade comercial continuam obrigadas a
usar API key.

## 8. Faça o gate de lançamento

Antes de publicar a aplicação:

1. teste a UI real e capture o `correlation_id` de cada execução;
2. consulte o log oficial dessa correlação e confirme ausência de erro;
3. rode a suíte oficial focada nos arquivos alterados;
4. rode lint e tipagem proporcionais ao slice;
5. confirme API, Worker e Scheduler ativos;
6. confirme que a oferta pública não anuncia operação ausente do plano;
7. confirme que a release ativa e os entitlements estão corretos;
8. para canal Meta, faça o último smoke com destinatário de teste autorizado;
9. só então publique a versão da aplicação.

O script `push_producao.sh` exige árvore Git limpa e promove a branch de produção. Primeiro feche o
gate local; depois faça commit, execute a promoção e repita um smoke curto no ambiente publicado.

## 9. Diagnóstico rápido

- **HTTP 500 no AG-UI em prod**: verifique provider, DSN e `ag_ui.run_events`.
- **RAG falha antes de buscar**: abra o log da correlação e valide o contexto canônico em
  `client_context.client`; campos estruturais não devem ser duplicados em `metadata_json`.
- **Produto mostra capacidade indevida**: compare operações da release ativa com operações dos
  planos ativos.
- **Assinatura não libera acesso**: confira status `active`, release ativa, entitlements e chaves do
  tenant.
- **WhatsApp/Instagram não responde**: separe falha de parsing, resolução de projeto, execução RAG e
  entrega Meta; cada etapa precisa de evidência própria.

## 10. Limite do que significa “validado”

Um canal é ponta a ponta somente quando uma conta Meta real e um destinatário autorizado recebem a
resposta. Na rodada local de 2026-07-12, o Worker iniciou o WhatsApp da Casa Moderna, resolveu suas
chaves e o RAG respondeu pelo YAML; não foi enviada mensagem externa porque não havia destinatário de
teste autorizado. Instagram não possuía canal ativo. Esses dois últimos passos devem ser executados
no lançamento com contas de teste reais.

O cenário agentic PDV Vendas abriu o stream AG-UI e persistiu eventos, mas uma consulta analítica
longa permaneceu mais de cinco minutos sem progresso entre eventos do supervisor e foi cancelada.
Use uma pergunta curta no smoke inicial e trate consultas analíticas longas como gate separado de
latência antes de oferecer esse produto ao público.
