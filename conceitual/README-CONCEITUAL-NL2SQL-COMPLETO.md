# Manual conceitual, executivo, comercial e estrategico: NL2SQL completo

## 1. O que e esta feature

NL2SQL, neste projeto, e a capacidade de transformar uma pergunta em linguagem natural em uma SQL proposta, revisavel e protegida por guardrails. O ponto central e este: ele nao tenta adivinhar uma consulta olhando o banco inteiro de forma cega nem entrega execucao automatica irrestrita. Ele primeiro recupera contexto de schema relevante via busca semantica sobre metadados estruturados, depois pede ao LLM uma proposta SQL no dialeto escolhido e, por fim, valida se a resposta e comprovadamente somente leitura.

Isso faz do slice uma ponte entre linguagem humana e bases de dados corporativas, especialmente bases de ERP, CRM, PDV, OMS, WMS ou qualquer outra base relacional operacional que costume ter muitas tabelas, muitas colunas e nomes pouco amigaveis. Em vez de depender de o operador conhecer toda a estrutura tecnica da base, o sistema tenta recuperar apenas o recorte de schema mais relevante para a pergunta e devolver uma SQL pronta para revisao.

## 2. Que problema ela resolve

O problema real que NL2SQL resolve nao e apenas "gerar SQL". O problema e tornar consultavel um patrimonio de dados grande, tecnicamente complexo e geralmente hostil para times de negocio, operacao e consultoria. Em bases de ERP, e comum encontrar centenas ou milhares de tabelas, schemas gigantes, colunas com siglas internas, nomes historicos ruins, relacoes pouco obvias e documentacao dispersa ou inexistente.

Sem uma camada como essa, a empresa fica presa a um de dois extremos ruins.

1. Depender sempre de especialistas tecnicos que conhecem a base por memoria.
2. Expor acesso SQL amplo demais para pessoas que nao deveriam escrever consulta arbitraria em ambiente corporativo.

O slice NL2SQL resolve esse dilema ao transformar o schema em acervo semantico, recuperar o recorte mais promissor e produzir uma SQL proposta que continua sujeita a revisao humana. O valor pratico e reduzir atrito para descobrir dados sem abrir mao de governanca.

## 3. Visao executiva

Executivamente, NL2SQL importa porque destrava uso de dados sem exigir que toda pergunta operacional vire ticket tecnico. Ele reduz dependencia de especialistas raros, acelera diagnostico de negocio e melhora a capacidade de times funcionais explorarem suas proprias bases de dados com menos friccao.

Tambem reduz risco. O desenho do codigo nao executa a pergunta direto contra a base. Ele produz uma proposta SQL com dialeto explicito, diagnosticos estruturados, correlation_id e guardrail de somente leitura. Isso muda o valor executivo da feature: nao e um atalho perigoso para qualquer pessoa rodar comando em producao. E um mecanismo governado de traducao assistida para investigacao e apoio a decisao.

## 4. Visao comercial

Comercialmente, NL2SQL vende muito bem porque responde a uma dor simples de explicar e cara de ignorar: "meus dados existem, mas cada pergunta nova depende de um tecnico que conheca por dentro uma base enorme e confusa". O slice atual permite responder essa dor sem prometer magia. Ele nao promete consulta perfeita sem revisao. Ele promete acelerar muito a etapa mais cara e lenta: encontrar o caminho tecnico inicial em bases operacionais complexas.

Em conversa comercial, o valor pode ser apresentado assim.

1. A plataforma entende a pergunta do usuario.
2. Recupera automaticamente o trecho de schema mais relevante, sem despejar a base inteira no prompt.
3. Gera SQL revisavel no dialeto do cliente.
4. Bloqueia SQL destrutiva antes de qualquer uso operacional.

Isso e especialmente forte em contas com ERP grande, legado antigo, multiplos modulos e nomenclatura ruim, onde a dor nao e falta de dados, e excesso de estrutura dificil de navegar.

## 5. Visao estrategica

Estrategicamente, NL2SQL fortalece a plataforma porque conecta tres ativos que normalmente ficam separados.

1. Metadados estruturados da base.
2. Recuperacao semantica orientada por contexto.
3. Governanca de SQL somente leitura.

Esse arranjo evita dois erros estrategicos.

1. Tratar linguagem natural como acesso direto e irrestrito ao banco.
2. Tratar schema metadata apenas como catalogo morto sem uso operacional.

No projeto, NL2SQL transforma schema metadata em motor de valor. Isso prepara a plataforma para analytics assistido, copilotos de operacao, supervisores especialistas em dados e interfaces AG-UI que precisem propor consultas ou diagnosticos sobre bases empresariais.

## 6. Conceitos necessarios para entender

### 6.1. NL2SQL

NL2SQL e o processo de converter linguagem natural em SQL. No projeto, isso nao significa execucao automatica nem prompt solto para o LLM. Significa gerar uma proposta revisavel em cima de contexto de schema recuperado.

### 6.2. Schema metadata

Schema metadata e o acervo estruturado da base: databases, schemas, tabelas, colunas, tipos, chaves primarias, chaves estrangeiras, descricoes e, quando permitido explicitamente, amostras de linhas. Ele existe para que o sistema trabalhe com representacoes do schema e nao com intuicao cega.

### 6.3. Schema RAG

Schema RAG e a aplicacao de recuperacao semantica sobre metadados de schema. Em vez de jogar todas as tabelas e colunas no prompt, o sistema busca apenas os documentos de schema mais proximos da pergunta. Esse conceito e a chave para bases gigantes.

### 6.4. Guardrail de somente leitura

Guardrail e a trava que separa proposta revisavel de risco operacional. Aqui ele valida se a SQL e realmente somente leitura, bloqueando mutacoes, multiplas sentencas e tipos de statement nao permitidos.

### 6.5. Top-k semantico

Top-k e a quantidade maxima de documentos de schema recuperados para compor o contexto. No slice atual, esse numero e controlado e explicitamente limitado. Isso impede que o LLM receba contexto indiscriminado e ajuda o sistema a escalar para estruturas muito grandes.

## 7. Como a feature funciona por dentro

O fluxo comeca no endpoint dedicado de NL2SQL. O operador envia prompt, user_email, dialect e uma fonte explicita de YAML. O backend resolve correlation_id, resolve o YAML, injeta user_session e garante que schema_metadata.vectorstore_id e schema_metadata.sql_dialect existam.

Depois disso, o servico dedicado instancia a factory schema_rag_sql. Essa factory usa o vector store configurado para buscar documentos de schema semanticamente proximos da pergunta, respeitando top_k. Os documentos recuperados sao filtrados para document_type=schema_metadata, formatados em um contexto textual unico e truncados se ultrapassarem o limite operacional de caracteres.

So entao o LLM recebe o prompt final. A resposta volta como SQL candidata. O servico remove code fences quando necessario, executa o guardrail central de somente leitura e devolve uma resposta estruturada com sucesso, correlation_id, SQL proposta, warnings, diagnostics e execution_context.

Se algo falhar, o retorno nao vira excecao opaca apenas. O slice adiciona diagnosticos estaveis como selecao do vector store, dialeto aplicado, necessidade de revisao, truncamento de contexto, falha de geracao ou bloqueio do guardrail.

## 8. Divisao em etapas ou submodulos

Detalhamento aprofundado por etapa:

1. [Endpoint dedicado](README-CONCEITUAL-NL2SQL-COMPLETO-ENDPOINT-DEDICADO.md)
2. [Servico dedicado](README-CONCEITUAL-NL2SQL-COMPLETO-SERVICO-DEDICADO.md)
3. [Engine schema rag sql](README-CONCEITUAL-NL2SQL-COMPLETO-ENGINE-SCHEMA-RAG-SQL.md)
4. [Catalogo de schema metadata](README-CONCEITUAL-SCHEMA-METADATA-PRE-REQUISITO-NL2SQL.md)
5. [Guardrail central](README-CONCEITUAL-NL2SQL-COMPLETO-GUARDRAIL-CENTRAL.md)

### 8.1. [Endpoint dedicado](README-CONCEITUAL-NL2SQL-COMPLETO-ENDPOINT-DEDICADO.md)

E a borda HTTP que recebe o pedido. Ela existe para manter NL2SQL como slice proprio, separado de fluxos legados de agente e separado de dyn_sql.

### 8.2. [Servico dedicado](README-CONCEITUAL-NL2SQL-COMPLETO-SERVICO-DEDICADO.md)

E a camada que organiza validacao de entrada, injecao de contexto, chamada da engine schema_rag_sql e montagem da resposta estruturada. Ela existe para transformar uma engine interna em contrato administrativo estavel.

### 8.3. [Engine schema_rag_sql](README-CONCEITUAL-NL2SQL-COMPLETO-ENGINE-SCHEMA-RAG-SQL.md)

E a camada que faz o trabalho realmente importante para bases gigantes: busca semantica no acervo de schema, formatacao do contexto e chamada do LLM. Ela existe para nao depender do schema inteiro nem de nomes tecnicos perfeitos.

### 8.4. [Catalogo de schema metadata](README-CONCEITUAL-SCHEMA-METADATA-PRE-REQUISITO-NL2SQL.md)

E a camada que prepara e persiste o conhecimento estrutural da base. Ela existe para que NL2SQL nao fique refem de introspecao ad hoc a cada pergunta.

### 8.5. [Guardrail central](README-CONCEITUAL-NL2SQL-COMPLETO-GUARDRAIL-CENTRAL.md)

E a camada que decide se a SQL proposta e aceitavel como somente leitura. Ela existe para impedir que o slice vire um canal de mutacao por linguagem natural.

## 9. Por que serve muito bem para bases de dados gigantes

O motivo principal e arquitetural: o slice nao tenta colocar a base inteira dentro do prompt. O codigo da SqlSchemaRagToolFactory busca apenas top_k documentos semanticamente relevantes no vector store de schema metadata. Depois, ainda aplica truncamento de contexto por limite de caracteres antes de chamar o LLM.

Isso muda completamente a escalabilidade do problema. Em uma base pequena, jogar tudo no prompt talvez ate funcione. Em uma base de ERP com milhares de colunas e tabelas, isso rapidamente vira ruido, custo alto e baixa precisao. O slice atual evita esse erro porque recupera um recorte e nao a totalidade.

Em termos simples, ele trabalha assim.

1. Pergunta do usuario entra.
2. Busca semantica encontra as partes do schema mais proximas daquela pergunta.
3. So esse recorte vira contexto do LLM.
4. O resto da base continua fora do prompt.

Esse desenho faz muito sentido em bases grandes porque a pergunta raramente precisa de tudo ao mesmo tempo. Ela precisa do subconjunto certo.

## 10. Por que ajuda quando tabelas e colunas tem nomes nao significativos

Em bases de ERP, nomes como tbl_mvto_01, cad_cli, codfil, numlan, tger02 ou siglas internas antigas sao comuns. O problema desses nomes nao e apenas estético. Eles tornam a navegacao humana e o prompt naive muito piores.

O slice atual ajuda porque nao trabalha apenas com nome cru de tabela. O acervo de schema metadata exportado por tabela inclui estrutura de database, schema, tabela, colunas, PK, FKs e descricoes quando elas existem. O ETL novo de schema metadata tambem captura relacoes, tipos e comentarios de coluna quando o SGBD fornece isso.

Na pratica, isso quer dizer que a recuperacao semantica pode se apoiar em mais do que o identificador tecnico isolado. Ela usa o documento de tabela como unidade de contexto. Isso melhora a chance de recuperar o lugar certo mesmo quando a nomenclatura nao e amigavel, porque o contexto inclui relacoes e descricoes, nao so o nome cru.

Importante: isso nao significa que o sistema "entende qualquer nome ruim magicamente". O ganho vem de transformar a base em acervo semantico mais rico do que um dump de nomes isolados.

## 11. Por que e generico para bases de dados de ERP

O slice e generico porque a logica de NL2SQL nao esta hardcoded para um dominio especifico. O endpoint recebe prompt, dialect, top_k e um YAML com schema_metadata.vectorstore_id. O servico injeta user_session, resolve o vector store e chama a mesma engine schema_rag_sql independentemente de o dominio ser vendas, financeiro, estoque, fiscal, RH, pedidos, faturamento ou qualquer outro modulo.

O ponto de generalidade esta no contrato.

1. O schema metadata e identificado por vectorstore_id, nao por PDV fixo.
2. O dialeto SQL e um parametro explicito.
3. A busca e feita sobre documentos de schema metadata, nao sobre tabelas de um dominio fechado no codigo.
4. A resposta final sempre e uma SQL proposta revisavel com diagnosticos.

Por isso, o slice serve muito bem para bases de dados de ERP em geral, desde que o schema tenha sido materializado como schema metadata e que o dialeto esteja dentro do suporte confirmado.

### 11.1. O caminho mais simples para uma nova base ERP

Se a pergunta pratica for "o que eu preciso fazer para um agente responder em linguagem natural sobre uma nova base ERP?", o codigo lido reduz isso a uma sequencia bem objetiva.

1. Escolher um identificador logico para a base, como database_code.
2. Extrair o schema para o catalogo dbschemas.
3. Indexar esse schema em um vector store proprio de schema metadata.
4. Apontar o runtime NL2SQL ou o agente para esse mesmo vectorstore_id.
5. Fazer a pergunta em linguagem natural.

O ponto mais importante e que o runtime nao exige modelagem manual tabela por tabela dentro do prompt. O trabalho pesado fica concentrado em uma preparacao tecnica que a plataforma ja sabe fazer.

### 11.2. Os tres artefatos que tornam isso simples

O caso canonico do repositório mostra que a ativacao controlada de NL2SQL para uma base nova pode ser pensada em tres artefatos, nao em dezenas.

1. Um YAML de ETL de schema metadata.
2. Um YAML de ingestao vetorial de schema metadata.
3. Um YAML de runtime para NL2SQL ou agente.

Na pratica, isso deixa o onboarding muito mais simples do que escrever prompts enormes ou cadastrar manualmente cada consulta. Voce troca alguns identificadores centrais e reaproveita o mesmo desenho.

### 11.3. O que realmente muda de um ERP para outro

No slice lido, a arquitetura nao muda por dominio. O que muda, em geral, e apenas o conjunto abaixo.

1. DSN da base de origem.
2. Tipo do banco e dialeto SQL.
3. Nome logico da base e schema alvo.
4. database_code.
5. vectorstore_id.
6. Credenciais do provedor de embeddings e do runtime.

Isso e o que torna a feature poderosa para ERP: a plataforma nao esta acoplada a PDV, fiscal, estoque ou vendas. Ela reaproveita o mesmo pipeline para qualquer base relacional empresarial dentro do suporte confirmado.

### 11.4. Como pensar o schema metadata sem complicacao

Schema metadata, aqui, nao deve ser visto como uma tarefa burocratica extra. Ele e o catalogo tecnico que faz o NL2SQL funcionar bem em bases grandes.

Em linguagem simples, o papel dele e este.

1. Descobrir as tabelas reais.
2. Descobrir as colunas reais.
3. Descobrir relacoes como PK e FK.
4. Transformar isso em documentos pesquisaveis.

Sem isso, o agente teria de chutar o caminho tecnico da consulta. Com isso, ele recupera o trecho certo da base antes de propor a SQL.

## 12. O que o codigo confirma e o que ele ainda nao confirma

O codigo confirma claramente estas coisas.

1. O endpoint dedicado aceita postgresql, mysql e mssql como dialetos validos para o fluxo NL2SQL.
2. O guardrail central valida somente leitura nesses tres dialetos.
3. A preparacao de schema metadata lida nesta tarefa tem readers para PostgreSQL, SQL Server e Oracle.
4. O fluxo administrativo usa um YAML PDV como exemplo real, mas o contrato do servico e generico.

Isso cria uma assimetria importante que precisa ser documentada honestamente: o pipeline de preparo de metadados lido nesta rodada confirma PostgreSQL, SQL Server e Oracle, enquanto o endpoint dedicado de NL2SQL valida explicitamente apenas postgresql, mysql e mssql. Portanto, dizer "qualquer base" como fato literal e exagero se significar suporte operacional completo a qualquer dialeto hoje. O que o codigo sustenta e outra coisa: a arquitetura e generica para bases empresariais, mas o suporte confirmado do slice executavel e explicitamente delimitado.

## 13. O que acontece em caso de sucesso

No caminho feliz, o operador envia a pergunta, o servico resolve o vector store correto, recupera o contexto de schema, gera a SQL, passa no guardrail de somente leitura e devolve sucesso com correlation_id, SQL pronta para revisao, warnings e diagnosticos. O operador percebe isso como aceleracao da descoberta, nao como execucao final automatica.

## 14. O que acontece em caso de erro

Os principais cenarios de falha confirmados no codigo sao estes.

1. YAML invalido ou sem schema_metadata.
2. Falta de schema_metadata.vectorstore_id.
3. Dialeto fora do conjunto permitido.
4. Falha na geracao da SQL pelo motor schema_rag_sql.
5. Contexto de schema truncado por limite operacional.
6. SQL bloqueada por nao ser somente leitura.

O valor arquitetural aqui e importante: esses erros voltam como diagnosticos estruturados, nao apenas como falha opaca do LLM.

## 15. Impacto tecnico

Tecnicamente, NL2SQL encapsula um problema dificil de forma correta: recuperacao semantica de schema, geracao assistida, dialeto explicito e validacao de somente leitura. Isso reduz acoplamento entre UI, dominio de dados e modelo, melhora rastreabilidade e transforma schema metadata em ativo operacional.

## 16. Impacto executivo

Executivamente, a feature reduz o gargalo de dependencia de poucos especialistas da base e acelera investigacao operacional. Isso aumenta previsibilidade de resposta e reduz custo oculto de descoberta em bases grandes.

## 17. Impacto comercial

Comercialmente, a feature ajuda a vender analytics assistido sobre bases de ERP complexas, com argumento forte de governanca. Ela responde bem a clientes que ja tem muitos dados, mas pouca capacidade de exploracao rapida.

Promessa que pode ser feita: acelerar muito a descoberta e a formulacao inicial da consulta em bases estruturadas complexas.

Promessa que nao pode ser feita com base no codigo lido: acesso irrestrito a qualquer dialeto, qualquer engine e qualquer acao SQL sem revisao.

## 18. Impacto estrategico

Estrategicamente, NL2SQL fortalece a plataforma porque cria uma ponte entre ETL/schema metadata e uso operacional de dados. Isso prepara o produto para copilotos de analise, workflows que precisem propor consulta e interfaces AG-UI orientadas por diagnostico de dados.

## 19. Exemplos praticos guiados

### 19.1. Pergunta sobre faturamento por loja

Cenario: a operacao quer saber o faturamento por loja.

Fluxo: a pergunta entra, o sistema recupera apenas o recorte de schema relacionado a vendas e lojas, gera uma SQL agregada e devolve a proposta para revisao.

Valor: evita que alguem precise lembrar manualmente todas as relacoes entre tabelas de venda e dimensoes de loja.

### 19.2. Base enorme com nomenclatura ruim

Cenario: o ERP tem milhares de colunas e nomes internos pouco intuitivos.

Fluxo: em vez de despejar tudo no prompt, o sistema busca documentos de schema semanticamente proximos e usa esse subconjunto como contexto.

Valor: melhora foco e reduz ruido do LLM.

### 19.3. Pedido destrutivo

Cenario: o operador pede algo como apagar registros.

Fluxo: o LLM pode ate devolver uma SQL destrutiva, mas o guardrail central a bloqueia e o slice retorna falha controlada.

Valor: a interface continua assistiva sem abrir canal mutavel por linguagem natural.

### 19.4. Nova base de ERP em onboarding

Cenario: a empresa quer perguntar em linguagem natural sobre uma nova base de ERP qualquer, como financeiro, estoque ou faturamento.

Fluxo:

1. O time cadastra o schema dessa base no pipeline de schema metadata.
2. O exportador gera documentos por tabela.
3. A ingestao indexa esses documentos no vector store escolhido.
4. O YAML runtime aponta para esse mesmo vectorstore_id.
5. O agente ou o endpoint dedicado passa a responder perguntas em NL sobre essa base.

Valor: o esforço de onboarding fica concentrado na preparacao do schema, nao em criar prompt artesanal por modulo de ERP.

## 20. Explicacao 101

Imagine uma base de ERP como uma cidade enorme e confusa. Sem NL2SQL, a pessoa precisa conhecer todas as ruas tecnicas para chegar ao dado certo. Com este slice, a pergunta do usuario primeiro vira uma busca pelos bairros mais relevantes dessa cidade, depois o sistema monta uma rota tecnica inicial em forma de SQL, e por fim um fiscal confere se essa rota nao faz nada perigoso. E por isso que ele funciona melhor em bases muito grandes do que abordagens ingenuas que tentam mostrar a cidade inteira de uma vez.

## 21. Limites e pegadinhas

1. NL2SQL aqui gera SQL proposta; nao deve ser confundido com execucao automatica.
2. O sucesso depende da qualidade do acervo de schema metadata.
3. O slice melhora muito a navegacao em bases grandes, mas nao elimina a necessidade de revisao humana.
4. O endpoint dedicado valida explicitamente apenas postgresql, mysql e mssql.
5. O pipeline de metadata lido nesta tarefa mostra readers para PostgreSQL, SQL Server e Oracle, o que indica uma assimetria de suporte que nao pode ser escondida.
6. Para uso agentic com a tool schema_rag_sql, o supervisor ou agente tambem precisa declarar schema_metadata.enabled, vectorstore_id e sql_dialect. O validator falha cedo se isso nao existir.

## 22. Checklist de entendimento

- Entendi o que e NL2SQL neste projeto.
- Entendi por que ele serve bem para bases de dados gigantes.
- Entendi por que ele ajuda em nomenclaturas ruins.
- Entendi em que sentido ele e generico para bases de ERP.
- Entendi a diferenca entre proposta SQL e execucao.
- Entendi os limites reais de dialeto e guardrail.

## 23. Evidencias no codigo

- src/api/routers/config_nl2sql_router.py
  - Motivo da leitura: confirmar boundary HTTP dedicado e resolucao de YAML.
  - Simbolo relevante: POST /config/nl2sql/generate.
  - Comportamento confirmado: o slice recebe linguagem natural e devolve resposta tipada de NL2SQL.

- src/api/services/nl2sql_service.py
  - Motivo da leitura: confirmar o fluxo principal, diagnosticos e guardrail.
  - Simbolo relevante: Nl2SqlService.generate_sql.
  - Comportamento confirmado: o servico injeta user_session, resolve dialect, chama schema_rag_sql e valida somente leitura.

- src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py
  - Motivo da leitura: confirmar recuperacao semantica top_k, truncamento de contexto e prompt de geracao.
  - Simbolo relevante: SqlSchemaRagToolFactory.
  - Comportamento confirmado: o sistema busca apenas documentos de schema metadata relevantes, nao o schema inteiro.

- src/ingestion_layer/schema_metadata/schema_metadata_exporter.py
  - Motivo da leitura: confirmar que o acervo de schema e exportado por tabela com estrutura rica.
  - Simbolo relevante: SchemaMetadataJsonExporter.
  - Comportamento confirmado: o documento por tabela inclui database, schema, tabela, colunas, PK e FKs.

- src/etl_layer/providers/data/table_schema_metadata_processor.py
  - Motivo da leitura: confirmar preparo generico do catalogo de schema.
  - Simbolo relevante: TableSchemaMetadataETLProcessor.
  - Comportamento confirmado: o pipeline processa configuracao de origem e destino sem hardcode de dominio.

- src/integrations/sql_read_only_guardrail.py
  - Motivo da leitura: confirmar protecao de somente leitura.
  - Simbolo relevante: SqlReadOnlyGuardrail.
  - Comportamento confirmado: multiplas sentencas e mutacoes sao bloqueadas.
