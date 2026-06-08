# Manual detalhado da etapa: Resolucao de intencao do NL2YAML

## 1. O que esta etapa faz

Esta etapa enquadra o briefing em um alvo agentic concreto. Ela existe porque workflow e deepagent nao compartilham o mesmo contrato estrutural oficial. Sem esse enquadramento, o sistema geraria AST em trilho errado e a validacao posterior viraria remendo.

Em linguagem simples: e a etapa que responde qual tipo de artefato a plataforma esta sendo solicitada a montar.

## 2. Onde ela entra no fluxo

No codigo lido, a resolucao acontece logo no inicio do objective_to_yaml e tambem no draft. Antes de preflight, draft, validate ou confirm, o service resolve qual target vale de fato para aquele request.

## 3. O que entra e o que sai

Entradas confirmadas:

- target pedido pelo cliente
- prompt em linguagem natural
- base_yaml opcional

Saidas confirmadas:

- resolved_target concreto
- rastro inicial de decisao com origem do target
- escolha do trilho semantico usado pelo resto da pipeline

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. se o request vier com target explicito diferente de auto, o service aceita esse alvo sem heuristica adicional;
2. se o target for auto e houver prompt, o IntentParser classifica o briefing e escolhe workflow ou deepagent;
3. se nao houver prompt suficiente, o service tenta inferir o alvo pela estrutura do YAML base;
4. se multi_agents existir e algum execution.type for deepagent, o alvo vira deepagent;
5. se multi_agents existir sem deepagent, o documento deve ser tratado como legado residual fora do caminho agentic moderno e exige migracao explicita antes de continuar;
6. se workflows existir, o alvo vira workflow;
7. se nada disso ajudar, o fallback final continua sendo workflow;
8. o service registra decision_trace indicando se a decisao veio do request, da classificacao NL ou da estrutura do YAML base.

O detalhe mais importante e que o modo auto nao elimina a necessidade de target concreto. Ele apenas atrasa essa decisao para um ponto governado do backend.

Outro detalhe importante: no caminho moderno, o modo auto escolhe apenas entre workflow e deepagent. Supervisor classico nao entra mais como opcao formal do NL2YAML.

Outro detalhe importante: o modo auto precisa separar workflow de deepagent, mas isso nao significa que workflow concorra com a trilha multi_agents. Workflow continua sendo backbone proprio; no caminho oficial agentic, o runtime suportado dessa familia e DeepAgent.

## 5. Decisoes tecnicas importantes

### 5.1. Target explicito vence heuristica

Quando o cliente informa target diferente de auto, o backend respeita essa escolha. Isso evita que a classificacao de linguagem natural reescreva intencao declarada.

### 5.2. O prompt so classifica quando realmente existe

Se o briefing textual nao estiver disponivel, o service nao inventa semantica pelo vazio. Ele recorre ao YAML base e, se necessario, ao default compatível.

### 5.3. Workflow continua sendo o default de compatibilidade

Na ausencia de sinais inequívocos, o sistema cai em workflow. Isso simplifica o caminho feliz, mas tambem reforca por que target explicito continua sendo preferivel em cenarios ambiguos.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- target auto pode cair em workflow quando o YAML base ainda nao expõe escopo agentic claro;
- YAML base com multiplos escopos potenciais aumenta ambiguidade;
- briefing pouco especifico pode classificar o artefato de forma insuficiente;
- confirm precisa resolver o target novamente se o ast_payload chegar sem target confiavel.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- requested_target versus resolved_target na resposta final;
- decision_trace com source igual a request.target, nl.classification ou base_yaml.structure;
- presence de multi_agents com execution deepagent no YAML base;
- classificacao do prompt quando o target veio do texto e nao do YAML.

Em linguagem simples: se a pipeline montou o tipo errado de artefato, quase sempre o problema nasce aqui.

## 8. Exemplo pratico guiado

Cenario: o cliente envia target=auto e escreve que quer um deepagent com validacao humana antes de publicar conclusoes sensiveis.

1. o service recebe target auto;
2. o prompt passa pelo IntentParser;
3. a classificacao detecta sinais de deepagent;
4. o resolved_target vira deepagent;
5. todo o restante da esteira usa esse alvo para preflight, draft, validate e confirm.

O valor desta etapa e impedir que a plataforma trate uma intencao de DeepAgent como se fosse apenas um workflow generico.

## 9. Evidencias no codigo

- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: _resolve_target
  - Comportamento confirmado: prioridade de target explicito, classificacao por prompt, inferencia por YAML base e fallback final para workflow.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: _build_objective_decision_trace
  - Comportamento confirmado: rastro de decisao informando a origem da escolha do target.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: _resolve_target_for_confirm
  - Comportamento confirmado: confirm reconstroi o target quando o ast_payload nao chega com alvo inequívoco.
