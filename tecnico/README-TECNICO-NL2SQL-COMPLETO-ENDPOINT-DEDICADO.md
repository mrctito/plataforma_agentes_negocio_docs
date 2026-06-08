# Manual técnico por etapa: endpoint dedicado do NL2SQL

## 1. O que esta etapa cobre

Esta etapa cobre o boundary HTTP administrativo do NL2SQL. Ela recebe o pedido em linguagem natural, resolve o contexto YAML e publica um contrato estável de geração de SQL revisável.

## 2. Superfície pública confirmada

O slice executável expõe POST /config/nl2sql/generate. O boundary usa permissão de configuração, rate limit administrativo e resposta tipada própria do domínio NL2SQL.

## 3. O que o endpoint valida antes do serviço

O router faz estas tarefas antes de delegar ao serviço:

- resolve correlation_id a partir do payload, do request state ou da factory central
- resolve um rótulo de usuário para logging
- aceita yaml_config em memória quando ele já veio desserializado
- caso contrário, usa o resolvedor compartilhado para yaml_config_path, yaml_inline_content ou encrypted_data
- traduz erro de contexto para HTTP 400
- traduz falha interna inesperada para HTTP 500 controlado

## 4. Por que esse boundary existe

Esse endpoint existe para manter NL2SQL separado de dois caminhos que teriam semântica muito diferente:

- execução agentic genérica
- dynamic SQL livre

O boundary deixa explícito que o objetivo aqui é gerar uma proposta de SQL com diagnósticos, não executar comandos arbitrários.

## 5. Onde esta etapa costuma falhar

Os erros mais típicos são:

- yaml_config com tipo inválido
- ausência de qualquer fonte de YAML válida
- erro de contexto antes da geração
- falha interna do serviço que precisa virar HTTP 500 controlado

## 6. Diagnóstico recomendado

Para investigar a borda:

1. validar a fonte de YAML enviada
2. conferir se correlation_id foi devolvido pelo boundary
3. revisar se o erro veio do resolvedor de configuração ou do serviço dedicado
4. confirmar que a chamada usou o endpoint certo, não outro fluxo agentic

## 7. Evidências no código

- src/api/routers/config_nl2sql_router.py
  - Motivo: boundary HTTP do slice.
  - Comportamento confirmado: POST /config/nl2sql/generate resolve YAML e chama o serviço dedicado.
- src/api/schemas/nl2sql_models.py
  - Motivo: contrato de entrada e saída.
  - Comportamento confirmado: a rota usa Nl2SqlRequest e Nl2SqlResponse como superfície pública estável.
