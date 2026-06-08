# Manual tecnico e operacional: exemplos YAML reais de DeepAgent em ERP

## 1. Escopo tecnico deste manual

Este manual mostra como ler, adaptar e diagnosticar exemplos reais de YAML DeepAgent com cara de ERP que ja existem no repositorio. O objetivo nao e repetir toda a documentacao do runtime DeepAgent. O objetivo e mostrar a sintaxe efetiva e a funcao operacional de cada bloco nos exemplos reais.

Os artefatos-base usados aqui foram:

1. app/yaml/system/rag-config-modelo.yaml.
2. app/yaml/rag-config-linx-food.yaml.
3. app/yaml/rag-config-linx-retail.yaml.
4. app/yaml/agent-linx-food-cockpit-prd.yaml.
5. app/yaml/rag-config-pdv-vendas-demo.yaml.

## 2. Quais exemplos reais existem hoje

### 2.1. Template canonico reutilizavel

Arquivo: app/yaml/system/rag-config-modelo.yaml.

Uso pratico: base para montar DeepAgent com SQL dinamico, API dinamica, RAG e memoria duravel.

### 2.2. DeepAgent generalista para food service

Arquivo: app/yaml/rag-config-linx-food.yaml.

Uso pratico: perguntas e operacoes de negocio que misturam clientes, vendas, categorias, APIs e conhecimento.

### 2.3. DeepAgent generalista para retail

Arquivo: app/yaml/rag-config-linx-retail.yaml.

Uso pratico: mesma espinha de negocio, com vocabulario retail.

### 2.4. DeepAgent de cockpit operacional

Arquivo: app/yaml/agent-linx-food-cockpit-prd.yaml.

Uso pratico: perguntas sobre lojas, saude fiscal e monitoramento de notas fiscais.

### 2.5. DeepAgent de AG-UI PDV

Arquivo: app/yaml/rag-config-pdv-vendas-demo.yaml.

Uso pratico: analytics, dashboard e leitura assistida de vendas, checkout, catalogo e clientes.

## 3. Espinha dorsal comum dos exemplos

Apesar das variacoes de negocio, os YAMLs reais lidos repetem o mesmo desenho macro.

```yaml
tools_library: []

selected_supervisor: "algum_supervisor"  # quando o YAML quer evitar ambiguidade

multi_agents:
  - id: "supervisor_x"
    execution:
      type: "deepagent"
      default_mode: "direct_async"
    middlewares:
      filesystem:
        enabled: true|false
      shell:
        enabled: false
      memory:
        enabled: true
        sources: []
      subagents:
        enabled: true
      human_in_the_loop:
        enabled: false
      summarization:
        enabled: false
      pii:
        enabled: true
        rules: []
      todo_list:
        enabled: true
      skills:
        enabled: false
    permissions:
      - operations: ["read"]
        paths: ["/memories/**"]
        mode: "allow"
    memory:
      - "/memories/user.md"
    backend:
      enabled: true
      type: "store"
      scope: "user"
      policy: "read_write"
      redis:
        url: "${...}"
        key_prefix: "..."
        ttl_seconds: 604800
    agents:
      - id: "subagente"
        tools:
          - "dyn_sql<alguma_query>"
        execution:
          default_mode: "direct_sync"
    local_tools_configuration:
      sql_dynamic:
      api_dynamic:
```

O ponto tecnico central e este: o dominio muda, mas o esqueleto agentic continua quase igual.

## 4. Exemplo 1: template canonico para ERP generalista

No modelo canonico, o supervisor DeepAgent aparece com execution.type igual a deepagent, middlewares padrao, `memory` top-level e `backend` persistente em Redis. Os subagentes mapeiam funcoes classicas de um ERP ampliado.

1. analista SQL.
2. especialista em API.
3. validador de documentos brasileiros.
4. pesquisador web e RAG.
5. pesquisador AWS.
6. assistente de compras.
7. utilitarios.

O fragmento mais importante e a combinacao entre subagente e tool governada.

```yaml
      - id: "sql_analyst"
        tools:
          - "dyn_sql<buscar_cliente>"
          - "dyn_sql<historico_compras_cliente>"
          - "dyn_sql<top_produtos_vendidos>"
          - "proc_sql<fechar_caixa>"
```

Isso significa que o agente de dados nao recebe SQL livre. Ele recebe consultas e procedures nomeadas.

### O que esse exemplo ensina

1. O DeepAgent pode coordenar varios especialistas em um unico supervisor.
2. dyn_sql, dyn_api e proc_sql sao o principal mecanismo de governanca do dominio.
3. O uso de execution.default_mode igual a direct_sync nos filhos e compatibilizado com default_mode mais assincrono no supervisor.

## 5. Exemplo 2: cockpit operacional por API

O arquivo agent-linx-food-cockpit-prd.yaml mostra um caso mais enxuto e mais corporativo. O supervisor cockpit_supervisor usa DeepAgent, memoria Redis e subagentes focados em operacao.

Trecho reduzido:

```yaml
multi_agents:
  - id: "cockpit_supervisor"
    execution:
      type: "deepagent"
      default_mode: "direct_async"
    memory:
      - "/memories/user.md"
    backend:
      enabled: true
      type: "store"
    agents:
      - id: "especialista_em_lojas"
        tools:
          - "dyn_api<listar_quantidade_de_lojas>"
          - "dyn_api<buscar_lojas_por_cnpj_ou_cpf>"
      - id: "especialista_fiscal"
        tools:
          - "dyn_api<consultar_saude_notas_fiscais>"
          - "dyn_api<consultar_data_atual>"
```

### Por que esse exemplo e tecnicamente importante

1. O dominio entra por endpoints din_api especificos, nao por cliente HTTP generico.
2. O YAML local declara os endpoints em local_tools_configuration.api_dynamic.endpoints.
3. O especialista fiscal carrega regra operacional no proprio YAML, como sempre consultar data atual antes de periodos relativos.

### Como adaptar

Troque os endpoint_id e o bloco api_dynamic sem mexer no esqueleto DeepAgent. Esse e o principal ganho arquitetural.

## 6. Exemplo 3: AG-UI PDV com selected_supervisor explicito

O arquivo rag-config-pdv-vendas-demo.yaml mostra um caso mais moderno e mais seguro para UI generativa. Aqui o YAML declara explicitamente qual supervisor deve ser o ativo.

```yaml
selected_supervisor: "ag_ui_pdv_vendas_supervisor"

multi_agents:
  - id: "ag_ui_pdv_vendas_supervisor"
    execution:
      type: "deepagent"
```

Isso e importante porque remove ambiguidade logo na raiz do documento.

### Subdominios reais do exemplo PDV

1. subdominio_vendas.
2. subdominio_checkout_ucp.
3. subdominio_catalogo.
4. subdominio_clientes.
5. subdominio_dashboard_dinamico.

Cada subdominio recebe apenas as queries aprovadas para aquele recorte.

```yaml
      - id: "subdominio_vendas"
        tools:
          - "dyn_sql<pdv_vendas_kpis_periodo>"
          - "dyn_sql<pdv_vendas_por_loja>"
          - "dyn_sql<pdv_vendas_por_produto>"
```

### O caso especial do dashboard dinamico

O subagente de dashboard usa response_format estruturado com JSON Schema e limita queryId a uma lista fechada.

```yaml
      - id: "subdominio_dashboard_dinamico"
        response_format:
          type: "object"
          additionalProperties: false
          properties:
            dataSources:
              items:
                properties:
                  queryId:
                    enum:
                      - "pdv_dashboard_series_vendas_periodo"
                      - "pdv_dashboard_ranking_dimensoes"
                      - "pdv_dashboard_mix_pagamento_entrega"
```

Isso muda completamente o comportamento do DeepAgent. Ele deixa de ser so um explicador e vira um montador de payload estruturado para interface visual.

## 7. O papel de memory e backend nesses exemplos

Os exemplos reais usam Redis como backend. Os padroes observados foram:

1. key_prefix generico deepagent_store no template e nos casos food/retail.
2. key_prefix especializado deepagent_store:ag_ui_pdv_vendas no caso AG-UI PDV.
3. scope e policy explicitados no exemplo AG-UI PDV.

Trecho real do PDV:

```yaml
    memory:
      - "/memories/user.md"
    backend:
      enabled: true
      type: "store"
      scope: "user"
      policy: "read_write"
      redis:
        url: "${REDIS_PROMETEU_GENERIC_RAG_URL}"
        key_prefix: "deepagent_store:ag_ui_pdv_vendas"
        ttl_seconds: 604800
```

### Leitura pratica

1. scope user e bom para historico por usuario.
2. key_prefix especializado evita colisao entre produtos ou modulos.
3. policy read_write e util quando o agente precisa aprender contexto de uso.

## 8. Filesystem e permissions nos exemplos ERP

Nos exemplos food, retail e cockpit, filesystem aparece habilitado, mas com permissions extremamente restritas ao namespace /memories.

```yaml
    permissions:
      - operations: ["read"]
        paths: ["/memories/**"]
        mode: "allow"
      - operations: ["write"]
        paths: ["/memories/**"]
        mode: "deny"
```

Leitura tecnica: o runtime permite o middleware, mas o exemplo de negocio continua conservador. Em ERP, isso reduz risco de manipular arquivos arbitrarios sem necessidade.

## 9. Como localizar o dominio no YAML

Os exemplos reais mostram tres pontos onde o dominio aparece.

1. Na descricao do supervisor e dos subagentes.
2. Nos ids das tools governadas.
3. Em local_tools_configuration, principalmente sql_dynamic e api_dynamic.

Se voce quer adaptar um exemplo para outro ERP, estes sao os blocos que mais mudam. O esqueleto DeepAgent tende a mudar pouco.

## 10. Como adaptar um exemplo real sem desmontar a arquitetura

### Passo 1. Escolha o molde certo

1. Use o modelo canonico para DeepAgent transversal.
2. Use o cockpit se o caso for monitoramento por API.
3. Use o AG-UI PDV se o caso for dashboard e analytics assistido.

### Passo 2. Renomeie supervisor e subagentes

Troque ids, nomes e descricoes para o dominio alvo, preservando execution.type, middlewares e limites basicos.

### Passo 3. Publique tools governadas

Substitua dyn_sql<...> e dyn_api<...> pelos ids aprovados do novo tenant.

### Passo 4. Ajuste local_tools_configuration

No exemplo local, troque queries e endpoints. Nao exponha SQL livre nem URL solta se o caso pede governanca.

### Passo 5. Se houver UI estruturada, feche response_format

Nao deixe o agente inventar payload visual aberto quando a UI espera contrato fixo.

## 11. Diferencas tecnicas mais relevantes entre os exemplos

| Aspecto                       | Template e Food/Retail          | Cockpit PRD                     | AG-UI PDV                |
| ----------------------------- | ------------------------------- | ------------------------------- | ------------------------ |
| Tipo de ferramenta dominante  | dyn_sql e dyn_api mistos        | dyn_api                         | dyn_sql                  |
| selected_supervisor explicito | nao observado nesses trechos    | nao observado nesses trechos    | sim                      |
| response_format estruturado   | nao relevante                   | nao observado                   | sim, fortemente restrito |
| filesystem                    | habilitado com leitura restrita | habilitado com leitura restrita | desabilitado             |
| foco de negocio               | ERP ampliado                    | operacao e fiscal               | analytics e dashboard    |

## 12. Sinais de configuracao ruim nesses cenarios

### Sintoma: o DeepAgent parece poderoso demais para o dominio

Causa provavel: muitos subagentes com tools genericas e sem recorte de negocio.

Acao recomendada: aproximar o desenho do exemplo cockpit ou do PDV, com subdominios mais fechados.

### Sintoma: a UI visual recebe payload inconsistente

Causa provavel: ausencia de response_format fechado ou permissao excessiva de queries.

Acao recomendada: seguir o padrao do subdominio_dashboard_dinamico.

### Sintoma: o agente responde certo, mas perde continuidade entre execucoes

Causa provavel: backend persistente desabilitado, escopo inadequado ou key_prefix mal segmentado.

Acao recomendada: revisar memory, backend e escolher escopo compativel com o processo.

## 13. O que estes exemplos nao devem ser usados para provar

1. Nao provam, por si so, HIL ERP em producao nesses YAMLs especificos.
2. Nao provam shell corporativo ativo nesses recortes de negocio.
3. Nao provam async_subagents em uso efetivo nesses arquivos.

Se o objetivo for documentar esses recursos, o manual canonicamente correto continua sendo o par completo do DeepAgent Supervisor.

## 14. Exemplo guiado de leitura do YAML

### Cenario 1. Quero um DeepAgent para perguntas sobre saude fiscal

Comece por agent-linx-food-cockpit-prd.yaml.

Olhe nesta ordem.

1. id do supervisor.
2. execution.type.
3. especialista_fiscal.
4. dyn_api usados por ele.
5. api_dynamic.endpoints correspondentes.

### Cenario 2. Quero um DeepAgent para dashboard de vendas

Comece por rag-config-pdv-vendas-demo.yaml.

Olhe nesta ordem.

1. selected_supervisor.
2. subdominio_dashboard_dinamico.
3. response_format.
4. tools dyn_sql permitidas.
5. queries correspondentes em local_tools_configuration.sql_dynamic.

### Cenario 3. Quero um DeepAgent mais geral para ERP comercial

Comece por rag-config-linx-food.yaml ou rag-config-linx-retail.yaml.

Olhe nesta ordem.

1. supervisor DeepAgent.
2. sql_analyst.
3. api_integration_specialist.
4. research_assistant.
5. blocos sql_dynamic e api_dynamic.

## 15. Explicacao 101

Se voce abrir os exemplos reais, vai perceber um padrao simples. O DeepAgent nao e configurado como um agente que sabe tudo. Ele e configurado como um coordenador que sabe para quem delegar e quais consultas aprovadas cada especialista pode usar.

Em ERP, isso faz toda a diferenca. O sistema deixa de ser um chat genérico e passa a ser uma camada coordenadora de capacidades aprovadas.

## 16. Checklist tecnico de leitura

- Entendi como identificar um supervisor DeepAgent no YAML.
- Entendi quando selected_supervisor aparece e por que ele ajuda.
- Entendi o papel de memory e backend persistente.
- Entendi como dyn_sql e dyn_api entram nos exemplos.
- Entendi por que o exemplo de AG-UI usa response_format fechado.
- Entendi quais recursos do runtime nao aparecem nesses exemplos especificos.

## 17. Evidencias no codigo

- app/yaml/system/rag-config-modelo.yaml
  - Motivo da leitura: confirmar a espinha dorsal canonicamente repetida pelos exemplos DeepAgent.
  - Comportamento confirmado: execution.type=deepagent, memory top-level, backend persistente em Redis e subagentes com dyn_sql e dyn_api.
- app/yaml/rag-config-linx-food.yaml
  - Motivo da leitura: confirmar caso real de DeepAgent generalista para food service e ERP ampliado.
  - Comportamento confirmado: supervisor DeepAgent com analista SQL, integracao de APIs, RAG e utilitarios.
- app/yaml/rag-config-linx-retail.yaml
  - Motivo da leitura: confirmar a variacao retail do mesmo padrao arquitetural.
  - Comportamento confirmado: espinha semelhante ao caso food, com dyn_sql e dyn_api governados.
- app/yaml/agent-linx-food-cockpit-prd.yaml
  - Motivo da leitura: confirmar exemplo real de cockpit operacional orientado a APIs aprovadas.
  - Comportamento confirmado: especialistas em lojas e fiscal, ambos presos a dyn_api especificas.
- app/yaml/rag-config-pdv-vendas-demo.yaml
  - Motivo da leitura: confirmar DeepAgent para AG-UI em contexto PDV.
  - Comportamento confirmado: selected_supervisor explicito, subdominios de negocio, backend com escopo user e response_format estruturado.
