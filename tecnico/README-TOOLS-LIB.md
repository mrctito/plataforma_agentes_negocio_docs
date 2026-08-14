# Biblioteca técnica de componentes reutilizáveis

## Objetivo, limite e fonte de verdade

Este documento é um inventário complementar para busca rápida por desenvolvedores e agentes de IA. O gate arquitetural obrigatório de reuso permanece em `.claude/rules/reuso-instructions.md`; este arquivo não o substitui.

O inventário foi reconstruído em 2026-07-29 a partir do código executável, antes da leitura da versão anterior deste catálogo. A descoberta cobriu 2.668 arquivos Python e 270 arquivos JavaScript, além de HTML, exports de pacote e testes de contrato usados para provar publicação e consumo. Foram examinados `__all__`, imports, `export`, `module.exports`, `window.*`, `globalThis.*`, `CustomEvent`, custom elements, scripts carregados por HTML e consumidores indiretos.

Convenções deste catálogo:

- **Uso atual: Sim** significa importação, chamada, composição, carregamento por HTML ou teste de contrato comprovado.
- **Uso atual: Potencial** significa API funcional e reutilizável, mas sem consumidor de produção adicional comprovado nesta rodada; a hipótese e a limitação são explícitas.
- Itens relacionados podem aparecer como uma família quando formam um único contrato público e não fazem sentido isoladamente.
- Caminhos são relativos à raiz do repositório e apontam para o arquivo executável que sustenta a entrada.
- Código legado, removido, experimental, comentado, vendor ou privado sem valor fora do módulo foi excluído.

## Prioridade alta — infraestrutura e contratos transversais Python

### BaseCorrelationComponent, BaseCorrelationManager e BaseCorrelationFactory

- Descrição: bases que validam e propagam `yaml_config`, `user_session`, `user_email`, `correlation_id` e logger oficial para componentes, managers e factories sem duplicar wiring de contexto.
- Tags: `correlação`, `logging`, `contexto`
- Tipo: classe
- Arquivo: `src/core/base_correlation_component.py`
- Linguagem: Python
- Responsabilidade principal: estabelecer contexto correlacionado e criar componentes filhos preservando a identidade da execução.
- Dependências principais: `src/core/correlation_id_factory.py`, `src/core/logging_system.py`.
- Acoplamento forte com domínio: Não; é uma base transversal.
- Uso atual: Sim; há consumidores em `src/channel_layer/processor.py`, `src/qa_layer/rag_engine/generation_engine.py` e dezenas de módulos de ingestão.
- Seguro reutilizar como está: Sim, quando o chamador já recebeu o YAML resolvido no boundary oficial.
- Riscos ou limitações: falha fechado sem sessão e não autoriza gerar uma nova correlação abaixo do boundary.
- Sugestão de melhoria: manter exemplos mínimos de composição de filhos junto à documentação arquitetural.
- Prioridade: Alta.

### CorrelationIdFactory, compose_correlation_id e resolve_http_request_correlation_id

- Descrição: família canônica para gerar e validar a identidade de uma execução no boundary, preservá-la em camadas internas e resolver sua precedência em requests HTTP.
- Tags: `correlação`, `http`, `boundary`
- Tipo: factory e helper
- Arquivos: `src/core/correlation_id_factory.py`, `src/core/correlation_utils.py`, `src/api/request_correlation.py`.
- Linguagem: Python
- Responsabilidade principal: impedir IDs ad hoc e centralizar geração, validação, propagação e leitura do header/estado HTTP.
- Dependências principais: relógio do sistema e request FastAPI.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; `CorrelationIdFactory` aparece em mais de cem imports e o resolvedor HTTP é usado por routers de logs, AG-UI, agentes e interações.
- Seguro reutilizar como está: Sim; geração somente no boundary dono da execução e composição somente para preservar valor existente.
- Riscos ou limitações: usar a factory em service, worker interno ou helper cria uma execução artificial e rompe rastreabilidade ponta a ponta.
- Sugestão de melhoria: migrar qualquer leitura manual remanescente de `X-Correlation-Id` para o resolvedor HTTP.
- Prioridade: Alta.

### Logging System canônico

- Descrição: runtime central que configura logging estruturado, nomes físicos correlacionados, sanitização e factories de logger para execuções reais e eventos técnicos sem correlação.
- Tags: `logging`, `observabilidade`, `correlação`
- Tipo: service
- Arquivo: `src/core/logging_system.py`
- Linguagem: Python
- Responsabilidade principal: expor `create_logger_with_correlation`, `create_component_logger`, `create_ingestion_logger`, `emit_canonical_log`, `setup_logging` e as classes `SystemLogger` e `RequestScopedLogger`.
- Dependências principais: `src/core/log_canonical_fields.py`, `src/core/log_local_storage.py`, `src/core/log_origin_metadata.py`, `src/config/settings.py`.
- Acoplamento forte com domínio: Não; é a escrita oficial de logs de produto.
- Uso atual: Sim; é o módulo Python mais consumido do repositório, com aproximadamente 290 imports diretos no recorte analisado.
- Seguro reutilizar como está: Sim, respeitando a distinção entre execução correlacionada e log técnico fora de processo real.
- Riscos ou limitações: logger paralelo, `FileHandler` no call site ou payload manual perde contrato, destino e correlação.
- Sugestão de melhoria: manter novas compatibilidades de logger dentro de `emit_canonical_log`, não nos consumidores.
- Prioridade: Alta.

### Catálogo global e builders de campos canônicos de log

- Descrição: fonte única dos campos globais, grupos canônicos e builder que rejeita `event_name` vazio e separa vocabulário transversal de campos locais de cada slice.
- Tags: `logging`, `schema`, `validação`
- Tipo: helper e constante
- Arquivos: `src/core/log_canonical_fields.py`, `src/core/log_event_name_contract.py`, `src/core/log_field_catalog.py`, `src/core/generated_log_event_catalog.py`.
- Linguagem: Python
- Responsabilidade principal: construir payloads estruturados, validar nomes de evento e publicar o catálogo consultado por logging, testes e analisador.
- Dependências principais: biblioteca padrão e builders de vocabulário dos slices.
- Acoplamento forte com domínio: Não; os vocabulários de domínio dependem dele, não o contrário.
- Uso atual: Sim; `log_canonical_fields.py` tem cerca de 195 imports e alimenta RAG, ingestão, Job Core, background e API.
- Seguro reutilizar como está: Sim; para campos locais, combinar com o builder oficial do slice.
- Riscos ou limitações: adicionar campo local ao catálogo global dilui ownership; criar alias como `phase` ou `step` quebra consultas baseadas em `stage`.
- Sugestão de melhoria: manter o catálogo gerado sincronizado com os produtores por gate automatizado.
- Prioridade: Alta.

### Vocabulários canônicos de observabilidade por slice

- Descrição: builders especializados que aplicam nomes de evento e campos permitidos para RAG, ingestão, workflow, tools, supervisor, MCP, canais, segurança, Job Core e background.
- Tags: `logging`, `catálogo`, `domínio`
- Tipo: helper
- Arquivos: `src/telemetry/rag/log_vocabulary.py`, `src/ingestion_layer/telemetry/log_vocabulary.py`, `src/agentic_layer/workflow/log_vocabulary.py`, `src/agentic_layer/tools/log_vocabulary.py`, `src/agentic_layer/supervisor/log_vocabulary.py`, `src/agentic_layer/mcp/log_vocabulary.py`, `src/channel_layer/log_vocabulary.py`, `src/security/log_vocabulary.py`, `src/core/job_core/log_payload_builder.py`, `src/agentic_layer/background_execution/log_vocabulary.py`.
- Linguagem: Python
- Responsabilidade principal: impedir que call sites montem `extra` arbitrário e padronizar fatos de cada domínio sobre o builder global.
- Dependências principais: `src/core/log_canonical_fields.py`.
- Acoplamento forte com domínio: Sim, deliberadamente, um vocabulário por slice.
- Uso atual: Sim; os dois vocabulários mais amplos têm cerca de 180 e 69 imports, e os demais possuem consumidores no próprio runtime.
- Seguro reutilizar como está: Sim, somente dentro do slice correspondente.
- Riscos ou limitações: trocar um vocabulário entre domínios cria nomes semanticamente falsos; ele não substitui o logger oficial.
- Sugestão de melhoria: migrar call sites remanescentes com payload manual para o builder do respectivo slice.
- Prioridade: Alta.

### BaseLogDestination e destinos concretos de escrita

- Descrição: porta e implementações para enviar logs ao filesystem, CloudWatch ou MinIO sem o produtor conhecer handlers, shippers ou decisões de armazenamento.
- Tags: `logging`, `storage`, `adapter`
- Tipo: classe e client
- Arquivo: `src/core/log_destinations.py`
- Linguagem: Python
- Responsabilidade principal: concentrar lifecycle e escrita dos destinos concretos sob `BaseLogDestination`.
- Dependências principais: logging padrão, configurações de provider e `src/core/log_object_storage.py`.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; composto pelo setup do logging e protegido por testes de destino e envio remoto.
- Seguro reutilizar como está: Sim; novos destinos devem implementar a base em vez de nascer no call site.
- Riscos ou limitações: instanciar destino concreto em código de produto volta a acoplar o produtor ao ambiente.
- Sugestão de melhoria: manter políticas de retry e deleção local centralizadas nos destinos.
- Prioridade: Alta.

### LogObjectStorage, armazenamento local e metadados de origem

- Descrição: contratos e helpers para chavear, enviar, materializar e relacionar arquivos de log por manifest/sidecar, preservando árvore de família e segregação de ambiente.
- Tags: `logging`, `minio`, `manifest`
- Tipo: client e helper
- Arquivos: `src/core/log_object_storage.py`, `src/core/log_local_storage.py`, `src/core/log_origin_metadata.py`, `src/core/log_filesystem_policy.py`, `src/core/log_lifecycle.py`, `src/core/log_gc.py`.
- Linguagem: Python
- Responsabilidade principal: abstrair storage de objetos, filesystem local, manifests, retenção e coleta de lixo sem varredura cega de `/logs`.
- Dependências principais: MinIO, filesystem, configurações de logging e campos canônicos.
- Acoplamento forte com domínio: Não; infraestrutura de observabilidade.
- Uso atual: Sim; consumido pelos destinos, provider de logs, reader canônico, locator e jobs de manutenção.
- Seguro reutilizar como está: Sim, pelas interfaces públicas e helpers de nomeação.
- Riscos ou limitações: bypass com `glob`, `rglob`, `listdir` ou deleção anterior ao upload viola o contrato físico.
- Sugestão de melhoria: manter toda nova política de retenção em `LogLifecyclePolicy`/`LogGC`.
- Prioridade: Alta.

### CanonicalLogReader e LogFileLocator

- Descrição: cadeia oficial que lê caminhos locais materializados e resolve arquivo primário, família e rotações de uma correlação por nomes exatos, manifest e sidecar.
- Tags: `logging`, `leitura`, `locator`
- Tipo: service e helper
- Arquivos: `src/api/services/canonical_log_reader.py`, `src/log_analyzer/io/locator.py`.
- Linguagem: Python
- Responsabilidade principal: impedir acesso direto ao filesystem e entregar ao analisador somente a família deterministicamente vinculada.
- Dependências principais: `src/core/log_origin_metadata.py`, `src/core/logging_system.py`.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; reader usado por routers e serviços de logs; locator usado pelos serviços do `src.log_analyzer`.
- Seguro reutilizar como está: Sim; quando remoto, o provider deve materializar antes da leitura.
- Riscos ou limitações: o locator falha fechado sem vínculo determinístico e não aceita URI remota.
- Sugestão de melhoria: nenhuma indicada pelo código atual.
- Prioridade: Alta.

### BaseLogProvider e providers de materialização

- Descrição: fachada de provider que prepara logs locais, CloudWatch, MinIO e Northflank para leitura, listagem, telemetria, download e exclusão por um contrato uniforme.
- Tags: `logging`, `provider`, `materialização`
- Tipo: service
- Arquivo: `src/api/services/log_provider_service.py`
- Linguagem: Python
- Responsabilidade principal: expor `prepare_logs_for_correlation` e resultados tipados como `PreparedLogSource`, escondendo origem remota e cleanup temporário dos consumidores.
- Dependências principais: `CanonicalLogReader`, `LogObjectStorage`, `httpx`, settings e retry externo.
- Acoplamento forte com domínio: Não; boundary de observabilidade.
- Uso atual: Sim; consumido por `src/api/routers/logs_router.py`, `src/api/routers/ingestion_runs_router.py` e serviços administrativos.
- Seguro reutilizar como está: Sim; o chamador deve usar o factory do provider, não instanciar clientes de storage.
- Riscos ou limitações: algumas operações são deliberadamente não suportadas por certos providers e falham explicitamente.
- Sugestão de melhoria: manter capacidades por provider explícitas em vez de fallback silencioso.
- Prioridade: Alta.

### LogAnalyzerService, LogQueryService e LogFilterService

- Descrição: fachadas oficiais para análise profunda, consulta rápida com parada antecipada e filtragem objetiva de logs estruturados.
- Tags: `logging`, `análise`, `consulta`
- Tipo: service
- Arquivos: `src/log_analyzer/service.py`, `src/log_analyzer/query_service.py`, `src/log_analyzer/filter_service.py`, `src/log_analyzer/contracts.py`.
- Linguagem: Python
- Responsabilidade principal: receber requests tipados e devolver resultados estruturados, evidências, problemas de integridade e lacunas de observabilidade sem abrir JSONL manualmente.
- Dependências principais: locator, loader, engines, registry e schema do `src.log_analyzer`.
- Acoplamento forte com domínio: Não; consulta operacional transversal.
- Uso atual: Sim; API de logs importa a fachada pública e há 29 imports diretos dos contratos no recorte analisado.
- Seguro reutilizar como está: Sim; selecionar `query`, `analyze` ou filtro conforme a pergunta.
- Riscos ou limitações: limites de records/timeouts precisam ser respeitados; resultado insuficiente deve registrar lacuna, não inferência.
- Sugestão de melhoria: ampliar question types junto com contratos, engine e oráculo de validação na mesma mudança.
- Prioridade: Alta.

### Engine, plugins, loader e validação do Log Analyzer

- Descrição: extensões internas reutilizáveis para carregar JSONL, registrar análises/insights, validar schema e comparar a CLI com um oráculo independente.
- Tags: `logging`, `plugin`, `validação`
- Tipo: classe e helper
- Arquivos: `src/log_analyzer/analysis/engine.py`, `src/log_analyzer/analysis/registry.py`, `src/log_analyzer/analysis/insights/base.py`, `src/log_analyzer/io/loader.py`, `src/log_analyzer/schema_registry.py`, `src/log_analyzer/validation/`.
- Linguagem: Python
- Responsabilidade principal: permitir novas análises e validações sem alterar as fachadas públicas nem duplicar parsing.
- Dependências principais: contratos do Log Analyzer e leitor/locator canônicos.
- Acoplamento forte com domínio: Não; plugins podem interpretar domínios, mas a engine é neutra.
- Uso atual: Sim; serviços montam loader/engine/registry e a validation suite é consumida por testes e scripts oficiais.
- Seguro reutilizar como está: Sim, dentro da arquitetura do analisador.
- Riscos ou limitações: plugin fora do registry não participa da análise; tolerância de schema e modo estrito têm finalidades diferentes.
- Sugestão de melhoria: manter cada extractor pequeno e com resultado JSON serializável.
- Prioridade: Alta.

### ExternalRetryConfig e executores de retry externo

- Descrição: política central de retry síncrono e assíncrono com backoff exponencial, classificação de erro transitório e logs de tentativa/exaustão.
- Tags: `retry`, `resiliência`, `integração`
- Tipo: helper
- Arquivo: `src/core/external_retry.py`
- Linguagem: Python
- Responsabilidade principal: evitar políticas locais divergentes em bancos, HTTP, storage, scheduler e LLM.
- Dependências principais: logger oficial, `tenacity` e funções injetadas.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; aproximadamente 56 imports em security, scheduler, ingestão, memória e integrações.
- Seguro reutilizar como está: Sim, para I/O comprovadamente transitório.
- Riscos ou limitações: não deve repetir validação, credencial inválida ou 4xx não recuperável; timeout continua sendo responsabilidade do call site.
- Sugestão de melhoria: manter classificadores específicos de provider próximos ao adapter e a mecânica aqui.
- Prioridade: Alta.

### DatabaseConnectionManager

- Descrição: factory central para pools psycopg sync/async, conexões SQLite e engines SQLAlchemy com probe, hardening e observabilidade uniformes.
- Tags: `banco-de-dados`, `pool`, `postgresql`
- Tipo: service
- Arquivo: `src/core/database_connection_manager.py`
- Linguagem: Python
- Responsabilidade principal: padronizar construção e validação de recursos de banco sem espalhar bootstrap de driver.
- Dependências principais: `psycopg`, `psycopg_pool`, SQLAlchemy, `aiosqlite`, logging e retry.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; cerca de 46 imports em stores, repositórios, startup e memória.
- Seguro reutilizar como está: Sim; o owner ainda deve fechar pools e respeitar afinidade de event loop.
- Riscos ou limitações: pool de longa vida fora do registry correto pode cruzar loops; a classe não autoriza DDL de runtime.
- Sugestão de melhoria: continuar concentrando probes por driver neste módulo.
- Prioridade: Alta.

### PostgresQueryExecutor, AsyncPostgresQueryExecutor e sessões transacionais

- Descrição: ponto central de execução SQL que mede duração, classifica erro, correlaciona operação/tabela e oferece sessões transacionais sync/async.
- Tags: `sql`, `observabilidade`, `transação`
- Tipo: service
- Arquivos: `src/core/postgres_query_executor.py`, `src/core/db_query_observability.py`.
- Linguagem: Python
- Responsabilidade principal: impedir `cursor.execute` espalhado e fornecer `fetch_one`, `fetch_all`, `execute` e transações observadas.
- Dependências principais: psycopg, logging canônico e providers de conexão/correlação.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; há cerca de 27 imports diretos e uso em Job Core, stores e manutenção de memória.
- Seguro reutilizar como está: Sim; é o boundary obrigatório para SQL da aplicação.
- Riscos ou limitações: o chamador continua responsável por statement/params corretos e por nunca executar DDL no runtime.
- Sugestão de melhoria: migrar repositories que ainda chamem execute concreto diretamente.
- Prioridade: Alta.

### RedisManager e helpers primitivos Redis

- Descrição: manager de conexões Redis com pool, retry e health check, acompanhado de helpers primitivos para operações comuns sem cliente paralelo.
- Tags: `redis`, `cache`, `conexão`
- Tipo: client e helper
- Arquivos: `src/core/cache/redis/redis_manager.py`, `src/core/cache/redis/redis_utils.py`.
- Linguagem: Python
- Responsabilidade principal: criar conexões, validar disponibilidade e padronizar operações técnicas Redis.
- Dependências principais: `redis`, system config, retry e logging oficial.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; o manager e utilitários têm dezenas de consumidores em cache, canais, sessão e runtime.
- Seguro reutilizar como está: Sim, sem instanciar `Redis(...)` diretamente no produto.
- Riscos ou limitações: conexão síncrona não deve ser usada em loop assíncrono; namespace de ambiente continua obrigatório.
- Sugestão de melhoria: manter health check e construção de pool num único ponto.
- Prioridade: Alta.

### RedisGenericCache, TypedDictRedisCache e StringValueRedisCache

- Descrição: abstrações de cache Redis que serializam valores, aplicam TTL, retry e namespace em vez de cada consumidor montar chaves e codecs.
- Tags: `redis`, `cache`, `serialização`
- Tipo: classe
- Arquivos: `src/core/cache/redis/redis_generic_cache.py`, `src/core/cache/redis/typed_dict_cache.py`, `src/core/cache/redis/string_value_cache.py`.
- Linguagem: Python
- Responsabilidade principal: oferecer caches tipados/de string sobre o manager oficial.
- Dependências principais: `RedisManager`, serialização do core e namespace de ambiente.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; usados por stores de sessão, SaaS e caches de runtime.
- Seguro reutilizar como está: Sim, quando a chave inclui ambiente/tenant conforme o contrato do dado.
- Riscos ou limitações: o próprio módulo genérico registra que anti-stampede ainda não está implementado; não usar para recomputação cara concorrente sem proteção adicional comprovada.
- Sugestão de melhoria: implementar anti-stampede centralmente se surgir caso real medido.
- Prioridade: Alta.

### Namespaces de ambiente para Redis e RabbitMQ

- Descrição: normalizadores que incorporam `ENVIRONMENT` em chaves Redis, filas, exchanges e routing keys para impedir colisão entre ambientes compartilhando infraestrutura.
- Tags: `namespace`, `redis`, `rabbitmq`
- Tipo: helper
- Arquivos: `src/core/cache/redis/redis_environment_namespace.py`, `src/core/rabbitmq_environment_namespace.py`, `src/core/environment_namespace.py`.
- Linguagem: Python
- Responsabilidade principal: construir identificadores persistentes e efêmeros segregados pelo ambiente oficial.
- Dependências principais: system config e validação de nomes.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; o namespace Redis tem aproximadamente 25 imports e RabbitMQ é usado pelo runtime de filas.
- Seguro reutilizar como está: Sim; deve anteceder qualquer chave/fila nova.
- Riscos ou limitações: concatenar ambiente manualmente pode divergir da normalização canônica.
- Sugestão de melhoria: ampliar adoção em identificadores remanescentes construídos manualmente.
- Prioridade: Alta.

### WarmResourcePool, RedisYamlCache, TTLCache e invalidação de cache

- Descrição: pool aquecido e caches locais/Redis para recursos caros, com TTL, epoch e sinais de invalidação coordenados.
- Tags: `cache`, `resource-pool`, `invalidação`
- Tipo: classe e helper
- Arquivos: `src/core/resource_pool.py`, `src/core/cache/cache_epoch.py`, `src/core/cache/invalidation_signals.py`, `src/core/cache/lru_cache.py`, `src/core/cache_key_registry.py`.
- Linguagem: Python
- Responsabilidade principal: reutilizar YAMLs e recursos estáveis sem transformar cache em persistência de estado funcional.
- Dependências principais: Redis, relógio, logging e registries de chave.
- Acoplamento forte com domínio: Não; `RedisYamlCache` é especializado em configuração.
- Uso atual: Sim; `resource_pool.py` tem cerca de 40 imports e é usado por config, API admin, stores e factories.
- Seguro reutilizar como está: Sim, quando chave, TTL, isolamento e invalidação são definidos explicitamente.
- Riscos ou limitações: processo é multi-processo; cache local não garante coerência global nem sobrevivência entre requests.
- Sugestão de melhoria: usar epoch/sinal existente antes de criar invalidação específica.
- Prioridade: Alta.

### AsyncResourceRegistry, run_sync e PersistentAsyncLoopRunner

- Descrição: infraestrutura para registrar recursos assíncronos por event loop, invalidá-los/fechá-los e manter loop persistente quando um boundary síncrono precisa hospedar recurso async de longa vida.
- Tags: `asyncio`, `event-loop`, `lifecycle`
- Tipo: service e helper
- Arquivos: `src/core/async_resource_registry.py`, `src/core/async_utils.py`.
- Linguagem: Python
- Responsabilidade principal: evitar reuso de pools, Redis, savers ou stores em loop diferente daquele que os criou.
- Dependências principais: `asyncio`, threading e logging canônico.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; registry tem cerca de 18 imports e `run_sync`/runner são usados por supervisor, stores e factories.
- Seguro reutilizar como está: Sim, respeitando owner e encerramento do recurso.
- Riscos ou limitações: usar `asyncio.run` repetidamente para recurso cacheado ainda pode envenenar afinidade; o runner persistente é necessário nesses casos.
- Sugestão de melhoria: registrar novos recursos assíncronos no registry em vez de criar singletons de módulo.
- Prioridade: Alta.

### Clock canônico, CancellationToken, InvokeTimeoutGuard e ExecutionMode

- Descrição: contratos pequenos para tempo operacional em `America/Sao_Paulo`, cancelamento cooperativo, timeout de invocação e seleção validada de modo de execução.
- Tags: `tempo`, `cancelamento`, `execução`
- Tipo: helper, classe e enum
- Arquivos: `src/core/clock.py`, `src/core/cancellation.py`, `src/core/invoke_timeout.py`, `src/core/execution_mode.py`.
- Linguagem: Python
- Responsabilidade principal: evitar `datetime.now()` solto, flags de cancelamento locais, timeouts artesanais e strings de modo sem validação.
- Dependências principais: biblioteca padrão, logging e schemas de execução.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; cancelamento tem cerca de 54 imports e os demais aparecem em runtimes, jobs, PDF e API.
- Seguro reutilizar como está: Sim, conforme a responsabilidade específica.
- Riscos ou limitações: cancelamento é cooperativo; código bloqueante precisa verificar o token em checkpoints reais.
- Sugestão de melhoria: manter novas superfícies operacionais usando o clock canônico.
- Prioridade: Alta.

### SystemConfigManager, AppSettings e settings

- Descrição: fonte central de configuração de processo que normaliza env vars, URLs, booleans, números e famílias tipadas de settings.
- Tags: `configuração`, `environment`, `settings`
- Tipo: configuração
- Arquivos: `src/config/config_api/system_config_manager.py`, `src/config/settings.py`, `src/config/logging_settings.py`.
- Linguagem: Python
- Responsabilidade principal: impedir `os.getenv` e parsing de configuração espalhados.
- Dependências principais: `src/utils/env.py`, `src/utils/database_url_parser.py`, defaults documentados pelo código.
- Acoplamento forte com domínio: Não; algumas subseções são específicas de infraestrutura.
- Uso atual: Sim; `SystemConfigManager` tem cerca de 97 imports e `settings` cerca de 54.
- Seguro reutilizar como está: Sim; secrets de tenant continuam vindo do YAML/diretório, não de settings globais.
- Riscos ou limitações: ler configuração global dentro de domínio pode ocultar dependência; prefira injeção depois do boundary.
- Sugestão de melhoria: manter novas env vars agrupadas na família tipada correspondente.
- Prioridade: Alta.

### YamlConfigManager, ConfigurationFactory e YAML Contract Validator

- Descrição: cadeia de carregamento/cache, composição e validação de configurações YAML, incluindo normalização de schema e contratos de vector store.
- Tags: `yaml`, `configuração`, `validação`
- Tipo: service, factory e validator
- Arquivos: `src/config/yaml_config_manager.py`, `src/config/config_cli/configuration_factory.py`, `src/config/validators/yaml_contract_validator.py`, `src/config/vector_store_contract.py`, `src/utils/yaml_schema_normalizer.py`.
- Linguagem: Python
- Responsabilidade principal: fornecer configuração consistente sem parser, default ou validador paralelo.
- Dependências principais: PyYAML, system config, cache e accessors canônicos.
- Acoplamento forte com domínio: Médio; conhece o contrato YAML-first da plataforma.
- Uso atual: Sim; consumido por API, CLI, ingestão, supervisor e testes de contrato.
- Seguro reutilizar como está: Sim, no boundary de configuração aplicável.
- Riscos ou limitações: `.get()` não pode mascarar chave obrigatória; normalizador não substitui resolução de credenciais/tenant.
- Sugestão de melhoria: manter qualquer campo novo rastreável do schema ao consumidor executável.
- Prioridade: Alta.

### Accessors e resolvers de configuração reutilizáveis

- Descrição: helpers que leem caminhos comuns de YAML, aplicam merge profundo e produzem snapshots tipados para PDF, JSON e Confluence.
- Tags: `yaml`, `resolver`, `normalização`
- Tipo: helper
- Arquivos: `src/utils/config_accessors.py`, `src/utils/dict_merge.py`, `src/utils/pdf_config_resolver.py`, `src/utils/pdf_runtime_snapshot.py`, `src/utils/json_config_resolver.py`, `src/utils/confluence_config_resolver.py`, `src/utils/yaml_default_tracker.py`.
- Linguagem: Python
- Responsabilidade principal: remover navegação de dicionário e defaults ad hoc dos consumidores.
- Dependências principais: YAML resolvido, logging canônico e dataclasses de snapshot.
- Acoplamento forte com domínio: Médio; cada resolver é específico do respectivo slice.
- Uso atual: Sim; accessors têm cerca de 20 imports, PDF 14 e o default tracker 23.
- Seguro reutilizar como está: Sim, somente para a árvore de configuração declarada pelo helper.
- Riscos ou limitações: resolver frontend ou local não substitui o boundary oficial `resolve_yaml_configuration`.
- Sugestão de melhoria: adicionar accessor central quando o mesmo caminho obrigatório surgir em múltiplos consumidores.
- Prioridade: Alta.

### resolve_yaml_configuration e materializadores SaaS

- Descrição: boundary oficial que localiza o YAML, resolve projeto SaaS, injeta contexto, expande placeholders e security keys e valida serialização antes do runtime.
- Tags: `yaml`, `saas`, `boundary`
- Tipo: service
- Arquivo: `src/api/routers/config_resolution.py`
- Linguagem: Python
- Responsabilidade principal: entregar a configuração efetiva por request sem cada endpoint criar resolvedor, expansor ou injetor paralelo.
- Dependências principais: CredentialManager, SecurityKeys, diretório de clientes, YAML manager e schemas SaaS.
- Acoplamento forte com domínio: Médio; boundary transversal da plataforma YAML-first.
- Uso atual: Sim; aproximadamente 67 imports do módulo em routers e testes.
- Seguro reutilizar como está: Sim; endpoints devem reutilizar esta cadeia.
- Riscos ou limitações: funções privadas internas não são API; pular etapas pode vazar placeholder ou usar tenant errado.
- Sugestão de melhoria: manter novos contextos como extensões do resultado tipado `ResolvedConfig`.
- Prioridade: Alta.

### CredentialManager, SecurityKeysStore e resolvers de secrets

- Descrição: cadeia central para obter credenciais multi-tenant, carregar security keys, expandir placeholders aninhados e detectar referências não resolvidas.
- Tags: `credenciais`, `secrets`, `multi-tenant`
- Tipo: service e helper
- Arquivos: `src/security/credential_manager.py`, `src/security/security_keys_resolver.py`, `src/security/security_keys_loader.py`, `src/security/security_keys_repository.py`.
- Linguagem: Python
- Responsabilidade principal: impedir leitura direta e dispersa de segredo em env/YAML e oferecer cache/resolução com contexto.
- Dependências principais: ClientDirectory, system config, accessors e logging de segurança.
- Acoplamento forte com domínio: Não; infraestrutura de segurança.
- Uso atual: Sim; CredentialManager tem cerca de 28 imports e o resolver de keys cerca de 23.
- Seguro reutilizar como está: Sim; o caller deve fornecer tenant/sessão resolvidos.
- Riscos ou limitações: fallback silencioso ou logging de conteúdo secreto é proibido; placeholders pendentes devem falhar.
- Sugestão de melhoria: manter novos tipos de secret no repositório central em vez de variáveis locais.
- Prioridade: Alta.

### CryptoManager, payload_crypto, SessionKeyManager e SessionCacheFactory

- Descrição: componentes para criptografia de payload, emissão/consumo de chaves de sessão e cache de sessão em memória, Redis ou arquivo por uma interface uniforme.
- Tags: `criptografia`, `sessão`, `cache`
- Tipo: service e factory
- Arquivos: `src/security/crypto_manager.py`, `src/security/payload_crypto.py`, `src/security/session_key_manager.py`, `src/security/session_cache.py`, `src/security/offline_key_store.py`.
- Linguagem: Python
- Responsabilidade principal: centralizar envelopes criptográficos e ciclo de vida de sessão sem codecs ou stores paralelos.
- Dependências principais: cryptography, Redis, DatabaseConnectionManager, security keys e logging.
- Acoplamento forte com domínio: Médio; ligado à autenticação/payload da plataforma.
- Uso atual: Sim; usado por API, Layout Mestre, offline keys e autenticação, com múltiplos imports e testes.
- Seguro reutilizar como está: Sim, pelo factory e contratos públicos.
- Riscos ou limitações: chaves e tokens nunca podem aparecer em log; backend em memória não persiste entre processos.
- Sugestão de melhoria: preferir Redis/store durável quando a sessão precisa atravessar processos.
- Prioridade: Alta.

### ClientDirectory, ClientDirectoryBase e modelos de diretório

- Descrição: fachada e base de repositório PostgreSQL para tenants, usuários, projetos, channels, access keys, YAMLs e integrações sob retry e contexto correlacionado comuns.
- Tags: `multi-tenant`, `repository`, `postgresql`
- Tipo: service e repository
- Arquivos: `src/security/client_directory.py`, `src/security/client_directory_base.py`, `src/security/client_directory_models.py`.
- Linguagem: Python
- Responsabilidade principal: padronizar acesso ao diretório de clientes sem cada bounded context abrir pool ou repetir scoping.
- Dependências principais: DatabaseConnectionManager, PostgresQueryExecutor, retry e logging de segurança.
- Acoplamento forte com domínio: Médio; é a infraestrutura compartilhada do domínio de identidade/tenant.
- Uso atual: Sim; a fachada/base somam dezenas de imports em security, integrations, chat e channels.
- Seguro reutilizar como está: Sim; novos repositórios do diretório devem herdar a base ou compor a fachada.
- Riscos ou limitações: não é DAO genérico para qualquer tabela; scoping de tenant/access key é parte do contrato.
- Sugestão de melhoria: manter queries observadas pelo executor central.
- Prioridade: Alta.

### AccessControlEvaluator, AccessKeyPolicy e catálogo de permissões

- Descrição: políticas e modelos tipados para avaliar principal, audiência, membership, precedência e permissões de endpoint sem booleanos de autorização espalhados.
- Tags: `autorização`, `permissões`, `policy`
- Tipo: validator, enum e service
- Arquivos: `src/security/access_control.py`, `src/security/access_key_policy.py`, `src/api/security/permissions.py`, `src/api/security/permission_registry.py`, `src/api/security/permission_metadata.py`.
- Linguagem: Python
- Responsabilidade principal: centralizar descritores, hierarquia, candidatos e enforcement de permissões no boundary HTTP.
- Dependências principais: diretório de clientes, FastAPI request, autenticação federada e logging.
- Acoplamento forte com domínio: Médio; segurança transversal da plataforma.
- Uso atual: Sim; catálogo/registry têm mais de quarenta consumidores e enforcement é middleware/dependency da API.
- Seguro reutilizar como está: Sim; permissões novas devem entrar no catálogo formal.
- Riscos ou limitações: validação manual no router pode divergir da precedência e da exigência de sessão federada.
- Sugestão de melhoria: manter metadados e registry sincronizados por teste de contrato.
- Prioridade: Alta.

### FederatedAuthManager e repositório/assinatura de sessão federada

- Descrição: abstração de provider de identidade, implementação Google e persistência/assinatura de sessão federada com claims tipados.
- Tags: `autenticação`, `oauth`, `sessão`
- Tipo: service e repository
- Arquivos: `src/api/security/federated_auth.py`, `src/api/security/federated_session_store.py`, `src/api/security/federated_session_signer.py`, `src/api/security/user_auth.py`.
- Linguagem: Python
- Responsabilidade principal: validar identidade externa, emitir/ler cookie assinado e combinar sessão com autorização local.
- Dependências principais: Google auth, itsdangerous, ClientDirectory e FastAPI.
- Acoplamento forte com domínio: Médio; autenticação da UI/API.
- Uso atual: Sim; consumido por auth router, guards, permission registry e dezenas de testes.
- Seguro reutilizar como está: Sim, nos boundaries oficiais de autenticação.
- Riscos ou limitações: assinatura escopada e expiração não podem ser reimplementadas; sessão federada não substitui permission check.
- Sugestão de melhoria: novos IdPs devem implementar `FederatedIdentityProvider`.
- Prioridade: Alta.

### ExecutionYamlSnapshot e TenantYamlBindingResolver

- Descrição: snapshot redigido/hashado do YAML efetivo e resolvedor de vínculo tenant/projeto que permitem reidratar execução assíncrona sem guardar segredo em payload aberto.
- Tags: `yaml`, `snapshot`, `tenant`
- Tipo: service e modelo
- Arquivos: `src/security/execution_yaml_snapshot.py`, `src/security/tenant_yaml_binding_resolver.py`.
- Linguagem: Python
- Responsabilidade principal: preservar contrato de configuração entre publicação e execução e comprovar seu vínculo ao tenant.
- Dependências principais: CredentialManager, ClientDirectory, hashing/serialização e security keys.
- Acoplamento forte com domínio: Médio; infraestrutura de execução YAML-first.
- Uso atual: Sim; snapshot tem cerca de 11 imports e binding resolver é usado por API e processos assíncronos.
- Seguro reutilizar como está: Sim; secrets devem ser re-resolvidos no destino.
- Riscos ou limitações: snapshot não é fonte autoral do YAML e não deve expor campos redigidos ao usuário.
- Sugestão de melhoria: manter alterações de formato versionadas e compatíveis com os consumidores atuais.
- Prioridade: Alta.

## Prioridade alta — runtimes, agentes, jobs e dados Python

### VectorStoreFactory, ConfiguredVectorStoreFactory e resolvedor de alvo físico

- Descrição: factory canônica que seleciona Qdrant/Azure Search e resolve coleção, índice, tenant e ambiente antes de expor o vector store às camadas de ingestão e QA.
- Tags: `vector-store`, `factory`, `multi-tenant`
- Tipo: factory e service
- Arquivos: `src/core/vector_store_factory.py`, `src/core/vector_runtime/target_resolver.py`.
- Linguagem: Python
- Responsabilidade principal: impedir que consumers instanciem clients concretos ou inventem nomes físicos de índice.
- Dependências principais: contracts RAG, system config, CredentialManager e implementações em `src/ingestion_layer/vector_stores/`.
- Acoplamento forte com domínio: Médio; infraestrutura transversal do RAG.
- Uso atual: Sim; há cerca de dez imports diretos da factory, incluindo persistence manager, vector manager e preview admin.
- Seguro reutilizar como está: Sim; deve ser o ponto inicial de qualquer novo acesso vetorial.
- Riscos ou limitações: backend novo exige registro formal; target físico não pode omitir tenant/ambiente.
- Sugestão de melhoria: manter provider e resolução física evoluindo juntos.
- Prioridade: Alta.

### Contratos RAG compartilhados e padronização de metadados

- Descrição: modelos `ContentChunk`, interface `IVectorStore` e helpers de metadados usados como fronteira estável entre ingestão, retrieval, tools e stores concretos.
- Tags: `rag`, `contrato`, `metadados`
- Tipo: interface, modelo e helper
- Arquivos: `src/shared/rag_contracts/interfaces.py`, `src/shared/rag_contracts/data_models.py`, `src/shared/rag_contracts/metadata.py`.
- Linguagem: Python
- Responsabilidade principal: reduzir dependência direta de LangChain/clients concretos nas camadas de domínio.
- Dependências principais: tipagem Python e modelos de documento compatíveis.
- Acoplamento forte com domínio: Médio; contrato base do RAG, não de uma regra de negócio.
- Uso atual: Sim; a fachada `src.shared.rag_contracts` aparece em aproximadamente 117 imports.
- Seguro reutilizar como está: Sim; adapters novos devem cumprir as interfaces sem alargá-las localmente.
- Riscos ou limitações: mudança de assinatura afeta múltiplas camadas e providers.
- Sugestão de melhoria: manter extensões aditivas e validar implementações por contrato.
- Prioridade: Alta.

### MetadataSchemaRegistry e configuração de metadados de domínio

- Descrição: catálogo de schemas e atributos de metadados para processadores e self-query RAG, com resolução por domínio e configuração comum.
- Tags: `metadados`, `schema`, `rag`
- Tipo: registry e configuração
- Arquivos: `src/shared/domain_metadata/metadata_schemas.py`, `src/shared/domain_metadata/config_utils.py`.
- Linguagem: Python
- Responsabilidade principal: evitar que cada processador declare campos, descrições e filtros de metadata do zero.
- Dependências principais: LangChain `AttributeInfo`, base correlacionada e configuração YAML.
- Acoplamento forte com domínio: Sim; reúne schemas concretos, mas o registry é compartilhado entre ingestão e QA.
- Uso atual: Sim; aproximadamente 16 imports em domain processing e domain-specific RAG.
- Seguro reutilizar como está: Sim, quando o domínio já está registrado.
- Riscos ou limitações: não é registry de schema universal; schema novo precisa refletir metadata realmente produzida.
- Sugestão de melhoria: separar novos schemas por domínio mantendo a fachada atual.
- Prioridade: Alta.

### MessageHistoryFactory e RunnableWithPersistentMessageHistory

- Descrição: factory de histórico conversacional que monta backends persistentes e adapter de chain para carregar, formatar e salvar mensagens sem acoplar QA/agentes ao storage.
- Tags: `memória`, `chat`, `factory`
- Tipo: factory e adapter
- Arquivo: `src/core/message_history_factory.py`
- Linguagem: Python
- Responsabilidade principal: centralizar history loaders e integração com chains compatíveis.
- Dependências principais: LangChain community, CredentialManager, security keys e path resolver.
- Acoplamento forte com domínio: Baixo; infraestrutura de conversa.
- Uso atual: Sim; consumido por `src/qa_layer/qa_setup_manager.py` e `src/agentic_layer/supervisor/agent_builder.py`.
- Seguro reutilizar como está: Sim, para backends suportados pelo factory.
- Riscos ou limitações: backend local não garante persistência multi-processo; credenciais precisam estar resolvidas.
- Sugestão de melhoria: manter novos backends atrás do mesmo factory.
- Prioridade: Alta.

### get_shared_postgres_store e encode_namespace_label

- Descrição: provider cacheado de `PostgresStore` LangGraph por DSN/ambiente, com pool central e codificação segura de segmentos de namespace.
- Tags: `postgresql`, `langgraph`, `store`
- Tipo: service e helper
- Arquivo: `src/core/store/postgres_store_provider.py`
- Linguagem: Python
- Responsabilidade principal: fornecer store persistente compartilhado para memória QA e backend DeepAgent sem pool/cache duplicado.
- Dependências principais: LangGraph PostgresStore, `DatabaseConnectionManager`, `resource_pool` e logging.
- Acoplamento forte com domínio: Não; o namespace é definido pelo consumidor.
- Uso atual: Sim; consumido por `src/qa_layer/memory/postgres_store_backend.py` e `src/agentic_layer/supervisor/deep_agent_supervisor.py`.
- Seguro reutilizar como está: Sim; sempre com namespace contendo ambiente e tenant.
- Riscos ou limitações: setup e afinidade do pool precisam respeitar o lifecycle oficial; namespace não é opcional.
- Sugestão de melhoria: documentar exemplos de namespace de usuário e organização.
- Prioridade: Alta.

### MemoryPurgeScope, MemoryPurgeService e bootstrap de purge

- Descrição: manutenção opt-in que reduz `store` e tabelas de checkpoint por retenção, preserva threads com HIL pendente, recusa escopo destrutivo inseguro e executa SQL pelo executor central.
- Tags: `memória`, `retenção`, `postgresql`
- Tipo: service, modelo e helper
- Arquivos: `src/core/memory/memory_purge_scope.py`, `src/core/memory/memory_purge_service.py`, `src/core/memory/memory_purge_bootstrap.py`.
- Linguagem: Python
- Responsabilidade principal: conter crescimento da memória/checkpointer no boot sem DDL e sem derrubar startup em falha.
- Dependências principais: `AsyncPostgresQueryExecutor`, `DatabaseConnectionManager`, system config e logger correlacionado.
- Acoplamento forte com domínio: Médio; conhece tabelas LangGraph e aprovação HIL persistida.
- Uso atual: Sim; chamado por `src/api/startup/runtime_bootstrap.py` e `src/api/service_api.py`, com testes em `tests/unit/core/test_02-22-40_memory_purge_service.py`.
- Seguro reutilizar como está: Sim, pelo bootstrap e com flags explicitamente habilitadas.
- Riscos ou limitações: é destrutivo; `keep=0`, wildcard de ambiente e produção possuem travas deliberadas que não podem ser contornadas.
- Sugestão de melhoria: manter qualquer novo store protegido na mesma transação e com teste de HIL.
- Prioridade: Alta.

### Modelos, store, eventos e fatos do Job Core

- Descrição: domínio genérico e durável de jobs com `JobEnvelope`, `JobExecutionContext`, estados, fatos de progresso, filhos, claims, heartbeats, cancelamento e eventos append-only.
- Tags: `job-core`, `ledger`, `contrato`
- Tipo: modelo, enum e interface
- Arquivos: `src/core/job_core/models.py`, `src/core/job_core/store.py`, `src/core/job_core/events.py`, `src/core/job_core/cancellation.py`.
- Linguagem: Python
- Responsabilidade principal: oferecer lifecycle host-transparente e persistível para ingestão, ETL, agents, workflow, scheduler e manutenção.
- Dependências principais: dataclasses, protocols, correlation e clock.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; a fachada `src.core.job_core` tem cerca de 49 imports e os contratos são usados por todos os processos assíncronos.
- Seguro reutilizar como está: Sim; novo job deve modelar apenas seu input/output sobre estes contratos.
- Riscos ou limitações: fato de progresso não é estado de negócio autoritativo e ledger não é fila.
- Sugestão de melhoria: manter extensões do envelope aditivas e justificadas por mais de um processo.
- Prioridade: Alta.

### JobProcess, JobProcessRegistry e JobCoreExecutor

- Descrição: protocolo, descritor, registry e executor que resolvem processo por `route_kind`/`dispatch_mode`, criam uma instância por execução e terminalizam o lifecycle exatamente uma vez.
- Tags: `job-core`, `registry`, `executor`
- Tipo: service e factory
- Arquivos: `src/core/job_core/job_process.py`, `src/core/job_core/registry.py`, `src/core/job_core/executor.py`.
- Linguagem: Python
- Responsabilidade principal: desacoplar processo funcional de store, host, fila e mutações de lifecycle.
- Dependências principais: modelos/store/eventos do Job Core e logger canônico.
- Acoplamento forte com domínio: Não; factories injetadas encapsulam processos concretos.
- Uso atual: Sim; `src/api/services/async_job_process_catalog.py` registra processos e o worker factory monta o executor oficial.
- Seguro reutilizar como está: Sim; registro deve ocorrer no catálogo único.
- Riscos ou limitações: segundo registry/runner ou callbacks de store dentro do processo quebram host transparency.
- Sugestão de melhoria: nenhuma indicada pelo código atual.
- Prioridade: Alta.

### PostgresJobRunStore e InMemoryJobRunStore

- Descrição: implementações durável e de teste do `JobRunStore`, com claim concorrente, eventos, heartbeat, filhos, cancelamento e terminalização observada.
- Tags: `job-core`, `postgresql`, `store`
- Tipo: repository
- Arquivos: `src/core/job_core/postgres_store.py`, `src/core/job_core/store.py`.
- Linguagem: Python
- Responsabilidade principal: persistir lifecycle genérico sem misturá-lo às projeções de domínio.
- Dependências principais: DatabaseConnectionManager, PostgresQueryExecutor, retry e schema `job_core` já provisionado.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; o worker runtime usa `PostgresJobRunStore.from_runtime_environment`; in-memory sustenta testes de contrato.
- Seguro reutilizar como está: Sim; produção deve usar o store durável.
- Riscos ou limitações: não executa DDL e não substitui tabelas canônicas de ingestão/ETL.
- Sugestão de melhoria: manter testes de concorrência/CAS junto de qualquer mudança de store.
- Prioridade: Alta.

### QueuedJobEnvelope, AsyncJobQueuePort e JobCoreJobQueue

- Descrição: boundary único de publicação assíncrona que converte envelope de transporte em criação durável no ledger, com chave de concorrência e correlações explícitas.
- Tags: `fila`, `async-job`, `boundary`
- Tipo: interface e service
- Arquivos: `src/api/services/async_job_queue_port.py`, `src/api/services/job_core_job_queue.py`.
- Linguagem: Python
- Responsabilidade principal: impedir publishers especializados de abrir transporte ou storage paralelo.
- Dependências principais: Job Core store/models e factories de runtime.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; publishers de ingestão, ETL, agents, workflow, background, HIL, chat e scheduler dependem desta porta.
- Seguro reutilizar como está: Sim; novos producers devem publicar `QueuedJobEnvelope`.
- Riscos ou limitações: envelope precisa carregar `route_kind`, `dispatch_mode` e correlações coerentes; queue não executa o trabalho.
- Sugestão de melhoria: manter codecs de input/output no descritor do processo.
- Prioridade: Alta.

### JobCoreWorkerRuntime, factory e reconciliador cancel-only

- Descrição: poller/host oficial que reclama jobs, executa processos registrados, reporta host/progresso e cancela runs comprovadamente órfãs sem requeue automático.
- Tags: `worker`, `job-core`, `reconciliação`
- Tipo: service
- Arquivos: `src/api/services/job_core_worker_runtime.py`, `src/api/services/async_job_worker_runtime_factory.py`, `src/core/job_core/reconciliation.py`, `src/api/services/process_host_reporter.py`.
- Linguagem: Python
- Responsabilidade principal: compor store, registry, executor, liveness e ownership em um único runtime de worker.
- Dependências principais: Job Core, system config, clock e logging.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; inicializado por `src/api/services/worker_bootstrap.py` e pelo runtime de processo.
- Seguro reutilizar como está: Sim; processos novos entram pelo catálogo, não alterando o poller.
- Riscos ou limitações: reconciliador apenas cancela órfão; não recupera, duplica ou reenvia trabalho.
- Sugestão de melhoria: manter intervalos de heartbeat/liveness coerentes com as configurações do worker.
- Prioridade: Alta.

### SchedulerScheduleSpec, SchedulerOccurrenceFactory e portas do scheduler

- Descrição: modelos imutáveis para agenda, job, ocorrência, confirmação e resultado de rodada, com factories e ports independentes de storage/transporte.
- Tags: `scheduler`, `agenda`, `contrato`
- Tipo: modelo, factory e interface
- Arquivos: `src/scheduler_layer/models.py`, `src/scheduler_layer/ports.py`.
- Linguagem: Python
- Responsabilidade principal: representar agenda factual e validar cron/intervalo, timezone, janela e ocorrência sem lógica em routers.
- Dependências principais: clock/timezone e dataclasses.
- Acoplamento forte com domínio: Não; scheduler genérico.
- Uso atual: Sim; modelos têm cerca de 34 imports e ports cerca de 12.
- Seguro reutilizar como está: Sim; schedules novos devem usar `SchedulerScheduleSpec`.
- Riscos ou limitações: payload funcional continua pertencendo ao processo registrado; scheduler não conhece domínio.
- Sugestão de melhoria: nenhuma indicada pelo código atual.
- Prioridade: Alta.

### SchedulerAgendaService, PostgresSchedulerRepository e AsyncJobSchedulerWorkPublisher

- Descrição: cadeia que lista schedules devidos, cria ocorrências determinísticas, persiste agenda e publica trabalho pelo AsyncJobQueuePort.
- Tags: `scheduler`, `postgresql`, `publisher`
- Tipo: service, repository e adapter
- Arquivos: `src/scheduler_layer/services.py`, `src/scheduler_layer/postgres_repository.py`, `src/scheduler_layer/async_job_publisher.py`.
- Linguagem: Python
- Responsabilidade principal: executar uma rodada de agenda sem misturar publicação, persistência e política temporal.
- Dependências principais: ports/modelos do scheduler, PostgresQueryExecutor, retry e Job Core queue.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; fachada exportada por `src/scheduler_layer/__init__.py` e composta pelo bootstrap/scheduler admin.
- Seguro reutilizar como está: Sim; novo tipo de trabalho entra pelo descriptor/queue.
- Riscos ou limitações: repository pressupõe schema provisionado e ambiente correto; publisher não deve inventar correlação de uma execução já existente.
- Sugestão de melhoria: manter jobs de manutenção como schedules declarados, não loops paralelos.
- Prioridade: Alta.

### ToolsFactory e tool_factory decorator

- Descrição: factory supervisor que carrega catálogo interno, aplica overrides, combina MCP e cria LangChain Tools, com decorator que publica metadata de factories descobríveis.
- Tags: `langchain`, `tools`, `factory`
- Tipo: factory e decorator
- Arquivos: `src/agentic_layer/supervisor/tools_factory.py`, `src/agentic_layer/tools/tool_factory_decorator.py`.
- Linguagem: Python
- Responsabilidade principal: ser o único caminho para resolver `tools` declaradas no YAML.
- Dependências principais: catálogo de tools, CredentialManager, MCP resolver, tool config resolver e logging.
- Acoplamento forte com domínio: Médio; infraestrutura agentic da plataforma.
- Uso atual: Sim; decorator tem cerca de 79 imports e `ToolsFactory` é exportada pela fachada do supervisor.
- Seguro reutilizar como está: Sim; factory nova deve usar o decorator e entrar no catálogo.
- Riscos ou limitações: instanciar tool diretamente ignora overrides, segurança, logs e descoberta.
- Sugestão de melhoria: manter descrições/metadata suficientemente específicas para roteamento.
- Prioridade: Alta.

### Helpers compartilhados de tools

- Descrição: resolvers e adaptadores para ler config de tool, despachar HTTP, normalizar resultados de busca, formatar erro seguro e emitir logs canônicos.
- Tags: `tools`, `http-client`, `normalização`
- Tipo: helper
- Arquivos: `src/agentic_layer/tools/tool_config_resolver.py`, `src/agentic_layer/tools/http_dispatch_utils.py`, `src/agentic_layer/tools/search_result_utils.py`, `src/agentic_layer/tools/error_format_utils.py`, `src/agentic_layer/tools/tool_logging_support.py`, `src/agentic_layer/tools/tools_library_cache.py`.
- Linguagem: Python
- Responsabilidade principal: evitar que cada vendor/domain tool repita headers, timeout, parsing, erros e logging.
- Dependências principais: HTTPX/requests, YAML, logging e contratos LangChain.
- Acoplamento forte com domínio: Não; adaptadores técnicos de tools.
- Uso atual: Sim; módulos principais têm de 15 a 24 imports e aparecem em várias famílias de tools.
- Seguro reutilizar como está: Sim, conforme o helper específico.
- Riscos ou limitações: normalizador de busca não deve ser usado para payload semântico diferente; erro deve continuar sem secret.
- Sugestão de melhoria: mover duplicações provadas em toolkits para estes helpers, sem generalização preventiva.
- Prioridade: Alta.

### SkillsStoreMaterializer e índice da skills_library

- Descrição: componentes compartilhados que revalidam e indexam a `skills_library` raiz e
  materializam a seleção de Skills no store dos runtimes DeepAgent e WorkflowAgent.
- Tags: `skills`, `agentic`, `store`
- Tipo: service e parser
- Arquivos: `src/agentic_layer/skills/skills_store_materializer.py`, `src/agentic_layer/skills/skills_library_index.py`.
- Linguagem: Python
- Responsabilidade principal: centralizar composição do `SKILL.md`, sanitização de `files`, versão
  por hash e reconciliação anti-fantasma, sem loader paralelo por consumidor.
- Dependências principais: AST/config agentic e contrato assíncrono do store; o materializador não
  emite log, e cada slice registra o resultado com seu builder canônico.
- Acoplamento forte com domínio: Médio; runtime agentic.
- Uso atual: Sim; o DeepAgent materializa catálogos por supervisor/subagente em
  `/skills/supervisor-<id>/main/` e `/skills/supervisor-<id>/subagent-<id>/`; o WorkflowAgent mantém
  `/skills/` como source default dentro de um namespace por node. O `DocumentCompiler` publica a
  `skills_library` para os dois targets.
- Seguro reutilizar como está: Sim; Skills devem entrar pela biblioteca declarada e por este
  materializador único.
- Riscos ou limitações: a source delimita catálogo e reconciliação, não ACL nem isolamento de
  filesystem. No DeepAgent, backend, namespace e rota `/skills/` continuam compartilhados; path e
  conteúdo precisam permanecer dentro do contrato de Skills.
- Sugestão de melhoria: manter índice e materializador sincronizados por teste de contrato.
- Prioridade: Alta.

### MCP ConfigResolver, ToolsResolver e catálogo local

- Descrição: cadeia que interpreta servidores MCP do YAML, resolve tools remotas/local IDs e persiste catálogo normalizado para descoberta pelo supervisor.
- Tags: `mcp`, `catálogo`, `resolver`
- Tipo: service e registry
- Arquivos: `src/agentic_layer/mcp/mcp_config_resolver.py`, `src/agentic_layer/mcp/mcp_tools_resolver.py`, `src/agentic_layer/mcp/local_tool_catalog.py`, `src/agentic_layer/mcp/mcp_catalog_builder.py`.
- Linguagem: Python
- Responsabilidade principal: impedir config/parsing MCP e IDs de tools paralelos.
- Dependências principais: YAML agentic, LangChain MCP adapters, ToolsFactory e logging MCP.
- Acoplamento forte com domínio: Médio; infraestrutura agentic.
- Uso atual: Sim; consumido pela factory do supervisor, assembly e testes de catálogo.
- Seguro reutilizar como está: Sim, dentro do pipeline MCP oficial.
- Riscos ou limitações: catálogo persistido precisa refletir tools realmente expostas; credenciais não podem ser materializadas em claro.
- Sugestão de melhoria: manter atualização do catálogo como operação explícita e observável.
- Prioridade: Alta.

### AgenticAssemblyService e modelos de assembly

- Descrição: fachada que coordena rascunho, parsing, validação, diff, confirmação e compilação de documentos agentic sem editar YAML como texto livre.
- Tags: `ast`, `yaml`, `agentic`
- Tipo: service
- Arquivos: `src/config/agentic_assembly/assembly_service.py`, `src/config/agentic_assembly/models.py`.
- Linguagem: Python
- Responsabilidade principal: manter AST como fonte da edição agentic e oferecer resultados tipados ao admin/studio.
- Dependências principais: parsers, validators, compilers, schema service, canonical editor e NL generators.
- Acoplamento forte com domínio: Sim; configuração agentic da plataforma.
- Uso atual: Sim; consumido por serviços/routers de assembly e protegido por suíte dedicada.
- Seguro reutilizar como está: Sim; qualquer editor/gerador agentic deve usar a fachada.
- Riscos ou limitações: escrever YAML compilado diretamente contorna validação semântica e drift detector.
- Sugestão de melhoria: manter operações novas representadas no AST antes de expô-las na UI.
- Prioridade: Alta.

### AST, parsers, validators e compilers agentic

- Descrição: família de modelos para documento, DeepAgent, supervisor, workflow, tool, skill e expression, com parsing YAML, validação semântica e compilação determinística.
- Tags: `ast`, `parser`, `validator`
- Tipo: modelo, parser, validator e compiler
- Arquivos: `src/config/agentic_assembly/ast/`, `src/config/agentic_assembly/parsers/`, `src/config/agentic_assembly/validators/`, `src/config/agentic_assembly/compilers/`.
- Linguagem: Python
- Responsabilidade principal: oferecer uma representação canônica compartilhada por editor, runtime e validação.
- Dependências principais: Pydantic/dataclasses, schema service, target/tool resolvers e YAML loader.
- Acoplamento forte com domínio: Sim; contrato agentic.
- Uso atual: Sim; ASTs de documento/workflow/deepagent têm múltiplos consumidores em src/tests.
- Seguro reutilizar como está: Sim; escolher parser/validator correspondente ao nó.
- Riscos ou limitações: mudança de modelo exige atualizar parser, validator, compiler, orchestration e documentação na mesma rodada.
- Sugestão de melhoria: manter mensagens de validação com caminho estrutural completo.
- Prioridade: Alta.

### WorkflowAgent, WorkflowState, NodeFactory e nodes reutilizáveis

- Descrição: motor LangGraph que compila workflow YAML em grafo e cria nodes de agent, DeepAgent, tool, function, condition, merge, planner, router, schema, set, subworkflow e transform por factory.
- Tags: `workflow`, `langgraph`, `factory`
- Tipo: service, factory e classe
- Arquivos: `src/agentic_layer/workflow/agent_workflow.py`, `src/agentic_layer/workflow/workflow_state.py`, `src/agentic_layer/workflow/nodes/`.
- Linguagem: Python
- Responsabilidade principal: oferecer nós composáveis e execução de workflow sem switch/callable local por endpoint.
- Dependências principais: LangGraph, config resolver, edge compiler, ToolsFactory, supervisor e logging workflow.
- Acoplamento forte com domínio: Médio; runtime genérico de workflow; nodes WhatsApp são específicos de canal.
- Uso atual: Sim; workflow state e engine têm dezenas de imports em orchestrators, API e testes.
- Seguro reutilizar como está: Sim, dentro do contrato YAML de nodes já suportados.
- Riscos ou limitações: node local privado ou tipo YAML novo sem factory/validator quebra a cadeia canônica.
- Sugestão de melhoria: manter nodes com responsabilidade única; integrações de canal específicas devem continuar claramente nomeadas.
- Prioridade: Alta.

### WorkflowConfigResolver, EdgeCompiler e workflow_utils

- Descrição: helpers que resolvem configuração, validam/compilam edges e oferecem acesso por caminho, templates e expressões seguras para nodes.
- Tags: `workflow`, `parser`, `expressão`
- Tipo: service e helper
- Arquivos: `src/agentic_layer/workflow/config_resolver.py`, `src/agentic_layer/workflow/edge_compiler.py`, `src/agentic_layer/workflow/workflow_utils.py`, `src/agentic_layer/workflow/node_shared.py`.
- Linguagem: Python
- Responsabilidade principal: impedir avaliação, alias de mensagem e montagem de edge duplicadas nos nodes.
- Dependências principais: AST workflow, schemas, LangChain messages e validação de expressão.
- Acoplamento forte com domínio: Médio; runtime workflow.
- Uso atual: Sim; utils/config têm aproximadamente 21 e 9 imports.
- Seguro reutilizar como está: Sim, nos nodes/workflows oficiais.
- Riscos ou limitações: expressões são deliberadamente restritas; não substituir por `eval`.
- Sugestão de melhoria: centralizar aliases novos em `node_shared`.
- Prioridade: Alta.

### ExecutionPolicyRunner e StructuredOutputHandler

- Descrição: execução de callable com retry/backoff/circuit breaker e parsing de saída estruturada por JSON Schema com tentativas controladas.
- Tags: `workflow`, `resiliência`, `serialização`
- Tipo: service e validator
- Arquivos: `src/agentic_layer/workflow/execution_policy.py`, `src/agentic_layer/workflow/structured_output.py`.
- Linguagem: Python
- Responsabilidade principal: padronizar políticas de execução e saída de nodes/agents sem loops locais.
- Dependências principais: logging workflow, JSON Schema e exceções tipadas.
- Acoplamento forte com domínio: Baixo; publicados pelo pacote workflow e utilizáveis por nodes distintos.
- Uso atual: Sim; consumidos por nodes/engine e protegidos por testes de política e structured output.
- Seguro reutilizar como está: Sim, quando a falha é elegível pela política declarada.
- Riscos ou limitações: retry de validação determinística pode só repetir custo; circuit state não é persistência distribuída.
- Sugestão de melhoria: manter políticas explícitas no config, sem defaults ocultos no node.
- Prioridade: Alta.

### MemoryFactory do supervisor

- Descrição: factory canônica de checkpointers LangGraph async (SQLite/PostgreSQL e backends suportados) com lifecycle/event-loop controlado para DeepAgents.
- Tags: `langgraph`, `memória`, `factory`
- Tipo: factory
- Arquivo: `src/agentic_layer/supervisor/memory_factory.py`
- Linguagem: Python
- Responsabilidade principal: criar savers persistentes por configuração sem usar o `MemoryManager` legado.
- Dependências principais: DatabaseConnectionManager, async utils/registry, path resolver e config models.
- Acoplamento forte com domínio: Médio; supervisor agentic.
- Uso atual: Sim; exportada por `src.agentic_layer.supervisor` e consumida por builder/supervisor.
- Seguro reutilizar como está: Sim; é o caminho oficial para memória do supervisor.
- Riscos ou limitações: saver precisa permanecer no loop dono; backend em RAM foi removido do contrato oficial.
- Sugestão de melhoria: manter novos backends na factory e fora do manager legado.
- Prioridade: Alta.

### Background execution models, ports, services e repositories

- Descrição: bounded context para requests/schedules/targets, comunicação/outbox e execução durável de DeepAgent/workflow em background sobre Job Core.
- Tags: `background`, `repository`, `outbox`
- Tipo: service, interface e repository
- Arquivos: `src/agentic_layer/background_execution/models.py`, `src/agentic_layer/background_execution/ports.py`, `src/agentic_layer/background_execution/services.py`, `src/agentic_layer/background_execution/runtime.py`, `src/agentic_layer/background_execution/postgres_repository.py`, `src/agentic_layer/background_execution/communication_outbox_repository.py`.
- Linguagem: Python
- Responsabilidade principal: separar estado funcional de background do lifecycle genérico do job e garantir comunicação auditável.
- Dependências principais: Job Core, ClientDirectory, PostgresQueryExecutor, workflow/deepagent runtime e logging do slice.
- Acoplamento forte com domínio: Médio; domínio background, mas compartilhado por vários tipos de agente.
- Uso atual: Sim; models/ports têm dezenas de imports e processos são registrados no catálogo assíncrono.
- Seguro reutilizar como está: Sim, para novos fluxos background sob os ports existentes.
- Riscos ou limitações: repository funcional não substitui ledger; outbox não deve ser pulada por envio direto.
- Sugestão de melhoria: manter novas comunicações atrás do enqueuer/outbox.
- Prioridade: Alta.

### Runtime multimodal neutro

- Descrição: fachada compartilhada de contratos de imagem, OCR, chunks, descriptors, configuração e embeddings de visão usada tanto na ingestão quanto na consulta.
- Tags: `multimodal`, `ocr`, `factory`
- Tipo: interface, factory e helper
- Arquivos: `src/core/multimodal_runtime/interfaces.py`, `src/core/multimodal_runtime/config_resolver.py`, `src/core/multimodal_runtime/image_descriptors.py`, `src/core/multimodal_runtime/ocr_processors.py`, `src/core/multimodal_runtime/vision_embedding_providers.py`.
- Linguagem: Python
- Responsabilidade principal: expor API neutra sem fazer QA importar implementação documental concreta.
- Dependências principais: providers OCR/vision, YAML e contratos de parsing PDF onde necessário.
- Acoplamento forte com domínio: Médio; multimodal, não uma regra de negócio.
- Uso atual: Sim; fachada e módulos têm consumidores em ingestão, QA e testes.
- Seguro reutilizar como está: Sim; usar factories/config resolver publicados em `src.core.multimodal_runtime`.
- Riscos ou limitações: subprocessos OCR possuem orçamento e lifecycle próprios; provider não configurado deve falhar explicitamente.
- Sugestão de melhoria: manter interfaces neutras e adapters específicos fora do core.
- Prioridade: Alta.

## Prioridade alta — ingestão, QA, API e integrações Python

### Modelos e interfaces centrais de ingestão

- Descrição: contratos para documentos, content/source types, requests, sources, clients, processors e orchestrators que permitem trocar implementações sem alterar o pipeline.
- Tags: `ingestão`, `contrato`, `documento`
- Tipo: modelo, enum e interface
- Arquivos: `src/ingestion_layer/core/data_models.py`, `src/ingestion_layer/core/interfaces.py`.
- Linguagem: Python
- Responsabilidade principal: padronizar entrada e saída de conteúdo textual, PDF, DOCX, Excel, JSON, imagem, web, cloud e Confluence.
- Dependências principais: dataclasses, ABCs e contratos RAG compartilhados.
- Acoplamento forte com domínio: Médio; fundação da ingestão, não de uma fonte específica.
- Uso atual: Sim; data models têm aproximadamente 225 imports e interfaces são implementadas por clients/processors.
- Seguro reutilizar como está: Sim, dentro do pipeline de ingestão.
- Riscos ou limitações: subclasses precisam implementar validação/metadata; `Any` em metadata é boundary dinâmico, não deve vazar para core de negócio.
- Sugestão de melhoria: manter tipos novos registrados na factory e dispatcher na mesma mudança.
- Prioridade: Alta.

### ContentClientFactory, ContentProcessorFactory e ComponentFactory

- Descrição: factories correlacionadas que registram e criam clients, processors e componentes de ingestão por tipo de conteúdo, com lazy loading e credenciais centralizadas.
- Tags: `ingestão`, `factory`, `pipeline`
- Tipo: factory
- Arquivo: `src/ingestion_layer/core/factories.py`
- Linguagem: Python
- Responsabilidade principal: impedir instância concreta e seleção de processor/client espalhadas pelo orchestrator.
- Dependências principais: BaseCorrelationFactory, CredentialManager, DataSourceFactory e VectorStoreFactory.
- Acoplamento forte com domínio: Médio; reutilização dentro da ingestão.
- Uso atual: Sim; consumida por orchestrator, adapters multimodais e testes de registro.
- Seguro reutilizar como está: Sim, para tipos registrados.
- Riscos ou limitações: `DisabledContentProcessor` é sentinela de falha lazy e não uma implementação funcional a escolher.
- Sugestão de melhoria: manter erro de import lazy observável e evitar fallbacks silenciosos.
- Prioridade: Alta.

### BaseDataSource e DataSourceFactory

- Descrição: abstração/factory para fontes filesystem, web, YouTube e Confluence, com contexto de correlação, autenticação e descoberta uniformes.
- Tags: `data-source`, `factory`, `ingestão`
- Tipo: classe e factory
- Arquivos: `src/ingestion_layer/datasources/base_data_source.py`, `src/ingestion_layer/datasources/data_source_factory.py`, `src/ingestion_layer/datasources/filesystem_data_source.py`, `src/ingestion_layer/datasources/web_data_source.py`, `src/ingestion_layer/datasources/youtube_data_source.py`, `src/ingestion_layer/datasources/confluence_data_source.py`.
- Linguagem: Python
- Responsabilidade principal: esconder transporte e origem concreta do restante da ingestão.
- Dependências principais: data models, CredentialManager, clients HTTP/cloud e logging de ingestão.
- Acoplamento forte com domínio: Médio; fontes de conteúdo.
- Uso atual: Sim; factory e bases são consumidas por clients, orchestrator e adapters.
- Seguro reutilizar como está: Sim; fonte nova deve herdar a base e entrar no factory.
- Riscos ou limitações: Confluence possui regras próprias de restrição/visibilidade que não devem migrar para a base.
- Sugestão de melhoria: manter suporte específico em módulos `confluence_*_support.py`.
- Prioridade: Alta.

### FileDiscoveryFactory, DiscoveryPatterns e BaseFileDiscovery

- Descrição: descoberta de arquivos por includes/excludes, limite e validação, com resultado tipado de encontrados, excluídos, escaneados e erros.
- Tags: `filesystem`, `discovery`, `validação`
- Tipo: factory, classe e modelo
- Arquivo: `src/ingestion_layer/core/file_discovery.py`
- Linguagem: Python
- Responsabilidade principal: reutilizar seleção de arquivos sem cada source aplicar glob e exclusões próprias.
- Dependências principais: filesystem/glob, BaseCorrelationComponent e logging de ingestão.
- Acoplamento forte com domínio: Médio; ingestão de arquivos.
- Uso atual: Sim; consumido por filesystem data source e pipelines locais, com testes dedicados.
- Seguro reutilizar como está: Sim, para diretórios explicitamente autorizados.
- Riscos ou limitações: usa mudança temporária de cwd e glob recursivo; `max_files` precisa ser definido para fontes amplas.
- Sugestão de melhoria: nenhuma indicada sem medição de problema de concorrência nesse fluxo.
- Prioridade: Alta.

### SidecarMetadataLoader, ChunkMetadataReducer e normalização de chunks

- Descrição: cadeia que lê sidecar de metadata, promove aliases/campos aninhados, remove blocos volumosos e normaliza payload de chunk antes do vector store.
- Tags: `metadados`, `normalização`, `vector-store`
- Tipo: service e helper
- Arquivos: `src/ingestion_layer/core/sidecar_metadata_loader.py`, `src/ingestion_layer/core/metadata_reducer.py`, `src/ingestion_layer/core/chunk_normalizer.py`.
- Linguagem: Python
- Responsabilidade principal: manter payload vetorial pequeno e com nomes estáveis entre processors distintos.
- Dependências principais: data models, logging de ingestão e catálogo de campos permitido.
- Acoplamento forte com domínio: Médio; metadata de ingestão/RAG.
- Uso atual: Sim; usado por file pipelines, Google Drive e upsert, com testes de redução/sidecar.
- Seguro reutilizar como está: Sim; somente campos do catálogo são preservados.
- Riscos ou limitações: `ChunkMetadataReducer` exige lista de tamanho idêntico ao catálogo padrão quando customizada; campo novo precisa de decisão explícita.
- Sugestão de melhoria: manter aliases documentados e cobertos por fixtures reais.
- Prioridade: Alta.

### ContentHasher, DuplicationManager e estratégias de upsert

- Descrição: componentes para gerar hash de conteúdo, identificar duplicação e escolher inserção/substituição/incremental sem lógica repetida em processors ou stores.
- Tags: `deduplicação`, `hash`, `upsert`
- Tipo: service, factory e helper
- Arquivos: `src/ingestion_layer/core/content_hash.py`, `src/ingestion_layer/core/duplication_manager.py`, `src/ingestion_layer/core/upsert_strategy.py`.
- Linguagem: Python
- Responsabilidade principal: padronizar identidade de conteúdo e decisão de persistência vetorial.
- Dependências principais: BaseDocument, vector store, config accessors e logging de ingestão.
- Acoplamento forte com domínio: Médio; ingestão vetorial.
- Uso atual: Sim; hash possui cerca de nove imports e estratégias são usadas pelo pipeline de persistência.
- Seguro reutilizar como está: Sim, dentro do contrato de ingestão.
- Riscos ou limitações: hash de conteúdo não substitui identidade/versionamento documental persistido.
- Sugestão de melhoria: manter identity/version helpers da telemetria como autoridade de versão.
- Prioridade: Alta.

### BaseVectorStore, BaseUnifiedStoreRetriever e DatasetLifecycleOrchestrator

- Descrição: base de vector store, contrato unificado de retrieval/config de busca e orquestrador de lifecycle de dataset sobre providers diferentes.
- Tags: `vector-store`, `retrieval`, `lifecycle`
- Tipo: classe, interface e service
- Arquivos: `src/ingestion_layer/vector_stores/base.py`, `src/ingestion_layer/vector_stores/retriever_contract.py`, `src/ingestion_layer/vector_stores/dataset_lifecycle_orchestrator.py`.
- Linguagem: Python
- Responsabilidade principal: oferecer operações comuns e um retriever consistente sem acoplar QA a Qdrant/Azure.
- Dependências principais: IVectorStore, BaseVectorStoreComponent, configs de busca e logging.
- Acoplamento forte com domínio: Médio; RAG/vector stores.
- Uso atual: Sim; implementado por Qdrant/Azure e consumido por factory, QA e lifecycle de datasets.
- Seguro reutilizar como está: Sim; provider novo deve implementar base/contract.
- Riscos ou limitações: capabilities dos backends diferem; config builder precisa normalizar limites e filtros.
- Sugestão de melhoria: manter diferenças de provider no adapter, não no consumidor.
- Prioridade: Alta.

### QdrantVectorStore e AzureCognitiveSearchVectorStore

- Descrição: adapters concretos observáveis para vector storage/retrieval em Qdrant e Azure Cognitive Search, ambos atrás das bases/factories canônicas.
- Tags: `qdrant`, `azure-search`, `adapter`
- Tipo: client
- Arquivos: `src/ingestion_layer/vector_stores/qdrant_client.py`, `src/ingestion_layer/vector_stores/azure_search_client.py`.
- Linguagem: Python
- Responsabilidade principal: traduzir contratos neutros para SDKs concretos e expor retrievers compatíveis.
- Dependências principais: SDK Qdrant/Azure, embeddings, retry e base vector store.
- Acoplamento forte com domínio: Médio; infraestrutura de retrieval.
- Uso atual: Sim; criados por `src/core/vector_store_factory.py` e exercitados por ingestão/QA/tests.
- Seguro reutilizar como está: Sim, exclusivamente via factory quando possível.
- Riscos ou limitações: instância direta pode ignorar alvo físico, tenant e environment.
- Sugestão de melhoria: manter capabilities/version diagnostics no adapter.
- Prioridade: Alta.

### CloudStorageClient e clients de storage de origem

- Descrição: abstrações para documentos e binários vindos de Google Drive, S3, MinIO e Azure Blob, com metadata de origem e credenciais centralizadas.
- Tags: `cloud-storage`, `client`, `ingestão`
- Tipo: client e classe-base
- Arquivos: `src/ingestion_layer/clients/cloud_storage_client.py`, `src/ingestion_layer/clients/google_drive_client.py`, `src/ingestion_layer/clients/s3_client.py`, `src/ingestion_layer/clients/minio_client.py`, `src/ingestion_layer/clients/azure_blob_client.py`.
- Linguagem: Python
- Responsabilidade principal: normalizar listagem/download/conversão de conteúdo remoto para documentos de ingestão.
- Dependências principais: provider SDKs, BaseDataSource, CredentialManager, sidecar loader e retry.
- Acoplamento forte com domínio: Médio; ingestão de storage remoto.
- Uso atual: Sim; `cloud_storage_client.py` tem aproximadamente 54 imports e clients concretos entram nas factories.
- Seguro reutilizar como está: Sim, dentro da ingestão e com credential manager.
- Riscos ou limitações: cada provider mantém paginação/metadata próprios; segredo não deve vir de env local no client.
- Sugestão de melhoria: manter operações comuns na base sem apagar diferenças de provider.
- Prioridade: Alta.

### StorageAssetService e ImageStorageClient

- Descrição: service correlacionado para persistir assets extraídos, consolidar métricas por backend/documento/página e devolver resultado tipado de upload.
- Tags: `storage`, `asset`, `imagem`
- Tipo: service e client
- Arquivos: `src/ingestion_layer/clients/storage_asset_service.py`, `src/ingestion_layer/clients/image_storage_client.py`.
- Linguagem: Python
- Responsabilidade principal: centralizar upload e contabilização de imagens/tabelas/assets sem lógica por processor.
- Dependências principais: storage clients, SourceType, storage constants e logging de ingestão.
- Acoplamento forte com domínio: Médio; assets da ingestão.
- Uso atual: Sim; usado por processors multimodais/web e possui cerca de nove imports diretos.
- Seguro reutilizar como está: Sim, para assets suportados.
- Riscos ou limitações: contextos/métricas esperam identidade documental; não é storage genérico para arquivos arbitrários do produto.
- Sugestão de melhoria: nenhuma indicada pelo código atual.
- Prioridade: Alta.

### MultimodalComponentFactory e contratos multimodais de ingestão

- Descrição: factory que combina extractors, OCR, descriptors e processor multimodal a partir do YAML, com helpers para web e Confluence.
- Tags: `multimodal`, `factory`, `ingestão`
- Tipo: factory e interface
- Arquivos: `src/ingestion_layer/multimodal/multimodal_factory.py`, `src/ingestion_layer/interfaces/multimodal_interfaces.py`.
- Linguagem: Python
- Responsabilidade principal: montar pipelines multimodais sem processors criarem engines/embeddings diretamente.
- Dependências principais: runtime multimodal neutro, storage de imagem, PDF engines e CredentialManager.
- Acoplamento forte com domínio: Médio; ingestão multimodal.
- Uso atual: Sim; interfaces têm aproximadamente 48 imports e factory é consumida por web/Confluence/adapters.
- Seguro reutilizar como está: Sim, pelos constructors publicados.
- Riscos ou limitações: config ausente pode desabilitar capacidade; não deve ativar OCR/vision fora do YAML.
- Sugestão de melhoria: manter resolução de config em um único snapshot.
- Prioridade: Alta.

### Contrato canônico de parsing PDF

- Descrição: dataclasses/protocol para bbox, blocos, seções, imagens, provenance, issues, tabelas e resultado de engine, mais normalização de capacidades efetivas.
- Tags: `pdf`, `parser`, `contrato`
- Tipo: modelo, interface e helper
- Arquivo: `src/ingestion_layer/pdf_tools/pdf_parsing_engine_contract.py`
- Linguagem: Python
- Responsabilidade principal: permitir composição/troca de Docling, Unstructured, PyMuPDF, custom e lego sem expor objetos de biblioteca ao downstream.
- Dependências principais: biblioteca padrão e enums de falha/capability do próprio módulo.
- Acoplamento forte com domínio: Médio; parsing PDF.
- Uso atual: Sim; aproximadamente 46 imports em engines, processors, runtime multimodal e testes.
- Seguro reutilizar como está: Sim; engine nova deve devolver `PdfParsingEngineResult` normalizado.
- Riscos ou limitações: modelos frozen/slots exigem evolução aditiva; resultado vazio não equivale a sucesso.
- Sugestão de melhoria: manter versão de contrato e capabilities sincronizadas.
- Prioridade: Alta.

### EngineSpec, PdfBaseOptionsParser e OrderedEngineSelector

- Descrição: parsers tipados para especificação YAML de engine e seletor ordenado que decide tentativas, triggers, skip e consolidação sem condicionais distribuídas.
- Tags: `pdf`, `yaml`, `selector`
- Tipo: parser e service
- Arquivos: `src/ingestion_layer/pdf_tools/pdf_engine_spec_parser.py`, `src/ingestion_layer/pdf_tools/pdf_base_options_parser.py`.
- Linguagem: Python
- Responsabilidade principal: transformar config declarativa em seleção determinística de engines.
- Dependências principais: contrato de parsing, logging de ingestão e policies de trigger.
- Acoplamento forte com domínio: Médio; parsing PDF.
- Uso atual: Sim; base options tem cerca de 11 imports e spec parser é usado por resolver/builder/engines.
- Seguro reutilizar como está: Sim, para a sintaxe comprovada no código.
- Riscos ou limitações: campo YAML ausente obrigatório falha; não criar fallback estrutural fora do parser.
- Sugestão de melhoria: manter mensagens de erro com `config_path` completo.
- Prioridade: Alta.

### Runtimes de engine PDF, circuit breaker e orçamento pesado

- Descrição: wrappers e policies para engines compostas, breaker por exaustão, orçamento de CPU/semaphore e lifecycle serializado por documento.
- Tags: `pdf`, `resiliência`, `concorrência`
- Tipo: service e helper
- Arquivos: `src/ingestion_layer/pdf_tools/deterministic_lego_pdf_parsing_engine.py`, `src/ingestion_layer/pdf_tools/breaker_aware_pdf_parsing_engine.py`, `src/ingestion_layer/pdf_tools/engine_circuit_breaker.py`, `src/ingestion_layer/pdf_tools/heavy_pdf_parse_budget.py`, `src/ingestion_layer/pdf_tools/serialized_native_pdf_runtime.py`, `src/ingestion_layer/pdf_tools/pdf_runtime_lifecycle.py`.
- Linguagem: Python
- Responsabilidade principal: coordenar engines pesadas sem exceder recursos nem insistir em falhas de exaustão conhecidas.
- Dependências principais: engine contract, clock/logging, env config e primitives de concorrência.
- Acoplamento forte com domínio: Médio; runtime PDF.
- Uso atual: Sim; breaker/budget/lego têm múltiplos consumers em OCR, processors e tests.
- Seguro reutilizar como está: Sim, dentro do pipeline PDF.
- Riscos ou limitações: estado de breaker em processo não é coordenação distribuída; orçamento é calculado por container/processo.
- Sugestão de melhoria: manter telemetria de decisão e capacidade junto das policies.
- Prioridade: Alta.

### Runner, codec e canal de subprocesso PDF

- Descrição: supervisor de subprocesso com cancelamento/timeout, codec de resultado e canal stdout isolado de logs para workers de parsing.
- Tags: `pdf`, `subprocesso`, `serialização`
- Tipo: runtime e helper
- Arquivos: `src/ingestion_layer/pdf_tools/pdf_subprocess_runner.py`, `src/ingestion_layer/pdf_tools/pdf_parsing_result_codec.py`, `src/ingestion_layer/pdf_tools/pdf_worker_stdout_channel.py`.
- Linguagem: Python
- Responsabilidade principal: evitar que cada engine replique spawn, IPC, terminate/kill, decode e separação stdout/stderr.
- Dependências principais: contrato PDF, JSON, subprocess e logging de ingestão.
- Acoplamento forte com domínio: Médio; workers PDF.
- Uso atual: Sim; consumido por Docling, PyMuPDF4LLM, custom e Unstructured workers.
- Seguro reutilizar como está: Sim, para worker que adote o protocolo publicado.
- Riscos ou limitações: `write_worker_result_json` exige launcher oficial; stdout fica reservado ao resultado.
- Sugestão de melhoria: manter qualquer pool de workers futuro sobre o mesmo codec/contrato.
- Prioridade: Alta.

### PdfParsingTool, registry de custom tools e MarkdownStructureExtractor

- Descrição: contrato/registry para tools determinísticas de parsing e helper que extrai seções/tabelas estruturadas de Markdown produzido por engines.
- Tags: `pdf`, `tool`, `markdown`
- Tipo: interface, factory e parser
- Arquivos: `src/ingestion_layer/pdf_tools/pdf_parsing_tool_contract.py`, `src/ingestion_layer/pdf_tools/custom_tools/custom_tool_registry.py`, `src/ingestion_layer/pdf_tools/markdown_structure_extractor.py`.
- Linguagem: Python
- Responsabilidade principal: compor receitas custom sem switch por tool e normalizar estrutura textual downstream.
- Dependências principais: engine contract, Fitz/pdfplumber tools e CSV/Markdown standard library.
- Acoplamento forte com domínio: Médio; parsing PDF.
- Uso atual: Sim; registry/contract são usados pela engine custom e extractor por engines estruturadas.
- Seguro reutilizar como está: Sim, dentro da recipe syntax implementada.
- Riscos ou limitações: tool não registrada falha explicitamente; parser Markdown depende da estrutura realmente produzida.
- Sugestão de melhoria: adicionar tool somente com implementação, capability e teste de recipe.
- Prioridade: Alta.

### UniversalJsonAnalyzer e contratos JSON avançados

- Descrição: analyzer extensível que detecta estrutura, tipos de campo, domínio, config e sugestões de consulta para JSON sem acoplar consumidores aos detectors concretos.
- Tags: `json`, `análise`, `factory`
- Tipo: service e interface
- Arquivos: `src/ingestion_layer/json_advanced/universal_analyzer.py`, `src/ingestion_layer/json_advanced/interfaces.py`, `src/ingestion_layer/json_advanced/field_type_detector.py`, `src/ingestion_layer/json_advanced/structure_detector.py`.
- Linguagem: Python
- Responsabilidade principal: normalizar entendimento de dados JSON em ingestão e QA.
- Dependências principais: contracts/detectors, BaseCorrelationComponent e logging.
- Acoplamento forte com domínio: Baixo; dados estruturados genéricos.
- Uso atual: Sim; interfaces têm cerca de 18 imports e adapters QA reusam a implementação canônica.
- Seguro reutilizar como está: Sim; adapters novos devem implementar os protocols.
- Riscos ou limitações: classificação heurística não é schema autoritativo; resultado deve conservar confidence/evidence.
- Sugestão de melhoria: manter adapters QA finos e evitar duplicar detector.
- Prioridade: Alta.

### SemanticQueryNormalizer, SemanticSchemaCatalogBuilder e TabularSemanticInterpreter

- Descrição: camada semântica genérica que adapta arrays JSON, constrói schema tabular, normaliza consulta e decide interpretação/clarificação com confidence policy.
- Tags: `semântica`, `tabular`, `parser`
- Tipo: service, parser e modelo
- Arquivos: `src/qa_layer/semantic_query/normalizer.py`, `src/qa_layer/semantic_query/schema_catalog_builder.py`, `src/qa_layer/semantic_query/tabular_semantic_engine.py`, `src/qa_layer/semantic_query/policy.py`, `src/qa_layer/semantic_query/json_tabular_adapter.py`.
- Linguagem: Python
- Responsabilidade principal: separar interpretação semântica da execução determinística de dados tabulares.
- Dependências principais: models/lexicon da mesma package e datasets JSON.
- Acoplamento forte com domínio: Não; lexicon é injetável.
- Uso atual: Sim; fachada tem aproximadamente 27 imports em QA/tabular e testes.
- Seguro reutilizar como está: Sim, com schema/lexicon explícitos.
- Riscos ou limitações: confidence baixa exige clarificação; não deve inventar coluna ausente.
- Sugestão de melhoria: manter decisões de clarificação na policy compartilhada.
- Prioridade: Alta.

### DeterministicTabularEngine e TabularQuestionParser

- Descrição: parser e engine que respondem filtros, agregações, ordenação e perguntas tabulares determinísticas sobre datasets normalizados, devolvendo resposta tipada.
- Tags: `tabular`, `consulta`, `determinístico`
- Tipo: parser e service
- Arquivo: `src/qa_layer/json_rag/tabular_deterministic_engine.py`
- Linguagem: Python
- Responsabilidade principal: resolver perguntas computáveis sem depender de resposta livre do LLM.
- Dependências principais: semantic query layer, JSON models e result factory.
- Acoplamento forte com domínio: Baixo; opera sobre schema/dataset fornecidos.
- Uso atual: Sim; consumido por JSON/Excel QA e possui suíte focada.
- Seguro reutilizar como está: Sim, para operações suportadas pelo parser.
- Riscos ou limitações: gramática é fechada; pergunta fora do contrato precisa seguir para clarificação/rota alternativa.
- Sugestão de melhoria: novas operações devem entrar com parser, evaluator e testes juntos.
- Prioridade: Alta.

### IngestionTelemetryFacade, ManifestManager e identidades documentais

- Descrição: fachada e helpers que registram progresso, manifest, source/document/version identity e payloads de persistência para reconstruir cada documento ingerido.
- Tags: `ingestão`, `telemetria`, `identidade`
- Tipo: service e helper
- Arquivos: `src/ingestion_layer/ingestion_telemetry_facade.py`, `src/ingestion_layer/telemetry/manifest_manager.py`, `src/ingestion_layer/telemetry/document_identity.py`, `src/ingestion_layer/telemetry/document_version_identity.py`, `src/ingestion_layer/telemetry/source_identity.py`, `src/ingestion_layer/telemetry/db_payload_builder.py`.
- Linguagem: Python
- Responsabilidade principal: padronizar fatos observáveis/persistíveis sem cada processor montar IDs ou payload SQL.
- Dependências principais: logging de ingestão, Postgres runtime, data models e JSON sanitizer.
- Acoplamento forte com domínio: Sim; telemetria da ingestão.
- Uso atual: Sim; manifest tem cerca de 12 imports e identities/facade são usados por file pipeline/persistence.
- Seguro reutilizar como está: Sim, no slice de ingestão.
- Riscos ou limitações: identidade documental, versão e vector key são conceitos distintos e não devem ser intercambiados.
- Sugestão de melhoria: manter cada fato com event/status canônicos.
- Prioridade: Alta.

### ContentQASystem e QARuntimeAssembly

- Descrição: fachada principal de perguntas/respostas e assembler que monta componentes modernos de retrieval, generation, memory, diagnostics e billing por request.
- Tags: `qa`, `rag`, `orquestração`
- Tipo: service e factory
- Arquivos: `src/qa_layer/content_qa_system.py`, `src/orchestrators/qa_runtime_assembly.py`.
- Linguagem: Python
- Responsabilidade principal: oferecer boundary único de Q&A sem routers montarem pipeline manualmente.
- Dependências principais: RAG factories/orchestrator, vector store, LLMFactory, PromptBuilder, memory e telemetry.
- Acoplamento forte com domínio: Médio; runtime Q&A transversal.
- Uso atual: Sim; ContentQASystem tem cerca de 11 imports e é usado por QuestionService/API/tests.
- Seguro reutilizar como está: Sim; criar via factory/assembler por request.
- Riscos ou limitações: runtime é stateless por request; guardar a instância como singleton funcional é incorreto.
- Sugestão de melhoria: manter wiring novo no assembler e não no router.
- Prioridade: Alta.

### ModernRAGChainFactory, builders e registry de chains

- Descrição: factory/registry que cria chains basic, streaming, fallback e parallel por interfaces e configuração moderna.
- Tags: `rag`, `factory`, `chain`
- Tipo: factory e registry
- Arquivos: `src/qa_layer/rag_engine/factories.py`, `src/qa_layer/rag_engine/chain_builders.py`, `src/qa_layer/rag_engine/interfaces.py`.
- Linguagem: Python
- Responsabilidade principal: desacoplar seleção/construção de chain do QA system e da ingestão.
- Dependências principais: LangChain, RAGConfig, retrievers, metrics e generation engine.
- Acoplamento forte com domínio: Médio; engine RAG.
- Uso atual: Sim; chamada pelo assembler/ContentQASystem e pelo wrapper de ingestão.
- Seguro reutilizar como está: Sim; novo builder deve implementar `IRAGChainBuilder` e registrar-se.
- Riscos ou limitações: chain type desconhecido deve falhar; wrapper duplicado fora da factory cria drift.
- Sugestão de melhoria: manter registry como única seleção de builder.
- Prioridade: Alta.

### QueryAnalyzer, AdaptiveQueryRouter e StrategyExecutorRegistry

- Descrição: cadeia que extrai features/tipo/domínio da pergunta, aplica regras de routing e executa a estratégia de retrieval registrada.
- Tags: `rag`, `roteamento`, `análise`
- Tipo: service, enum e registry
- Arquivos: `src/qa_layer/rag_engine/query_analyzer.py`, `src/qa_layer/rag_engine/adaptive_router.py`, `src/qa_layer/rag_engine/routing_decision_maker.py`, `src/qa_layer/rag_engine/strategy_executor_registry.py`.
- Linguagem: Python
- Responsabilidade principal: separar classificação, decisão e execução sem condicionais monolíticas no orchestrator.
- Dependências principais: RAGConfig, semantic query, logging e retrievers.
- Acoplamento forte com domínio: Médio; retrieval.
- Uso atual: Sim; query analyzer e adaptive router têm múltiplos consumers em orchestrator/tests.
- Seguro reutilizar como está: Sim, com perfis/regras declarados.
- Riscos ou limitações: descrições/regras sobrepostas tornam routing ambíguo; decisão precisa carregar evidência.
- Sugestão de melhoria: registrar estratégia nova no registry em vez de `if` local.
- Prioridade: Alta.

### Retrievers, HybridFusion e reranker

- Descrição: implementations para vector, cache, hybrid e multi-query retrieval, com fusão configurável e reescrita. O rerank NÃO mora mais aqui: desde 2026-08-14 é capacidade nativa do vector store (ColBERT late interaction server-side no Qdrant, dentro de `search_hybrid`), e o `reranker.py` com cross-encoder HuggingFace foi removido.
- Tags: `retrieval`, `hybrid-search`
- Tipo: service e strategy
- Arquivos: `src/qa_layer/rag_engine/retrievers.py`, `src/qa_layer/rag_engine/caching_retriever.py`, `src/qa_layer/rag_engine/hybrid_retriever.py`, `src/qa_layer/rag_engine/multi_query_retriever.py`, `src/qa_layer/rag_engine/fusion_algorithms.py`, `src/qa_layer/rag_engine/query_rewriter.py`.
- Linguagem: Python
- Responsabilidade principal: oferecer strategies intercambiáveis sobre contratos de retrieval.
- Dependências principais: vector store/retriever contract, embeddings/LLM, RAG config e metrics.
- Acoplamento forte com domínio: Médio; RAG.
- Uso atual: Sim; composto por IntelligentRAGOrchestrator e factories, com testes por estratégia.
- Seguro reutilizar como está: Sim, quando capability/config do provider for compatível.
- Riscos ou limitações: paralelismo/fusão sem thresholds corretos pode aumentar custo e latência; config deve vir do YAML.
- Sugestão de melhoria: manter métricas de decisão e latência por strategy.
- Prioridade: Alta.

### IntelligentRAGOrchestrator e RagQaLangGraphRuntime

- Descrição: orchestrator que coordena analysis, routing, retrieval, cache, fusion, generation e métricas, e adapter LangGraph usado pelo AG-UI RAG.
- Tags: `rag`, `langgraph`, `orquestração`
- Tipo: service e adapter
- Arquivos: `src/qa_layer/rag_engine/intelligent_orchestrator.py`, `src/qa_layer/rag_engine/rag_qa_langgraph_runtime.py`.
- Linguagem: Python
- Responsabilidade principal: executar o pipeline RAG moderno e expô-lo em grafo compatível com streaming/interrupções AG-UI.
- Dependências principais: analyzer/router/retrievers/cache/generation/metrics e LangGraph.
- Acoplamento forte com domínio: Médio; runtime RAG.
- Uso atual: Sim; orchestrator tem cerca de dez imports e runtime LangGraph é usado pelo adapter AG-UI.
- Seguro reutilizar como está: Sim, montado por request com dependências resolvidas.
- Riscos ou limitações: não deve manter estado de conversa somente em RAM; alterações concorrentes no arquivo durante esta análise pertencem ao worktree e não foram modificadas por este inventário.
- Sugestão de melhoria: manter branches de estratégia protegidos por testes de routing/wiring.
- Prioridade: Alta.

### SemanticCache, CacheEngine e backends Qdrant/Azure

- Descrição: porta/factory e implementations para cache semântico de respostas, com embeddings, thresholds, TTL e providers vetoriais.
- Tags: `cache-semântico`, `qdrant`, `azure-search`
- Tipo: service, interface e factory
- Arquivos: `src/qa_layer/rag_engine/semantic_cache.py`, `src/qa_layer/rag_engine/cache_engine.py`, `src/qa_layer/rag_engine/ports/semantic_cache_backend_port.py`, `src/qa_layer/rag_engine/semantic_cache_backend_factory.py`, `src/qa_layer/rag_engine/backends/qdrant_semantic_cache_backend.py`, `src/qa_layer/rag_engine/backends/azure_semantic_cache_backend.py`.
- Linguagem: Python
- Responsabilidade principal: reutilizar respostas semanticamente equivalentes sem acoplar orchestrator ao provider.
- Dependências principais: embeddings, vector clients, config e retry.
- Acoplamento forte com domínio: Médio; otimização RAG.
- Uso atual: Sim; composto pelo IntelligentRAGOrchestrator/CacheEngine e coberto por testes de backend.
- Seguro reutilizar como está: Sim, com namespace isolado e config explícita.
- Riscos ou limitações: cache não é memória autoritativa; threshold baixo pode servir resposta incorreta.
- Sugestão de melhoria: manter hit/miss e score observáveis por provider.
- Prioridade: Alta.

### UserMemoryManager, MemoryBackendFactory e PostgresStoreMemoryBackend

- Descrição: memória longa de usuário para Q&A sobre um port abstrato, com backend PostgreSQL/Store e namespace por ambiente/tenant.
- Tags: `memória`, `qa`, `postgresql`
- Tipo: service, interface e factory
- Arquivos: `src/qa_layer/memory/user_memory_manager.py`, `src/qa_layer/memory/backend_factory.py`, `src/qa_layer/memory/postgres_store_backend.py`.
- Linguagem: Python
- Responsabilidade principal: persistir/recuperar memória útil entre requests sem depender da RAM do QA system.
- Dependências principais: shared PostgresStore provider, config accessors, correlation e logging.
- Acoplamento forte com domínio: Médio; memória Q&A.
- Uso atual: Sim; consumido pelo QA setup/diagnostics e testes de persistência.
- Seguro reutilizar como está: Sim, no Q&A e com namespace oficial.
- Riscos ou limitações: não confundir com checkpointer de execução; ambos têm retenções/semânticas distintas.
- Sugestão de melhoria: manter backend novo atrás de `MemoryBackend`/factory.
- Prioridade: Alta.

### PromptBuilder, QAContextBuilder e diagnósticos de pipeline

- Descrição: helpers para compor prompt/contexto, analisar evidências e produzir diagnósticos de pipeline sem lógica duplicada no QuestionService.
- Tags: `prompt`, `qa`, `diagnóstico`
- Tipo: service e helper
- Arquivos: `src/qa_layer/prompt_builder.py`, `src/qa_layer/qa_context_builder.py`, `src/qa_layer/evidence_analyzer.py`, `src/qa_layer/pipeline_diagnostics_reporter.py`, `src/services/question/pipeline_diagnostics_builder.py`, `src/services/question/answer_quality_analyzer.py`.
- Linguagem: Python
- Responsabilidade principal: separar preparação de prompt/contexto e avaliação de qualidade/diagnóstico da orquestração.
- Dependências principais: prompts compartilhados, RAG results, telemetry e protocols QA.
- Acoplamento forte com domínio: Médio; Q&A.
- Uso atual: Sim; consumidos por ContentQASystem, QuestionService e testes unitários.
- Seguro reutilizar como está: Sim, no pipeline QA.
- Riscos ou limitações: diagnostics descrevem evidência disponível e não podem inventar causa ausente.
- Sugestão de melhoria: manter novos sinais de qualidade como campos tipados, não texto livre.
- Prioridade: Alta.

### Channel models, ChannelRegistry e ChannelQueueFactory

- Descrição: contratos multicanal tipados, registry de definição por tenant e factory de filas inline, Redis ou RabbitMQ.
- Tags: `canais`, `fila`, `registry`
- Tipo: modelo, service e factory
- Arquivos: `src/channel_layer/models.py`, `src/channel_layer/registry.py`, `src/channel_layer/queue.py`.
- Linguagem: Python
- Responsabilidade principal: padronizar mensagem inbound/outbound, execution/queue mode e escolha de transporte por canal.
- Dependências principais: Pydantic, ClientDirectory, Redis/RabbitMQ e namespaces de ambiente.
- Acoplamento forte com domínio: Médio; camada multicanal.
- Uso atual: Sim; models têm cerca de 63 imports e registry/queue são usados por processor, worker e routers.
- Seguro reutilizar como está: Sim, dentro de canais.
- Riscos ou limitações: `ChannelQueue` conhece `ChannelDefinition`; não é fila universal do produto.
- Sugestão de melhoria: canal novo deve entrar pelos models/registry/factory existentes.
- Prioridade: Alta.

### ChannelMessageProcessor, BaseChannelResponder e make_json_safe

- Descrição: runtime que resolve canal e execução, delega ao agent/workflow, integra HIL e normaliza resposta/serialização para providers distintos.
- Tags: `canais`, `adapter`, `serialização`
- Tipo: service, classe-base e helper
- Arquivos: `src/channel_layer/processor.py`, `src/channel_layer/responders/base.py`, `src/channel_layer/serialization_utils.py`, `src/channel_layer/execution_engine.py`.
- Linguagem: Python
- Responsabilidade principal: evitar processor/responder completo por provider e garantir payload JSON seguro.
- Dependências principais: ChannelRegistry/Queue, orchestrators, HIL bridge e logging channel.
- Acoplamento forte com domínio: Médio; canais.
- Uso atual: Sim; processor/router/worker e responders WhatsApp/Instagram usam estas abstrações.
- Seguro reutilizar como está: Sim, implementando responder/client específico.
- Riscos ou limitações: metadata específica do provider deve ficar no adapter; JSON sanitizer não substitui validação de schema.
- Sugestão de melhoria: novos canais devem estender bases, não duplicar processor.
- Prioridade: Alta.

### StartupPolicy, StartupOrchestrator e RuntimeBootstrap

- Descrição: política e orquestração de startup por papel de processo, com estado explícito, decisões, infraestrutura obrigatória, workers, scheduler e manutenção.
- Tags: `startup`, `runtime`, `policy`
- Tipo: service e configuração
- Arquivos: `src/api/startup/policy.py`, `src/api/startup/orchestrator.py`, `src/api/startup/runtime_bootstrap.py`, `src/api/startup/state_host.py`.
- Linguagem: Python
- Responsabilidade principal: centralizar o que API/worker/scheduler inicializam e encerram.
- Dependências principais: worker bootstrap, scheduler agenda, infrastructure guardrails, cache epoch, memory purge e logging.
- Acoplamento forte com domínio: Não; bootstrap de processo.
- Uso atual: Sim; chamado por `src/api/service_api.py` e runners, com testes de política/bootstrap.
- Seguro reutilizar como está: Sim; novo bootstrap deve entrar como etapa/policy, não side effect de import.
- Riscos ou limitações: estado global de processo é técnico; não deve guardar estado funcional de request.
- Sugestão de melhoria: manter decisões idempotentes e com cleanup simétrico.
- Prioridade: Alta.

### Guardrails e locks de infraestrutura no startup

- Descrição: probes tipados de PostgreSQL/Redis e lock Redis de startup que falham com diagnóstico seguro antes de ativar runtimes dependentes.
- Tags: `readiness`, `redis`, `postgresql`
- Tipo: validator e service
- Arquivos: `src/api/startup/infrastructure_guardrails.py`, `src/api/startup/locks.py`, `src/api/startup/decision_helpers.py`.
- Linguagem: Python
- Responsabilidade principal: validar contratos obrigatórios e coordenar boot sem lógica ad hoc nos runners.
- Dependências principais: DatabaseConnectionManager, Redis manager, system config e logging.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; composto por RuntimeBootstrap/StartupOrchestrator e coberto por testes de readiness.
- Seguro reutilizar como está: Sim, no startup oficial.
- Riscos ou limitações: probe não substitui health/retry durante operação; lock depende de namespace/TTL corretos.
- Sugestão de melhoria: manter mensagens sem DSN/secret e com metadata de host mascarada.
- Prioridade: Alta.

### Planos de fanout de documentos e ETL

- Descrição: modelos imutáveis e helpers de work key/codec que dividem uma requisição em itens independentes de documento ou pipeline ETL.
- Tags: `fanout`, `etl`, `ingestão`
- Tipo: modelo e helper
- Arquivos: `src/services/document_fanout_plan.py`, `src/services/etl_fanout_plan.py`.
- Linguagem: Python
- Responsabilidade principal: padronizar granularidade, identidade e payload de filhos antes da publicação no Job Core.
- Dependências principais: JSON/gzip/base64, identifiers e models de source/ETL.
- Acoplamento forte com domínio: Médio; fanout de ingestão/ETL.
- Uso atual: Sim; ambos têm vários imports em coordinators, processes e testes.
- Seguro reutilizar como está: Sim, nos respectivos fanouts.
- Riscos ou limitações: work key precisa ser determinística; codec de YAML não autoriza expor secrets.
- Sugestão de melhoria: manter o plano puro e deixar I/O para coordinator/executor.
- Prioridade: Alta.

### DocumentFanoutCoordinator e DocumentFanoutChildExecutorService

- Descrição: coordinator de pai e executor de filho com ports injetáveis para pipeline/persistência, mantendo lifecycle no Job Core e resultado funcional separado.
- Tags: `fanout`, `job-core`, `ingestão`
- Tipo: service
- Arquivos: `src/services/document_fanout_coordinator.py`, `src/services/document_fanout_child_executor_service.py`.
- Linguagem: Python
- Responsabilidade principal: executar paralelismo real por documento sem misturar store/queue no pipeline funcional.
- Dependências principais: fanout plan, Job Core context, IngestionService e result store.
- Acoplamento forte com domínio: Médio; ingestão.
- Uso atual: Sim; usado pelos processos de ingestão/fanout e testes de boundary.
- Seguro reutilizar como está: Sim, com dependências pelos protocols publicados.
- Riscos ou limitações: falha de persistência de resultado é distinta da falha de processamento e deve permanecer explícita.
- Sugestão de melhoria: manter side effects atrás dos ports.
- Prioridade: Alta.

### AG-UI run orchestrator, adapter registry e capability packs

- Descrição: backend AG-UI que prepara contexto de run, seleciona adapter RAG/DeepAgent/workflow por registry e publica capabilities via packs declarativos.
- Tags: `ag-ui`, `registry`, `orquestração`
- Tipo: service, registry e modelo
- Arquivos: `src/api/services/ag_ui_run_orchestrator.py`, `src/api/services/ag_ui_adapter_registry.py`, `src/api/services/ag_ui_capability_pack.py`, `src/api/services/ag_ui_capabilities_service.py`.
- Linguagem: Python
- Responsabilidade principal: impedir que router conheça runtimes concretos ou que cada adapter invente capabilities.
- Dependências principais: adapters AG-UI, event store, YAML preparation e schemas AG-UI.
- Acoplamento forte com domínio: Médio; protocolo AG-UI.
- Uso atual: Sim; router importa registry/orchestrator e adapters dependem de `AgUiRunContext`.
- Seguro reutilizar como está: Sim; execution kind novo deve registrar adapter/capability pack.
- Riscos ou limitações: registry ambíguo ou pack sem adapter correspondente falha o contrato.
- Sugestão de melhoria: manter capability IDs únicos e cobertos por testes de catálogo.
- Prioridade: Alta.

### AG-UI event encoder e event store

- Descrição: encoder de eventos canônicos/SSE e store de runs/eventos que preservam sequência, replay e correlação para o cliente oficial.
- Tags: `ag-ui`, `eventos`, `sse`
- Tipo: service e repository
- Arquivos: `src/api/services/ag_ui_event_encoder.py`, `src/api/services/ag_ui_event_store.py`.
- Linguagem: Python
- Responsabilidade principal: centralizar serialização e persistência de eventos AG-UI sem parser/protocolo paralelo.
- Dependências principais: schemas AG-UI, Job Core/PostgreSQL conforme store e logging.
- Acoplamento forte com domínio: Médio; AG-UI.
- Uso atual: Sim; router/workflow adapter/orchestrator e testes de replay usam estes componentes.
- Seguro reutilizar como está: Sim, no boundary AG-UI.
- Riscos ou limitações: ordem/IDs de evento fazem parte do protocolo; write manual pode quebrar replay.
- Sugestão de melhoria: manter formato alinhado ao SDK oficial.
- Prioridade: Alta.

### UiSpecValidator, registry YAML e materialização de UI governada

- Descrição: validador Python de `UiSpec`, registry de specs declaradas no YAML e materializador que liga somente dados aprovados a uma estrutura visual fixa.
- Tags: `ag-ui`, `validator`, `generative-ui`
- Tipo: validator, registry e service
- Arquivos: `src/api/schemas/ag_ui_ui_spec_models.py`, `src/api/services/ag_ui_yaml_ui_spec_registry.py`, `src/api/services/ag_ui_ui_spec_materialization.py`, `src/api/services/ag_ui_governed_ui_spec_service.py`.
- Linguagem: Python
- Responsabilidade principal: validar catálogo/componentes, resolver spec pelo target e produzir estado visual declarativo sem HTML gerado pelo agente.
- Dependências principais: Pydantic, run context, target resolver e adapters AG-UI.
- Acoplamento forte com domínio: Médio; UI governada AG-UI.
- Uso atual: Sim; `UiSpecValidator` é usado pelo validator de AST, registry e materializer, além de testes unitários.
- Seguro reutilizar como está: Sim; é o validador backend atual, apesar de o antigo catálogo ter citado incorretamente uma remoção total.
- Riscos ou limitações: ausência de um antigo validador JavaScript não torna o contrato Python morto; frontend deve renderizar apenas envelopes suportados.
- Sugestão de melhoria: manter validator, AST semantic validator e renderer sincronizados.
- Prioridade: Alta.

### AG-UI runtime adapter support e adapters RAG/DeepAgent/workflow

- Descrição: helpers e adapters que convertem runtimes oficiais em fluxo AG-UI, incluindo LangGraph agent factory, target resolution, HIL mapper e resume workflow.
- Tags: `ag-ui`, `adapter`, `langgraph`
- Tipo: adapter, factory e helper
- Arquivos: `src/api/services/ag_ui_runtime_adapter_support.py`, `src/api/services/ag_ui_langgraph_agent_factory.py`, `src/api/services/ag_ui_execution_target_resolver.py`, `src/api/services/ag_ui_hil_protocol_mapper.py`, `src/api/services/ag_ui_rag_qa_adapter.py`, `src/api/services/ag_ui_deepagent_adapter.py`, `src/api/services/ag_ui_workflow_adapter.py`, `src/api/services/workflow_ag_ui_resume_service.py`.
- Linguagem: Python
- Responsabilidade principal: compartilhar extração de YAML/metadata, interrupções e streaming sem duplicar código entre execution kinds.
- Dependências principais: LangGraph/AG-UI, runtimes QA/supervisor/workflow, HIL services e orchestrator.
- Acoplamento forte com domínio: Médio; adapter layer AG-UI.
- Uso atual: Sim; todos são registrados/composicionados pelo adapter registry/orchestrator.
- Seguro reutilizar como está: Sim; adapter novo deve usar o support comum.
- Riscos ou limitações: adapter não deve buscar dado para UI; materialização pertence ao service governado.
- Sugestão de melhoria: manter prompts de renderer curtos e specs fixos.
- Prioridade: Alta.

### Serviços HIL de aprovação, continuação e bridge de canal

- Descrição: cadeia persistente para criar/revisar/decidir aprovação humana, retomar agent/workflow/background e notificar/receber decisão por canais.
- Tags: `hil`, `aprovação`, `workflow`
- Tipo: service e adapter
- Arquivos: `src/api/services/hil_background_approval_service.py`, `src/api/services/hil_approval_decision_service.py`, `src/api/services/hil_approval_review_query_service.py`, `src/api/services/agent_hil_continuation_service.py`, `src/api/services/workflow_hil_continuation_service.py`, `src/api/services/hil_approval_channel_bridge.py`, `src/api/services/background_hil_continuation_submission_service.py`.
- Linguagem: Python
- Responsabilidade principal: centralizar estado/decisão HIL e continuação sem endpoint ou canal editar checkpointer diretamente.
- Dependências principais: approval repository/models, Job Core queue, agent/workflow runtimes, channels e logging.
- Acoplamento forte com domínio: Médio; HIL transversal a agents/workflows/channels.
- Uso atual: Sim; routers agent/channel, AG-UI adapters e processos assíncronos usam estes services.
- Seguro reutilizar como está: Sim, pelo service do caso correspondente.
- Riscos ou limitações: decisão idempotente e ownership precisam ser preservados; checkpoint de HIL pendente é intocável.
- Sugestão de melhoria: manter novos canais atrás da bridge.
- Prioridade: Alta.

### ChatConversationService, repositories e persistência assíncrona de turnos

- Descrição: API de conversa/mensagem com ownership, replay idempotente e publicação de persistência de turno como Job Core process.
- Tags: `chat`, `repository`, `async-job`
- Tipo: service, repository e publisher
- Arquivos: `src/chat/service.py`, `src/chat/repository.py`, `src/chat/models.py`, `src/chat/turn_persistence_publisher.py`, `src/chat/turn_persistence_execution.py`, `src/chat/turn_persistence_process.py`, `src/chat/boundary_scope.py`.
- Linguagem: Python
- Responsabilidade principal: reutilizar CRUD/hidratação/persistência de chats entre webchat, DNIT e hosts futuros.
- Dependências principais: ClientDirectoryBase, PostgresQueryExecutor, Job Core queue e correlation.
- Acoplamento forte com domínio: Médio; domínio chat, compartilhado por várias UIs.
- Uso atual: Sim; API/hosts e maintenance services usam a factory/service/repositories.
- Seguro reutilizar como está: Sim; boundaries devem resolver tenant/email pelos helpers do chat.
- Riscos ou limitações: ownership e replay conflict falham explicitamente; publicação degradada deve permanecer observável.
- Sugestão de melhoria: manter host frontend sem persistência paralela quando backend sessions estão habilitadas.
- Prioridade: Alta.

### IntegrationRegistryRepository e modelos de integração

- Descrição: modelos tipados e repositories para grupos, Swagger sources, auth profiles, operações API, conexões SQL, queries, procedures e tools builtin.
- Tags: `integração`, `repository`, `catálogo`
- Tipo: modelo e repository
- Arquivos: `src/integrations/models.py`, `src/integrations/repository.py`, `src/integrations/builtin_tool_repository.py`.
- Linguagem: Python
- Responsabilidade principal: manter um catálogo administrativo compartilhado por UI, ToolsFactory e importadores.
- Dependências principais: Pydantic, ClientDirectoryBase e PostgresQueryExecutor.
- Acoplamento forte com domínio: Médio; bounded context de integrações.
- Uso atual: Sim; models/schema/repository têm múltiplos imports em admin, tools e tests.
- Seguro reutilizar como está: Sim, para integrações suportadas.
- Riscos ou limitações: connection secret não pertence ao record público; publicação e storage têm políticas próprias.
- Sugestão de melhoria: manter record novo validado no model e exposto pelo mesmo repository.
- Prioridade: Alta.

### SwaggerImportApplicationService e SqlReadOnlyGuardrail

- Descrição: importador HTTP de OpenAPI/Swagger com retry/parsing estruturado e validador SQL que recusa statements não somente-leitura antes de publicar tools dinâmicas.
- Tags: `openapi`, `sql`, `validação`
- Tipo: service e validator
- Arquivos: `src/integrations/swagger_import_service.py`, `src/integrations/sql_read_only_guardrail.py`.
- Linguagem: Python
- Responsabilidade principal: padronizar importação de API e proteger queries publicáveis contra escrita/DDL.
- Dependências principais: HTTPX, retry, integration models/repository e parser SQL do módulo.
- Acoplamento forte com domínio: Médio; integrações administrativas.
- Uso atual: Sim; consumidos por admin services, dynamic SQL tools e testes de guardrail/import.
- Seguro reutilizar como está: Sim, nos boundaries de integração.
- Riscos ou limitações: guardrail é validação sintática/política e não substitui credencial read-only no banco.
- Sugestão de melhoria: manter novos dialects cobertos por fixtures adversariais.
- Prioridade: Alta.

### TransactionalEmailApplicationService e providers Brevo/Resend

- Descrição: boundary funcional para email transacional com providers HTTP concretos, retry, timeout, logs e resultado estruturado compartilhado por auth e tools.
- Tags: `email`, `http-client`, `provider`
- Tipo: service e client
- Arquivos: `src/services/transactional_email_service.py`, `src/services/brevo_transactional_email_client.py`, `src/services/resend_transactional_email_client.py`.
- Linguagem: Python
- Responsabilidade principal: evitar que routers/tools conheçam autenticação e payload dos providers.
- Dependências principais: HTTPX, ExternalRetry, logging e settings.
- Acoplamento forte com domínio: Baixo; capacidade transversal de envio.
- Uso atual: Sim; auth router e factories Brevo/Resend usam o application service.
- Seguro reutilizar como está: Sim, pela camada de aplicação.
- Riscos ou limitações: provider configurado não possui fallback implícito; `correlation_id` e sender válidos são obrigatórios.
- Sugestão de melhoria: evoluir HTML/templates/anexos primeiro no contrato interno.
- Prioridade: Alta.

### SmtpEmailApplicationService e SMTPMailClient

- Descrição: boundary SMTP com request/result tipados, normalização de destinatários, retry e client técnico sobre `smtplib`.
- Tags: `smtp`, `email`, `client`
- Tipo: service e client
- Arquivo: `src/services/smtp_email_service.py`
- Linguagem: Python
- Responsabilidade principal: oferecer envio SMTP sem detalhes de MIME, TLS e autenticação nos consumidores.
- Dependências principais: smtplib, security keys, ExternalRetry e logging.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; notification service HIL e tool SMTP usam o application service.
- Seguro reutilizar como está: Sim, pela factory/service público.
- Riscos ou limitações: configuração inválida falha; client baixo nível não deve ser instanciado no endpoint.
- Sugestão de melhoria: manter classificação de falha transitória no client.
- Prioridade: Alta.

### PythonTestSuiteCatalog, Runner, StateStore e plugin pytest

- Descrição: runtime tipado da suíte oficial que resolve modos/targets, exige `run_id`, executa, persiste state/telemetry e coleta markers explícitos.
- Tags: `testes`, `runner`, `telemetria`
- Tipo: service, configuração e plugin
- Arquivos: `suite_de_testes_padrao.py`, `src/python_test_suite/catalog.py`, `src/python_test_suite/runner.py`, `src/python_test_suite/state.py`, `src/python_test_suite/pytest_plugin.py`, `src/python_test_suite/models.py`.
- Linguagem: Python
- Responsabilidade principal: ser a única fonte de planos e artefatos da suíte Python.
- Dependências principais: pytest, subprocess, filesystem em `.sandbox/tmp` e models tipados.
- Acoplamento forte com domínio: Não; infraestrutura de qualidade.
- Uso atual: Sim; entrypoint oficial, CI/docs e testes da própria suíte.
- Seguro reutilizar como está: Sim; modo/target novo deve entrar no catálogo.
- Riscos ou limitações: exit code isolado não prova verde; state/report precisam ser lidos.
- Sugestão de melhoria: manter marker family explícito e ratchets dentro do runner/plugin.
- Prioridade: Alta.

### SuiteCheckupReader e gates auxiliares da suíte

- Descrição: leitores/gates reutilizáveis para inspecionar rodada, detectar nodeids duplicados e aplicar ratchet de mypy sem scripts one-shot.
- Tags: `testes`, `checkup`, `gate`
- Tipo: service e validator
- Arquivos: `src/python_test_suite/checkup.py`, `src/python_test_suite/duplication_gate.py`, `src/python_test_suite/mypy_ratchet_gate.py`.
- Linguagem: Python
- Responsabilidade principal: validar artefatos e invariantes da suíte por APIs tipadas.
- Dependências principais: state store/models e reports pytest/mypy.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; CLI e testes unitários da suite usam os componentes.
- Seguro reutilizar como está: Sim, sobre um `run_id` existente.
- Riscos ou limitações: falha fechado com artefato ausente/corrompido; não deve criar segunda telemetria.
- Sugestão de melhoria: manter payload de checkup pequeno e estável.
- Prioridade: Alta.

### Helpers gerais de environment, URL, path, JSON e identificadores

- Descrição: funções puras para ler/coagir env, parsear URLs MySQL/Redis, resolver paths do projeto, sanitizar JSON, normalizar identifiers e combinar dicts.
- Tags: `utilitário`, `parser`, `normalização`
- Tipo: helper
- Arquivos: `src/utils/env.py`, `src/utils/database_url_parser.py`, `src/utils/path_resolver.py`, `src/core/json_sanitizer.py`, `src/core/identifier_utils.py`, `src/utils/dict_merge.py`, `src/utils/variant_field_sanitizer.py`.
- Linguagem: Python
- Responsabilidade principal: evitar parsing/coerção/normalização local repetida em config, tools e pipelines.
- Dependências principais: biblioteca padrão.
- Acoplamento forte com domínio: Não; alguns sanitizers têm vocabulário de payload.
- Uso atual: Sim; env tem cerca de 22 imports, JSON sanitizer 31 e identifier utils nove.
- Seguro reutilizar como está: Sim, escolhendo o helper adequado.
- Riscos ou limitações: sanitização não substitui validação de domínio; path resolver não autoriza saída do escopo permitido.
- Sugestão de melhoria: promover helper novo somente após duplicação comprovada.
- Prioridade: Alta.

### Agentic utils

- Descrição: helpers para validar `user_session`, autenticação/config e construir chamadas comuns de toolkits agentic de múltiplos vendors.
- Tags: `agentic`, `validação`, `toolkit`
- Tipo: helper
- Arquivo: `src/utils/agentic_utils.py`
- Linguagem: Python
- Responsabilidade principal: impedir que cada vendor tool replique leitura de sessão, headers e campos obrigatórios.
- Dependências principais: YAML/session, HTTP helpers e logging.
- Acoplamento forte com domínio: Médio; tools agentic.
- Uso atual: Sim; aproximadamente 31 imports em toolkits ecommerce, food, sports, finance e outros.
- Seguro reutilizar como está: Sim, em factories/tools do catálogo.
- Riscos ou limitações: não substitui CredentialManager ou permission enforcement do boundary.
- Sugestão de melhoria: manter mensagens de validação sem secret.
- Prioridade: Alta.

## Prioridade alta — runtimes e componentes compartilhados JavaScript

### PrometeuAdminApiClient

- Descrição: cliente HTTP administrativo que centraliza base URL, access key, email, headers, JSON, timeout, erros, `correlation_id` e payload offline para dezenas de páginas.
- Tags: `frontend`, `http-client`, `admin`
- Tipo: client
- Arquivo: `app/ui/static/js/shared/admin-api-client.js`
- Linguagem: JavaScript
- Responsabilidade principal: impedir `fetch` e tratamento de autenticação/erro/correlação próprios em cada tela admin.
- Dependências principais: AdminLayoutBridge/WorkspaceShell quando presentes, storage e Fetch API.
- Acoplamento forte com domínio: Médio; transversal ao frontend administrativo.
- Uso atual: Sim; consumido por ingestão, ETL, scheduler, users, schema metadata, integrations e outras páginas; possui testes de contrato de contexto/correlação.
- Seguro reutilizar como está: Sim, em páginas admin sob o mesmo contrato.
- Riscos ou limitações: fallback de `X-API-Key` exige contexto válido; não deve ser usado para criar payload de domínio sem client específico.
- Sugestão de melhoria: manter normalização de `PrometeuApiError`/correlation em uma única implementação.
- Prioridade: Alta.

### PrometeuAdminUtils

- Descrição: coleção compartilhada de escaping/sanitização, coerção booleana, masking, formatação, extração de erros e montagem de URL para páginas administrativas.
- Tags: `frontend`, `sanitização`, `formatter`
- Tipo: helper
- Arquivo: `app/ui/static/js/shared/admin-utils.js`
- Linguagem: JavaScript
- Responsabilidade principal: remover utilitários DOM/texto repetidos das telas admin.
- Dependências principais: APIs padrão do browser.
- Acoplamento forte com domínio: Baixo; utilidades de UI administrativa.
- Uso atual: Sim; referenciado por dezenas de scripts e carregado por múltiplos HTML admin.
- Seguro reutilizar como está: Sim, escolhendo a função publicada.
- Riscos ou limitações: escaping textual não substitui DOM seguro nem validação backend.
- Sugestão de melhoria: promover somente helpers com múltiplos consumidores reais.
- Prioridade: Alta.

### PrometeuPermissionCatalogPicker

- Descrição: seletor compartilhado do catálogo de permissões de máquina. Busca o catálogo no backend (`GET /api/auth/admin/permission-catalog`), filtra o que aceita credencial técnica, desenha os cards descritivos e o checklist marcável (plano ou agrupado por família) e lê de volta o que ficou marcado.
- Tags: `frontend`, `permissões`, `admin`
- Tipo: componente
- Arquivo: `app/ui/static/js/shared/permission-catalog-picker.js`
- Linguagem: JavaScript
- Responsabilidade principal: existir uma única implementação do "como desenhar e ler uma lista de permissões" nas telas admin — o catálogo em si nunca é duplicado em JavaScript, porque a fonte única é `PERMISSION_CATALOG` no backend.
- Dependências principais: `PrometeuAdminApiClient.adminFetchJson` (único caminho HTTP; `fetch` cru é proibido), `PrometeuAdminUtils.escapeHtml` e o CSS `permission-catalog-card` / `permission-toggle-card` de `admin-access-governance.css`.
- Acoplamento forte com domínio: Baixo; é parametrizado por container e não registra listener próprio — quem consome decide o efeito do `change`.
- Uso atual: Sim; consumido por `admin-users.js` (tela de Credenciais) e por `admin-gov-permissoes-api-key.js` (wizard de permissões de chave). Coberto por `tests/frontend/permission_catalog_picker_contract.test.js`.
- Seguro reutilizar como está: Sim. Opções: `readOnlyKeys` desabilita chaves não editáveis, `noteByKey` acrescenta a explicação de origem por chave e `grouped` agrupa por família em blocos recolhíveis (o catálogo passa de 70 permissões; em lista plana a tela vira uma parede de milhares de pixels).
- Riscos ou limitações: `readSelection` lê o DOM, então checkbox desabilitada ou escondida por filtro **não** é reportada — quem filtra precisa preservar a marcação fora do DOM, sob pena de desmarcar em silêncio. O componente renderiza e lê; ele **não** grava, e não deve ser acoplado a nenhum fluxo de gravação.
- Sugestão de melhoria: ligar `grouped` também na tela de Credenciais, que hoje sofre a mesma lista plana.
- Prioridade: Alta.

### PrometeuInspectableDataGrid

- Descrição: família global para montar grids de dados virtuais com seleção neutra, inspetor lateral tipado, edição explícita, validação, rollback e preferências visuais isoladas, sem expor o Tabulator nem acoplar o componente a uma regra de negócio. A facade compõe os globals `PrometeuDataGridAdapter`, `PrometeuCellInspector`, `PrometeuInspectorSplitter`, `PrometeuValueTypeDetector` e `PrometeuGridSettingsStore` atrás de `PrometeuInspectableDataGrid.create(options)`.
- Tags: `frontend`, `data-grid`, `inspeção`
- Tipo: componente e facade
- Arquivos: `app/ui/static/js/shared/inspectable-data-grid.js`, `app/ui/static/js/shared/tabulator-grid-adapter.js`, `app/ui/static/js/shared/cell-inspector.js`, `app/ui/static/js/shared/inspector-splitter.js`, `app/ui/static/js/shared/value-type-detector.js`, `app/ui/static/js/shared/grid-settings-store.js`, `app/ui/static/css/inspectable-data-grid.css`.
- Linguagem: JavaScript
- Responsabilidade principal: compor uma API pública independente do provider para dados/colunas dinâmicos, estados loading/empty/error, seleção de linha e célula, inspeção completa de valores primitivos/JSON, edição somente quando autorizada, redimensionamento acessível e persistência exclusiva de preferências visuais.
- Dependências principais: Tabulator local `6.5.2` carregado antes do adapter, `createJSONEditor` de `vanilla-jsoneditor` local `3.13.0` injetado pela página ESM, APIs DOM/Pointer Events e `localStorage` ou storage compatível injetado.
- Acoplamento forte com domínio: Não; colunas, registros, identidade da linha, validators, formatters e callbacks entram pelo contrato neutro da facade.
- Uso atual: Potencial; o consumo funcional comprovado está apenas em `app/ui/static/ui-inspectable-data-grid-demo.html` e `app/ui/static/js/inspectable-data-grid-demo.js`, com proteção Jest em `tests/frontend/inspectable_data_grid_core_contract.test.js`, `tests/frontend/inspectable_data_grid_vendor_contract.test.js`, `tests/frontend/inspectable_data_grid_adapter_contract.test.js`, `tests/frontend/inspectable_data_grid_inspector_contract.test.js`, `tests/frontend/inspectable_data_grid_composition_contract.test.js` e `tests/frontend/inspectable_data_grid_demo_contract.test.js`, além de `tests/playwright/test_08-01-11_inspectable_data_grid_demo.py`; nenhuma tela real do produto consome a família nesta rodada.
- Seguro reutilizar como está: Sim, dentro do contrato comprovado pelos testes de core, vendor, adapter, inspetor, composição, demo e browser; o host deve respeitar a ordem de carregamento, injetar `createJSONEditor` e assumir explicitamente a persistência de alterações aceita pelo callback.
- Riscos ou limitações: os scripts clássicos e o provider precisam existir antes da facade, o editor JSON precisa ser injetado por ESM, a família não persiste alterações de células nem chama backend e nenhuma tela real do produto a consome nesta rodada; portanto, aprovação local pelo callback não prova persistência externa.
- Sugestão de melhoria: validar a primeira adoção em uma tela real sem ampliar a API pública nem criar adapter, inspetor, store ou fluxo de edição paralelo.
- Prioridade: Alta.

#### Exemplo mínimo comprovado

Carregue os estilos e scripts clássicos nesta ordem. O módulo ESM da página vem por último, pois é ele que importa e injeta `createJSONEditor`:

```html
<link rel="stylesheet" href="/ui/static/css/vendor/tabulator.min.css">
<link rel="stylesheet" href="/ui/static/css/inspectable-data-grid.css">

<div id="inspectable-grid-demo-host"></div>

<script src="/ui/static/js/vendor/tabulator.min.js"></script>
<script src="/ui/static/js/shared/value-type-detector.js"></script>
<script src="/ui/static/js/shared/grid-settings-store.js"></script>
<script src="/ui/static/js/shared/tabulator-grid-adapter.js"></script>
<script src="/ui/static/js/shared/cell-inspector.js"></script>
<script src="/ui/static/js/shared/inspector-splitter.js"></script>
<script src="/ui/static/js/shared/inspectable-data-grid.js"></script>
<script type="module" src="/ui/static/js/inspectable-data-grid-demo.js"></script>
```

No arquivo ESM da página, importe a factory local e passe-a explicitamente para `create`. O callback abaixo aceita a alteração apenas em memória e guarda somente o identificador necessário para um rollback posterior; ele não chama backend nem persiste o conteúdo editado:

```javascript
import { createJSONEditor } from '/ui/static/js/vendor/vanilla-jsoneditor.js';

let lastChangeId = null;
const grid = window.PrometeuInspectableDataGrid.create({
    settingsKey: 'demo.inspectable-data-grid.v1',
    rowIdField: 'id',
    data: [{ id: 1, textoCurto: 'Registro fictício' }],
    columns: [
        { field: 'id', title: 'ID', type: 'number', editable: false },
        { field: 'textoCurto', title: 'Texto curto', type: 'string', editable: true },
    ],
    editable: true,
    jsonEditorFactory: createJSONEditor,
    onCellChange: async function rememberAcceptedChange(event) {
        lastChangeId = event.changeId;
    },
});

const host = document.querySelector('#inspectable-grid-demo-host');
if (!(host instanceof HTMLElement)) {
    throw new Error('Host do grid não encontrado.');
}
grid.mount(host);
```

O fluxo de edição é único: selecionar a célula abre o inspetor; `Aplicar` valida o rascunho e a coluna, atualiza o dado local e chama `onCellChange`; se o callback lançar/rejeitar, a facade restaura automaticamente o valor anterior. Se o callback resolver, o host pode usar depois `grid.revertChange(lastChangeId)` para desfazer a última mudança ainda válida; `Cancelar` apenas descarta o rascunho. O `settingsKey` deve ser estável e exclusivo de cada grid porque separa ordem/largura/visibilidade de colunas, filtros, sorters e estado do inspetor. Nunca grave linhas, seleção ou conteúdo de célula nesse storage: `PrometeuGridSettingsStore` mantém somente a allowlist visual. A implementação completa e executável está em `app/ui/static/ui-inspectable-data-grid-demo.html` e `app/ui/static/js/inspectable-data-grid-demo.js`.

### PrometeuLayoutMestreContextRuntime

- Descrição: runtime global que centraliza contexto de usuário, credenciais, storage snapshot, regras same-origin e publicação do evento `prometeu:credenciais-alteradas`.
- Tags: `frontend`, `contexto`, `evento`
- Tipo: runtime
- Arquivo: `app/ui/static/js/shared/layout-mestre.js`
- Linguagem: JavaScript
- Responsabilidade principal: ser a fonte de contexto para Layout Mestre e bridges administrativos, sem leitura de storage dispersa.
- Dependências principais: local/session storage, YAML extractor, Alpine e APIs do browser.
- Acoplamento forte com domínio: Médio; shell/layout da plataforma.
- Uso atual: Sim; publica `window.PrometeuLayoutMestreContextRuntime`, `window.prometeuLayoutMestre` e evento consumido por bridge/runners; carregado por dezenas de páginas.
- Seguro reutilizar como está: Sim, respeitando a ordem de scripts.
- Riscos ou limitações: credenciais/tokens não podem ir ao console; consumidores devem reagir ao evento em vez de guardar cópia stale.
- Sugestão de melhoria: manter novas leituras de contexto dentro da API publicada.
- Prioridade: Alta.

### PrometeuAdminLayoutBridge

- Descrição: facade UMD sobre o runtime do Layout Mestre que resolve access key, email, snapshot, bind/clear e traduz mudanças de credenciais para consumers admin.
- Tags: `frontend`, `bridge`, `credenciais`
- Tipo: helper e facade
- Arquivo: `app/ui/static/js/shared/admin-layout-bridge.js`
- Linguagem: JavaScript
- Responsabilidade principal: desacoplar telas administrativas dos detalhes internos do layout/storage.
- Dependências principais: `PrometeuLayoutMestreContextRuntime` e evento `prometeu:credenciais-alteradas`.
- Acoplamento forte com domínio: Médio; frontend admin.
- Uso atual: Sim; consumido por users, scheduler, vector preview, tenant secrets, integrations, offline keys, logs central e outros; coberto por testes de bridge/contexto.
- Seguro reutilizar como está: Sim; é o acesso padrão ao contexto admin.
- Riscos ou limitações: fallback manual fora da bridge pode divergir do runtime e perder atualização de credenciais.
- Sugestão de melhoria: manter toda compatibilidade de storage dentro da bridge.
- Prioridade: Alta.

### resolveAdminLayoutAccessKey e buildAdminLayoutStorageSnapshot

- Descrição: helpers ESM puros que resolvem access key e snapshot de storage para runners administrativos que não consomem diretamente o global UMD.
- Tags: `frontend`, `contexto`, `esm`
- Tipo: helper
- Arquivo: `app/ui/static/js/shared/admin-layout-context-runtime.js`
- Linguagem: JavaScript
- Responsabilidade principal: oferecer a mesma regra de contexto em módulos de ingestão/ETL/PDF.
- Dependências principais: objeto de runtime/storage injetado.
- Acoplamento forte com domínio: Médio; runners admin.
- Uso atual: Sim; importado por `admin-ingestao.js`, `admin-etl.js` e `admin-ingestao-pdf.js`, com teste de contrato.
- Seguro reutilizar como está: Sim, em módulos ESM do shell admin.
- Riscos ou limitações: snapshot é leitura do estado atual; o módulo consumidor deve assinar o evento para mudanças posteriores.
- Sugestão de melhoria: nenhuma indicada pelo código atual.
- Prioridade: Alta.

### PrometeuLayoutMestreApi e PrometeuApiError

- Descrição: client/facade do Layout Mestre para payload criptografado, endpoints QA/agent/workflow, polling, HIL e extração consistente de erro/correlação.
- Tags: `frontend`, `api`, `layout-mestre`
- Tipo: client e classe
- Arquivo: `app/ui/static/js/shared/layout-mestre-api.js`
- Linguagem: JavaScript
- Responsabilidade principal: fornecer transporte compartilhado ao webchat e telas de análise sem protocolo paralelo.
- Dependências principais: PayloadCrypto/LocalCrypto conforme host, Fetch API e correlation runtime.
- Acoplamento forte com domínio: Médio; protocolo da plataforma.
- Uso atual: Sim; publica `window.prometeuLayoutMestreApi`, `window.PrometeuLayoutMestreApi` e é usado por webchat, análise de interações e SQL natural.
- Seguro reutilizar como está: Sim, dentro dos modos/endpoints suportados.
- Riscos ou limitações: não é client HTTP genérico; host não deve montar criptografia/envelope próprio.
- Sugestão de melhoria: manter contratos de execução/poll/HIL alinhados ao EmbeddableChatRuntime.
- Prioridade: Alta.

### PrometeuAdminCorrelationRuntime, AuthGateRuntime e ActionFeedbackRuntime

- Descrição: shell administrativo que publica runtimes globais para exibir correlação, bloquear ações sem sessão/contexto e padronizar feedback de operações.
- Tags: `frontend`, `shell`, `autenticação`
- Tipo: runtime
- Arquivo: `app/ui/static/js/shared/admin-workspace-shell.js`
- Linguagem: JavaScript
- Responsabilidade principal: evitar que cada página admin implemente banner de correlation, auth gate e toast/status próprios.
- Dependências principais: Layout bridge, DOM e eventos do browser.
- Acoplamento forte com domínio: Médio; shell administrativo.
- Uso atual: Sim; carregado por cerca de cinquenta páginas e consumido por AdminApiClient, LayoutMestreApi, webchat utils e CorrelationSurface.
- Seguro reutilizar como está: Sim, em páginas dentro do workspace admin.
- Riscos ou limitações: IDs/data attributes esperados no HTML fazem parte do contrato; ordem de bootstrap importa.
- Sugestão de melhoria: manter testes de contrato para eventos/IDs públicos.
- Prioridade: Alta.

### PrometeuAreaNavigationCatalog e PrometeuAdminHomeCards

- Descrição: catálogos declarativos compartilhados para navegação entre áreas e composição dos cards da home administrativa.
- Tags: `frontend`, `catálogo`, `navegação`
- Tipo: constante e helper
- Arquivos: `app/ui/static/js/shared/admin-area-navigation-catalog.js`, `app/ui/static/js/shared/admin-home-cards.js`.
- Linguagem: JavaScript
- Responsabilidade principal: manter títulos, rotas, grupos e cards fora de HTMLs duplicados.
- Dependências principais: DOM e contratos de áreas admin.
- Acoplamento forte com domínio: Médio; navegação do produto.
- Uso atual: Sim; catálogo é carregado por dezenas de HTMLs e home cards por múltiplas páginas/contratos.
- Seguro reutilizar como está: Sim, para novas superfícies admin aderentes ao catálogo.
- Riscos ou limitações: rota adicionada sem página/permissão real vira navegação quebrada.
- Sugestão de melhoria: manter catálogo, permission metadata e testes de links sincronizados.
- Prioridade: Alta.

### PrometeuGlobalHeader

- Descrição: runtime global do header, sessão federada, estado read-only e menu do usuário compartilhado por páginas da plataforma.
- Tags: `frontend`, `header`, `sessão`
- Tipo: runtime
- Arquivo: `app/ui/static/js/shared/global-area-header.js`
- Linguagem: JavaScript
- Responsabilidade principal: renderizar e atualizar o cabeçalho sem markup/auth duplicados em cada HTML.
- Dependências principais: federated session endpoints, Admin/Prometeu API, Alpine confirm e DOM.
- Acoplamento forte com domínio: Médio; chrome global da UI.
- Uso atual: Sim; carregado por mais de sessenta páginas e protegido por contratos/browser tests.
- Seguro reutilizar como está: Sim, carregado na ordem prevista pelo shell.
- Riscos ou limitações: flag federada e sessão precisam refletir o backend; dados sensíveis não podem aparecer no DOM/log.
- Sugestão de melhoria: manter variações via opções/atributos, não por cópia do header.
- Prioridade: Alta.

### Alpine components globais

- Descrição: componentes/funções compartilhados de access key, cache de YAML, API, confirmação e modal de mensagem para páginas Alpine.
- Tags: `frontend`, `alpine`, `modal`
- Tipo: helper e componente
- Arquivo: `app/ui/static/js/shared/alpine-components.js`
- Linguagem: JavaScript
- Responsabilidade principal: publicar `prometeuConfirmar`, `prometeuConfirm`, `prometeuAccessKey`, `prometeuYamlCache`, `prometeuApi` e mensagem via `prometeu:message`.
- Dependências principais: Alpine.js, DOM, storage e CustomEvent.
- Acoplamento forte com domínio: Baixo; primitives de UI.
- Uso atual: Sim; carregado por dezenas de páginas e `prometeuConfirmar` é chamado por header/ações administrativas.
- Seguro reutilizar como está: Sim, em hosts Alpine que carreguem o script antes do componente.
- Riscos ou limitações: nomes/IDs globais formam contrato; duplicar modal ou listener causa ações repetidas.
- Sugestão de melhoria: manter um único dialog global por documento.
- Prioridade: Alta.

### Admin runner request core

- Descrição: runtime de request para runners que resolve base URL/contexto, valida preflight e prepara payload criptografado sem repetir lógica de autenticação.
- Tags: `frontend`, `runner`, `request`
- Tipo: factory e helper
- Arquivo: `app/ui/static/js/shared/admin-runner-request-core.js`
- Linguagem: JavaScript
- Responsabilidade principal: expor `createAdminRunnerApiBaseRuntime` e primitives de request usadas por ingestão e ETL.
- Dependências principais: AdminApiClient, Layout context e crypto/payload APIs.
- Acoplamento forte com domínio: Médio; runners admin.
- Uso atual: Sim; consumido por `admin-ingestao.js` e `admin-etl.js`.
- Seguro reutilizar como está: Sim, para runner com o mesmo boundary.
- Riscos ou limitações: domínio ainda precisa construir seu payload específico; storage fallback não substitui bridge.
- Sugestão de melhoria: manter preflight e encryption concentrados aqui.
- Prioridade: Alta.

### Admin runner core

- Descrição: núcleo compartilhado de status, polling, cancelamento, timers e consolidação de execução para runners administrativos.
- Tags: `frontend`, `polling`, `cancelamento`
- Tipo: helper
- Arquivo: `app/ui/static/js/shared/admin-runner-core.js`
- Linguagem: JavaScript
- Responsabilidade principal: impedir loops de polling/cancel/state divergentes entre ingestão, ETL e PDF.
- Dependências principais: timers, AbortController e callbacks/clients injetados.
- Acoplamento forte com domínio: Médio; runners de job admin.
- Uso atual: Sim; importado por runners de ingestão, ETL e PDF, com testes de contrato.
- Seguro reutilizar como está: Sim, quando status/cancel callbacks cumprem o contrato.
- Riscos ou limitações: payload de status específico precisa ser normalizado antes; polling sem terminal state pode continuar indefinidamente se o caller errar.
- Sugestão de melhoria: manter estados terminais em uma única lista publicada.
- Prioridade: Alta.

### createIngestionExecutionClient

- Descrição: client especializado que dispara ingestão administrativa sobre o request core e normaliza resposta/correlação do endpoint.
- Tags: `frontend`, `ingestão`, `client`
- Tipo: client e factory
- Arquivo: `app/ui/static/js/shared/ingestion-execution-client.js`
- Linguagem: JavaScript
- Responsabilidade principal: separar contrato de execução de ingestão da página e do core genérico.
- Dependências principais: Admin runner request core e AdminApiClient.
- Acoplamento forte com domínio: Sim; ingestão administrativa.
- Uso atual: Sim; consumido por `app/ui/static/js/admin-ingestao.js` e testes de correlação.
- Seguro reutilizar como está: Sim, em novas telas de ingestão.
- Riscos ou limitações: não é client de job universal; conhece payload/endpoint de ingestão.
- Sugestão de melhoria: manter mudanças de endpoint/payload neste client.
- Prioridade: Alta.

### Clientes e sessão de monitor de ingestão

- Descrição: factories que consultam runs/documentos, mantêm cursor de log e entregam uma sessão de monitoramento com estado de polling reutilizável.
- Tags: `ingestão`, `monitoramento`, `client`
- Tipo: client e service
- Arquivos: `app/ui/static/js/shared/ingestion-runs-dashboard-client.js`, `app/ui/static/js/shared/ingestion-log-monitor-session.js`, `app/ui/static/js/shared/ingestion-runs-dashboard-api.js`.
- Linguagem: JavaScript
- Responsabilidade principal: esconder endpoints e controle incremental de logs das telas de dashboard.
- Dependências principais: AdminApiClient, runner core e timers.
- Acoplamento forte com domínio: Sim; monitor operacional de ingestão.
- Uso atual: Sim; consumido pelas telas de ingestão geral/PDF e testes do dashboard.
- Seguro reutilizar como está: Sim, em dashboards de ingestão.
- Riscos ou limitações: offset/correlation selecionados precisam acompanhar o run ativo.
- Sugestão de melhoria: manter cursor e cancellation dentro da sessão.
- Prioridade: Alta.

### Cadeia State → Render → Polling → LogViewer do dashboard de ingestão

- Descrição: módulos UMD que separam estado, renderização DOM, polling e visualização de logs do dashboard de runs, compostos pelo controller de seleção.
- Tags: `frontend`, `ingestão`, `dashboard`
- Tipo: store, renderer, service e controller
- Arquivos: `app/ui/static/js/shared/ingestion-runs-dashboard-state.js`, `app/ui/static/js/shared/ingestion-runs-dashboard-render.js`, `app/ui/static/js/shared/ingestion-runs-dashboard-polling.js`, `app/ui/static/js/shared/ingestion-runs-dashboard-log-viewer.js`, `app/ui/static/js/shared/ingestion-selected-run-controller.js`.
- Linguagem: JavaScript
- Responsabilidade principal: fornecer componentes composáveis do dashboard em vez de uma god function por página.
- Dependências principais: dashboard clients/API, live payload parser, DOM e timers.
- Acoplamento forte com domínio: Sim; runs de ingestão.
- Uso atual: Sim; carregado/composto por dashboards de ingestão e protegido por testes de contrato frontend.
- Seguro reutilizar como está: Sim, preservando a ordem de scripts/factories globais.
- Riscos ou limitações: IDs/data attributes do HTML são parte do adapter de DOM; state não é persistência.
- Sugestão de melhoria: manter parsing de payload fora de render/polling.
- Prioridade: Alta.

### ingestion-live-payload-parser

- Descrição: funções puras que normalizam run, filhos, documentos, progresso, status e evidência de paralelismo do payload operacional vivo.
- Tags: `ingestão`, `parser`, `normalização`
- Tipo: parser e helper
- Arquivo: `app/ui/static/js/shared/ingestion-live-payload-parser.js`
- Linguagem: JavaScript
- Responsabilidade principal: proteger renderers do shape cru e de campos ausentes/inconsistentes.
- Dependências principais: JavaScript padrão.
- Acoplamento forte com domínio: Sim; telemetria de ingestão.
- Uso atual: Sim; consumido por dashboards de ingestão/PDF e testes unitários frontend, corrigindo o status “Potencial” do catálogo anterior.
- Seguro reutilizar como está: Sim, para o payload operacional atual.
- Riscos ou limitações: mudanças backend de naming exigem atualizar parser e contrato juntos.
- Sugestão de melhoria: manter funções puras e fixtures de payload real.
- Prioridade: Alta.

### PrometeuEditorsAPI

- Descrição: facade global para montar/destruir CodeMirror, ler/escrever conteúdo e renderizar Markdown em páginas que precisam de editor compartilhado.
- Tags: `frontend`, `editor`, `codemirror`
- Tipo: facade e factory
- Arquivo: `app/ui/static/js/codemirror-setup.js`
- Linguagem: JavaScript
- Responsabilidade principal: impedir setup/configuração CodeMirror duplicados em studios/admin.
- Dependências principais: CodeMirror, DOM e renderer Markdown configurado.
- Acoplamento forte com domínio: Baixo; editor genérico da UI.
- Uso atual: Sim; consumido por Assembly AST e detalhe DNIT, com contratos frontend.
- Seguro reutilizar como está: Sim, quando esta implementação é carregada como fonte única.
- Riscos ou limitações: `app/ui/static/js/yaml-editor.js` também publica `PrometeuEditorsAPI`; carregar ambos pode sobrescrever o contrato conforme ordem de scripts.
- Sugestão de melhoria: convergir as duas publicações para uma única facade oficial.
- Prioridade: Alta.

### PrometeuYamlExtractor

- Descrição: parser leve compartilhado para extrair access key, email e modo de YAML/payload textual na UX do browser.
- Tags: `yaml`, `parser`, `credenciais`
- Tipo: helper
- Arquivo: `app/ui/static/js/shared/yaml-access-key-extractor.js`
- Linguagem: JavaScript
- Responsabilidade principal: evitar regex/parsing local em páginas que precisam apenas preparar contexto visual/request.
- Dependências principais: JavaScript padrão.
- Acoplamento forte com domínio: Médio; UI YAML-first.
- Uso atual: Sim; consumido por ingestão, ETL, offline keys, cache, Assembly, Layout e WebChat.
- Seguro reutilizar como está: Sim, apenas para UX/local preparation.
- Riscos ou limitações: não substitui `resolve_yaml_configuration` no backend e não torna secret seguro para exibição/log.
- Sugestão de melhoria: manter extrações novas no mesmo parser somente se houver múltiplos consumers.
- Prioridade: Alta.

### PrometeuUiLogDownloads

- Descrição: helper canônico de frontend para montar URLs e acionar downloads de logs correlacionados sem endpoints/nomes de arquivo próprios em cada página.
- Tags: `logging`, `frontend`, `download`
- Tipo: helper
- Arquivo: `app/ui/static/js/shared/ui-log-downloads.js`
- Linguagem: JavaScript
- Responsabilidade principal: padronizar downloads administrativos de log pelo provider/backend oficial.
- Dependências principais: AdminApiClient/Layout context e DOM.
- Acoplamento forte com domínio: Baixo; observabilidade UI.
- Uso atual: Sim; consumido por integrações e várias telas admin com ações de log.
- Seguro reutilizar como está: Sim, em superfícies autorizadas.
- Riscos ou limitações: não deve construir path local nem acessar storage remoto diretamente.
- Sugestão de melhoria: manter qualquer novo formato de download na API compartilhada.
- Prioridade: Alta.

### PrometeuCorrelationSurface

- Descrição: componente leve para superfícies fora do workspace exibirem e atualizarem a correlação oficial de requests.
- Tags: `correlação`, `frontend`, `componente`
- Tipo: helper e componente
- Arquivo: `app/ui/static/js/shared/correlation-surface.js`
- Linguagem: JavaScript
- Responsabilidade principal: reutilizar apresentação/cópia de correlation em auth gateway, account center e client portal.
- Dependências principais: AdminCorrelationRuntime quando disponível e DOM.
- Acoplamento forte com domínio: Não.
- Uso atual: Sim; consumido por três scripts/hosts externos ao shell e protegido por teste estrutural.
- Seguro reutilizar como está: Sim, fora do workspace.
- Riscos ou limitações: não cria nem corrige correlação; somente exibe a devolvida pelo backend.
- Sugestão de melhoria: nenhuma indicada pelo código atual.
- Prioridade: Alta.

### WebchatRuntimeUtils e WebchatAsyncRuntime

- Descrição: helpers para montar/interpretar execução, status, correlation, respostas e Markdown seguro, mais polling/cancelamento/retry assíncrono compartilhado.
- Tags: `webchat`, `polling`, `normalização`
- Tipo: helper e runtime
- Arquivos: `app/ui/static/js/shared/ui-webchat-runtime-utils.js`, `app/ui/static/js/shared/ui-webchat-async-runtime.js`.
- Linguagem: JavaScript
- Responsabilidade principal: manter semântica de chat fora dos hosts; os utilitários de
  resposta são usados pelo componente, enquanto o AsyncRuntime atende consumidores clássicos
  que realmente fazem polling.
- Dependências principais: Fetch/AbortController, DOM/Markdown sanitizer e LayoutMestreApi.
- Acoplamento forte com domínio: Médio; chat da plataforma.
- Uso atual: Sim; `WebchatRuntimeUtils` é consumido por webchat v3,
  EmbeddableChatRuntime, Gesdoc e Layout API. `WebchatAsyncRuntime` não é chamado pelo
  componente embutível atual.
- Seguro reutilizar como está: Sim, para os modos QA/agent/deepagent/workflow suportados.
- Riscos ou limitações: sanitização de erro/secret deve continuar fail-closed; polling precisa propagar correlation oficial.
- Sugestão de melhoria: manter novos mappers de resposta nesta fonte única.
- Prioridade: Alta.

### HilContract, HilReviewPanel e WebchatBackgroundAlert

- Descrição: contrato de normalização/build de payload HIL, painel DOM reutilizável de revisão/decisão e alerta de pendências em background.
- Tags: `hil`, `frontend`, `componente`
- Tipo: helper e componente
- Arquivos: `app/ui/static/js/shared/ui-webchat-hil-contract.js`, `app/ui/static/js/shared/hil-review-panel.js`, `app/ui/static/js/shared/ui-webchat-background-alert.js`.
- Linguagem: JavaScript
- Responsabilidade principal: evitar mappers/painéis de aprovação humana diferentes entre chat, sidecar e página de aprovação.
- Dependências principais: DOM, WebchatRuntimeUtils e API/resume callbacks injetados.
- Acoplamento forte com domínio: Médio; HIL transversal ao chat/AG-UI.
- Uso atual: Sim; usado por EmbeddableChatRuntime, sidecar, Layout API e approval page, com testes de contrato.
- Seguro reutilizar como está: Sim, com callbacks oficiais.
- Riscos ou limitações: payload inválido falha fechado; uma pendência deve bloquear novo envio até decisão.
- Sugestão de melhoria: manter tipos de decisão alinhados aos modelos backend.
- Prioridade: Alta.

### PrometeuEmbeddableChatRuntime

- Descrição: componente oficial que renderiza o próprio DOM e encapsula mensagens, payload,
  ramo HTTP clássico `direct_sync`, streaming SSE opt-in, HIL, cancelamento, Markdown
  seguro, correlation, hidratação, actions e sessões headless.
- Tags: `chat`, `runtime`, `componente`
- Tipo: runtime e factory
- Arquivo: `app/ui/static/js/shared/embeddable-chat-runtime.js`
- Linguagem: JavaScript
- Responsabilidade principal: publicar `window.EmbeddableChatRuntime.createGenericEmbeddableChat` e o alias de sessão como fonte única da experiência conversacional.
- Dependências principais: LayoutMestreApi, WebchatRuntimeUtils, HilContract/ReviewPanel e
  bridges opcionais AG-UI/spec. Não depende do AsyncRuntime para polling.
- Acoplamento forte com domínio: Baixo; chat configurável sobre YAML já carregado pelo host.
- Uso atual: Sim; usado por webchat v3, detalhe DNIT/Gesdoc, SaaS e bancada, com testes de
  contrato. O Playwright isolado cobre o ramo clássico interceptado; não prova SSE API-live.
- Seguro reutilizar como está: Sim; é o default obrigatório para nova área de conversa.
- Riscos ou limitações: host deve fornecer uma origem de configuração autorizada
  (YAML/payload ou `projectKey`) e e-mail, carregar dependências antes e persistir a lista
  de sessões. O gate SSE exige `jspuro` + flag e `qa`, `deepagent` ou `projectKey`; não há
  callback público `onToken`. Depois do primeiro fragmento, o cancelamento integral do
  stream não é garantido pela API pública atual.
- Sugestão de melhoria: evoluir este runtime quando surgir requisito transversal, sem criar chat paralelo.
- Prioridade: Alta.

### PrometeuChatSessionStore

- Descrição: store do host para CRUD, hidratação e persistência local/backend de múltiplas sessões de chat, separado do componente headless.
- Tags: `chat`, `sessão`, `storage`
- Tipo: service e store
- Arquivo: `app/ui/static/js/shared/chat-session-store.js`
- Linguagem: JavaScript
- Responsabilidade principal: impedir que hosts repitam mappers, nomes de chave e merge de mensagens.
- Dependências principais: localStorage ou API backend injetada, relógio do browser e EmbeddableChatRuntime.
- Acoplamento forte com domínio: Médio; hosts de chat.
- Uso atual: Sim; consumido por webchat v3 e detalhe Gesdoc, com testes de sessão
  local/backend.
- Seguro reutilizar como está: Sim; componente deve ser montado antes da leitura/hidratação.
- Riscos ou limitações: store local é por browser e não substitui backend em uso multi-dispositivo; credenciais/YAML não devem ser persistidos.
- Sugestão de melhoria: manter backend/local como adapters do mesmo contrato.
- Prioridade: Alta.

### EmbeddableChatAgUiTransport e bridge global

- Descrição: transport que executa run AG-UI oficial a partir do chat embutível e bridge que publica `PrometeuEmbeddableChatAgUiTransport` ao runtime clássico.
- Tags: `ag-ui`, `chat`, `bridge`
- Tipo: client e bridge
- Arquivos: `app/ui/static/js/shared/embeddable-chat-ag-ui-transport.js`, `app/ui/static/js/shared/ag-ui-embeddable-transport-bridge.js`.
- Linguagem: JavaScript
- Responsabilidade principal: ligar o componente UMD ao client AG-UI ESM sem parser SSE
  artesanal; acumular texto numa única mensagem e preservar fontes Q&A, HIL, A2UI,
  `threadId` e `X-Correlation-Id`.
- Dependências principais: `@prometeu/ag-ui-runtime`, AG-UI client/state store e EmbeddableChatRuntime.
- Acoplamento forte com domínio: Médio; chat AG-UI.
- Uso atual: Sim; carregado pelos hosts AG-UI/chat e protegido por contratos de bridge/transport.
- Seguro reutilizar como está: Sim, para `qa`, `deepagent` ou `projectKey` sob o gate
  opt-in do componente.
- Riscos ou limitações: replay/retry implícito é desligado por padrão; bridge deve ser carregada antes de ativar transporte.
- Sugestão de melhoria: manter transformação de evento no SDK/client oficial.
- Prioridade: Alta.

### EmbeddableChatSpecRuntime e spec render bridge

- Descrição: runtime que detecta envelopes estruturados em respostas do chat e bridge que compõe safe content, component catalog, capabilities e A2UI renderer.
- Tags: `generative-ui`, `chat`, `renderer`
- Tipo: runtime e bridge
- Arquivos: `app/ui/static/js/shared/embeddable-chat-spec-runtime.js`, `app/ui/static/js/shared/ag-ui-spec-render-bridge.js`.
- Linguagem: JavaScript
- Responsabilidade principal: renderizar somente specs reconhecidas e cair para texto seguro quando ausentes/inválidas.
- Dependências principais: AG-UI safe content, event renderer, A2UI renderer e component catalog.
- Acoplamento forte com domínio: Médio; renderização AG-UI no chat.
- Uso atual: Sim; publicado como `PrometeuEmbeddableChatSpecRuntime` e consumido pelo EmbeddableChatRuntime/hosts.
- Seguro reutilizar como está: Sim, com dependências carregadas.
- Riscos ou limitações: ordem de scripts é contrato; nunca renderizar HTML vindo do agente.
- Sugestão de melhoria: manter catálogo fechado e fallback textual.
- Prioridade: Alta.

### Fachada @prometeu/ag-ui-runtime e client oficial

- Descrição: pacote ESM interno que reexporta client/store/renderers/sidecar e client HTTP
  sobre `@ag-ui/client`/RxJS para streaming oficial, headers, correlation e cancelamento.
- Tags: `ag-ui`, `sdk`, `http-client`
- Tipo: facade e client
- Arquivos: `packages/ag-ui-runtime/index.js`, `packages/ag-ui-runtime/official-http-client.js`, `packages/ag-ui-runtime/browser-entry.js`.
- Linguagem: JavaScript
- Responsabilidade principal: ser a entrada interna estável para consumidores module/browser
  do protocolo AG-UI. Para parceiro externo, o caminho é `@ag-ui/client` + BFF.
- Dependências principais: `@ag-ui/client`, RxJS e helpers Prometeu.
- Acoplamento forte com domínio: Médio; runtime AG-UI da plataforma.
- Uso atual: Sim; reexportado no pacote/browser e consumido pelo bridge/client compartilhado.
- Seguro reutilizar como está: Sim; preferir `createPrometeuAgUiOfficialClient`.
- Riscos ou limitações: qualquer export vira contrato público; retry default é zero para não repetir run automaticamente.
- Sugestão de melhoria: manter changelog de exports e versões do SDK.
- Prioridade: Alta.

### Helpers Prometeu do runtime AG-UI

- Descrição: funções para endpoint, headers, correlation, tenant, capabilities, frontend tools e catálogo, incluindo guard explícito de APIs SSE removidas.
- Tags: `ag-ui`, `helper`, `protocolo`
- Tipo: helper e constante
- Arquivo: `packages/ag-ui-runtime/prometeu-helpers.js`
- Linguagem: JavaScript
- Responsabilidade principal: concentrar detalhes específicos da plataforma sobre o SDK oficial.
- Dependências principais: Web APIs e contratos AG-UI.
- Acoplamento forte com domínio: Médio; protocolo Prometeu AG-UI.
- Uso atual: Sim; consumido por official client e reexportado pela fachada/browser.
- Seguro reutilizar como está: Sim, dentro do pacote/runtime.
- Riscos ou limitações: `assertPrometeuAgUiRuntimeApiAvailable` falha de propósito para parsers SSE antigos; não recriar as APIs removidas.
- Sugestão de melhoria: manter discovery/capability helpers puros.
- Prioridade: Alta.

### createAgUiStateStore e contrato de interrupção

- Descrição: store canônico de mensagens, estado, tools, patches e interrupções, acompanhado de normalizador para eventos de interrupt.
- Tags: `ag-ui`, `state-store`, `interrupção`
- Tipo: store e helper
- Arquivos: `app/ui/static/js/shared/ag-ui-state-store.js`, `app/ui/static/js/shared/ag-ui-interrupt-contract.js`.
- Linguagem: JavaScript
- Responsabilidade principal: manter aplicação de eventos AG-UI determinística entre sidecar, chat e renderers.
- Dependências principais: eventos oficiais AG-UI e JavaScript padrão.
- Acoplamento forte com domínio: Médio; AG-UI.
- Uso atual: Sim; consumido pelo sidecar/client/bridges e reexportado pelo pacote.
- Seguro reutilizar como está: Sim, em surfaces AG-UI.
- Riscos ou limitações: shape de evento/interrupt é contrato; mutar state fora das actions pode quebrar replay.
- Sugestão de melhoria: manter reducers cobertos para eventos fora de ordem.
- Prioridade: Alta.

### Renderers, timeline, safe content e catálogo AG-UI

- Descrição: componentes que renderizam eventos/tools, acumulam timeline, recusam conteúdo inseguro e resolvem componentes por catálogo/registry fechado.
- Tags: `ag-ui`, `renderer`, `segurança`
- Tipo: renderer, helper e registry
- Arquivos: `app/ui/static/js/shared/ag-ui-event-renderer.js`, `app/ui/static/js/shared/ag-ui-tool-timeline.js`, `app/ui/static/js/shared/ag-ui-safe-content.js`, `app/ui/static/js/shared/ag-ui-component-catalog.js`.
- Linguagem: JavaScript
- Responsabilidade principal: separar dados/eventos da criação segura de DOM.
- Dependências principais: DOM e contrato AG-UI.
- Acoplamento forte com domínio: Médio; UI agentic.
- Uso atual: Sim; usados por state/sidecar/spec bridge/A2UI e testes de segurança/render.
- Seguro reutilizar como está: Sim, no catálogo suportado.
- Riscos ou limitações: chave/string proibida invalida spec; timeline não deve inferir resultado ausente.
- Sugestão de melhoria: manter novos componentes registrados e testados antes de publicação.
- Prioridade: Alta.

### AgUiChartAdapter, ApexChartsAdapter e A2UI renderer

- Descrição: porta neutra de gráficos, implementação ApexCharts e renderer fail-closed do envelope A2UI para Card/Column/Row/Text/Divider/BarChart/LineChart/DataTable.
- Tags: `ag-ui`, `a2ui`, `gráfico`
- Tipo: interface, adapter e renderer
- Arquivos: `app/ui/static/js/shared/ag-ui-chart-adapter.js`, `app/ui/static/js/shared/ag-ui-chart-adapter-apexcharts.js`, `app/ui/static/js/shared/ag-ui-a2ui-surface-renderer.js`.
- Linguagem: JavaScript
- Responsabilidade principal: impedir dependência direta de ApexCharts no renderer e transformar spec fixa em DOM seguro.
- Dependências principais: safe content, event renderer, DOM e ApexCharts somente no adapter concreto.
- Acoplamento forte com domínio: Médio; generative UI, sem domínio de negócio específico.
- Uso atual: Sim; A2UI é composto pelo spec bridge e hosts; contratos cobrem desacoplamento e catálogo.
- Seguro reutilizar como está: Sim, para o envelope/catalog atual.
- Riscos ou limitações: componente desconhecido invalida a árvore; ampliar catálogo exige renderer/adapter/testes.
- Sugestão de melhoria: manter o agente produzindo spec/dados, nunca HTML.
- Prioridade: Alta.

### createAgUiSidecarChat e AgUiGovernedDemoPageController

- Descrição: sidecar reutilizável sobre client/store/renderers e controller de páginas demo governadas com variante especializada de varejo.
- Tags: `ag-ui`, `sidecar`, `controller`
- Tipo: component e service
- Arquivos: `app/ui/static/js/shared/ag-ui-sidecar-chat.js`, `app/ui/static/js/shared/ag-ui-retail-demo-page.js`.
- Linguagem: JavaScript
- Responsabilidade principal: montar chat lateral e wiring de demo sem remontar runtime AG-UI em cada página.
- Dependências principais: client AG-UI, state store, interrupt contract e renderers.
- Acoplamento forte com domínio: Médio; controller base é genérico, variante de varejo é específica.
- Uso atual: Sim; sidecar é reexportado e demo controller sustenta múltiplas páginas AG-UI.
- Seguro reutilizar como está: Sim; domínio novo deve especializar controller sem copiar o runtime.
- Riscos ou limitações: demos não são boundary de produção; capability pack/adapter precisam existir no backend.
- Sugestão de melhoria: manter dados de demonstração fora do controller base.
- Prioridade: Alta.

### PrometeuAdminIntegracoesApi

- Descrição: facade de frontend para catálogo/import/actions de integrações, composta com AdminApiClient, LayoutBridge, feedback e downloads de log.
- Tags: `frontend`, `integração`, `api`
- Tipo: client e facade
- Arquivo: `app/ui/static/js/shared/admin-integracoes-api.js`
- Linguagem: JavaScript
- Responsabilidade principal: compartilhar requests e tratamento entre páginas de grupos, APIs, SQL e tools builtin.
- Dependências principais: AdminApiClient, AdminLayoutBridge, ActionFeedbackRuntime e UiLogDownloads.
- Acoplamento forte com domínio: Médio; admin de integrações.
- Uso atual: Sim; consumido por quatro páginas da família de integrações e testes de contrato.
- Seguro reutilizar como está: Sim, dentro desse bounded context.
- Riscos ou limitações: não é client admin universal; conhece endpoints/models de integração.
- Sugestão de melhoria: manter novos endpoints desta família na facade.
- Prioridade: Alta.

## Prioridade média — reuso contextual ou limitado

### Schema metadata readers, factory, writer e ingestor

- Descrição: pipeline para ler schemas PostgreSQL/SQL Server/Oracle, normalizar tabelas/colunas/PK/FK e persistir catálogo de metadados para Schema-RAG.
- Tags: `schema`, `banco-de-dados`, `factory`
- Tipo: service, factory e client
- Arquivos: `src/schema_metadata/base_reader.py`, `src/schema_metadata/reader_factory.py`, `src/schema_metadata/readers.py`, `src/schema_metadata/writer.py`, `src/schema_metadata/ingestor.py`, `src/schema_metadata/config.py`.
- Linguagem: Python
- Responsabilidade principal: abstrair dialect/driver e padronizar metadata estrutural.
- Dependências principais: drivers PostgreSQL/MSSQL/Oracle, DatabaseConnectionManager e config tipada.
- Acoplamento forte com domínio: Médio; Schema-RAG/admin metadata.
- Uso atual: Sim; fachada `src.schema_metadata` e admin service/CLI usam o pipeline.
- Seguro reutilizar como está: Sim, para fontes suportadas e schema destino provisionado.
- Riscos ou limitações: writer não autoriza DDL no runtime; tipos específicos de dialect precisam de reader próprio.
- Sugestão de melhoria: provider novo deve implementar `BaseMetadataReader` e factory.
- Prioridade: Média.

### DomainConfigurationService e ports/factories

- Descrição: facade para carregar e resolver configuração unificada de domínios por modelos, repository ports e factories.
- Tags: `configuração`, `domínio`, `factory`
- Tipo: service e interface
- Arquivos: `src/domain_configuration/service.py`, `src/domain_configuration/models.py`, `src/domain_configuration/ports/`, `src/domain_configuration/factories/`.
- Linguagem: Python
- Responsabilidade principal: evitar acesso local a configuração de domain processing/QA.
- Dependências principais: repositories/config YAML e modelos do bounded context.
- Acoplamento forte com domínio: Sim; configuração por domínio.
- Uso atual: Sim; exportado por `src.domain_configuration` e usado por factories/processors.
- Seguro reutilizar como está: Sim, dentro desse bounded context.
- Riscos ou limitações: não substitui SystemConfigManager nem YAML resolution global.
- Sugestão de melhoria: manter ports livres de adapters concretos.
- Prioridade: Média.

### ExtractTransformLoadOrchestrator e BaseApifyETLPipeline

- Descrição: orchestrator/foundation para executar pipelines ETL declarados e base Apify com coleta, transformação, persistência e stats comuns.
- Tags: `etl`, `apify`, `pipeline`
- Tipo: service e classe-base
- Arquivos: `src/etl_layer/orchestrator.py`, `src/etl_layer/providers/apify/base_apify_pipeline.py`.
- Linguagem: Python
- Responsabilidade principal: evitar orquestração e lifecycle repetidos nos pipelines Booking/Hotels/TripAdvisor.
- Dependências principais: ingestion clients/models, billing, persistence utils e logging ETL.
- Acoplamento forte com domínio: Médio; ETL/Apify.
- Uso atual: Sim; subclasses concretas e ETL service/processes usam estas bases.
- Seguro reutilizar como está: Sim, para pipeline ETL compatível.
- Riscos ou limitações: pipelines de hospitality têm regras específicas e não devem ser promovidas à base.
- Sugestão de melhoria: manter stats/persistence compartilhados fora das subclasses.
- Prioridade: Média.

### Telemetria operacional de ingestão

- Descrição: status machine, insights/scorecards e repositories de active/archive/overview que alimentam dashboards e leitura operacional sem consultar vector store bruto em cada endpoint.
- Tags: `telemetria`, `ingestão`, `dashboard`
- Tipo: service, enum e repository
- Arquivos: `src/telemetry/ingestion/status_machine.py`, `src/telemetry/ingestion/operational_insights.py`, `src/telemetry/ingestion/vector_active_archive_repository.py`, `src/telemetry/ingestion/vector_store_ingestion_overview_repository.py`, `src/telemetry/ingestion/ingestion_operational_read_port.py`.
- Linguagem: Python
- Responsabilidade principal: padronizar status/contadores/read models operacionais.
- Dependências principais: PostgresQueryExecutor, vector identity, logging e API read services.
- Acoplamento forte com domínio: Sim; ingestão.
- Uso atual: Sim; repositories têm dezenas de imports em routers/services/admin e status machine em dashboards.
- Seguro reutilizar como está: Sim, no slice operacional.
- Riscos ou limitações: projeção não é store físico autoritativo para conteúdo vetorial; claims de existência exigem fonte real.
- Sugestão de melhoria: manter canonicalização de status centralizada.
- Prioridade: Média.

### InteractionTelemetryManager e PopularQueryTermsService

- Descrição: managers/repositories para persistir runs de interação e termos populares, com batch/fallback e contexto causal tipado.
- Tags: `telemetria`, `interação`, `consulta`
- Tipo: service e repository
- Arquivos: `src/telemetry/interaction/interaction_telemetry_manager.py`, `src/telemetry/interaction/interaction_runs_repository.py`, `src/telemetry/interaction/interaction_telemetry_persistence.py`, `src/telemetry/query/popular_query_terms_service.py`.
- Linguagem: Python
- Responsabilidade principal: oferecer telemetria de produto reutilizável entre perguntas/agents/workflows.
- Dependências principais: PostgreSQL, logging canônico, models e batch writers.
- Acoplamento forte com domínio: Médio; observabilidade de interações.
- Uso atual: Sim; managers são chamados por question/telemetry services e routers de análise.
- Seguro reutilizar como está: Sim, com contexto/correlation explícitos.
- Riscos ou limitações: fallback local não é persistência autoritativa em cloud sem filesystem persistente.
- Sugestão de melhoria: manter fallback visível no log e reconciliação explícita.
- Prioridade: Média.

### LogAnalysisRenderer

- Descrição: renderer clássico que transforma análise estruturada de logs em HTML/DOM seguro para o WebChat.
- Tags: `logging`, `frontend`, `renderer`
- Tipo: renderer
- Arquivo: `app/ui/static/js/shared/log-analysis-renderer.js`
- Linguagem: JavaScript
- Responsabilidade principal: evitar templates de diagnóstico duplicados no host.
- Dependências principais: DOM/CSS `log-analysis-renderer.css` e contrato do Log Analyzer API.
- Acoplamento forte com domínio: Médio; visualização de análise de logs.
- Uso atual: Sim; carregado por `ui-webchat-v3.html` e chamado por `ui-webchat-v3.js`, com teste de contrato.
- Seguro reutilizar como está: Sim, para payload do analisador suportado.
- Riscos ou limitações: não substitui safe content geral e não deve executar HTML arbitrário do log.
- Sugestão de melhoria: manter compatibilidade de schema no renderer central.
- Prioridade: Média.

### PrometeuYamlIngestionFieldsExtractor

- Descrição: parser compartilhado de campos de destino/execução do YAML de ingestão PDF usado pela confirmação e card de destino.
- Tags: `yaml`, `ingestão`, `parser`
- Tipo: helper
- Arquivo: `app/ui/static/js/shared/yaml-ingestion-fields-extractor.js`
- Linguagem: JavaScript
- Responsabilidade principal: impedir duas leituras divergentes do mesmo YAML dentro da tela PDF.
- Dependências principais: JavaScript padrão e schema YAML de ingestão conhecido.
- Acoplamento forte com domínio: Sim; ingestão PDF.
- Uso atual: Sim, mas limitado a um fluxo; carregado por `ui-admin-plataforma-ingestao-pdf.html`, usado duas vezes em `admin-ingestao-pdf.js` e coberto por testes.
- Seguro reutilizar como está: Sim, em outro consumidor do mesmo contrato de ingestão.
- Riscos ou limitações: não é resolvedor YAML backend e ainda não há segundo fluxo de produção comprovado.
- Sugestão de melhoria: manter como helper compartilhado apenas enquanto card/confirmação continuarem consumidores distintos.
- Prioridade: Média.

### LocalCrypto.createLocalCryptoApi

- Descrição: factory UMD de criptografia local PBKDF2/AES-GCM usada para payloads que precisam permanecer cifrados no browser.
- Tags: `criptografia`, `browser`, `storage`
- Tipo: factory e helper
- Arquivo: `app/ui/static/js/shared/local-crypto-api.js`
- Linguagem: JavaScript
- Responsabilidade principal: centralizar derivação, encrypt/decrypt e encoding em vez de Web Crypto ad hoc.
- Dependências principais: Web Crypto API e password resolver injetado.
- Acoplamento forte com domínio: Baixo.
- Uso atual: Sim; consumido por client portal e fluxo WhatsApp, além de testes.
- Seguro reutilizar como está: Com cautela; somente quando criptografia local for requisito explícito.
- Riscos ou limitações: password resolver vazio/deprecated wrappers tornam uso incorreto inseguro; não substitui criptografia/autorização backend.
- Sugestão de melhoria: remover wrappers deprecated quando consumidores atuais estiverem migrados.
- Prioridade: Média.

### GesdocApi

- Descrição: facade compartilhada da família DNIT/Gesdoc para projetos, sessão e chamadas de chat, reaproveitando WebchatRuntimeUtils em vez de transporte por página.
- Tags: `frontend`, `gesdoc`, `api`
- Tipo: client e facade
- Arquivo: `app/ui/static/js/shared/gesdoc-api.js`
- Linguagem: JavaScript
- Responsabilidade principal: centralizar requests e eventos da família Gesdoc.
- Dependências principais: WebchatRuntimeUtils, LayoutMestreApi/Admin client e session context.
- Acoplamento forte com domínio: Sim; DNIT/Gesdoc.
- Uso atual: Sim; consumido por múltiplas páginas/scripts da família e testes de sessão/backend.
- Seguro reutilizar como está: Sim, dentro dessa família.
- Riscos ou limitações: não é API genérica de chat; o detalhe usa EmbeddableChatRuntime como motor de conversa.
- Sugestão de melhoria: manter hosts Gesdoc finos sobre esta facade e o chat oficial.
- Prioridade: Média.

## Itens com potencial de reuso ainda não comprovado em produção

### plataforma-agentes-ia-components

- Descrição: catálogo de primitives visuais que publica `window.PrometeuComponents`/`window.PC` para cards, badges, botões e markup administrativo.
- Tags: `frontend`, `componentes`, `dom`
- Tipo: componente e helper
- Arquivo: `app/ui/static/js/shared/plataforma-agentes-ia-components.js`
- Linguagem: JavaScript
- Responsabilidade principal: concentrar HTML de primitives visuais.
- Dependências principais: DOM e CSS da plataforma.
- Acoplamento forte com domínio: Baixo.
- Uso atual: Potencial; o script é carregado por páginas, mas a busca code-first não encontrou chamada nomeada externa de `PrometeuComponents`/`PC` nesta rodada.
- Seguro reutilizar como está: Hipótese; API publicada é funcional, porém falta consumidor comprovado fora do próprio módulo.
- Riscos ou limitações: HTML retornado precisa permanecer escapado; carregamento sem chamada não prova adoção real.
- Sugestão de melhoria: provar um consumidor ou remover o carregamento ocioso antes de promover a prioridade.
- Prioridade: Baixa.

## Exclusões comprovadas e correções de drift

Os itens abaixo não fazem parte da biblioteca reutilizável atual:

- `app/ui/static/js/shared/assembly-supervisor-builder-adapter.js`: declara mensagem de depreciação e lança erro; não é base para implementação nova.
- `src/agentic_layer/memory/memory_manager.py`: o próprio módulo marca `LEGACY_RUNTIME_BOUNDARY = "fora_do_runtime_oficial"`; o runtime canônico é `src/agentic_layer/supervisor/memory_factory.py`.
- `app/ui/static/js/shared/ui-webchat-legacy-utils.js` e `app/ui/static/js/shared/log-analysis-actions.js`: sem consumidor de produção comprovado na cadeia atual; o chat oficial usa `EmbeddableChatRuntime` e `WebchatRuntimeUtils`.
- `app/ui/static/js/shared/plataforma-agentes-ia-log-card.js`: define custom element, mas nenhuma tag/load/instância consumidora foi encontrada; não foi inventariado como reutilizável ativo.
- `app/ui/static/js/shared/admin-x-key-manager.js`: publica API global/CommonJS, mas nenhum consumidor real foi encontrado nesta rodada.
- `app/ui/static/js/shared/user-menu.js`: testes de contrato exigem que páginas atuais não o carreguem; o menu corrente pertence ao `PrometeuGlobalHeader`.
- `DnitProjectChatRuntime`: não existe no código atual e `tests/frontend/ui_gesdoc_project_detail_runtime_contract.test.js` exige sua ausência; o host usa `EmbeddableChatRuntime`.
- `AdminSchedulerExecutionService`: não existe em `src/api/services/admin/` e `tests/frontend/admin_scheduler_page_contract.test.js` exige que o router não o referencie; o catálogo anterior era falso positivo.
- O antigo `UiSpecValidator` JavaScript realmente não existe, mas `src/api/schemas/ag_ui_ui_spec_models.py::UiSpecValidator` está ativo e foi inventariado; a versão anterior do catálogo apagava essa distinção.

## Checagem anti-falso-negativo

A checagem final não usou este catálogo como filtro. Ela voltou ao código e confirmou:

1. **Runtime global:** `PrometeuLayoutMestreContextRuntime`, `PrometeuAdminCorrelationRuntime`, `EmbeddableChatRuntime`, `PrometeuGlobalHeader` e os runtimes AG-UI publicados continuam inventariados.
2. **Bridge/facade:** a cadeia `layout-mestre.js` → `admin-layout-bridge.js` → helpers/context consumers foi percorrida; no chat, `@prometeu/ag-ui-runtime` → bridges de transport/spec → `EmbeddableChatRuntime` → hosts também foi percorrida.
3. **Shared helpers:** todos os arquivos em `app/ui/static/js/shared/` foram triados; módulos sem consumidor foram excluídos com evidência, não promovidos por estarem na pasta `shared`.
4. **Eventos globais:** `prometeu:credenciais-alteradas`, eventos de correlation/action feedback, `prometeu:message` e eventos namespaced do chat foram ligados aos publishers e listeners reais.
5. **Testes de contrato:** contratos frontend/browser confirmaram ordem de scripts, globals, consumers, elementos/atributos esperados e ausências legadas. Em Python, `__all__`, facades e módulos com múltiplos imports foram confrontados com os consumidores.
6. **Drift anterior:** falsos positivos removidos, consumers/paths obsoletos não foram copiados e lacunas recentes de Job Core, scheduler, AG-UI, chat, SQL executor, cache/async, segurança e frontend global foram incorporadas.

## Consulta rápida por problema

- Correlação/log: `BaseCorrelationComponent` → `Logging System` → builder do vocabulário do slice.
- SQL/PostgreSQL: `DatabaseConnectionManager` → `PostgresQueryExecutor` → repository/service do domínio.
- Redis/cache: namespace de ambiente → `RedisManager`/cache tipado → epoch/invalidation quando necessário.
- Job assíncrono: `QueuedJobEnvelope` → `AsyncJobQueuePort`/`JobCoreJobQueue` → descriptor no catálogo → `JobCoreWorkerRuntime`.
- YAML de request: `resolve_yaml_configuration`; YAML agentic editável: `AgenticAssemblyService` e AST.
- Vector store: `VectorStoreFactory` e `PhysicalVectorTargetResolver`.
- Tools/Skills/MCP: `ToolsFactory`, decorator, materializador de Skills e resolvers MCP.
- Chat: `PrometeuEmbeddableChatRuntime`; sessões no host com `PrometeuChatSessionStore`.
- AG-UI browser: `packages/ag-ui-runtime/index.js`; backend: `AgUiRunOrchestrator`/registry/adapters.
- Frontend admin: `PrometeuAdminApiClient` + `PrometeuAdminLayoutBridge` + `admin-workspace-shell.js`.
