# Manual detalhado da etapa: Servico dedicado do NL2SQL

## 1. O que esta etapa faz

Esta etapa organiza a execucao backend do slice. Ela existe para validar entradas de negocio, preparar o YAML de runtime, escolher o dialeto, chamar a engine schema_rag_sql e transformar tudo isso em uma resposta administrativa estavel com diagnosticos seguros.

Em linguagem simples: e a camada que converte um pedido HTTP em uma execucao governada de NL2SQL.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa mora em Nl2SqlService. Ela e acionada pelo router e chama a factory schema_rag_sql antes de acionar o guardrail.

## 3. O que entra e o que sai

Entradas confirmadas:

- prompt
- user_email
- yaml_config
- dialect
- top_k
- correlation_id
- source_hint opcional

Saidas confirmadas:

- Nl2SqlResponse com success verdadeiro ou falso
- diagnostics estruturados e estaveis
- warnings com exigencia de revisao humana
- execution_context com vectorstore_id, top_k, dialect, schema_context e resultado do guardrail

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. valida que yaml_config seja um mapeamento;
2. valida prompt e user_email obrigatorios;
3. gera correlation_id se ele nao vier pronto;
4. normaliza o dialeto usando o mesmo contrato do guardrail;
5. prepara uma copia do YAML e injeta user_session e sql_dialect no bloco schema_metadata;
6. exige que schema_metadata.vectorstore_id exista;
7. instancia a SqlSchemaRagToolFactory com o YAML preparado e o top_k desejado;
8. executa a geracao da SQL para a pergunta;
9. remove code fences quando a resposta do motor vier em markdown;
10. monta diagnosticos de vector store, dialeto, revisao obrigatoria, source hint e truncamento de contexto;
11. se a engine nao produzir SQL utilizavel, devolve falha controlada;
12. se houver SQL candidata, submete ao guardrail central de somente leitura;
13. devolve success true apenas quando a SQL passa no guardrail e fica pronta para revisao humana.

O detalhe mais importante e que o servico nunca promete execucao. Ele sempre devolve uma proposta de SQL com review_required verdadeiro.

## 5. Decisoes tecnicas importantes

### 5.1. O YAML de trabalho e enriquecido, nao usado cru

O servico injeta user_session e sql_dialect no runtime. Isso garante que a engine trabalhe com contexto consistente para auditoria e geração.

### 5.2. Diagnosticos fazem parte do contrato publico

O retorno nao expõe apenas success ou error. Ele registra codigos estaveis como selecao do vector store, dialeto aplicado, truncamento, falha de geração e resultado do guardrail.

### 5.3. Revisao humana e obrigatoria por contrato

Mesmo no caminho feliz, o servico devolve aviso fixo de revisao e review_required verdadeiro. Isso reforca que NL2SQL aqui e geracao assistida, nao automacao cega.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- yaml_config invalido ou nao mapeavel bloqueia o servico;
- prompt vazio ou user_email vazio interrompem a execucao;
- dialect fora de postgresql, mysql ou mssql gera erro explicito;
- schema_metadata ausente ou sem vectorstore_id impede o runtime;
- user_session malformado no YAML tambem bloqueia o fluxo;
- a engine pode nao devolver SQL utilizavel mesmo com contexto resolvido.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- diagnostics como NL2SQL_VECTORSTORE_SELECTED, NL2SQL_DIALECT_SELECTED, NL2SQL_GENERATION_FAILED e NL2SQL_SCHEMA_CONTEXT_TRUNCATED;
- warnings com lembrete de revisao obrigatoria;
- execution_context mostrando vectorstore_id, top_k e resumo do schema_context;
- log de inicio e fim do servico com vectorstore_id e dialeto.

Em linguagem simples: se o router funcionou, mas a resposta ainda nao faz sentido, o problema geralmente esta no preparo que este servico faz do runtime.

## 8. Exemplo pratico guiado

Cenario: o operador pergunta pelo faturamento por loja e fornece o YAML de runtime correto.

1. o servico valida prompt, email e dialeto;
2. injeta user_session e sql_dialect no YAML;
3. resolve vectorstore_id;
4. chama a engine schema_rag_sql;
5. recebe a SQL candidata;
6. executa o guardrail;
7. devolve resposta tipada pronta para revisao.

O valor desta etapa e transformar a engine interna em um contrato de produto claro e auditavel.

## 9. Evidencias no codigo

- src/api/services/nl2sql_service.py
  - Simbolo relevante: generate_sql
  - Comportamento confirmado: orquestracao do fluxo dedicado, diagnosticos estruturados e resposta revisavel.
- src/api/services/nl2sql_service.py
  - Simbolo relevante: preparacao do YAML de runtime
  - Comportamento confirmado: exige schema_metadata, vectorstore_id e injeta user_session e sql_dialect.
- src/api/services/nl2sql_service.py
  - Simbolo relevante: normalizacao de dialeto e limpeza de code fence
  - Comportamento confirmado: o servico estabiliza a entrada do guardrail e a saida do motor.
