# Manual técnico por etapa: validação semântica do NL2YAML

## 1. O que esta etapa cobre

Esta etapa cobre o validate oficial da AST. Ela decide se o payload montado pelo draft é semanticamente aceitável para o alvo escolhido e se o YAML governado está consistente com o contrato atual do assembly.

## 2. O que o validate realmente faz

O caminho confirmado no código é este:

1. parseia o ast_payload para AgenticDocumentAST
2. resolve o catálogo efetivo de tools para o alvo
3. chama a validacao semantica correta para workflow, deepagent ou trilha legada descontinuada
4. executa detecção de drift do YAML governado
5. agrega diagnósticos da política de base mista quando aplicável
6. devolve validation_report consolidado e compiled_fragment

Quando a dúvida específica for o hash governado, a regra de geração correta, os diagnósticos de drift ou a forma oficial de sincronização via helper e CLI, a referência principal deixou de ficar implícita neste manual e passou a estar centralizada em:

- [docs/README-CONCEITUAL-HASH-GOVERNADO-AGENTIC.md](../conceitual/README-CONCEITUAL-HASH-GOVERNADO-AGENTIC.md)
- [docs/README-TECNICO-HASH-GOVERNADO-AGENTIC.md](README-TECNICO-HASH-GOVERNADO-AGENTIC.md)

## 3. O papel do strict

O request público aceita strict. Quando strict é verdadeiro, success depende diretamente de merged_report.is_valid. Isso é importante porque objective to yaml usa validate em modo estrito antes de seguir para o confirm em dry run.

## 4. O que esta etapa protege

Ela protege o fluxo contra quatro classes de erro:

- AST sintaticamente bem formada, mas semanticamente inválida
- uso incorreto de catálogo ou tools indisponíveis
- drift entre YAML governado atual e o fingerprint esperado
- mistura perigosa entre base legada e fragmento governado atual

## 5. Sinais de diagnóstico mais importantes

Os sinais operacionais mais relevantes desta etapa são:

- validation_report.is_valid
- error_count e warning_count
- diagnostics agregados do validador semântico
- compiled_fragment já calculado pelo validate
- diagnósticos de drift e de mixed base policy

## 6. Diagnóstico recomendado

Quando o draft parece bom, mas o fluxo trava em validate, a investigação correta é:

1. revisar o target efetivo usado no validate
2. conferir o catálogo efetivo de tools para esse alvo
3. comparar ast_payload com compiled_fragment
4. verificar se o bloqueio veio de drift governado ou da semântica do documento

Se o bloqueio vier de drift governado, use os manuais dedicados acima para seguir a jornada correta de refresh pelo confirm oficial, em vez de tentar reparar metadata manualmente.

## 7. Evidências no código

- src/config/agentic_assembly/assembly_service.py
  - Motivo: validate oficial do slice.
  - Comportamento confirmado: validate agrega parse, catálogo, validação semântica, drift e política de base mista.
- src/config/agentic_assembly/models.py
  - Motivo: contrato público de validação.
  - Comportamento confirmado: a API devolve diagnostics, validation_report, compiled_fragment e correlation_id.
