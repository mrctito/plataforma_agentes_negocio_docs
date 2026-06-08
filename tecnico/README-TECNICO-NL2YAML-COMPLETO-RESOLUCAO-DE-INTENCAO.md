# Manual técnico por etapa: resolução de intenção do NL2YAML

## 1. O que esta etapa cobre

Esta etapa cobre como o briefing em linguagem natural é classificado antes da montagem do draft AST. O foco aqui não é gerar YAML ainda, mas decidir qual alvo agentic faz sentido e quais sinais do texto precisam virar capacidades, perguntas ou restrições estruturadas.

## 2. Onde essa etapa roda

O fluxo passa por duas camadas principais:

- o router /config/assembly/objective-to-yaml resolve correlation_id e delega ao serviço
- o AgenticAssemblyService resolve base_yaml, requested_target e target efetivo
- quando a geração parte de prompt, o serviço chama o IntentParser

## 3. Como o alvo é resolvido

O contrato público desta trilha oficial aceita três targets:

- auto
- workflow
- deepagent

Quando o target já vem explícito, ele prevalece como contrato da chamada. Quando vem como auto, o parser híbrido usa pesos por sinais textuais para escolher entre workflow e deepagent.

## 4. O que o parser realmente faz

O IntentParser não gera YAML direto. Ele executa um passo intermediário mais seguro:

- classifica o target provável
- registra evidências textuais da classificação
- marca ambiguidade quando os sinais competem demais
- infere capacidades e configurações exigidas
- produz perguntas de clarificação quando ainda falta decisão estrutural
- monta uma IR estruturada específica do alvo escolhido

Em termos práticos, o parser evita dois extremos ruins: chutar um tipo de runtime cedo demais ou despejar prompt livre direto em compilação AST.

## 5. Como a ambiguidade é tratada

Se o texto não deixa claro se o caso é workflow ou deepagent, o parser não inventa uma resposta confiante. Ele devolve question estruturada e diagnóstico NL_TARGET_AMBIGUO.

Isso é importante porque o produto prefere bloquear por falta de clareza a gerar YAML no alvo errado.

## 6. O que sai desta etapa

Os principais artefatos técnicos produzidos aqui são:

- resolved_target
- decision_trace inicial
- intent parse result
- diagnostics de classificação
- questions de clarificação quando necessário
- IR estruturada pronta para virar AST

## 7. Diagnóstico recomendado

Quando o target resolvido parecer estranho, o caminho correto é:

1. conferir requested_target e resolved_target
2. revisar sinais textuais relevantes do prompt
3. verificar se houve questions de clarificação
4. inspecionar os diagnostics do parser antes de culpar o draft ou a validação

## 8. Evidências no código

- src/config/agentic_assembly/assembly_service.py
  - Motivo: resolução inicial de base_yaml, target e trilha de decisão.
  - Comportamento confirmado: objective to yaml resolve target antes de entrar em preflight e draft.
- src/config/agentic_assembly/nl/intent_parser.py
  - Motivo: classificação do briefing.
  - Comportamento confirmado: target auto usa pesos e perguntas de clarificação quando a intenção é ambígua.
- src/config/agentic_assembly/models.py
  - Motivo: contrato público do target.
  - Comportamento confirmado: a base de modelos ainda carrega compatibilidades históricas, mas esta documentação publica somente auto, workflow e deepagent como trilha oficial.
