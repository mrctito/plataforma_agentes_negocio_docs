# Manual técnico por etapa: prontidão operacional do NL2YAML

## 1. O que esta etapa cobre

Esta etapa cobre o preflight operacional do assembly. Ela decide se o ambiente está pronto para gerar um draft confiável antes de consumir LLM, catálogo ou validadores mais caros.

## 2. O que o preflight verifica

Os checks confirmados no código cobrem pelo menos estes pontos:

- autenticação administrativa no boundary
- permissão config.generate
- feature flag do assembly ativa
- user_email disponível para rastreamento
- provider LLM estruturado, quando o modo exige isso
- catálogo efetivo de tools disponível e não vazio

## 3. Como o modo de geração interfere

O comportamento muda conforme generation_mode:

- heuristic trata provider estruturado como não obrigatório
- llm_schema exige provider apto para with structured output
- auto só admite fallback heurístico quando houve opt in explícito por constraints ou política equivalente

Isso é central para troubleshooting porque um mesmo ambiente pode estar pronto para heuristic e bloqueado para llm_schema.

## 4. O que acontece quando o preflight bloqueia

Quando ready é falso, o fluxo objective to yaml encerra imediatamente com:

- success igual a false
- blocking_stage igual a preflight
- diagnostics consolidados de bloqueio
- preflight_checks e preflight_summary preenchidos

O sistema não tenta draft parcial nem geração “para ver se dá certo”.

## 5. Sinais operacionais importantes

Os sinais mais úteis desta etapa são:

- AST_PREFLIGHT_FEATURE_DISABLED
- AST_PREFLIGHT_USER_EMAIL_MISSING
- AST_PREFLIGHT_LLM_CONFIG_MISSING
- AST_PREFLIGHT_LLM_BLOCKED
- AST_PREFLIGHT_HEURISTIC_MODE_READY
- AST_PREFLIGHT_AUTO_FALLBACK_READY
- AST_PREFLIGHT_CATALOG_EMPTY

Em termos práticos, esses códigos dizem se o bloqueio veio de feature, identidade, provider, política de fallback ou catálogo.

## 6. Diagnóstico recomendado

Quando o operador disser que NL2YAML “não começa”, a ordem correta é:

1. verificar FEATURE_AGENTIC_AST_ENABLED
2. confirmar user_email no request
3. revisar generation_mode escolhido
4. validar se o YAML base tem llm configurado quando o modo exige provider estruturado
5. checar se o catálogo efetivo de tools foi resolvido e não está vazio

## 7. Evidências no código

- src/config/agentic_assembly/assembly_service.py
  - Motivo: preflight operacional.
  - Comportamento confirmado: preflight valida feature flag, user_email, provider estruturado e catálogo antes do draft.
- src/config/agentic_assembly/models.py
  - Motivo: contrato de prontidão.
  - Comportamento confirmado: a resposta pública devolve ready, summary, checks, diagnostics e correlation_id.
