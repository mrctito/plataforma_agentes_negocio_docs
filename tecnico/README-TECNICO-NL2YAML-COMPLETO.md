# Manual técnico e operacional: NL2YAML governado por AST

## 1. O que é esta feature

O slice técnico de NL2YAML é o fluxo HTTP e de serviço que recebe um briefing em linguagem natural e devolve um YAML agentic final, ou então um bloqueio estruturado indicando por que o processo não pode seguir. O caminho canônico confirmado no código é o endpoint POST /config/assembly/objective-to-yaml.

Esse fluxo não grava arquivo por padrão. Ele gera um resultado governado com AST, relatório de validação, diff preview, rastro de decisão, tools escolhidas e, quando ainda faltam decisões, questions para consolidação posterior.

Detalhamento técnico por etapa:

1. [Resolução de intenção](README-TECNICO-NL2YAML-COMPLETO-RESOLUCAO-DE-INTENCAO.md)
2. [Prontidão operacional](README-TECNICO-NL2YAML-COMPLETO-PRONTIDAO-OPERACIONAL.md)
3. [Draft assistido](README-TECNICO-NL2YAML-COMPLETO-DRAFT-ASSISTIDO.md)
4. [Validação semântica](README-TECNICO-NL2YAML-COMPLETO-VALIDACAO-SEMANTICA.md)
5. [Consolidação e preview final](README-TECNICO-NL2YAML-COMPLETO-CONSOLIDACAO-E-PREVIEW-FINAL.md)
6. [Publicação segura](README-TECNICO-NL2YAML-COMPLETO-PUBLICACAO-SEGURA.md)

## 2. Conceitos técnicos necessários

### 2.1. AssemblyTarget

O contrato público desta trilha oficial aceita três valores de target:

- auto
- workflow
- deepagent

O valor auto permite que o serviço resolva o alvo a partir do briefing e do contexto do YAML base.

### 2.2. Modos de geração

O contrato público expõe três modos:

- llm_schema
- heuristic
- auto

O modo llm_schema exige provider LLM com saída estruturada. O modo heuristic prioriza geração determinística sem depender desse provider. O modo auto admite fallback heurístico com opt-in explícito.

### 2.3. AST pública do assembly

O envelope central é AgenticDocumentAST. Ele representa o documento agentic como estrutura tipada com campos como target, selected_workflow, selected_supervisor, workflows, multi_agents, deepagent_multi_agents, tools_library e blocos de configuração relacionados.

Essa AST é a base para parse, validação, merge parcial e compilação final.

## 3. Entrypoint HTTP real

O endpoint publicado é POST /config/assembly/objective-to-yaml.

O router faz cinco coisas principais antes de chamar o serviço:

1. exige a permissão config.generate;
2. verifica se a feature está habilitada;
3. resolve o rótulo do usuário para logging;
4. injeta correlation_id no payload quando ele veio no request mas não no corpo;
5. delega o processamento ao AgenticAssemblyService.

O router também trata erros em três classes:

- FileNotFoundError vira HTTP 400;
- ValueError vira HTTP 400;
- exceções não previstas viram HTTP 500 com mensagem genérica de falha interna.

## 4. Contrato público de entrada

O request do fluxo é AssemblyObjectiveToYamlRequest. Os campos confirmados são:

- prompt: obrigatório, mínimo de 1 caractere;
- user_email: obrigatório;
- target: opcional, default auto;
- base_yaml: opcional;
- template_path: opcional;
- constraints: opcional;
- generation_mode: default llm_schema;
- correlation_id: opcional.

Na UI standalone, o corpo enviado inclui prompt, user_email, target, base_yaml e generation_mode. Quando o modo auto é escolhido, a UI também injeta constraints com auto_heuristic_fallback_enabled igual a true.

## 5. Contrato público de saída

A resposta é AssemblyObjectiveToYamlResponse. Os campos mais importantes são:

- success;
- requested_target;
- resolved_target;
- blocking_stage;
- final_yaml;
- final_yaml_text;
- ast_payload;
- diff_preview;
- chosen_tools;
- decision_trace;
- questions;
- diagnostics;
- preflight_ready;
- preflight_summary;
- preflight_checks;
- validation_report;
- correlation_id.

Esse contrato deixa explícito que o fluxo pode terminar com sucesso ou parar em um estágio conhecido: preflight, draft, validate ou confirm.

## 6. Fluxo interno passo a passo

### 6.1. Resolução de base_yaml e target

O serviço resolve primeiro o YAML base e o target efetivo. Isso é importante porque tanto preflight quanto validação dependem desse contexto.

O rastro de decisão inicial já é montado nessa fase por _build_objective_decision_trace.

### 6.2. Preflight

O serviço chama preflight com user_email, target resolvido, base_yaml, constraints, generation_mode e correlation_id.

Se preflight.ready for false, o fluxo retorna imediatamente:

- success=false;
- blocking_stage=preflight;
- final_yaml ausente;
- diagnostics consolidados com o código AST_OBJECTIVE_TO_YAML_PREFLIGHT_BLOQUEADO.

Isso evita qualquer tentativa de draft final quando o ambiente ainda não está pronto.

### 6.3. Draft

Se o ambiente estiver pronto, o serviço chama draft. O draft devolve:

- ast_draft;
- draft_fragment;
- merged_preview;
- diff_preview;
- diagnostics;
- questions;
- validation_report.

Depois disso, o serviço extrai as tools candidatas por _collect_selected_tools a partir de merged_preview, draft_fragment ou ast_draft.

Se houver questions ou validation_report inválido, o fluxo retorna:

- success=false;
- blocking_stage=draft;
- ast_payload preenchido;
- chosen_tools preenchido com base no draft;
- questions preenchidas;
- diagnostics com AST_OBJECTIVE_TO_YAML_QUESTOES_PENDENTES quando aplicável.

### 6.4. Validate

Se o draft não bloquear, o serviço chama validate com:

- target resolvido;
- base_yaml;
- ast_payload do draft;
- strict=true;
- correlation_id.

O validate faz parse do payload AST, resolve catálogo efetivo de tools, seleciona o validador semântico correto e também roda a detecção de drift do YAML governado.

Se o objetivo for entender em profundidade o que é esse hash, por que ele existe, como o runtime compara esse selo e como corrigir drift sem editar metadata manualmente, a leitura complementar oficial é:

- [docs/README-CONCEITUAL-HASH-GOVERNADO-AGENTIC.md](../conceitual/README-CONCEITUAL-HASH-GOVERNADO-AGENTIC.md)
- [docs/README-TECNICO-HASH-GOVERNADO-AGENTIC.md](README-TECNICO-HASH-GOVERNADO-AGENTIC.md)

Se a validação consolidada for inválida, o objective-to-yaml retorna:

- success=false;
- blocking_stage=validate;
- final_yaml ausente;
- ast_payload preservado.

### 6.5. Confirm em dry-run

Se validate passar, o serviço chama confirm com apply=false e force=false. Esse detalhe é central: o fluxo objective-to-yaml consolida o YAML final, mas não persiste arquivo.

O confirm:

1. resolve o target real do payload;
2. aplica answers se houver;
3. valida novamente o documento AST;
4. detecta drift do YAML governado;
5. seleciona o fragmento governado conforme o alvo;
6. faz merge no YAML base via DocumentCompiler;
7. estampa fingerprint do bloco governado;
8. gera diff preview.

Se confirm não conseguir produzir um YAML final utilizável, o fluxo retorna success=false e blocking_stage=confirm.

### 6.6. Resposta final de sucesso

Quando tudo fecha, o serviço devolve:

- success=true;
- final_yaml e final_yaml_text;
- ast_payload do draft;
- diff_preview do confirm;
- chosen_tools agora coletadas do YAML final;
- decision_trace;
- preflight_summary e preflight_checks;
- validation_report;
- correlation_id.

## 7. Como a AST é organizada

### 7.1. Envelope do documento

AgenticDocumentAST organiza o documento inteiro e consegue recortar o fragmento governado via to_fragment conforme o alvo.

Para workflow, o fragmento inclui selected_workflow, workflows_defaults, workflows e, quando houver, tools_library.

Para deepagent, o fragmento inclui selected_supervisor, multi_agents e, se necessário, tools_library. O nome selected_supervisor permanece como chave histórica; no caminho oficial ele escolhe o DeepAgent ativo.

### 7.2. WorkflowAST

WorkflowAST modela um workflow completo com nodes, edges e settings. Os nodes usam discriminação por mode. O código confirma suporte tipado para agent, router, planner, executor, if, set, merge, function, transform, rule_router, tool, schema_validator, sub_workflow, whatsapp_media_resolver, whatsapp_send e unsupported.

Isso importa porque NL2YAML não produz um blob genérico. Ele pode montar grafos determinísticos com nós especializados.

### 7.4. DeepAgentSupervisorAST

DeepAgentSupervisorAST herda de SupervisorAST e adiciona middlewares, skills, response_format, interrupt_on, permissions, async_subagents e agents especializados. Há um model_validator que força execution.type para deepagent.

Esse ponto é importante para o usuário: quando o objetivo textual aponta para deepagent, o contrato final muda de categoria, não apenas de rótulo.

## 8. Merge canônico e proteção contra sobrescrita indevida

O merge final é feito por DocumentCompiler. Ele não reescreve o YAML inteiro por padrão.

As regras confirmadas são:

- para workflow, só atualiza selected_workflow, workflows_defaults, workflows e tools_library;
- para deepagent, faz merge seletivo em multi_agents e tools_library por id;
- quando há material legado no mesmo bloco, o compiler isola o slice deepagent pelo execution.type;
- no modo auto, copia apenas as chaves presentes no fragmento produzido.

Esse desenho existe para evitar dano colateral em partes do documento que não pertencem ao alvo governado atual.

## 9. Validação semântica e guardrails

DocumentSemanticValidator delega a validação conforme AssemblyTarget.

As rotas confirmadas são:

- WorkflowSemanticValidator para workflows;
- DeepAgentSemanticValidator para deepagent;
- ToolsSemanticValidator para tools_library no documento e dentro de supervisores.

No target auto, o serviço agrega diagnósticos de workflow, deepagent e tools, além das compatibilidades internas ainda vivas no código.

Além disso, validate e confirm também verificam drift do YAML governado. O código da família de drift usa o diagnóstico AGENTIC_AST_GOVERNED_YAML_DRIFT e metadados em metadata.agentic_assembly.governed_hashes.

Para operação direta, helper público e CLI de refresh do hash governado estão documentados em detalhes em:

- [docs/README-CONCEITUAL-HASH-GOVERNADO-AGENTIC.md](../conceitual/README-CONCEITUAL-HASH-GOVERNADO-AGENTIC.md)
- [docs/README-TECNICO-HASH-GOVERNADO-AGENTIC.md](README-TECNICO-HASH-GOVERNADO-AGENTIC.md)

## 10. Modos de geração e comportamento real

### 10.1. llm_schema

É o modo padrão. Quando o provider estruturado não está pronto, o fluxo pode bloquear por indisponibilidade de LLM estruturado.

Na UI standalone, esse caso recebe mensagem acionável pedindo configuração de security_keys ou llm.azure_openai, ou mudança do modo de geração.

### 10.2. heuristic

Os testes confirmam que esse modo não bloqueia o preflight por ausência de LLM Azure estruturado. O response traz preflight_checks como AST_PREFLIGHT_HEURISTIC_MODE_READY.

Na prática, isso permite primeira versão previsível mesmo em ambientes sem provider estruturado pronto.

### 10.3. auto

Os testes confirmam que o modo auto envia constraints com auto_heuristic_fallback_enabled igual a true. O preflight devolve sinais como AST_PREFLIGHT_AUTO_FALLBACK_READY e AST_PREFLIGHT_AUTO_FALLBACK_AUTORIZADO.

O ponto importante é que esse fallback não é implícito. Ele depende de opt-in explícito.

## 11. UI standalone e experiência do operador

O frontend dedicado está em ui-plataforma-nl2yaml-studio.html e objective-yaml-studio.js.

O objetivo dessa tela é atender usuário final sem abrir a interface técnica completa do AST designer. O comportamento confirmado inclui:

- escolha de target por cards;
- exemplos prontos de briefing;
- escolha de modo de geração;
- YAML base sincronizado do contexto ou colado manualmente;
- consumo de POST /config/assembly/objective-to-yaml;
- exibição de diagnostics, validationReport, questions, chosenTools e decisionTrace;
- exibição de correlation_id;
- download de log correlacionado e system.log;
- resolução guiada de questions via POST /config/assembly/confirm com apply=false;
- sugestão assistida de output_path;
- publicação posterior com perímetro controlado.

### 11.1. Autorização na UI

O runtime reutiliza o cliente HTTP administrativo compartilhado. Para autenticação, ele resolve a chave nesta ordem prática:

- access_key extraída do YAML ativo, quando disponível;
- chave manual informada no contexto;
- sessão federada/cookies quando aplicável.

Os testes de frontend confirmam que authentication.access_key do YAML pode prevalecer até sobre uma X-API-Key manual antiga do contexto.

### 11.2. Tratamento de 403

Quando a API responde 403 por falta de permissão, a UI traduz isso para mensagem acionável dizendo que a credencial ativa não possui config.generate.

## 12. Publicação segura

Objective-to-yaml não grava arquivo. A gravação é responsabilidade de confirm com apply=true.

As regras confirmadas para publicação são:

- output_path é obrigatório quando apply=true;
- output_path deve apontar para arquivo .yaml ou .yml;
- output_path deve permanecer dentro de app/yaml;
- qualquer tentativa fora desse perímetro gera ValueError.

Isso protege a publicação governada contra escrita arbitrária em outros diretórios do repositório.

## 13. Observabilidade e diagnóstico

### 13.1. correlation_id

O router injeta correlation_id no payload quando ele já veio do request. O response sempre devolve correlation_id. A UI mostra esse valor e permite baixar o log correlacionado.

### 13.2. Log markers

O fluxo usa markers específicos, incluindo:

- AGENTIC_AST_OBJECTIVE_TO_YAML_START
- AGENTIC_AST_OBJECTIVE_TO_YAML_END

Além disso, validate, confirm, schema e catalog possuem markers próprios no assembly.

### 13.3. blocking_stage

Esse campo é a chave para troubleshooting rápido. Ele informa em qual estágio o fluxo travou: preflight, draft, validate ou confirm.

### 13.4. decision_trace e chosen_tools

Esses dois campos ajudam a entender o que o sistema decidiu. O primeiro explica decisões críticas, como target resolvido. O segundo mostra quais tools efetivamente ficaram referenciadas no payload final.

## 14. O que acontece em caso de sucesso

No caminho feliz confirmado pelos testes:

1. objective_to_yaml chama preflight, draft, validate e confirm nessa ordem;
2. success vem como true;
3. blocking_stage vem null;
4. resolved_target aponta para o alvo final, como workflow;
5. final_yaml e final_yaml_text são preenchidos;
6. ast_payload é preservado;
7. chosen_tools é preenchido a partir do YAML final;
8. decision_trace contém ao menos a decisão de target.

## 15. O que acontece em caso de erro ou bloqueio

### 15.1. Preflight bloqueado

Sinal principal: success=false e blocking_stage=preflight.

Causa provável: ambiente não pronto para geração segura.

Como diagnosticar: olhar preflight_ready, preflight_summary, preflight_checks e diagnostics.

### 15.2. Questions pendentes no draft

Sinal principal: success=false, blocking_stage=draft e questions não vazias.

Causa provável: lacuna de decisão obrigatória, como tool ambígua.

Como diagnosticar: olhar questions, diagnostics e validation_report.

### 15.3. Validate inválido

Sinal principal: success=false e blocking_stage=validate.

Causa provável: erro semântico do alvo, tools, base governada ou drift.

Como diagnosticar: olhar validation_report.diagnostics e compiled_fragment.

### 15.4. Confirm bloqueado

Sinal principal: success=false e blocking_stage=confirm.

Causa provável: impossibilidade de consolidar YAML final utilizável após validação.

### 15.5. Drift governado

Sinal principal: diagnóstico AGENTIC_AST_GOVERNED_YAML_DRIFT em validate ou confirm.

Causa provável: alteração manual no bloco governado depois da última estampa de fingerprint.

## 16. Exemplos práticos guiados

### 16.1. Workflow gerado com sucesso

Cenário: briefing pede um fluxo de vendas consultivas.

Entrada: prompt, user_email, target workflow e YAML base com provider openai.

Processamento: preflight pronto, draft válido, validate válido, confirm dry-run bem-sucedido.

Saída: selected_workflow preenchido, final_yaml_text pronto para revisão, chosen_tools listada e publicação liberada.

### 16.2. Tool ambígua

Cenário: o briefing exige uma tool, mas a escolha ainda está ambígua.

Entrada: prompt insuficientemente específico.

Processamento: draft devolve question obrigatória.

Saída: blocking_stage=draft, ast_payload parcial e questions para o operador responder.

### 16.3. Ambiente sem LLM estruturado

Cenário: o ambiente não está pronto para llm_schema.

Opção 1: modo llm_schema. O fluxo pode bloquear cedo.

Opção 2: modo heuristic. O preflight não bloqueia pela ausência desse provider e o fluxo pode seguir.

Opção 3: modo auto com opt-in explícito. O backend recebe constraints permitindo fallback heurístico.

## 17. Como colocar para funcionar

O caminho confirmado no código para uso manual é:

1. garantir que a feature do assembly AST esteja habilitada;
2. chamar POST /config/assembly/objective-to-yaml com prompt, user_email e, se necessário, base_yaml;
3. revisar success, blocking_stage, questions, diagnostics e validation_report;
4. se houver questions, resolver via confirm sem apply;
5. só publicar com confirm apply=true e output_path dentro de app/yaml.

Na UI, esse caminho já está encapsulado pela página standalone ui-plataforma-nl2yaml-studio.html.

## 18. Troubleshooting

### 18.1. A tela gera 403 mesmo com contexto carregado

Causa provável: a credencial ativa não possui config.generate.

Como confirmar: a UI mostra mensagem específica de permissão negada.

Ação recomendada: usar X-API-Key autorizada ou carregar um YAML com authentication.access_key que tenha essa permissão.

### 18.2. O preview não libera publicação

Causa provável: result.success falso, questions pendentes, ausência de ast_payload ou ausência de YAML final.

Como confirmar: revisar previewReadyForPublish, questions e validationReport.

### 18.3. output_path é rejeitado

Causa provável: caminho fora de app/yaml ou extensão inválida.

Como confirmar: a normalização do serviço exige .yaml ou .yml e proíbe sair de app/yaml.

### 18.4. O validate acusa drift

Causa provável: edição manual posterior no bloco governado do YAML base.

Como confirmar: procurar diagnóstico AGENTIC_AST_GOVERNED_YAML_DRIFT.

## 19. Explicação 101

Tecnicamente, NL2YAML aqui funciona como uma linha de montagem. O texto do usuário entra como briefing. O sistema transforma isso numa estrutura tipada, verifica se o ambiente está pronto, pergunta o que ainda falta, monta um preview confiável e só depois permite publicar. O ganho é que a plataforma deixa de confiar em texto livre e passa a confiar em contrato.

## 20. Evidências no código

- src/api/routers/config_assembly_router.py
  - Motivo da leitura: confirmar a borda HTTP do fluxo.
  - Símbolo relevante: objective_to_yaml_agentic_ast.
  - Comportamento confirmado: permissionamento, feature flag, correlação e tratamento de erro.

- src/config/agentic_assembly/models.py
  - Motivo da leitura: confirmar requests e responses públicas.
  - Símbolo relevante: AssemblyObjectiveToYamlRequest e AssemblyObjectiveToYamlResponse.
  - Comportamento confirmado: contrato de entrada, saída, questions, diagnostics e blocking_stage.

- src/config/agentic_assembly/assembly_service.py
  - Motivo da leitura: seguir o pipeline real.
  - Símbolo relevante: objective_to_yaml, validate e confirm.
  - Comportamento confirmado: preflight, draft, validate, confirm, diff preview, dry-run e publicação segura.

- src/config/agentic_assembly/ast/document.py
  - Motivo da leitura: confirmar o envelope AST governado.
  - Símbolo relevante: AgenticDocumentAST.
  - Comportamento confirmado: fragmentação por target.

- src/config/agentic_assembly/ast/workflow.py
  - Motivo da leitura: confirmar o contrato de workflow.
  - Símbolo relevante: WorkflowAST e WorkflowNodeAST.
  - Comportamento confirmado: nodes discriminados por mode e suporte a edges.

- src/config/agentic_assembly/ast/supervisor.py
  - Motivo da leitura: confirmar o contrato de supervisor.
  - Símbolo relevante: SupervisorAST.
  - Comportamento confirmado: agents, tools_library, directives e execução tipada.

- src/config/agentic_assembly/ast/deepagent.py
  - Motivo da leitura: confirmar o contrato DeepAgent.
  - Símbolo relevante: DeepAgentSupervisorAST.
  - Comportamento confirmado: middlewares, skills, interrupt_on, permissions e coerção do execution.type.

- src/config/agentic_assembly/compilers/document_compiler.py
  - Motivo da leitura: confirmar merge parcial canônico.
  - Símbolo relevante: DocumentCompiler.merge.
  - Comportamento confirmado: atualização seletiva do slice governado e merge por id para supervisors/tools.

- src/config/agentic_assembly/validators/document_validator.py
  - Motivo da leitura: confirmar validação semântica por alvo.
  - Símbolo relevante: DocumentSemanticValidator.validate.
  - Comportamento confirmado: roteamento para validators específicos e validação complementar de tools.

- app/ui/static/js/objective-yaml-studio.js
  - Motivo da leitura: confirmar a experiência standalone.
  - Símbolo relevante: submitObjective, resolveQuestions e buildPermissionDeniedMessage.
  - Comportamento confirmado: consumo do objective-to-yaml, fallback explícito no modo auto e UX de bloqueio acionável.

- tests/unit/test_02-01-78_agentic_assembly_service.py
  - Motivo da leitura: confirmar cenários críticos do serviço.
  - Símbolo relevante: testes objective_to_yaml_*.
  - Comportamento confirmado: encadeamento completo, bloqueio por questions, modo heuristic e modo auto com opt-in.

- tests/frontend/nl2yaml_studio_contract.test.js
  - Motivo da leitura: confirmar o contrato real do frontend.
  - Símbolo relevante: runtime consome objective-to-yaml e libera publicação.
  - Comportamento confirmado: payload HTTP esperado, preenchimento do preview, uso de access_key do YAML e mensagens de permissão.
