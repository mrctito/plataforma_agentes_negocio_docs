# Manual detalhado da etapa: Registry e adapters do AG-UI

## 1. O que esta etapa faz

Esta etapa conecta o protocolo comum aos runtimes e dominios concretos. Ela existe para permitir que AG-UI suporte deepagent, workflow e dominios governados sem if hardcoded no core do lifecycle. O supervisor classico `agent` foi desligado deste boundary.

Em linguagem simples: e a tomada de adaptadores que diz quem executa cada executionKind.

## 2. Onde ela entra no fluxo

No codigo lido, o router resolve o AgUiAdapterRegistry padrao, transforma esse registro em mapping e entrega o mapping ao orchestrator. Quando um run chega, o orchestrator escolhe o adapter pelo execution_kind informado no contexto.

## 3. O que entra e o que sai

Entradas confirmadas:

- execution_kind textual vindo do request
- adapters registrados explicitamente
- services auxiliares para discovery e suporte de resume/HIL

Saidas confirmadas:

- mapping fechado de execution_kind para adapter
- catalogo publico de capabilities por execution_kind
- traducao do resultado de runtimes internos para eventos AG-UI

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. AgUiAdapterRegistry.default registra deepagent, workflow, retail_demo e erp_backoffice_demo;
2. register rejeita execution_kind vazio e registro duplicado;
3. as_mapping devolve copia por convencao imutavel do catalogo;
4. AgUiCapabilitiesService.default publica um menu publico por execution_kind;
5. deepagent usa `LangGraphAgent` oficial sobre `CompiledStateGraph`;
6. workflow usa `Workflowagent`, expoe `CompiledStateGraph` e tambem delega para `LangGraphAgent`;
7. workflow preserva resume AG-UI por executor dedicado de continuidade, validando `interruptId` e carregando decisao estruturada (`approve`, `reject`, `cancel` ou `edit` com `edited_payload`) ate o contrato canonico do workflow;
8. retail_demo e erp_backoffice_demo plugam dominios governados proprios sem depender dos runtimes agentic.

O detalhe mais importante e que extensibilidade aqui nao significa improviso. Existe catalogo explicito, contrato unico e falha fechada quando um execution_kind nao foi registrado.

## 5. Decisoes tecnicas importantes

### 5.1. Registry explicito evita fallback escondido

O catalogo default registra apenas quatro execution kinds. Se um quinto tipo for necessario, ele precisa ser plugado conscientemente.

### 5.2. Supervisor classico nao e adapter AG-UI suportado

O runtime `agent` classico nao recebe adapter novo e nao aparece no discovery publico. Tentativas de usar esse caminho pelo boundary publico devem falhar com erro de deprecacao.

### 5.3. DeepAgent e Workflow usam LangGraphAgent

DeepAgent e Workflow expoem grafos LangGraph compilados para o SDK oficial AG-UI. Em linguagem simples: o boundary nao inventa uma traducao paralela quando o runtime ja e LangGraph.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- execution_kind nao registrado quebra antes de qualquer runtime rodar;
- registrar duas vezes o mesmo execution_kind gera erro imediato;
- esquecer de expor capability publica para um novo adapter cria assimetria entre discovery e execucao;
- supervisor classico legado nao e runtime AG-UI suportado;
- adapter novo que nao normalize direito seu resultado compromete o contrato do stream.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- erro AG_UI_ADAPTER_NOT_FOUND ao tentar um executionKind novo;
- lista de execution kinds conhecidos pelo registry;
- discovery de capabilities sem o execution_kind esperado;
- erro de deprecacao quando a UI tenta usar o supervisor classico legado;
- pausa HIL sem retomada funcional quando o adapter nao usa o suporte comum correto.

Em linguagem simples: se AG-UI parece pouco generico, esta camada mostra exatamente onde a generalizacao esta ou nao esta de fato implementada.

## 8. Exemplo pratico guiado

Cenario: o produto quer adicionar um execution_kind novo para um modulo financeiro.

1. cria-se um adapter que traduza o runtime financeiro para eventos AG-UI;
2. registra-se esse adapter no registry;
3. publica-se a capability correspondente no discovery;
4. o orchestrator passa a resolvelo sem mudar sua propria logica de lifecycle.

O valor desta etapa e permitir crescimento do slice sem reescrever o core a cada novo dominio.

## 9. Evidencias no codigo

- src/api/services/ag_ui_adapter_registry.py
  - Simbolo relevante: AgUiAdapterRegistry.default
  - Comportamento confirmado: registro explicito de deepagent, workflow, retail_demo e erp_backoffice_demo.
- src/api/services/ag_ui_adapter_registry.py
  - Simbolo relevante: register
  - Comportamento confirmado: proibicao de execution_kind vazio ou duplicado.
- src/api/services/ag_ui_capabilities_service.py
  - Simbolo relevante: AgUiCapabilitiesService.default
  - Comportamento confirmado: discovery publico por execution_kind, incluindo catalogo retail_demo.
- src/api/services/ag_ui_deepagent_adapter.py
  - Simbolo relevante: AgUiDeepAgentAdapter
  - Comportamento confirmado: deepagent usa LangGraphAgent oficial no caminho padrao e continuidade agentic para resume.
- src/api/services/ag_ui_workflow_adapter.py
  - Simbolo relevante: AgUiWorkflowAdapter.execute
  - Comportamento confirmado: workflow usa LangGraphAgent oficial no caminho padrao e executor dedicado para resume estruturado, sem runtime paralelo fora de LangGraph.
