# Manual técnico por etapa: draft assistido do NL2YAML

## 1. O que esta etapa cobre

Esta etapa cobre a geração do draft AST a partir do briefing, já depois do preflight. Aqui o objetivo é transformar intenção em uma AST válida ou pelo menos reparável, com diagnostics e questions estruturadas.

## 2. O que entra no draft

O draft recebe:

- prompt
- user_email
- target pedido
- base_yaml
- constraints
- generation_mode
- correlation_id

O serviço também resolve o catálogo efetivo de tools e, quando necessário, um catálogo heurístico complementar para o fluxo NL.

## 3. Como o draft é montado

O caminho técnico observado no código é:

1. resolver base_yaml e target
2. chamar build draft from prompt quando há briefing
3. usar IntentParser para produzir IR estruturada
4. converter a IR em AgenticDocumentAST pelo archetype factory
5. submeter a AST a um repair loop controlado
6. validar o documento gerado contra catálogo, alvo e YAML base
7. devolver ast_draft, draft_fragment, merged_preview, diff_preview, diagnostics e questions

## 4. O papel do repair loop

O repair loop é importante porque o draft não assume que a primeira AST montada já está consistente. Ele tenta reparar o documento com base na própria validação oficial, em vez de empurrar um payload quebrado para o usuário final.

## 5. Como os modos afetam o draft

Na prática:

- llm_schema tenta draft com provider estruturado
- heuristic gera draft determinístico sem depender desse provider
- auto decide entre os dois apenas quando a política de fallback permitir isso explicitamente

Isso significa que o draft não é apenas “usar LLM ou não”. Ele é governado por política operacional e por validação posterior.

## 6. O que pode bloquear nesta etapa

O fluxo objective to yaml bloqueia em draft quando:

- ainda existem questions obrigatórias
- validation_report do draft veio inválido
- o parser ou a geração não conseguiram materializar uma AST utilizável

Nesse cenário, o usuário recebe ast_payload, questions e chosen_tools do próprio draft, mas não recebe final_yaml.

## 7. Diagnóstico recomendado

Para investigar problemas de draft:

1. conferir generation_mode real usado na chamada
2. revisar questions pendentes antes de tentar validar ou publicar
3. inspecionar ast_draft e merged_preview para ver se o alvo foi montado como esperado
4. usar diagnostics do draft antes de culpar o confirm

## 8. Evidências no código

- src/config/agentic_assembly/assembly_service.py
  - Motivo: pipeline do draft.
  - Comportamento confirmado: draft resolve catálogos, monta AST, executa repair loop e devolve preview governado.
- src/config/agentic_assembly/nl/intent_parser.py
  - Motivo: geração da IR que alimenta o draft.
  - Comportamento confirmado: o parser resolve capacidades, perguntas e target antes da AST.
- src/config/agentic_assembly/models.py
  - Motivo: contrato público do draft.
  - Comportamento confirmado: draft expõe ast_draft, draft_fragment, merged_preview, diff_preview, diagnostics e questions.
