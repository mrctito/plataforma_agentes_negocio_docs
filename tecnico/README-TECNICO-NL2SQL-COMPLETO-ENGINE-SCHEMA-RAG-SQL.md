# Manual técnico por etapa: engine schema rag sql do NL2SQL

## 1. O que esta etapa cobre

Esta etapa cobre a factory SqlSchemaRagToolFactory, que faz a recuperação semântica do schema, monta o contexto enviado ao LLM e extrai a SQL candidata de volta.

## 2. O que a engine realmente faz

O caminho técnico confirmado no código é:

1. resolve o vector store configurado por vectorstore_id
2. executa similarity_search ou search_similar com retry central
3. limita a recuperação por top_k
4. filtra apenas documentos com document_type igual a schema_metadata
5. formata o contexto por tabela
6. trunca o contexto quando o limite operacional de caracteres é excedido
7. monta o prompt final com dialeto, schema_context e pergunta
8. chama o LLM com retry central
9. extrai a SQL do texto retornado, inclusive quando ela vier dentro de code fence

## 3. Por que isso escala para bases grandes

Três mecanismos confirmados explicam a escalabilidade:

- busca top_k em vez de dump do schema inteiro
- filtro explícito para schema_metadata válido
- truncamento operacional do contexto antes da chamada ao LLM

Na prática, a engine trabalha com o recorte mais promissor do acervo de schema, não com a base inteira no prompt.

## 4. Erros e limites relevantes

Os principais problemas desta etapa são:

- vector store indisponível
- implementação concreta sem método de busca suportado
- nenhum documento válido de schema_metadata no acervo
- truncamento excessivo do contexto em bases muito grandes
- LLM devolvendo resposta inútil ou de erro

## 5. Diagnóstico recomendado

Para investigar qualidade ruim da SQL:

1. conferir document_count e schema_document_count
2. verificar se truncated ficou verdadeiro
3. revisar top_k usado na chamada
4. inspecionar raw_response antes de culpar o serviço ou o guardrail

## 6. Evidências no código

- src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py
  - Motivo: motor principal do NL2SQL.
  - Comportamento confirmado: a engine busca schema metadata por similaridade, monta contexto e chama o LLM.
- src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py
  - Motivo: proteção operacional do contexto.
  - Comportamento confirmado: top_k, filtro por document_type e truncamento são parte do caminho oficial.
