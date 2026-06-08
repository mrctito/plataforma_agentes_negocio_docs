# Manual técnico e operacional: ETL completo, integrações, transformação de dados e troubleshooting

## 1. Objetivo deste manual

Este manual explica o ETL real do código sob a perspectiva operacional e técnica. O foco é mostrar como a execução começa, como o YAML controla o comportamento, como as integrações externas entram, como os dados são transformados e onde investigar quando algo falha.

Este documento não trata ETL como sinônimo de ingestão documental nem como sinônimo de NL2SQL. Ele cobre o slice próprio de transformação estruturada da plataforma.

## 2. Entry points reais

Os entry points confirmados no código são estes.

1. POST de ETL dedicado na camada RAG, que recebe o request operacional.
2. Compatibilidade de runtime que resolve YAML, correlation_id, user_email e modo de execução.
3. Worker oficial, que consome PreparedEtlJobCommand.
4. Coordenador de fan-out, quando mais de um pipeline real está habilitado.

O request do ETL aceita os campos correlation_id, user_email, encrypted_data, execution_mode e estimated_duration_seconds. A resposta assíncrona usa o mesmo contrato-base das execuções monitoráveis da plataforma e devolve task_id, status, message e campos de acompanhamento.

## 3. Ciclo de vida de execução

O ciclo operacional confirmado no código é este.

1. A API recebe o request e resolve o YAML a partir de encrypted_data.
2. A autenticação usa o YAML já resolvido para validar a permissão de ETL.
3. O runtime garante um correlation_id válido e resolve user_email a partir do request ou de user_session no YAML.
4. O task_id é derivado do correlation_id por hash SHA1 com prefixo etl_.
5. O runtime seleciona o modo de execução.
6. O YAML é recomposto com a sessão final da execução.
7. O job é devolvido como aceito ou executado síncronamente, conforme o modo.
8. No caminho assíncrono, o comando preparado é entregue ao worker oficial.
9. O worker chama o handler PreparedEtlJobHandler.
10. O handler delega a _execute_etl_command no executor assíncrono.

O detalhe importante aqui é que o ETL não depende do request bruto depois da preparação inicial. O prepared YAML passa a ser o contrato operacional da execução.

## 4. Modos de execução

O request público do ETL aceita auto, direct_sync e direct_async.

No runtime lido, `subprocess` não segue como modo público real do HTTP. Se ele vier explicitamente no request, a API rejeita o valor de forma clara. Quando a heurística automática pede isolamento, o boundary HTTP publica `direct_async`. Isso importa porque qualquer documentação ou operação que suponha um modo subprocess dedicado no request estaria incorreta para o slice atual.

O comportamento prático é este.

1. direct_sync: a API executa o service no próprio fluxo e só responde ao final.
2. direct_async: a API aceita, registra e agenda a execução em background.
3. auto: o seletor decide com base na estimativa e no contexto.
4. subprocess: rejeitado explicitamente no request HTTP; quando a heurística automática pede isolamento, o modo final publicado é direct_async.

## 5. Service e validação inicial

O service dedicado do ETL tem uma responsabilidade central: falhar cedo antes de chamar integração externa ou banco quando o YAML não sustenta a execução.

As validações confirmadas são estas.

1. A seção extract_transform_load precisa existir.
2. extract_transform_load.enabled precisa ser verdadeiro.
3. Pelo menos um subsistema ETL precisa estar habilitado.
4. O subsistema habilitado hoje precisa ser apify ou schema_metadata.

Isso significa que o ETL não tenta “adivinhar” intenção do operador. Se a configuração não estiver completa, ele interrompe antes da execução pesada.

## 6. Orquestrador e seleção dos pipelines

O orquestrador de ETL faz coordenação explícita por pipeline. Ele monta um resultado raiz e então avalia, na ordem lida no código, estes ramos.

1. Apify Booking.com.
2. Apify Hotels.com.
3. Apify TripAdvisor.
4. Schema metadata.

Cada ramo só roda se o bloco correspondente estiver explicitamente com enabled igual a true. Caso contrário, o orquestrador registra warning dizendo que o pipeline está desabilitado.

O orquestrador não tenta inventar fallback entre provedores. Ele apenas coordena os pipelines explicitamente habilitados.

## 7. Paralelismo e fan-out por pipeline

O paralelismo do ETL é calculado a partir das unidades reais de execução, não a partir de um número abstrato configurado sem relação com o YAML. O módulo de telemetria lê o YAML preparado e conta quantos pipelines reais estão habilitados.

No slice atual, as unidades de paralelização reconhecidas são quatro.

1. apify_booking
2. apify_hotels_com
3. apify_tripadvisor
4. schema_metadata

Quando há mais de um pipeline real e o paralelismo efetivo fica maior do que um, o coordenador de fan-out publica um job filho por pipeline. Para isso ele clona o YAML e deixa habilitado apenas um pipeline por filho, preservando o correlation_id lógico da API.

Esse detalhe é crítico: o job filho ganha um escopo operacional próprio de pipeline_execution_scope, mas não ganha um correlation_id lógico novo. O código trata divergência entre worker_execution_correlation_id e parent_correlation_id como erro.

## 8. Integração com Apify

### 8.1. Onde o token vem

O cliente base do Apify procura o token em três lugares, nesta ordem.

1. extract_transform_load.apify.api_token.
2. security_keys.apify_token.
3. settings carregado de ambiente por APIFY_TOKEN.

Sem token válido, a integração falha explicitamente.

### 8.2. Configuração global confirmada

O cliente base também resolve parâmetros globais de timeout, retries, backoff, memória, concorrência e watchdog. Isso significa que a integração externa com Apify não depende só do actor_id. Ela depende também de política de execução.

As famílias de configuração confirmadas incluem timeout_seconds, max_retries, retry_delay_seconds, retry_backoff_multiplier, retry_backoff_max_seconds, memory_mbytes, max_concurrency e proxy.

### 8.3. Proxy e segurança

O cliente base resolve proxy a partir do YAML, de security_keys e de ambiente. Isso é importante porque scraping de provedores públicos pode depender de proxy estável para evitar bloqueio e instabilidade operacional.

## 9. Pipeline Apify Booking.com

O pipeline Booking.com usa um actor de busca e, opcionalmente, um actor de reviews. O fluxo observado é este.

1. O actor de busca normaliza search_list e defaults vindos do YAML.
2. Cada alvo gera um payload BookingInput com campos de busca, datas, quartos, hóspedes, score mínimo e filtros adicionais.
3. O actor executa a coleta de hotéis.
4. Cada hotel bruto é validado por BookingHotel.
5. O hotel validado é convertido para place_input e hotel_details.
6. Se reviews estiverem habilitadas, o actor de reviews busca reviews por hotel_url.
7. Cada review bruta é validada por BookingReview.
8. A review validada é convertida para review_input interno.
9. O pipeline persiste place, detalhes e reviews em hospitality.
10. Se review_nlp estiver habilitado, o pós-processador envia reviews para análise NLP.

O Booking é o pipeline Apify mais completo do slice lido porque combina busca, reviews e pós-processamento.

## 10. Pipeline Apify Hotels.com

O Hotels.com segue a mesma ideia geral, mas com esteiras mais simples no slice lido.

1. O actor normaliza place, datas e limites.
2. Cada destino da search_list gera um payload HotelsComInput.
3. O actor coleta hotéis para cada destino.
4. Cada hotel bruto é validado por HotelsComHotel.
5. O hotel validado é convertido para place_input e hotel_details.
6. O pipeline persiste o place e os detalhes em hospitality.

No código lido, o Hotels.com não mostra a mesma camada rica de coleta de reviews que Booking e TripAdvisor. Isso precisa ficar claro em qualquer explicação operacional.

## 11. Pipeline Apify TripAdvisor

O TripAdvisor é o pipeline Apify mais rico do slice lido em termos de orquestração de busca.

O fluxo técnico confirmado é este.

1. O pipeline exige search_list explícita.
2. Para cada item da search_list, ele monta query com hotel, cidade e UF.
3. O actor de busca consulta o Apify para obter places do TripAdvisor.
4. O pipeline filtra os resultados aprovados e rejeitados.
5. Os hotéis aprovados são validados por TripAdvisorHotel.
6. Os hotéis são convertidos para place_input e hotel_details.
7. Para cada hotel persistido, o actor de reviews busca avaliações.
8. Cada review é validada por TripAdvisorReview.
9. A review validada é convertida para review_input interno.
10. O pipeline persiste tudo em hospitality e opcionalmente executa review_nlp.

O código também mostra tratamento explícito de ApifyActorError, estatísticas por item de busca e rastreamento separado de hotéis aprovados, rejeitados, persistidos e reviews.

## 12. Transformação de dados Apify

O ETL Apify não trata o dataset externo como estrutura final. Ele passa por uma cadeia explícita de transformação.

### 12.1. Normalização de entrada

Cada actor normaliza search_list e aliases do YAML para um formato único. Isso evita depender rigidamente do nome original da chave.

### 12.2. Validação por modelos

Cada item bruto recebido do dataset é validado por um modelo Pydantic específico, como BookingHotel, BookingReview, HotelsComHotel, TripAdvisorHotel ou TripAdvisorReview. Isso reduz risco de persistir dado bruto inconsistente.

### 12.3. Conversão para contratos internos

Depois da validação, funções de conversão da camada hospitality transformam o dado do provedor em place_input, hotel_details e review_input. Esse é o ponto em que o ETL deixa de falar a linguagem do provider e passa a falar a linguagem interna da plataforma.

### 12.4. Persistência compartilhada

O utilitário persist_place_with_details centraliza a persistência de place e de detalhes de hotel. O ganho é manter consistência entre pipelines Apify.

### 12.5. Pós-processamento NLP

Quando review_nlp está habilitado, o ReviewPostProcessor executa análise adicional em reviews persistidas. Isso amplia o valor analítico da coleta, mas não é obrigatório para o ETL básico funcionar.

## 13. Infraestrutura hospitality

Todos os pipelines Apify dependem da infraestrutura hospitality. O código faz uma checagem explícita antes de iniciar a persistência. Se hospitality não estiver configurado, o pipeline aborta cedo com erro observável.

Isso é importante porque evita um falso positivo operacional do tipo “o actor coletou dados, então o ETL funcionou”. No slice atual, coletar e não conseguir persistir em hospitality significa falha funcional.

## 14. Pipeline schema metadata

O pipeline schema metadata segue outro tipo de integração. Em vez de chamar provider de scraping, ele conecta em banco de origem e banco de destino.

No caminho moderno, a configuração é separada em source_database e target_database. O processor valida essas estruturas, conecta no banco de origem, usa inspect do SQLAlchemy para descobrir tabelas e então grava o catálogo em dbschemas via MetadataWriter.

O fluxo técnico confirmado é este.

1. Parse do raw_config para SourceDatabaseConfig e TargetDatabaseConfig.
2. Teste de conexão com engine da origem.
3. Teste de conexão do writer no destino.
4. ensure_database e ensure_schema no catálogo técnico.
5. Descoberta das tabelas disponíveis no schema de origem.
6. Filtragem por lista de tables com suporte a wildcard.
7. Upsert de tabela por tabela.
8. Upsert de colunas em lote.
9. Inserção de chave primária, quando existe.
10. Inserção de chaves estrangeiras, quando existem.
11. Upsert de amostras de linhas, quando include_sample_rows estiver ativo.

## 15. Contrato novo e legado de schema metadata

O orquestrador ainda suporta dois contratos.

### 15.1. Contrato novo

O contrato novo é o recomendado. Ele separa claramente source_database e target_database e torna explícitos DSN, tipo de banco, schema, tables, include_sample_rows e database_code do destino.

### 15.2. Contrato legado

O contrato legado ainda existe por compatibilidade. Quando o bloco schema_metadata não segue o formato novo, o orquestrador ainda consegue cair no caminho antigo baseado em source_dsn e database_code, delegando para o módulo schema_metadata legado.

Isso precisa ser tratado como compatibilidade operacional, não como caminho preferencial para novos casos.

## 16. Relação entre ETL, schema metadata e NL2SQL

O ETL de schema metadata prepara e persiste o catálogo técnico do banco. Isso não significa que ele sozinho já entregue o runtime final de geração de SQL.

O ponto técnico correto é este.

1. O ETL grava o catálogo estrutural em dbschemas.
2. O runtime de consulta posterior usa outra camada para indexar e recuperar contexto.
3. A tool schema_rag_sql trabalha com documentos schema_metadata em vector store, não com consulta direta ao catálogo do ETL.

Em termos operacionais, ETL e NL2SQL são complementares, mas não são o mesmo pipeline.

## 17. Configurações que mais mudam comportamento

Estas são as configurações mais relevantes confirmadas no código.

1. extract_transform_load.enabled.
2. extract_transform_load.apify.enabled.
3. extract_transform_load.apify.booking.enabled.
4. extract_transform_load.apify.hotels_com.enabled.
5. extract_transform_load.apify.tripadvisor.enabled.
6. extract_transform_load.apify.global com timeout, retries, memória, watchdog e proxy.
7. extract_transform_load.apify.*.search.search_list.
8. extract_transform_load.apify.*.reviews.enabled.
9. extract_transform_load.apify.*.reviews.post_processing.review_nlp.enabled.
10. extract_transform_load.schema_metadata.enabled.
11. extract_transform_load.schema_metadata.source_database.tables.
12. extract_transform_load.schema_metadata.source_database.include_sample_rows.
13. extract_transform_load.schema_metadata.target_database.database_code.
14. execution_mode do request.

## 18. Status, streaming e cancelamento

O ETL usa o mesmo modelo operacional monitorável da plataforma para execução assíncrona. Isso significa que o cliente não recebe apenas um “ok, estamos processando”. Ele recebe task_id e superfícies de acompanhamento.

O cancelamento é cooperativo. Os actors e pipelines checam sinais de cancelamento em pontos definidos. No fan-out, o job pai também pode propagar cancelamento aos filhos.

## 19. O que acontece em caso de sucesso

No caminho feliz, o resultado final traz análise consolidada, duração, success, correlation_id, tenant_id e payload do resultado interno.

Nos pipelines Apify, o sucesso se reflete em documents_processed, chunks_created, chunks_stored, lista de processed_documents e eventuais review_nlp_summary. No schema metadata, o sucesso se reflete em tabelas processadas, colunas processadas, PKs, FKs e metadados do banco origem e destino.

## 20. O que acontece em caso de erro

Os principais erros confirmados no código são estes.

1. user_email não resolvido no request ou no YAML.
2. Falha de autenticação da permissão RAG_ETL.
3. Bloco extract_transform_load ausente ou desativado.
4. Nenhum subsistema ETL habilitado.
5. Token Apify ausente.
6. Falha ao inicializar apify-client.
7. Infraestrutura hospitality ausente.
8. Actor externo falhando ou retornando dataset sem conteúdo útil.
9. Falha em model_validate ou nas funções de conversão.
10. Falha de conexão no banco de origem ou destino de schema metadata.
11. Configuração incompleta do schema metadata.
12. Divergência de correlation_id no fan-out.

## 21. Observabilidade e diagnóstico

Para investigar o ETL, siga esta ordem.

1. Veja se o request foi aceito e qual task_id ele recebeu.
2. Veja qual correlation_id ficou efetivamente gravado no YAML preparado.
3. Veja se o service falhou antes do orquestrador.
4. Veja quais pipelines foram descobertos como enabled.
5. Se houve fan-out, inspecione os filhos individualmente.
6. Em Apify, separe erro de actor, erro de dataset, erro de transformação e erro de persistência.
7. Em schema metadata, separe erro de origem, de filtro de tabela e de writer do destino.

No slice atual, o melhor começo quase nunca é o provider externo. O melhor começo é confirmar qual pipeline o YAML realmente mandou executar.

## 22. Como colocar para funcionar

O caminho confirmado no código exige estes pré-requisitos.

1. Permissão RAG_ETL válida para o usuário que chama a API.
2. YAML com extract_transform_load.enabled igual a true.
3. Pelo menos um subsistema ETL habilitado.
4. Para Apify: token válido, actor_id configurado ou default utilizável e infraestrutura hospitality pronta.
5. Para schema metadata: DSN válido na origem e no destino, além do schema e das tabelas desejadas.
6. Worker oficial ativo para o caminho assíncrono.

No caminho operacional mínimo, a ordem correta é esta.

1. Preparar o YAML.
2. Garantir user_email e correlation_id.
3. Chamar o endpoint de ETL.
4. Acompanhar task_id, polling e streaming.
5. Investigar o pipeline específico em caso de falha.

## 23. Exercícios guiados

### 23.1. Exercitar o raciocínio de habilitação

Objetivo: provar que você sabe prever quais pipelines o orquestrador vai chamar.

Passos:

1. Leia um YAML com apify.enabled verdadeiro e apenas booking.enabled verdadeiro.
2. Explique por que hotels_com, tripadvisor e schema_metadata não devem rodar.
3. Explique se há fan-out ou não.

### 23.2. Exercitar a transformação Apify

Objetivo: provar que você entendeu a cadeia de transformação.

Passos:

1. Siga o caminho hotel bruto para Booking ou TripAdvisor.
2. Explique onde a validação por modelo ocorre.
3. Explique quando o dado vira place_input e review_input.

### 23.3. Exercitar schema metadata

Objetivo: provar que você entendeu a separação entre origem e destino.

Passos:

1. Explique o papel de source_database.
2. Explique o papel de target_database.
3. Explique quando sample_rows aparecem.

## 24. Checklist de entendimento

- Entendi como o request de ETL vira job monitorável.
- Entendi como o service falha cedo por YAML inválido.
- Entendi quais pipelines reais existem no código.
- Entendi como o fan-out cria filhos por pipeline.
- Entendi como o Apify entra no fluxo.
- Entendi como hotéis e reviews são transformados em estruturas internas.
- Entendi como o schema metadata lê origem e grava destino.
- Entendi por que ETL não é igual a NL2SQL.
- Entendi onde começar o troubleshooting.

## 25. Evidências no código

- src/api/routers/rag_router.py
  - Motivo da leitura: request e endpoint do ETL.
  - Símbolos relevantes: ExtractTransformLoadRequest, ExtractTransformLoadResponse, extract_transform_load_endpoint.
  - Comportamento confirmado: a API responde 202 no caminho assíncrono e autentica com YAML resolvido.

- src/api/routers/rag_runtime_etl_compat.py
  - Motivo da leitura: seleção de modo, task_id e composição da execução.
  - Símbolos relevantes: execute_etl.
  - Comportamento confirmado: task_id nasce do correlation_id, `subprocess` explícito é rejeitado no HTTP e o modo automático é normalizado para `direct_async` quando a heurística pedir isolamento.

- src/api/services/async_job_worker_payload_executor.py
  - Motivo da leitura: handler do worker para ETL.
  - Símbolos relevantes: PreparedEtlJobHandler.
  - Comportamento confirmado: o worker consome PreparedEtlJobCommand no dispatch_mode prepared_yaml.

- src/services/etl_service.py
  - Motivo da leitura: validação e consolidação do ETL.
  - Símbolos relevantes: ExtractTransformLoadService, _ensure_etl_ready.
  - Comportamento confirmado: o ETL só avança quando o bloco extract_transform_load está realmente pronto.

- src/etl_layer/orchestrator.py
  - Motivo da leitura: coordenação dos pipelines concretos.
  - Símbolos relevantes: classe principal do orquestrador, decisões de habilitação dos pipelines e execução do ramo schema metadata.
  - Comportamento confirmado: a seleção dos pipelines segue estritamente os enabled do YAML.

- src/api/services/etl_pipeline_fanout_coordinator.py
  - Motivo da leitura: fan-out real por pipeline.
  - Símbolos relevantes: execute, _publish_children, _resolve_worker_execution_correlation_id.
  - Comportamento confirmado: o job pai publica um filho por pipeline e preserva o correlation_id lógico.

- src/telemetry/etl_parallelism_runtime.py
  - Motivo da leitura: descoberta de pipelines e snapshot operacional.
  - Símbolos relevantes: _PIPELINE_DEFINITIONS, resolve_enabled_etl_pipeline_descriptors, clone_yaml_for_single_etl_pipeline.
  - Comportamento confirmado: o paralelismo do ETL é por pipeline real habilitado.

- src/ingestion_layer/clients/apify/base_apify_client.py
  - Motivo da leitura: autenticação, retries, watchdog e proxy do Apify.
  - Símbolos relevantes: resolução do token, configuração global do client e composição de proxy.
  - Comportamento confirmado: a integração Apify resolve token em múltiplas fontes e aplica política global de execução.

- src/etl_layer/providers/apify/booking_pipeline.py
  - Motivo da leitura: pipeline Booking com hotéis, reviews e NLP opcional.
  - Símbolos relevantes: classe do pipeline, extração de configuração de reviews e processamento das reviews por hotel.
  - Comportamento confirmado: o Booking transforma hotéis e reviews em contratos internos antes de persistir.

- src/etl_layer/providers/apify/hotelscom_pipeline.py
  - Motivo da leitura: pipeline Hotels.com.
  - Símbolos relevantes: HotelsComETLPipeline, _run_pipeline.
  - Comportamento confirmado: o Hotels.com persiste places e detalhes na camada hospitality.

- src/etl_layer/providers/apify/tripadvisor_pipeline.py
  - Motivo da leitura: pipeline TripAdvisor com search_list, filtragem e NLP opcional.
  - Símbolos relevantes: TripAdvisorETLPipeline, _run_pipeline.
  - Comportamento confirmado: o pipeline exige search_list e faz orquestração rica por item de busca.

- src/etl_layer/providers/data/table_schema_metadata_processor.py
  - Motivo da leitura: pipeline moderno de schema metadata.
  - Símbolos relevantes: classe principal do processor, processamento do catálogo, filtro de tabelas e resolução de sample rows.
  - Comportamento confirmado: o processor lê schema relacional, filtra tabelas e grava catálogo técnico no destino.
