# Manual técnico, executivo, comercial e estratégico: tools por finalidade

## 1. O que é esta visão por finalidade

Esta página organiza o catálogo de tools pela pergunta que o usuário realmente faz no dia a dia: qual ferramenta resolve este problema de negócio. Em vez de começar por arquivo, factory ou provider, este manual começa pela dor prática.

Essa visão é especialmente importante numa plataforma YAML-first e agentic. O catálogo não existe para ser admirado; ele existe para compor agentes, workflows, supervisores, AG-UI e integrações governadas com o menor atrito possível.

## 2. Que problema ela resolve

Uma lista alfabética ajuda quando alguém já sabe o nome da tool. Mas a maior parte das decisões de solução não nasce assim. O que normalmente chega é algo como:

- preciso responder clientes em canais sociais;
- preciso consultar catálogo e pedidos de varejo;
- preciso buscar dados esportivos confiáveis;
- preciso falar com uma API externa sem criar código novo;
- preciso acessar SQL de forma governada;
- preciso resumir documentos ou pesquisar na web.

Sem uma organização por finalidade, a plataforma parece mais complexa do que ela realmente é. Este documento troca nomes dispersos por capacidades de negócio compreensíveis.

## 3. Visão conceitual da plataforma

O conceito central observado no código é o de plataforma de capacidades reutilizáveis. Uma tool não é apenas uma função isolada. Ela é um ativo operacional que passa por descoberta, catálogo, ativação administrativa, injeção em runtime e uso governado por agentes.

Isso significa que a plataforma trabalha em três níveis ao mesmo tempo.

- Nível conceitual: capacidade de negócio, como atendimento, consulta de pedidos, busca de catálogo ou publicação social.
- Nível técnico: tool concreta, factory, credencial, timeout, cache, validação e logs com correlation_id.
- Nível operacional: um catálogo consistente que pode ser ligado a diferentes jornadas sem refazer integração do zero.

## 4. Visão técnica

O fluxo técnico resumido é este.

![4. Visão técnica](../../assets/diagrams/docs-tools-por-finalidade-diagrama-01.svg)

O diagrama mostra por que a plataforma consegue organizar as mesmas tools por vários prismas. A mesma tool pode aparecer no catálogo alfabético, na visão por finalidade e em um manual de domínio, sem perder consistência com o runtime.

## 5. Visão executiva

Para liderança, a leitura por finalidade mostra maturidade de produto. Ela deixa claro que a plataforma não é apenas um mecanismo de prompts. Ela já entrega capacidades prontas para operação comercial, marketing, atendimento, analytics governada, pesquisa e integrações corporativas.

Na prática, isso reduz custo de descoberta interna, acelera propostas e ajuda a decidir onde investir expansão de catálogo.

## 6. Visão comercial

Comercialmente, esta página é a ponte entre tecnologia e valor percebido. Ela permite explicar a plataforma por pacotes vendáveis.

- Pacote de varejo: catálogo, pedidos, marketplaces, food delivery e analytics.
- Pacote social: WhatsApp, Instagram, Chatwoot e Twitter/X.
- Pacote esportivo: múltiplos provedores para fixtures, tabelas, odds e elencos.
- Pacote governado: dyn_sql, dyn_api, RAG, observabilidade e automação.

Isso ajuda a demonstrar que a plataforma já chega com valor pronto e com caminho claro para expansão por tenant.

## 7. Mapa de finalidades

| Finalidade                | Problema resolvido                                          | Famílias principais                                                             | Destaque                        |
| ------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------- |
| Varejo e ecommerce        | consultar catálogo, pedidos, estoque e operações comerciais | marketplace, ecommerce, food delivery, UCP, dyn_sql                             | alta aderência ao produto atual |
| Social e mensageria       | publicar, responder, monitorar e orquestrar canais          | WhatsApp, Instagram, Chatwoot, Twitter/X                                        | alto valor comercial            |
| Esportes                  | consultar competições, partidas, elencos e odds             | Sportsradar, Sportmonks, API-Sports, GoalServe, Opta, Enetpulse                 | forte especialização de domínio |
| Dados governados          | expor SQL e APIs aprovadas sem criar código novo            | dyn_sql, dyn_api, proc_sql                                                      | base de governança e escala     |
| Pesquisa e web            | buscar, navegar e coletar contexto externo                  | Brave, DuckDuckGo, Google Serper, Tavily, YouTube, Playwright, Brightdata       | apoio a agentes de pesquisa     |
| Produtividade corporativa | atuar em CRM, email, colaboração e gestão                   | Brevo, Gmail, O365, Teams, HubSpot, RD Station, Jira, Trello, Slack, IFTTT, n8n | automação interna e GTM         |
| Observabilidade e suporte | medir, registrar e diagnosticar                             | Datadog, Loki, Sentry, observability_health_check                               | governança operacional          |
| Conhecimento e IA         | recuperar contexto, trabalhar dados e apoiar análise        | qa_rag, qa_search, qa_multi_vector_store, pandas, json, schema_rag_sql          | base da camada agentic          |

## 8. Finalidades prioritárias da plataforma atual

### 8.1. Varejo e ecommerce

É a frente mais estratégica do catálogo atual porque junta canais externos, operação interna e analytics governada. O código confirma ferramentas para marketplaces, suites de ecommerce, food delivery, descoberta UCP e composição com dyn_sql.

Ferramentas mais representativas:

- magalu_hub_buscar_produtos
- amazon_sp_api_buscar_produtos
- pluggto_buscar_produtos
- shopify_*, woocommerce_* e vtex_*
- ifood_*, goomer_*, food99_*, repediu_*, zedelivery_buscar_pedidos
- ucp_discovery_tool
- dyn_sql<query_id>

Leitura aprofundada: [./varejo.md](./varejo.md)

### 8.2. Social e mensageria

É a frente que transforma agentes em operadores de canal. O catálogo confirmado permite publicar, enviar mensagens, consultar insights, buscar buzz e operar inbox.

Ferramentas mais representativas:

- whatsapp_send_text_message
- whatsapp_send_template_message
- instagram_publish_media
- instagram_send_direct_message
- instagram_fetch_insights
- chatwoot_listar_conversas e demais tools Chatwoot
- twitter_search, twitter_mentions_search, twitter_user_timeline, twitter_user_lookup

Leituras aprofundadas: [./social.md](./social.md) e [./whatsapp_business.md](./whatsapp_business.md)

### 8.3. Esportes

É a frente em que a plataforma adota estratégia multi-provider. Em vez de depender de uma única fonte, o catálogo traz seis provedores com especializações complementares.

Ferramentas mais representativas:

- sportsradar_*
- sportmonks_*
- apisports_*
- goalserve_*
- opta_*
- enetpulse_*

Leitura aprofundada: [./esportes.md](./esportes.md)

### 8.4. Dados governados

É a frente mais importante para escalar integrações sem multiplicar código. dyn_sql e dyn_api permitem publicar capacidades aprovadas para agentes usando YAML e registros administrativos, com guardrails claros.

Leituras aprofundadas: [./sql_dinamico.md](./sql_dinamico.md) e [./api_dinamica.md](./api_dinamica.md)

### 8.5. Produtividade corporativa e e-mail transacional

Esta frente cobre automações corporativas em que o agente precisa
enviar comunicacao útil sem reinventar integração nova. Com a entrada de
Brevo e Resend no mesmo trilho interno, a plataforma passa a ter uma
capacidade nativa para e-mail transacional alinhada ao runtime usado pelo
produto e selecionável por configuração oficial.

Ferramentas mais representativas:

- brevo_send_email
- gmail_send_email
- gmail_search_emails
- o365_send_mail
- resend_send_email
- teams_send_channel_message
- teams_send_chat_message
- hubspot_*
- rdstation_*

Leitura complementar: [../README-TOOLS-LIB.md](../README-TOOLS-LIB.md)

## 9. Como escolher a tool certa

Use esta lógica simples.

1. Se o problema é de domínio de negócio pronto, procure primeiro nas famílias de varejo, social ou esportes.
2. Se o problema é falar com um endpoint REST ou uma query aprovada do cliente, procure dyn_api ou dyn_sql.
3. Se o problema é contexto, pesquisa ou navegação, procure as tools de busca, scraping e RAG.
4. Se o problema é automação corporativa, procure primeiro as tools internas de e-mail transacional, `brevo_send_email` ou `resend_send_email`, e depois Gmail, O365, CRM, colaboração ou observabilidade.

Essa ordem evita dois erros frequentes.

- Usar uma tool genérica quando já existe uma tool de negócio mais segura e expressiva.
- Criar integração nova quando o catálogo já oferece um contrato governado suficiente.

## 10. Limites e pegadinhas

- Esta organização por finalidade é uma camada de leitura, não uma taxonomia rígida do banco.
- A mesma tool pode aparecer em mais de uma finalidade. Exemplo: Chatwoot serve atendimento, operação e CRM de suporte.
- dyn_sql e dyn_api não são ferramentas prontas de domínio; são contratos de governança que precisam de configuração válida.
- Ter uma tool descoberta no código não garante que ela esteja habilitada para todos os tenants ou com segredos configurados.

## 11. Evidências no código

- src/agentic_layer/tools/tools_library_builder.py
  - Motivo da leitura: confirmar a descoberta do catálogo e a distribuição por categorias.
  - Comportamento confirmado: o builder descobriu 238 tools concretas no código lido.
- src/config/config_cli/configuration_factory.py
  - Motivo da leitura: confirmar como o catálogo entra no runtime.
  - Comportamento confirmado: tools_library deve chegar vazia e recebe o catálogo builtin ativo do banco.
- src/config/agentic_assembly/tool_resolver.py
  - Motivo da leitura: confirmar composição do catálogo efetivo por escopo.
  - Comportamento confirmado: raiz, workflow ou escopo local de DeepAgent podem sobrescrever ids; o catálogo builtin preenche ausências.
