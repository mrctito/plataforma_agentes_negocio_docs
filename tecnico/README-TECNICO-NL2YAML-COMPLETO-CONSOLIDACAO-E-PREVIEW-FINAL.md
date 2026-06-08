# Manual técnico por etapa: consolidação e preview final do NL2YAML

## 1. O que esta etapa cobre

Esta etapa cobre o ponto em que objective to yaml deixa de ser apenas preflight ou draft e passa a consolidar um YAML final governado em memória, com diff preview, chosen_tools e decision_trace completos.

## 2. Como essa consolidação acontece

No caminho de sucesso, objective to yaml encadeia:

1. preflight
2. draft
3. validate
4. confirm com apply igual a false

O confirm em dry run é a peça que produz o final_yaml sem gravar arquivo. Ele seleciona o fragmento governado do target, faz merge canônico no YAML base, estampa fingerprint e calcula diff preview.

## 3. O que entra na resposta consolidada

No sucesso, o contrato público devolve:

- final_yaml
- final_yaml_text
- ast_payload
- diff_preview
- chosen_tools
- decision_trace
- preflight_summary e preflight_checks
- validation_report
- correlation_id

Essa resposta é exatamente o que a UI administrativa precisa para revisão humana antes da publicação.

## 4. Como os bloqueios são expostos

Se o fluxo falha em qualquer estágio, objective to yaml não mascara isso. Ele devolve blocking_stage explícito com um de quatro valores:

- preflight
- draft
- validate
- confirm

Isso reduz muito o tempo de suporte porque o operador sabe imediatamente onde a cadeia parou.

## 5. chosen_tools e decision_trace

Esses dois campos têm papel operacional diferente:

- chosen_tools responde quais ferramentas acabaram referenciadas pelo draft ou pelo YAML final
- decision_trace responde por que o serviço tomou decisões críticas de target, questions, modo de geração e bloqueios

Em termos práticos, um explica conteúdo; o outro explica caminho de decisão.

## 6. Diagnóstico recomendado

Quando o YAML final parece estranho, a ordem correta é:

1. revisar blocking_stage
2. comparar ast_payload com final_yaml
3. ler diff_preview antes de aplicar arquivo
4. conferir chosen_tools e decision_trace para entender o caminho que levou ao resultado

## 7. Evidências no código

- src/config/agentic_assembly/assembly_service.py
  - Motivo: encadeamento principal de objective to yaml.
  - Comportamento confirmado: o fluxo consolida final_yaml em memória usando confirm com apply false.
- src/config/agentic_assembly/models.py
  - Motivo: contrato consolidado de saída.
  - Comportamento confirmado: response pública expõe blocking_stage, final_yaml, diff_preview, chosen_tools, decision_trace e validation_report.
