# Manual detalhado da etapa: Guardrail central do NL2SQL

## 1. O que esta etapa faz

Esta etapa decide se a SQL proposta pode ser tratada como somente leitura. Ela existe para impedir que NL2SQL vire um canal de mutacao por linguagem natural e para separar claramente proposta revisavel de risco operacional.

Em linguagem simples: e o fiscal que impede a rota SQL de virar acao perigosa.

## 2. Onde ela entra no fluxo

No codigo lido, o guardrail roda dentro do Nl2SqlService, depois que a engine schema_rag_sql devolve uma SQL candidata e antes que a resposta final seja marcada como sucesso.

## 3. O que entra e o que sai

Entradas confirmadas:

- sql_candidate normalizada
- dialect explicito

Saidas confirmadas:

- resultado allowed verdadeiro ou falso
- reason_code estavel
- statement_type e statement_count
- bloqueio da resposta quando a SQL nao e somente leitura

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. o servico chama o guardrail com a SQL candidata e o dialeto normalizado;
2. o guardrail mapeia o dialeto do produto para o dialeto interno do sqlglot;
3. bloqueia SQL vazia;
4. tenta parsear a sentenca;
5. bloqueia quando ha mais de uma sentenca;
6. percorre a arvore sintatica para encontrar nos mutaveis como insert, update, delete, drop, create e merge;
7. mesmo sem nos mutaveis, bloqueia statement types fora do conjunto de somente leitura;
8. so libera select, union, intersect, except, show e describe;
9. devolve resultado estruturado com allowed, reason_code, statement_type e statement_count;
10. o servico traduz esse resultado em diagnosticos de bloqueio ou de autorizacao.

O detalhe mais importante e que a decisao nao depende de regex fraca. Ela depende de parse sintatico por dialeto.

## 5. Decisoes tecnicas importantes

### 5.1. Guardrail usa AST SQL por dialeto

Ao parsear a SQL com sqlglot, o produto deixa de depender de heuristicas textuais superficiais para dizer se uma consulta e segura.

### 5.2. Uma unica sentenca por validacao

Mesmo que todas fossem de leitura, multiplas sentencas sao bloqueadas. Isso reduz ambiguidade e superficie de abuso.

### 5.3. Sucesso do slice depende do guardrail

NL2SQL so retorna success verdadeiro quando a SQL candidata passa explicitamente pela validacao de somente leitura.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- dialeto nao suportado impede validacao;
- SQL vazia ou malformada e bloqueada;
- multiplas sentencas sao rejeitadas;
- qualquer operacao mutavel e bloqueada;
- mesmo uma sentenca unica pode ser bloqueada se o tipo nao estiver na lista de leitura.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- diagnostics NL2SQL_SQL_GUARDRAIL_BLOCKED e NL2SQL_SQL_GUARDRAIL_ALLOWED;
- reason_code, statement_type e statement_count no execution_context;
- response success falso com sql nula quando o guardrail reprova;
- logs do servico indicando bloqueio pelo guardrail.

Em linguagem simples: quando a pergunta parece inocente, mas a resposta nao vira sucesso, a diferenca muitas vezes esta no que o guardrail comprovou sobre a SQL gerada.

## 8. Exemplo pratico guiado

Cenario: o LLM devolve uma SQL com delete ou duas sentencas no mesmo texto.

1. o servico extrai a SQL candidata;
2. passa esse texto ao guardrail;
3. o guardrail parseia a arvore sintatica;
4. detecta operacao mutavel ou contagem invalida de sentencas;
5. devolve allowed falso com reason_code especifico;
6. o servico bloqueia a resposta e devolve diagnostico estruturado em vez de liberar a SQL.

O valor desta etapa e manter o slice util para descoberta sem abrir um canal destrutivo.

## 9. Evidencias no codigo

- src/integrations/sql_read_only_guardrail.py
  - Simbolo relevante: SqlReadOnlyGuardrail.validate
  - Comportamento confirmado: validacao sintatica por dialeto com bloqueio de mutacao, multiplas sentencas e tipos nao permitidos.
- src/integrations/sql_read_only_guardrail.py
  - Simbolo relevante: mapa de dialetos e tipos permitidos de leitura
  - Comportamento confirmado: suporte confirmado do guardrail para postgresql, mysql e mssql.
- src/api/services/nl2sql_service.py
  - Simbolo relevante: trecho de aplicacao do guardrail no caminho dedicado
  - Comportamento confirmado: o slice so marca sucesso quando a SQL proposta e validada como somente leitura.
