# Tutorial 101: preparar credenciais Shopify e Instagram para o AIDAN

Este tutorial explica o que uma pessoa precisa fazer manualmente na Shopify,
na Meta e na plataforma para conectar a loja e o Instagram de um clube ao
AIDAN. Ele foi escrito para o cenário em que:

- o clube é dono da loja Shopify, da Página do Facebook e da conta profissional
  do Instagram;
- a empresa que opera o AIDAN é dona da aplicação de integração na Meta;
- o clube autoriza essa empresa a acessar somente os ativos e dados necessários;
- as integrações serão usadas no servidor e pelos agentes, nunca diretamente no
  navegador do torcedor.

> **Regra mais importante:** email, usuário e senha servem para uma pessoa entrar
> nos painéis. As APIs usam `Client ID`, `Client Secret`, tokens, IDs e permissões.
> Nunca informe a senha da loja ou do Instagram em um wizard da plataforma.

## 1. O que estará pronto ao final

Você terá reunido estes dados:

| Integração | Dado | É segredo? | Origem |
| --- | --- | --- | --- |
| Shopify Admin | domínio `*.myshopify.com` | não | Shopify Admin |
| Shopify Admin | Client ID | identificador, mas deve ser protegido | Shopify Dev Dashboard |
| Shopify Admin | Client Secret | sim | Shopify Dev Dashboard |
| Shopify Admin | token de acesso da loja | sim | OAuth após o clube instalar a aplicação |
| Shopify Storefront | token privado do Storefront | sim | canal Headless da loja |
| Meta | App ID | identificador, mas deve ser protegido | Meta for Developers |
| Meta | App Secret | sim | Meta for Developers |
| Meta | Facebook Page ID | não | Graph API |
| Meta | Instagram Business Account ID | não | Graph API |
| Meta | Page Access Token | sim | Graph API/OAuth |
| Meta | URL pública do webhook | não | infraestrutura da plataforma |
| Meta | Verify Token do webhook | sim | gerado pela empresa operadora |
| Plataforma | X-API-Key com `provision.instagram` | sim | administração da plataforma |

Guarde os segredos em um cofre de segredos. Um `.env` local ignorado pelo Git
pode ser usado apenas durante o desenvolvimento. Não grave tokens em YAML,
prints, tickets, planilhas, mensagens ou documentação.

## 2. Quem faz o quê

| Pessoa | Ação manual |
| --- | --- |
| Responsável do clube | autoriza a aplicação Shopify, vincula Instagram e Página e concede acesso aos ativos Meta |
| Administrador da empresa operadora | configura as aplicações Shopify e Meta, URLs, permissões e revisão |
| Operador da plataforma | cadastra as credenciais pelos fluxos autorizados e executa os testes de conexão |
| Plataforma | troca códigos OAuth por tokens, verifica webhooks, armazena segredos e chama as APIs |

O clube não deve entregar sua senha para a operação cotidiana. O fluxo correto é
o responsável do clube entrar no próprio painel e aprovar a aplicação e as
permissões apresentadas.

## 3. Shopify: preparar a integração da loja

### 3.1 Escolher o fluxo correto

Para uma loja pertencente a um cliente, use uma aplicação da empresa operadora
com **distribuição personalizada** e autorização OAuth. A Shopify informa que
novas aplicações personalizadas já não são criadas dentro do Shopify Admin; o
caminho atual é Dev Dashboard ou Shopify CLI. A distribuição personalizada
permite gerar um link de instalação para uma loja específica. Consulte
[Create apps using the Dev Dashboard](https://shopify.dev/docs/apps/build/dev-dashboard/create-apps-using-dev-dashboard),
[Select a distribution method](https://shopify.dev/docs/apps/launch/distribution/select-distribution-method)
e [About token acquisition](https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens).

Não use o fluxo `client_credentials` para uma loja de outro comerciante. A
Shopify restringe esse fluxo às aplicações e lojas pertencentes à mesma
organização. Para lojas de clientes, a autorização deve ocorrer por OAuth.

### 3.2 Na conta da empresa: criar ou abrir a aplicação Shopify

1. Entre no [Shopify Dev Dashboard](https://dev.shopify.com/dashboard) com a
   conta da empresa operadora.
2. Abra **Apps**.
3. Se a aplicação AIDAN já existir, abra-a. Se não existir, selecione
   **Create app** e depois **Start from Dev Dashboard**.
4. Dê um nome inequívoco, por exemplo `AIDAN - Integração de Clubes`.
5. Abra **Versions** e crie uma versão.
6. Informe a URL HTTPS da aplicação e as URLs de redirecionamento OAuth. Essas
   URLs precisam apontar para endpoints reais da plataforma; não use uma URL
   fictícia apenas para liberar a tela.
7. Selecione uma versão estável da API e dos webhooks suportada no momento da
   configuração.
8. Configure somente os escopos necessários.
9. Publique a versão com **Release**.

### 3.3 Escopos Admin recomendados para o AIDAN

Comece com leitura mínima:

- `read_products`: produtos, variantes e preços;
- `read_inventory`: saldos de estoque;
- `read_locations`: locais aos quais o estoque pertence;
- `read_orders`: pedidos recentes necessários ao atendimento.

Adicione `read_customers` somente quando o vínculo entre cliente Shopify e
torcedor estiver aprovado, documentado e protegido. Dados de clientes são dados
pessoais protegidos. O acesso a pedidos antigos pode exigir `read_all_orders`,
que possui aprovação separada. Não solicite `write_products`, `write_inventory`,
`write_customers` ou `write_orders` apenas por conveniência. A referência oficial
é [Shopify API access scopes](https://shopify.dev/docs/api/usage/access-scopes).

Carrinho e checkout não exigem que o agente crie um pedido pela Admin API. O
desenho correto é usar a Storefront Cart API e entregar ao torcedor o
`checkoutUrl` oficial da Shopify.

### 3.4 Gerar o link e o clube instalar a aplicação

1. Na área de distribuição da aplicação, selecione **Custom distribution**.
2. Informe exatamente o domínio permanente da loja, no formato
   `<loja>.myshopify.com`.
3. Gere o link de instalação.
4. Envie o link somente ao responsável autorizado do clube.
5. O responsável abre o link, entra na Shopify, confere os escopos e seleciona
   **Install**.
6. A Shopify redirecionará o navegador para a URL OAuth da plataforma com um
   código temporário.
7. A plataforma deverá validar a assinatura e o `state`, trocar o código por um
   token offline e armazená-lo no cofre do tenant.

O passo 7 é automático e ainda depende do boundary Shopify previsto para o
AIDAN. Não existe um token permanente para o operador copiar manualmente nesse
fluxo. Não tente usar `Client ID` ou `Client Secret` como se fossem o token da
Admin API.

### 3.5 Copiar Client ID e Client Secret

1. No Dev Dashboard, abra a aplicação.
2. Entre em **Settings**.
3. Copie o **Client ID**.
4. Revele e copie o **Client Secret** somente para o cofre da plataforma.
5. Registre qual aplicação, organização e loja correspondem ao segredo.

O segredo nunca deve aparecer no frontend. A Shopify orienta obter essas
credenciais na página Settings da aplicação e rotacioná-las quando houver
suspeita de exposição: [About client credentials](https://shopify.dev/docs/apps/build/authentication-authorization/client-secrets).

### 3.6 Preparar Storefront, carrinho e checkout

Este é um segundo acesso, separado da Admin API:

1. O responsável do clube entra no Shopify Admin.
2. Instala o canal oficial **Headless** pela Shopify App Store.
3. Abre **Sales channels > Headless**.
4. Seleciona **Create storefront**.
5. Dá um nome claro, como `AIDAN - Carrinho Conversacional`.
6. Abre **Storefront API permissions > Edit**.
7. Libera somente produtos/coleções, disponibilidade, carrinho e checkout. Só
   libere clientes ou metafields se o caso de uso realmente exigir.
8. Salva as permissões.
9. Copia o **private access token** para o cofre do tenant.

A Shopify documenta esse fluxo em
[Getting started with the Storefront API](https://shopify.dev/docs/storefronts/headless/building-with-the-storefront-api/getting-started).
O token privado fica somente no servidor. A integração deverá usar a Cart API,
como `cartCreate`, e o `checkoutUrl`, em vez da antiga Checkout API. Consulte a
[Storefront API](https://shopify.dev/docs/api/storefront/latest) e
[`cartCreate`](https://shopify.dev/docs/api/storefront/latest/mutations/cartCreate).

### 3.7 O que anotar para o futuro wizard Shopify

Sem colocar valores reais neste documento, prepare uma ficha com:

```text
tenant: aidan
shop_domain: <loja>.myshopify.com
shopify_app_client_id: <no cofre>
shopify_app_client_secret: <no cofre>
admin_oauth_status: aguardando | autorizado
storefront_private_token: <no cofre>
admin_api_version: <versão suportada e fixada>
storefront_api_version: <versão suportada e fixada>
granted_scopes: <lista efetivamente aprovada>
```

## 4. Meta e Instagram: preparar a conta e a aplicação

O wizard atual usa o modelo **Instagram API with Facebook Login**: a conta
profissional do Instagram precisa estar vinculada a uma Página do Facebook e a
plataforma recebe um Page Access Token. Não misture esse modelo com o fluxo mais
novo **Instagram Login**, que usa outros tokens e nomes de permissão.

### 4.1 No clube: conferir Instagram e Página

1. Entre no Instagram do clube pelo aplicativo oficial.
2. Confirme em **Configurações e privacidade** que a conta é **Business** ou
   **Creator**. Conta pessoal não serve para este wizard.
3. No Meta Business Suite, confirme que existe uma Página do Facebook do clube.
4. Vincule a conta profissional do Instagram a essa Página.
5. Ative autenticação de dois fatores nas contas administrativas.
6. Teste se o responsável consegue abrir a Página, o Instagram, comentários e
   caixa de mensagens pela Business Suite.

Se a Página não estiver vinculada, a Graph API não devolverá o
`instagram_business_account` exigido pela tela.

### 4.2 No Meta Business Settings: conceder acesso à empresa

O responsável do clube executa esta parte:

1. Abra **Business Settings** do portfólio empresarial do clube.
2. Em **Accounts > Pages**, confira que a Página está no portfólio.
3. Em **Accounts > Instagram accounts**, confira que o Instagram está no mesmo
   portfólio.
4. Em **Partners**, adicione o Business Portfolio ID da empresa operadora.
5. Compartilhe somente a Página e a conta do Instagram desse clube.
6. Conceda as tarefas necessárias para conteúdo, mensagens, comentários e
   análise. Não conceda finanças, anúncios ou outros ativos sem necessidade.
7. Confirme que a pessoa que fará a autorização também está atribuída aos dois
   ativos.

### 4.3 Na aplicação Meta da empresa

1. Entre em [Meta for Developers](https://developers.facebook.com/apps/).
2. Abra a aplicação de integração da empresa.
3. Em **App settings > Basic**, copie o **App ID**.
4. Revele o **App Secret** somente para gravá-lo no cofre.
5. Confira email de contato, domínio da aplicação, política de privacidade e
   instrução ou URL de exclusão de dados.
6. Adicione/configure os produtos ou casos de uso de **Facebook Login for
   Business**, **Instagram API** e **Webhooks** aplicáveis à aplicação.
7. Cadastre a URL de redirecionamento OAuth exata da plataforma.
8. Não troque a aplicação para **Live** antes de concluir testes, verificação da
   empresa e App Review das permissões avançadas.

### 4.4 Permissões para o fluxo atual

Para descobrir a Página e a conta e operar o conjunto atual, prepare a revisão
das permissões efetivamente usadas:

- `pages_show_list`;
- `pages_read_engagement`;
- `pages_manage_metadata`;
- `instagram_basic`;
- `instagram_manage_comments`;
- `instagram_manage_messages`.

Para o escopo completo do projeto AIDAN, a aplicação também poderá precisar de
`instagram_content_publish` e `instagram_manage_insights`. Elas não tornam as
respectivas telas automaticamente funcionais; cada capacidade ainda precisa de
wiring, teste e aprovação próprios.

No modo Development, apenas administradores, desenvolvedores, testadores e
ativos atribuídos à aplicação conseguem testar. Para autorizar contas reais de
clientes fora desses papéis, conclua **Business Verification**, solicite
**Advanced Access** em **App Review > Permissions and Features**, entregue as
evidências pedidas pela Meta e só então publique a aplicação em Live.

### 4.5 Gerar um token de teste sem confundir com produção

1. Abra o [Graph API Explorer](https://developers.facebook.com/tools/explorer/).
2. Selecione a aplicação correta da empresa.
3. Selecione **User Token**.
4. Adicione somente as permissões listadas na seção anterior.
5. Entre com a pessoa que possui acesso aos ativos do clube e aprove a tela de
   consentimento.
6. Use esse token apenas para validar o vínculo e descobrir IDs.

O token produzido pelo Explorer pode ser temporário. Não o trate como credencial
de produção. O fluxo de produção deve usar OAuth, validar `state` e callback e
guardar o token resultante no cofre. Confira tipo, aplicação, escopos e validade
no [Access Token Debugger](https://developers.facebook.com/tools/debug/accesstoken/).

### 4.6 Descobrir Page ID, Page Access Token e Instagram Business Account ID

No Graph API Explorer, execute:

```http
GET /me/accounts?fields=name,access_token,tasks,instagram_business_account
```

Na resposta, localize a Página do clube e copie para o cofre ou ficha de
provisionamento:

```json
{
  "id": "<FACEBOOK_PAGE_ID>",
  "access_token": "<PAGE_ACCESS_TOKEN>",
  "instagram_business_account": {
    "id": "<INSTAGRAM_BUSINESS_ACCOUNT_ID>"
  }
}
```

Esse é o fluxo publicado na coleção oficial da Meta:
[Instagram API — Get Access Tokens of Pages You Manage](https://www.postman.com/meta/instagram/documentation/6yqw8pt/instagram-api).

Se `instagram_business_account` não aparecer, pare e corrija o vínculo entre
Página e Instagram ou a atribuição de ativos. Não invente um ID a partir do
nome de usuário.

### 4.7 Preparar o webhook

1. Defina uma URL HTTPS pública e estável para o webhook da plataforma.
2. Confirme que a URL implementa o `GET` de verificação da Meta e responde com o
   `hub.challenge` quando o Verify Token coincide.
3. Confirme que a mesma URL recebe os eventos `POST`, valida a assinatura e
   responde rapidamente com sucesso antes de processar assincronamente.
4. Gere o **Verify Token** em um gerenciador de senhas. Ele não é a senha do
   Instagram nem o App Secret.
5. Grave o mesmo Verify Token no cofre da plataforma e na configuração do
   webhook.
6. Assine apenas os campos necessários, como comentários, menções e mensagens.
7. Teste um comentário e uma DM reais de uma conta autorizada.

A referência oficial é
[Graph API Webhooks](https://developers.facebook.com/docs/graph-api/webhooks/getting-started).

## 5. Preencher o wizard Instagram da plataforma

A tela atual fica em:

```text
/ui/static/ui-admin-gov-provisionamento-instagram.html
```

Antes de abri-la, coloque no bloco de identidade uma X-API-Key do tenant AIDAN
com a permissão `provision.instagram`. Depois preencha:

| Campo da tela | Valor correto |
| --- | --- |
| Instagram Business Account ID | `instagram_business_account.id` devolvido por `/me/accounts` |
| Facebook Page ID | `id` da Página devolvido por `/me/accounts` |
| Nome Comercial | nome do clube exibido ao operador |
| Nome da Página | nome da Página; opcional |
| Email Comercial | email operacional do clube |
| Channel ID | identificador lógico único; opcional |
| Page Access Token | `access_token` da Página, nunca a senha do Instagram |
| App ID | App ID da aplicação Meta da empresa |
| App Secret | App Secret da mesma aplicação |
| Callback URL | URL pública real que implementa verificação e eventos |
| Verify Token | segredo gerado para esse webhook |
| Graph API Version | versão ainda suportada pela Meta e validada pela plataforma |

Ao selecionar **Provisionar conta**, o backend atual tenta, nesta ordem:

1. consultar o perfil profissional pelo Instagram Business Account ID;
2. assinar a Página para eventos de mensagens;
3. cadastrar a assinatura de webhook da aplicação;
4. registrar a conta no diretório da plataforma.

Guarde o `correlation_id` exibido pela tela. Ele é a referência para investigar
qual etapa falhou sem expor os segredos.

## 6. Limites atuais que impedem declarar produção pronta

As credenciais podem ser preparadas agora, mas os seguintes pontos ainda
precisam ser implementados ou corrigidos antes do uso produtivo:

| Parte | Situação atual | Consequência |
| --- | --- | --- |
| Shopify AIDAN | o boundary Admin/Storefront e o wizard AIDAN ainda não existem | é possível criar aplicação, autorização e token Storefront, mas ainda não cadastrar e provar a conexão pela UI AIDAN |
| OAuth Shopify | não existe callback AIDAN que conclua a autorização da loja do cliente | o token Admin de produção ainda não pode ser adquirido e armazenado pelo fluxo correto |
| Webhook Instagram | o repositório possui teste de payload, mas não um receiver público completo de verificação e eventos | não há Callback URL interna pronta para informar à Meta |
| Segredos Instagram | o provisionador atual guarda hashes, não credenciais recuperáveis pelo runtime | o registro da conta não torna o token disponível aos agentes depois |
| Versão Meta | a tela usa `v20.0` como valor inicial | o operador não deve aceitar esse valor sem confirmar que ainda é suportado; a plataforma precisa fixar uma versão vigente |
| Validação de escopos | o provisionador prova acesso pelas chamadas, mas não lista e compara formalmente todos os escopos concedidos | “sucesso” não é uma auditoria completa das permissões |

Portanto:

- **não clique no wizard Instagram em produção** até existir Callback URL real,
  persistência segura dos segredos e versão Graph suportada;
- **não procure um wizard Shopify AIDAN hoje**, porque essa tela ainda está
  explicitamente indisponível;
- conclua agora os passos manuais de propriedade, acesso, aplicação, escopos,
  revisão e geração do Storefront token;
- depois da implementação dos boundaries, use os wizards para OAuth, teste de
  saúde e ativação, sem copiar segredo para YAML.

## 7. Checklist de liberação

### Shopify

- [ ] aplicação criada na organização da empresa;
- [ ] versão publicada com URLs HTTPS reais;
- [ ] distribuição personalizada limitada à loja do clube;
- [ ] escopos mínimos revisados pelo clube;
- [ ] link de instalação aprovado pelo clube;
- [ ] canal Headless e Storefront privado criados;
- [ ] Client Secret e Storefront token no cofre;
- [ ] callback OAuth e armazenamento de token Admin implementados;
- [ ] consulta real de produto, estoque e pedido aprovada;
- [ ] carrinho real retorna `checkoutUrl` da Shopify.

### Meta/Instagram

- [ ] Instagram profissional vinculado à Página;
- [ ] Página e Instagram atribuídos à empresa/pessoa autorizada;
- [ ] App ID e App Secret confirmados;
- [ ] permissões mínimas concedidas e depuradas;
- [ ] Page ID, Page Access Token e Instagram Business Account ID obtidos;
- [ ] Business Verification e App Review concluídos quando exigidos;
- [ ] aplicação em Live somente depois da aprovação;
- [ ] receiver público HTTPS implementado e assinatura validada;
- [ ] Verify Token guardado no cofre;
- [ ] versão Graph vigente fixada;
- [ ] comentário, menção e DM reais recebidos com `correlation_id`.

## 8. O que nunca fazer

- não usar a senha da conta como token de API;
- não enviar segredos por email, WhatsApp, Instagram ou ticket;
- não gravar segredos em YAML ou Git;
- não expor App Secret, Page Token ou Storefront private token no navegador;
- não pedir permissões `write_*` sem uma operação aprovada que as justifique;
- não usar token do Graph API Explorer como solução permanente;
- não marcar integração como saudável sem uma chamada real à fonte externa;
- não confundir “conta registrada” com “canal pronto para produção”.

