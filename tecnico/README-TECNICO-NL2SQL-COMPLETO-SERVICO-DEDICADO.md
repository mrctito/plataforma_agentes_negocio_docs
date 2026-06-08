# Manual técnico por etapa: serviço dedicado do NL2SQL

## 1. O que esta etapa cobre

Esta etapa cobre o Nl2SqlService, que organiza a execução backend do slice. Ele valida entrada, enriquece o YAML, chama a engine schema rag sql, aplica o guardrail e monta a resposta final com diagnostics e execution_context.

## 2. Validações obrigatórias do serviço

Antes de chamar a engine, o serviço exige:

- yaml_config como mapeamento válido
- prompt não vazio
- user_email não vazio
- dialect entre postgresql, mysql e mssql
- schema_metadata presente no YAML
- schema_metadata.vectorstore_id preenchido

## 3. Enriquecimento do runtime

O serviço não usa o YAML cru. Ele cria uma cópia de trabalho e injeta:

- schema_metadata.sql_dialect com o dialeto normalizado
- user_session.user_email
- user_session.correlation_id

Isso garante que a engine e a trilha de auditoria recebam um runtime coerente sem mutar o payload original de forma lateral.

## 4. Montagem da resposta estruturada

Depois da geração, o serviço monta diagnostics estáveis para:

- vector store selecionado
- dialeto aplicado
- revisão humana obrigatória
- source hint do YAML
- truncamento do schema_context
- falha de geração
- resultado do guardrail
- pronto para revisão

Essa resposta existe para suportar UI, suporte e auditoria sem depender de parsing frágil do texto livre do LLM.

## 5. Como o serviço trata falha

Há dois bloqueios técnicos principais além das validações iniciais:

- a engine não devolve SQL utilizável
- a SQL candidata é reprovada pelo guardrail de somente leitura

Nos dois casos, o serviço retorna success falso, review_required verdadeiro e execution_context seguro para troubleshooting.

## 6. Diagnóstico recomendado

Quando o endpoint responde, mas a proposta não chega pronta para revisão, a ordem correta é:

1. revisar vectorstore_id e dialect no execution_context
2. confirmar se a falha veio da engine ou do guardrail
3. usar os diagnostics estruturados antes de olhar apenas raw_response

## 7. Evidências no código

- src/api/services/nl2sql_service.py
  - Motivo: orquestração backend do slice.
  - Comportamento confirmado: o serviço valida o YAML, injeta contexto, chama a engine e devolve diagnostics estruturados.
- src/api/services/nl2sql_service.py
  - Motivo: integração com guardrail.
  - Comportamento confirmado: success só fica true quando a SQL passa no guardrail de somente leitura.
