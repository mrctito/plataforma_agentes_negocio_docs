# Manual detalhado da etapa: Draft assistido do NL2YAML

## 1. O que esta etapa faz

Esta etapa transforma o briefing em uma AST inicial validavel e em um preview mesclado. Ela existe para criar um ponto de partida governado, mas sem fingir completude quando ainda faltam decisoes importantes.

Em linguagem simples: e a primeira versao estruturada do que o usuario pediu construir.

## 2. Onde ela entra no fluxo

No codigo lido, o draft e executado depois do preflight. Ele pode seguir pelo caminho estruturado com LLM, cair para heuristica no modo auto com opt-in explicito ou rodar diretamente no modo heuristic.

## 3. O que entra e o que sai

Entradas confirmadas:

- prompt
- generation_mode
- requested_target e resolved_target
- base_yaml
- constraints
- tool_catalog efetivo e catalogo heuristico

Saidas confirmadas:

- ast_draft
- draft_fragment
- merged_preview
- diff_preview
- diagnostics
- questions
- validation_report do rascunho

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. draft resolve base_yaml e target efetivo;
2. resolve o catalogo efetivo e o catalogo heuristico de tools;
3. se houver prompt, chama _build_draft_from_prompt;
4. no modo llm_schema ou auto, o backend tenta gerar ast_payload estruturado com o gerador LLM;
5. se o LLM devolver AST, o service executa reparo guiado e validacao da AST;
6. se a AST continuar invalida e o modo for auto, o fluxo pode cair para heuristica, mas so com autorizacao explicita;
7. se o LLM nao devolver payload utilizavel e o modo for auto, o fallback heuristico tambem pode ser acionado;
8. no modo heuristic, o fluxo vai direto para a construcao heuristica do documento;
9. depois do documento AST existir, o service seleciona o fragmento governado, faz merge com o YAML base e monta diff_preview;
10. se houver perguntas obrigatorias ou invalidade semantica, o objective_to_yaml bloqueia no draft e devolve essas pendencias.

O detalhe mais importante e que o draft nao significa sucesso. Ele pode terminar com AST parcial, perguntas pendentes e preview util para revisão, mas ainda sem direito a YAML final.

## 5. Decisoes tecnicas importantes

### 5.1. LLM estruturado nao e caminho unico

O produto aceita que o draft seja montado por heuristica quando o modo escolhido pedir isso ou quando o modo auto tiver opt-in explicito para fallback. Isso reduz dependencia cega de um unico mecanismo de geração.

### 5.2. Perguntas pendentes sao parte do contrato do draft

Quando faltam decisoes, o sistema devolve questions em vez de preencher lacunas por inferencia fraca. Isso e fundamental para governanca.

### 5.3. Preview ja nasce com merge e diff

O draft nao gera so AST interna. Ele tambem devolve merged_preview e diff_preview para que a interface mostre impacto real antes da publicacao.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- LLM pode nao devolver ast_payload utilizavel;
- AST do LLM pode continuar invalida mesmo apos reparo guiado;
- fallback heuristico do modo auto pode estar desabilitado e bloquear o fluxo;
- perguntas obrigatorias impedem o fluxo de seguir como se o documento estivesse pronto;
- preview pode existir sem que o draft seja semanticamente valido.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- questions na resposta do draft ou do objective_to_yaml;
- diagnostics como AST_DRAFT_AUTO_LLM_INVALIDO, AST_DRAFT_AUTO_FALLBACK_AUTORIZADO e AST_DRAFT_FALLBACK_HEURISTICO;
- validation_report.is_valid do draft;
- merged_preview e diff_preview para ver o que foi tentado montar;
- generation_mode retornado no decision_trace.

Em linguagem simples: quando o usuario diz que o sistema entendeu a intencao so pela metade, a verdade geralmente esta aqui, entre rascunho, perguntas e fallback.

## 8. Exemplo pratico guiado

Cenario: o operador usa generation_mode=auto com opt-in para fallback heuristico.

1. o draft tenta primeiro o gerador LLM estruturado;
2. o payload volta invalido mesmo apos reparo;
3. o backend registra diagnostico de LLM invalido;
4. como ha autorizacao explicita, aciona fallback heuristico;
5. monta um novo draft, preview e diff;
6. se ainda houver perguntas, bloqueia em draft e devolve somente o que falta decidir.

O valor desta etapa e produzir o maximo de estrutura confiavel possivel sem disfarcar lacunas.

## 9. Evidencias no codigo

- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: draft
  - Comportamento confirmado: resolve target, catalogos, AST draft, preview mesclado e diff do rascunho.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: _build_draft_from_prompt
  - Comportamento confirmado: encadeamento entre LLM estruturado, reparo, perguntas e caminho heuristico.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: _build_draft_from_auto_heuristic_fallback
  - Comportamento confirmado: fallback heuristico do modo auto so ocorre com autorizacao explicita.
- app/ui/static/js/objective-yaml-studio.js
  - Simbolo relevante: objectiveStudioBuildGenerationConstraints e objectiveStudioBuildBlockedMessage
  - Comportamento confirmado: a UI envia o opt-in do modo auto e explica bloqueios acionaveis ao operador.
