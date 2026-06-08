# Manual detalhado da etapa: Engine schema_rag_sql do NL2SQL

## 1. O que esta etapa faz

Esta etapa recupera o recorte de schema mais promissor e usa esse contexto para pedir ao LLM uma SQL candidata. Ela existe para evitar a abordagem ingenua de despejar o schema inteiro no prompt ou de tentar adivinhar tabelas e colunas no escuro.

Em linguagem simples: e o motor que encontra o bairro certo do banco antes de propor a rota SQL.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa vive na SqlSchemaRagToolFactory. O servico dedicado instancia a factory, e a factory faz busca semantica no vector store, formata o contexto e chama o LLM.

## 3. O que entra e o que sai

Entradas confirmadas:

- user_question
- vectorstore_id resolvido no YAML
- top_k
- sql_dialect
- configuracao LLM e vector store do runtime

Saidas confirmadas:

- schema_context estruturado com quantidade de documentos e truncamento
- raw_response do motor
- sql_text candidata ou nula
- erro controlado quando o vector store ou a geração falham

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. a factory resolve e cria o vector store configurado;
2. executa similarity_search ou search_similar com retry central e top_k configurado;
3. se o vector store nao suportar metodo conhecido, retorna erro estruturado;
4. filtra apenas documentos cujo metadata.document_type seja schema_metadata;
5. se nenhum documento valido restar, devolve contexto de erro e nao segue cegamente;
6. formata os documentos validos em um contexto textual unico, um bloco por tabela;
7. mede chars_before e aplica truncamento se o contexto exceder o limite operacional;
8. monta o prompt final com dialeto, schema_context e pergunta do usuario;
9. chama o LLM com retry central;
10. limpa a resposta para extrair SQL de blocos markdown quando necessario;
11. devolve um resultado estruturado contendo sql_text, raw_response e schema_context.

O detalhe mais importante e que a engine nao considera qualquer documento recuperado como contexto valido. Ela exige explicitamente document_type igual a schema_metadata.

## 5. Decisoes tecnicas importantes

### 5.1. Busca semantica usa top_k controlado

O motor nao tenta carregar o schema inteiro. Ele limita a recuperacao aos documentos mais proximos da pergunta, o que reduz ruido e custo operacional.

### 5.2. Contexto de schema e filtrado por tipo documental

Mesmo que o vector store devolva outros documentos, a engine ignora o que nao for schema_metadata. Isso protege a qualidade do contexto.

### 5.3. Truncamento e assumido como parte do runtime

Quando o contexto fica grande demais, ele e truncado explicitamente e o slice registra esse fato depois como diagnostico. O sistema prefere contexto parcial controlado a prompt gigante e opaco.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- o vector store pode nao estar acessivel;
- o mecanismo de busca pode nao ser suportado pela implementacao concreta;
- a recuperacao pode trazer zero documentos validos de schema_metadata;
- o contexto pode ser truncado e reduzir cobertura semantica;
- o LLM pode devolver texto de erro ou SQL inutilizavel.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- schema_context.document_count e schema_document_count;
- flag truncated e contadores chars_before e chars_after;
- diagnostico NL2SQL_SCHEMA_CONTEXT_TRUNCATED na camada superior;
- raw_response parcial quando a geracao falha;
- logs da factory sobre top_k, tipo do vector store e quantidade de documentos validos.

Em linguagem simples: quando a SQL sai ruim, a primeira pergunta correta nao e sobre o prompt, e sobre que contexto de schema esta engine realmente entregou ao LLM.

## 8. Exemplo pratico guiado

Cenario: a base tem milhares de tabelas, mas a pergunta fala de faturamento por loja.

1. a factory busca semanticamente os documentos mais proximos da pergunta;
2. filtra apenas documentos schema_metadata;
3. monta um contexto com tabelas e relacoes do recorte relevante;
4. trunca se o tamanho explodir;
5. pede ao LLM uma SQL no dialeto informado;
6. devolve a candidata para a camada de servico.

O valor desta etapa e tornar bases grandes navegaveis sem exigir que o modelo veja a cidade inteira de uma vez.

## 9. Evidencias no codigo

- src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py
  - Simbolo relevante: generate_sql_for_question
  - Comportamento confirmado: fluxo completo de recuperação de schema e geração da SQL candidata.
- src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py
  - Simbolo relevante: busca de schema metadata com top_k
  - Comportamento confirmado: uso de similarity_search ou search_similar com retry central.
- src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py
  - Simbolo relevante: filtro por documentos validos de schema metadata
  - Comportamento confirmado: apenas document_type igual a schema_metadata entra no contexto.
- src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py
  - Simbolo relevante: formatacao e truncamento do schema_context
  - Comportamento confirmado: o contexto enviado ao LLM e limitado operacionalmente antes da geracao.
