# Manual conceitual, executivo, comercial e estratégico: pipeline de schema metadata como pré-requisito operacional do NL2SQL

## 1. O que é esta feature

O pipeline de schema metadata é a esteira que transforma a estrutura de uma base relacional em ativo operacional consultável pelo NL2SQL. Ele não é um detalhe secundário do produto. Ele é a base que permite ao runtime de geração de SQL trabalhar com evidência concreta de schema, em vez de depender de intuição do modelo ou de acesso direto ao banco transacional em tempo real.

Na prática, essa capacidade acontece em quatro camadas encadeadas:

1. ETL estrutural que lê origem e grava catálogo em dbschemas.
2. Exportação técnica que gera um documento JSON por tabela.
3. Ingestão dinâmica que indexa esses documentos no vector store correto.
4. Runtime de NL2SQL que consulta esse acervo pelo schema_metadata.vectorstore_id.

Por isso, schema metadata não é apenas um catálogo técnico parado. É o pré-requisito operacional que transforma um banco grande, confuso e cheio de nomenclatura de ERP em contexto recuperável para geração segura de SQL.

## 2. Que problema ela resolve

NL2SQL funciona bem quando o sistema consegue responder três perguntas antes de gerar SQL:

- quais tabelas existem de verdade;
- como elas se relacionam;
- quais colunas e chaves fazem sentido para a pergunta.

Sem o pipeline de schema metadata, o runtime teria duas alternativas ruins.

Primeira alternativa ruim: mandar o schema inteiro no prompt. Isso explode contexto, piora custo, degrada precisão e não escala para bases corporativas com milhares de tabelas e colunas.

Segunda alternativa ruim: deixar o modelo inferir a estrutura “no escuro”. Isso cria SQL frágil, alucina tabelas e confunde nomes técnicos pouco significativos, algo comum em bases de ERP.

O pipeline existe para quebrar esse problema em etapas controladas. Ele separa extração estrutural, materialização documental, indexação e consumo por RAG. Esse desenho reduz ruído, melhora rastreabilidade e preserva o runtime de NL2SQL de acoplamento com leitura direta do banco transacional.

## 3. Visão executiva

Executivamente, o maior valor do pipeline é reduzir risco operacional na camada de analytics assistido.

Ele evita que uma feature sensível, como geração de SQL por linguagem natural, dependa de improviso ou de conexão online com schema bruto toda vez que alguém faz uma pergunta. Em vez disso, a plataforma trabalha sobre um acervo técnico preparado, revisável e reprocessável.

Isso aumenta previsibilidade, porque a organização passa a ter uma trilha clara de preparo dos metadados antes do uso analítico. Também reduz incidentes causados por schema inconsistente, contexto insuficiente ou leitura sem governança de estruturas internas do banco.

## 4. Visão comercial

Comercialmente, esse pipeline ajuda a sustentar uma promessa importante: a plataforma não responde perguntas de dados “porque adivinhou o banco”, e sim porque prepara o conhecimento estrutural da base antes de operar sobre ele.

Isso é especialmente forte em cenários de ERP, onde os nomes de tabelas e colunas nem sempre são amigáveis. O produto ganha um argumento claro:

- consegue trabalhar com bases grandes;
- organiza estrutura técnica antes da pergunta do usuário;
- reduz dependência de conhecimento tribal do banco;
- melhora a explicabilidade do que o NL2SQL está usando como contexto.

O benefício percebido pelo cliente é simples: a geração de SQL deixa de parecer mágica opaca e passa a parecer operação apoiada por catálogo técnico estruturado.

## 5. Visão estratégica

Estrategicamente, o pipeline fortalece a plataforma em quatro frentes.

Primeira: desacopla analytics assistido do banco operacional vivo.

Segunda: transforma schema em ativo de plataforma, não só em detalhe de infraestrutura.

Terceira: permite reuso do mesmo acervo por NL2SQL, tools como schema_rag_sql e futuras experiências agentic orientadas a dados.

Quarta: cria um caminho escalável para bases grandes, porque a recuperação passa a ser por documentos de schema relevantes, e não por dump integral do banco.

## 6. Conceitos necessários para entender

### 6.1. Schema metadata

Schema metadata é a representação estruturada do banco: database, schema, tabela, colunas, tipos, chaves primárias, chaves estrangeiras e, quando permitido explicitamente, amostras de linhas. Ele existe para que o runtime trabalhe com fatos sobre a base, e não com suposição.

### 6.2. dbschemas

dbschemas é o catálogo técnico persistido onde o ETL grava a estrutura extraída. Ele funciona como base intermediária estável entre a leitura do banco de origem e a indexação vetorial usada pelo NL2SQL.

### 6.3. Exportação por tabela

O exportador não gera um documento monolítico do banco inteiro. Ele gera um JSON por tabela. Isso é crítico para escalabilidade, porque o motor de recuperação consegue buscar apenas os documentos mais relevantes para a pergunta.

### 6.4. Vector store de schema metadata

O runtime de NL2SQL não lê dbschemas diretamente. Ele lê documentos já indexados no vector store apontado por schema_metadata.vectorstore_id. Em outras palavras, dbschemas é o catálogo persistido; o vector store é a superfície de recuperação semântica.

### 6.5. Processing profile schema_metadata

Na ingestão JSON, schema metadata usa um perfil próprio: preserva estrutura, não achata arrays, limita a um chunk por documento e desliga processamento de domínio e fallback chunking. Isso existe para manter fidelidade estrutural e evitar fragmentação semântica desnecessária.

### 6.6. Opt-in de sample rows

O pipeline só coleta amostras de linhas quando include_sample_rows=true. O default observado no caminho canônico do PDV é false. Isso é um ponto forte de governança, porque reduz exposição acidental de dados reais.

## 7. Como a feature funciona por dentro

O pipeline começa no YAML de ETL. Quando extract_transform_load.schema_metadata.enabled está ativo, o orquestrador dedicado de ETL dispara o TableSchemaMetadataETLProcessor. Esse processor valida o contrato, testa conexões, lê tabelas da origem, aplica filtro por wildcard, extrai colunas, PKs, FKs e grava tudo em dbschemas.

Depois entra a segunda camada: o SchemaMetadataJsonExporter lê dbschemas e monta um documento JSON por tabela. Esse documento já nasce com metadados técnicos suficientes para ser indexado como schema_metadata.

Na terceira camada, a ingestão dinâmica carrega o exportador, produz BaseDocuments e envia esse material para o vector store configurado. A própria trilha de testes confirma que os documentos indexados mantêm document_type igual a schema_metadata e source_identifier igual a schema_metadata_exporter.

Só então a quarta camada, o runtime de NL2SQL, consegue operar. O serviço exige schema_metadata.vectorstore_id, injeta sql_dialect no bloco schema_metadata e chama a engine schema_rag_sql, que recupera somente documentos válidos de schema metadata.

## 8. Divisão em etapas ou submódulos

### 8.1. ETL estrutural da origem para dbschemas

É a etapa que transforma o banco de origem em catálogo técnico persistido. Ela existe para separar leitura estrutural da base viva do consumo semântico posterior.

Recebe DSN, tipo de banco, nome lógico, schema, filtros de tabela e configuração de destino. Entrega tabelas, colunas, PKs, FKs e sample rows opcionais gravados no destino.

O que pode dar errado: contrato incompleto, conexão inválida, nenhum match nas tabelas ou schema alvo inconsistente.

Valor para o fluxo completo: cria a fonte de verdade técnica do schema.

### 8.2. Materialização documental por tabela

É a etapa que lê dbschemas e gera documentos JSON estruturados por tabela. Ela existe porque o motor de recuperação trabalha melhor com documentos discretos do que com um superdocumento de banco inteiro.

Recebe database_code e, opcionalmente, schema_name. Entrega documentos com database, schema, tabela, colunas, PK, FKs e sample rows quando autorizadas.

O que pode dar errado: database_code inexistente, dbschemas_dsn ausente ou exportação vazia.

Valor para o fluxo completo: prepara o material que o vector store consegue indexar e recuperar semanticamente.

### 8.3. Ingestão vetorial do schema

É a etapa que transforma o JSON exportado em documento indexado. Ela existe para ligar o catálogo técnico persistido ao mecanismo de recuperação do NL2SQL.

Recebe a saída do exportador e entrega chunks indexados no vector store correto, com metadados preservando document_type=schema_metadata.

O que pode dar errado: vectorstore_id incorreto, pipeline de ingestão não executado ou perfil errado de processamento.

Valor para o fluxo completo: torna o schema recuperável pelo runtime.

### 8.4. Consumo semântico pelo NL2SQL

É a etapa final, onde o runtime consulta apenas o que já foi preparado. Ela existe para evitar consulta direta ao banco estrutural e para reduzir contexto ao conjunto top_k de documentos relevantes.

Recebe prompt, vectorstore_id e dialeto. Entrega SQL proposta baseada em schema relevante, não no schema inteiro.

## 9. Pipeline principal

1. O YAML de ETL ativa extract_transform_load.schema_metadata.
2. O orquestrador de ETL dispara o TableSchemaMetadataETLProcessor.
3. O processor testa origem e destino, filtra tabelas e grava metadados em dbschemas.
4. O YAML de ingestão dinâmica chama SchemaMetadataJsonExporter.
5. O exportador gera um documento JSON por tabela.
6. A ingestão indexa os documentos no vector store do schema metadata.
7. O YAML runtime de NL2SQL aponta para o mesmo vectorstore_id e define o dialeto.
8. A engine schema_rag_sql recupera documentos relevantes e gera SQL com guardrail de somente leitura.

Essa ordem importa. Se qualquer elo for pulado, o runtime perde contexto operacional real.

## 10. Decisões técnicas e trade-offs

### 10.1. Separar ETL de schema do runtime de NL2SQL

Ganho: reduz acoplamento com banco operacional e melhora reprocessamento.

Custo: exige pipeline prévio, e não operação instantânea direto no schema vivo.

### 10.2. Persistir em dbschemas antes de indexar

Ganho: cria catálogo auditável e reutilizável.

Custo: adiciona uma camada intermediária.

### 10.3. Exportar por tabela em vez de um dump único

Ganho: escala melhor para bases grandes e melhora a recuperação top_k.

Custo: exige mais disciplina de indexação e metadados por documento.

### 10.4. sample rows por opt-in explícito

Ganho: reduz risco de exposição de dados sensíveis.

Custo: quando desativado, o contexto perde um possível apoio diagnóstico.

### 10.5. Perfil de ingestão especializado para schema_metadata

Ganho: preserva estrutura e evita chunking destrutivo.

Custo: menos flexibilidade para heurísticas genéricas de processamento JSON.

## 11. Pontos fortes

Os pontos fortes confirmados no código são estes.

- Escala para bases grandes porque o runtime consulta documentos relevantes por tabela, não o schema inteiro.
- É genérico no contrato: o ETL não hardcodeia domínio de negócio e o exportador opera por database_code e schema_name.
- Suporta mais de um SGBD na leitura estrutural, com readers confirmados para PostgreSQL, SQL Server e Oracle.
- Usa retry explícito com backoff exponencial em pontos críticos de acesso externo, como criação de engine e exportação.
- Mantém sample rows desligadas por padrão no caminho canônico de PDV, o que melhora governança de dados.
- Protege fidelidade estrutural com perfil schema_metadata de um único chunk, sem fallback genérico.
- Mantém separação saudável entre catálogo persistido, indexação vetorial e runtime de pergunta.
- Reforça precisão do NL2SQL em bases de ERP com nomes pouco significativos, porque leva junto PKs, FKs, descrições e topologia relacional.

## 12. O que acontece em caso de sucesso

Sucesso real significa mais do que “o ETL rodou”. O caminho completo só está pronto quando:

1. dbschemas recebeu os metadados da origem;
2. o exportador conseguiu gerar documentos por tabela;
3. a ingestão indexou esses documentos no vector store correto;
4. o runtime aponta para esse mesmo vectorstore_id.

Quando isso acontece, o NL2SQL consegue buscar schema_metadata relevante e propor SQL com contexto técnico suficiente.

## 13. O que acontece em caso de erro

Os cenários de erro mais importantes são:

- contrato ETL incompleto para source_database ou target_database;
- falha de conexão com banco de origem ou destino;
- filtro de tabelas que não encontra nada;
- database_code ou schema_name errados na exportação;
- ingestão dinâmica não executada ou executada no vector store errado;
- runtime de NL2SQL sem schema_metadata.vectorstore_id.

O efeito prático é direto: o runtime de NL2SQL continua existindo, mas perde a base operacional que o torna confiável.

## 14. Impacto técnico

Tecnicamente, o pipeline encapsula um problema difícil com separação de responsabilidades correta. O ETL lê schema. O exportador materializa documentos. A ingestão indexa. O runtime consulta. Isso melhora coesão, reuso e observabilidade.

## 15. Impacto executivo

Executivamente, o pipeline reduz risco na camada de análise assistida e aumenta previsibilidade do rollout de NL2SQL em clientes com bases complexas.

## 16. Impacto comercial

Comercialmente, o principal diferencial é explicar que a plataforma não “chuta SQL”. Ela prepara a base estrutural primeiro. Isso é particularmente forte em projetos com ERP, legado e nomenclatura técnica difícil.

## 17. Impacto estratégico

Estrategicamente, schema metadata vira infraestrutura de conhecimento para a plataforma. O mesmo acervo pode sustentar NL2SQL hoje e outras experiências de agentes orientados a dados depois.

## 18. Explicação 101

Imagine que o NL2SQL é um consultor que precisa conhecer o mapa da empresa antes de responder. O pipeline de schema metadata é quem constrói esse mapa. Primeiro ele lê a estrutura do banco. Depois organiza isso em catálogo. Depois transforma esse catálogo em documentos pesquisáveis. Só então o consultor recebe a pergunta. Sem esse mapa, ele até tenta responder, mas responde muito pior.

## 19. Limites e pegadinhas

- ETL concluído não significa que o vector store já está pronto.
- dbschemas pronto não significa que a ingestão dinâmica já aconteceu.
- vectorstore_id correto no runtime não compensa ingestão vazia ou desatualizada.
- sample rows desligadas melhoram governança, mas reduzem contexto descritivo de valores.
- reader suportado na extração não significa dialeto suportado no runtime final de NL2SQL.

## 20. Checklist de entendimento

- Entendi que schema metadata é pré-requisito operacional do NL2SQL.
- Entendi a diferença entre dbschemas e vector store.
- Entendi por que existe um documento por tabela.
- Entendi por que a ingestão JSON usa perfil especial para schema metadata.
- Entendi o papel do vectorstore_id como elo entre preparação e runtime.
- Entendi os principais pontos fortes do pipeline.

## 21. Evidências no código

- src/etl_layer/providers/data/table_schema_metadata_processor.py
  - Motivo da leitura: confirmar o ETL estrutural moderno.
  - Símbolo relevante: TableSchemaMetadataETLProcessor.
  - Comportamento confirmado: origem -> dbschemas, filtro por tabela, PK/FK e sample rows por opt-in.

- src/ingestion_layer/schema_metadata/schema_metadata_exporter.py
  - Motivo da leitura: confirmar a materialização documental.
  - Símbolo relevante: SchemaMetadataJsonExporter.
  - Comportamento confirmado: um JSON por tabela com document_type schema_metadata.

- tests/unit/ingestion_layer/test_02-28-09_dynamic_data.py
  - Motivo da leitura: confirmar indexação operacional.
  - Símbolo relevante: test_quando_pipeline_schema_metadata_roda_via_dynamic_data_entao_indexa_documentos.
  - Comportamento confirmado: loader -> documento -> chunk -> vector store com source_identifier schema_metadata_exporter.

- src/api/services/nl2sql_service.py
  - Motivo da leitura: confirmar o pré-requisito no runtime.
  - Símbolo relevante: _prepare_yaml_config.
  - Comportamento confirmado: schema_metadata precisa existir e vectorstore_id é obrigatório.

- tests/unit/test_02-04-74_nl2sql_pdv_yaml_contract.py
  - Motivo da leitura: confirmar o encadeamento canônico do PDV.
  - Símbolo relevante: contratos ETL, ingest e runtime.
  - Comportamento confirmado: mesmo vectorstore_id entre ETL, ingestão e NL2SQL.
