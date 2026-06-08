# Manual técnico, executivo, comercial e estratégico: catálogo alfabético de tools

## 1. O que é este catálogo

Este documento é a porta de entrada para o catálogo de tools da plataforma visto pelo ângulo mais simples possível: ordem alfabética. Ele não é a fonte de verdade do runtime. A fonte de verdade continua sendo o código que declara tools com decorators, o builder que descobre essas implementações e o catálogo builtin persistido em banco.

No código lido, o builder descobre 239 tools concretas a partir de src/agentic_layer/tools. Além dessas 239 ferramentas materializadas, o runtime também reconhece famílias parametrizadas que não aparecem como uma tool concreta no discovery, porque só se tornam reais quando recebem um identificador, como dyn_sql<query_id> e dyn_api<endpoint_id>.

Em termos práticos, esta página responde à pergunta mais operacional de todas: qual nome usar quando eu preciso localizar uma tool ou conferir se ela existe no catálogo do projeto.

## 2. Que problema este documento resolve

Sem uma visão alfabética, a consulta do catálogo fica lenta por três motivos.

- Quem já conhece o nome da tool precisa navegar por vários manuais temáticos até confirmar se ela existe.
- Quem está depurando YAML, assembly agentic ou prompts perde tempo tentando descobrir se errou o nome ou se a tool realmente não existe.
- Quem faz atendimento, demonstração ou suporte precisa de um ponto rápido para checar se um capability pedido pelo cliente já está coberto pelo catálogo.

Este manual resolve isso organizando o inventário por ordem de nome, mas sem esconder o funcionamento real da plataforma.

## 3. Como as tools entram na plataforma

O ciclo observado no código é este:

1. Um módulo Python declara uma tool direta com @tool ou uma factory com @tool_factory.
2. O builder percorre src/agentic_layer/tools, identifica funções, classes e factories válidas e monta a lista descoberta.
3. O sincronizador grava esse inventário em integrations.builtin_tool_registry, preservando metadados e estado administrativo.
4. O cache compartilhado carrega o catálogo persistido do banco.
5. A ConfigurationFactory exige que tools_library exista e chegue vazia no YAML recebido.
6. O catálogo builtin ativo é então injetado automaticamente na raiz do YAML.
7. O resolver de catálogo compõe o catálogo efetivo do alvo, respeitando precedência entre raiz, escopo local e catálogo builtin persistido.

Esse desenho importa porque evita o problema clássico de catálogos divergentes: uma coisa no código, outra no YAML e outra na documentação.

## 4. Visão técnica

Tecnicamente, o catálogo atual se divide em seis grupos confirmados pelo builder.

| Categoria          | Quantidade descoberta no código | Significado prático                                                            |
| ------------------ | ------------------------------: | ------------------------------------------------------------------------------ |
| vendor_tools       |                             130 | integrações prontas com provedores externos e plataformas de negócio           |
| domain_tools       |                              60 | ferramentas específicas do domínio da plataforma, inclusive operação governada |
| external_tools     |                              39 | busca web, navegação, scraping e chamadas HTTP genéricas                       |
| vector_store_tools |                               4 | recuperação RAG e busca vetorial                                               |
| custom_tools       |                               4 | utilidades especializadas do projeto                                           |
| system_tools       |                               2 | controles operacionais e interação humana                                      |

Há também três contratos parametrizados reconhecidos pelo validador agentic, mas não descobertos como tools concretas pelo builder.

- dyn_sql<query_id>
- dyn_api<endpoint_id>
- proc_sql<procedure_id>

Esses contratos existem para que o runtime crie a tool sob demanda a partir de configuração YAML ou de registros publicados em banco.

## 5. Visão executiva

Para liderança, este catálogo mostra maturidade de plataforma. O valor não está só na quantidade de tools, mas no fato de elas entrarem por um pipeline governado: descoberta automática, persistência central, ativação administrativa e injeção controlada no runtime.

Na prática, isso reduz custo de implantação, acelera novas demos, melhora previsibilidade operacional e evita dependência de scripts isolados por projeto.

## 6. Visão comercial

Comercialmente, o catálogo é a prova de que a plataforma não depende de um único assistente genérico. Ela já traz blocos vendáveis para varejo, canais sociais, produtividade corporativa, observabilidade, esportes, integrações web e governança de dados.

Isso ajuda a responder duas perguntas comuns de cliente.

- A plataforma já vem com integrações reais ou tudo precisa ser desenvolvido do zero?
- O agente só conversa ou ele também executa trabalho útil em sistemas de negócio?

O catálogo comprovado no código sustenta a resposta: a plataforma já vem com um acervo operacional relevante e com mecanismo claro para crescer.

## 7. Segmentos em destaque

Os pedidos mais estratégicos observados nesta base hoje estão concentrados em quatro frentes.

- Varejo: marketplace, food delivery, ecommerce e analytics governada com dyn_sql.
- Social: WhatsApp Cloud, Instagram Graph, Chatwoot e Twitter/X.
- Esportes: seis provedores com dados complementares de competição, partida, elenco, tabela e odds.
- Governança: dyn_api, dyn_sql, RAG, observabilidade e catálogo builtin administrável.

Se o objetivo for entender a lógica de negócio por trás das tools, a navegação mais útil não é esta página. Nesse caso, consulte [./por_finalidade.md](./por_finalidade.md), [./varejo.md](./varejo.md), [./social.md](./social.md), [./esportes.md](./esportes.md), [./sql_dinamico.md](./sql_dinamico.md) e [./api_dinamica.md](./api_dinamica.md).

## 8. Catálogo alfabético consolidado

### A

- advanced_string_inverter
- alloy_buscar_pedidos
- amazon_sp_api_buscar_produtos
- anp_revendedores_buscar
- api_http_get
- api_http_post_json
- apisports_classificacao_liga
- apisports_jogadores_por_time
- apisports_partida_por_id
- apisports_partidas_por_data
- apisports_times_por_nome

### B

- brevo_send_email
- brasilapi_consultar_cep
- brasilapi_consultar_cnpj
- brasilapi_consultar_feriados
- brasilapi_listar_bancos
- brave_search_wrapper
- brightdata_webscraper_wrapper

### C

- calculator
- cardapio_web_buscar_pedidos
- chatwoot_atualizar_conversa
- chatwoot_criar_contato
- chatwoot_enviar_mensagem
- chatwoot_health_check
- chatwoot_listar_agentes
- chatwoot_listar_contatos
- chatwoot_listar_conversas
- chatwoot_obter_conversa
- clima_comparar_servicos
- cnpj_validar_numero
- consulta_remedios_buscar_medicamentos
- cpf_validar_numero
- crm_food_buscar_pedidos

### D

- datadog_criar_dashboard
- datadog_enviar_metrica
- dataforseo_search_answer
- dataforseo_search_json
- delivery_much_buscar_pedidos
- duckduckgo_search_wrapper
- dyn_api<endpoint_id>
- dyn_sql<query_id>

### E

- ecommerce_health_check
- enetpulse_buscar_classificacao
- enetpulse_buscar_equipes
- enetpulse_buscar_eventos
- enetpulse_buscar_fixtures
- enetpulse_buscar_odds

### F

- fazer_transferencia
- food99_atualizar_status
- food99_autenticar
- food99_buscar_pedidos
- food99_listar_pedidos
- food99_obter_pedido
- food99_sincronizar_cardapio

### G

- gdrive_fetch_files
- gmail_search_emails
- gmail_send_email
- goalserve_classificacao_liga
- goalserve_jogadores_por_time
- goalserve_partida_por_id
- goalserve_partidas_por_data
- goalserve_times_por_nome
- google_serper_wrapper
- google_trends
- goomer_atualizar_disponibilidade_em_lote
- goomer_atualizar_status_pedido
- goomer_autenticar
- goomer_buscar_pedidos
- goomer_definir_disponibilidade
- goomer_listar_cardapios
- goomer_listar_itens
- goomer_listar_pedidos
- goomer_obter_pedido

### H

- http_get
- http_post
- hubspot_associar_contato_negocio
- hubspot_buscar_contato_por_email
- hubspot_criar_contato
- hubspot_criar_negocio
- hubspot_registrar_nota
- human_gate

### I

- ifood_atualizar_status_pedido
- ifood_autenticar
- ifood_buscar_eventos
- ifood_confirmar_eventos
- ifood_listar_eventos
- ifood_listar_status_pedido
- ifood_obter_pedido
- ifood_obter_relatorio_financeiro
- ifttt_trigger
- instagram_fetch_insights
- instagram_publish_media
- instagram_send_direct_message

### J

- jira_get_projects
- jira_issue_create
- jira_search_issues
- json_format
- json_minify
- json_parse
- json_validate

### K

- keeta_buscar_pedidos

### L

- linx_delivery_hub_buscar_pedidos
- linx_neemo_buscar_pedidos
- loki_push_logs
- loki_query_logs

### M

- magalu_hub_buscar_produtos
- mercadopago_criar_preferencia
- mercadopago_obter_pagamento
- meus_postos_buscar

### N

- n8n_trigger

### O

- o365_list_events
- o365_send_mail
- o365_upload_file
- observability_health_check
- openweather_alertas
- openweather_clima_atual
- openweather_previsao
- opta_buscar_classificacao
- opta_buscar_competicoes
- opta_buscar_equipes
- opta_buscar_fixtures
- opta_buscar_jogadores

### P

- pandas_dataframe_wrapper
- pbi_execute_dax
- pbi_list_datasets
- pbi_refresh_dataset
- pd_agrupar_somar
- pd_converter_planilha
- pd_criar_coluna
- pd_estatisticas_basicas
- pd_exportar_planilha_json
- pd_filtrar_planilha
- pd_inserir_linha
- pd_ler_planilha
- pd_salvar_planilha
- pluggto_buscar_produtos
- proc_sql<procedure_id>
- pw_click
- pw_close
- pw_content
- pw_extract_text
- pw_navigate
- pw_type
- python_repl_domain_wrapper

### Q

- qa_multi_vector_store
- qa_rag
- qa_rag_with_sources
- qa_search

### R

- rdstation_criar_negocio
- rdstation_criar_ou_atualizar_lead
- rdstation_registrar_nota
- resend_send_email
- repediu_atualizar_disponibilidade_itens
- repediu_atualizar_status_pedido
- repediu_autenticar
- repediu_listar_itens
- repediu_listar_pedidos
- repediu_obter_pedido

### S

- schema_rag_sql
- sentry_capturar_erro
- sentry_criar_release
- shopify_atualizar_cliente
- shopify_atualizar_estoque
- shopify_atualizar_produto
- shopify_atualizar_status_pedido
- shopify_criar_cliente
- shopify_criar_produto
- shopify_delivery_atualizar_estoque
- shopify_delivery_definir_disponibilidade
- shopify_delivery_listar_pedidos
- shopify_delivery_resumo_vendas
- shopify_listar_clientes
- shopify_listar_pedidos
- shopify_listar_produtos
- shopify_obter_pedido
- shopify_obter_produto
- shopify_remover_produto
- smtp_fetch_emails
- smtp_send_email
- sportmonks_classificacao_temporada
- sportmonks_jogador_por_id
- sportmonks_partida_por_id
- sportmonks_partidas_por_data
- sportmonks_time_por_id
- sportsradar_classificacao_competicao
- sportsradar_partidas_competicao
- sportsradar_perfil_jogador
- sportsradar_resumo_evento
- sportsradar_times_competicao
- string_inverter
- stripe_criar_payment_intent
- stripe_criar_refund

### T

- tavily_search_wrapper
- teams_create_online_meeting
- teams_list_channels
- teams_send_channel_message
- teams_send_chat_message
- trello_add_checkitem
- trello_add_checklist
- trello_add_comment
- trello_add_label
- trello_add_member
- trello_attach_url
- trello_create_card
- trello_create_list
- trello_get_board_lists
- trello_get_card
- trello_get_my_boards
- trello_move_card_to_list
- trello_remove_label
- trello_remove_member
- trello_search_cards
- trello_update_card
- twitter_mentions_search
- twitter_search
- twitter_user_lookup
- twitter_user_timeline

### U

- uber_buscar_produtos
- ucp_discovery_tool
- uuid_generate

### V

- viacep_consultar_cep
- vtex_atualizar_estoque
- vtex_atualizar_status_pedido
- vtex_buscar_cliente_por_email
- vtex_listar_pedidos
- vtex_listar_produtos
- vtex_obter_pedido
- vtex_obter_produto

### W

- weatherapi_clima_atual
- weatherapi_previsao
- whatsapp_send_template_message
- whatsapp_send_text_message
- woocommerce_atualizar_estoque
- woocommerce_atualizar_produto
- woocommerce_criar_produto
- woocommerce_listar_clientes
- woocommerce_listar_pedidos
- woocommerce_listar_produtos
- woocommerce_obter_pedido
- woocommerce_obter_produto
- woocommerce_remover_produto
- write_todos

### Y

- youtube_search_wrapper

### Z

- zedelivery_buscar_pedidos

## 9. Limites e pegadinhas

- Esta lista mostra o catálogo descoberto no código, não o estado administrativo final de ativo ou desativado no banco.
- dyn_sql<...>, dyn_api<...> e proc_sql<...> não são entradas concretas descobertas pelo builder; são contratos parametrizados do runtime.
- Uma tool aparecer aqui não significa que qualquer tenant já tenha as credenciais, conexões ou registros necessários para usá-la.
- Alguns domínios coexistem com duas trilhas complementares, como iFood operacional e iFood por SDK de eventos.

## 10. Evidências no código

- src/agentic_layer/tools/tools_library_builder.py
  - Motivo da leitura: confirmar descoberta automática, categorias e total de tools concretas.
  - Comportamento confirmado: o builder percorre src/agentic_layer/tools e monta o catálogo descoberto por @tool, @tool_factory, funções create_* e classes Tool.
- src/agentic_layer/tools/tools_library_cache.py
  - Motivo da leitura: confirmar a origem do catálogo efetivo do runtime.
  - Comportamento confirmado: o catálogo builtin é carregado de integrations.builtin_tool_registry com cache thread-safe.
- src/config/config_cli/configuration_factory.py
  - Motivo da leitura: confirmar a injeção em tools_library.
  - Comportamento confirmado: o YAML precisa trazer tools_library vazia e recebe injeção automática do catálogo ativo do banco.
- src/config/agentic_assembly/tool_resolver.py
  - Motivo da leitura: confirmar precedência entre raiz, escopo e catálogo builtin.
  - Comportamento confirmado: tools locais sobrescrevem por id e o catálogo builtin preenche ausências.
