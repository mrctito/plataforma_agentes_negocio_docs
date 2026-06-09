# Manual técnico e operacional: MCP no sistema

> Escopo desta integração (decisão de arquitetura): o projeto **NÃO hospeda servidor MCP
> próprio**. O MCP existe aqui **somente como cliente**: ele descobre tools em servidores MCP
> externos e as transforma em **tools de agente/workflow**, exatamente simétricas às tools
> nativas do catálogo builtin. O eixo de hospedagem (servidor `qa_system_server`, gateway
> `MCPGatewayManager`, permissões `MCP_SERVERS_*`) **foi removido por completo** (commit Fase A
> `5c075a946`). Qualquer menção a hospedar/gerenciar servidor MCP próprio é legado e não vale
> mais. A única assimetria legítima preservada é o **proxy stdio em `/mcp`** — necessidade
> física do transporte stdio do padrão MCP (ver §5.5 e §6.2).

## 1. O que é esta feature

Tecnicamente, o MCP do projeto é um subsistema de **resolução de configuração** e **carga de
tools** que incorpora tools de servidores MCP externos ao runtime agentic como se fossem
tools nativas. O objetivo operacional não é “hospedar MCP” nem “falar com um servidor MCP” de
forma genérica: é colocar uma tool MCP dentro de um supervisor, agente ou workflow respeitando
o contrato YAML-first, o escopo agentic, a governança do catálogo builtin e a segurança do
boundary HTTP.

Em uma frase: **MCP vira tool**. O caminho oficial é `ToolsFactory -> MCPToolsMerger ->
MCPToolsResolver` (que usa o `MultiServerMCPClient` da lib `langchain-mcp-adapters`). A
distinção entre uma tool nativa e uma tool MCP é feita pelo campo `tool_type=='mcp'` no
catálogo (ver §3.5).

## 2. Que problema ela resolve

O problema técnico resolvido é este: como colocar tools externas dentro de supervisor, agente e
workflow sem criar integração nativa para cada caso e sem abrir um atalho fora do assembly
agentic — e sem o projeto precisar manter o ciclo de vida de um servidor MCP. O subsistema
resolve isso em camadas:

- configuração declarativa (YAML-first);
- normalização por escopo (global, supervisor local, agente/workflow local);
- **descoberta e persistência do catálogo** de tools MCP no mesmo registro builtin das nativas;
- carga das tools MCP em runtime, com cache, e merge ao conjunto local;
- proxy do transporte apenas quando o servidor usa stdio (assimetria física legítima).

## 3. Conceitos necessários para entender

### 3.1 Camadas de configuração

O resolver trabalha com três camadas para agentes e duas para workflows.

- Agente: global_mcp_configuration, local_mcp_configuration do supervisor, local_mcp_configuration do agente.
- Workflow: global_mcp_configuration, local_mcp_configuration do workflow.
- Global puro: apenas global_mcp_configuration.

### 3.2 Merge por servidor

Servidores são mesclados por id. Se o overlay trouxer o mesmo id do base, o merge é profundo. Se trouxer id novo, ele é acrescentado ao conjunto respeitando a ordem.

### 3.3 Proxy stdio

Quando o transporte é stdio, o resolver não devolve o bloco command e args como conexão final do consumo agentic. Ele devolve uma conexão streamable_http apontando para /mcp, com query params que identificam o YAML físico e o escopo.

### 3.4 Catálogo persistido e governado (não mais sintético)

Este é o ponto que mais mudou (Fase B, commits `57b694b4f` e `745e99aa7`). Antes, o sistema
fabricava entradas “sintéticas” de catálogo MCP em runtime, a partir do que o YAML declarava em
`local_mcp_configuration.tools`. **Esse caminho sintético foi cortado por completo** (T10).

Agora o catálogo de tools MCP é **descoberto** dos servidores configurados e **persistido** no
**mesmo registro builtin** das tools nativas (`integrations.builtin_tool_registry`), com
`tool_type='mcp'`. Ele é governado igual às nativas: cada tenant pode ter a tool `active` ou
`disabled`, e a injeção no runtime passa pelo **mesmo boundary com fail-closed** do
`tools_library` (ver §6.1 e §9.4).

A descoberta acontece por **comando explícito** (espelhando o builder nativo), nunca no startup
nem por TTL:

```bash
python -m src.agentic_layer.mcp.mcp_catalog_builder --yaml <path-do-yaml> [--user-email ...]
```

Consequência operacional importante (simetria com as nativas): **declarar uma tool MCP no YAML
sem ter rodado o `mcp_catalog_builder` NÃO resolve a tool** — exatamente como acontece com uma
tool nativa ausente do catálogo builtin. Não existe mais “atalho sintético” que materialize a
tool só porque ela apareceu no YAML.

### 3.5 tool_type = 'mcp' (o terceiro tipo)

No modelo `ToolLibraryEntry` (catálogo builtin), o campo `tool_type` admite três valores:
`direct`, `factory_generated` e `mcp`. O valor `mcp` identifica uma tool consumida de um
servidor MCP externo. Diferença prática: uma tool `mcp` é **isenta de binding local de
execução** — ela não tem `impl` nem `factory_impl`, porque não roda em processo local; é
resolvida em runtime pelo `MCPToolsResolver` (que contata o servidor MCP). É exatamente esse
`tool_type=='mcp'` que a `ToolsFactory` usa para separar tools locais de tools MCP antes de
delegar ao `MCPToolsMerger`.

## 4. Como a feature funciona por dentro

### 4.1 Resolução de configuração

O ponto central é MCPConfigResolver.

- resolve_for_agent monta as camadas global, supervisor local e agente local.
- resolve_for_workflow monta global e workflow local.
- resolve_global usa apenas o bloco global.

Depois do merge, o resolver expande placeholders com expand_placeholders, valida enabled, tool_name_prefix e cache_ttl_s, e materializa o dicionário final de connections.

Há validações estruturais explícitas:

- servidor sem id válido gera erro;
- servidor duplicado no mesmo bloco gera erro;
- transport fora da lista suportada gera erro;
- url ausente em transporte não stdio gera erro;
- command ou args ausentes em stdio geram erro;
- auth.type diferente de bearer ou api_key gera erro.

### 4.2 Tratamento por transporte

Transportes suportados pelo código lido:

- stdio
- sse
- http
- streamable_http
- streamable-http
- websocket

Comportamento por transporte:

- stdio: convertido para streamable_http local com url apontando para /mcp.
- sse: timeout e sse_read_timeout entram como valores numéricos.
- http e streamable_http: timeout e sse_read_timeout entram como timedelta.
- websocket: o resolver aceita o transport, mas o comportamento operacional detalhado desse cliente não foi confirmado além da normalização da conexão.

### 4.3 Requisitos adicionais do caminho stdio

O stdio convertido depende de dois requisitos rígidos:

- _config_metadata precisa existir no YAML.
- _config_metadata.yaml_source precisa apontar para arquivo físico, não memory:// nem inline://.

Se isso falhar, o resolver não consegue gerar a query yaml_config para o proxy e aborta a resolução.

### 4.4 Carga e cache de tools no runtime agentic

O runtime usa MCPToolsResolver.

- Resolve a configuração do escopo.
- Calcula um cache_key baseado em tenant, escopo, connections e tool_name_prefix.
- Se houver cache válido, reutiliza.
- Caso contrário, instancia MultiServerMCPClient e chama get_tools.

Observação operacional importante (mudança da Fase A, T5): se o carregamento de tools falhar
por `ExceptionGroup`, `RuntimeError`, `ValueError`, `TypeError`, `OSError`, `KeyError` ou
`ImportError`, o resolver registra `logger.exception` e **propaga a falha (raise)**. Acabou o
`return []` silencioso. A política é **simétrica às tools nativas**: a falha de carga de uma
tool MCP é **visível** e quebra a execução, exatamente como uma falha ao montar uma tool nativa
(`tools_factory_engine` sempre faz `logger.exception` + `raise`). O motivo é evitar a
degradação invisível em que “MCP sumiu” silenciosamente sem que ninguém perceba.

### 4.5 Merge com tools locais

O merge é centralizado em MCPToolsResolver.merge_tools e consumido por MCPToolsMerger.

- merge_for_agent é usado no fluxo de resolução de tools de agentes.
- merge_for_workflow é usado na resolução de tools de nós de workflow.

Regra de conflito: se uma tool MCP chegar com nome já existente no conjunto base, ela é descartada com warning. O sistema preserva a tool base já presente.

## 5. Divisão em submódulos

### 5.1 MCPConfigResolver

Responsabilidade: transformar YAML MCP em conexões executáveis por escopo.

Recebe: yaml_config.

Entrega: MCPResolvedConfig com enabled, tool_name_prefix, cache_ttl_s e connections.

### 5.2 MCPToolsResolver

Responsabilidade: carregar tools MCP e aplicar cache.

Recebe: MCPResolvedConfig e escopo lógico.

Entrega: lista de BaseTool.

### 5.3 MCPToolsMerger

Responsabilidade: unir tools locais e MCP sem substituir silenciosamente nomes já existentes.

### 5.4 MCPCatalogBuilder

Responsabilidade: **descobrir** as tools dos servidores MCP configurados e **persistir** o
catálogo no registro builtin (`integrations.builtin_tool_registry`), com `tool_type='mcp'`.

Recebe: `yaml_config` (com `global_mcp_configuration`). Roda por **comando explícito**
(`main()` / CLI), nunca no startup nem por TTL — exatamente como o `tools_library_builder`
nativo.

Entrega: resumo com servidores contatados com sucesso, servidores falhos e total de tools
descobertas; e o `sync_summary` da persistência.

Reuso (sem caminho gêmeo do builder nativo): usa `MCPConfigResolver.resolve_global()` para
descobrir os servidores, `MultiServerMCPClient(...).get_tools(server_name=...)` para listar as
tools de cada servidor, e `BuiltinToolCatalogSynchronizer.sync_catalog(...)` para persistir.

Comportamento offline (decisão T7-X): a descoberta é **servidor a servidor**. Se um servidor
MCP estiver indisponível, a falha é **isolada nele**: o builder registra o evento canônico
`mcp.catalog.server.unavailable` (via `logger.exception`), **preserva o último catálogo
persistido daquele servidor** e segue para os demais — **sem abortar a rodada** e sem apagar o
catálogo por indisponibilidade transitória. A reconciliação de obsoletos é **escopada**: só
remove entradas MCP dos servidores que foram contatados com sucesso (predicado
`reconcile_obsolete_predicate`), preservando tools MCP de servidores offline e todas as tools
não-MCP.

### 5.5 McpStdioProxyService

Responsabilidade: expor via HTTP um conjunto de tools provenientes de servidores MCP stdio.

Recebe: Request HTTP com X-API-Key, yaml_config e parâmetros de escopo.

Entrega: listagem e execução de tools como servidor MCP streamable_http.

## 6. Pipeline ou fluxo principal

### 6.1 Fluxo agentic

0. **Pré-requisito (uma vez, por comando):** o `mcp_catalog_builder` descobriu as tools dos
   servidores configurados e as persistiu no catálogo builtin (`tool_type='mcp'`).
1. No boundary de configuração, o `tools_library` (já com nativas e MCP persistidas) é
   **injetado** no runtime pela cadeia oficial, com **fail-closed** se vier preenchido pelo
   cliente (evento `config.configuration_factory.tools_library.injected`).
2. A `ToolsFactory` recebe os ids de tools pedidos e **separa locais de MCP** olhando
   `tool_type=='mcp'` no índice do catálogo (evento `mcp.tools.factory.split`). Id pedido que
   não está no catálogo persistido **não** é tratado como MCP (simetria com as nativas).
3. A `ToolsFactory` resolve as tools **locais** nativamente.
4. O `MCPToolsMerger` chama o `MCPToolsResolver` do mesmo escopo para as tools MCP.
5. O resolver consulta o cache; se necessário, o `MultiServerMCPClient` carrega as tools (se a
   carga falhar, **propaga o erro** — §4.4).
6. O merger anexa as tools MCP sem sobrescrever nomes existentes (conflito de nome = descarte
   da MCP com warning, evento `mcp.tools.merge.conflict_ignored`).
7. O runtime final executa vendo o conjunto combinado.

### 6.2 Fluxo stdio via /mcp

1. O resolver materializa uma conexão local streamable_http apontando para /mcp.
2. A FastAPI monta mcp_http_proxy_app em /mcp.
3. A camada ASGI valida X-API-Key com PermissionKeys.MCP_TOOLS_INVOKE.
4. O serviço do proxy parseia yaml_config e mcp_scope da query string.
5. O serviço resolve o YAML físico via resolve_yaml_configuration.
6. O serviço autentica o request com base no YAML e na permissão da operação.
7. O serviço injeta user_email e correlation_id na user_session.
8. O serviço resolve o MCP daquele escopo e filtra apenas conexões stdio.
9. O builder do catálogo lista tools nos servidores stdio e publica um catálogo em memória.
10. list_tools devolve as tools publicadas; call_tool roteia para o servidor certo e executa a tool real.

## 7. Ordem de execução real

O ponto importante é que o proxy stdio não usa a configuração MCP resolvida no momento do bootstrap da API. Ele reavalia o YAML por request. Isso torna a resolução aderente ao YAML físico e ao contexto de autenticação da chamada, mas também significa que o problema operacional pode estar tanto na configuração estática quanto no request que chegou ao proxy.

## 8. Configurações que mudam o comportamento

### 8.1 Exemplo real de configuração global

O arquivo de modelo em app/yaml/system/rag-config-modelo.yaml confirma este formato:

```yaml
global_mcp_configuration:
  enabled: true
  tool_name_prefix: true
  cache_ttl_s: 300
  servers:
    - id: "agora_mcp"
      enabled: false
      transport: "stdio"
      command: "uvx"
      args:
        - "agora-mcp"
      env:
        FEWSATS_API_KEY: "${FEWSATS_API_KEY}"
    - id: "aws_knowledge_mcp"
      transport: "http"
      url: "https://knowledge-mcp.global.api.aws"
```

### 8.2 Exemplo real de especialização local em agente

O mesmo YAML de modelo confirma que um agente pode declarar tools MCP específicas no seu escopo:

```yaml
local_mcp_configuration:
  enabled: true
  tool_name_prefix: false
  tools:
    - "search_documentation"
    - "read_documentation"
    - "recommend"
    - "list_regions"
    - "get_regional_availability"
```

### 8.3 O que cada campo controla

- enabled: desliga todo o MCP do escopo, mesmo que haja servidores configurados.
- tool_name_prefix: controla nomes como servidor_tool.
- cache_ttl_s: controla o reaproveitamento do catálogo.
- defaults: permite compartilhar headers, transport e outros campos entre servidores.
- servers: define servidores individuais.
- tools no local_mcp_configuration: seleciona, dentro do escopo, quais ids MCP **já descobertos
  e persistidos** devem ser visíveis para aquele agente/workflow. Não fabrica tool: se o id não
  estiver no catálogo persistido, ele não resolve.

### 8.4 Disparar a descoberta do catálogo MCP

Antes de qualquer execução agentic com tool MCP, o catálogo precisa ter sido descoberto e
persistido. O comando é explícito (não roda no startup):

```bash
python -m src.agentic_layer.mcp.mcp_catalog_builder --yaml app/yaml/<seu-yaml>.yaml --user-email voce@exemplo.com
```

O comando lê `global_mcp_configuration`, contata cada servidor, descobre as tools e as persiste
com `tool_type='mcp'`. Servidores offline são preservados; um servidor inacessível não derruba
a rodada (ver §5.4).

## 9. Contratos, entradas e saídas

### 9.1 Contrato do resolver

Entrada: yaml_config completo.

Saída: MCPResolvedConfig.

Campos de saída confirmados:

- enabled
- tool_name_prefix
- cache_ttl_s
- connections

### 9.2 Contrato do proxy HTTP

Query params confirmados:

- yaml_config
- mcp_scope
- supervisor_id quando scope=agent
- agent_id quando scope=agent
- workflow_id quando scope=workflow

Headers relevantes:

- X-API-Key (obrigatório no boundary ASGI).
- x-correlation-id **obrigatório**. O proxy **não gera** mais um correlation_id local: se o
  header estiver ausente ou inválido, o request **falha** com `McpHttpProxyError` (ver §12.2 e
  `_resolve_base_correlation_id`). O proxy preserva o `correlation_id` oficial recebido — ele
  nunca inventa identificador.

### 9.3 Contrato de permissões

Permissões confirmadas no catálogo de autorização (Fase A removeu as permissões de hospedagem):

- MCP_TOOLS_LIST (`mcp.tools.list`)
- MCP_TOOLS_INVOKE (`mcp.tools.invoke`)

As permissões `MCP_SERVERS_START`, `MCP_SERVERS_STOP` e `MCP_SERVERS_HEALTH` **não existem
mais**: eram do eixo de hospedagem (gateway), removido por completo no commit Fase A
`5c075a946`. O caminho canônico do proxy stdio usa `MCP_TOOLS_LIST` na listagem e
`MCP_TOOLS_INVOKE` na execução.

### 9.4 Contrato do catálogo persistido (builtin)

- Tabela: `integrations.builtin_tool_registry` (mesma das tools nativas).
- Discriminante: `tool_type='mcp'`.
- Governança: por tenant, `status` `active`/`disabled`.
- Origem: `created_by='mcp_catalog_builder'`, `discovered_from='mcp_server::<server>'`,
  `metadata.mcp_server=<server>` (usado na reconciliação escopada).
- Binding: tools MCP **não têm** `impl`/`factory_impl` (isentas de execução local).
- Injeção: pelo mesmo boundary com **fail-closed** do `tools_library`. Se o cliente enviar
  `tools_library` preenchida no YAML do endpoint, o sistema falha fechado (regra global do
  catálogo builtin).

## 10. O que acontece em caso de sucesso

### 10.1 Runtime agentic

O agente ou workflow recebe um conjunto de BaseTool já enriquecido com tools MCP. Se houver cache válido, esse conjunto é reaproveitado. Se não houver, ele é carregado na hora.

### 10.2 Proxy stdio

O cliente HTTP consegue listar tools e invocar a tool correta. O proxy registra logs de listagem e execução com correlation_id e server_id.

## 11. O que acontece em caso de erro

### 11.1 Erros de configuração

Erros como transport inválido, missing url, missing command, missing args, cache_ttl_s inválido, tool_name_prefix inválido, yaml_source ausente ou escopo incompleto geram exceção explícita.

### 11.2 Erros de autenticação e permissão

No proxy, ausência ou invalidez de X-API-Key no boundary HTTP gera resposta JSON com status apropriado. Depois disso, o serviço ainda autentica o request com o YAML resolvido e a permissão da operação.

### 11.3 Erros de catálogo ou roteamento

- Tool desconhecida no proxy gera McpHttpProxyError com log de erro.
- Servidor ausente no roteamento gera erro.
- Ausência de conexões stdio após a filtragem gera erro explícito.

### 11.4 Erros de comunicação com stdio

O cliente stdio usa retry com tenacity.

- stop_after_attempt(5)
- wait_exponential com multiplicador 1, mínimo 1 e máximo 10
- retry em OSError, RuntimeError e TimeoutError

Isso vale tanto para list_tools quanto para call_tool.

### 11.5 Erros na descoberta do catálogo (mcp_catalog_builder)

- Servidor MCP indisponível durante a descoberta: **não** aborta a rodada. O builder registra
  `mcp.catalog.server.unavailable`, preserva o catálogo persistido daquele servidor e segue
  (decisão T7-X). O resumo final lista o servidor em `failed_servers`.
- Nenhum servidor habilitado no `global_mcp_configuration`: nada a sincronizar; o builder
  registra `mcp.catalog.sync.skipped` (reason `no_enabled_servers`) e encerra sem erro.
- `user_email` ausente (multi-tenant): `SystemExit` no `main()` — informe `--user-email` ou
  `user_session.user_email` no YAML.

### 11.6 Carga de tools MCP no runtime falha (simetria com nativas)

Diferente do comportamento antigo: a falha de carga **não** vira lista vazia silenciosa. O
`MCPToolsResolver` faz `logger.exception` e **propaga** (`RuntimeError`), quebrando a execução —
igual a uma tool nativa que não monta (ver §4.4). Evento `mcp.tools.load.failed`.

## 12. Observabilidade e diagnóstico

### 12.1 Logs relevantes

Os logs do slice MCP usam o builder canônico `build_mcp_log_context(...)`
(`src/agentic_layer/mcp/log_vocabulary.py`) e `event_name`s do vocabulário `mcp.*`. Os
principais eventos confirmados no código:

- `config.configuration_factory.tools_library.injected` — injeção do catálogo (nativas + MCP)
  no boundary de configuração;
- `mcp.tools.factory.split` — `ToolsFactory` separou ids locais de ids MCP (por
  `tool_type=='mcp'`);
- `mcp.catalog.sync.started` / `mcp.catalog.sync.skipped` / `mcp.catalog.sync.completed` —
  descoberta+persistência do catálogo MCP;
- `mcp.catalog.server.discovered` — tools descobertas em um servidor;
- `mcp.catalog.server.unavailable` — servidor offline na descoberta (catálogo preservado);
- `mcp.tools.load.started` / `mcp.tools.load.completed` / `mcp.tools.load.failed` — carga de
  tools MCP no runtime (a falha **propaga**);
- `mcp.tools.cache.hit` / `mcp.tools.cache.updated` — cache de tools MCP;
- `mcp.tools.selection.applied` / `mcp.tools.selection.failed` — seleção/filtragem de tools
  solicitadas;
- `mcp.tools.merge.conflict_ignored` — tool MCP descartada por conflito de nome com tool local;
- eventos do proxy stdio: listagem, chamada e execução por servidor, e erros de tool
  desconhecida, servidor ausente e ausência de conexões stdio.

### 12.2 Correlation id

O proxy stdio **exige** o header `x-correlation-id` (`_resolve_base_correlation_id`): se o
header estiver ausente ou inválido, o request **falha** com `McpHttpProxyError` — o proxy
**não** gera identificador local. A partir do `correlation_id` oficial recebido, ele recompõe o
`correlation_id` final com base no `user_email` da sessão (`compose_correlation_id`) e segue
logando nesse id. Em outras palavras: o `correlation_id` nasce no chamador (o
`MCPToolsResolver`/runtime agentic, que tem o id oficial), é enviado no header, e o proxy apenas
preserva e propaga — nunca inventa. Isso respeita o contrato canônico de `correlation_id`
(nascido uma única vez no boundary do processo).

### 12.3 Onde começar a investigar

Se a tool não aparece no runtime agentic:

- **verificar primeiro se o catálogo MCP foi descoberto/persistido** (rodou o
  `mcp_catalog_builder`?): id MCP só resolve se estiver no catálogo builtin com `tool_type='mcp'`
  e `status='active'` para o tenant. Tool só declarada no YAML, sem descoberta, **não resolve**;
- verificar `local_mcp_configuration.tools` no escopo (seleção dentro do já descoberto);
- verificar conflito de nome com tool local (evento `mcp.tools.merge.conflict_ignored`);
- verificar logs de falha do MCPToolsResolver (`mcp.tools.load.failed` — agora a falha
  **propaga**, não some).

Se o problema é no /mcp:

- verificar query params yaml_config e mcp_scope;
- verificar X-API-Key;
- verificar se o YAML físico existe e contém _config_metadata.yaml_source válido;
- verificar se há servidores stdio habilitados naquele escopo.

## 13. Limites e pegadinhas

### 13.1 O proxy stdio não é cacheado por escopo

No código lido, o catálogo do proxy é cacheado apenas por tenant_key. Isso pode ser suficiente quando o tenant usa o mesmo conjunto stdio em todos os escopos, mas é uma limitação importante se diferentes escopos do mesmo tenant publicarem catálogos distintos.

### 13.2 O projeto não hospeda MCP (eixo de gateway removido)

O `MCPGatewayManager` e o servidor `qa_system_server` **não existem mais** (Fase A, commit
`5c075a946`). Não há, no runtime, nenhum caminho em que o projeto suba/gerencie um servidor MCP
próprio. O único componente que expõe HTTP é o **proxy stdio em `/mcp`**, e ele existe apenas
para dar transporte HTTP a servidores MCP **stdio** externos — não para hospedar tools do
projeto como MCP. Esta é a única assimetria legítima preservada (ver §5.5 e §6.2).

### 13.3 Falha de carga de tool MCP é visível (não some mais)

Antes, algumas falhas de carga viravam lista vazia e davam a impressão de que “MCP sumiu”. Isso
**acabou** (Fase A, T5): a falha de carga de tool MCP no runtime **propaga** e quebra a
execução, igual a uma tool nativa que não monta. A degradação invisível foi eliminada.

### 13.4 Catálogo MCP precisa ser descoberto antes de usar

Simetria pedida e legítima: declarar uma tool MCP no YAML **não basta**. Sem rodar o
`mcp_catalog_builder` para descobrir e persistir o catálogo, a tool não resolve — exatamente
como uma tool nativa ausente do catálogo builtin. Não existe mais materialização “sintética” em
runtime que mascarava esse pré-requisito (Fase B, T10).

## 14. Troubleshooting

### Sintoma: MCP habilitado, mas erro dizendo que não há servidores

Confirmação: enabled=true com servers ausente ou vazio no escopo efetivo.

Ação: revisar o merge entre global_mcp_configuration e local_mcp_configuration.

### Sintoma: tool MCP declarada no agente não resolve

Confirmação: a tool está em `agent.tools` (e em `local_mcp_configuration.tools`), mas a
`ToolsFactory` não a trata como MCP e a execução não enxerga a tool.

Causa mais comum: o catálogo MCP **não foi descoberto/persistido**. Como não existe mais
catálogo sintético em runtime, um id que não está no catálogo builtin com `tool_type='mcp'` não
é classificado como MCP.

Ação: rodar a descoberta — `python -m src.agentic_layer.mcp.mcp_catalog_builder --yaml <path>
--user-email <email>` — e confirmar no banco que a tool foi gravada com `tool_type='mcp'` e
`status='active'` para o tenant. Só depois disso ela resolve.

### Sintoma: stdio configurado com YAML inline falha

Confirmação: yaml_source ausente, memory:// ou inline://.

Ação: usar YAML físico para o caminho do proxy stdio.

### Sintoma: listagem do proxy funciona, mas chamada da tool falha

Confirmação: a tool aparece em routing, mas call_tool retorna erro do servidor stdio.

Ação: revisar command, args, env e disponibilidade do processo publicado pelo servidor MCP.

## 15. Diagramas

### 15.1 Sequência do caminho stdio

![15.1 Sequência do caminho stdio](../assets/diagrams/docs-readme-tecnico-mcp-integracao-uso-sistema-diagrama-01.svg)

O diagrama mostra que o proxy reabre o contexto do YAML e da autenticação a cada request. Isso é importante para diagnóstico, porque um mesmo problema pode nascer da query, da segurança, do YAML ou do processo stdio.

## 16. Como colocar para funcionar

### 16.1 Pré-requisitos confirmados

- A API FastAPI precisa estar em execução.
- O YAML precisa existir em arquivo físico quando houver uso de stdio.
- O YAML precisa trazer global_mcp_configuration ou local_mcp_configuration compatíveis.
- **O catálogo MCP precisa ter sido descoberto/persistido** com o `mcp_catalog_builder` antes da
  execução agentic (sem isso, a tool MCP não resolve — §3.4 e §13.4).
- O cliente precisa enviar X-API-Key com permissão apropriada (e, no proxy stdio,
  `x-correlation-id` obrigatório).

### 16.2 Passo a passo mínimo para um caso remoto HTTP

Primeiro, descobrir e persistir o catálogo a partir do YAML (uma vez, por comando):

```bash
python -m src.agentic_layer.mcp.mcp_catalog_builder --yaml app/yaml/<seu-yaml>.yaml --user-email voce@exemplo.com
```

Depois, o YAML do agente:

```yaml
global_mcp_configuration:
  enabled: true
  tool_name_prefix: true
  cache_ttl_s: 300
  servers:
    - id: "aws_knowledge_mcp"
      transport: "http"
      url: "https://knowledge-mcp.global.api.aws"

multi_agents:
  - id: "sup"
    enabled: true
    agents:
      - id: "aws_documentation_researcher"
        tools:
          - "search_documentation"
          - "read_documentation"
        local_mcp_configuration:
          enabled: true
          tool_name_prefix: false
          tools:
            - "search_documentation"
            - "read_documentation"
```

### 16.3 Passo a passo mínimo para um caso stdio

```yaml
global_mcp_configuration:
  enabled: true
  tool_name_prefix: true
  servers:
    - id: "agora_mcp"
      transport: "stdio"
      command: "uvx"
      args:
        - "agora-mcp"

multi_agents:
  - id: "sup"
    enabled: true
    agents:
      - id: "agora_shopping_assistant"
        local_mcp_configuration:
          enabled: true
          tool_name_prefix: true
          tools:
            - "agora_mcp_agora_search"
```

O que esperar: o resolver converterá o stdio para uma URL local /mcp com parâmetros de escopo; o proxy assumirá a listagem e a execução real.

### 16.4 Como validar que funcionou

- Verificar logs de carregamento de tools MCP no runtime agentic.
- Verificar logs do proxy stdio ao listar ou chamar tool.
- Confirmar que a tool aparece no conjunto final do agente ou workflow sem warning de conflito de nome.

## 17. Exemplos práticos guiados

### 17.1 Exemplo real de proxy gerado para agente

Os testes unitários confirmam que um servidor stdio pode virar uma URL como:

```text
http://localhost:9000/mcp?yaml_config=app%2Fyaml%2Ftest-config.yaml&mcp_scope=agent&supervisor_id=sup&agent_id=agent
```

### 17.2 Exemplo real de proxy gerado para workflow

Os testes confirmam também o formato:

```text
http://127.0.0.1:8001/mcp?yaml_config=app%2Fyaml%2Ftest-config.yaml&mcp_scope=workflow&workflow_id=wf
```

## 18. Explicação 101

Do ponto de vista técnico simples, o sistema faz três perguntas antes de usar MCP:

1. Quais servidores MCP existem para este YAML?
2. Neste escopo específico, quais tools desse universo devem ser vistas?
3. O acesso será direto por transporte remoto ou indireto por /mcp no caso stdio?

Quando essas três respostas estão coerentes, o agente usa a tool como se ela fosse parte do seu conjunto normal de ferramentas.

## 19. Checklist de entendimento

- Entendi o papel de MCPConfigResolver.
- Entendi a diferença entre carga agentic e proxy stdio.
- Entendi o papel de local_mcp_configuration.tools no catálogo efetivo.
- Entendi o comportamento de cache no runtime agentic e no proxy.
- Entendi como o /mcp autentica e resolve o YAML.
- Entendi os erros duros e as degradações silenciosas do desenho atual.

## 20. Evidências no código

- src/agentic_layer/mcp/mcp_config_resolver.py
  - Símbolos relevantes: resolve_for_agent, resolve_for_workflow, _build_local_http_proxy_connection.
  - Comportamento confirmado: merge por escopo, transports suportados e geração da URL /mcp para stdio.
- src/agentic_layer/mcp/mcp_tools_resolver.py
  - Símbolos relevantes: _resolve_tools, _load_tools, merge_tools.
  - Comportamento confirmado: cache por tenant e escopo, carga via MultiServerMCPClient e descarte de conflito por nome.
- src/agentic_layer/supervisor/mcp_tools_merger.py
  - Símbolos relevantes: merge_for_agent, merge_for_workflow.
  - Comportamento confirmado: ponto de incorporação das tools MCP ao runtime.
- src/agentic_layer/supervisor/tools_factory.py
  - Símbolos relevantes: resolve_workflow_node_tools, resolve_agent_tools_with_context.
  - Comportamento confirmado: tools locais são resolvidas primeiro e depois enriquecidas com MCP.
- src/agentic_layer/mcp/http_proxy/mcp_stdio_http_proxy.py
  - Símbolos relevantes: McpStdioProxyService, McpStdioClient, McpStdioToolCatalogBuilder.
  - Comportamento confirmado: proxy stdio com autenticação, retry, catálogo em memória e roteamento por tool.
- src/api/routers/mcp_http_proxy_router.py
  - Símbolos relevantes: McpHttpProxyApp, McpHttpProxyServer.
  - Comportamento confirmado: servidor MCP HTTP streamable e validação de X-API-Key no boundary ASGI.
- src/api/service_api.py
  - Símbolo relevante: mount de /mcp.
  - Comportamento confirmado: proxy MCP faz parte do runtime HTTP principal.
- src/config/agentic_assembly/tool_resolver.py
  - Símbolo relevante: _append_local_mcp_tools.
  - Comportamento confirmado: local_mcp_configuration.tools entra no catálogo efetivo do assembly.
- tests/unit/test_02-04-51_mcp_resolvers.py
  - Motivo da leitura: confirmar exemplos executáveis de merge, cache e geração da URL do proxy.
  - Comportamento confirmado: casos de agente e workflow para stdio convertido em /mcp.
