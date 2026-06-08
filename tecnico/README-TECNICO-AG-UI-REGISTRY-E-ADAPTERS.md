# Manual técnico por etapa: registry e adapters do AG-UI

## 1. O que esta etapa cobre

Esta etapa cobre o catálogo explícito de adapters e o papel técnico de cada executionKind suportado no slice AG-UI.

O objetivo do registry é simples: impedir fallback implícito e concentrar o mapeamento entre executionKind e runtime real em um único lugar.

## 2. Registry padrão confirmado

O registry padrão registra exatamente quatro executionKinds:

- deepagent
- workflow
- retail_demo
- erp_backoffice_demo

O registro falha cedo se o nome vier vazio ou se houver duplicidade. Isso evita dois tipos de fragilidade: wiring hardcoded espalhado e sobrescrita silenciosa de adapter.

## 3. Papel técnico de cada adapter

### 3.1. deepagent

Usa o runtime de DeepAgent, expõe `CompiledStateGraph` e delega a execução padrão para `LangGraphAgent` oficial do pacote `ag-ui-langgraph`. O resume continua usando a continuidade agentic canônica.

### 3.2. workflow

Inicializa `Workflowagent`, obtém o `CompiledStateGraph` e delega a execução padrão para `LangGraphAgent`. O resume usa executor dedicado que valida a interrupção persistida, traduz `approve`, `reject`, `cancel` e `edit` com `edited_payload` para o contrato canônico do workflow e chama o serviço canônico de continuação sem criar runtime paralelo fora de LangGraph.

### 3.3. retail_demo

É o adapter de domínio governado que entrega capabilities fechadas de varejo e dashboard dinâmico sem liberar SQL livre no browser.

### 3.4. erp_backoffice_demo

É o capability pack governado de backoffice, atualmente focado em `fechar_caixa`, sem expor procedure, DSN ou segredo no discovery público.

## 4. Por que isso importa tecnicamente

Sem o registry, o router ou o orquestrador precisariam saber detalhes de cada runtime. Com o registry explícito:

- o core só resolve executionKind
- cada adapter controla a tradução do seu domínio para AG-UI
- capacidades novas podem ser plugadas sem ifs espalhados
- executionKind inexistente falha de forma observável

## 5. Erros e limites importantes

Os comportamentos mais importantes desta etapa são:

- adapter não registrado não tem fallback
- adapter duplicado não sobrescreve registro anterior
- resume permanece dependente da capacidade real do adapter
- modos legados não aparecem no registry público AG-UI
- Workflow e DeepAgent usam o adapter oficial LangGraph quando o runtime expõe `CompiledStateGraph`

## 6. Diagnóstico recomendado

Quando um executionKind não funciona:

1. confira se ele está no registry padrão
2. valide se o adapter suporta resume ou apenas execução simples
3. verifique se a falha veio do domínio do adapter ou do catálogo de registro

## 7. Evidências no código

- src/api/services/ag_ui_adapter_registry.py
  - Motivo: catálogo explícito do slice.
  - Comportamento confirmado: o registry padrão registra apenas deepagent, workflow, retail_demo e erp_backoffice_demo.
- src/api/services/ag_ui_adapter_registry.py
  - Motivo: proteção contra wiring frágil.
  - Comportamento confirmado: nomes vazios e duplicados são rejeitados explicitamente.
- src/api/services/ag_ui_run_orchestrator.py
  - Motivo: acoplamento entre registry e execução.
  - Comportamento confirmado: executionKind inexistente é convertido em erro estruturado, sem adapter implícito.
