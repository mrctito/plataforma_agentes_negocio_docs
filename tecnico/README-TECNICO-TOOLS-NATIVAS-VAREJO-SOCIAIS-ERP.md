# Manual técnico e operacional: tools nativas para varejo, sociais e ERP

## 1. O que este manual cobre

Este documento explica o caminho técnico real das tools nativas com foco em três blocos de maior valor operacional no código lido.

1. Varejo e food service.
2. Canais sociais e relacionamento.
3. ERP e dados corporativos governados.

O objetivo não é listar todo o catálogo do repositório. O objetivo é explicar como o catálogo builtin funciona, onde a tabela fica, como ele entra no runtime e quais toolkits concretos já estão implementados nesses domínios.

## 2. Fonte de verdade técnica

O fluxo confirmado no código é este.

1. O decorator `tool_factory` marca factories publicáveis no catálogo.
2. O `ToolsLibraryBuilder` percorre `src/agentic_layer/tools` e descobre tools e factories.
3. O builder adiciona explicitamente as families parametrizadas `dyn_sql`, `dyn_api` e `proc_sql`.
4. O sincronizador persiste o snapshot em `integrations.builtin_tool_registry`.
5. O `ConfigurationFactory` injeta as tools ativas quando o YAML chega com `tools_library: []`.

Em outras palavras, o catálogo builtin nasce no código, vira tabela operacional e só depois aparece no runtime do agente.

## 3. A tabela de tools

### 3.1. Nome da tabela

A tabela canônica confirmada no DDL é `integrations.builtin_tool_registry`.

### 3.2. O que ela guarda

Os campos principais confirmados no DDL incluem:

1. `id`
2. `impl`
3. `factory_impl`
4. `tool_name`
5. `factory_returns`
6. `description`
7. `tool_description`
8. `config`
9. `category`
10. `tags`
11. `status`
12. `discovered_from`
13. `factory_function`
14. `tool_type`
15. `decorator`
16. `function_name`
17. `path_verified`
18. `created_by`
19. `updated_by`
20. `metadata_json`

### 3.3. Como interpretar a tabela

Na prática, a linha da tabela responde:

1. qual capability builtin existe;
2. se ela é direta ou por factory;
3. qual nome canônico entra no catálogo;
4. se ela está `active`, `disabled` ou `deprecated`.

### 3.4. Guardrails do schema

O schema aplica checagens importantes.

1. `id`, `description`, `tool_description` e `category` não podem ficar vazios.
2. `status` é restrito a `active`, `disabled` e `deprecated`.
3. `factory_returns` aceita apenas `list`, `single`, `tool` ou `callable`.
4. `tool_type` aceita `direct` ou `factory_generated`.
5. `config` precisa ser objeto JSON.
6. `tags` precisa ser array JSON.
7. O binding de execução exige consistência entre `impl` e `factory_impl`.

### 3.5. Implicação operacional

Essa tabela não é decorativa. Ela sustenta:

1. listagem administrativa;
2. enable e disable por operação;
3. sincronização do builder;
4. injeção automática no runtime agentic.

## 4. Como o catálogo entra no runtime

O contrato técnico do YAML é rígido.

1. A chave `tools_library` precisa existir na raiz.
2. Ela deve chegar vazia.
3. Se vier preenchida, a `ConfigurationFactory` falha fechado.
4. Quando vazia, a factory injeta as tools ativas carregadas do banco.

Isso evita drift entre YAML manual e catálogo builtin persistido.

## 5. Superfícies administrativas e de montagem

O código lido confirma pelo menos estas superfícies relevantes.

1. `/admin/integrations/builtin-tools` para administração do catálogo builtin.
2. `/config/assembly/catalog` para consulta do catálogo efetivo no fluxo de assembly.
3. `/config/assembly/recommend-tools` para recomendação de tools conforme a situação descrita.

Essas superfícies são importantes porque mostram que o catálogo builtin não é só detalhe interno do builder. Ele também participa da governança e da experiência de montagem agentic.

## 6. Families nativas de varejo confirmadas no código

## 6.1. iFood legado

Arquivo confirmado: `src/agentic_layer/tools/domain_tools/food_delivery_tools/ifood_tools.py`

Factory confirmada: `create_ifood_tools`

Tools publicadas:

1. `ifood_autenticar`
2. `ifood_listar_eventos`
3. `ifood_confirmar_eventos`
4. `ifood_obter_pedido`
5. `ifood_atualizar_status_pedido`
6. `ifood_obter_relatorio_financeiro`
7. `ifood_listar_status_pedido`

O que isso habilita tecnicamente:

1. consumo de fila de eventos;
2. ACK de processamento;
3. consulta detalhada de pedido;
4. mudança de etapa operacional;
5. leitura de impacto financeiro.

Pré-requisitos confirmados no código:

1. `security_keys.IFOOD_CLIENT_ID`
2. `security_keys.IFOOD_CLIENT_SECRET`
3. opcionalmente `security_keys.IFOOD_AUTH_SCOPE`
4. `user_session.correlation_id`

## 6.2. iFood via SDK

Arquivo confirmado: `src/agentic_layer/tools/vendor_tools/ifood_tools/ifood_sdk_tools.py`

Factory confirmada: `create_ifood_sdk_tools`

Tools publicadas:

1. `ifood_buscar_eventos`
2. `ifood_confirmar_eventos`

Papel técnico:

Esse toolkit é mais enxuto e serve como wrapper operacional focado em evento, útil quando o fluxo quer apenas leitura filtrada e ACK sem usar o conjunto mais amplo do toolkit legado.

## 6.3. Goomer

Arquivo confirmado: `src/agentic_layer/tools/domain_tools/food_delivery_tools/goomer_tools.py`

Factory confirmada: `create_goomer_tools`

Tools publicadas:

1. `goomer_autenticar`
2. `goomer_listar_cardapios`
3. `goomer_listar_itens`
4. `goomer_definir_disponibilidade`
5. `goomer_listar_pedidos`
6. `goomer_buscar_pedidos`
7. `goomer_atualizar_status_pedido`
8. `goomer_obter_pedido`
9. `goomer_atualizar_disponibilidade_em_lote`

Papel técnico:

Esse toolkit cobre tanto catálogo quanto operação. É uma family forte para workflows de disponibilidade, ruptura, atraso e triagem operacional de pedidos.

## 6.4. Shopify Foodservice

Arquivo confirmado: `src/agentic_layer/tools/domain_tools/food_delivery_tools/shopify_foodservice_tools.py`

Factory confirmada: `create_shopify_foodservice_tools`

Tools publicadas:

1. `shopify_delivery_listar_pedidos`
2. `shopify_delivery_atualizar_estoque`
3. `shopify_delivery_definir_disponibilidade`
4. `shopify_delivery_resumo_vendas`

Papel técnico:

Ela combina pedido, estoque, disponibilidade e resumo agregado. Isso é especialmente útil para workflows de operação de delivery e ajuste tático de cardápio.

## 6.5. Zé Delivery

Arquivo confirmado: `src/agentic_layer/tools/domain_tools/food_delivery_tools/zedelivery_tools.py`

Factory confirmada: `create_zedelivery_tools`

Tool publicada:

1. `zedelivery_buscar_pedidos`

Papel técnico:

É uma surface mais simples, focada em consulta operacional de pedidos por termo ou status.

## 6.6. Toolkit genérico de e-commerce

Arquivo confirmado: `src/agentic_layer/tools/vendor_tools/ecommerce_tools/ecommerce_toolkit.py`

Factory confirmada: `create_ecommerce_tools`

Cobertura confirmada por plataforma:

Shopify:

1. `shopify_listar_produtos`
2. `shopify_obter_produto`
3. `shopify_listar_pedidos`
4. `shopify_atualizar_estoque`
5. `shopify_criar_produto`
6. `shopify_atualizar_produto`
7. `shopify_remover_produto`
8. `shopify_listar_clientes`
9. `shopify_criar_cliente`
10. `shopify_atualizar_cliente`
11. `shopify_obter_pedido`
12. `shopify_atualizar_status_pedido`

WooCommerce:

1. `woocommerce_listar_produtos`
2. `woocommerce_obter_produto`
3. `woocommerce_listar_pedidos`
4. `woocommerce_atualizar_estoque`
5. `woocommerce_criar_produto`
6. `woocommerce_atualizar_produto`
7. `woocommerce_remover_produto`
8. `woocommerce_obter_pedido`
9. `woocommerce_listar_clientes`

VTEX:

1. `vtex_listar_produtos`
2. `vtex_obter_produto`
3. `vtex_listar_pedidos`
4. `vtex_atualizar_estoque`
5. `vtex_obter_pedido`
6. `vtex_atualizar_status_pedido`
7. `vtex_buscar_cliente_por_email`

Tool transversal:

1. `ecommerce_health_check`

Papel técnico:

Esse toolkit é o bloco nativo mais amplo do comércio digital no catálogo lido. Ele é valioso tanto para agente operacional quanto para workflows de catálogo, preço, pedido e estoque.

## 6.7. Plugg.to

Arquivo confirmado: `src/agentic_layer/tools/vendor_tools/ecommerce_tools/pluggto_tools.py`

Factory confirmada: `create_pluggto_tools`

Tool publicada:

1. `pluggto_buscar_produtos`

## 6.8. Magalu Hub

Arquivo confirmado: `src/agentic_layer/tools/vendor_tools/ecommerce_tools/magalu_hub_tools.py`

Factory confirmada: `create_magalu_hub_tools`

Tool publicada:

1. `magalu_hub_buscar_produtos`

## 7. Families nativas de sociais e relacionamento

## 7.1. WhatsApp Cloud

Arquivo confirmado: `src/agentic_layer/tools/vendor_tools/whatsapp_tools/whatsapp_toolkit.py`

Factory confirmada: `create_whatsapp_cloud_tools`

Tools publicadas:

1. `whatsapp_send_text_message`
2. `whatsapp_send_template_message`

Pré-requisitos confirmados:

1. `security_keys.WHATSAPP_CLOUD_API_TOKEN`
2. `tool_config.whatsapp_cloud.phone_number_id`
3. `user_session.correlation_id`

## 7.2. Instagram Graph

Arquivo confirmado: `src/agentic_layer/tools/vendor_tools/instagram_tools/instagram_toolkit.py`

Factory confirmada: `create_instagram_tools`

Tools publicadas:

1. `instagram_publish_media`
2. `instagram_send_direct_message`
3. `instagram_fetch_insights`

Pré-requisitos confirmados:

1. `security_keys.INSTAGRAM_GRAPH_ACCESS_TOKEN`
2. `tool_config.instagram_graph.business_account_id`
3. `user_session.correlation_id`

## 7.3. Chatwoot

Arquivo confirmado: `src/agentic_layer/tools/vendor_tools/chatwoot_tools/chatwoot_toolkit.py`

Factory confirmada: `create_chatwoot_tools`

Tools publicadas:

1. `chatwoot_listar_conversas`
2. `chatwoot_obter_conversa`
3. `chatwoot_enviar_mensagem`
4. `chatwoot_atualizar_conversa`
5. `chatwoot_listar_contatos`
6. `chatwoot_criar_contato`
7. `chatwoot_listar_agentes`
8. `chatwoot_health_check`

Pré-requisitos confirmados:

1. `security_keys.chatwoot_base_url`
2. `security_keys.chatwoot_api_token`
3. `security_keys.chatwoot_account_id`
4. `user_session.correlation_id`

## 7.4. RD Station

Arquivo confirmado: `src/agentic_layer/tools/vendor_tools/rdstation_tools/rdstation_toolkit.py`

Factory confirmada: `create_rdstation_tools`

Tools publicadas:

1. `rdstation_criar_ou_atualizar_lead`
2. `rdstation_criar_negocio`
3. `rdstation_registrar_nota`

Pré-requisitos confirmados:

1. `security_keys.rdstation_client_id`
2. `security_keys.rdstation_client_secret`
3. `security_keys.rdstation_refresh_token`
4. `user_session.correlation_id`

## 8. Families nativas para ERP e dados corporativos governados

## 8.1. dyn_sql

Arquivo confirmado: `src/agentic_layer/tools/domain_tools/dynamic_sql_tools/dynamic_sql_factory.py`

Entrada canônica de catálogo: `dyn_sql`

Sintaxe confirmada no builder: `dyn_sql<query_id>`

Papel técnico:

1. resolve query governada por YAML ou registro persistido;
2. injeta `tool_config.query_id` antes da construção;
3. usa cache de tool dinâmica por tenant e configuração;
4. permite materialização lazy de uma query específica.

Uso ideal:

Consultas recorrentes e homologadas do ERP, como vendas por loja, títulos em aberto ou posição de estoque.

## 8.2. dyn_api

Arquivo confirmado: `src/agentic_layer/tools/domain_tools/dynamic_api_tools/dynamic_api_factory.py`

Entrada canônica de catálogo: `dyn_api`

Sintaxe confirmada no builder: `dyn_api<endpoint_id>`

Papel técnico:

1. resolve endpoint governado por YAML ou registro persistido;
2. injeta `tool_config.endpoint_id` antes da construção;
3. usa cache por tenant e contexto;
4. materializa operações HTTP aprovadas sem hardcode em cada agente.

Uso ideal:

Chamadas controladas para APIs de ERP, CRM, backoffice, logística ou integrações corporativas.

## 8.3. proc_sql

Entrada canônica adicionada explicitamente pelo builder: `proc_sql`

Sintaxe confirmada no builder: `proc_sql<procedure_id>`

Papel técnico:

Publicar procedures governadas como capability agentic, útil para rotinas de ERP encapsuladas no banco.

## 8.4. schema_rag_sql

Arquivo confirmado: `src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py`

Entrada canônica de catálogo: `schema_rag_sql`

Nome runtime padrão confirmado: `schema_rag_sql_query_generator`

Papel técnico:

1. gera SQL a partir de linguagem natural;
2. usa schema metadata indexado;
3. publica uma tool singular via factory;
4. serve como base de copilotos de ERP para perguntas inéditas.

## 9. Casos de uso reais de agentes e workflows

## 9.1. Workflow de atraso de pedido com múltiplos canais

Sequência técnica plausível e aderente ao catálogo lido:

1. `ifood_listar_eventos` ou `goomer_buscar_pedidos` detecta pedido problemático.
2. `dyn_api<consultar_pedido_erp>` ou `dyn_sql<pedidos_atrasados>` cruza o contexto interno.
3. Se a política exigir aprovação, o workflow entra em HIL.
4. Após decisão, `goomer_atualizar_status_pedido` ou `ifood_atualizar_status_pedido` corrige a operação.
5. `whatsapp_send_template_message` avisa o cliente.
6. `chatwoot_enviar_mensagem` registra ou continua o atendimento humano.

## 9.2. Workflow de ruptura de estoque em food service

1. `dyn_sql<estoque_critico>` identifica itens em ruptura.
2. `goomer_atualizar_disponibilidade_em_lote` retira itens indisponíveis.
3. `shopify_delivery_definir_disponibilidade` sincroniza a vitrine do canal.
4. `whatsapp_send_text_message` notifica o gerente da unidade.

## 9.3. Workflow de social selling até CRM

1. `instagram_send_direct_message` ou um fluxo de canal captura interesse inicial.
2. O agente qualifica o contato.
3. `rdstation_criar_ou_atualizar_lead` publica o lead.
4. `rdstation_criar_negocio` abre oportunidade.
5. `rdstation_registrar_nota` grava contexto comercial ou operacional.

## 9.4. Copiloto de ERP para consultoria e suporte

1. O usuário pergunta em linguagem natural.
2. O agente escolhe entre `dyn_sql`, `dyn_api` e `schema_rag_sql`.
3. Se a dúvida já tem consulta homologada, usa `dyn_sql<...>`.
4. Se a dúvida depende de API governada, usa `dyn_api<...>`.
5. Se a pergunta é inédita, usa `schema_rag_sql` para propor SQL revisável.

## 10. O que revisar quando uma tool “não aparece” para o agente

1. Verifique se a linha existe e está `active` em `integrations.builtin_tool_registry`.
2. Verifique se o YAML traz `tools_library: []` na raiz.
3. Verifique se o tenant resolveu corretamente `security_keys` e `tool_config`.
4. Verifique se a capability dinâmica foi publicada no registro persistido quando o caso for `dyn_sql`, `dyn_api` ou `proc_sql`.
5. Verifique se a family exige `user_session.correlation_id`.

## 11. O que essas tools não fazem sozinhas

1. Não substituem governança de permissão e tenant.
2. Não garantem credenciais prontas em qualquer ambiente.
3. Não tornam segura uma operação sensível sem política de aprovação.
4. Não dispensam publicação prévia no registro quando o caso é tool dinâmica governada.

## 12. Explicação 101

O catálogo builtin é como uma central de conectores e ações oficiais da plataforma.

O banco guarda a lista do que existe.

O builder atualiza essa lista a partir do código.

O YAML do agente não precisa declarar manualmente tudo isso. Ele só diz “quero usar o catálogo oficial” deixando `tools_library` vazia.

Depois disso, o agente pode combinar pieces de varejo, sociais e ERP dentro do mesmo fluxo.

## 13. Evidências no código

- `src/config/config_cli/configuration_factory.py`
  - Motivo da leitura: confirmar a injeção automática de `tools_library`.
  - Símbolo relevante: `_inject_tools_library`.
  - Comportamento confirmado: o YAML precisa trazer `tools_library` vazia para receber o catálogo builtin persistido.

- `src/integrations/schema.py`
  - Motivo da leitura: confirmar o DDL da tabela builtin.
  - Símbolo relevante: `_ddl_builtin_tool_registry`.
  - Comportamento confirmado: o schema persiste a tabela `builtin_tool_registry` com guardrails de status, tipo e binding.

- `src/api/services/admin/builtin_tools_service.py`
  - Motivo da leitura: confirmar governança administrativa do catálogo.
  - Símbolo relevante: `AdminBuiltinToolsService`.
  - Comportamento confirmado: a operação consegue listar e alterar status do catálogo builtin.

- `src/agentic_layer/tools/vendor_tools/whatsapp_tools/whatsapp_toolkit.py`
  - Motivo da leitura: confirmar tools sociais de WhatsApp.
  - Símbolo relevante: `create_whatsapp_cloud_tools`.
  - Comportamento confirmado: a factory publica envio de texto e template.

- `src/agentic_layer/tools/vendor_tools/chatwoot_tools/chatwoot_toolkit.py`
  - Motivo da leitura: confirmar tools de inbox operacional.
  - Símbolo relevante: `create_chatwoot_tools`.
  - Comportamento confirmado: a factory cobre conversas, contatos, agentes e health check.

- `src/agentic_layer/tools/vendor_tools/rdstation_tools/rdstation_toolkit.py`
  - Motivo da leitura: confirmar tools de CRM.
  - Símbolo relevante: `create_rdstation_tools`.
  - Comportamento confirmado: a factory cobre lead, negócio e nota.
