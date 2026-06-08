# Manual conceitual e de decisão: DeepAgent x Workflowagent

## 1. O que este documento responde

Este documento existe para responder uma dúvida prática que aparece com frequência: DeepAgent e Workflowagent são a mesma coisa com nomes diferentes ou são duas espinhas dorsais reais da plataforma.

A resposta correta, baseada no código executável atual, é direta: eles são duas entidades fortes, com vida própria, contratos próprios e propósitos diferentes.

Em linguagem simples, o Workflowagent é o motor de fluxo determinístico da plataforma. O DeepAgent é o motor de supervisão agentic com autonomia governada. Os dois convivem no mesmo ecossistema, mas não resolvem o mesmo problema.

## 2. Resposta curta

Se a necessidade principal é desenhar um processo com etapas, desvios, nodes, edges e estado previsível, o backbone correto tende a ser o Workflowagent.

Se a necessidade principal é operar um agente supervisor com ferramentas, memória, middlewares, permissões, HIL, subagentes e execução mais aberta, o backbone correto tende a ser o DeepAgent.

O erro mais comum é tratar workflow como sucessor automático de qualquer outro runtime agentic. O código do projeto não sustenta isso. DeepAgent e WorkflowAgent continuam sendo espinhas dorsais diferentes.

## 3. Como o código prova que são duas espinhas dorsais próprias

O primeiro ponto de prova é a resolução do alvo agentic.

Em [../tools/vscode-agentic-language-server/server/src/agentic_yaml_lsp/target_resolution.py](../tools/vscode-agentic-language-server/server/src/agentic_yaml_lsp/target_resolution.py), o sistema resolve `workflows` como alvo `workflow` e resolve `multi_agents` com `execution.type=deepagent` como alvo DeepAgent.

Isso importa porque o runtime não trata tudo como um mesmo bloco genérico. Ele identifica qual backbone está ativo antes de validar e executar.

O segundo ponto de prova é o contrato oficial dos alvos.

Em [../src/config/agentic_assembly/models.py](../src/config/agentic_assembly/models.py), o enum `AssemblyTarget` mantém contratos distintos para workflow e DeepAgent, além de compatibilidades históricas que não fazem parte da trilha oficial.

Em [../src/config/agentic_assembly/schema_service.py](../src/config/agentic_assembly/schema_service.py), o serviço de schema publica `WorkflowAST` e `DeepAgentSupervisorAST` como schemas distintos.

O terceiro ponto de prova é a AST.

Em [../src/config/agentic_assembly/ast/workflow.py](../src/config/agentic_assembly/ast/workflow.py), o contrato do workflow é centrado em `workflows`, `nodes`, `edges` e `settings`.

Em [../src/config/agentic_assembly/ast/deepagent.py](../src/config/agentic_assembly/ast/deepagent.py), o contrato do DeepAgent adiciona `middlewares`, `skills`, `interrupt_on`, `permissions`, `async_subagents` e agentes especializados. Além disso, o `model_validator` de `DeepAgentSupervisorAST` força `execution.type` para `deepagent`.

O quarto ponto de prova é o runtime real.

Em [../src/agentic_layer/background_execution/runtime.py](../src/agentic_layer/background_execution/runtime.py), o executor background decide explicitamente entre três ramos: `agent`, `deepagent` e `workflow`. Para DeepAgent, ele usa `DeepAgentSupervisor`. Para workflow, ele usa `WorkflowOrchestrator`.

O quinto ponto de prova é a validação e o controle de deriva.

Em [../src/config/agentic_assembly/drift_detector.py](../src/config/agentic_assembly/drift_detector.py), o fragmento governado de `workflow` usa `selected_workflow`, `workflows_defaults`, `workflows` e `tools_library`. Já o fragmento governado de DeepAgent usa `selected_supervisor`, `multi_agents` e `tools_library`.

Isso mostra que até o bloco do YAML monitorado como contrato oficial muda conforme o backbone.

## 4. O que é o Workflowagent na prática

O Workflowagent é a espinha dorsal de automação determinística.

Na prática, isso significa que ele foi desenhado para executar um processo declarado com começo, meio e fim previsíveis, mesmo quando alguns pontos do caminho usam IA para decidir algo.

O centro de gravidade do workflow não é um supervisor autônomo. O centro de gravidade é o fluxo.

No código, isso aparece de forma explícita em [README-CONCEITUAL-AGENTE-WORKFLOW-COMPLETO.md](README-CONCEITUAL-AGENTE-WORKFLOW-COMPLETO.md) e em [../src/agentic_layer/workflow/agent_workflow.py](../src/agentic_layer/workflow/agent_workflow.py). A classe `Workflowagent` compila nodes, monta um `StateGraph`, controla transições e depende de um contexto de workflow ativo.

Em linguagem simples, o Workflowagent funciona como uma linha de produção configurável.

Ele é especialmente forte quando você precisa de:

- etapas claras;
- transições previsíveis;
- grafo declarativo;
- rastreabilidade de qual node executou;
- controle forte sobre o caminho do processo;
- subfluxos e reuso de partes do processo.

## 5. O que é o DeepAgent na prática

O DeepAgent é a espinha dorsal de supervisão agentic governada.

Na prática, isso significa que ele foi desenhado para coordenar agentes, ferramentas, memória, permissões, shell, filesystem, HIL, subagentes e execução longa sem reduzir tudo a um grafo determinístico.

O centro de gravidade do DeepAgent não é o desenho do fluxo. O centro de gravidade é a supervisão operacional do trabalho do agente.

No código, isso aparece em [README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md](README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md) e em [../src/agentic_layer/supervisor/deep_agent_supervisor.py](../src/agentic_layer/supervisor/deep_agent_supervisor.py). A classe `DeepAgentSupervisor` monta middlewares, telemetria, regras de tool, limites, memória e integrações de execução agentic mais rica.

Em linguagem simples, o DeepAgent funciona como um supervisor operacional com autonomia governada.

Ele é especialmente forte quando você precisa de:

- coordenação de agentes especialistas;
- memória operacional durável;
- uso de ferramentas reais com governança;
- permissões explícitas de filesystem e shell;
- HIL com continuidade real;
- subagentes síncronos ou assíncronos;
- execução longa ou em background com mais autonomia.

## 6. Diferença estrutural

Esta é a diferença mais importante.

Workflowagent organiza o sistema em torno de um grafo declarativo.

- A unidade principal é o workflow.
- O contrato central é `workflows`.
- Os elementos principais são `nodes`, `edges`, `settings`, `selected_workflow` e estado do grafo.
- A pergunta dominante é: qual é o próximo passo do processo.

DeepAgent organiza o sistema em torno de uma coordenação agentic governada.

- A unidade principal é o DeepAgent.
- O contrato central é `multi_agents` com `execution.type=deepagent`.
- Os elementos principais são `middlewares`, `skills`, `permissions`, `interrupt_on`, `async_subagents`, memória e catálogo de tools.
- A pergunta dominante é: como o supervisor deve conduzir e governar o trabalho do agente.

Em termos simples, o workflow modela trilho. O DeepAgent modela supervisão.

## 7. Diferença de propósito

O propósito do Workflowagent é transformar processo em fluxo executável.

O propósito do DeepAgent é transformar autonomia agentic em operação governada.

Isso gera consequências práticas.

Quando o problema é “preciso de um processo previsível, com etapas, desvios e pontos de escrita/leitura de estado”, o workflow tende a ser a escolha correta.

Quando o problema é “preciso de um agente que trabalhe com ferramentas, memória, aprovações, permissões e coordenação mais livre”, o DeepAgent tende a ser a escolha correta.

## 8.1. Ponte canônica entre os dois backbones

O código atual agora suporta uma ponte oficial entre as duas espinhas dorsais sem misturá-las: o node `deepagent_call` do Workflowagent.

Em termos simples, isso significa o seguinte.

- o Workflowagent continua decidindo quando chamar, quando pausar e para onde seguir depois;
- o DeepAgent continua decidindo como executar a tarefa aberta, com ferramentas, HIL e subagentes próprios;
- a ponte entre os dois é uma delegação explícita, não um atalho escondido e não uma fusão de contratos.

Na prática, `deepagent_call` permite que um workflow pare em um passo, chame um DeepAgent específico por `supervisor_id`, receba o retorno síncrono e continue o grafo. Se o DeepAgent entrar em aprovação humana, o próprio workflow pausa com `interrupt(...)` e, na retomada, traduz a decisão para `Command(resume=...)` do DeepAgent usando a mesma `thread_id`.

O efeito arquitetural é importante: agora existe cooperação oficial entre os dois backbones, mas cada um continua com vida própria.

## 9. O que eles não são

Workflowagent não é um substituto natural do DeepAgent.

Ele pode até resolver alguns casos por fluxo determinístico, mas isso não o transforma em sucessor do universo DeepAgent.

DeepAgent não é “workflow com mais prompt”.

Ele é outro backbone, com outro contrato, outra AST, outra validação, outro runtime e outro conjunto de capacidades.

## 10. Quando usar um ou outro

Use Workflowagent quando:

- o processo precisa ser explicado como fluxo passo a passo;
- o desenho do caminho é mais importante do que a autonomia do agente;
- você quer nodes e edges como contrato principal;
- a previsibilidade do processo é prioridade alta.

Use DeepAgent quando:

- o trabalho precisa de ferramentas, memória e supervisão mais rica;
- existe HIL, permissões ou governança operacional mais pesada;
- a execução pode ser longa, exploratória ou multiagente;
- você quer autonomia real, mas sob política explícita.

Avalie com cuidado quando:

- o problema parece fluxo e supervisão ao mesmo tempo;
- há mistura de processo corporativo rígido com investigação aberta;
- existe vontade de usar DeepAgent só porque ele é mais poderoso no papel.

Nesses casos, a decisão correta não é “sempre usar o mais poderoso”. A decisão correta é escolher o backbone que mais se aproxima da natureza do problema. Quando for necessário usar os dois, a ponte canônica passa a ser `deepagent_call`.

## 11. Regra prática de arquitetura

Se a pergunta principal do projeto for “qual é o fluxo e quais são as transições”, comece pensando em Workflowagent.

Se a pergunta principal do projeto for “como o agente vai operar, lembrar, pedir aprovação, usar tools e coordenar trabalho”, comece pensando em DeepAgent.

Se a tarefa for sair de material legado ou de um desenho agentic inadequado, a hipótese inicial correta é avaliar DeepAgent e WorkflowAgent separadamente, em vez de presumir substituição automática.

## 12. Leitura complementar

Para entender o Workflowagent em profundidade, use [README-CONCEITUAL-AGENTE-WORKFLOW-COMPLETO.md](README-CONCEITUAL-AGENTE-WORKFLOW-COMPLETO.md).

Para entender o DeepAgent em profundidade, use [README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md](README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md).

Para entender como o YAML, a AST e o assembly tratam esses contratos, use [README-CONCEITUAL-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md](README-CONCEITUAL-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md) e [README-CONCEITUAL-NL2YAML-COMPLETO.md](README-CONCEITUAL-NL2YAML-COMPLETO.md).

## 13. Resumo final

Workflowagent e DeepAgent não são duas embalagens para a mesma ideia. O Workflowagent existe para governar fluxo determinístico. O DeepAgent existe para governar supervisão agentic com autonomia operacional ampliada.
