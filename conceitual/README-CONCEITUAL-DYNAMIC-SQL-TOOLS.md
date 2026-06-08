# Manual conceitual, executivo, comercial e estratégico: Dyn SQL

## 1. O que é esta feature

Dyn SQL é a capacidade da plataforma de transformar uma consulta SQL previamente governada em uma tool reutilizável por agentes, workflows e experiências de interface agentic, sem exigir uma nova implementação Python para cada pergunta recorrente ao banco.

O ponto central da feature não é “deixar o agente escrever SQL”. O ponto central é o oposto: impedir SQL improvisada e expor apenas consultas já conhecidas, com vínculo a uma conexão técnica, parâmetros definidos, limites operacionais e decisão explícita de publicação para agentes.

Em linguagem simples, Dyn SQL funciona como um catálogo de consultas aprovadas que podem virar capabilities do runtime. O agente não ganha liberdade para consultar qualquer coisa. Ele ganha acesso apenas ao que já foi cadastrado, validado e publicado.

## 2. Que problema ela resolve

Sem Dyn SQL, a plataforma cairia com frequência em uma destas alternativas ruins.

1. Criar código novo para cada consulta recorrente.
2. Espalhar SQL por YAML, prompt, tela administrativa ou código de interface.
3. Aproximar o agente demais do banco com consultas livres.
4. Duplicar a mesma consulta em múltiplos fluxos e múltiplos clientes.

Dyn SQL resolve isso separando duas responsabilidades que não deveriam andar misturadas.

1. A governança da consulta.
2. O uso da consulta no runtime.

Essa separação é o coração do valor da feature. A consulta nasce como ativo administrável, não como improviso de execução.

## 3. Visão executiva

Executivamente, Dyn SQL reduz custo de evolução da camada de dados usada por agentes. Ele permite publicar novas capacidades de leitura sobre banco sem transformar cada nova consulta em um mini projeto de integração.

O ganho não é apenas velocidade. O ganho real é governança com escalabilidade. A plataforma continua reutilizando o mesmo runtime agentic, enquanto a variação de capabilities passa para uma camada administrável de catálogo e publicação.

Isso impacta diretamente.

1. Tempo para habilitar novas perguntas recorrentes de negócio.
2. Menor acoplamento entre necessidade comercial e deploy de código.
3. Menor risco operacional, porque a exposição ao banco passa por regras administrativas.
4. Melhor capacidade de desligar rapidamente uma consulta sem reescrever fluxo inteiro.

## 4. Visão comercial

Comercialmente, Dyn SQL responde a uma dor real: clientes querem agentes conectados ao dado do negócio, mas não aceitam abrir a porta para SQL livre, nem querem financiar uma nova integração dedicada para cada dashboard, lookup ou relatório.

O discurso comercial correto não é “o agente roda qualquer SQL”. O discurso correto é este: a plataforma consegue transformar consultas aprovadas em ferramentas reutilizáveis, com publicação controlada, parâmetros conhecidos e governança por tenant.

Isso é especialmente útil em cenários como.

1. Varejo e PDV.
2. Backoffice e operação.
3. Atendimento com consulta operacional.
4. Dashboards assistidos.
5. Interfaces AG-UI que precisam de capabilities de leitura já aprovadas.

Exemplos corporativos fortes para esse desenho são estes.

1. PDV consultando KPI de vendas, funil de checkout ou ruptura de estoque sem expor SQL livre ao operador.
2. ERP consultando posição de pedido, saldo, agenda de entrega ou exceções operacionais a partir de queries homologadas.
3. E-commerce consultando mix, conversão, abandono ou disponibilidade por canal com queries governadas reaproveitáveis.
4. Autoatendimento e atendimento assistido consultando segunda via, status de protocolo, limite ou histórico resumido sem abrir acesso amplo ao banco.

## 5. Visão estratégica

Estratégicamente, Dyn SQL fortalece a plataforma porque transforma o banco de dados em fonte governada de capability agentic, sem transformar a plataforma em executor genérico de SQL arbitrária.

Isso cria um meio-termo arquitetural valioso.

1. O produto não precisa escrever uma tool Python para cada consulta.
2. O produto também não precisa aceitar SQL livre para ganhar flexibilidade.

Esse desenho aumenta reutilização, melhora governança por tenant e cria uma ponte importante entre catálogo administrativo, runtime agentic e experiências de UI orientadas a capability.

## 6. Conceitos necessários para entender

### 6.1. dyn_sql

É a família parametrizada de tools SQL dinâmicas. O runtime a reconhece pela sintaxe dyn_sql menor que query_id maior que.

### 6.2. query_id e query_code

No uso do runtime, o identificador prático é o query_id pedido na tool parametrizada. No catálogo persistido, o código estável equivalente é o query_code. Na prática, eles se encontram quando o resolvedor tenta achar a consulta governada correspondente.

### 6.3. connection_id e connection_code

Representam o vínculo técnico entre a consulta e a conexão SQL governada. A query não existe sozinha. Ela sempre depende de uma conexão aprovada.

### 6.4. publish_to_agents

É a trava administrativa que decide se a consulta pode ou não virar capability do runtime agentic.

### 6.5. read_only

É a política de segurança da conexão SQL quando a resolução vem do registro persistido. Nesse modo, dyn_sql exige conexão marcada como somente leitura.

### 6.6. Catálogo parametrizado

Dyn SQL não aparece no runtime como uma lista infinita de tools prontas. O catálogo builtin registra apenas a família base. A tool concreta só nasce quando o runtime recebe um identificador real.

## 7. Como a feature funciona por dentro

O fluxo começa quando um agente, workflow ou interface pede uma tool como dyn_sql menor que clientes_ativos maior que. O runtime reconhece que isso não é uma tool estática comum. É uma família parametrizada.

Nesse momento, a plataforma tenta resolver a consulta por ordem de precedência.

1. Primeiro no YAML efetivo, dentro de tools_config.sql_dynamic.queries.
2. Se não encontrar ali, no registro persistido do tenant.

Se a resolução vier do registro, o sistema converte a consulta e a conexão encontradas para o mesmo formato que a factory consome no YAML. Isso é importante porque evita ter dois runtimes diferentes. O catálogo persistido entra como fonte de verdade alternativa, mas a materialização final continua seguindo o contrato da factory dinâmica.

Depois disso, a tool concreta é construída com schema de parâmetros, conexão resolvida, retry e cache. A capability fica pronta para uso sem exigir tool Python exclusiva para aquela consulta.

## 8. Divisão em etapas ou submódulos

### 8.1. Catálogo builtin da família dyn_sql

Essa etapa existe para registrar que dyn_sql é uma família reconhecida pelo runtime. O catálogo não precisa conhecer cada query concreta. Ele precisa conhecer a capacidade de materialização.

### 8.2. Catálogo administrativo de conexões SQL

Essa etapa separa a conexão técnica da consulta. Isso evita que o produto trate segredo, engine, modo de conexão e política de leitura como detalhe espalhado em vários lugares.

### 8.3. Catálogo administrativo de queries SQL

Essa etapa transforma a consulta em ativo governado, com código estável, descrição, limite de linhas, timeout, parâmetros e decisão explícita de publicação.

### 8.4. Resolvedor do registro persistido

Essa etapa existe para converter o registro administrativo em estrutura consumível pela factory de runtime. Ela também é onde entram os guardrails mais importantes, como tenant obrigatório, query ativa, publicação para agentes e conexão read_only.

### 8.5. Factory de materialização

Essa etapa constrói a tool concreta. É onde a consulta deixa de ser apenas um ativo cadastrado e passa a ser uma capability executável pelo agente.

### 8.6. Camada de uso agentic e UI

Essa etapa é onde a capability finalmente aparece para agentes, workflows ou experiências de interface, como dashboards assistidos e AG-UI.

## 9. Pipeline ou fluxo principal

O fluxo principal, explicado em linguagem operacional, é este.

1. Um fluxo agentic pede dyn_sql para uma query específica.
2. O runtime identifica que se trata de uma factory parametrizada.
3. Ele tenta resolver a query no YAML atual.
4. Se não encontrar, tenta no registro persistido do tenant.
5. Se o registro estiver apto para publicação, a consulta é convertida para o formato de runtime.
6. A factory constrói a tool com contrato de parâmetros e conexão técnica.
7. O agent usa a capability materializada.

O ponto importante é que a query não nasce do nada na hora da execução. Ela chega já governada ao runtime.

## 10. Decisões técnicas e trade-offs

### 10.1. Registrar a família, não cada tool concreta

Ganho: reduz explosão de catálogo e mantém o runtime flexível.

Custo: a tool concreta só pode ser entendida plenamente quando o query_id é resolvido.

### 10.2. Buscar no YAML antes do registro persistido

Ganho: preserva o princípio YAML-first da plataforma.

Custo: obriga o operador a entender que a mesma capability pode vir de duas origens controladas, não de uma fonte única simplista.

### 10.3. Exigir read_only no modo por registro

Ganho: reduz risco de mutação indevida ao publicar capacidades SQL para agentes.

Custo: limita casos de uso que queiram usar o mesmo mecanismo para escrita ou procedure.

### 10.4. Separar dyn_sql de NL2SQL

Ganho: impede confusão entre consulta governada e geração assistida de SQL.

Custo: o produto precisa manter discursos e fluxos distintos para duas capacidades próximas, mas conceitualmente diferentes.

## 11. Configurações e decisões que mudam o comportamento

Com base no código lido, estas decisões mudam o comportamento real da feature.

1. A existência ou não de tools_config.sql_dynamic no YAML.
2. A presença ou ausência de tenant_id em user_session.
3. O status is_active da query no registro.
4. O status publish_to_agents da query.
5. O status is_active da conexão técnica.
6. O atributo read_only da conexão técnica.
7. O schema de parâmetros da consulta.
8. O timeout e o limite de linhas definidos no registro.

## 12. O que acontece em caso de sucesso

No caminho feliz, a plataforma encontra uma query publicada, valida a conexão correta, materializa a capability e entrega ao runtime uma tool utilizável. O agente passa a enxergar a consulta como capability pronta, e não como SQL para inventar.

Para o usuário final, o efeito prático é simples: a pergunta de negócio vira uma ação de runtime previsível e governada.

## 13. O que acontece em caso de erro

Os erros confirmados no código lido mostram a filosofia da feature.

1. Sem tenant_id explícito, o runtime não consulta o registro persistido.
2. Query inativa não vira capability.
3. Query não publicada para agentes não vira capability.
4. Conexão inativa não sustenta a capability.
5. Conexão sem read_only não sustenta dyn_sql por registro.
6. Query sem conexão associada falha antes da materialização.

Em linguagem simples, Dyn SQL falha fechado. Ele prefere não existir no runtime a existir fora da governança.

## 14. Observabilidade e diagnóstico

Para investigar Dyn SQL, a pergunta certa não é “qual SQL o agente inventou”. A pergunta certa é “de onde essa capability deveria ter vindo e por que a governança não fechou?”.

A ordem mais segura de investigação é esta.

1. Confirmar se a capability estava no YAML ou dependia do registro persistido.
2. Confirmar tenant_id em user_session.
3. Confirmar se a query está ativa e publicada.
4. Confirmar se a conexão está ativa e marcada como somente leitura.
5. Só depois investigar parâmetro, banco e resultado.

## 15. Impacto técnico

Tecnicamente, Dyn SQL reduz duplicação de tools, centraliza governança de consulta, melhora segurança operacional no modo por registro e preserva o princípio YAML-first mesmo quando usa catálogo persistido.

## 16. Impacto executivo

Executivamente, Dyn SQL reduz custo de customização, acelera publicação de novas capacidades analíticas e melhora governança sobre o que os agentes realmente podem consultar.

## 17. Impacto comercial

Comercialmente, Dyn SQL melhora a proposta de valor em clientes que precisam de acesso a dados reais com forte controle de segurança e publicação.

## 18. Impacto estratégico

Estratégicamente, Dyn SQL ajuda a plataforma a crescer por catálogo governado de capabilities, e não apenas por novas integrações codificadas manualmente.

## 19. Exemplos práticos guiados

### 19.1. Dashboard assistido por capability de vendas

Cenário: o cliente quer um painel assistido que mostre uma consulta recorrente de vendas por loja.

Dyn SQL permite publicar essa consulta como capability governada e reutilizá-la em dashboard, AG-UI e agente especialista sem criar uma tool Python nova para o mesmo caso.

### 19.2. Agent operacional que consulta estoque aprovado

Cenário: o agente precisa consultar posição de estoque, mas a organização não aceita SQL livre.

Dyn SQL resolve isso ao expor apenas a query aprovada, com parâmetros conhecidos e conexão governada.

### 19.3. Tenant que precisa desligar rapidamente uma consulta

Cenário: uma query publicada não deve mais aparecer para agentes.

A organização pode retirá-la da publicação ou desativá-la no catálogo administrativo, sem refatorar o runtime inteiro.

### 19.4. ERP com cockpit financeiro assistido

Cenário: o time financeiro quer um cockpit assistido para consultar títulos em atraso, pedidos bloqueados e aprovações pendentes.

Dyn SQL permite expor cada leitura recorrente como capability governada, mantendo parâmetros, limite de linhas e conexão técnica sob controle administrativo.

### 19.5. E-commerce com radar de conversão e sortimento

Cenário: uma operação digital quer ler rapidamente abandono, disponibilidade de catálogo e performance por categoria.

Dyn SQL permite reaproveitar a mesma query aprovada em agente especialista, dashboard assistido e AG-UI sem duplicar implementação.

## 20. Explicação 101

Pense em Dyn SQL como um balcão de consultas autorizadas. O agente não entra na cozinha para inventar um prato novo. Ele pede algo que já está no cardápio. O catálogo administrativo define o cardápio. O runtime só entrega o que já foi aprovado.

## 21. Limites e pegadinhas

1. Dyn SQL não é gerador de SQL por linguagem natural.
2. Dyn SQL não é executor de SQL arbitrária enviada pelo usuário.
3. Dyn SQL não substitui proc_sql quando o problema real é procedure.
4. Dyn SQL não substitui o endpoint dedicado de NL2SQL quando o objetivo é propor SQL nova a partir de pergunta livre.
5. A existência da sintaxe dyn_sql não garante que a query exista, esteja ativa ou publicada.

## 22. Troubleshooting

### 22.1. Sintoma: a capability dyn_sql não aparece

Causa provável: a query não existe no YAML e também não foi encontrada no catálogo persistido do tenant.

### 22.2. Sintoma: a capability existe, mas não materializa por registro

Causa provável: user_session.tenant_id ausente ou inválido para a resolução governada.

### 22.3. Sintoma: a query cadastrada não chega aos agentes

Causa provável: publish_to_agents está falso.

### 22.4. Sintoma: a query foi publicada, mas o runtime bloqueia a conexão

Causa provável: a conexão técnica está inativa ou não está marcada como read_only.

## 23. Checklist de entendimento

- Entendi que Dyn SQL publica consultas aprovadas, não SQL livre.
- Entendi a diferença entre Dyn SQL e NL2SQL.
- Entendi a diferença entre Dyn SQL e proc_sql.
- Entendi por que tenant_id, is_active, publish_to_agents e read_only são guardrails críticos.
- Entendi por que a família dyn_sql é parametrizada e não uma lista fixa de tools.

## 24. Evidências no código

- src/agentic_layer/tools/domain_tools/dynamic_sql_tools/dynamic_sql_factory.py
  - Motivo da leitura: confirmar como a tool concreta é materializada.
  - Comportamento confirmado: o runtime resolve query_id, constrói schema de parâmetros, cria conexão SQL com retry e usa cache.

- src/agentic_layer/tools/domain_tools/dynamic_tool_registry_resolver.py
  - Motivo da leitura: confirmar o fluxo de resolução pelo catálogo persistido.
  - Comportamento confirmado: tenant_id, query ativa, publish_to_agents e conexão read_only são exigidos no modo por registro.

- src/agentic_layer/tools/tools_library_builder.py
  - Motivo da leitura: confirmar o papel de dyn_sql como família parametrizada builtin.
  - Comportamento confirmado: o catálogo registra dyn_sql como factory base, não como lista de queries concretas.

- src/config/agentic_assembly/validators/tools_semantic_validator.py
  - Motivo da leitura: confirmar como a AST reconhece referências parametrizadas.
  - Comportamento confirmado: dyn_sql menor que query_id maior que é tratado como prefixo válido no contrato de tools.

- src/api/services/admin/integrations_service.py
  - Motivo da leitura: confirmar o boundary administrativo de queries SQL governadas.
  - Comportamento confirmado: a plataforma já possui CRUD administrativo para registrar e publicar queries funcionais.

- src/api/schemas/admin_integrations_models.py
  - Motivo da leitura: confirmar o contrato administrativo dos registros SQL.
  - Comportamento confirmado: query_code, connection_id, parameter_schema_json, max_rows, timeout_seconds e publish_to_agents fazem parte do ativo governado.
