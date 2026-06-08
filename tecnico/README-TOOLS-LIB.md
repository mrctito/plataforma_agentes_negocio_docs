# Biblioteca de Reuso do Projeto

## Objetivo

Este documento resume classes, helpers, services, factories e modulos compartilhados que ja existem no repositorio e devem ser consultados antes de criar codigo novo. O foco aqui e evitar duplicacao, manter contratos canonicos do projeto e reduzir o risco de abrir caminhos paralelos para logging, correlation_id, retry, YAML-first, scheduler, canais e AG-UI.

## Escopo e criterio usado nesta analise

- Foram lidos modulos Python em `src` e superfices JavaScript em `app/ui/static/js/shared` e `packages/ag-ui-runtime`, alem de call sites e testes quando necessario para confirmar uso real.
- Entram neste catalogo apenas componentes com uso transversal real ou com potencial claro de reuso em mais de um fluxo, endpoint, worker, pagina ou modulo.
- Nao entram regras muito locais de negocio, funcoes privadas sem valor fora do arquivo, bundles vendor, codigo comentado ou codigo sem evidencias suficientes.

## Como usar este catalogo

- `Uso atual observado: Sim` significa que a analise encontrou import, instancia, chamada ou teste de contrato em outro ponto do repositorio.
- `Uso atual observado: Potencial` significa que o componente parece bom para reuso, mas o call site alem do proprio modulo nao foi confirmado nesta rodada.
- `Seguro reutilizar como esta?` indica se o componente parece pronto para plugar em outro fluxo sem exigir refatoracao estrutural.
- `Prioridade` indica o quanto vale conhecer o item antes de escrever implementacao nova.

## Infraestrutura transversal em Python

### BaseCorrelationComponent, BaseCorrelationManager e BaseCorrelationFactory

- Descricao: base obrigatoria para componentes que precisam propagar `correlation_id`, `yaml_config`, `user_email`, `user_session` e logger canonicamente, sem cada classe reinventar wiring de contexto.
- Tags: correlacao, logging, base
- Tipo: classe-base
- Arquivo: [src/core/base_correlation_component.py](../src/core/base_correlation_component.py)
- Linguagem: Python
- Responsabilidade principal: padronizar criacao de componentes com isolamento por correlacao e validacao minima de contexto multi-tenant.
- Dependencias principais: [src/core/logging_system.py](../src/core/logging_system.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura transversal.
- Uso atual observado: Sim. Herdado ou usado por [src/channel_layer/worker.py](../src/channel_layer/worker.py), [src/channel_layer/processor.py](../src/channel_layer/processor.py), [src/qa_layer/rag_engine/generation_engine.py](../src/qa_layer/rag_engine/generation_engine.py) e muitos componentes de ingestao.
- Seguro reutilizar como esta?: Sim, quando o fluxo ja tem `yaml_config` canonicamente resolvido.
- Riscos ou limitacoes: falha fechado se faltarem campos obrigatorios de contexto; nao serve para scripts soltos sem sessao de usuario.
- Sugestao de melhoria: manter um guia curto de onboarding para autores de novas classes transversais.
- Prioridade: Alta

### Logging System canonico

- Descricao: provider central de logging com isolamento por `correlation_id`, sanitizacao de segredos e integracao com armazenamento local e cloud sem exigir bypass por modulo.
- Tags: logging, observabilidade, correlacao
- Tipo: service
- Arquivo: [src/core/logging_system.py](../src/core/logging_system.py)
- Linguagem: Python
- Responsabilidade principal: criar e configurar loggers do projeto com formato e regras uniformes de observabilidade, preservando o contrato estrutural do payload emitido.
- Dependencias principais: [src/config/settings.py](../src/config/settings.py), `structlog`, [src/core/log_local_storage.py](../src/core/log_local_storage.py), [src/core/log_origin_metadata.py](../src/core/log_origin_metadata.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura obrigatoria.
- Uso atual observado: Sim. Consumido pela base de correlacao em [src/core/base_correlation_component.py](../src/core/base_correlation_component.py) e por servicos de conexao e storage como [src/core/database_connection_manager.py](../src/core/database_connection_manager.py) e [src/security/project_file_storage.py](../src/security/project_file_storage.py).
- Seguro reutilizar como esta?: Sim. E o caminho correto para logs novos.
- Riscos ou limitacoes: exige respeitar o provider canonico; nao deve ser contornado por loggers locais nem por payload global montado manualmente em call sites quando ja houver builder oficial. Fora do boundary oficial, nao autoriza criar logger com identidade propria nem `correlation_id` novo por conveniencia; se a classe precisar de uma instancia local, ela deve usar o mesmo `correlation_id` oficial ja presente no contexto.
- Sugestao de melhoria: manter documentacao curta de marcadores de log mais usados.
- Prioridade: Alta

### create_component_logger

- Descricao: helper canônico para logger técnico de componente apenas quando a execução está fora de qualquer processo real correlacionado e ainda não existe `correlation_id` oficial para preservar.
- Tags: logging, observabilidade, componente
- Tipo: helper
- Arquivo: [src/core/logging_system.py](../src/core/logging_system.py)
- Linguagem: Python
- Responsabilidade principal: criar logger compartilhado de componente com `component` e `origin`, sem ocupar o campo `correlation_id`, para eventos técnicos anteriores ou externos a um processo real correlacionado.
- Dependencias principais: [src/core/logging_system.py](../src/core/logging_system.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura transversal de observabilidade.
- Uso atual observado: Sim. Consumido por [src/core/resource_pool.py](../src/core/resource_pool.py), [src/core/database_connection_manager.py](../src/core/database_connection_manager.py), [src/api/service_api.py](../src/api/service_api.py), [src/config/config_api/system_config_manager.py](../src/config/config_api/system_config_manager.py) e demais hotspots técnicos já migrados.
- Seguro reutilizar como esta?: Sim, somente quando nao existir `correlation_id` oficial ja criado, recebido ou propagavel para a execucao.
- Riscos ou limitacoes: a classificacao do componente como tecnico nao basta para liberar este helper; se a execucao ja faz parte de um processo real correlacionado, usar `create_component_logger` esconde rastreabilidade e viola o contrato canonico. Se o componente for o iniciador nao interativo da execucao real, como scheduler ou runner dono do boundary, ele deve primeiro criar o `correlation_id` oficial e depois usar o logger oficial da execucao, nao este helper tecnico.
- Sugestao de melhoria: continuar migrando call sites de processo real correlacionado para `create_logger_with_correlation(...)`, sempre com `correlation_id` ja resolvido canonicamente no boundary oficial.

### emit_canonical_log

- Descricao: helper compartilhado para emitir payload canônico em loggers heterogeneos sem perder contexto estruturado quando o chamador recebe `SystemLogger` ou logger padrão do Python.
- Tags: logging, observabilidade, compatibilidade
- Arquivo: [src/core/logging_system.py](../src/core/logging_system.py)
- Responsabilidade principal: encapsular a compatibilidade entre `SystemLogger` e logging padrão, sempre preservando `event_name` e o payload canônico em `canonical_payload`, sem fallback silencioso para mensagem solta.
- Dependencias principais: [src/core/logging_system.py](../src/core/logging_system.py), [src/core/log_canonical_fields.py](../src/core/log_canonical_fields.py)
- Uso atual observado: Sim. Reutilizado por [src/core/async_utils.py](../src/core/async_utils.py), [src/core/decorators.py](../src/core/decorators.py), [src/core/lazy_libs.py](../src/core/lazy_libs.py), [src/core/message_history_factory.py](../src/core/message_history_factory.py), [src/core/cache/llm/llm_cache_manager.py](../src/core/cache/llm/llm_cache_manager.py) e [src/core/job_core/executor.py](../src/core/job_core/executor.py).
- Seguro reutilizar como esta?: Sim. E o caminho correto quando um slice precisa emitir log canônico mas o tipo concreto do logger pode variar.
- Riscos ou limitacoes: se o logger concreto nao aceitar nem `extra=` nem `**payload`, o helper falha fechado com `TypeError` em vez de descartar o contexto.
- Sugestao de melhoria: manter futuras adaptacoes de compatibilidade concentradas aqui, sem recriar detectores locais de `SystemLogger` nos slices.
- Prioridade: Alta

### initialize_sentry_infrastructure

- Descricao: bootstrap central e idempotente do Sentry para processos de infraestrutura, evitando que API, worker, scheduler ou CLI inventem inicializacao paralela do SDK e consolidando o estado desligado explicito quando a integracao nao pode ser ativada.
- Tags: sentry, observabilidade, bootstrap
- Tipo: helper
- Arquivo: [src/core/sentry_bootstrap.py](../src/core/sentry_bootstrap.py)
- Linguagem: Python
- Responsabilidade principal: resolver configuracao do Sentry a partir de [src/config/settings.py](../src/config/settings.py), carregar o `sentry_sdk` pelo lazy loader oficial, inicializar o SDK uma vez por processo e registrar em console apenas quando existir anomalia operacional real, sem usar o logger canônico.
- Dependencias principais: [src/core/lazy_libs.py](../src/core/lazy_libs.py), [src/config/settings.py](../src/config/settings.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura transversal de observabilidade.
- Uso atual observado: Sim. Consumido por [src/api/service_api.py](../src/api/service_api.py), [app/runners/api_runner.py](../app/runners/api_runner.py), [app/runners/worker_runner.py](../app/runners/worker_runner.py), [app/runners/scheduler_runner.py](../app/runners/scheduler_runner.py) e [app/runners/ingestion_runner.py](../app/runners/ingestion_runner.py).
- Seguro reutilizar como esta?: Sim, quando a necessidade for inicializar o SDK do Sentry sem espalhar `sdk.init(...)` por varios call sites e sem transformar ausencia de configuracao em erro de runtime.
- Riscos ou limitacoes: depende de `SENTRY_ENABLED` e `SENTRY_DSN`; se a flag estiver desligada, o helper mantem a integracao desligada em silencio. Se o DSN estiver vazio, se o pacote nao existir ou se o init falhar, o helper mantem a integracao desligada e escreve no console o motivo consolidado apenas uma vez por processo.
- Sugestao de melhoria: se no futuro for necessario enriquecer eventos com tags adicionais de dominio, fazer isso por extensao do bootstrap central e do handler opcional em vez de abrir inicializadores locais.
- Prioridade: Alta

### build_sentry_logging_handler

- Descricao: builder canônico do handler opcional que conecta o pipeline central de logging ao reporter de erro do Sentry sem espalhar wiring manual pelos call sites.
- Tags: sentry, logging, handler
- Tipo: helper
- Arquivo: [src/core/sentry_bootstrap.py](../src/core/sentry_bootstrap.py)
- Linguagem: Python
- Responsabilidade principal: decidir se o runtime deve expor um `logging.Handler` ativo para envio ao Sentry, respeitando `SENTRY_ENABLED`, `SENTRY_DSN`, a disponibilidade do SDK e o estado inicializado do bootstrap central.
- Dependencias principais: [src/config/settings.py](../src/config/settings.py), [src/core/lazy_libs.py](../src/core/lazy_libs.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura transversal de observabilidade.
- Uso atual observado: Sim. Consumido por [src/core/logging_system.py](../src/core/logging_system.py) como ponto unico de composicao do handler opcional no `setup_logging`.
- Seguro reutilizar como esta?: Sim, quando o objetivo for anexar o destino Sentry apenas no boundary canônico do logging, sem monkey patch em `logger.exception` e sem adapters paralelos.
- Riscos ou limitacoes: nao substitui o bootstrap do SDK; se o estado estiver desligado, o builder devolve `None` e o runtime segue sem handler Sentry.
- Sugestao de melhoria: manter qualquer evolucao do reporter ou do payload de excecao concentrada neste helper e no bootstrap, nunca em call sites de log de negocio.
- Prioridade: Alta

### resolve_http_request_correlation_id

- Descricao: helper compartilhado para boundaries HTTP que precisam reaproveitar o `correlation_id` oficial ja presente em `request.state`, no header `X-Correlation-Id` ou, em ultimo caso de boundary sem contexto previo, no valor explicito recebido pelo proprio endpoint antes de gerar um novo identificador canônico.
- Tags: correlacao, http, boundary
- Tipo: helper
- Arquivo: [src/api/request_correlation.py](../src/api/request_correlation.py)
- Linguagem: Python
- Responsabilidade principal: centralizar a ordem canônica de prioridade do `correlation_id` em endpoints HTTP para evitar que cada router invente regra local, ignore o contexto vivo do request ou volte a priorizar payload quando o request ja traz a correlacao oficial.
- Dependencias principais: [src/core/correlation_id_factory.py](../src/core/correlation_id_factory.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura transversal de boundary HTTP.
- Uso atual observado: Sim. Reutilizado por [src/api/service_api.py](../src/api/service_api.py), [src/api/routers/logs_router.py](../src/api/routers/logs_router.py), [src/api/routers/config_nl2sql_router.py](../src/api/routers/config_nl2sql_router.py), [src/api/routers/ag_ui_router.py](../src/api/routers/ag_ui_router.py), [src/api/routers/interaction_runs_router.py](../src/api/routers/interaction_runs_router.py) e [src/api/routers/agent_router.py](../src/api/routers/agent_router.py).
- Seguro reutilizar como esta?: Sim, quando o fluxo for HTTP e o objetivo for preservar a correlacao oficial do request antes de considerar qualquer valor explicito do proprio endpoint.
- Riscos ou limitacoes: nao substitui o middleware oficial de criacao de correlacao; se o endpoint estiver fora do boundary HTTP ou ja estiver dentro de um processo real correlacionado, o correto continua sendo receber e propagar o `correlation_id` resolvido em vez de reler o request.
- Sugestao de melhoria: expandir a adocao para outros routers que ainda leiam `request.state`, header ou payload manualmente ao resolver correlacao.
- Prioridade: Alta

### Catalogo canonico de campos globais de log

- Descricao: catalogo central dos campos canônicos globais de observabilidade e builder compartilhado para montar a parte transversal do payload de log sem espalhar ownership por slices, boundaries ou helpers locais.
- Tags: logging, observabilidade, contrato-canonico
- Tipo: helper
- Arquivo: [src/core/log_canonical_fields.py](../src/core/log_canonical_fields.py)
- Linguagem: Python
- Responsabilidade principal: definir a fonte única de verdade dos campos globais, dos grupos de campos transversais e do builder reutilizável que normaliza `event_name`, aceita apenas campos permitidos e preserva o contrato global fora do domínio.
- Dependencias principais: tipagem Python e `pathlib`; não depende de módulos de domínio.
- Acoplamento forte com dominio?: Nao. Infraestrutura transversal de observabilidade.
- Uso atual observado: Sim. Consumido por [src/telemetry/rag/log_vocabulary.py](../src/telemetry/rag/log_vocabulary.py), [src/ingestion_layer/telemetry/log_vocabulary.py](../src/ingestion_layer/telemetry/log_vocabulary.py), [src/api/service_api.py](../src/api/service_api.py), [src/api/services/async_job_dramatiq.py](../src/api/services/async_job_dramatiq.py), [src/agentic_layer/background_execution/runtime.py](../src/agentic_layer/background_execution/runtime.py) e [src/telemetry/interaction/interaction_telemetry_manager.py](../src/telemetry/interaction/interaction_telemetry_manager.py).
- Seguro reutilizar como esta?: Sim, sempre que a necessidade for montar a parte global do payload ou compor grupos canônicos sem arrastar semântica de domínio para o core.
- Riscos ou limitacoes: não substitui vocabulário de domínio; usar este módulo para campos específicos de RAG, ingestão ou fluxo local seria acoplamento errado. O desvio oposto também é problema: inventar campos globais ou aliases locais fora deste catálogo quebra consulta e auditoria.
- Sugestao de melhoria: continuar migrando call sites manuais para o builder central e manter a lista de grupos sincronizada com os produtores oficiais de runtime e com os builders oficiais dos slices.
- Prioridade: Alta

### CanonicalLogReader

- Descricao: leitor canônico de arquivos de log por `correlation_id`, com resolucao de arquivo primario, familia de logs (worker + scheduler) e arquivos rotacionados via manifest ou heuristica de nome de arquivo.
- Tags: logging, observabilidade, io
- Tipo: service
- Arquivo: [src/api/services/canonical_log_reader.py](../src/api/services/canonical_log_reader.py)
- Linguagem: Python
- Responsabilidade principal: encapsular toda a logica de descoberta de arquivos de log sem expor regras de nomeacao, manifest JSONL ou heuristica de familia para chamadores externos.
- Dependencias principais: [src/core/log_origin_metadata.py](../src/core/log_origin_metadata.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura de I/O de observabilidade.
- Uso atual observado: Sim. Consumido por [src/log_analyzer/io/locator.py](../src/log_analyzer/io/locator.py) como dependencia injetavel.
- Seguro reutilizar como esta?: Sim, sempre que for necessario localizar arquivos de log por `correlation_id` sem acesso direto ao filesystem.
- Riscos ou limitacoes: contem heuristica que pode nao cobrir todos os padroes de nomeacao futuros; a ordem canônica (manifest > sidecar meta > nome direto) deve ser preservada.
- Prioridade: Alta

### LogAnalyzerService

- Descricao: fachada unica de analise de logs V1.0 que orquestra o pipeline `LogFileLocator -> LogRecordLoader -> AbstractAnalysisEngine -> AnalysisRegistry` sem expor classes internas a chamadores externos. Nao depende de `yaml_config`, `settings` nem de qualquer outro componente de produto.
- Tags: logging, observabilidade, analise, pipeline
- Tipo: service
- Arquivo: [src/log_analyzer/service.py](../src/log_analyzer/service.py)
- Linguagem: Python
- Responsabilidade principal: receber contratos de entrada (`IndividualAnalysisRequest`, `AggregatedAnalysisRequest`), executar o pipeline de analise e sempre retornar `AnalysisResult` sem levantar excecao.
- Dependencias principais: [src/log_analyzer/io/locator.py](../src/log_analyzer/io/locator.py), [src/log_analyzer/io/loader.py](../src/log_analyzer/io/loader.py), [src/log_analyzer/analysis/pandas_engine.py](../src/log_analyzer/analysis/pandas_engine.py), [src/log_analyzer/analysis/registry.py](../src/log_analyzer/analysis/registry.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura transversal de observabilidade.
- Uso atual observado: Sim. Consumido por [src/api/services/logs_admin_service.py](../src/api/services/logs_admin_service.py), [src/api/routers/logs_router.py](../src/api/routers/logs_router.py) (`/logs/v1/analyze`) e [src/agentic_layer/mcp/servers/qa_system_server.py](../src/agentic_layer/mcp/servers/qa_system_server.py).
- Seguro reutilizar como esta?: Sim. E o ponto de entrada oficial para toda analise de logs.
- Riscos ou limitacoes: a engine atual e Pandas; para datasets muito grandes, considerar o limite `max_records` em `IndividualAnalysisRequest`.
- Prioridade: Alta

### LogFileLocator

- Descricao: localiza arquivo primario e familia de logs para um `correlation_id` sem usar `settings` globais. Segue ordem: manifest JSONL > CanonicalLogReader > heuristica de familia.
- Tags: logging, io, localizacao
- Tipo: helper
- Arquivo: [src/log_analyzer/io/locator.py](../src/log_analyzer/io/locator.py)
- Linguagem: Python
- Responsabilidade principal: resolver os caminhos de arquivo fisico de um log sem conhecer o produto nem as regras de negocio.
- Dependencias principais: [src/api/services/canonical_log_reader.py](../src/api/services/canonical_log_reader.py), [src/core/log_origin_metadata.py](../src/core/log_origin_metadata.py)
- Acoplamento forte com dominio?: Nao.
- Uso atual observado: Sim. Consumido exclusivamente por [src/log_analyzer/service.py](../src/log_analyzer/service.py).
- Seguro reutilizar como esta?: Sim, para localizacao de logs por `correlation_id`.
- Prioridade: Media

### LogAnalyzer CLI Validation Support

- Descricao: núcleo compartilhado de validação forense da CLI do log analyzer, com execução controlada do boundary real, congelamento de snapshots e oráculo manual independente para confronto de contrato.
- Tags: logging, observabilidade, validacao, cli
- Tipo: helper
- Arquivo: [src/log_analyzer/validation](../src/log_analyzer/validation)
- Linguagem: Python
- Responsabilidade principal: evitar scripts efêmeros em runtime, permitindo que testes e scripts operacionais reutilizem a mesma base para comparar `question_type` da CLI contra uma verdade manual reproduzível.
- Dependencias principais: [src/log_analyzer/__main__.py](../src/log_analyzer/__main__.py), [src/log_analyzer/io/locator.py](../src/log_analyzer/io/locator.py), [src/log_analyzer/parsing/field_extractor.py](../src/log_analyzer/parsing/field_extractor.py), [src/core/log_origin_metadata.py](../src/core/log_origin_metadata.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura de validação de observabilidade.
- Uso atual observado: Sim. Consumido por [tests/unit/test_02-07-31_log_analyzer_cli_truth_corpus.py](../tests/unit/test_02-07-31_log_analyzer_cli_truth_corpus.py), [tests/unit/test_02-07-32_log_analyzer_validation_manual_truth.py](../tests/unit/test_02-07-32_log_analyzer_validation_manual_truth.py) e pelos scripts em [.github/.scripts/testar_cli_log_analyzer](../.github/.scripts/testar_cli_log_analyzer).
- Seguro reutilizar como esta?: Sim, sempre que a necessidade for validar a CLI do log analyzer ou materializar snapshots auditáveis sem usar o próprio pipeline interno como juiz.
- Riscos ou limitacoes: o oráculo manual precisa acompanhar a evolução do catálogo de `question_type`; quando um tipo novo surgir, a suíte e os scripts devem ser atualizados no mesmo commit.
- Prioridade: Alta

### LogRecordLoader

- Descricao: carregador streaming de registros JSONL que nunca le o arquivo inteiro na memoria e respeita limite `max_records`. Linhas malformadas sao silenciosamente ignoradas.
- Tags: logging, io, streaming
- Tipo: helper
- Arquivo: [src/log_analyzer/io/loader.py](../src/log_analyzer/io/loader.py)
- Linguagem: Python
- Responsabilidade principal: carregar registros de multiplos arquivos JSONL com controle de limite, truncamento e contagem.
- Dependencias principais: `json` e `pathlib` — sem dependencia de dominio.
- Acoplamento forte com dominio?: Nao.
- Uso atual observado: Sim. Consumido por [src/log_analyzer/service.py](../src/log_analyzer/service.py).

### LogAnalyzer CLI Validation Helpers

- Descricao: conjunto de helpers compartilhados para congelar snapshots de famílias de log, executar o boundary real da CLI e construir uma verdade manual independente para auditoria do contrato público.
- Tags: logging, observabilidade, validacao, cli
- Tipo: helper
- Arquivos: [src/log_analyzer/validation/__init__.py](../src/log_analyzer/validation/__init__.py), [src/log_analyzer/validation/cli_runtime.py](../src/log_analyzer/validation/cli_runtime.py), [src/log_analyzer/validation/manual_truth.py](../src/log_analyzer/validation/manual_truth.py), [src/log_analyzer/validation/snapshot.py](../src/log_analyzer/validation/snapshot.py)
- Linguagem: Python
- Responsabilidade principal: evitar snippets efêmeros em runtime ao materializar um núcleo reutilizável para snapshot, execução da CLI, reconstrução de expectativa manual e comparação determinística de contrato.
- Dependencias principais: [src/log_analyzer/__main__.py](../src/log_analyzer/__main__.py), [src/log_analyzer/io/locator.py](../src/log_analyzer/io/locator.py), [src/core/log_origin_metadata.py](../src/core/log_origin_metadata.py), [src/log_analyzer/schema_registry.py](../src/log_analyzer/schema_registry.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura de validação da ferramenta de observabilidade.
- Uso atual observado: Sim. Consumido por [tests/unit/test_02-07-31_log_analyzer_cli_truth_corpus.py](../tests/unit/test_02-07-31_log_analyzer_cli_truth_corpus.py) e pelos scripts em .github/.scripts/testar_cli_log_analyzer/README.md.
- Seguro reutilizar como esta?: Sim, quando a necessidade for validar a CLI do log analyzer contra uma leitura manual independente sem reimplementar snapshot e parsing a cada rodada.
- Riscos ou limitacoes: o oráculo manual precisa evoluir junto com novos `question_type`; se a CLI ganhar semântica nova e esse núcleo não for atualizado, a auditoria passa a falhar por drift de contrato.
- Prioridade: Alta
- Seguro reutilizar como esta?: Sim.
- Prioridade: Media

### External Retry helpers

- Descricao: mecanismo central de retry para recursos externos, com configuracao declarativa, backoff exponencial e logging por tentativa, evitando cada integracao inventar sua propria politica.
- Tags: retry, resiliencia, integracao
- Tipo: helper
- Arquivo: [src/core/external_retry.py](../src/core/external_retry.py)
- Linguagem: Python
- Responsabilidade principal: encapsular retries sincronos e assincronos com classificacao de falhas transitivas.
- Dependencias principais: `tenacity`, logging do projeto
- Acoplamento forte com dominio?: Nao. Infraestrutura transversal.
- Uso atual observado: Sim. Chamado por [src/telemetry/job_runs/canonical_job_runs_manager.py](../src/telemetry/job_runs/canonical_job_runs_manager.py), [src/config/agentic_assembly/nl/structured_llm_client.py](../src/config/agentic_assembly/nl/structured_llm_client.py), [src/scheduler_layer/postgres_repository.py](../src/scheduler_layer/postgres_repository.py), [src/ingestion_layer/hospitality/repository.py](../src/ingestion_layer/hospitality/repository.py), [src/ingestion_layer/hospitality/nlp_repository.py](../src/ingestion_layer/hospitality/nlp_repository.py), [src/qa_layer/memory/postgres_backend.py](../src/qa_layer/memory/postgres_backend.py) e [src/ucp/ucp_fallback_runtime.py](../src/ucp/ucp_fallback_runtime.py).
- Seguro reutilizar como esta?: Sim, principalmente para I/O externo e storage remoto.
- Riscos ou limitacoes: usar retry em erro nao transitivo e um anti-pattern; cada chamada ainda precisa escolher bem o timeout e a classificacao de excecao.
- Sugestao de melhoria: consolidar exemplos de configuracao por tipo de recurso e continuar removendo wrappers locais que ainda escondam telemetria ou semantica propria de retry.
- Prioridade: Alta

### TransactionalEmailApplicationService

- Descricao: service interno reutilizavel que traduz pedidos funcionais de envio transacional para o provider oficial sem deixar endpoint, caso de uso ou tool conhecer payload, headers ou autenticacao da Brevo.
- Tags: email-transacional, service, boundary-interno
- Tipo: service
- Uso geral: Sim. Este e o contrato interno canônico para fluxos do produto e para tools que precisem enviar e-mail transacional sem provider paralelo.
- Arquivo: [src/services/transactional_email_service.py](../src/services/transactional_email_service.py)
- Linguagem: Python
- Responsabilidade principal: receber pedidos funcionais de envio, normalizar destinatarios e conteúdo, expor contrato estável de resultado e oferecer caso de uso pronto para reset local de senha.
- Dependencias principais: [src/config/settings.py](../src/config/settings.py), [src/services/brevo_transactional_email_client.py](../src/services/brevo_transactional_email_client.py)
- Acoplamento forte com dominio?: Baixo. E uma camada de aplicacao reutilizavel para envio administrativo.
- Uso atual observado: Sim. Consumido por [src/api/routers/auth_router.py](../src/api/routers/auth_router.py) no boundary oficial do reset local e por [src/agentic_layer/tools/vendor_tools/brevo_tools/brevo_toolkit.py](../src/agentic_layer/tools/vendor_tools/brevo_tools/brevo_toolkit.py) como capacidade interna compartilhada.
- Seguro reutilizar como esta?: Sim, quando o fluxo precisar enviar e-mail transacional sem abrir provider paralelo.
- Riscos ou limitacoes: exige `correlation_id` oficial ja resolvido no boundary e configuracao Brevo valida em `WEB_FEDERATED_AUTH__BREVO_*`; nao fornece fallback implicito para SMTP.
- Sugestao de melhoria: se novos fluxos administrativos surgirem, expandir pedidos funcionais por composicao neste service em vez de duplicar clientes de provider.
- Prioridade: Alta

### BrevoTransactionalEmailClient

- Descricao: implementacao concreta do provider transacional Brevo por HTTP, centralizando autenticacao, payload tecnico, timeout, retry canônico, logs estruturados e captura de `messageId`.
- Tags: brevo, provider, email-transacional
- Tipo: service
- Uso geral: Sim, por composicao. Esta e a API interna geral da Brevo no runtime do produto e deve ser consumida pelo service transacional, nao diretamente por endpoint, router ou tool.
- Arquivo: [src/services/brevo_transactional_email_client.py](../src/services/brevo_transactional_email_client.py)
- Linguagem: Python
- Responsabilidade principal: falar com a API da Brevo em um unico ponto de infraestrutura, devolver resultado estruturado e manter observabilidade coerente do envio transacional.
- Dependencias principais: [src/core/external_retry.py](../src/core/external_retry.py), [src/core/log_canonical_fields.py](../src/core/log_canonical_fields.py), [src/core/logging_system.py](../src/core/logging_system.py), `httpx`
- Acoplamento forte com dominio?: Nao. Infraestrutura externa especializada em provider de e-mail.
- Uso atual observado: Sim. Instanciado por [src/services/transactional_email_service.py](../src/services/transactional_email_service.py) como provider oficial do envio transacional.
- Seguro reutilizar como esta?: Sim, desde que o chamador use o contrato do service interno em vez de instanciar este client diretamente em endpoint ou tool.
- Riscos ou limitacoes: foi desenhado para a API transacional da Brevo e depende de configuracao valida de remetente e `api_key`; uso direto fora da camada de aplicacao reintroduz acoplamento arquitetural.
- Sugestao de melhoria: manter evolucao de logs e erros concentrada neste modulo sempre que o contrato HTTP da Brevo mudar.
- Prioridade: Alta

### create_brevo_tools / brevo_send_email

- Descricao: factory e tool agentic da Brevo para envio de e-mail transacional usando a mesma capacidade interna do produto, sem criar provider paralelo.
- Tags: brevo, tool, email-transacional, vendor-tools
- Tipo: tool_factory
- Uso geral: Sim. Pode ser reutilizada por agentes e workflows sempre que o escopo agentic precisar enviar e-mail transacional usando a infraestrutura interna compartilhada.
- Arquivo: [src/agentic_layer/tools/vendor_tools/brevo_tools/brevo_toolkit.py](../src/agentic_layer/tools/vendor_tools/brevo_tools/brevo_toolkit.py)
- Linguagem: Python
- Responsabilidade principal: validar `user_session`, exigir `correlation_id`, traduzir argumentos agentic para `TransactionalEmailRequest` e delegar o envio ao `TransactionalEmailApplicationService`.
- Dependencias principais: [src/services/transactional_email_service.py](../src/services/transactional_email_service.py), [src/agentic_layer/tools/tool_factory_decorator.py](../src/agentic_layer/tools/tool_factory_decorator.py)
- Acoplamento forte com dominio?: Baixo. A tool conhece apenas o contrato interno e o envelope agentic.
- Uso atual observado: Sim. Disponibiliza a tool `brevo_send_email` para catálogos agentic que carreguem a factory `create_brevo_tools`.
- Seguro reutilizar como esta?: Sim, desde que o chamador forneca `user_session.correlation_id` valido e aceite o contrato textual simples atual.
- Riscos ou limitacoes: hoje a tool cobre envio textual simples; HTML, anexos ou templates precisam evoluir primeiro o contrato interno.
- Sugestao de melhoria: manter novas capacidades no service interno e deixar a tool apenas refletir o contrato compartilhado.
- Prioridade: Alta

### ResendTransactionalEmailClient

- Descricao: implementacao concreta do provider transacional Resend por HTTP, centralizando autenticacao, payload tecnico, timeout, retry canonico, logs estruturados e captura do identificador tecnico devolvido pela API.
- Tags: resend, provider, email-transacional
- Tipo: service
- Uso geral: Sim, por composicao. Esta e a API interna geral da Resend no runtime do produto e deve ser consumida pelo service transacional, nao diretamente por endpoint, router ou tool.
- Arquivo: [src/services/resend_transactional_email_client.py](../src/services/resend_transactional_email_client.py)
- Linguagem: Python
- Responsabilidade principal: falar com a API da Resend em um unico ponto de infraestrutura, devolver resultado estruturado e manter observabilidade coerente do envio transacional.
- Dependencias principais: [src/core/external_retry.py](../src/core/external_retry.py), [src/core/log_canonical_fields.py](../src/core/log_canonical_fields.py), [src/core/logging_system.py](../src/core/logging_system.py), `httpx`
- Acoplamento forte com dominio?: Nao. Infraestrutura externa especializada em provider de e-mail.
- Uso atual observado: Sim. Instanciado por [src/services/transactional_email_service.py](../src/services/transactional_email_service.py) quando `web_federated_auth.transactional_email_provider=resend`.
- Seguro reutilizar como esta?: Sim, desde que o chamador use o contrato do service interno em vez de instanciar este client diretamente em endpoint ou tool.
- Riscos ou limitacoes: foi desenhado para a API transacional da Resend e depende de configuracao valida de remetente e `api_key`; uso direto fora da camada de aplicacao reintroduz acoplamento arquitetural.
- Sugestao de melhoria: manter evolucao de logs e erros concentrada neste modulo sempre que o contrato HTTP da Resend mudar.
- Prioridade: Alta

### create_resend_tools / resend_send_email

- Descricao: factory e tool agentic da Resend para envio de e-mail transacional usando a mesma capacidade interna do produto, sem criar provider paralelo.
- Tags: resend, tool, email-transacional, vendor-tools
- Tipo: tool_factory
- Uso geral: Sim. Pode ser reutilizada por agentes e workflows sempre que o escopo agentic precisar enviar e-mail transacional usando a infraestrutura interna compartilhada.
- Arquivo: [src/agentic_layer/tools/vendor_tools/resend_tools/resend_toolkit.py](../src/agentic_layer/tools/vendor_tools/resend_tools/resend_toolkit.py)
- Linguagem: Python
- Responsabilidade principal: validar `user_session`, exigir `correlation_id`, traduzir argumentos agentic para `TransactionalEmailRequest` e delegar o envio ao `TransactionalEmailApplicationService`.
- Dependencias principais: [src/services/transactional_email_service.py](../src/services/transactional_email_service.py), [src/agentic_layer/tools/tool_factory_decorator.py](../src/agentic_layer/tools/tool_factory_decorator.py)
- Acoplamento forte com dominio?: Baixo. A tool conhece apenas o contrato interno e o envelope agentic.
- Uso atual observado: Sim. Disponibiliza a tool `resend_send_email` para catálogos agentic que carreguem a factory `create_resend_tools`.
- Seguro reutilizar como esta?: Sim, desde que o chamador forneca `user_session.correlation_id` valido e aceite o contrato textual simples atual.
- Riscos ou limitacoes: hoje a tool cobre envio textual simples; HTML, anexos ou templates precisam evoluir primeiro o contrato interno.
- Sugestao de melhoria: manter novas capacidades no service interno e deixar a tool apenas refletir o contrato compartilhado.
- Prioridade: Alta

### SmtpEmailApplicationService e SmtpEmailRequest / SmtpEmailResult

- Descricao: boundary interno geral para envio SMTP acima do cliente tecnico, com contrato estavel de entrada, resultado estruturado de plataforma, retry canonico e logs canônicos do envio.
- Tags: smtp, email, service, boundary-interno
- Tipo: service
- Uso geral: Sim. Este e o contrato interno canônico para fluxos do produto e tools que precisem enviar e-mail via SMTP sem falar diretamente com o client tecnico.
- Arquivo: [src/services/smtp_email_service.py](../src/services/smtp_email_service.py)
- Linguagem: Python
- Responsabilidade principal: receber pedidos estruturados de envio SMTP, preservar `correlation_id`, concentrar retry e observabilidade do envio e devolver resultado estruturado sem vazar `smtplib` nem detalhes operacionais para o chamador.
- Dependencias principais: [src/core/external_retry.py](../src/core/external_retry.py), [src/core/log_canonical_fields.py](../src/core/log_canonical_fields.py), [src/core/logging_system.py](../src/core/logging_system.py), `smtplib`
- Acoplamento forte com dominio?: Nao. E uma camada de aplicacao reutilizavel para capacidade tecnica de envio SMTP.
- Uso atual observado: Sim. Consumido por [src/api/services/hil_approval_notification_service.py](../src/api/services/hil_approval_notification_service.py) e por [src/agentic_layer/tools/vendor_tools/smtp_tools/smtp_toolkit.py](../src/agentic_layer/tools/vendor_tools/smtp_tools/smtp_toolkit.py) como boundary oficial do envio SMTP no recorte validado.
- Seguro reutilizar como esta?: Sim, quando o fluxo precisar enviar e-mail por SMTP sem abrir provider paralelo nem decidir retry localmente.
- Riscos ou limitacoes: exige `correlation_id` oficial resolvido no boundary do processo e configuracao SMTP valida em `tool_config.smtp_email` mais `security_keys.SMTP_*`; nao fornece fallback implicito para outro provider.
- Sugestao de melhoria: se surgirem novos casos de uso SMTP, evoluir o contrato neste service e manter tool e adapters apenas como camadas de adaptacao.
- Prioridade: Alta

### SMTPMailClient

- Descricao: implementacao tecnica SMTP reutilizada por composicao sob a API interna geral de envio, isolando `smtplib`, montagem de mensagem e classificacao basica de falhas transitorias.
- Tags: smtp, provider, infraestrutura
- Tipo: service
- Uso geral: Sim, por composicao. Deve ser consumido pela API interna SMTP e nao diretamente por endpoint, caso de uso ou tool.
- Arquivo: [src/services/smtp_email_service.py](../src/services/smtp_email_service.py)
- Linguagem: Python
- Responsabilidade principal: executar o envio SMTP em baixo nivel, montar a mensagem MIME textual, autenticar no servidor e classificar falhas transitorias para a politica canonica de retry.
- Dependencias principais: `smtplib`, `email.message`
- Acoplamento forte com dominio?: Nao. Infraestrutura tecnica especializada em SMTP.
- Uso atual observado: Sim. Instanciado por [src/services/smtp_email_service.py](../src/services/smtp_email_service.py) como provider tecnico do boundary oficial SMTP.
- Seguro reutilizar como esta?: Sim, somente por composicao dentro da API interna SMTP.
- Riscos ou limitacoes: uso direto fora da API interna reintroduz fragmentacao de retry, logs e contrato de retorno.
- Sugestao de melhoria: manter evolucao tecnica e classificacao de erros neste client, sem empurrar detalhes SMTP para os consumidores.
- Prioridade: Alta

### DatabaseConnectionManager

- Descricao: utilitario central para construir pools `psycopg` sync e async, engines SQLAlchemy e conexoes SQLite sync e async com logging uniforme, evitando duplicacao de bootstrap de banco e readiness checks.
- Tags: banco-de-dados, pool, postgres
- Tipo: service
- Arquivo: [src/core/database_connection_manager.py](../src/core/database_connection_manager.py)
- Linguagem: Python
- Responsabilidade principal: criar e validar conexoes com banco de forma padronizada e observavel, inclusive quando um boundary sincronico precisa inicializar recursos assincronos do fabricante.
- Dependencias principais: `psycopg`, `psycopg_pool`, `sqlalchemy`, `aiosqlite`, [src/core/logging_system.py](../src/core/logging_system.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura transversal.
- Uso atual observado: Sim. Usado em [src/telemetry/job_runs/canonical_job_runs_manager.py](../src/telemetry/job_runs/canonical_job_runs_manager.py), [src/qa_layer/rag_engine/fts_bootstrap.py](../src/qa_layer/rag_engine/fts_bootstrap.py), [src/security/offline_key_store.py](../src/security/offline_key_store.py), [src/agentic_layer/supervisor/memory_factory.py](../src/agentic_layer/supervisor/memory_factory.py) e varios repositores operacionais.
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: continua sendo responsabilidade do chamador escolher ciclo de vida do pool e evitar transacoes longas demais; em recursos async de longa vida, o caller ainda precisa definir como manter o loop vivo e quando encerrar cleanup.
- Sugestao de melhoria: expor exemplos por padrao de uso, como pool singleton, pool lazy, bootstrap async em boundary sync e engine read-only.
- Prioridade: Alta

### async_utils: run_sync e PersistentAsyncLoopRunner

- Descricao: utilitarios canonicos para interoperar codigo sincronico e assincrono sem inventar bridges locais. `run_sync` resolve a chamada pontual; `PersistentAsyncLoopRunner` mantem um loop dedicado vivo quando a bridge precisa durar mais que uma chamada, como checkpointers async oficiais usados por um boundary sincronico.
- Tags: async, bridge, loop-dedicado
- Tipo: helper
- Arquivo: [src/core/async_utils.py](../src/core/async_utils.py)
- Linguagem: Python
- Responsabilidade principal: executar awaitables a partir de callers sincronicos com regra previsivel de loop atual, thread auxiliar e loop externo persistente.
- Dependencias principais: `asyncio`, `concurrent.futures`, `threading`
- Acoplamento forte com dominio?: Nao. Infraestrutura transversal.
- Uso atual observado: Sim. Consumido por [src/agentic_layer/supervisor/memory_factory.py](../src/agentic_layer/supervisor/memory_factory.py) para sustentar AsyncSqliteSaver e AsyncPostgresSaver no boundary oficial do DeepAgent.
- Seguro reutilizar como esta?: Sim, quando o problema real for interoperabilidade sync-async duravel e nao logica de negocio.
- Riscos ou limitacoes: loop persistente sem politica de cleanup pode vazar thread; use junto com ciclo de vida explicito do recurso anexado.
- Sugestao de melhoria: manter exemplos curtos de uso para bootstrap async pontual e para recursos de longa vida que dependem de `run_coroutine_threadsafe`.
- Prioridade: Alta

### WarmResourcePool, RedisYamlCache e TTLCache

- Descricao: pool compartilhado de recursos quentes por tenant e ambiente, incluindo cache YAML e objetos de runtime caros de reconstruir, para reduzir custo de bootstrap repetido.
- Tags: cache, resource-pool, yaml
- Tipo: service
- Arquivo: [src/core/resource_pool.py](../src/core/resource_pool.py)
- Linguagem: Python
- Responsabilidade principal: reter e reciclar recursos com TTL, especialmente configuracoes YAML e dependencias de runtime.
- Dependencias principais: Redis, [src/core/vector_store_factory.py](../src/core/vector_store_factory.py), criadores de LLM e vector store do projeto
- Acoplamento forte com dominio?: Medio. E transversal, mas conhece artefatos concretos de runtime.
- Uso atual observado: Sim. Consumido por [src/config/yaml_config_manager.py](../src/config/yaml_config_manager.py), [src/api/services/admin/cache_service.py](../src/api/services/admin/cache_service.py) e [src/api/routers/admin/admin_runtime_router.py](../src/api/routers/admin/admin_runtime_router.py).
- Seguro reutilizar como esta?: Sim, quando a necessidade for cache operacional compartilhado de runtime.
- Riscos ou limitacoes: dependencias pesadas e mistura de varios tipos de recurso no mesmo modulo; nao e um cache generico neutro.
- Sugestao de melhoria: separar melhor cache de YAML, cache de runtime e factories de recurso pesado.
- Prioridade: Alta

### VectorStoreFactory e ConfiguredVectorStoreFactory

- Descricao: fabrica canonica de vector stores, com registro de tipos, resolucao por prefixo e operacoes padronizadas de criacao, reset e exclusao de stores fisicos.
- Tags: vector-store, factory, rag
- Tipo: factory
- Arquivo: [src/core/vector_store_factory.py](../src/core/vector_store_factory.py)
- Linguagem: Python
- Responsabilidade principal: instanciar a implementacao correta de vector store a partir do YAML e do identificador fisico.
- Dependencias principais: [src/shared/rag_contracts/interfaces.py](../src/shared/rag_contracts/interfaces.py), resolutores de alvo fisico, configuracao do projeto
- Acoplamento forte com dominio?: Medio. E infraestrutura de RAG, mas transversal dentro do dominio de retrieval.
- Uso atual observado: Sim. Usado por [src/ingestion_layer/document_persistence_manager.py](../src/ingestion_layer/document_persistence_manager.py), [src/ingestion_layer/vector_manager.py](../src/ingestion_layer/vector_manager.py) e [src/qa_layer/rag_engine/bm25_rehydrator.py](../src/qa_layer/rag_engine/bm25_rehydrator.py).
- Seguro reutilizar como esta?: Sim. Deve ser o ponto de entrada padrao para vector stores.
- Riscos ou limitacoes: acoplado aos tipos registrados hoje; adicionar provider novo exige registro formal.
- Sugestao de melhoria: documentar claramente o contrato para registrar novos backends.
- Prioridade: Alta

### MessageHistoryFactory

- Descricao: fabrica compartilhada de historico de conversa, capaz de montar loaders persistentes para memoria de chat sem cada fluxo conhecer detalhes de Redis, SQLite ou credenciais, e de expor um adapter canônico para chains que precisam injetar e persistir histórico sem depender de APIs depreciadas do LangChain.
- Tags: memoria, chat-history, factory
- Tipo: factory
- Arquivo: [src/core/message_history_factory.py](../src/core/message_history_factory.py)
- Linguagem: Python
- Responsabilidade principal: centralizar criacao de backend de historico conversacional e o adapter padrão de invocação com histórico persistente.
- Dependencias principais: `langchain_community`, [src/security/credential_manager.py](../src/security/credential_manager.py), [src/security/security_keys_resolver.py](../src/security/security_keys_resolver.py)
- Acoplamento forte com dominio?: Baixo. E mais infra de conversa do que regra de negocio.
- Uso atual observado: Sim. Usado em [src/qa_layer/qa_setup_manager.py](../src/qa_layer/qa_setup_manager.py) e [src/agentic_layer/supervisor/agent_builder.py](../src/agentic_layer/supervisor/agent_builder.py).
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: depende de configuracao de credenciais e de backends suportados hoje.
- Sugestao de melhoria: adicionar exemplo simples de escolha de backend por ambiente.
- Prioridade: Alta

### Contrato minimo IVectorStore

- Descricao: interface compartilhada que define o contrato minimo de vector store visivel por varias camadas do sistema, ajudando a manter baixo acoplamento entre dominio, factories e implementacoes concretas.
- Tags: contrato, vector-store, interface
- Tipo: interface
- Arquivo: [src/shared/rag_contracts/interfaces.py](../src/shared/rag_contracts/interfaces.py)
- Linguagem: Python
- Responsabilidade principal: fornecer a abstracao estavel que implementacoes e factories precisam respeitar.
- Dependencias principais: tipagem Python
- Acoplamento forte com dominio?: Medio. E o contrato base do dominio de RAG.
- Uso atual observado: Sim. Referenciado por [src/core/vector_store_factory.py](../src/core/vector_store_factory.py) e [src/ingestion_layer/vector_stores/base.py](../src/ingestion_layer/vector_stores/base.py).
- Seguro reutilizar como esta?: Sim, para novas implementacoes de vector store ou adapters.
- Riscos ou limitacoes: qualquer mudanca de contrato pode quebrar varias implementacoes ao mesmo tempo.
- Sugestao de melhoria: manter exemplos de implementacao minima perto do contrato.
- Prioridade: Alta

## Infraestrutura da suite oficial de testes

Nota de escopo: esta secao e inventario tecnico de componentes. O guia
operacional curto da suite fica em `docs/README-TESTS.MD`.

### Boundary publico da nova suite Python

- Descricao: entrypoint oficial que recebe o comando publico e delega integralmente para a CLI da nova suite Python, sem manter logica shell como fonte de verdade.
- Tags: suite-oficial, boundary, cli
- Tipo: boundary
- Arquivo: [suite_de_testes_padrao.py](../suite_de_testes_padrao.py)
- Linguagem: Python
- Responsabilidade principal: preservar o comando publico do time e redirecionar a execucao para a camada tipada em `src/python_test_suite`.
- Dependencias principais: [src/python_test_suite/cli.py](../src/python_test_suite/cli.py)
- Acoplamento forte com dominio?: Nao. E infraestrutura de validacao.
- Uso atual observado: Sim. E o caminho usado pelo workflow oficial e pela documentacao canônica da suite.
- Seguro reutilizar como esta?: Sim. Toda evolucao publica da suite deve continuar entrando por aqui.
- Riscos ou limitacoes: recolocar logica operacional no boundary ou reativar wrappers shell cria drift de contrato.
- Sugestao de melhoria: manter esse arquivo como casca fina e empurrar comportamento novo para a camada tipada.
- Prioridade: Alta

### CLI publica da nova suite Python

- Descricao: parser publico do boundary novo, com modos exclusivos, flags aditivas permitidas, validacao de `--focus-tests`, `--plan-only` e `--run-id` obrigatorio.
- Tags: suite-oficial, cli, contrato-publico
- Tipo: service
- Arquivo: [src/python_test_suite/cli.py](../src/python_test_suite/cli.py)
- Linguagem: Python
- Responsabilidade principal: resolver o modo publico e transformar a chamada do usuario em um `SuitePlan` valido.
- Dependencias principais: [src/python_test_suite/catalog.py](../src/python_test_suite/catalog.py), [src/python_test_suite/runner.py](../src/python_test_suite/runner.py)
- Acoplamento forte com dominio?: Nao. E infraestrutura de teste.
- Uso atual observado: Sim. Todo fluxo oficial da suite passa por essa CLI.
- Seguro reutilizar como esta?: Sim. Novas flags e novos modos devem nascer aqui, nunca em script paralelo.
- Riscos ou limitacoes: criar contrato publico por fora da CLI quebra validacao e mensagens de erro da suite.
- Sugestao de melhoria: manter as restricoes de combinacao de flags centralizadas aqui.
- Prioridade: Alta

### Catalogo canonico da nova suite Python

- Descricao: fonte unica dos modos, targets e composicoes do plano oficial da suite.
- Tags: suite-oficial, catalogo, plano
- Tipo: service
- Arquivo: [src/python_test_suite/catalog.py](../src/python_test_suite/catalog.py)
- Linguagem: Python
- Responsabilidade principal: montar o plano canônico para `--test-suite`, `--minimal`, `--focus-tests`, `--audit`, familias backend, `--frontend`, `--complete-lite`, `--complete`, `--complete-full` e flags aditivas de integracao, e2e e slow, sempre por `marker_expression` explicita ou por seletores explicitos de `--focus-tests`.
- Dependencias principais: [src/python_test_suite/models.py](../src/python_test_suite/models.py)
- Acoplamento forte com dominio?: Nao. E modelagem de execucao da suite.
- Uso atual observado: Sim. A CLI resolve o plano sempre a partir desse catalogo.
- Seguro reutilizar como esta?: Sim. Qualquer mudanca de composicao da suite deve partir daqui.
- Riscos ou limitacoes: espalhar marker expressions ou listas de targets em docs e scripts reabre contrato paralelo; nome de arquivo e pasta nao substituem esse catalogo.
- Sugestao de melhoria: continuar mantendo os recortes especializados de backend dentro desse catalogo.
- Prioridade: Alta

### Leitor checkup da nova suite Python

- Descricao: leitor read-only por `run_id` que inspeciona `state.json`, `telemetry.json`, `report_file` e sidecars `.live-progress.ndjson` da nova suíte sem criar pastas nem chamar o runner.
- Tags: suite-oficial, checkup, leitura
- Tipo: service
- Arquivo: [src/python_test_suite/checkup.py](../src/python_test_suite/checkup.py)
- Linguagem: Python
- Responsabilidade principal: montar uma análise humana ou JSON da rodada persistida para diagnóstico rápido do estado final já gravado, incluindo detalhes por target e agregados de report/live progress quando existirem.
- Dependencias principais: [src/python_test_suite/state.py](../src/python_test_suite/state.py)
- Acoplamento forte com dominio?: Nao. Infraestrutura read-only da suíte.
- Uso atual observado: Sim. Consumido pela CLI pública no modo `--checkup`.
- Seguro reutilizar como esta?: Sim, quando o objetivo for só ler artefatos oficiais da rodada.
- Riscos ou limitacoes: falha fechado se `state.json`, `telemetry.json`, `report_file` referenciado ou sidecar `.live-progress.ndjson` estiver ausente, corrompido ou incompleto quando a leitura daquele artefato for necessária.
- Sugestao de melhoria: manter o payload pequeno e estável, sem transformar o `checkup` em segundo runtime nem em segundo formato autoritativo de telemetria.
- Prioridade: Media

### Runner tipado da nova suite Python

- Descricao: motor de execucao que aplica o plano resolvido, controla o estado da rodada e consolida o resumo final.
- Tags: suite-oficial, runner, telemetria
- Tipo: service
- Arquivo: [src/python_test_suite/runner.py](../src/python_test_suite/runner.py)
- Linguagem: Python
- Responsabilidade principal: executar os targets do plano, impedir reuso indevido de `run_id` novo, gravar resultados por target e produzir o payload final da rodada.
- Dependencias principais: [src/python_test_suite/state.py](../src/python_test_suite/state.py), [src/python_test_suite/pytest_plugin.py](../src/python_test_suite/pytest_plugin.py)
- Acoplamento forte com dominio?: Nao. E infraestrutura de validacao.
- Uso atual observado: Sim. Toda execucao real da suite passa por esse runner.
- Seguro reutilizar como esta?: Sim. Novas politicas de execucao e bloqueio devem ser modeladas aqui.
- Riscos ou limitacoes: duplicar persistencia ou resumo final fora do runner gera artefatos inconsistentes.
- Sugestao de melhoria: manter bloqueios estruturados e o resumo final sempre sincronizados com a telemetria persistida.
- Prioridade: Alta

### State store e artefatos da nova suite Python

- Descricao: camada que define o formato persistido de `telemetry.json`, `state.json`, logs por target e reports pytest da rodada.
- Tags: suite-oficial, state-store, artefatos
- Tipo: service
- Arquivo: [src/python_test_suite/state.py](../src/python_test_suite/state.py)
- Linguagem: Python
- Responsabilidade principal: criar e ler o escopo `./.sandbox/tmp/python_test_suite/<run_id>/` e serializar o estado tipado da rodada.
- Dependencias principais: [src/python_test_suite/models.py](../src/python_test_suite/models.py)
- Acoplamento forte com dominio?: Nao. E persistencia operacional da suite.
- Uso atual observado: Sim. Consumido diretamente pelo runner e pelo fluxo de resume.
- Seguro reutilizar como esta?: Sim. E a fonte certa para evoluir artefatos oficiais da suite.
- Riscos ou limitacoes: documentar ou auditar artefatos antigos fora do state store atual cria leitura errada da rodada atual.
- Sugestao de melhoria: manter toda mudanca de artefato centralizada nessa camada e protegida por testes unitarios da suite.
- Prioridade: Alta

### Plugin pytest da nova suite Python

- Descricao: plugin que coleta e serializa resultados pytest por target dentro do boundary novo, exigindo marker explicito de familia e falhando fechado quando o arquivo coletavel nao declara esse atributo da suite.
- Tags: suite-oficial, pytest, plugin
- Tipo: helper
- Arquivo: [src/python_test_suite/pytest_plugin.py](../src/python_test_suite/pytest_plugin.py)
- Linguagem: Python
- Responsabilidade principal: registrar itens executados, markers e relatorio JSON por target para a rodada atual, sem inferir familia por pasta, nome de arquivo, import ou helper.
- Dependencias principais: [src/python_test_suite/runner.py](../src/python_test_suite/runner.py)
- Acoplamento forte com dominio?: Nao. E adaptador da suite para pytest.
- Uso atual observado: Sim. Ativado pelo runner nos targets pytest.
- Seguro reutilizar como esta?: Sim. Mudancas no contrato de report devem nascer aqui e nos testes da pasta `tests/unit/python_test_suite/`.
- Riscos ou limitacoes: instrumentar pytest fora desse plugin recria duplicidade de coleta; documentar recortes por caminho como se fossem contrato oficial conflita com o comportamento real do runtime.
- Sugestao de melhoria: manter o contrato do report alinhado com `telemetry.json` e `state.json`.
- Prioridade: Alta

## Runtime assincrono e Job Core em Python

### JobEnvelope, JobExecutionContext e JobExecutionResult

- Descricao: contrato minimo e generico do Job Core V1 para representar o envelope recebido pelo worker, o contexto de execucao e o resultado terminal sem importar dominio de ingestao, ETL ou background.
- Tags: job-core, contrato, runtime
- Tipo: contrato
- Arquivo: [src/core/job_core/models.py](../src/core/job_core/models.py)
- Linguagem: Python
- Responsabilidade principal: congelar os campos obrigatorios do envelope e os estados/erros minimos do lifecycle generico.
- Dependencias principais: tipagem Python e validacao do proprio pacote `job_core`.
- Acoplamento forte com dominio?: Nao. O objetivo e justamente evitar acoplamento com um caso de uso especifico.
- Uso atual observado: Sim. Consumido pelo bridge oficial em [src/api/services/async_job_dramatiq.py](../src/api/services/async_job_dramatiq.py) e pelos publishers que montam `QueuedJobEnvelope` para o runtime assincrono.
- Seguro reutilizar como esta?: Sim, quando o fluxo novo realmente precisa entrar no runtime assincrono oficial.
- Riscos ou limitacoes: nao deve virar deposito de regra de negocio; qualquer campo novo precisa provar valor transversal real.
- Sugestao de melhoria: manter os exemplos de publish sempre alinhados a esse contrato para evitar produtores paralelos.
- Prioridade: Alta

### JobHandlerRegistry e JobCoreExecutor

- Descricao: dupla central do runtime generico do Job Core. O registry resolve handlers por `route_kind + dispatch_mode`, e o executor coordena ledger, eventos e resultado terminal sem conhecer RabbitMQ, Dramatiq ou dominio concreto.
- Tags: job-core, executor, registry
- Tipo: service
- Arquivo: [src/core/job_core/executor.py](../src/core/job_core/executor.py)
- Linguagem: Python
- Responsabilidade principal: executar envelopes genericos de forma deterministica e observavel, mantendo separacao entre transporte e caso de uso.
- Dependencias principais: [src/core/job_core/registry.py](../src/core/job_core/registry.py), [src/core/job_core/store.py](../src/core/job_core/store.py), [src/core/job_core/events.py](../src/core/job_core/events.py)
- Acoplamento forte com dominio?: Nao. Ele conhece apenas contratos genericos do Job Core.
- Uso atual observado: Sim. O boundary oficial em [src/api/services/async_job_dramatiq.py](../src/api/services/async_job_dramatiq.py) monta esse executor para entregar o envelope consumido ao handler correto.
- Seguro reutilizar como esta?: Sim, desde que o fluxo novo entre pelo contrato oficial de envelope e nao por um dispatcher paralelo.
- Riscos ou limitacoes: registry e executor nao substituem o transporte; tentar usalos como fila ou scheduler paralelo recria o erro arquitetural que o plano corrigiu.
- Sugestao de melhoria: manter testes de contrato cobrindo ordem de eventos e resolucao de handler sempre que surgirem novos modos de dispatch.
- Prioridade: Alta

### PostgresJobRunStore

- Descricao: implementacao duravel do ledger generico do Job Core, gravando `job_runs` e `job_run_events` no schema `job_core` com pool `psycopg`, retry canonico e fail-close quando `INGESTION_TELEMETRY_DSN` nao existe.
- Tags: job-core, postgres, ledger
- Tipo: service
- Arquivo: [src/core/job_core/postgres_store.py](../src/core/job_core/postgres_store.py)
- Linguagem: Python
- Responsabilidade principal: ser a autoridade duravel do lifecycle generico no runtime oficial do worker.
- Dependencias principais: [src/core/database_connection_manager.py](../src/core/database_connection_manager.py), [src/core/external_retry.py](../src/core/external_retry.py), [src/config/config_api/system_config_manager.py](../src/config/config_api/system_config_manager.py)
- Acoplamento forte com dominio?: Nao. Ele persiste apenas o contrato generico do Job Core.
- Uso atual observado: Sim. O runtime oficial em [src/api/services/async_job_dramatiq.py](../src/api/services/async_job_dramatiq.py) resolve esse store por `from_runtime_environment(...)` e o teste de integracao em [tests/integration/test_03-01-23_job_core_runtime_durable_ledger.py](../tests/integration/test_03-01-23_job_core_runtime_durable_ledger.py) prova a gravacao real no schema `job_core`.
- Seguro reutilizar como esta?: Sim, quando a necessidade for ledger generico de job no runtime oficial.
- Riscos ou limitacoes: nao deve ser tratado como fila, nem como substituto das tabelas canonicas de dominio; ele guarda lifecycle generico, nao estado de negocio.
- Sugestao de melhoria: manter a documentacao de schema e os testes de migracao alinhados sempre que novas colunas do envelope forem realmente necessarias.
- Prioridade: Alta

### OperationalRunReconciliationService

- Descricao: reconciliador canônico compartilhado para runs persistidos do runtime oficial. Ele concentra a reconciliação operacional de ETL e do Job Core, decide entre `stale` e `orphaned` e fecha o lifecycle em estado terminal sem abrir caminho paralelo de manutenção.
- Tags: operational-run, reconciliation, maintenance, job-core
- Tipo: service
- Arquivo: [src/api/services/operational_run_reconciliation_service.py](../src/api/services/operational_run_reconciliation_service.py)
- Linguagem: Python
- Responsabilidade principal: transformar sinais duráveis de ownership e liveness em decisão operacional explícita dentro do corredor canônico de manutenção, sem empurrar a regra para adapters paralelos ou schedulers especializados.
- Dependencias principais: [src/core/job_core/store.py](../src/core/job_core/store.py), [src/core/job_core/events.py](../src/core/job_core/events.py), [src/core/canonical_job_runs_manager.py](../src/core/canonical_job_runs_manager.py)
- Acoplamento forte com dominio?: Nao. O service olha apenas status genéricos, ownership e heartbeat do ledger persistido e dos runs operacionais oficiais.
- Uso atual observado: Sim. O maintenance scheduler oficial em [src/api/startup/runtime_bootstrap.py](../src/api/startup/runtime_bootstrap.py) agenda [src/api/services/job_core_reconciliation_maintenance_job.py](../src/api/services/job_core_reconciliation_maintenance_job.py), que agora delega a reconciliação do Job Core para este service canônico.
- Seguro reutilizar como esta?: Sim, quando o objetivo for reconciliar lifecycle operacional persistido sem criar requeue, fallback ou monitor paralelo.
- Riscos ou limitacoes: nao substitui regra de dominio nem detecta saude de pipelines especializadas; ele fecha apenas o contrato operacional persistido do runtime oficial.
- Sugestao de melhoria: manter a janela `stale_after_seconds` alinhada com o heartbeat real do worker e com os testes de boundary do runtime oficial.
- Prioridade: Alta

### AsyncJobQueuePort e QueuedJobEnvelope

- Descricao: boundary canonico de publicacao assincrona. Padroniza o ato de publicar trabalho para RabbitMQ + Dramatiq usando envelope generico e sem expor publishes especializados por dominio.
- Tags: async-job, envelope, boundary
- Tipo: contrato
- Arquivo: [src/api/services/async_job_queue_port.py](../src/api/services/async_job_queue_port.py)
- Linguagem: Python
- Responsabilidade principal: garantir que produtores publiquem `QueuedJobEnvelope` pelo mesmo trilho oficial, com `correlation_id`, `worker_execution_correlation_id`, `route_kind` e `dispatch_mode` coerentes.
- Dependencias principais: [src/core/job_core/models.py](../src/core/job_core/models.py)
- Acoplamento forte com dominio?: Nao. E o contrato transversal do transporte assincrono.
- Uso atual observado: Sim. Consumido por publishers de ingestao, ETL, scheduler e background, e implementado pelo adapter oficial em [src/api/services/async_job_dramatiq.py](../src/api/services/async_job_dramatiq.py).
- Seguro reutilizar como esta?: Sim. Deve ser a primeira opcao para qualquer novo produtor de job assíncrono.
- Riscos ou limitacoes: criar metodo especializado fora desse boundary reabre caminho paralelo de transporte.
- Sugestao de melhoria: manter exemplos curtos de montagem de envelope por tipo de produtor oficial.
- Prioridade: Alta

## Ingestao e retrieval em Python

### Fabrica central da ingestao

- Descricao: conjunto de factories da camada de ingestao para clients, processors, vector stores e componentes auxiliares, evitando que cada pipeline conheca diretamente todas as classes concretas.
- Tags: ingestao, factory, pipeline
- Tipo: factory
- Arquivo: [src/ingestion_layer/core/factories.py](../src/ingestion_layer/core/factories.py)
- Linguagem: Python
- Responsabilidade principal: orquestrar instanciacao consistente de pecas da ingestao moderna.
- Dependencias principais: [src/core/base_correlation_component.py](../src/core/base_correlation_component.py), [src/core/vector_store_factory.py](../src/core/vector_store_factory.py), factories de data source e processors da ingestao
- Acoplamento forte com dominio?: Sim. Fortemente ligado ao dominio de ingestao, mas util para qualquer novo fluxo dessa camada.
- Uso atual observado: Sim. O proprio modulo reune as fabricas padrao da ingestao moderna e depende da factory canonica de vector store.
- Seguro reutilizar como esta?: Sim, dentro do dominio de ingestao.
- Riscos ou limitacoes: pouco util fora do pipeline de ingestao; nao deve ser levado para fluxos nao relacionados.
- Sugestao de melhoria: separar melhor as fabricas por tipo de componente para reduzir o tamanho do modulo.
- Prioridade: Alta

### wrap_ingestion_write_logging_pool e wrap_ingestion_write_logging_connection

- Descricao: proxies canonicos para logar mutacoes SQL em tabelas `ingestion_*` logo apos `INSERT`, `UPDATE` e `DELETE`, inferindo identificador principal, chaves de contexto e distinguindo falha transitória recuperável de falha terminal sem criar logging paralelo por repositorio.
- Tags: ingestao, observabilidade, banco-de-dados
- Tipo: helper
- Arquivo: [src/ingestion_layer/telemetry/ingestion_write_logging.py](../src/ingestion_layer/telemetry/ingestion_write_logging.py)
- Linguagem: Python
- Responsabilidade principal: instrumentar pools e conexoes `psycopg` para que writes do dominio de ingestao contem a historia certa no log canonico, com foco em tabelas `ingestion_*`.
- Dependencias principais: `psycopg`, [src/telemetry/ingestion/observability_severity.py](../src/telemetry/ingestion/observability_severity.py), [src/ingestion_layer/telemetry/log_vocabulary.py](../src/ingestion_layer/telemetry/log_vocabulary.py), logger canonico do projeto.
- Acoplamento forte com dominio?: Medio. O helper e especifico do slice de ingestao, mas resolve um problema transversal de observabilidade do dominio.
- Uso atual observado: Sim. Consumido por [src/ingestion_layer/telemetry/db_persistence_adapter.py](../src/ingestion_layer/telemetry/db_persistence_adapter.py), [src/ingestion_layer/telemetry/db_runtime.py](../src/ingestion_layer/telemetry/db_runtime.py), [src/telemetry/ingestion/ingestion_runs_repository.py](../src/telemetry/ingestion/ingestion_runs_repository.py), [src/telemetry/ingestion/dataset_lifecycle_repository.py](../src/telemetry/ingestion/dataset_lifecycle_repository.py) e [src/telemetry/job_runs/canonical_job_runs_manager.py](../src/telemetry/job_runs/canonical_job_runs_manager.py).
- Seguro reutilizar como esta?: Sim, sempre que o objetivo for observar mutacoes em tabelas `ingestion_*` sem reinventar proxies de pool ou de conexao.
- Riscos ou limitacoes: ele nao substitui retry nem corrige semantica de negocio; so deve ser usado quando o write real pertence ao dominio de ingestao e o caller ja usa o logger canonico.
- Sugestao de melhoria: manter novos repositorios do slice plugando neste helper em vez de abrir proxies locais por tabela ou por adapter.
- Prioridade: Alta

### log_vocabulary

- Descricao: vocabulário canônico do slice de execução agentic em background, com builder pequeno e formal para impedir que runtime e services reapareçam com `extra` manual ou logger técnico em fluxo correlacionado.
- Tags: background-execution, observabilidade, catalogo
- Tipo: helper
- Arquivo: [src/agentic_layer/background_execution/log_vocabulary.py](../src/agentic_layer/background_execution/log_vocabulary.py)
- Linguagem: Python
- Responsabilidade principal: centralizar os campos locais aprovados do runtime background, como `request_id`, `schedule_id`, `target_id`, `approval_request_id`, `next_run_at`, `cancelled` e `updated`, sempre delegando a montagem final ao builder canônico global.
- Dependencias principais: [src/core/log_canonical_fields.py](../src/core/log_canonical_fields.py) e consumidores do slice de execução em background.
- Acoplamento forte com dominio?: Medio. E específico do runtime background e existe para impedir drift semântico entre runtime, dispatcher e worker handler.
- Uso atual observado: Sim. Consumido por [src/agentic_layer/background_execution/runtime.py](../src/agentic_layer/background_execution/runtime.py) e [src/agentic_layer/background_execution/services.py](../src/agentic_layer/background_execution/services.py).
- Seguro reutilizar como esta?: Sim, sempre que o fluxo pertencer ao runtime background e precisar emitir eventos do slice sem reinventar `event_name` ou payload manual local.
- Riscos ou limitacoes: nao substitui o logger canônico do projeto e nao deve ser usado fora do slice sem evidência de semântica realmente compartilhada.
- Sugestao de melhoria: continuar puxando qualquer novo evento de dispatcher, HIL ou worker do mesmo slice para este builder antes de endurecer o core contra payload paralelo.
- Prioridade: Alta

### log_vocabulary

- Descricao: vocabulário canônico do slice de workflow, com builders formais para eventos de roteamento e execução de nodes sem espalhar `event_name` e chaves locais pelos call sites.
- Tags: workflow, observabilidade, catalogo
- Tipo: helper
- Arquivo: [src/agentic_layer/workflow/log_vocabulary.py](../src/agentic_layer/workflow/log_vocabulary.py)
- Linguagem: Python
- Responsabilidade principal: centralizar os campos aprovados e os builders do slice de workflow para que `RouterNode`, `RuleRouterNode`, `AgentNode`, `IfNode` e consumidores adjacentes emitam logs com o mesmo contrato sem payload manual paralelo.
- Dependencias principais: [src/core/log_canonical_fields.py](../src/core/log_canonical_fields.py) e consumidores do slice de workflow.
- Acoplamento forte com dominio?: Medio. E especifico do runtime de workflow e existe para evitar drift semantico dentro desse slice.
- Uso atual observado: Sim. Consumido por [src/agentic_layer/workflow/nodes/router_node.py](../src/agentic_layer/workflow/nodes/router_node.py), [src/agentic_layer/workflow/nodes/rule_router_node.py](../src/agentic_layer/workflow/nodes/rule_router_node.py), [src/agentic_layer/workflow/nodes/agent_node.py](../src/agentic_layer/workflow/nodes/agent_node.py) e [src/agentic_layer/workflow/nodes/if_node.py](../src/agentic_layer/workflow/nodes/if_node.py).
- Seguro reutilizar como esta?: Sim, sempre que o fluxo pertencer ao runtime de workflow e precisar emitir contexto de node/router sem inventar `event_name` ou aliases locais.
- Riscos ou limitacoes: nao substitui o logger canônico do projeto e nao deve ser usado para slices fora de workflow sem evidência de semântica realmente compartilhada.
- Sugestao de melhoria: continuar migrando os demais nodes e runtime helpers de workflow para este builder antes de endurecer o core contra `structured_noncanonical`.
- Prioridade: Alta

### log_vocabulary

- Descricao: vocabulário canônico do slice de canais, com builder formal para boundaries HTTP e logs de endpoint do `channel_router` sem depender de `extra` ad hoc por rota.
- Tags: channels, api, observabilidade, catalogo
- Tipo: helper
- Arquivo: [src/channel_layer/log_vocabulary.py](../src/channel_layer/log_vocabulary.py)
- Linguagem: Python
- Responsabilidade principal: centralizar os campos aprovados do slice channel e montar payloads canônicos para endpoints, filas e identificadores operacionais desse domínio sem criar contrato paralelo no router.
- Dependencias principais: [src/core/log_canonical_fields.py](../src/core/log_canonical_fields.py) e consumidores de [src/api/routers/channel_router.py](../src/api/routers/channel_router.py).
- Acoplamento forte com dominio?: Medio. E específico da camada multicanal e existe para impedir drift semântico entre router, worker e integrações do slice.
- Uso atual observado: Sim. Consumido por [src/api/routers/channel_router.py](../src/api/routers/channel_router.py).
- Seguro reutilizar como esta?: Sim, sempre que o fluxo pertencer ao slice channel e precisar emitir `event_name` estável com contexto de endpoint/canal sem campos arbitrários.
- Riscos ou limitacoes: ainda não cobre 100% dos eventos do `channel_router`; por enquanto o primeiro boundary migrado é o helper `_log_endpoint_start`.
- Sugestao de melhoria: expandir o uso do builder para os logs de sucesso/erro do `channel_router` e depois para worker/runtime do mesmo slice antes de endurecer o core.
- Prioridade: Alta

### log_vocabulary

- Descricao: vocabulário canônico mínimo do slice de termos populares de consulta, criado para impedir mini-schema local no serviço de telemetria de query e para manter o `event_name` sob o mesmo contrato global.
- Tags: query, observabilidade, catalogo
- Tipo: helper
- Arquivo: [src/telemetry/query/log_vocabulary.py](../src/telemetry/query/log_vocabulary.py)
- Linguagem: Python
- Responsabilidade principal: oferecer um builder oficial do slice de query que continue delegando ao catálogo global, sem reabrir `_build_log_extra` nem payload paralelo no serviço de termos populares.
- Dependencias principais: [src/core/log_canonical_fields.py](../src/core/log_canonical_fields.py) e consumidores do slice de query.
- Acoplamento forte com dominio?: Baixo. O builder e pequeno, mas existe para dar um ponto oficial de extensão ao domínio de query antes de novos logs surgirem ad hoc.
- Uso atual observado: Sim. Consumido por [src/telemetry/query/popular_query_terms_service.py](../src/telemetry/query/popular_query_terms_service.py).
- Seguro reutilizar como esta?: Sim, sempre que o fluxo pertencer ao slice de query telemetry e precisar emitir eventos do domínio sem inventar schema local.
- Riscos ou limitacoes: como o slice ainda e pequeno, ha risco de o builder ser ignorado em futuras expansoes se o gate estrutural nao continuar cobrindo esse uso.
- Sugestao de melhoria: manter qualquer novo evento de query telemetry entrando por este builder antes de adicionar campos próprios em call sites.
- Prioridade: Alta

- Descricao: dicionario central do slice de ingestao para campos canonicos e familias de `event_name`, cobrindo documento, run, HTTP, worker, runtime async, chunking PDF, mutacoes SQL e vetor sem criar logger paralelo.
- Tags: ingestao, observabilidade, catalogo
- Tipo: helper
- Arquivo: [src/ingestion_layer/telemetry/log_vocabulary.py](../src/ingestion_layer/telemetry/log_vocabulary.py)
- Linguagem: Python
- Responsabilidade principal: dar uma autoridade unica para o vocabulário observável da ingestao PDF, permitindo que diferentes emitters contem a mesma historia operacional com os mesmos nomes de evento e os mesmos campos base.
- Dependencias principais: `pathlib`, `urllib.parse`, `uuid`, logger canonico do projeto e consumidores do slice de ingestao.
- Acoplamento forte com dominio?: Medio. Ele e especifico da ingestao e justamente existe para evitar drift entre os boundaries desse dominio.
- Uso atual observado: Sim. Consumido por [src/api/services/ingestion_http_execution_services.py](../src/api/services/ingestion_http_execution_services.py), [src/api/services/async_job_worker_payload_executor.py](../src/api/services/async_job_worker_payload_executor.py), [src/api/services/rag_async_execution_service.py](../src/api/services/rag_async_execution_service.py), [src/ingestion_layer/file_pipeline_services.py](../src/ingestion_layer/file_pipeline_services.py), [src/ingestion_layer/processors/pdf_chunking_service.py](../src/ingestion_layer/processors/pdf_chunking_service.py), [src/ingestion_layer/processors/txt_processor.py](../src/ingestion_layer/processors/txt_processor.py), [src/ingestion_layer/text_fallback_processor.py](../src/ingestion_layer/text_fallback_processor.py), [src/ingestion_layer/telemetry/ingestion_write_logging.py](../src/ingestion_layer/telemetry/ingestion_write_logging.py), [src/ingestion_layer/telemetry/manifest_manager.py](../src/ingestion_layer/telemetry/manifest_manager.py), [src/telemetry/ingestion/operational_insights.py](../src/telemetry/ingestion/operational_insights.py), [src/telemetry/ingestion/ingestion_runs_repository.py](../src/telemetry/ingestion/ingestion_runs_repository.py) e [src/ingestion_layer/vector_stores/qdrant_client.py](../src/ingestion_layer/vector_stores/qdrant_client.py).
- Seguro reutilizar como esta?: Sim, sempre que o fluxo pertencer ao slice de ingestao e precise usar o vocabulário canônico já aprovado em vez de inventar chaves ou `event_name` locais.
- Riscos ou limitacoes: nao substitui o logger canonico, nao e um catalogo global do repositorio inteiro e nao deve ser usado para dominios fora da ingestao sem evidência de semantica realmente compartilhada.
- Sugestao de melhoria: manter novas fases da ingestao entrando aqui antes de adicionarem semantica propria de log.
- Prioridade: Alta

### log_vocabulary

- Descricao: vocabulário canônico dos runners de processo, criado para consolidar o bootstrap de API, worker e scheduler sob um único builder e remover aliases locais como `erro`, `signal`, `locales_tentados` e `scheduler_block_reason` dos payloads.
- Tags: runners, bootstrap, observabilidade, catalogo
- Tipo: helper
- Arquivo: [app/runners/log_vocabulary.py](../app/runners/log_vocabulary.py)
- Linguagem: Python
- Responsabilidade principal: centralizar os campos operacionais aprovados para processos long-lived, incluindo papel do processo, locale, sinais, readiness, bootstrap, shutdown e nomes de arquivos de log.
- Dependencias principais: [src/core/log_canonical_fields.py](../src/core/log_canonical_fields.py) e consumidores de `app/runners`.
- Acoplamento forte com dominio?: Medio. O helper e específico do bootstrap de processos da aplicação e existe para impedir drift semântico entre API, worker, scheduler e suporte de sinais.
- Uso atual observado: Sim. Consumido por [app/runners/api_runner.py](../app/runners/api_runner.py), [app/runners/signal_support.py](../app/runners/signal_support.py), [app/runners/worker_runner.py](../app/runners/worker_runner.py) e [app/runners/scheduler_runner.py](../app/runners/scheduler_runner.py).
- Seguro reutilizar como esta?: Sim, sempre que o fluxo pertencer aos runners oficiais da aplicação e precisar emitir eventos de bootstrap, readiness, locale ou shutdown sem abrir payload local paralelo.
- Riscos ou limitacoes: nao substitui logs de dominio fora dos runners e nao deve ser usado para qualquer processo arbitrário sem semântica compartilhada com esse bootstrap.
- Sugestao de melhoria: manter novas etapas de startup e shutdown dos runners oficiais plugando aqui antes de crescerem em helpers locais independentes.
- Prioridade: Alta

### IngestionObservabilitySeverityPolicy

- Descricao: policy pequena e compartilhada para classificar severidade observavel no slice de ingestao, mantendo coerencia entre falha transitória recuperável de write, bloqueio seguro do fan-out e indisponibilidade do plano de controle.
- Tags: ingestao, observabilidade, severidade
- Tipo: helper
- Arquivo: [src/telemetry/ingestion/observability_severity.py](../src/telemetry/ingestion/observability_severity.py)
- Linguagem: Python
- Responsabilidade principal: centralizar a regua minima de `info`, `warning` e `error` usada pelos pontos criticos de observabilidade da ingestao tocados neste ciclo.
- Dependencias principais: tipagem Python e os consumidores do slice, como [src/ingestion_layer/telemetry/ingestion_write_logging.py](../src/ingestion_layer/telemetry/ingestion_write_logging.py) e [src/services/document_fanout_execution_gate.py](../src/services/document_fanout_execution_gate.py).
- Acoplamento forte com dominio?: Medio. E especifica da ingestao, mas foi desenhada para evitar drift entre modulos criticos do mesmo dominio.
- Uso atual observado: Sim. Reutilizada por [src/ingestion_layer/telemetry/ingestion_write_logging.py](../src/ingestion_layer/telemetry/ingestion_write_logging.py) e [src/services/document_fanout_execution_gate.py](../src/services/document_fanout_execution_gate.py).
- Seguro reutilizar como esta?: Sim, quando a necessidade for classificar severidade observavel desses mesmos tipos de evento no dominio de ingestao.
- Riscos ou limitacoes: nao e uma taxonomia global do repositorio; expandir essa policy para outros dominios so faz sentido se houver evidencia de semantica realmente compartilhada.
- Sugestao de melhoria: se novos eventos do slice precisarem da mesma regua, plugar aqui antes de criar nova convencao local.
- Prioridade: Media

### FileDiscoveryFactory e abstraacoes de descoberta de arquivo

- Descricao: conjunto reutilizavel para descobrir arquivos por include/exclude, validar padroes e encapsular a estrategia de localizacao sem espalhar `glob` e `fnmatch` em pipelines.
- Tags: discovery, arquivos, ingestao
- Tipo: factory
- Arquivo: [src/ingestion_layer/core/file_discovery.py](../src/ingestion_layer/core/file_discovery.py)
- Linguagem: Python
- Responsabilidade principal: montar estrategias de descoberta de arquivos com contrato comum.
- Dependencias principais: `glob`, `fnmatch`, [src/core/base_correlation_component.py](../src/core/base_correlation_component.py)
- Acoplamento forte com dominio?: Medio. Nasceu na ingestao, mas o problema resolvido e generico.
- Uso atual observado: Sim. O proprio modulo expoe `FileDiscoveryFactory.create_discovery(...)` e o padrao de descoberta e parte do trilho oficial da ingestao.
- Seguro reutilizar como esta?: Sim, principalmente em fluxos de coleta e varredura de assets.
- Riscos ou limitacoes: ainda conhece configuracoes com cara de ingestao; pode exigir adaptacao para fontes nao baseadas em arquivo.
- Sugestao de melhoria: separar o contrato generico da implementacao especifica da ingestao.
- Prioridade: Media

### UpsertStrategyFactory e estrategias de persistencia vetorial

- Descricao: camada que encapsula o processo de transformar chunks em payloads vetoriais, controlar duplicacao, montar vetores multimodais e delegar a estrategia correta de upsert.
- Tags: upsert, vector-store, multimodal
- Tipo: factory
- Arquivo: [src/ingestion_layer/core/upsert_strategy.py](../src/ingestion_layer/core/upsert_strategy.py)
- Linguagem: Python
- Responsabilidade principal: padronizar persistencia vetorial e normalizacao de payload antes de gravar no backend.
- Dependencias principais: normalizadores de chunk, hashes de conteudo, BM25, [src/shared/rag_contracts/interfaces.py](../src/shared/rag_contracts/interfaces.py)
- Acoplamento forte com dominio?: Sim. E um ponto de reuso forte dentro do dominio de ingestao vetorial.
- Uso atual observado: Sim. Instanciado por [src/ingestion_layer/vector_stores/qdrant_client.py](../src/ingestion_layer/vector_stores/qdrant_client.py).
- Seguro reutilizar como esta?: Sim, para novos backends ou novos modos de persistencia do pipeline vetorial.
- Riscos ou limitacoes: modulo grande e denso; risco de acumular responsabilidade demais.
- Sugestao de melhoria: dividir serializacao, deduplicacao e estrategia de envio em componentes menores.
- Prioridade: Alta

### BaseVectorStore

- Descricao: classe base das implementacoes de vector store do projeto, com politicas de lifecycle, `if_exists`, BM25, preview mode e contratos comuns de ingestao e leitura.
- Tags: vector-store, abstracao, lifecycle
- Tipo: classe
- Arquivo: [src/ingestion_layer/vector_stores/base.py](../src/ingestion_layer/vector_stores/base.py)
- Linguagem: Python
- Responsabilidade principal: concentrar comportamento compartilhado entre providers de vector store.
- Dependencias principais: contratos de vector store, telemetria de ingestao, dataset lifecycle, BM25 runtime
- Acoplamento forte com dominio?: Sim. E o nucleo da camada vetorial.
- Uso atual observado: Sim. Serve de base para a familia de vector stores da ingestao e conversa diretamente com contratos compartilhados do projeto.
- Seguro reutilizar como esta?: Sim, para novos providers ou extensoes do ciclo de vida vetorial.
- Riscos ou limitacoes: mexer nessa base impacta varios backends ao mesmo tempo.
- Sugestao de melhoria: manter testes de contrato por backend sempre alinhados a qualquer extensao do contrato base.
- Prioridade: Alta

### RetrieverSearchConfig e BaseUnifiedStoreRetriever

- Descricao: contrato unificado para retrievers sincronos e assincronos, com normalizacao de configuracao de busca e adaptacao de stores diferentes para uma superficie comum.
- Tags: retriever, busca, contrato
- Tipo: contrato
- Arquivo: [src/ingestion_layer/vector_stores/retriever_contract.py](../src/ingestion_layer/vector_stores/retriever_contract.py)
- Linguagem: Python
- Responsabilidade principal: evitar que cada retriever reinvente configuracao de busca e interface publica.
- Dependencias principais: `asyncio`, `dataclasses`, `Protocol`
- Acoplamento forte com dominio?: Medio. Forte no dominio de retrieval, baixo fora dele.
- Uso atual observado: Sim. Faz parte da infraestrutura compartilhada de retrieval da camada vetorial.
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: exige que implementacoes realmente respeitem a interface sincronica e assincrona.
- Sugestao de melhoria: adicionar mais exemplos de adapters prontos por provider.
- Prioridade: Alta

### resolve_hospitality_enabled_from_yaml

- Descricao: helper compartilhado dos pipelines Apify para decidir se o ETL hospitality esta habilitado, priorizando `extract_transform_load.hospitality.enabled` quando o YAML explicita esse bloco e caindo no checker base apenas quando o YAML nao decide o assunto.
- Tags: etl, apify, hospitality
- Tipo: helper
- Arquivo: [src/etl_layer/providers/apify/base_apify_pipeline.py](../src/etl_layer/providers/apify/base_apify_pipeline.py)
- Linguagem: Python
- Responsabilidade principal: impedir drift entre Booking, Hotels.com e TripAdvisor na leitura do gate tecnico de hospitality.
- Dependencias principais: [src/ingestion_layer/hospitality/__init__.py](../src/ingestion_layer/hospitality/__init__.py), providers Apify do slice ETL.
- Acoplamento forte com dominio?: Medio. E especifico do slice ETL Apify, mas resolve uma regra tecnica que era repetida em tres providers.
- Uso atual observado: Sim. Consumido por [src/etl_layer/providers/apify/booking_pipeline.py](../src/etl_layer/providers/apify/booking_pipeline.py), [src/etl_layer/providers/apify/hotelscom_pipeline.py](../src/etl_layer/providers/apify/hotelscom_pipeline.py) e [src/etl_layer/providers/apify/tripadvisor_pipeline.py](../src/etl_layer/providers/apify/tripadvisor_pipeline.py).
- Seguro reutilizar como esta?: Sim, sempre que outro provider Apify precisar obedecer a mesma regra de enablement de hospitality.
- Riscos ou limitacoes: nao substitui validacao de infraestrutura nem a factory do repositorio hospitality; ele resolve apenas o gate de configuracao.
- Sugestao de melhoria: se surgirem outros providers Apify de hospitality, plugar aqui em vez de reabrir wrapper local.
- Prioridade: Alta

## Compatibilidades residuais monitoradas em Python

### Fachadas residuais de QA, BM25 e memoria agentic antiga

- Descricao: conjunto de modulos mantidos por compatibilidade de importacao e por testes, mas que nao representam o boundary oficial atual do produto. O objetivo desses modulos e preservar contrato residual monitorado, nao servir como ponto de partida para features novas.
- Tags: compatibilidade, residual, imports
- Tipo: contrato residual
- Arquivos principais: [src/qa_layer/rag_engine/adapter.py](../src/qa_layer/rag_engine/adapter.py), [src/qa_layer/json_rag/__init__.py](../src/qa_layer/json_rag/__init__.py), [src/ingestion_layer/core/bm25_config_resolver.py](../src/ingestion_layer/core/bm25_config_resolver.py), [src/ingestion_layer/core/generic_keyword_extractor.py](../src/ingestion_layer/core/generic_keyword_extractor.py), [src/ingestion_layer/core/vocabulary_policy.py](../src/ingestion_layer/core/vocabulary_policy.py), [src/ingestion_layer/core/rake_keyword_extractor.py](../src/ingestion_layer/core/rake_keyword_extractor.py), [src/agentic_layer/memory/implementations.py](../src/agentic_layer/memory/implementations.py), [src/agentic_layer/memory/memory_manager.py](../src/agentic_layer/memory/memory_manager.py)
- Linguagem: Python
- Responsabilidade principal: deixar explicito no proprio codigo que essas superfices permanecem fora do runtime oficial e dependem de decisao explicita antes de qualquer remocao contratual.
- Dependencias principais: boundary oficial de QA em [src/qa_layer/content_qa_system.py](../src/qa_layer/content_qa_system.py), JSON canonico em [src/ingestion_layer/json_advanced.py](../src/ingestion_layer/json_advanced.py), BM25 oficial em [src/core/bm25_runtime/config_resolver.py](../src/core/bm25_runtime/config_resolver.py) e memoria oficial nova em [src/agentic_layer/supervisor/memory_factory.py](../src/agentic_layer/supervisor/memory_factory.py).
- Acoplamento forte com dominio?: Medio. Essas superfices ainda carregam nomes historicos do dominio, mas seu papel atual e de compatibilidade monitorada.
- Uso atual observado: Sim, principalmente por imports de pacote, testes de contrato e compatibilidade interna controlada.
- Seguro reutilizar como esta?: Nao para codigo novo. Em manutencao, reutilize apenas quando a tarefa for explicitamente manter compatibilidade residual existente.
- Riscos ou limitacoes: tratar essas superfices como nucleo oficial reabre legado, confunde onboarding e pode religar contratos que o boundary novo ja desligou.
- Sugestao de melhoria: quando houver decisao explicita de produto e evidencia de consumidores, promover ou remover cada superficie individualmente, nunca por inferencia fraca.
- Prioridade: Alta

## Seguranca e configuracao em Python

### SecurityKeys resolver

- Descricao: resolvedor central de `security_keys`, placeholders e segredos com suporte a fallback controlado para `.env`, expansao recursiva e verificacao de placeholders nao resolvidos.
- Tags: seguranca, secrets, yaml
- Tipo: helper
- Arquivo: [src/security/security_keys_resolver.py](../src/security/security_keys_resolver.py)
- Linguagem: Python
- Responsabilidade principal: resolver segredos e placeholders do YAML do jeito canonico do projeto.
- Dependencias principais: configuracao do sistema, logging canonico
- Acoplamento forte com dominio?: Nao. E um contrato transversal de configuracao segura.
- Uso atual observado: Sim. Baseia [src/security/credential_manager.py](../src/security/credential_manager.py), [src/core/message_history_factory.py](../src/core/message_history_factory.py) e outros fluxos que precisam ler segredos do YAML.
- Seguro reutilizar como esta?: Sim. Deve ser o primeiro ponto a consultar quando um fluxo receber YAML com placeholders.
- Riscos ou limitacoes: qualquer caminho alternativo para resolver segredo cria inconsistencia arquitetural.
- Sugestao de melhoria: documentar exemplos de placeholders validos e invalidos.
- Prioridade: Alta

### CredentialManager

- Descricao: gerenciador central de credenciais baseado no YAML e no resolvedor de segredos, evitando espalhar leitura de chaves, prompts padrao e lookup de recursos por varios modulos.
- Tags: credenciais, seguranca, config
- Tipo: service
- Arquivo: [src/security/credential_manager.py](../src/security/credential_manager.py)
- Linguagem: Python
- Responsabilidade principal: entregar credenciais e configuracoes sensiveis de forma consistente para runtime e integracoes.
- Dependencias principais: [src/security/security_keys_resolver.py](../src/security/security_keys_resolver.py), cache em memoria, [src/shared/prompts/default_prompts.py](../src/shared/prompts/default_prompts.py)
- Acoplamento forte com dominio?: Medio. E transversal, mas conhece convencoes especificas do projeto.
- Uso atual observado: Sim. Chamado diretamente por componentes como [src/core/message_history_factory.py](../src/core/message_history_factory.py) e sustenta resolucao de segredo em varios fluxos.
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: muito conhecimento acumulado num unico modulo; alteracoes aqui podem quebrar varios caminhos de autenticacao.
- Sugestao de melhoria: continuar extraindo responsabilidades de lookup por tipo de recurso.
- Prioridade: Alta

### ProjectFileStorageFactory e providers de storage de arquivo

- Descricao: camada de storage plugavel para arquivos persistidos, com providers S3 compativel e Azure Blob, retry e objetos estruturados de arquivo armazenado.
- Tags: storage, s3, azure
- Tipo: factory
- Arquivo: [src/security/project_file_storage.py](../src/security/project_file_storage.py)
- Linguagem: Python
- Responsabilidade principal: abstrair leitura e escrita de arquivos persistidos do projeto sem expor SDK concreto ao dominio.
- Dependencias principais: `boto3` opcional, SDK Azure Blob opcional, `tenacity`, logging do projeto
- Acoplamento forte com dominio?: Medio. E generico na infraestrutura, mas hoje aparece ligado a artefatos de projeto.
- Uso atual observado: Sim. Usado por [src/api/services/dnit_projects_service.py](../src/api/services/dnit_projects_service.py) e coberto por [tests/unit/test_02-03-10_dnit_projects_router.py](../tests/unit/test_02-03-10_dnit_projects_router.py).
- Seguro reutilizar como esta?: Sim, quando a necessidade for storage de arquivo persistido com o contrato ja suportado.
- Riscos ou limitacoes: ainda conhece convencoes do projeto e nao substitui um storage universal para qualquer binario do sistema.
- Sugestao de melhoria: isolar ainda mais naming e convencoes de path por dominio.
- Prioridade: Media

### SessionCacheFactory e caches de sessao

- Descricao: familia de caches de sessao efemera com implementacoes em memoria, Redis e arquivo, mais fabrica central para escolher o backend sem espalhar condicionais de infraestrutura.
- Tags: sessao, cache, redis
- Tipo: factory
- Arquivo: [src/security/session_cache.py](../src/security/session_cache.py)
- Linguagem: Python
- Responsabilidade principal: guardar estado temporario de sessao e chaves efemeras.
- Dependencias principais: `cachetools`, Redis, filesystem local controlado
- Acoplamento forte com dominio?: Baixo. Infraestrutura de sessao.
- Uso atual observado: Sim. Usado por [src/api/security/federated_session_store.py](../src/api/security/federated_session_store.py), [src/api/security/totp_activation_cache.py](../src/api/security/totp_activation_cache.py), [src/security/session_key_manager.py](../src/security/session_key_manager.py) e testado em [tests/unit/test_02-05-71_session_cache.py](../tests/unit/test_02-05-71_session_cache.py).
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: backend de arquivo pode nao servir em ambientes sem filesystem persistente; o chamador ainda precisa definir TTL e limpeza corretamente.
- Sugestao de melhoria: documentar quando escolher cada backend em API, worker e desenvolvimento local.
- Prioridade: Alta

### RedisManager, redis_ok e get_redis_connection

- Descricao: trilho canonico para bootstrap, health check e obtencao de conexao Redis, evitando que cada modulo abra client proprio ou replique verificacao de disponibilidade.
- Tags: redis, cache, infraestrutura
- Tipo: service
- Arquivo: [src/core/cache/redis/redis_manager.py](../src/core/cache/redis/redis_manager.py)
- Linguagem: Python
- Responsabilidade principal: centralizar readiness, pool e acesso compartilhado ao Redis do projeto.
- Dependencias principais: `redis`, configuracao do projeto, logging canonico
- Acoplamento forte com dominio?: Nao. Infraestrutura transversal.
- Uso atual observado: Sim. Consumido por [src/security/session_cache.py](../src/security/session_cache.py), [src/channel_layer/queue.py](../src/channel_layer/queue.py), [src/api/startup/runtime_bootstrap.py](../src/api/startup/runtime_bootstrap.py), [src/api/services/async_job_runtime_support_store.py](../src/api/services/async_job_runtime_support_store.py), [src/core/redis_mysql_handler.py](../src/core/redis_mysql_handler.py) e [src/workers/log_mysql_worker.py](../src/workers/log_mysql_worker.py).
- Seguro reutilizar como esta?: Sim. Deve ser o ponto de entrada padrao para conexao Redis nova no repositorio.
- Riscos ou limitacoes: nao substitui a definicao de chaves, TTL, locking ou topologia de filas; o chamador ainda precisa respeitar o contrato do dominio.
- Sugestao de melhoria: manter excecoes reais pequenas e explicitas; se um modulo nao puder usar esse trilho, isso precisa aparecer no inventario e nos testes.
- Prioridade: Alta

## YAML-first e assembly agentic em Python

### ConfigurationFactory

- Descricao: factory canonica para materializar configuracoes YAML do produto a partir de payload, arquivo ou conteudo inline fora do editor AST agentic.
- Tags: yaml-first, config, factory
- Tipo: factory
- Arquivo: [src/config/config_cli/configuration_factory.py](../src/config/config_cli/configuration_factory.py)
- Linguagem: Python
- Responsabilidade principal: criar configuracoes operacionais reutilizaveis do runtime sem abrir parse local com `yaml.safe_load` em cada helper ou boundary.
- Dependencias principais: loaders e validadores de configuracao do projeto, resolucao de placeholders e segredos
- Acoplamento forte com dominio?: Medio. E infraestrutura de configuracao do produto.
- Uso atual observado: Sim. Consumido por [src/api/routers/config_resolution.py](../src/api/routers/config_resolution.py), [src/api/routers/config_contract_router.py](../src/api/routers/config_contract_router.py) e [src/api/service_api.py](../src/api/service_api.py).
- Seguro reutilizar como esta?: Sim, principalmente em payloads e helpers fora do boundary HTTP.
- Riscos ou limitacoes: nao substitui a AST agentic nem deve virar editor generico de YAML agentic.
- Sugestao de melhoria: manter call sites de payload e helper convergidos aqui em vez de reabrir parse cru local.
- Prioridade: Alta

### resolve_yaml_configuration

- Descricao: helper canonico de boundary HTTP para resolver YAML recebido por endpoints, apoiado pelo `ConfigurationFactory` e pelas regras oficiais de entrada do runtime.
- Tags: yaml-first, endpoint, boundary
- Tipo: helper
- Arquivo: [src/api/routers/config_resolution.py](../src/api/routers/config_resolution.py)
- Linguagem: Python
- Responsabilidade principal: unificar a resolucao de YAML no boundary HTTP antes de entregar configuracao pronta ao restante do fluxo.
- Dependencias principais: [src/config/config_cli/configuration_factory.py](../src/config/config_cli/configuration_factory.py), resolucao de payloads e utilitarios de endpoint
- Acoplamento forte com dominio?: Medio. E um helper de boundary, nao um parser isolado.
- Uso atual observado: Sim. Consumido por [src/channel_layer/config_resolver.py](../src/channel_layer/config_resolver.py), [src/api/routers/admin/admin_runtime_router.py](../src/api/routers/admin/admin_runtime_router.py), [src/api/routers/workflow_router.py](../src/api/routers/workflow_router.py), [src/api/routers/logs_router.py](../src/api/routers/logs_router.py), [src/api/services/worker_bootstrap.py](../src/api/services/worker_bootstrap.py) e varios routers/servicos HTTP do runtime.
- Seguro reutilizar como esta?: Sim, quando o fluxo realmente estiver resolvendo YAML no contrato oficial de endpoint.
- Riscos ou limitacoes: usar esse helper fora do boundary HTTP pode esconder responsabilidades de origem e mascarar um contrato que deveria ser local ao chamador.
- Sugestao de melhoria: manter explicita a fronteira entre endpoint HTTP (`resolve_yaml_configuration`) e helper/payload fora do HTTP (`ConfigurationFactory`).
- Prioridade: Alta

### AgenticAssemblyService

- Descricao: orquestrador principal do fluxo canonico de AST agentic, cobrindo parse, validacao, montagem, confirmacao, reparo e diffs sem criar caminhos paralelos para editar YAML agentic.
- Tags: yaml-first, ast, agentic
- Tipo: service
- Arquivo: [src/config/agentic_assembly/assembly_service.py](../src/config/agentic_assembly/assembly_service.py)
- Linguagem: Python
- Responsabilidade principal: centralizar o ciclo completo de montagem e validacao do documento agentic.
- Dependencias principais: parsers, validators, compiler, schema service, diff preview, `YamlLoader`
- Acoplamento forte com dominio?: Sim. E o contrato central do ecossistema agentic YAML-first.
- Uso atual observado: Sim. Exposto pelo pacote [src/config/agentic_assembly/__init__.py](../src/config/agentic_assembly/__init__.py), usado por [src/api/routers/config_assembly_router.py](../src/api/routers/config_assembly_router.py) e fortemente coberto por [tests/unit/test_02-01-78_agentic_assembly_service.py](../tests/unit/test_02-01-78_agentic_assembly_service.py).
- Seguro reutilizar como esta?: Sim. Deve ser o caminho obrigatorio para fluxos agentic.
- Riscos ou limitacoes: muito central; qualquer bypass cria drift entre AST, YAML e validacao.
- Sugestao de melhoria: manter este documento e a documentacao agentic sempre sincronizados com testes de contrato.
- Prioridade: Alta

### YamlLoader

- Descricao: loader do assembly agentic para ler YAML de path ou payload e inspecionar blocos top-level dentro do fluxo AST, sem se confundir com o resolvedor oficial de endpoint do produto.
- Tags: yaml, parser, loader
- Tipo: parser
- Arquivo: [src/config/agentic_assembly/parsers/yaml_loader.py](../src/config/agentic_assembly/parsers/yaml_loader.py)
- Linguagem: Python
- Responsabilidade principal: padronizar leitura e decomposicao do YAML dentro do assembly.
- Dependencias principais: parser YAML do projeto
- Acoplamento forte com dominio?: Medio. Nasceu no assembly agentic, mas o problema resolvido e bem generico.
- Uso atual observado: Sim. Consumido por [src/config/agentic_assembly/assembly_service.py](../src/config/agentic_assembly/assembly_service.py) e [src/config/agentic_assembly/canonical_yaml_editor.py](../src/config/agentic_assembly/canonical_yaml_editor.py).
- Seguro reutilizar como esta?: Sim, principalmente em fluxos agentic.
- Riscos ou limitacoes: fora do assembly agentic ele nao e o trilho oficial do runtime HTTP e pode reabrir ambiguidade se for promovido como resolvedor de produto.
- Sugestao de melhoria: manter explicito no inventario e nos testes que endpoints HTTP usam `resolve_yaml_configuration`, enquanto payloads e helpers fora do HTTP usam `ConfigurationFactory`.
- Prioridade: Alta

## Agentic e runtime interno em Python

### shared_helpers do supervisor e do Workflow

- Descricao: helper compartilhado para convergencia tecnica entre runtimes agentic irmaos, centralizando deduplicacao de `tools_library` e normalizacao do contrato de memoria aceito pelos runtimes oficiais.
- Tags: agentic, workflow, supervisor
- Tipo: helper
- Arquivo: [src/agentic_layer/supervisor/shared_helpers.py](../src/agentic_layer/supervisor/shared_helpers.py)
- Linguagem: Python
- Responsabilidade principal: evitar drift silencioso entre DeepAgent e Workflow em regras pequenas, mas estruturais, de preparo do runtime.
- Dependencias principais: modelos do slice supervisor e contratos canônicos de memoria/checkpointer.
- Acoplamento forte com dominio?: Medio. E transversal dentro do dominio agentic, mas nao deve vazar para fora dele.
- Uso atual observado: Sim. Consumido por [src/agentic_layer/workflow/agent_workflow.py](../src/agentic_layer/workflow/agent_workflow.py) e por componentes do slice supervisor.
- Seguro reutilizar como esta?: Sim, sempre que a necessidade for preparar `tools_library` ou memoria canônica para DeepAgent/Workflow.
- Riscos ou limitacoes: a normalizacao compartilhada segue o contrato canônico atual; se um runtime precisar regra diferente por requisito real, a divergencia precisa ficar pequena, explicita e coberta por teste.
- Sugestao de melhoria: manter testes estruturais cobrindo o uso pelo Workflow para impedir retorno de helpers locais duplicados.
- Prioridade: Alta

### local_tool_catalog do MCP agentic

- Descricao: helper compartilhado para coletar ids MCP declarados no YAML e gerar entradas sintéticas canônicas de catálogo, evitando que assembly, resolvers de runtime e slice MCP reinventem a mesma regra em paralelo.
- Tags: mcp, catalogo-efetivo, agentic
- Tipo: helper
- Arquivo: [src/agentic_layer/mcp/local_tool_catalog.py](../src/agentic_layer/mcp/local_tool_catalog.py)
- Linguagem: Python
- Responsabilidade principal: centralizar a leitura de local_mcp_configuration.tools, deduplicar ids MCP declarados e produzir entradas sintéticas compatíveis com tools_library e com o catálogo efetivo do assembly.
- Dependencias principais: tipagem Python e contratos leves de dicionário; não depende do client MCP nem do ToolsFactory.
- Acoplamento forte com dominio?: Medio. E transversal ao slice agentic/MCP, mas continua isolado do transporte e do carregamento real das tools.
- Uso atual observado: Sim. Consumido por [src/config/agentic_assembly/tool_resolver.py](../src/config/agentic_assembly/tool_resolver.py), [src/agentic_layer/workflow/config_resolver.py](../src/agentic_layer/workflow/config_resolver.py), [src/agentic_layer/supervisor/config_resolver.py](../src/agentic_layer/supervisor/config_resolver.py), [src/config/agentic_assembly/validators/workflow_semantic_validator.py](../src/config/agentic_assembly/validators/workflow_semantic_validator.py) e [src/agentic_layer/mcp/mcp_config_resolver.py](../src/agentic_layer/mcp/mcp_config_resolver.py).
- Seguro reutilizar como esta?: Sim, sempre que a necessidade for refletir local_mcp_configuration.tools no catálogo efetivo ou no tools_library sem abrir uma implementação local paralela.
- Riscos ou limitacoes: ele só representa declaração e catálogo sintético; não resolve conexão, não carrega tools reais e não substitui o filtro posterior do MCPToolsResolver.
- Sugestao de melhoria: manter os testes estruturais cobrindo assembly, validator e runtime juntos para impedir drift entre catálogo sintético e seleção real de tools MCP.
- Prioridade: Alta

### tool_logging_support

- Descricao: helper compartilhado para logging resiliente de tools agentic, encapsulando o logger correlacionado com fallback nulo seguro para impedir falha funcional secundaria quando a observabilidade falha.
- Tags: agentic, tools, logging
- Tipo: helper
- Arquivo: [src/agentic_layer/tools/tool_logging_support.py](../src/agentic_layer/tools/tool_logging_support.py)
- Linguagem: Python
- Responsabilidade principal: expor `ResilientToolLogger` e `build_resilient_tool_logger` para que o boundary oficial da tool continue registrando eventos sem recorrer a `print` ou a `logger = None` no caminho principal.
- Dependencias principais: factory de logger correlacionado recebida pelo call site, `_NullToolLogger` e `contextlib.suppress`.
- Acoplamento forte com dominio?: Medio. E transversal dentro do dominio agentic de tools, mas nao deve virar logger generico do produto fora desse slice.
- Uso atual observado: Sim. Consumido por [src/agentic_layer/tools/system_tools/human_input.py](../src/agentic_layer/tools/system_tools/human_input.py), [src/agentic_layer/tools/external_tools/brave_search.py](../src/agentic_layer/tools/external_tools/brave_search.py) e [src/agentic_layer/tools/external_tools/playwright_browser.py](../src/agentic_layer/tools/external_tools/playwright_browser.py).
- Seguro reutilizar como esta?: Sim, quando uma tool do slice agentic precisar proteger o fluxo funcional contra falha de criacao ou uso do logger canonico.
- Riscos ou limitacoes: o fallback e silencioso por design; isso evita derrubar a tool, mas pode ocultar perda de observabilidade se for usado fora dos boundaries previstos ou sem testes estruturais nos call sites.
- Sugestao de melhoria: manter testes cobrindo os call sites para impedir retorno de `print`, `TOOL_EXECUTED` textual como semantica principal ou regressao para logger opcional inseguro.
- Prioridade: Alta

## Canais, scheduler e telemetria em Python

### ChannelQueueFactory e providers de fila

- Descricao: abstracao de filas multicanal com implementacoes inline, Redis e RabbitMQ, mais fabrica para escolher a estrategia de enfileiramento por definicao de canal.
- Tags: fila, redis, rabbitmq
- Tipo: factory
- Arquivo: [src/channel_layer/queue.py](../src/channel_layer/queue.py)
- Linguagem: Python
- Responsabilidade principal: padronizar enqueue, iteracao e drenagem de mensagens entre modos de fila diferentes.
- Dependencias principais: modelos de canal, Redis, RabbitMQ, excecoes da camada de canais
- Acoplamento forte com dominio?: Medio. E especifico de canais, mas reutilizavel em toda a camada multicanal.
- Uso atual observado: Sim. Usado por [src/channel_layer/worker.py](../src/channel_layer/worker.py) e [src/channel_layer/processor.py](../src/channel_layer/processor.py), com cobertura forte em [tests/unit/test_02-02-40_channel_queue.py](../tests/unit/test_02-02-40_channel_queue.py).
- Seguro reutilizar como esta?: Sim, dentro do dominio de canais.
- Riscos ou limitacoes: nao e uma fila universal para qualquer contexto; conhece `ChannelDefinition` e modos da camada.
- Sugestao de melhoria: manter o contrato desacoplado de detalhes de modelo quando surgir necessidade fora de canais.
- Prioridade: Alta

### ChannelRegistry

- Descricao: registro central de definicoes de canal autorizado a resolver qual configuracao de canal vale no runtime, evitando espalhar lookup de diretorio de clientes e regras de selecao.
- Tags: canais, registry, runtime
- Tipo: service
- Arquivo: [src/channel_layer/registry.py](../src/channel_layer/registry.py)
- Linguagem: Python
- Responsabilidade principal: localizar e validar definicoes de canal para processor e worker.
- Dependencias principais: diretorio de clientes, [src/core/base_correlation_component.py](../src/core/base_correlation_component.py)
- Acoplamento forte com dominio?: Sim. Especifico da camada de canais.
- Uso atual observado: Sim. Usado em [src/channel_layer/worker.py](../src/channel_layer/worker.py) e [src/channel_layer/processor.py](../src/channel_layer/processor.py).
- Seguro reutilizar como esta?: Sim, dentro de canais.
- Riscos ou limitacoes: pouco util fora desse dominio; nao deve virar registry generico para qualquer entidade.
- Sugestao de melhoria: explicitar melhor o contrato de resolucao e erros esperados.
- Prioridade: Alta

### SchedulerHandlerRegistry

- Descricao: registry OO de handlers do scheduler universal, evitando `if/else` por tipo de job e permitindo ligar novos handlers ao trilho oficial.
- Tags: scheduler, registry, jobs
- Tipo: registry
- Arquivo: [src/scheduler_layer/handler_registry.py](../src/scheduler_layer/handler_registry.py)
- Linguagem: Python
- Responsabilidade principal: mapear tipos de job para handlers compativeis com o scheduler.
- Dependencias principais: contratos do scheduler e validacoes da camada
- Acoplamento forte com dominio?: Sim. Especifico do scheduler.
- Uso atual observado: Sim. Instanciado por [src/api/services/async_job_worker_payload_executor.py](../src/api/services/async_job_worker_payload_executor.py) e exportado por [src/scheduler_layer/__init__.py](../src/scheduler_layer/__init__.py).
- Seguro reutilizar como esta?: Sim, para novos handlers do scheduler.
- Riscos ou limitacoes: so faz sentido dentro do contrato do scheduler universal.
- Sugestao de melhoria: documentar exemplos de registro de novo handler e regras de validacao.
- Prioridade: Alta

### SchedulerDispatchService e SchedulerClaimService

- Descricao: servicos de aplicacao do scheduler para claim e despacho de trabalho, mantendo a camada HTTP e jobs finos e desacoplados do repositorio e do publisher.
- Tags: scheduler, application-service, dispatch
- Tipo: service
- Arquivo: [src/scheduler_layer/services.py](../src/scheduler_layer/services.py)
- Linguagem: Python
- Responsabilidade principal: coordenar fluxo de despacho e reserva de execucao.
- Dependencias principais: portas `SchedulerRepositoryPort` e `SchedulerWorkPublisherPort`
- Acoplamento forte com dominio?: Sim. Especifico do scheduler.
- Uso atual observado: Sim. Consumido por [src/api/services/scheduler_dispatch_maintenance_job.py](../src/api/services/scheduler_dispatch_maintenance_job.py) e testado em [tests/unit/test_02-05-52_scheduler_dispatch_service.py](../tests/unit/test_02-05-52_scheduler_dispatch_service.py).
- Seguro reutilizar como esta?: Sim, sempre que o fluxo fizer parte do scheduler oficial.
- Riscos ou limitacoes: nao substitui uma fila generica; depende das portas do scheduler.
- Sugestao de melhoria: manter exemplos de wiring com portas fake para testes.
- Prioridade: Alta

### SchedulerAccessContextResolver, SchedulerAdminRepositoryFactory, SchedulerScheduleAdminPrimitive e SchedulerExecutionAdminPrimitive

- Descricao: primitivas canônicas do scheduler admin para resolver contexto de acesso por access key, construir repositório oficial no boundary HTTP e centralizar operações de schedule e execução sem duplicar regra nos serviços de endpoint.
- Tags: scheduler-admin, primitive, boundary-http
- Tipo: service
- Arquivo: [src/api/services/admin/scheduler_admin_primitives.py](../src/api/services/admin/scheduler_admin_primitives.py)
- Linguagem: Python
- Responsabilidade principal: concentrar contratos administrativos de listagem/criação/atualização/cancelamento de schedules e seleção/remoção de execuções, preservando tenant, correlation_id e validações canônicas.
- Dependencias principais: [src/api/request_correlation.py](../src/api/request_correlation.py), [src/security/client_directory.py](../src/security/client_directory.py), [src/scheduler_layer/postgres_repository.py](../src/scheduler_layer/postgres_repository.py), [src/scheduler_layer/models.py](../src/scheduler_layer/models.py)
- Acoplamento forte com dominio?: Medio. E específico do domínio scheduler admin, mas com responsabilidade transversal para múltiplos endpoints administrativos.
- Uso atual observado: Sim. Consumido por [src/api/services/admin/scheduler_service.py](../src/api/services/admin/scheduler_service.py), [src/api/services/admin/scheduler_execution_service.py](../src/api/services/admin/scheduler_execution_service.py) e coberto por [tests/unit/test_02-01-27_admin_scheduler_service.py](../tests/unit/test_02-01-27_admin_scheduler_service.py).
- Seguro reutilizar como esta?: Sim, para novos endpoints administrativos de scheduler no mesmo contrato de tenant/access key.
- Riscos ou limitacoes: possui defaults opinativos do domínio (`job_type=ingestion_job`, fallback de e-mail e `queue_name=ingestion`), então não deve ser usado como primitiva genérica fora do recorte admin scheduler sem evolução explícita do contrato.
- Sugestao de melhoria: manter validações de payload e políticas de default centralizadas neste módulo para evitar drift entre serviços administrativos.
- Prioridade: Alta

### AdminSchedulerService

- Descricao: fachada administrativa para endpoints de schedule que delega operações para primitivas canônicas e mantém respostas HTTP padronizadas para o painel admin.
- Tags: scheduler-admin, facade, api-service
- Tipo: service
- Arquivo: [src/api/services/admin/scheduler_service.py](../src/api/services/admin/scheduler_service.py)
- Linguagem: Python
- Responsabilidade principal: expor operações assíncronas de listar, criar, atualizar e cancelar jobs com tratamento uniforme de erro e payload de resposta administrativo.
- Dependencias principais: [src/api/services/admin/scheduler_admin_primitives.py](../src/api/services/admin/scheduler_admin_primitives.py), [src/api/schemas/admin_models.py](../src/api/schemas/admin_models.py), [src/core/logging_system.py](../src/core/logging_system.py)
- Acoplamento forte com dominio?: Medio. Específico de administração de scheduler via API.
- Uso atual observado: Sim. Consumido por [src/api/routers/admin/scheduler_router.py](../src/api/routers/admin/scheduler_router.py) e [src/api/routers/admin/runtime_scheduler_compat.py](../src/api/routers/admin/runtime_scheduler_compat.py), com cobertura em [tests/unit/test_02-01-27_admin_scheduler_service.py](../tests/unit/test_02-01-27_admin_scheduler_service.py).
- Seguro reutilizar como esta?: Sim, para boundaries HTTP administrativos que precisem manter o contrato já publicado.
- Riscos ou limitacoes: é uma camada de boundary; uso fora de HTTP ou em fluxo interno de domínio pode acoplar indevidamente exceções e schemas de API.
- Sugestao de melhoria: manter a lógica de domínio nas primitivas e evitar crescimento de regras de negócio na fachada.
- Prioridade: Alta

### AdminSchedulerExecutionService

- Descricao: fachada administrativa para histórico e purge de execuções do scheduler, incluindo preview/confirm e remoção de logs correlatos via provider canônico.
- Tags: scheduler-admin, executions, purge
- Tipo: service
- Arquivo: [src/api/services/admin/scheduler_execution_service.py](../src/api/services/admin/scheduler_execution_service.py)
- Linguagem: Python
- Responsabilidade principal: listar execuções por job no tenant correto e orquestrar purge seguro de execuções + logs correlacionados mantendo contrato de confirmação explícita.
- Dependencias principais: [src/api/services/admin/scheduler_admin_primitives.py](../src/api/services/admin/scheduler_admin_primitives.py), [src/api/services/logs_admin_service.py](../src/api/services/logs_admin_service.py), [src/core/log_origin_metadata.py](../src/core/log_origin_metadata.py)
- Acoplamento forte com dominio?: Medio. Específico de administração de execuções e logs do scheduler.
- Uso atual observado: Sim. Consumido por [src/api/routers/admin/scheduler_router.py](../src/api/routers/admin/scheduler_router.py) e testado em [tests/unit/test_02-01-27_admin_scheduler_service.py](../tests/unit/test_02-01-27_admin_scheduler_service.py).
- Seguro reutilizar como esta?: Sim, para operações administrativas de execução/purge sob o mesmo contrato.
- Riscos ou limitacoes: não é serviço de limpeza genérico; depende de `access_key`, limites fixos de segurança e semântica de preview/confirm do domínio admin scheduler.
- Sugestao de melhoria: manter os limites de purge e regras de confirmação como política explícita centralizada para evitar variações entre endpoints.
- Prioridade: Alta

### CanonicalJobRunsManager

- Descricao: gerenciador canonico de persistencia de job runs operacionais, com conexao de banco, retry, callback de progresso e maquina de estados para workers.
- Tags: job-runs, telemetria, persistencia
- Tipo: service
- Arquivo: [src/telemetry/job_runs/canonical_job_runs_manager.py](../src/telemetry/job_runs/canonical_job_runs_manager.py)
- Linguagem: Python
- Responsabilidade principal: registrar historico operacional de jobs sem cada worker implementar sua propria persistencia.
- Dependencias principais: [src/core/database_connection_manager.py](../src/core/database_connection_manager.py), [src/core/external_retry.py](../src/core/external_retry.py), status machine do dominio
- Acoplamento forte com dominio?: Medio. E transversal aos jobs, mas focado em execucao operacional.
- Uso atual observado: Sim. O proprio modulo combina pool compartilhado com retry canonico e e parte do caminho de persistencia worker-only dos jobs.
- Seguro reutilizar como esta?: Sim, para novos tipos de job que entram no trilho oficial.
- Riscos ou limitacoes: nao e uma telemetria generica de qualquer evento; e especifico para job runs canonicos.
- Sugestao de melhoria: documentar claramente quais transicoes de estado sao responsabilidade do manager e quais sao do chamador.
- Prioridade: Alta

## Shared Python com reuso importante, mas mais orientado a dominio

### PromptBundle e prompts padrao compartilhados

- Descricao: catalogo de prompt padrao com composicao segura de `system_template`, fallback explicito e montagem de bundle para QA e supervision.
- Tags: prompts, qa, defaults
- Tipo: helper
- Arquivo: [src/shared/prompts/default_prompts.py](../src/shared/prompts/default_prompts.py)
- Linguagem: Python
- Responsabilidade principal: centralizar prompt padrao e composicao de bundles de prompt.
- Dependencias principais: tipagem Python
- Acoplamento forte com dominio?: Medio. O uso e forte em QA e agentes.
- Uso atual observado: Sim. Consumido por [src/qa_layer/prompt_builder.py](../src/qa_layer/prompt_builder.py), [src/qa_layer/qa_setup_manager.py](../src/qa_layer/qa_setup_manager.py), [src/security/credential_manager.py](../src/security/credential_manager.py) e coberto por [tests/unit/test_02-05-07_prompt_bundle.py](../tests/unit/test_02-05-07_prompt_bundle.py).
- Seguro reutilizar como esta?: Sim, para novos fluxos que precisem do prompt padrao do produto.
- Riscos ou limitacoes: nao deve virar deposito de prompts de negocio locais.
- Sugestao de melhoria: separar com mais nitidez prompts globais de prompts de dominio especifico.
- Prioridade: Media

### MetadataSchemaRegistry e schemas de metadados de dominio

- Descricao: registro e familia de schemas de metadados estruturados para dominios de RAG, padronizando campos, descricoes e filtragem sem cada processador declarar schema do zero.
- Tags: metadados, schema, dominio
- Tipo: registry
- Arquivo: [src/shared/domain_metadata/metadata_schemas.py](../src/shared/domain_metadata/metadata_schemas.py)
- Linguagem: Python
- Responsabilidade principal: fornecer schemas de metadados e um registry para resolve-los por dominio.
- Dependencias principais: `AttributeInfo` do LangChain, [src/core/base_correlation_component.py](../src/core/base_correlation_component.py)
- Acoplamento forte com dominio?: Sim. O reuso e forte dentro de processadores e RAG de dominio.
- Uso atual observado: Sim. Usado por [src/ingestion_layer/processors/domain_plugins/domain_processing_resolver.py](../src/ingestion_layer/processors/domain_plugins/domain_processing_resolver.py), [src/qa_layer/domain_specific_rag/domain_specific_rag.py](../src/qa_layer/domain_specific_rag/domain_specific_rag.py) e coberto por varios testes como [tests/unit/test_02-03-31_domain_specific_rag.py](../tests/unit/test_02-03-31_domain_specific_rag.py).
- Seguro reutilizar como esta?: Sim, quando o problema for schema de metadata de RAG por dominio.
- Riscos ou limitacoes: nao e um registry generico de schema para qualquer caso; conhece semanticamente dominios ja cadastrados.
- Sugestao de melhoria: separar melhor o contrato base da colecao de schemas concretos por dominio.
- Prioridade: Media

## Frontend compartilhado administrativo

### PrometeuGlobalHeader

- Descricao: runtime compartilhado do header global, sessao opcional e menu do usuario para as paginas HTML do produto.
- Tags: frontend, header, sessao
- Tipo: runtime
- Arquivo: [app/ui/static/js/shared/global-area-header.js](../app/ui/static/js/shared/global-area-header.js)
- Linguagem: JavaScript
- Responsabilidade principal: montar o header unico da UI, hidratar estado de acesso, promover navegacao secundaria legada quando ainda existir e aplicar o modo guest readonly no caminho oficial.
- Dependencias principais: DOM do browser, endpoint `/api/auth/federated/session`, [app/ui/static/css/global-area-header.css](../app/ui/static/css/global-area-header.css)
- Acoplamento forte com dominio?: Medio. E transversal nas paginas HTML do produto, mas o contrato e focado na superficie web compartilhada.
- Uso atual observado: Sim. Carregado nas paginas HTML de `app/ui/static/*.html`, com superficie publica `window.PrometeuGlobalHeader` e prova runtime em [tests/playwright/test_08-01-09_auth_header_runtime_etl_page.py](../tests/playwright/test_08-01-09_auth_header_runtime_etl_page.py).
- Seguro reutilizar como esta?: Sim, para header, sessao opcional e menu do usuario no frontend HTML do projeto.
- Riscos ou limitacoes: paginas com markup legado ainda dependem da promocao do header antigo; forcar menu/sessao por outro helper paralelo reabre drift estrutural.
- Sugestao de melhoria: continuar reduzindo dependencia de markup legado nas paginas que ainda precisam de promocao.
- Prioridade: Alta

### PrometeuAdminCorrelationRuntime

- Descricao: runtime compartilhado do shell administrativo para capturar, publicar e consultar o `correlation_id` oficial e as acoes correlacionadas do frontend admin.
- Tags: frontend, shell, correlacao
- Tipo: runtime
- Arquivo: [app/ui/static/js/shared/admin-workspace-shell.js](../app/ui/static/js/shared/admin-workspace-shell.js)
- Linguagem: JavaScript
- Responsabilidade principal: manter a fonte unica de verdade do `correlation_id` nas telas admin, incluindo captura, contexto vivo, subscription e downloads correlacionados.
- Dependencias principais: DOM do browser, storage local, superfícies admin e clientes HTTP compartilhados que chamam `capture`.
- Acoplamento forte com dominio?: Medio. E especifico do shell administrativo, mas bem transversal dentro desse espaco.
- Uso atual observado: Sim. Publicado por [app/ui/static/js/shared/admin-workspace-shell.js](../app/ui/static/js/shared/admin-workspace-shell.js), consumido por [app/ui/static/js/shared/admin-api-client.js](../app/ui/static/js/shared/admin-api-client.js), [app/ui/static/js/shared/layout-mestre-api.js](../app/ui/static/js/shared/layout-mestre-api.js), [app/ui/static/js/shared/ui-webchat-runtime-utils.js](../app/ui/static/js/shared/ui-webchat-runtime-utils.js) e pelo componente leve [app/ui/static/js/shared/correlation-surface.js](../app/ui/static/js/shared/correlation-surface.js).
- Seguro reutilizar como esta?: Sim, quando a tela admin precisa ler ou refletir o `correlation_id` oficial sem recriar runtime paralelo.
- Riscos ou limitacoes: nao substitui a superficie leve para paginas fora do shell completo e depende de as paginas admin carregarem o shell oficial.
- Sugestao de melhoria: documentar melhor o contrato publico minimo para consumidores indiretos fora do shell completo.
- Prioridade: Alta

### PrometeuAdminApiClient

- Descricao: cliente HTTP compartilhado das paginas administrativas, com `adminFetch`, parse de erro, extracao de `correlation_id`, leitura opcional de sessao federada, marcador explicito `X-Prometeu-UI-Session-Required` e propagacao da sessao web oficial nas acoes de tela.
- Tags: frontend, api, correlacao
- Tipo: client
- Arquivo: [app/ui/static/js/shared/admin-api-client.js](../app/ui/static/js/shared/admin-api-client.js)
- Linguagem: JavaScript
- Responsabilidade principal: padronizar chamadas administrativas no frontend sem cada pagina reinventar `fetch`, extracao de erro, correlacao, marcador de acao oficial da UI e envio do cookie de sessao no caminho canonico.
- Dependencias principais: [app/ui/static/js/shared/admin-utils.js](../app/ui/static/js/shared/admin-utils.js)
- Acoplamento forte com dominio?: Medio. Especifico de paginas admin, mas bem transversal dentro desse espaco.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/admin-ingestao.js](../app/ui/static/js/admin-ingestao.js), [app/ui/static/js/admin-etl.js](../app/ui/static/js/admin-etl.js), [app/ui/static/js/admin-scheduler.js](../app/ui/static/js/admin-scheduler.js), [app/ui/static/js/admin-schema-metadata.js](../app/ui/static/js/admin-schema-metadata.js) e [app/ui/static/js/admin-users.js](../app/ui/static/js/admin-users.js), com contratos em [tests/frontend/admin_api_client_correlation.test.js](../tests/frontend/admin_api_client_correlation.test.js), [tests/frontend/admin_context_request_contract.test.js](../tests/frontend/admin_context_request_contract.test.js) e prova browser em [tests/playwright/test_08-01-09_auth_header_runtime_etl_page.py](../tests/playwright/test_08-01-09_auth_header_runtime_etl_page.py).
- Seguro reutilizar como esta?: Sim, em paginas administrativas do projeto.
- Riscos ou limitacoes: depende do runtime global `window.PrometeuAdminApiClient`; fora desse padrao a pagina precisa adaptar bootstrap. Em telas que nao representam acoes oficiais da UI, forcar esse cliente por conveniencia pode apertar o gate de sessao sem necessidade.
- Sugestao de melhoria: reduzir dependencia de globals em paginas novas e expor uma superficie ES module mais direta.
- Prioridade: Alta

### PrometeuCorrelationSurface

- Descricao: superficie leve compartilhada para exibir o `correlation_id` oficial ou a ausencia explicita dele em paginas fora do shell administrativo completo, sem criar runtime paralelo nem gerar valor local.
- Tags: frontend, correlacao, observabilidade
- Tipo: component
- Arquivo: [app/ui/static/js/shared/correlation-surface.js](../app/ui/static/js/shared/correlation-surface.js)
- Linguagem: JavaScript
- Responsabilidade principal: montar um card minimo de observabilidade que reaproveita [app/ui/static/js/shared/admin-api-client.js](../app/ui/static/js/shared/admin-api-client.js) para extrair `correlation_id` e, quando existir, prioriza o contexto vivo do runtime oficial do shell administrativo.
- Dependencias principais: [app/ui/static/js/shared/admin-api-client.js](../app/ui/static/js/shared/admin-api-client.js), `window.PrometeuAdminCorrelationRuntime`, DOM do browser
- Acoplamento forte com dominio?: Nao. E um componente transversal de observabilidade para familias de tela que fazem request real fora do shell completo.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/auth-gateway.js](../app/ui/static/js/auth-gateway.js), [app/ui/static/js/account-center.js](../app/ui/static/js/account-center.js) e [app/ui/static/js/client-portal.js](../app/ui/static/js/client-portal.js), com host declarativo em [app/ui/static/ui-account-center.html](../app/ui/static/ui-account-center.html), [app/ui/static/ui-area-account-center.html](../app/ui/static/ui-area-account-center.html) e [app/ui/static/ui-client-portal.html](../app/ui/static/ui-client-portal.html), alem de criacao dinamica no [app/ui/static/auth-gateway.html](../app/ui/static/auth-gateway.html). O contrato estrutural esta protegido por [tests/frontend/outside_shell_correlation_contract.test.js](../tests/frontend/outside_shell_correlation_contract.test.js).
- Seguro reutilizar como esta?: Sim, quando a tela precisar mostrar o valor oficial devolvido pelo backend sem puxar o shell admin inteiro.
- Riscos ou limitacoes: depende de a pagina carregar primeiro o cliente HTTP compartilhado; nao substitui o runtime completo de correlation das paginas administrativas nem corrige endpoint que nao devolve `correlation_id`.
- Sugestao de melhoria: evoluir com API modular complementar quando mais paginas fora do shell precisarem do mesmo padrao sem depender de global browser.
- Prioridade: Alta

### PrometeuLayoutMestreApi

- Descricao: camada compartilhada para falar com o Layout Mestre e fluxos QA/agent/workflow, incluindo criptografia de payload, extracao de `correlation_id` e persistencia offline operacional.
- Tags: frontend, layout-mestre, api
- Tipo: client
- Arquivo: [app/ui/static/js/shared/layout-mestre-api.js](../app/ui/static/js/shared/layout-mestre-api.js)
- Linguagem: JavaScript
- Responsabilidade principal: encapsular o protocolo de chamada da UI para o backend do layout mestre.
- Dependencias principais: `PayloadCrypto`, runtime do layout
- Acoplamento forte com dominio?: Sim. Focado no contrato do layout mestre.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/ui-analise-interacoes.js](../app/ui/static/js/ui-analise-interacoes.js), [app/ui/static/js/admin-sql-natural.js](../app/ui/static/js/admin-sql-natural.js) e coberto por [tests/frontend/admin_workspace_shell_correlation_contract.test.js](../tests/frontend/admin_workspace_shell_correlation_contract.test.js).
- Seguro reutilizar como esta?: Sim, quando a pagina seguir o contrato de Layout Mestre do projeto.
- Riscos ou limitacoes: nao e um cliente HTTP generico; conhece payload criptografado e convencoes desse layout.
- Fonte unica de montagem de envelope/chamada de API consumida pelo componente embutivel e pela host de exemplo (envelope identico ao v3). Expoe a opcao `registrarPayloadOffline` (default `true`): quando `false`, pula o POST em `/crypto/offline-store`, replicando o v3 — usado pelo componente embutivel. Demais consumidores seguem no default `true`.
- Sugestao de melhoria: documentar melhor o contrato de inicializacao minima da pagina consumidora.
- Prioridade: Alta

### DnitProjectChatRuntime

- Descricao: runtime compartilhado do chat lateral do detalhe DNIT, construído sobre os helpers canônicos do WebChat e do Layout Mestre para reaproveitar carga de YAML fixo, criptografia de payload, polling assíncrono, `correlation_id` e HIL sem manter um cliente paralelo na página.

### PrometeuEmbeddableChatRuntime

- Descricao: componente de chat embutível com DOM próprio, configurável por propriedades e eventos, desenhado para ser montado por uma tela host sem criar cliente HTTP paralelo nem depender do runtime DNIT.
- Tags: frontend, chat, embutido, runtime
- Tipo: runtime
- Arquivo: [app/ui/static/js/shared/embeddable-chat-runtime.js](../app/ui/static/js/shared/embeddable-chat-runtime.js)
- Linguagem: JavaScript
- Responsabilidade principal: encapsular renderização do chat, envio de perguntas, polling assíncrono, exportação de estado e publicação de eventos para a página hospedeira.
- Dependencias principais: [app/ui/static/js/shared/layout-mestre-api.js](../app/ui/static/js/shared/layout-mestre-api.js), [app/ui/static/js/shared/ui-webchat-runtime-utils.js](../app/ui/static/js/shared/ui-webchat-runtime-utils.js), [app/ui/static/js/shared/ui-webchat-async-runtime.js](../app/ui/static/js/shared/ui-webchat-async-runtime.js)
- Acoplamento forte com dominio?: Baixo. Reaproveita o contrato canônico da UI, mas não depende mais do runtime lateral DNIT.
- Uso atual observado: Sim. Coberto por [tests/frontend/embeddable_chat_runtime_contract.test.js](../tests/frontend/embeddable_chat_runtime_contract.test.js) e pelo E2E Playwright [tests/playwright/test_08-01-10_embeddable_chat_isolated.py](../tests/playwright/test_08-01-10_embeddable_chat_isolated.py). Bancada de validação isolada em [app/ui/static/ui-embeddable-chat-test.html](../app/ui/static/ui-embeddable-chat-test.html). Host de exemplo oficial em [app/ui/static/ui-admin-plataforma-webchat.html](../app/ui/static/ui-admin-plataforma-webchat.html). Comportamento HTTP do chat alinhado ao v3 (não chama `/crypto/offline-store`; instancia o cliente com `registrarPayloadOffline: false`). Provado em runtime real (FastAPI local) em desktop e mobile.
- Seguro reutilizar como esta?: Sim, para telas que precisem montar um chat reutilizável e reagir ao estado fora dele.
- Riscos ou limitacoes: depende da presença explícita de `prometeuLayoutMestreApi`, `WebchatRuntimeUtils` e `WebchatAsyncRuntime`; sem isso ele falha fechado.
- Sugestao de melhoria: manter documentação e exemplos sincronizados sempre que a API pública do componente evoluir.
- Prioridade: Media

- Guia de uso detalhado: docs/GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md
- Tags: frontend, dnit, webchat
- Tipo: runtime
- Arquivo: [app/ui/static/js/shared/dnit-project-chat-runtime.js](../app/ui/static/js/shared/dnit-project-chat-runtime.js)
- Linguagem: JavaScript
- Responsabilidade principal: centralizar a sessão de conversa do painel lateral DNIT e expor uma API pequena para enviar perguntas, responder HIL, limpar conversa e sincronizar estado reativo da tela.
- Dependencias principais: [app/ui/static/js/shared/layout-mestre-api.js](../app/ui/static/js/shared/layout-mestre-api.js), [app/ui/static/js/shared/ui-webchat-runtime-utils.js](../app/ui/static/js/shared/ui-webchat-runtime-utils.js), [app/ui/static/js/shared/ui-webchat-async-runtime.js](../app/ui/static/js/shared/ui-webchat-async-runtime.js), [app/ui/static/js/shared/ui-webchat-hil-contract.js](../app/ui/static/js/shared/ui-webchat-hil-contract.js)
- Acoplamento forte com dominio?: Medio. Ele conhece o contrato do detalhe DNIT e o envelope do webchat oficial, mas foi isolado para evitar repetição no arquivo da página.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/dnit-project-detail.js](../app/ui/static/js/dnit-project-detail.js) e protegido por [tests/frontend/dnit_project_chat_runtime_contract.test.js](../tests/frontend/dnit_project_chat_runtime_contract.test.js).
- Seguro reutilizar como esta?: Sim, para páginas ou sidecars que usem o mesmo contrato de assistente lateral baseado em YAML fixo do backend DNIT.
- Riscos ou limitacoes: depende da presença dos helpers globais do runtime oficial no browser e de o backend devolver `assistant_config` com `yaml_file` e `yaml_public_url` válidos.
- Sugestao de melhoria: extrair uma camada ainda mais genérica se outras telas passarem a usar o mesmo padrão de chat embutido com YAML fixo.
- Prioridade: Alta

### PrometeuLayoutMestreContextRuntime

- Descricao: runtime global compartilhado que publica o snapshot operacional do Layout Mestre e centraliza leitura, escrita e refresh de `apiKey`, `userEmail`, YAML e payload para consumidores indiretos do shell HTML padrao.
- Tags: frontend, layout-mestre, runtime
- Tipo: runtime
- Arquivo: [app/ui/static/js/shared/layout-mestre.js](../app/ui/static/js/shared/layout-mestre.js)
- Linguagem: JavaScript
- Responsabilidade principal: expor a API publica de snapshot e mutacao do contexto do Layout Mestre como fonte unica de verdade para bridges, helpers e paginas HTML.
- Dependencias principais: `window`, `localStorage`, `CustomEvent`, configuracao `LAYOUT_MESTRE_CONFIG`
- Acoplamento forte com dominio?: Medio. E especifico do contrato do Layout Mestre, mas transversal para varias paginas administrativas e shells HTML do projeto.
- Uso atual observado: Sim. Publicado por [app/ui/static/js/shared/layout-mestre.js](../app/ui/static/js/shared/layout-mestre.js), consumido por [app/ui/static/js/shared/admin-layout-bridge.js](../app/ui/static/js/shared/admin-layout-bridge.js) e [app/ui/static/js/shared/ag-ui-retail-demo-page.js](../app/ui/static/js/shared/ag-ui-retail-demo-page.js), com contrato protegido por [tests/frontend/layout_mestre_contract.test.js](../tests/frontend/layout_mestre_contract.test.js) e [tests/frontend/admin_layout_bridge_contract.test.js](../tests/frontend/admin_layout_bridge_contract.test.js).
- Seguro reutilizar como esta?: Sim, para paginas e helpers que participem do contrato do Layout Mestre.
- Riscos ou limitacoes: e uma API global; mudancas no nome, no shape do snapshot ou no evento compartilhado afetam consumidores indiretos ao mesmo tempo.
- Sugestao de melhoria: documentar formalmente a superficie publica do runtime e seus campos estaveis para reduzir drift entre paginas.
- Prioridade: Alta

### PrometeuAdminLayoutBridge

- Descricao: bridge/facade compartilhada entre o Layout Mestre e as consoles administrativas, convertendo o runtime global do layout em uma API conveniente de snapshot, `apiKey`, `userEmail`, espelho de input e bind reativo de contexto.
- Tags: frontend, bridge, contexto
- Tipo: bridge
- Arquivo: [app/ui/static/js/shared/admin-layout-bridge.js](../app/ui/static/js/shared/admin-layout-bridge.js)
- Linguagem: JavaScript
- Responsabilidade principal: desacoplar paginas admin de leitura direta de DOM e storage, delegando o contexto operacional ao runtime canonico do Layout Mestre.
- Dependencias principais: [app/ui/static/js/shared/layout-mestre.js](../app/ui/static/js/shared/layout-mestre.js), `CustomEvent`, DOM do browser
- Acoplamento forte com dominio?: Medio. E especifico do shell admin baseado no Layout Mestre, mas compartilhado por varias superficies administrativas.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/admin-offline-keys.js](../app/ui/static/js/admin-offline-keys.js), [app/ui/static/js/admin-scheduler.js](../app/ui/static/js/admin-scheduler.js), [app/ui/static/js/objective-yaml-studio.js](../app/ui/static/js/objective-yaml-studio.js), [app/ui/static/js/admin-vector-preview.js](../app/ui/static/js/admin-vector-preview.js), [app/ui/static/js/admin-tenant-secrets.js](../app/ui/static/js/admin-tenant-secrets.js), [app/ui/static/js/admin-users.js](../app/ui/static/js/admin-users.js), [app/ui/static/js/admin-schema-metadata.js](../app/ui/static/js/admin-schema-metadata.js), [app/ui/static/js/admin-channel-end-users.js](../app/ui/static/js/admin-channel-end-users.js), [app/ui/static/js/admin-logs-central/app.js](../app/ui/static/js/admin-logs-central/app.js) e [app/ui/static/js/shared/admin-integracoes-api.js](../app/ui/static/js/shared/admin-integracoes-api.js), com guardrails em [tests/frontend/admin_layout_bridge_contract.test.js](../tests/frontend/admin_layout_bridge_contract.test.js), [tests/frontend/admin_context_request_contract.test.js](../tests/frontend/admin_context_request_contract.test.js) e [tests/frontend/nl2yaml_studio_contract.test.js](../tests/frontend/nl2yaml_studio_contract.test.js).
- Seguro reutilizar como esta?: Sim, em paginas que precisem ler ou sincronizar contexto operacional do Layout Mestre fora do mesmo `x-data` que controla o layout.
- Riscos ou limitacoes: depende de `window.PrometeuLayoutMestreContextRuntime` e do evento `prometeu:credenciais-alteradas`; nao substitui um helper modular mais leve quando a pagina precisa apenas de uma leitura local simples.
- Sugestao de melhoria: oferecer tambem uma superficie ES module para reduzir dependencia de globals em paginas novas.
- Prioridade: Alta

### resolveAdminLayoutAccessKey e buildAdminLayoutStorageSnapshot

- Descricao: kernel modular compartilhado dos runners administrativos para resolver a `access_key` efetiva e montar o snapshot basico do contexto persistido pelo Layout Mestre sem duplicar leitura de DOM e `localStorage`.
- Tags: frontend, contexto, runner
- Tipo: helper
- Arquivo: [app/ui/static/js/shared/admin-layout-context-runtime.js](../app/ui/static/js/shared/admin-layout-context-runtime.js)
- Linguagem: JavaScript
- Responsabilidade principal: fornecer funcoes canonicas de leitura de `apiKey`, YAML, payload e filenames do contexto padrao do Layout Mestre para consoles administrativas.
- Dependencias principais: DOM do browser, `window.localStorage`, `window.LAYOUT_MESTRE_CONFIG`
- Acoplamento forte com dominio?: Medio. E especifico de consoles admin que leem o bloco padrao do Layout Mestre, mas transversal nesse espaco.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/admin-ingestao.js](../app/ui/static/js/admin-ingestao.js), [app/ui/static/js/admin-etl.js](../app/ui/static/js/admin-etl.js) e [app/ui/static/js/admin-ingestao-pdf.js](../app/ui/static/js/admin-ingestao-pdf.js), com contrato protegido por [tests/frontend/admin_runner_layout_context_contract.test.js](../tests/frontend/admin_runner_layout_context_contract.test.js).
- Seguro reutilizar como esta?: Sim, quando a tela precisa apenas do snapshot basico ou da `X-API-Key` atual no padrao dos runners admin.
- Riscos ou limitacoes: nao observa mudancas reativamente por si so; depende dos IDs e storage keys canonicos do Layout Mestre; fora desse contrato exige adaptacao.
- Sugestao de melhoria: documentar melhor quando usar este helper modular e quando subir para a bridge completa do layout.
- Prioridade: Alta

### createAdminRunnerApiBaseRuntime

- Descricao: runtime compartilhado para preparar requests de runners administrativos, validar pre-condicoes e resolver payload criptografado sem duplicar esse fluxo em cada pagina.
- Tags: frontend, runner, request
- Tipo: helper
- Arquivo: [app/ui/static/js/shared/admin-runner-request-core.js](../app/ui/static/js/shared/admin-runner-request-core.js)
- Linguagem: JavaScript
- Responsabilidade principal: montar o contexto base de request para paginas admin que executam jobs longos.
- Dependencias principais: storage do browser, `location`, [app/ui/static/js/shared/admin-api-client.js](../app/ui/static/js/shared/admin-api-client.js)
- Acoplamento forte com dominio?: Medio. Foi feito para runners admin, mas resolve problema bem recorrente nesse espaco.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/admin-ingestao.js](../app/ui/static/js/admin-ingestao.js) e [app/ui/static/js/admin-etl.js](../app/ui/static/js/admin-etl.js).
- Seguro reutilizar como esta?: Sim, em novas paginas admin do mesmo padrao.
- Riscos ou limitacoes: depende de convencoes de storage e do cliente admin global.
- Sugestao de melhoria: extrair interfaces menores para validacao, criptografia e persistencia offline.
- Prioridade: Alta

### admin-runner-core

- Descricao: nucleo compartilhado para polling, cancelamento, processamento de status e consolidacao de estado dos runners administrativos, evitando copiar logica de acompanhamento de job.
- Tags: frontend, polling, job-runner
- Tipo: helper
- Arquivo: [app/ui/static/js/shared/admin-runner-core.js](../app/ui/static/js/shared/admin-runner-core.js)
- Linguagem: JavaScript
- Responsabilidade principal: padronizar o comportamento de monitoramento de execucoes administrativas.
- Dependencias principais: [app/ui/static/js/shared/admin-api-client.js](../app/ui/static/js/shared/admin-api-client.js)
- Acoplamento forte com dominio?: Medio. Muito util dentro do frontend admin.
- Uso atual observado: Sim. Declaradamente depende do cliente admin compartilhado e e parte do runtime reutilizado por runners administrativos.
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: nao substitui a modelagem de payload especifica de cada job; espera um contrato de status compativel.
- Sugestao de melhoria: formalizar interfaces de payload de status para reduzir condicionais internas.
- Prioridade: Alta

### createIngestionExecutionClient

- Descricao: cliente especializado para disparo de ingestao administrativa, em composicao com o runtime generico de request dos runners admin.
- Tags: ingestao, frontend, client
- Tipo: client
- Arquivo: [app/ui/static/js/shared/ingestion-execution-client.js](../app/ui/static/js/shared/ingestion-execution-client.js)
- Linguagem: JavaScript
- Responsabilidade principal: encapsular o request de execucao da ingestao, inclusive detalhes de payload e validacoes ligadas a esse fluxo.
- Dependencias principais: [app/ui/static/js/shared/admin-runner-request-core.js](../app/ui/static/js/shared/admin-runner-request-core.js)
- Acoplamento forte com dominio?: Sim. Focado em ingestao administrativa.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/admin-ingestao.js](../app/ui/static/js/admin-ingestao.js) e coberto por [tests/frontend/admin_ingestao_correlation_contract.test.js](../tests/frontend/admin_ingestao_correlation_contract.test.js).
- Seguro reutilizar como esta?: Sim, para novas telas de ingestao que sigam o mesmo contrato.
- Riscos ou limitacoes: nao e um client admin universal; conhece o payload da ingestao.
- Sugestao de melhoria: manter a fronteira entre runtime generico e detalhes da ingestao sempre nitida.
- Prioridade: Alta

### ingestion-live-payload-parser

- Descricao: parser canonico do payload vivo de monitoramento da ingestao, com normalizacao de status, filhos, documentos e resumo de paralelismo para a UI.
- Tags: ingestao, parser, monitoramento
- Tipo: parser
- Arquivo: [app/ui/static/js/shared/ingestion-live-payload-parser.js](../app/ui/static/js/shared/ingestion-live-payload-parser.js)
- Linguagem: JavaScript
- Responsabilidade principal: traduzir payload operacional cru em estrutura segura e consistente para renderizacao.
- Dependencias principais: JavaScript padrao
- Acoplamento forte com dominio?: Sim. Totalmente ligado ao monitor de ingestao.
- Uso atual observado: Potencial. O modulo e claramente reutilizavel no frontend de ingestao, mas esta rodada nao abriu um call site alem da propria superficie compartilhada.
- Seguro reutilizar como esta?: Sim, dentro de telas de monitoramento da ingestao.
- Riscos ou limitacoes: o contrato de payload precisa continuar alinhado ao backend; mudancas de naming quebram a normalizacao.
- Sugestao de melhoria: manter testes de contrato especificos do payload vivo quando o modulo crescer.
- Prioridade: Media

### yaml-access-key-extractor

- Descricao: extrator compartilhado de `access_key` e `user_email` a partir de YAML ou payload textual, reduzindo duplicacao de parsing defensivo em telas administrativas.
- Tags: yaml, parsing, autenticacao
- Tipo: helper
- Arquivo: [app/ui/static/js/shared/yaml-access-key-extractor.js](../app/ui/static/js/shared/yaml-access-key-extractor.js)
- Linguagem: JavaScript
- Responsabilidade principal: limpar comentarios inline, normalizar candidatos e extrair credenciais basicas do YAML recebido pela UI.
- Dependencias principais: JavaScript padrao
- Acoplamento forte com dominio?: Medio. O problema e comum na UI administrativa.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/admin-ingestao.js](../app/ui/static/js/admin-ingestao.js), [app/ui/static/js/admin-etl.js](../app/ui/static/js/admin-etl.js), [app/ui/static/js/admin-offline-keys.js](../app/ui/static/js/admin-offline-keys.js), [app/ui/static/js/admin-cache.js](../app/ui/static/js/admin-cache.js), [app/ui/static/js/admin-assembly-ast.js](../app/ui/static/js/admin-assembly-ast.js), [app/ui/static/js/shared/layout-mestre.js](../app/ui/static/js/shared/layout-mestre.js) e [app/ui/static/js/ui-webchat-v3.js](../app/ui/static/js/ui-webchat-v3.js), com guardrails em [tests/frontend/admin_api_client_correlation.test.js](../tests/frontend/admin_api_client_correlation.test.js).
- Seguro reutilizar como esta?: Sim, em telas admin que realmente precisem ler esses campos do YAML.
- Riscos ou limitacoes: parser leve de texto; nao substitui resolucao canonica de YAML no backend.
- Sugestao de melhoria: manter claro na documentacao que este helper so serve para UX local e nunca para resolver YAML oficial de runtime; paginas que o carregam contratualmente nao devem voltar para regex local silenciosa.
- Prioridade: Media

## AG-UI compartilhado

### createAgUiStateStore

- Descricao: store de estado AG-UI para mensagens, tools, interrupcoes e patches, servindo como nucleo de estado compartilhado entre sidecar, renderers e runtime do pacote.
- Tags: ag-ui, state-store, runtime
- Tipo: store
- Arquivo: [app/ui/static/js/shared/ag-ui-state-store.js](../app/ui/static/js/shared/ag-ui-state-store.js)
- Linguagem: JavaScript
- Responsabilidade principal: manter estado do chat e dos eventos AG-UI de forma consistente.
- Dependencias principais: JavaScript padrao
- Acoplamento forte com dominio?: Medio. E transversal dentro do ecossistema AG-UI.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/shared/ag-ui-sidecar-chat.js](../app/ui/static/js/shared/ag-ui-sidecar-chat.js) e reexportado por [packages/ag-ui-runtime/index.js](../packages/ag-ui-runtime/index.js).
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: qualquer mudanca de contrato impacta UI local e pacote runtime ao mesmo tempo.
- Sugestao de melhoria: manter testes de contrato sobre o formato do estado e eventos aplicados.
- Prioridade: Alta

### UiSpecValidator

- Descricao: validador seguro da `UiSpec` materializada, evitando renderizacao de payload invalido, inseguro ou estruturalmente inconsistente no frontend AG-UI.
- Tags: ag-ui, validacao, ui-spec
- Tipo: validator
- Arquivo: [app/ui/static/js/shared/ag-ui-ui-spec-validator.js](../app/ui/static/js/shared/ag-ui-ui-spec-validator.js)
- Linguagem: JavaScript
- Responsabilidade principal: validar o contrato da UI espec before render.
- Dependencias principais: [app/ui/static/js/shared/ag-ui-safe-content.js](../app/ui/static/js/shared/ag-ui-safe-content.js)
- Acoplamento forte com dominio?: Medio. Especifico de AG-UI, mas transversal nesse ecossistema.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/shared/ag-ui-ui-spec-renderer.js](../app/ui/static/js/shared/ag-ui-ui-spec-renderer.js) e reexportado via [packages/ag-ui-runtime/index.js](../packages/ag-ui-runtime/index.js).
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: precisa evoluir junto com o contrato da `UiSpec`.
- Sugestao de melhoria: manter matriz de exemplos validos e invalidos por versao da especificacao.
- Prioridade: Alta

### validateDashboardSpec

- Descricao: validador seguro do contrato de dashboard AG-UI, com verificacao estrutural e protecao contra conteudo proibido antes da renderizacao.
- Tags: ag-ui, dashboard, validacao
- Tipo: validator
- Arquivo: [app/ui/static/js/shared/ag-ui-dashboard-validator.js](../app/ui/static/js/shared/ag-ui-dashboard-validator.js)
- Linguagem: JavaScript
- Responsabilidade principal: validar payloads de dashboard dinamico.
- Dependencias principais: [app/ui/static/js/shared/ag-ui-safe-content.js](../app/ui/static/js/shared/ag-ui-safe-content.js)
- Acoplamento forte com dominio?: Medio. Especifico de dashboards AG-UI.
- Uso atual observado: Sim. Consumido por [app/ui/static/js/ag-ui-dashboard-dinamico.js](../app/ui/static/js/ag-ui-dashboard-dinamico.js) e reexportado via [packages/ag-ui-runtime/index.js](../packages/ag-ui-runtime/index.js).
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: qualquer extensao da spec exige atualizar validacao e testes em conjunto.
- Sugestao de melhoria: manter um inventario publico dos codigos de erro mais comuns.
- Prioridade: Alta

### AgUiDashboardRenderer

- Descricao: renderer OO de `DashboardSpec`, acoplado aos validadores e renderers de evento para transformar a especificacao segura em interface navegavel.
- Tags: ag-ui, dashboard, renderer
- Tipo: classe
- Arquivo: [app/ui/static/js/shared/ag-ui-dashboard-renderer.js](../app/ui/static/js/shared/ag-ui-dashboard-renderer.js)
- Linguagem: JavaScript
- Responsabilidade principal: renderizar dashboard AG-UI a partir da especificacao validada.
- Dependencias principais: [app/ui/static/js/shared/ag-ui-dashboard-validator.js](../app/ui/static/js/shared/ag-ui-dashboard-validator.js), [app/ui/static/js/shared/ag-ui-event-renderer.js](../app/ui/static/js/shared/ag-ui-event-renderer.js)
- Acoplamento forte com dominio?: Medio. Especifico do runtime AG-UI.
- Uso atual observado: Sim. Reexportado via [packages/ag-ui-runtime/index.js](../packages/ag-ui-runtime/index.js).
- Seguro reutilizar como esta?: Sim, para dashboards AG-UI.
- Riscos ou limitacoes: depende do shape de `DashboardSpec` e do renderer de eventos.
- Sugestao de melhoria: explicitar melhor pontos de extensao de componentes visuais customizados.
- Prioridade: Alta

### AG-UI tool timeline

- Descricao: utilitarios para acumular timeline de tools AG-UI e aplicar eventos de ferramenta em um estado canonicamente reconhecido pela UI.
- Tags: ag-ui, tools, timeline
- Tipo: helper
- Arquivo: [app/ui/static/js/shared/ag-ui-tool-timeline.js](../app/ui/static/js/shared/ag-ui-tool-timeline.js)
- Linguagem: JavaScript
- Responsabilidade principal: consolidar eventos de tool call em uma timeline coerente para exibicao ou analise.
- Dependencias principais: JavaScript padrao
- Acoplamento forte com dominio?: Medio. Bem especifico ao runtime AG-UI.
- Uso atual observado: Potencial. O modulo e tipicamente reutilizavel no ecossistema AG-UI, mas esta rodada nao abriu o consumidor direto alem do shared/runtime.
- Seguro reutilizar como esta?: Sim, em telas e runtimes AG-UI.
- Riscos ou limitacoes: depende da estabilidade do formato de evento de tool.
- Sugestao de melhoria: adicionar testes de regressao para eventos fora de ordem e eventos incompletos.
- Prioridade: Media

### AG-UI safe content scanner

- Descricao: scanner recursivo de conteudo inseguro em payloads AG-UI, barrando chaves proibidas e strings potencialmente perigosas antes da renderizacao.
- Tags: ag-ui, seguranca, sanitizacao
- Tipo: helper
- Arquivo: [app/ui/static/js/shared/ag-ui-safe-content.js](../app/ui/static/js/shared/ag-ui-safe-content.js)
- Linguagem: JavaScript
- Responsabilidade principal: detectar conteudo proibido em specs e catalogos da UI agentica.
- Dependencias principais: JavaScript padrao
- Acoplamento forte com dominio?: Baixo dentro do dominio AG-UI; e uma utilidade de seguranca.
- Uso atual observado: Sim. Usado por [app/ui/static/js/shared/ag-ui-ui-spec-validator.js](../app/ui/static/js/shared/ag-ui-ui-spec-validator.js), [app/ui/static/js/shared/ag-ui-dashboard-validator.js](../app/ui/static/js/shared/ag-ui-dashboard-validator.js) e [app/ui/static/js/shared/ag-ui-component-catalog.js](../app/ui/static/js/shared/ag-ui-component-catalog.js).
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: listas de proibicao precisam acompanhar a superficie real de risco.
- Sugestao de melhoria: documentar criterios de bloqueio e permitir fixture suite focada em payload malicioso.
- Prioridade: Alta

### createAgUiSidecarChat

- Descricao: sidecar de chat AG-UI reutilizavel para paginas estaticas, integrando store, cliente e renderizacao sem obrigar cada pagina a remontar esse trilho.
- Tags: ag-ui, chat, sidecar
- Tipo: helper
- Arquivo: [app/ui/static/js/shared/ag-ui-sidecar-chat.js](../app/ui/static/js/shared/ag-ui-sidecar-chat.js)
- Linguagem: JavaScript
- Responsabilidade principal: montar a experiencia de chat sidecar com estado e botoes auxiliares.
- Dependencias principais: [app/ui/static/js/shared/ag-ui-client.js](../app/ui/static/js/shared/ag-ui-client.js), [app/ui/static/js/shared/ag-ui-state-store.js](../app/ui/static/js/shared/ag-ui-state-store.js), renderers de evento
- Acoplamento forte com dominio?: Medio. Especifico de UIs AG-UI em pagina estatica.
- Uso atual observado: Sim. Consome a store compartilhada e e reexportado em [packages/ag-ui-runtime/index.js](../packages/ag-ui-runtime/index.js).
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: assume contrato do cliente e do store AG-UI do projeto.
- Sugestao de melhoria: isolar melhor opcionalidade de UI para paginas minimalistas.
- Prioridade: Alta

### createPrometeuAgUiOfficialClient

- Descricao: cliente oficial do runtime AG-UI sobre `@ag-ui/client`, encapsulando o protocolo suportado pela plataforma sem depender de wrappers experimentais locais.
- Tags: ag-ui, sdk-oficial, client
- Tipo: client
- Arquivo: [packages/ag-ui-runtime/official-http-client.js](../packages/ag-ui-runtime/official-http-client.js)
- Linguagem: JavaScript
- Responsabilidade principal: construir o cliente oficial da plataforma para AG-UI.
- Dependencias principais: `@ag-ui/client`, `rxjs`, [packages/ag-ui-runtime/prometeu-helpers.js](../packages/ag-ui-runtime/prometeu-helpers.js)
- Acoplamento forte com dominio?: Medio. E a fronteira oficial da plataforma com o SDK AG-UI.
- Uso atual observado: Sim. Reexportado por [packages/ag-ui-runtime/index.js](../packages/ag-ui-runtime/index.js), [packages/ag-ui-runtime/browser-entry.js](../packages/ag-ui-runtime/browser-entry.js) e consumido por [app/ui/static/js/shared/ag-ui-client.js](../app/ui/static/js/shared/ag-ui-client.js).
- Seguro reutilizar como esta?: Sim. Deve ser preferido para consumidores AG-UI novos.
- Riscos ou limitacoes: segue o SDK oficial; qualquer drift do pacote externo precisa ser tratado aqui, nao em wrappers paralelos.
- Sugestao de melhoria: manter exemplos minimos de bootstrap por ambiente browser e pacote.
- Prioridade: Alta

### Helpers do runtime AG-UI Prometeu

- Descricao: conjunto de helpers para URLs, headers, replay, discovery, diagnostics e leitura de capacidade do runtime AG-UI da plataforma.
- Tags: ag-ui, helpers, runtime
- Tipo: helper
- Arquivo: [packages/ag-ui-runtime/prometeu-helpers.js](../packages/ag-ui-runtime/prometeu-helpers.js)
- Linguagem: JavaScript
- Responsabilidade principal: padronizar detalhes de protocolo do runtime AG-UI da plataforma.
- Dependencias principais: JavaScript padrao
- Acoplamento forte com dominio?: Medio. E infra compartilhada do runtime AG-UI do projeto.
- Uso atual observado: Sim. Consumido por [packages/ag-ui-runtime/official-http-client.js](../packages/ag-ui-runtime/official-http-client.js) e pelo pacote runtime compartilhado.
- Seguro reutilizar como esta?: Sim.
- Riscos ou limitacoes: conhece naming e endpoints especificos da plataforma.
- Sugestao de melhoria: separar helpers de transporte, diagnostics e capability mapping em arquivos menores se o modulo crescer mais.
- Prioridade: Alta

### Fachada publica do runtime AG-UI

- Descricao: entrada publica do pacote `@prometeu/ag-ui-runtime`, reexportando cliente oficial, store, sidecar, renderers e validadores para consumidores browser e modulo.
- Tags: ag-ui, facade, package
- Tipo: outro
- Arquivo: [packages/ag-ui-runtime/index.js](../packages/ag-ui-runtime/index.js)
- Linguagem: JavaScript
- Responsabilidade principal: oferecer superficie unica e consistente de consumo do runtime AG-UI do projeto.
- Dependencias principais: [packages/ag-ui-runtime/official-http-client.js](../packages/ag-ui-runtime/official-http-client.js), [app/ui/static/js/shared/ag-ui-state-store.js](../app/ui/static/js/shared/ag-ui-state-store.js), validadores e renderers compartilhados
- Acoplamento forte com dominio?: Medio. E a fachada oficial do ecossistema AG-UI da plataforma.
- Uso atual observado: Sim. Encapsula e reexporta varios modulos compartilhados que tambem aparecem no frontend browser.
- Seguro reutilizar como esta?: Sim. Deve ser a porta de entrada para novos consumidores do runtime.
- Riscos ou limitacoes: qualquer export novo aqui vira contrato publico do pacote.
- Sugestao de melhoria: manter changelog ou inventario de exports publicos para evitar regressao silenciosa.
- Prioridade: Alta

## Reuso com cautela

### local-crypto-api

- Descricao: helper local de criptografia com PBKDF2 e AES-GCM para fluxos de armazenamento local no browser.
- Tags: criptografia, browser, local-storage
- Tipo: helper
- Arquivo: [app/ui/static/js/shared/local-crypto-api.js](../app/ui/static/js/shared/local-crypto-api.js)
- Linguagem: JavaScript
- Responsabilidade principal: encapsular criptografia local de payloads no navegador.
- Dependencias principais: Web Crypto API
- Acoplamento forte com dominio?: Baixo. O problema resolvido e generico, mas a implementacao carrega convencoes especificas da UI.
- Uso atual observado: Potencial. O modulo e reutilizavel, mas o proprio codigo sinaliza cautela e a analise nao o confirmou como base preferencial para novos fluxos.
- Seguro reutilizar como esta?: Com cautela. So quando houver motivo forte para criptografia local no browser e alinhamento com o trilho oficial.
- Riscos ou limitacoes: o proprio codigo aponta sinal de depreciacao; pode ser uma superficie inadequada para padronizar novos fluxos.
- Sugestao de melhoria: definir claramente se o modulo segue suportado ou se deve ser substituido por uma alternativa oficial do projeto.
- Prioridade: Baixa

## Resumo pragmatico

- Antes de escrever codigo novo de backend, procure primeiro em [src/core/base_correlation_component.py](../src/core/base_correlation_component.py), [src/core/logging_system.py](../src/core/logging_system.py), [src/core/external_retry.py](../src/core/external_retry.py), [src/core/database_connection_manager.py](../src/core/database_connection_manager.py) e [src/core/vector_store_factory.py](../src/core/vector_store_factory.py).
- Antes de abrir novo caminho para YAML agentic, consulte [src/config/agentic_assembly/assembly_service.py](../src/config/agentic_assembly/assembly_service.py) e [src/config/agentic_assembly/parsers/yaml_loader.py](../src/config/agentic_assembly/parsers/yaml_loader.py).
- Antes de criar utilitario novo no frontend admin, verifique [app/ui/static/js/shared/admin-api-client.js](../app/ui/static/js/shared/admin-api-client.js), [app/ui/static/js/shared/admin-runner-request-core.js](../app/ui/static/js/shared/admin-runner-request-core.js) e [app/ui/static/js/shared/admin-runner-core.js](../app/ui/static/js/shared/admin-runner-core.js).
- Antes de criar runtime AG-UI novo, comece por [packages/ag-ui-runtime/index.js](../packages/ag-ui-runtime/index.js), [packages/ag-ui-runtime/official-http-client.js](../packages/ag-ui-runtime/official-http-client.js) e [app/ui/static/js/shared/ag-ui-state-store.js](../app/ui/static/js/shared/ag-ui-state-store.js).
