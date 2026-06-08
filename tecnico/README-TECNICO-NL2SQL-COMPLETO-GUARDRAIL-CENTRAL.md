# Manual técnico por etapa: guardrail central do NL2SQL

## 1. O que esta etapa cobre

Esta etapa cobre o SqlReadOnlyGuardrail, que decide se a SQL candidata é comprovadamente somente leitura antes que o slice marque sucesso.

## 2. Como o guardrail funciona

O guardrail usa sqlglot para parsear a SQL por dialeto e então aplica uma política sintática explícita:

- bloqueia SQL vazia
- bloqueia parse inválido
- bloqueia múltiplas sentenças
- bloqueia nós mutáveis como insert, update, delete, drop, create, alter, merge, truncate e execute
- libera apenas tipos raiz de leitura, como select, union, intersect, except, show e describe

Isso é importante porque a decisão não depende de regex textual fraca. Ela depende de árvore sintática por dialeto.

## 3. Dialetos suportados

O mapeamento confirmado do guardrail cobre:

- postgresql
- mysql
- mssql

Esse conjunto precisa permanecer alinhado com o contrato do serviço dedicado para que a validação seja consistente de ponta a ponta.

## 4. Como o slice usa o resultado

O serviço consome o resultado estruturado do guardrail e devolve:

- diagnóstico de bloqueio quando allowed é falso
- diagnóstico de liberação quando allowed é verdadeiro
- statement_type, statement_count e reason_code dentro do execution_context

Com isso, a UI e o suporte entendem por que a proposta foi aceita ou recusada sem reparsear a SQL manualmente.

## 5. Diagnóstico recomendado

Quando a SQL candidata parece boa, mas o fluxo retorna falha:

1. revisar reason_code do guardrail
2. verificar statement_type e statement_count
3. confirmar se o dialeto informado corresponde ao dialeto real esperado

## 6. Evidências no código

- src/integrations/sql_read_only_guardrail.py
  - Motivo: política central de somente leitura.
  - Comportamento confirmado: o guardrail parseia por dialeto, bloqueia mutação e aceita apenas sentenças de leitura.
- src/api/services/nl2sql_service.py
  - Motivo: uso do guardrail no slice executável.
  - Comportamento confirmado: o serviço só marca sucesso quando a SQL proposta é validada como somente leitura.
