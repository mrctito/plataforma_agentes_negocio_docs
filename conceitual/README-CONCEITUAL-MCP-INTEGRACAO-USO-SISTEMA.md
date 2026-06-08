# Manual conceitual, executivo, comercial e estratégico: MCP no sistema

## 1. O que é esta feature

O MCP neste projeto é a capacidade de conectar ferramentas externas ao runtime agentic sem embutir cada integração manualmente dentro do código do supervisor, do workflow ou do agente especialista. Na prática, o YAML declara servidores MCP, o runtime resolve essas conexões por escopo e as tools publicadas por esses servidores passam a conviver com as tools locais do catálogo da plataforma.

Isso significa que MCP aqui não é apenas “um protocolo externo”. Ele virou uma peça da plataforma YAML-first. A configuração pode nascer no bloco global do YAML, ser refinada no supervisor, no agente ou no workflow, e então influenciar diretamente quais tools o runtime enxerga, em qual escopo elas aparecem, como são nomeadas e como são autenticadas.

O código lido mostra duas formas de uso dentro do sistema:

- Consumo de tools MCP pelo runtime agentic, via resolução e merge com as tools locais.
- Exposição HTTP local de servidores MCP em modo stdio, por meio do endpoint montado em /mcp.

## 2. Que problema ela resolve

Sem MCP, cada integração externa precisaria ser transformada em tool nativa da plataforma, com código próprio, wiring próprio, autenticação própria e manutenção própria. Isso aumenta acoplamento, amplia o catálogo interno de forma artificial e torna mais caro experimentar novos provedores ou novos conjuntos de ferramentas.

O MCP reduz esse custo separando dois problemas diferentes:

- A plataforma continua governando quem pode usar qual ferramenta, em que escopo e com qual YAML.
- O servidor MCP continua responsável por publicar as capacidades especializadas daquele domínio.

Na prática, isso reduz o tempo para incorporar novos ecossistemas de tools, evita duplicação de wrappers internos e permite compor agentes com capacidades externas sem transformar todo novo caso de uso em desenvolvimento de tool nativa.

## 3. Visão executiva

Executivamente, MCP importa porque aumenta a velocidade de expansão funcional da plataforma sem exigir que cada nova integração vire um projeto de engenharia dedicado. O ganho não é só técnico. O negócio passa a conseguir plugar capacidades novas, como busca documental especializada ou catálogos de produtos, sem crescer o núcleo da plataforma na mesma proporção.

O desenho atual também reduz risco operacional em três pontos importantes:

- O uso é governado por YAML e permissões, então a integração não nasce “solta” no runtime.
- O catálogo efetivo do runtime e o catálogo efetivo validado pela semântica do assembly usam a mesma lista declarada em local_mcp_configuration.tools.
- Quando o escopo declara uma tool MCP específica e ela não materializa no carregamento, o runtime falha fechado em vez de mascarar a ausência.

Isso apoia governança porque permite dizer com clareza quais tools externas estão habilitadas globalmente, quais foram expostas apenas a um workflow específico e quais foram autorizadas para um agente específico.

## 4. Visão comercial

Comercialmente, MCP pode ser explicado como a camada que permite conectar a plataforma a ecossistemas externos de ferramentas sem reescrever tudo como integração proprietária. Para o cliente, isso se traduz em menor tempo para disponibilizar novas capacidades no agente e menor dependência de uma fila longa de desenvolvimento interno.

Um exemplo suportado pelo código é o uso de um MCP de documentação AWS para um agente pesquisador técnico e o uso de um MCP de catálogo de compras para um agente de busca de produtos. A mensagem comercial viável é esta: a plataforma consegue combinar ferramentas nativas e ferramentas publicadas por servidores MCP dentro do mesmo runtime governado.

O que não deve ser prometido com base no código lido:

- Não há evidência de descoberta automática universal de qualquer servidor MCP sem configuração YAML.
- Não há evidência de um painel administrativo completo para cadastrar e operar todo o ciclo de vida de servidores MCP no caminho canônico.
- Não há evidência de que o gateway manager legado seja a porta operacional principal do sistema atual.

## 5. Visão estratégica

Estrategicamente, MCP fortalece a direção YAML-first e a arquitetura agentic por três razões.

Primeiro, ele desacopla capacidade funcional de implementação nativa. A plataforma deixa de depender apenas do catálogo builtin para crescer.

Segundo, ele preserva governança. As tools MCP não entram no runtime como um atalho lateral. Elas passam por resolução de configuração por escopo, por validação semântica do catálogo efetivo, por seleção explícita dos ids declarados no YAML e por merge com o conjunto local.

Terceiro, ele prepara a plataforma para cenários em que diferentes agentes ou workflows precisem conversar com ecossistemas distintos sem que o core precise conhecer os detalhes de cada servidor. Isso é especialmente importante para uma arquitetura de agentes especialistas, onde o valor está em montar composições diferentes a partir do mesmo trilho operacional.

## 6. Conceitos necessários para entender

### 6.1 MCP

MCP, aqui, é o contrato pelo qual um servidor externo publica ferramentas e o cliente as consome como tools. O ponto importante no projeto é que MCP não substitui a governança da plataforma. Ele fornece a fonte externa de capabilities; a plataforma decide onde essas capabilities entram.

### 6.2 Escopo global e escopo local

O sistema trabalha com camadas de configuração. O bloco global define um baseline para todo o YAML. Depois disso, supervisor, agente e workflow podem sobrescrever partes desse baseline com local_mcp_configuration. Isso permite habilitar um servidor para todo o tenant, mas só expor certas tools a um agente específico, por exemplo.

### 6.3 Tool name prefix

Quando tool_name_prefix está habilitado, o nome publicado para a tool passa a carregar o id do servidor. Isso existe para evitar conflito entre tools com o mesmo nome vindas de servidores diferentes ou entre uma tool MCP e uma tool local.

### 6.4 stdio versus HTTP

No MCP deste projeto, stdio e HTTP não são apenas transportes equivalentes. Quando o servidor é declarado como stdio, o runtime agentic não fala diretamente com esse processo no ponto de consumo das tools. O código converte esse servidor em uma URL streamable_http local apontando para /mcp, e esse proxy HTTP é quem lida com o processo stdio de fato.

### 6.5 Catálogo efetivo

O catálogo efetivo é o conjunto de tools que realmente existe naquele escopo depois da composição entre catálogo builtin, tools locais do YAML e tools MCP declaradas. Essa noção importa porque o validator semântico e a resolução de runtime precisam concordar sobre quais ids existem de verdade naquele contexto.

## 7. Como a feature funciona por dentro

O fluxo começa no YAML. O bloco global_mcp_configuration define o baseline de habilitação, prefixo de nomes, TTL de cache, defaults compartilhados e a lista de servidores. Depois, local_mcp_configuration pode aparecer em multi_agents, em agentes individuais e em workflows.

Quando o runtime precisa materializar tools para um agente ou workflow, o MCPConfigResolver lê essas camadas, faz merge em ordem, expande placeholders sensíveis e também preserva a lista declarada de ids MCP do escopo. A partir daí, o MCPToolsResolver cria o cliente MultiServerMCPClient, carrega as tools brutas, guarda o resultado em cache por tenant e escopo e só depois aplica o filtro pelos ids explicitamente pedidos.

Por fim, o MCPToolsMerger une esse conjunto MCP às tools locais já resolvidas pelo runtime. Se o escopo não pediu nenhuma tool MCP, ele preserva apenas o conjunto local. Se houver colisão de nome, a tool MCP é ignorada com warning, preservando a tool que já existia no conjunto base.

No caso específico de servidores stdio, existe um segundo fluxo. Em vez de devolver command e args diretamente ao runtime de tools, o resolver gera uma URL local para /mcp com parâmetros de query que identificam o YAML físico e o escopo. O proxy HTTP, montado pela FastAPI, relê esse YAML, autentica a chamada, recompõe a sessão do usuário, sobe clientes stdio para listar ou chamar tools e expõe isso como servidor MCP HTTP.

## 8. Divisão em etapas ou submódulos

### 8.1 Declaração e herança de configuração

É a etapa em que o YAML diz o que existe. O problema resolvido aqui é evitar repetição e permitir override local sem duplicar toda a configuração global. O merge acontece por camadas, preservando a ordem e mesclando defaults e servidores por id.

### 8.2 Normalização para conexões executáveis

Depois da herança, o sistema precisa transformar a configuração em algo que um cliente MCP consiga usar. É aqui que entram validações de transport, URL, command, args, headers, auth e timeouts. Se a configuração estiver inconsistente, a resolução falha cedo.

### 8.3 Incorporação ao catálogo efetivo

O catálogo agentic precisa reconhecer que certas tools MCP existem naquele escopo. Por isso o resolver de catálogo adiciona placeholders de tools MCP locais quando elas são declaradas em local_mcp_configuration.tools. O runtime usa a mesma lista declarada para compor tools_library e para separar ids locais de ids MCP antes da materialização. Essa etapa existe para manter coerência entre validação semântica e runtime.

### 8.4 Carregamento e cache das tools

Depois da resolução, o runtime precisa conversar com os servidores MCP e trazer as tools. Esse passo é potencialmente caro, então o sistema usa cache. No runtime agentic, o cache é calculado com tenant, escopo, conexões e policy de prefixo, mas o filtro por ids declarados acontece depois da leitura do cache. Isso evita recalcular o catálogo bruto só porque um agente ou workflow pediu subconjuntos diferentes da mesma conexão. No proxy stdio HTTP, o catálogo listado é mantido em memória para reaproveitamento.

### 8.5 Exposição HTTP do stdio

Essa etapa existe porque um processo stdio local não é naturalmente um endpoint HTTP consumível por todo o restante do ecossistema. O proxy traduz esse servidor em uma superfície streamable_http protegida por permissão e autenticada com base no YAML e na chave de API da requisição.

## 9. Pipeline ou fluxo principal

1. O YAML declara global_mcp_configuration e, quando necessário, local_mcp_configuration em supervisor, agente ou workflow.
2. O assembly agentic inclui no catálogo efetivo os ids de tools MCP locais declarados no escopo.
3. O runtime compõe tools_library do escopo, incluindo entradas sintéticas MCP para os ids declarados.
4. O ToolsFactory separa ids locais de ids MCP usando o tipo da tool no catálogo efetivo.
5. O runtime resolve a configuração MCP do mesmo escopo, preservando os ids MCP selecionados no YAML.
6. O sistema carrega as tools MCP, aplica cache ao catálogo bruto e então filtra apenas os ids explicitamente pedidos.
7. O merger junta tools locais e MCP filtradas, descartando nomes conflitantes do lado MCP.
8. O agente ou workflow executa já vendo esse conjunto final como seu catálogo utilizável.

No caso de stdio, entre os passos 4 e 5 existe uma conversão intermediária: a conexão vira uma URL local para /mcp, e a listagem ou a execução real da tool passa pelo proxy HTTP.

## 10. Decisões técnicas e trade-offs

### 10.1 MCP é complementar às tools locais, não substituto

Ganho: preserva o catálogo interno como núcleo governado e usa MCP para expansão.

Custo: o operador precisa entender dois mundos, tools locais e tools MCP.

### 10.2 Merge por escopo em vez de um catálogo MCP único global

Ganho: reduz exposição indevida de tools e melhora especialização por agente ou workflow, porque o próprio runtime só materializa as tools MCP pedidas naquele escopo.

Custo: aumenta a complexidade de raciocínio sobre qual tool aparece em qual escopo e exige manter a lista declarada de ids sincronizada com o que o servidor realmente publica.

### 10.3 stdio convertido em proxy HTTP local

Ganho: unifica o consumo e cria uma fronteira HTTP autenticável para um backend que originalmente seria apenas processo local.

Custo: adiciona uma camada intermediária e um segundo cache específico do proxy.

### 10.4 Conflito de nome resolve preservando a tool base

Ganho: evita substituir silenciosamente uma tool que o runtime já conhecia.

Custo: a tool MCP em conflito some do conjunto final e o operador precisa observar os warnings para perceber isso.

## 11. Configurações que mudam o comportamento

As configurações mais importantes confirmadas no código são estas.

- enabled: liga ou desliga MCP no escopo.
- tool_name_prefix: controla se o nome publicado recebe prefixo do servidor.
- cache_ttl_s: define por quanto tempo o catálogo MCP pode ser reutilizado.
- defaults: baseline compartilhado entre servidores do mesmo bloco.
- servers: lista de servidores MCP declarados.
- transport: aceita stdio, sse, http, streamable_http, streamable-http e websocket.
- auth.type e auth.token: permitem autenticação bearer ou api_key.
- headers: adiciona cabeçalhos estáticos.
- timeout_s e sse_read_timeout_s: ajustam timeouts conforme o transporte.
- command e args: obrigatórios para stdio.
- url: obrigatória para transportes não stdio.
- tools em local_mcp_configuration: lista os ids de tools MCP que o escopo quer expor no catálogo efetivo.

## 12. Contratos, entradas e saídas

No uso agentic, a entrada principal é o YAML com os blocos MCP. A saída prática é uma lista de BaseTool já materializada e incorporada ao conjunto final de tools do agente ou workflow, respeitando exatamente os ids MCP declarados naquele escopo.

No uso via proxy, a entrada é a chamada HTTP para /mcp com query params que apontam para um YAML físico e definem o escopo. O proxy responde como servidor MCP HTTP streamable, expondo list_tools e call_tool.

Há invariantes importantes no código:

- stdio exige YAML com _config_metadata.yaml_source apontando para arquivo físico.
- MCP habilitado sem servidores configurados gera erro.
- tool_name_prefix precisa ser booleano.
- cache_ttl_s precisa ser inteiro positivo.
- agent scope exige supervisor_id e agent_id.
- workflow scope exige workflow_id.

## 13. O que acontece em caso de sucesso

Quando tudo está correto, o YAML define os servidores, o runtime resolve a camada efetiva do escopo, as tools MCP declaradas para aquele escopo são carregadas e passam a aparecer junto das tools locais. No caso de stdio, o proxy local também consegue listar e chamar as tools depois de autenticar a requisição e reconstituir a sessão do usuário.

Para o usuário final, o sucesso aparece como uma capacidade nova disponível no agente ou workflow sem que essa capacidade precise ter sido reimplementada como tool nativa.

## 14. O que acontece em caso de erro

Os erros confirmados no código mostram um comportamento misto de falha dura e degradação silenciosa, com uma exceção importante para seleção explícita de tools MCP.

- Erros estruturais de configuração, como transport inválido, ausência de url, ausência de command ou args e yaml_source inválido, geram exceção explícita.
- Se MCP estiver habilitado mas sem servidores, a resolução falha explicitamente.
- No carregamento de tools para o runtime agentic, se o cliente MCP falhar ao buscar tools e o escopo não tiver pedido ids específicos, o resolver registra exception e devolve lista vazia.
- Se o escopo declarou ids MCP específicos e essas tools não materializam, o runtime registra falha de seleção e levanta erro explícito.
- No proxy HTTP, tool desconhecida, servidor ausente ou ausência de conexões stdio geram erro explícito do proxy.

Esse desenho significa que problemas de contrato são tratados de forma dura. No carregamento agentic, ainda existe degradação controlada quando o escopo não pediu nenhuma tool MCP específica, mas a ausência de uma tool explicitamente declarada é tratada como erro real.

## 15. Observabilidade e diagnóstico

O código registra eventos importantes em três momentos:

- carregamento de tools MCP;
- aplicação da seleção explícita de ids MCP e falha de seleção quando faltar tool declarada;
- reaproveitamento e atualização de cache;
- listagem e execução de tools no proxy stdio.

O correlation_id é propagado tanto no carregamento das tools quanto no proxy HTTP. No caso do proxy, o serviço também recompõe a sessão do usuário e compõe um correlation_id final associado ao user_email.

Para investigar MCP, a pergunta certa é sempre: o problema está na declaração YAML, na resolução do escopo, na autenticação do proxy, na comunicação com o servidor MCP ou no merge final com as tools locais?

## 16. Impacto técnico

Tecnicamente, MCP reduz a necessidade de wrappers nativos e reforça uma arquitetura onde a plataforma orquestra e governa, enquanto a capability especializada pode morar fora do núcleo. Também reforça a separação entre catálogo efetivo, configuração declarativa e transporte executável.

## 17. Impacto executivo

Executivamente, a feature reduz tempo para disponibilizar novas capacidades, melhora flexibilidade do produto e reduz acoplamento entre roadmap funcional e backlog de tool nativa.

## 18. Impacto comercial

Comercialmente, permite demonstrar especialização rápida por domínio. O mesmo core pode ganhar um agente com documentação AWS, um agente com catálogo de compras ou um workflow com ferramentas específicas sem exigir um pacote inteiro de desenvolvimento de integração proprietária.

## 19. Impacto estratégico

Estrategicamente, o MCP aproxima a plataforma de um papel de orquestrador governado de capabilities. Isso é coerente com uma arquitetura de agentes especialistas, workflows configuráveis e expansão contínua por YAML.

## 20. Exemplos práticos guiados

### 20.1 Exemplo feliz: agente com MCP de documentação AWS

O YAML global habilita um servidor HTTP remoto e o agente declara em local_mcp_configuration quais tools quer expor. O runtime resolve as conexões, carrega essas tools e o agente passa a poder chamar search_documentation, read_documentation e outras tools declaradas naquele escopo.

### 20.2 Exemplo feliz: agente com stdio local via proxy

O YAML declara um servidor stdio com command e args. O resolver converte essa entrada em uma URL local para /mcp. O consumo final ocorre via proxy HTTP, e o operador não precisa lidar diretamente com processo stdio na camada agentic.

### 20.3 Exemplo de limitação real

No código lido, o cache do catálogo do proxy stdio é indexado apenas por tenant_key. Se o mesmo tenant usar escopos stdio diferentes, o reaproveitamento pode não refletir a granularidade do escopo. Isso é uma observação importante de operação e revisão.

## 21. Explicação 101

Pense no MCP como uma tomada padronizada para ferramentas externas. A plataforma continua sendo a casa: ela decide em qual cômodo aquela tomada existe, quem pode usá-la e com qual chave. O servidor MCP é o aparelho especializado conectado nessa tomada. O valor da feature está em permitir trocar ou acrescentar aparelhos sem reconstruir a casa.

## 22. Limites e pegadinhas

- Declarar um servidor MCP não basta para o catálogo efetivo do escopo reconhecer automaticamente qualquer nome de tool que você quiser usar. No código lido, a lista de ids em local_mcp_configuration.tools é parte importante dessa governança e também dirige a seleção real no runtime.
- Em conflito de nome, a tool MCP não substitui a tool base.
- No caminho agentic, falha de carregamento de tools MCP pode resultar em lista vazia apenas quando o escopo não pediu ids MCP específicos. Se pediu, a ausência vira erro explícito.
- O caminho canônico encontrado para stdio passa por /mcp, não pelo MCPGatewayManager.
- O índice anterior da documentação apontava para README-MCP.md, mas esse arquivo não existe no workspace lido.

## 23. Troubleshooting

### Sintoma: a tool MCP não aparece no agente

Causa provável: local_mcp_configuration.tools não declarou o id esperado, houve conflito de nome com tool base ou a carga MCP falhou e retornou lista vazia.

### Sintoma: o proxy /mcp responde com erro

Causa provável: falta de X-API-Key válido, YAML físico ausente, escopo incompleto na query string ou ausência de servidores stdio naquele escopo.

### Sintoma: stdio configurado, mas o runtime tenta URL HTTP

Isso não é anomalia. No desenho atual, stdio é convertido em streamable_http local apontando para /mcp.

## 24. Diagramas

### 24.1 Fluxo macro do MCP na plataforma

![24.1 Fluxo macro do MCP na plataforma](../assets/diagrams/docs-readme-conceitual-mcp-integracao-uso-sistema-diagrama-01.svg)

O diagrama mostra que MCP não é um caminho lateral. Ele entra no pipeline declarativo, passa por resolução, pode usar cache e só depois integra o runtime final.

## 25. Mapa de navegação conceitual

- Configuração declarativa: global_mcp_configuration e local_mcp_configuration.
- Governança de catálogo: tool_resolver e validators.
- Runtime agentic: MCPToolsResolver e MCPToolsMerger.
- Transporte stdio governado: proxy HTTP em /mcp.
- Segurança: permissões MCP e autenticação com YAML.

## 26. Como colocar para funcionar

O caminho confirmado no código é este:

1. Declarar global_mcp_configuration no YAML, com pelo menos um servidor.
2. Declarar local_mcp_configuration no supervisor, agente ou workflow quando quiser restringir ou especializar o escopo.
3. Se o transporte for stdio, garantir que o YAML venha de arquivo físico, porque o proxy depende de _config_metadata.yaml_source.
4. Executar a API FastAPI, porque o endpoint /mcp é montado no processo HTTP.
5. Garantir permissões MCP válidas para listagem e invocação.

## 27. Exercícios guiados

### Exercício 1

Objetivo: entender a diferença entre declaração global e local.

Passos: localizar um YAML que tenha global_mcp_configuration e comparar com um agente que tenha local_mcp_configuration.tools.

O que observar: o global declara servidores e defaults; o local restringe e especializa o escopo.

### Exercício 2

Objetivo: entender o papel do proxy stdio.

Passos: seguir no código a resolução de um servidor stdio até a geração da URL /mcp.

O que observar: o runtime agentic não usa command e args diretamente nesse ponto; ele consome o proxy HTTP local.

## 28. Checklist de entendimento

- Entendi que MCP aqui é capability governada por YAML, não integração avulsa.
- Entendi a diferença entre configuração global e local.
- Entendi por que stdio vira /mcp.
- Entendi como as tools MCP entram no catálogo efetivo.
- Entendi como o merge com tools locais funciona.
- Entendi os principais riscos operacionais.
- Entendi o valor executivo, comercial e estratégico.

## 29. Evidências no código

- src/agentic_layer/mcp/mcp_config_resolver.py
  - Motivo da leitura: confirmar merge por escopo, transports suportados e conversão de stdio para /mcp.
  - Comportamento confirmado: global_mcp_configuration e local_mcp_configuration geram conexões executáveis por escopo.
- src/agentic_layer/mcp/mcp_tools_resolver.py
  - Motivo da leitura: confirmar carga e cache das tools MCP.
  - Comportamento confirmado: tools MCP são carregadas por escopo e mescladas ao runtime.
- src/agentic_layer/mcp/http_proxy/mcp_stdio_http_proxy.py
  - Motivo da leitura: confirmar o fluxo real do proxy stdio.
  - Comportamento confirmado: /mcp autentica, resolve YAML, recompõe sessão e conversa com processo stdio.
- src/api/service_api.py
  - Motivo da leitura: confirmar qual superfície HTTP está montada no runtime.
  - Comportamento confirmado: o caminho ativo expõe /mcp com mcp_http_proxy_app.
- src/config/agentic_assembly/tool_resolver.py
  - Motivo da leitura: confirmar como tools MCP locais entram no catálogo efetivo.
  - Comportamento confirmado: ids declarados em local_mcp_configuration.tools são adicionados ao catálogo materializado.
