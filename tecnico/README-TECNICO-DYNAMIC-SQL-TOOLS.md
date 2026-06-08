# Manual técnico e operacional: Dyn SQL

## 1. Objetivo deste manual

Este manual explica o funcionamento técnico real do dyn_sql no repositório: como a família parametrizada é registrada, como o runtime resolve a query, como o catálogo persistido entra no fluxo e quais guardrails realmente bloqueiam a materialização da tool.

O foco é dyn_sql como capability de consulta governada. Stored procedures e NL2SQL aparecem apenas como comparações necessárias para evitar interpretação errada.

## 2. O que o runtime realmente reconhece

O código confirma três famílias parametrizadas irmãs no catálogo builtin.

1. dyn_sql para consultas SQL governadas.
2. dyn_api para operações HTTP governadas.
3. proc_sql para stored procedures governadas.

No caso deste manual, o recorte principal é dyn_sql, cuja sintaxe pública é dyn_sql menor que query_id maior que.

## 3. Onde dyn_sql entra no sistema

Dyn SQL participa de quatro camadas principais.

1. Catálogo builtin de tools parametrizadas.
2. Validação semântica do contrato agentic.
3. Resolução da consulta e da conexão.
4. Materialização e execução da tool.

Essas quatro camadas explicam o slice inteiro sem precisar confundir o leitor com mapa de diretórios.

## 4. Registro builtin da família parametrizada

O builder do catálogo builtin adiciona dyn_sql manualmente como factory base parametrizada. Isso acontece porque dyn_sql não pode ser descoberto como uma lista fixa de tools concretas. Cada tool real depende de um identificador de query resolvido em runtime.

O comportamento técnico importante é este.

1. O catálogo reconhece dyn_sql como id base.
2. A descrição da família já declara a sintaxe dyn_sql menor que query_id maior que.
3. O factory_impl aponta para create_dynamic_sql_tool.

Sem esse passo, a plataforma não saberia que dyn_sql é uma referência válida no ecossistema agentic.

## 5. Reconhecimento semântico no assembly agentic

O validador semântico de tools trata dyn_sql como prefixo parametrizado reconhecido. Isso importa porque a AST não precisa conhecer antecipadamente cada query concreta. Ela precisa apenas reconhecer que dyn_sql menor que algo maior que é uma referência aceitável, desde que a família base exista no catálogo efetivo.

Na prática, isso permite que workflows e supervisores referenciem dyn_sql sem exigir cadastro estático de cada tool concreta em tools_library.

## 6. Precedência de resolução da query

O código da factory confirma esta ordem exata.

1. O runtime recebe query_id via tool_config.
2. A factory procura query_id em tools_config.sql_dynamic.queries.
3. Se não encontrar, chama DynamicToolRegistryResolver.resolve_sql_dynamic_section.
4. O resolvedor consulta o registro persistido e devolve uma seção compatível com o contrato do YAML.
5. A factory faz merge dessa seção no config atual.
6. A partir daí, o restante do pipeline segue como se a query sempre tivesse estado no YAML.

Esse detalhe é arquiteturalmente importante: o catálogo persistido não cria um runtime paralelo. Ele abastece o mesmo runtime.

## 7. Contrato local esperado no YAML

Quando dyn_sql é resolvida localmente no YAML, a factory espera pelo menos dois blocos.

1. tools_config.sql_dynamic.connections.
2. tools_config.sql_dynamic.queries.

Na query, o runtime depende principalmente de.

1. connection.
2. sql.
3. description.
4. parameters.
5. result_format.
6. fetch_mode.
7. include_columns.

Na conexão, o runtime depende de.

1. uri direta.
2. Ou secret_key apontando para security_keys.

## 8. Contrato persistido usado pelo resolvedor

Quando a query não está no YAML, o runtime consulta o catálogo persistido. O comportamento lido no código exige estes elementos.

### 8.1. user_session.tenant_id

Sem tenant_id explícito, a busca no catálogo persistido é bloqueada.

### 8.2. Registro de query ativo e publicado

O registro em sql_query_registry precisa estar ativo e com publish_to_agents verdadeiro.

### 8.3. Registro de conexão ativo e read_only

A conexão em sql_connection_registry precisa estar ativa e marcada como somente leitura.

### 8.4. Modo de conexão suportado

O resolvedor suporta secret_ref e inline_encrypted. Outros modos são rejeitados.

## 9. Como o resolvedor converte o catálogo persistido

O DynamicToolRegistryResolver transforma os registros administrativos em um bloco que a factory já entende.

Ele produz basicamente duas partes.

1. connections, com a conexão resolvida em formato de runtime.
2. queries, com a consulta resolvida em formato de runtime.

Depois disso, merge_sql_dynamic_section injeta esse conteúdo em tools_config.sql_dynamic. Esse desenho evita bifurcação de comportamento e mantém a factory simples: ela continua lendo a mesma estrutura final, independentemente da origem.

## 10. Guardrails confirmados no código

### 10.1. Guardrails do catálogo persistido

Os guardrails confirmados nesse fluxo são.

1. tenant_id obrigatório.
2. query precisa existir.
3. query precisa estar ativa.
4. query precisa estar publicada para agentes.
5. conexão precisa existir.
6. conexão precisa estar ativa.
7. conexão precisa estar marcada como read_only.

### 10.2. Guardrails da factory

Mesmo depois da resolução, a factory continua exigindo.

1. Prefixo dyn_sql_ no nome final da tool.
2. connection_id obrigatório.
3. query_id obrigatório.
4. SQL não vazia.
5. Conexão resolvível por uri ou secret_key.

## 11. Construção do schema de parâmetros

Dyn SQL extrai placeholders do SQL no formato arroba p número, por exemplo p1, p2 e p3. Em seguida, monta um schema Pydantic dinâmico a partir da lista de parâmetros declarada na configuração.

Esse schema inclui.

1. Os parâmetros SQL propriamente ditos.
2. fetch opcional.
3. include_columns opcional.

Esse desenho permite que a tool concreta tenha contrato de entrada consistente, mesmo sem existir como classe Python específica para cada consulta.

## 12. Execução e retry

Dyn SQL possui dois pontos claros de retry.

1. Na criação da conexão SQLDatabase.
2. Na execução da consulta.

Nos dois casos, o retry cobre falhas transitórias de banco, como OperationalError e DatabaseError. Isso está alinhado com a política geral do projeto de não acessar recurso externo sem retry explícito.

## 13. Cache e reutilização

Dyn SQL usa o resource pool com chave determinística construída a partir de.

1. tenant.
2. nome da tool.
3. connection_id.
4. query_id.
5. SQL.
6. parâmetros.
7. force_cache_refresh.

Na prática, isso significa que a plataforma não reconstrói a mesma tool dinâmica toda vez que o runtime a pede. Ela reaproveita a versão materializada quando a configuração lógica continua a mesma.

## 14. Fluxo técnico ponta a ponta

O fluxo completo, em narrativa técnica, é este.

1. Um workflow, DeepAgent ou UI pede dyn_sql menor que query_id maior que.
2. O runtime identifica dyn_sql como família parametrizada builtin.
3. O query_id é injetado em tool_config.
4. A factory tenta resolver a query no YAML.
5. Se não encontrar, consulta o catálogo persistido por tenant.
6. O resolvedor aplica guardrails e converte query mais conexão para o formato do runtime.
7. A factory monta o schema de entrada, resolve a conexão e cria a tool concreta.
8. O resource pool devolve a tool do cache ou a constrói pela primeira vez.
9. A execução da tool substitui placeholders para o formato SQLAlchemy e roda a query com retry.
10. O resultado é formatado e devolvido ao agente.

## 15. Relação com proc_sql

Proc_sql é a família irmã para stored procedures governadas. O contrato é semelhante em espírito, mas o objetivo é diferente. Dyn SQL é voltada a consulta SQL parametrizada. Proc_sql é voltada a chamada de procedure.

Isso importa porque não é correto documentar dyn_sql como mecanismo geral de qualquer execução SQL. O produto já separa consulta e procedure em famílias distintas.

## 16. Relação com NL2SQL

NL2SQL dedicado é outro slice diferente. Ele usa schema_rag_sql para propor uma SQL a partir de linguagem natural e submete a proposta ao guardrail central de somente leitura.

Dyn SQL não gera SQL. Dyn SQL materializa SQL já aprovada. Essa diferença muda operação, risco e posicionamento.

## 17. Boundary administrativo confirmado no código

O serviço administrativo do repositório já possui CRUD para queries SQL governadas. O contrato administrativo expõe, entre outros, estes campos.

1. query_code.
2. connection_id.
3. query_kind.
4. sql_text.
5. parameter_schema_json.
6. result_format.
7. max_rows.
8. timeout_seconds.
9. publish_to_agents.

Isso confirma que Dyn SQL não depende apenas de YAML local. Existe uma trilha oficial de governança administrativa para publicar e manter esse catálogo.

No banco, o ativo central desse slice fica em integrations.sql_query_registry, vinculado a integrations.sql_connection_registry por connection_id. Em termos operacionais, isso significa que dyn_sql por registro sempre nasce do encontro entre uma query governada e uma conexão técnica igualmente governada.

## 18. O que acontece em caso de sucesso

No caminho feliz, a query é encontrada, a conexão correta é validada, a tool concreta é construída e o agent recebe uma capability reutilizável. Para o runtime, a consulta passa a ser mais um recurso governado do catálogo de tools.

## 19. O que acontece em caso de erro

Os cenários principais confirmados no código incluem.

1. Query ausente do YAML e do registro persistido.
2. tenant_id ausente para resolução por registro.
3. Query inativa.
4. Query não publicada para agentes.
5. Conexão ausente.
6. Conexão inativa.
7. Conexão sem read_only no modo por registro.
8. Secret key ausente em security_keys.
9. SQL vazia.
10. Query sem connection definida.

## 20. Observabilidade e diagnóstico

Para diagnosticar Dyn SQL, siga esta ordem.

1. Descubra se a origem esperada era YAML local ou catálogo persistido.
2. Confirme query_id e tenant_id.
3. Confirme publish_to_agents e is_active da query.
4. Confirme is_active e read_only da conexão.
5. Confirme se security_keys contém a chave exigida no modo secret_ref.
6. Só depois investigue banco, retry ou formato do resultado.

## 21. Como colocar para funcionar

O caminho mínimo confirmado no código é este.

### 21.1. Via YAML local

1. Declarar dyn_sql como capability parametrizada em tools.
2. Garantir tools_config.sql_dynamic.connections.
3. Garantir tools_config.sql_dynamic.queries.
4. Garantir security_keys quando a conexão usa secret_key.

### 21.2. Via catálogo persistido

1. Registrar conexão SQL governada.
2. Registrar query SQL governada apontando para a conexão.
3. Marcar query como publish_to_agents.
4. Garantir user_session.tenant_id no runtime.
5. Garantir read_only na conexão quando dyn_sql depender do registro.

## 22. Exemplos práticos guiados

### 22.1. Query local no YAML

Cenário: um workflow carrega localmente a consulta que precisa usar.

Resultado técnico: a factory resolve a query diretamente em tools_config.sql_dynamic e não consulta o catálogo persistido.

### 22.2. Query publicada por tenant

Cenário: o YAML pede dyn_sql para uma query que não está localmente no arquivo, mas já foi publicada administrativamente.

Resultado técnico: o resolvedor consulta sql_query_registry e sql_connection_registry, aplica os guardrails e injeta a configuração equivalente no runtime.

### 22.3. Query bloqueada por governança

Cenário: o cadastro existe, mas publish_to_agents está falso ou a conexão não está read_only.

Resultado técnico: a materialização falha antes de virar capability do agente.

### 22.4. PDV usando KPI de vendas homologado

Cenário: uma tela operacional ou agente de varejo pede dyn_sql para consultar KPI de vendas do período.

Resultado técnico: o runtime resolve query_code no catálogo do tenant, valida a conexão read_only e devolve a capability pronta sem expor SQL livre ao frontend.

### 22.5. ERP consultando posição de pedido governada

Cenário: um DeepAgent de ERP precisa consultar uma leitura recorrente de pedidos bloqueados.

Resultado técnico: a mesma query governada pode alimentar workflow, DeepAgent ou interface assistida, desde que o tenant tenha publish_to_agents ativo e a conexão correta cadastrada.

### 22.6. E-commerce em dashboard assistido

Cenário: uma UI assistida precisa materializar um painel de conversão com consultas já aprovadas.

Resultado técnico: dyn_sql vira a fonte governada da capability de leitura e evita duplicação de lógica entre dashboard, AG-UI e agente analítico.

## 23. Explicação 101

Dyn SQL funciona como uma moldura pronta para consultas aprovadas. A estrutura de execução já existe. O que muda é qual consulta governada vai ocupar essa moldura. Se a consulta estiver aprovada e ligada à conexão certa, a tool nasce. Se não estiver, o runtime bloqueia.

## 24. Limites e pegadinhas

1. Dyn SQL não é um executor universal de SQL.
2. Dyn SQL não substitui proc_sql.
3. Dyn SQL não gera SQL nova a partir de linguagem natural.
4. Tenant ausente quebra a resolução por registro.
5. Publicação administrativa é parte do contrato, não detalhe opcional.
6. Read_only é requisito explícito no modo por registro, não sugestão.

## 25. Troubleshooting

### 25.1. Sintoma: dyn_sql funciona em um tenant e falha em outro

Causa provável: diferença no catálogo persistido ou ausência de tenant_id correto.

### 25.2. Sintoma: a query existe no catálogo, mas a tool não nasce

Causa provável: query inativa, não publicada ou conexão em estado incompatível.

### 25.3. Sintoma: a tool nasce, mas falha ao conectar

Causa provável: secret_key ausente, uri inválida ou falha do banco fora do contrato esperado.

### 25.4. Sintoma: a AST rejeita a referência parametrizada

Causa provável: a família base dyn_sql não está no catálogo efetivo ou a referência local à tool está inconsistente.

## 26. Checklist de entendimento

- Entendi como dyn_sql é registrado no catálogo builtin.
- Entendi como a AST reconhece dyn_sql menor que query_id maior que.
- Entendi a precedência YAML primeiro e registro depois.
- Entendi os guardrails do catálogo persistido.
- Entendi como a tool concreta monta schema, retry e cache.
- Entendi por que dyn_sql é diferente de proc_sql e NL2SQL.

## 27. Evidências no código

- src/agentic_layer/tools/tools_library_builder.py
  - Motivo da leitura: confirmar o registro builtin da família parametrizada.
  - Comportamento confirmado: dyn_sql é adicionado manualmente ao catálogo como factory base parametrizada.

- src/config/agentic_assembly/validators/tools_semantic_validator.py
  - Motivo da leitura: confirmar validação semântica de referências parametrizadas.
  - Comportamento confirmado: dyn_sql menor que algo maior que é reconhecido como referência válida quando a família base existe.

- src/agentic_layer/tools/domain_tools/dynamic_sql_tools/dynamic_sql_factory.py
  - Motivo da leitura: confirmar precedência de resolução, schema dinâmico, retry e cache.
  - Comportamento confirmado: a factory resolve query_id, pode buscar no registro, monta a tool concreta e usa resource pool.

- src/agentic_layer/tools/domain_tools/dynamic_tool_registry_resolver.py
  - Motivo da leitura: confirmar guardrails do catálogo persistido.
  - Comportamento confirmado: tenant obrigatório, query ativa e publicada, conexão ativa e read_only no modo dyn_sql por registro.

- src/api/services/admin/integrations_service.py
  - Motivo da leitura: confirmar CRUD administrativo de queries SQL governadas.
  - Comportamento confirmado: a plataforma possui boundary oficial para cadastrar, atualizar e ativar queries funcionais.

- src/api/schemas/admin_integrations_models.py
  - Motivo da leitura: confirmar campos do ativo administrativo.
  - Comportamento confirmado: query_code, sql_text, parameter_schema_json, max_rows, timeout_seconds e publish_to_agents fazem parte do contrato.

- src/api/services/nl2sql_service.py
  - Motivo da leitura: diferenciar dyn_sql do fluxo dedicado de NL2SQL.
  - Comportamento confirmado: NL2SQL é um fluxo separado, com geração assistida de SQL e guardrail somente leitura.

- src/integrations/sql_read_only_guardrail.py
  - Motivo da leitura: confirmar a proteção central usada pelo slice NL2SQL.
  - Comportamento confirmado: o guardrail valida SQL somente leitura e diferencia esse fluxo do dyn_sql governado.
