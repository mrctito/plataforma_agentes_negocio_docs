# Manual técnico e operacional: pipeline de schema metadata como pré-requisito operacional do NL2SQL

## 1. O que é esta feature

O pipeline operacional de schema metadata é a cadeia técnica que prepara o contexto consumido pelo NL2SQL. O caminho real confirmado no código não é um único módulo. Ele é um encadeamento entre ETL dedicado, exportação dinâmica, ingestão/indexação e runtime.

O fluxo canônico observado no slice PDV é:

1. YAML de ETL: rag-config-pdv-schema-metadata-etl.yaml.
2. YAML de ingestão: rag-config-pdv-schema-metadata-ingest.yaml.
3. YAML runtime: rag-config-pdv-vendas-nl2sql.yaml.

O elo crítico entre os três é o mesmo vectorstore_id: schema_metadata_pdv.

## 2. Fluxo técnico ponta a ponta

### 2.1. Ativação do ETL dedicado

O ponto de entrada técnico do ETL é o bloco extract_transform_load. O orquestrador dedicado de ETL só executa o ramo schema_metadata quando:

- extract_transform_load.enabled=true;
- extract_transform_load.schema_metadata.enabled=true.

O arquivo de configuração canônico do PDV confirma esse formato e explicita origem, destino e política do vector store.

### 2.2. Execução do TableSchemaMetadataETLProcessor

O ExtractTransformLoadOrchestrator delega o ramo schema_metadata para TableSchemaMetadataETLProcessor.

Esse processor:

1. valida o contrato bruto;
2. normaliza source_database e target_database;
3. cria engine de origem com retry;
4. testa conexão de origem e destino;
5. garante database e schema de destino no writer;
6. lista tabelas disponíveis;
7. aplica filtro por padrões;
8. extrai colunas, PKs e FKs;
9. grava tudo em dbschemas;
10. opcionalmente grava sample rows.

### 2.3. Materialização do catálogo em dbschemas

O destino do ETL não é o vector store. É o catálogo dbschemas. Esse detalhe é decisivo para entender a arquitetura.

O processor usa MetadataWriter para:

- ensure_database;
- ensure_schema;
- upsert_table;
- upsert_columns_batch;
- insert_primary_key;
- insert_foreign_keys;
- upsert_table_sample.

O resultado operacional devolve métricas como tables_processed, columns_processed, pks_processed e fks_processed.

### 2.4. Exportação por tabela

Depois do ETL, o SchemaMetadataJsonExporter lê dbschemas e monta um documento JSON por tabela.

Esse exportador exige:

- dbschemas_dsn;
- database_code;
- opcionalmente schema_name;
- include_sample_rows opcional.

O documento final inclui estrutura de database, schema, tabela, colunas, PK, FKs e amostras quando autorizadas. O próprio exportador registra document_type=schema_metadata.

### 2.5. Ingestão dinâmica e indexação

O YAML de ingestão canônico usa dynamic_data com um único source:

- script: src/ingestion_layer/schema_metadata/schema_metadata_exporter.py:SchemaMetadataJsonExporter;
- output_mode: memory;
- cleanup_after_ingestion: true.

Os testes confirmam que essa trilha gera BaseDocuments válidos e que a indexação resultante carrega metadados como:

- source=dynamic_data;
- source_identifier=schema_metadata_exporter;
- document_id por tabela.

### 2.6. Consumo pelo runtime de NL2SQL

No runtime, o Nl2SqlService exige que schema_metadata exista como dicionário e que schema_metadata.vectorstore_id esteja preenchido. O serviço também injeta schema_metadata.sql_dialect antes de chamar a SqlSchemaRagToolFactory.

Ou seja, o runtime só consegue operar se o bloco schema_metadata apontar para o acervo vetorial preparado anteriormente.

## 3. Contratos YAML que mudam o comportamento

### 3.1. YAML de ETL

O contrato canônico do PDV confirma:

- vector_store.type=qdrant;
- vector_store.id=schema_metadata_pdv;
- vector_store.if_exists=overwrite;
- extract_transform_load.schema_metadata.enabled=true;
- source_database.tables=["*"];
- source_database.include_sample_rows=false;
- target_database.schema=dbschemas.

Impacto prático:

- o ETL lê todas as tabelas da origem por default canônico do caso PDV;
- sample rows ficam desligadas por padrão;
- o catálogo vai para dbschemas.

### 3.2. YAML de ingestão

O contrato canônico de ingestão confirma:

- mesmo vector_store.id=schema_metadata_pdv;
- dynamic_data.enabled=true;
- source script apontando para SchemaMetadataJsonExporter;
- output_mode=memory;
- include_sample_rows=false;
- database_code=pdv_demo.

Impacto prático:

- o material exportado vai direto para a indexação sem etapa intermediária de arquivo;
- a indexação permanece alinhada ao mesmo vectorstore_id esperado pelo runtime.

### 3.3. YAML runtime de NL2SQL

O contrato canônico de runtime confirma:

- schema_metadata.enabled=true;
- schema_metadata.vectorstore_id=schema_metadata_pdv;
- schema_metadata.sql_dialect=postgresql;
- vector_store.id=schema_metadata_pdv.

Impacto prático:

- o runtime aponta explicitamente para o acervo vetorial preparado;
- o dialeto é declarado, não inferido implicitamente.

## 4. Componentes técnicos principais

### 4.1. ExtractTransformLoadOrchestrator

Responsabilidade: ligar ou desligar o pipeline schema_metadata conforme o YAML.

Ponto forte: não mistura detalhe do schema metadata com os pipelines Apify. Ele apenas coordena o ramo habilitado.

### 4.2. TableSchemaMetadataETLProcessor

Responsabilidade: fazer leitura estrutural da origem e carga no catálogo dbschemas.

Pontos fortes confirmados:

- retry explícito na criação de engine;
- teste de conexão antes da carga;
- filtro por wildcard de tabelas;
- normalização de database_code;
- serialização segura de tipos como datetime, date e Decimal;
- sample rows por opt-in explícito.

### 4.3. MetadataReaderFactory

Responsabilidade: escolher o reader certo pela DSN.

SGBDs confirmados:

- PostgreSQL;
- SQL Server;
- Oracle.

Ponto forte: o ETL novo é genérico no contrato de origem. O mesmo processor pode ler mais de um SGBD suportado sem reescrever o pipeline.

### 4.4. SchemaMetadataJsonExporter

Responsabilidade: converter dbschemas em documentos JSON por tabela.

Pontos fortes:

- retry explícito para acesso ao dbschemas;
- output_mode file ou memory;
- metadata rica por documento;
- document_type=schema_metadata;
- documents_metadata consolidados no resultado.

### 4.5. JsonContentProcessor profile schema_metadata

Responsabilidade: preservar integridade estrutural durante a ingestão JSON.

Comportamento confirmado:

- preserve_structure=true;
- flatten_arrays=false;
- max_chunks_per_document=1;
- include_json_structure_metadata=false;
- allow_domain_processing=false;
- allow_fallback_chunking=false.

Impacto prático: schema metadata não é tratado como JSON genérico de domínio. Ele entra em uma rota mais restrita e mais fiel à estrutura técnica.

### 4.6. SqlSchemaRagToolFactory

Responsabilidade: consumir o acervo vetorial preparado.

Ela exige:

- user_session.correlation_id;
- user_session.user_email;
- schema_metadata.vectorstore_id;
- schema_metadata.sql_dialect.

Também confirma que o dialeto suportado no runtime é postgresql, mysql ou mssql, mesmo que o ETL de extração tenha readers confirmados para PostgreSQL, SQL Server e Oracle.

## 5. Por que esse pipeline funciona bem em bases grandes

O desenho observado no código tem quatro vantagens objetivas para bases de ERP grandes.

### 5.1. Não tenta enfiar o schema inteiro no prompt

O runtime schema_rag_sql busca top_k documentos relevantes no vector store. Isso só funciona porque o pipeline materializa um documento por tabela, e não um dump monolítico.

### 5.2. Leva estrutura relacional, não só nomes crus

O exportador carrega colunas, PKs, FKs e descrições estruturais. Isso ajuda quando a nomenclatura do ERP é pouco amigável.

### 5.3. Desacopla leitura do banco vivo do momento da pergunta

O runtime não precisa abrir o banco operacional para inspecionar schema a cada consulta. Isso melhora previsibilidade e reduz acoplamento operacional.

### 5.4. Mantém governança de dados por default

O caso canônico lido desativa sample rows por padrão. Isso preserva valor técnico do catálogo sem abrir exposição desnecessária de dado real.

## 6. Contratos, entradas e saídas

### 6.1. Entrada do ETL

O processor exige um objeto schema_metadata com:

- source_database.database_dsn;
- source_database.database_type;
- source_database.database_name;
- source_database.schema;
- target_database.database_dsn;
- target_database.database_type;
- target_database.database_name;
- target_database.database_code;
- target_database.schema.

source_database.tables é opcional e defaulta para ["*"]. include_sample_rows é opcional e defaulta para false, mas precisa ser booleano quando informado.

### 6.2. Saída do ETL

O resultado padrão inclui:

- pipeline=schema_metadata;
- success;
- errors;
- warnings;
- documents_processed;
- chunks_created;
- chunks_stored;
- metadata com tables_processed, columns_processed, pks_processed, fks_processed e dados de origem/destino.

### 6.3. Saída do exportador

O exportador devolve DynamicDataResult com:

- success;
- documents ou file_paths;
- total_documents;
- content_type=json;
- metadata consolidada.

### 6.4. Entrada do runtime

O Nl2SqlService exige schema_metadata como dicionário e vectorstore_id não vazio. Se isso faltar, o serviço falha com ValueError explícito.

## 7. O que acontece em caso de sucesso

O sucesso operacional completo do pré-requisito acontece quando:

1. o ETL grava o catálogo em dbschemas;
2. o exportador encontra tabelas para o database_code e schema escolhidos;
3. a ingestão dinâmica indexa documentos schema_metadata no vector store esperado;
4. o runtime aponta para esse vectorstore_id e recebe sql_dialect válido.

Quando esse encadeamento fecha, a engine schema_rag_sql consegue recuperar documentos válidos de schema metadata e montar contexto controlado.

## 8. O que acontece em caso de erro

### 8.1. Contrato ETL inválido

Sintoma: erro rápido com mensagem como schema_metadata deve ser um objeto, source_database e target_database são obrigatórios ou campo obrigatório ausente.

Impacto: o catálogo nem começa a ser preparado.

### 8.2. Conexão de origem ou destino falha

Sintoma: result.errors recebe falha na conexão com o banco de origem ou destino.

Impacto: dbschemas não é atualizado.

### 8.3. Nenhuma tabela encontrada

Sintoma: IngestResult com success=false e erro de nenhuma tabela encontrada para processar.

Impacto: ETL roda, mas não materializa conteúdo útil.

### 8.4. Exportação vazia

Sintoma: exportador devolve total_documents=0.

Impacto: o vector store pode continuar vazio mesmo com dbschemas existente.

### 8.5. Runtime sem schema_metadata válido

Sintoma: NL2SQL falha com ValueError por schema_metadata ausente ou vectorstore_id vazio.

Impacto: a geração de SQL não começa.

## 9. Observabilidade e diagnóstico

### 9.1. Sinais do ETL

O ETL devolve métricas explícitas de tabelas, colunas, PKs e FKs processadas. Esse é o primeiro ponto de checagem.

### 9.2. Sinais do exportador

O exportador registra documents_generated e inclui documents_metadata no resultado. Esse é o segundo ponto de checagem.

### 9.3. Sinais da ingestão dinâmica

Os testes confirmam metadados de chunk como source_identifier=schema_metadata_exporter e document_id por tabela. Esse é o terceiro ponto de checagem.

### 9.4. Sinais do runtime NL2SQL

O runtime devolve vectorstore_id e dialect no execution_context. Esse é o quarto ponto de checagem.

O diagnóstico certo segue esta ordem:

1. ETL preencheu dbschemas?
2. Exportador gerou documentos?
3. Ingestão indexou documentos schema_metadata?
4. Runtime aponta para o mesmo vectorstore_id?

## 10. Troubleshooting

### 10.1. Sintoma: o ETL terminou, mas NL2SQL continua ruim

Causa provável: dbschemas foi atualizado, mas a ingestão vetorial não rodou, rodou com outro vectorstore_id ou indexou zero documentos.

Como confirmar: comparar ETL YAML, ingest YAML e runtime YAML pelo mesmo id do vector store.

### 10.2. Sintoma: o runtime responde 400 por configuração inválida

Causa provável: schema_metadata ausente ou vectorstore_id vazio no YAML runtime.

Como confirmar: olhar a mensagem gerada pelo _prepare_yaml_config do Nl2SqlService.

### 10.3. Sintoma: o ETL roda, mas não encontra tabelas

Causa provável: schema errado, filtro tables incompatível ou database de origem sem o schema esperado.

Como confirmar: revisar source_database.schema e source_database.tables.

### 10.4. Sintoma: o contexto fica pobre demais

Causa provável: sample rows desligadas, descrições ausentes na origem ou catálogo desatualizado.

Como confirmar: revisar o conteúdo exportado por tabela e verificar se a atualização do catálogo está recente.

### 10.5. Sintoma: reader não reconhece a origem

Causa provável: DSN com prefixo fora do conjunto suportado.

Como confirmar: MetadataReaderFactory.detect_sgbd_type retorna None.

## 11. Pontos fortes técnicos

- Retry com backoff exponencial em ETL e exportação.
- Filtro por wildcard para tabelas sem hardcode manual de lista completa.
- Normalização segura de database_code.
- Sample rows sob opt-in explícito.
- Documento por tabela, o que favorece recuperação top_k.
- Perfil de ingestão restrito para preservar estrutura.
- Mesmo vectorstore_id costurando ETL, ingestão e runtime.
- Separação nítida entre catálogo persistido e runtime de pergunta.

## 12. Como colocar para funcionar

O caminho operacional confirmado pelo código e pelos YAMLs canônicos é:

1. executar o YAML de ETL de schema metadata para preencher dbschemas;
2. executar o YAML de ingestão de schema metadata para indexar o material no vector store;
3. garantir que o YAML runtime de NL2SQL aponte para o mesmo schema_metadata.vectorstore_id e declare sql_dialect;
4. só depois chamar /config/nl2sql/generate.

Se qualquer etapa for omitida, o NL2SQL perde o pré-requisito operacional que o torna confiável.

## 13. Explicação 101

Tecnicamente, o pipeline é uma linha de produção. Primeiro ele lê a planta do banco. Depois guarda essa planta num catálogo próprio. Depois transforma esse catálogo em documentos pesquisáveis. Por fim, o NL2SQL consulta esses documentos em vez de tentar descobrir o banco na hora. Esse é o motivo pelo qual a feature fica mais robusta em bases grandes.

## 14. Checklist de entendimento

- Entendi que dbschemas e vector store não são a mesma coisa.
- Entendi que o pré-requisito do NL2SQL exige ETL e ingestão, não só runtime.
- Entendi que o mesmo vectorstore_id precisa atravessar as três etapas.
- Entendi por que o exportador trabalha por tabela.
- Entendi por que sample rows ficam desligadas por padrão no caminho canônico.

## 15. Evidências no código

- src/etl_layer/orchestrator.py
  - Motivo da leitura: confirmar o wiring do ramo schema_metadata no ETL.
  - Símbolo relevante: ExtractTransformLoadOrchestrator.
  - Comportamento confirmado: dispara TableSchemaMetadataETLProcessor quando schema_metadata.enabled=true.

- src/etl_layer/providers/data/table_schema_metadata_processor.py
  - Motivo da leitura: confirmar o fluxo moderno do ETL.
  - Símbolo relevante: TableSchemaMetadataETLProcessor.
  - Comportamento confirmado: valida contrato, conecta, filtra tabelas, extrai colunas/PK/FK e grava em dbschemas.

- src/schema_metadata/reader_factory.py
  - Motivo da leitura: confirmar SGBDs suportados na extração.
  - Símbolo relevante: MetadataReaderFactory.
  - Comportamento confirmado: PostgreSQL, SQL Server e Oracle.

- src/ingestion_layer/schema_metadata/schema_metadata_exporter.py
  - Motivo da leitura: confirmar a exportação por tabela.
  - Símbolo relevante: SchemaMetadataJsonExporter.
  - Comportamento confirmado: um JSON por tabela com document_type schema_metadata.

- src/ingestion_layer/processors/json_processor.py
  - Motivo da leitura: confirmar o perfil especializado de ingestão.
  - Símbolo relevante: perfil schema_metadata.
  - Comportamento confirmado: um chunk por documento, sem fallback e sem domain processing.

- tests/unit/ingestion_layer/schema_metadata/test_02-46-01_schema_metadata_exporter.py
  - Motivo da leitura: confirmar a trilha exportador -> loader -> documento.
  - Símbolo relevante: test_quando_loader_executa_exportador_schema_metadata_entao_retorna_json_document.
  - Comportamento confirmado: documento com dynamic_data_class=SchemaMetadataJsonExporter e document_type=schema_metadata.

- tests/unit/ingestion_layer/test_02-28-09_dynamic_data.py
  - Motivo da leitura: confirmar a indexação operacional.
  - Símbolo relevante: test_quando_pipeline_schema_metadata_roda_via_dynamic_data_entao_indexa_documentos.
  - Comportamento confirmado: chunk indexado com source_identifier schema_metadata_exporter.

- tests/unit/test_02-04-74_nl2sql_pdv_yaml_contract.py
  - Motivo da leitura: confirmar o encadeamento canônico do caso PDV.
  - Símbolo relevante: contratos ETL, ingest e runtime.
  - Comportamento confirmado: vectorstore_id alinhado entre as três etapas.

- src/api/services/nl2sql_service.py
  - Motivo da leitura: confirmar o consumo final pelo runtime.
  - Símbolo relevante: _prepare_yaml_config.
  - Comportamento confirmado: schema_metadata e vectorstore_id são obrigatórios para NL2SQL.
