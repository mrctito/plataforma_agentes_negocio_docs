# Manual conceitual, executivo, comercial e estrategico: dyn_api

## 1. O que e esta feature

dyn_api e a capacidade da plataforma que transforma uma operacao HTTP aprovada em uma tool utilizavel por agentes e workflows sem exigir um conector novo em Python para cada endpoint. Na pratica, ela pega um contrato declarativo de chamada HTTP, materializa esse contrato como tool parametrizada e entrega ao runtime um meio controlado de chamar APIs externas ja homologadas.

Isso significa que dyn_api nao e um executor livre de qualquer URL nem um proxy generico aberto. O codigo lido mostra o contrario: a family so funciona quando existe um endpoint declarado no YAML local ou quando existe uma operacao governada registrada para o tenant. O resultado pratico e importante: a plataforma ganha velocidade para expandir integracoes simples, mas continua impondo catalogo, publicacao explicita para agentes e checagens de compatibilidade.

## 2. Que problema ela resolve

Sem dyn_api, cada endpoint externo que um agente precisasse consultar tenderia a virar uma implementacao dedicada. Isso aumenta custo de manutencao, obriga deploy para mudancas pequenas e espalha regras de autenticacao, timeout e contrato por varios conectores.

dyn_api encapsula essa complexidade em uma capability governada. Ela resolve quatro dores ao mesmo tempo:

- reduz repeticao de integracoes HTTP simples;
- separa o que e capacidade homologada do que e experimento improvisado;
- permite publicar operacoes por tenant sem reescrever o runtime;
- preserva controle de seguranca porque a operacao precisa estar ativa, publicada e compativel com o resolvedor.

## 3. Visao executiva

Executivamente, dyn_api reduz atrito entre demanda de negocio e entrega tecnica. Em vez de abrir ciclo completo de desenvolvimento para cada endpoint homologado, a plataforma pode cadastrar a operacao, associar autenticacao e liberar a capacidade para agentes quando a governanca aprovar.

O ganho pratico para lideranca e previsibilidade. O modelo reduz backlog de conectores pequenos, concentra risco em um runtime unico e facilita auditoria porque a superficie publicada para agentes deixa de depender de codigo espalhado. Tambem melhora segregacao de responsabilidade: o time tecnico continua decidindo o que pode ser publicado, enquanto o produto ganha mais velocidade para evoluir catalogos por tenant.

## 4. Visao comercial

Comercialmente, dyn_api ajuda a explicar que a plataforma integra sistemas externos com mais velocidade sem abrir mao de controle. A mensagem correta nao e "conecta em qualquer API". A mensagem correta e "publica operacoes aprovadas de APIs externas como capacidades reutilizaveis para agentes".

Isso responde dores comuns de clientes corporativos:

- demora para expor operacoes simples de ERP, CRM ou servicos internos;
- dependencia excessiva de deploy para ajustes pequenos;
- receio de deixar agentes acessarem APIs externas sem catalogo e sem autenticação governada.

Um exemplo comercial honesto e um ERP que precisa consultar status de pedido, limite de credito ou disponibilidade de agenda em uma API interna. Se o contrato HTTP for estavel e declarativo, dyn_api reduz o tempo entre a homologacao da operacao e a disponibilizacao para agentes.

Outros exemplos corporativos fortes, coerentes com o desenho atual, sao estes.

- CRM: consultar situacao de cliente, limite de credito ou historico resumido sem abrir uma integracao nova para cada operacao pequena.
- ERP: consultar pedido, nota, agenda de entrega, saldo de estoque ou aprovacao de compra usando operacoes REST homologadas.
- E-commerce: ler status de pedido, disponibilidade de entrega, campanha ou catalogo a partir de APIs internas ou externas aprovadas.
- Autoatendimento e atendimento assistido: consultar dados de cliente, protocolo, segunda via ou status operacional por meio de um conjunto restrito de operacoes publicadas.

## 5. Visao estrategica

Estrategicamente, dyn_api fortalece a tese da plataforma como runtime YAML-first governado. Ela amplia o conjunto de capacidades reutilizaveis sem transformar cada nova necessidade externa em um ramo especial da arquitetura.

O efeito mais importante e arquitetural: a plataforma passa a crescer por catalogo e contratos, nao apenas por codigo novo. Isso aproxima integracoes HTTP da mesma logica usada em outras superficies governadas do produto, como dyn_sql e o assembly agentic. O ganho nao e so produtividade. E tambem padronizacao, isolamento de responsabilidade e base mais forte para multi-tenant.

## 6. Conceitos necessarios para entender

### Tool parametrizada

Uma tool parametrizada e uma family de tool que recebe um identificador estavel para escolher qual capacidade concreta deve ser materializada. Em dyn_api, o identificador usado pelo runtime e o endpoint_id. Isso evita cadastrar manualmente uma tool fixa para cada operacao HTTP.

### Catalogo governado

Catalogo governado e o conjunto de registros persistidos que um tenant pode publicar para uso operacional. No caso de dyn_api, o codigo mostra que operacoes HTTP e perfis de autenticacao podem viver nesse registro e so entram no runtime se estiverem ativas e publicadas da forma esperada.

### YAML-first

YAML-first, neste contexto, nao significa que tudo vem apenas de um arquivo YAML local. Significa que o runtime prioriza configuracao declarativa e depois complementa a capacidade com registro governado quando o endpoint nao existe localmente. O YAML continua sendo o primeiro lugar de resolucao, mas nao o unico.

### Perfil de autenticacao

Perfil de autenticacao e o contrato reutilizavel que descreve como uma API exige credenciais. O projeto suporta perfis estaticos e perfis que precisam buscar token dinamicamente. Esse detalhe importa porque dyn_api nao trata autenticacao como detalhe solto do endpoint. Ela a materializa como parte governada da capacidade.

### Tabela de operacoes API

No slice lido, a tabela operacional mais importante para dyn_api e integrations.api_operation_registry. E nela que a operacao HTTP governada fica persistida com codigo estavel, metodo, base_url, path_template, schemas de entrada, timeout, publish_to_agents e, quando aplicavel, vinculo com auth profile e origem Swagger.

Em linguagem simples, essa tabela funciona como o cardapio de operacoes HTTP que podem, ou nao, virar capability de agente.

### Origem Swagger

Quando a operacao vem de uma importacao OpenAPI, a trilha de origem fica guardada em integrations.api_swagger_source_registry. Isso nao e detalhe burocratico. Essa tabela preserva rastreabilidade da fonte importada, tipo de origem, hash do documento e relacao entre a especificacao importada e as operacoes criadas ou atualizadas.

### Capability governada

Capability governada e uma capacidade que o agente pode usar somente depois de passar por regra explicita de cadastro, ativacao e publicacao. Esse conceito e central para entender por que dyn_api nao e um executor HTTP generico e por que isso e bom para a plataforma.

## 7. Como a feature funciona por dentro

O fluxo comeca quando um agente ou workflow referencia a family dyn_api com um endpoint especifico. O loader converte essa referencia em payload de factory e injeta o parametro correto para a materializacao.

Depois disso, a factory tenta resolver o endpoint no bloco local de configuracao. Se encontrar, usa o contrato local. Se nao encontrar, sobe um nivel de governanca e pede ao resolvedor de registro persistido que encontre uma operacao HTTP publicada para o tenant. Quando essa segunda trilha e usada, o runtime tambem pode materializar o auth profile associado.

Com o endpoint resolvido, a factory monta dinamicamente o schema de entrada da tool. Isso e importante porque a tool deixa de ser um disparo cego: ela valida path, query e body antes de chamar a API externa. Em seguida, resolve headers, placeholders, URL final e autenticacao. So depois a chamada HTTP e executada, com timeout e retry no cliente dedicado.

Existe ainda uma etapa anterior importante para operacoes importadas. Antes de virar capability, uma especificacao Swagger pode ser importada para o catalogo governado, gerando ou atualizando a origem em api_swagger_source_registry e as operacoes em api_operation_registry. So depois disso a governanca decide o que realmente sera publicado para agentes.

## 8. Divisao em etapas ou submodulos

### 8.1 Resolucao da capacidade

Esta etapa decide qual operacao HTTP realmente sera exposta. Primeiro ela tenta o YAML local. Se falhar, usa o registro governado por tenant. O valor entregue por essa etapa e um contrato unico, coerente e apto a ser materializado como tool.

O que pode dar errado aqui e estrutural: tenant ausente, operacao nao publicada, operacao inativa ou protocolo incompatível. O valor dessa etapa e impedir que o runtime improvise uma integracao que nao foi aprovada.

### 8.2 Materializacao do contrato

Depois de resolver a operacao, a factory transforma o contrato em schema executavel. Ela cria parametros dinamicos para path, query e body, organiza headers e prepara a interpolacao de placeholders.

Essa etapa existe para tirar ambiguidade do uso da tool. O agente nao deve inventar campos fora do contrato nem mandar payload sem forma conhecida. O ganho aqui e previsibilidade operacional.

### 8.3 Materializacao da autenticacao

Se a operacao aponta para autenticacao, o runtime precisa decidir se usa cabecalho estatico ou fluxo de token dinamico. Quando o perfil e dinamico, o AuthenticationManager resolve placeholders, consulta credenciais protegidas e tenta reaproveitar token em cache.

Essa etapa separa o problema de chamar a API do problema de provar identidade para a API. Sem essa separacao, a manutencao vira mistura de contrato HTTP com segredo e renovacao de token.

### 8.4 Execucao HTTP resiliente

Com contrato e autenticacao prontos, o cliente HTTP executa a chamada usando timeout e retry para erros transitórios de rede e timeout. Essa etapa entrega o resultado bruto ou a falha executavel para o restante do fluxo.

O que pode dar errado aqui e de infraestrutura: timeout, erro de conexao, resposta invalida ou indisponibilidade do provider externo. O valor desta etapa e encapsular resiliência num cliente unico, em vez de repetir codigo de HTTP por integracao.

### 8.5 Importacao Swagger como esteira de cadastro

Esta etapa existe para acelerar onboarding de APIs sem pular governanca. O documento OpenAPI e lido, validado, normalizado e convertido em registros administrativos. A origem vai para api_swagger_source_registry. As operacoes vao para api_operation_registry.

O ponto mais importante aqui e operacional: importar nao significa publicar. O runtime continua exigindo status ativo e publicacao explicita para agentes.

## 9. Decisoes tecnicas e trade-offs

### Priorizar YAML local antes do registro persistido

Ganho: preserva previsibilidade do runtime local e permite configuracao embarcada por YAML.

Custo: o leitor precisa entender duas fontes de resolucao.

Impacto: o produto continua YAML-first sem perder a capacidade de publicar operacoes governadas por tenant.

### Exigir publicacao explicita para agentes

Ganho: impede que todo cadastro administrativo vire capacidade agentic automaticamente.

Custo: aumenta o numero de passos de governanca.

Impacto: reduz risco de expor operacao sensivel por acidente.

### Tornar a importacao Swagger idempotente e sem publicacao automatica

Ganho: facilita onboarding e sincronizacao de contrato sem transformar a importacao em exposicao imediata para agentes.

Custo: adiciona uma etapa administrativa posterior de revisao e publicacao.

Impacto: protege a plataforma contra o erro comum de tratar documento importado como operacao automaticamente segura para uso agentic.

### Separar operacao HTTP de auth profile

Ganho: aumenta reuso e deixa a autenticacao tratada como contrato proprio.

Custo: cria dependencia entre dois registros governados.

Impacto: melhora manutencao e evita duplicacao de segredos e fluxos de token em cada endpoint.

### Restringir o registro a protocol_type rest_json

Ganho: o runtime falha fechado para protocolos fora do escopo suportado.

Custo: reduz a promessa comercial para APIs que nao caibam nesse modelo.

Impacto: evita vender flexibilidade falsa e protege a coerencia da factory.

## 10. Impacto executivo

dyn_api reduz custo de expansao de integracoes homologadas, melhora governanca do que pode virar capacidade de agente e centraliza auditoria. Isso reduz o risco de backlog tecnico inflado por conectores pequenos e melhora previsibilidade para priorizacao de roadmap.

## 11. Impacto comercial

O impacto comercial esta em demonstrar velocidade com controle. A plataforma pode mostrar que consegue transformar operacoes aprovadas de APIs corporativas em capacidades reutilizaveis sem depender de um desenvolvimento dedicado para cada caso simples. A promessa que deve ser evitada e a de integracao irrestrita com qualquer API sem analise previa.

## 12. Impacto estrategico

O impacto estrategico e consolidar uma plataforma orientada a capacidades governadas. dyn_api fortalece um caminho em que o crescimento do produto acontece por contratos declarativos, catalogos e policies, nao por proliferacao de conectores especiais. Isso prepara o terreno para catálogos maiores por tenant, governanca mais forte e combinacao futura com workflows e agentes mais especializados.

## 13. Explicacao 101

Pense em dyn_api como um jeito de transformar uma chamada HTTP aprovada em um botao seguro para o agente. O agente nao ganha liberdade para sair navegando em qualquer API. Ele recebe um conjunto limitado de operacoes ja descritas, com parametros esperados, autenticacao conhecida e regras de publicacao. Isso acelera a entrega sem perder controle.

## 14. Limites e pegadinhas

- dyn_api nao substitui conectores dedicados quando a integracao exige logica complexa, transformacao pesada ou seguranca fora do escopo declarativo lido.
- dyn_api nao e NL2SQL, nao e dyn_sql e nao e MCP. Ela resolve operacoes HTTP aprovadas, nao geracao de SQL nem acesso generico a tools externas arbitrarias.
- operacao cadastrada nao significa operacao publicada. O codigo exige status ativo e publicacao para agentes.
- operacao importada por Swagger tambem nao significa operacao publicada. A importacao popula catalogo e rastreabilidade, mas nao libera a capability automaticamente.
- autenticacao dinamica nao e detalhe cosmético. Quando usada, depende de cache de token e de infraestrutura Redis disponivel.

## 15. Checklist de entendimento

- Entendi que dyn_api e uma capability governada, nao um executor HTTP livre.
- Entendi que o runtime tenta primeiro YAML local e depois registro persistido por tenant.
- Entendi por que publicacao para agentes e status ativo sao filtros obrigatorios.
- Entendi por que auth profile e separado da operacao HTTP.
- Entendi que a promessa comercial correta e velocidade com governanca, nao liberdade irrestrita.

## 16. Evidencias no codigo

- src/agentic_layer/tools/domain_tools/dynamic_api_tools/dynamic_api_factory.py
  - Motivo da leitura: confirmar a ordem de resolucao, a montagem do schema dinamico e a execucao HTTP.
  - Simbolo relevante: DynamicApiToolFactory.
  - Comportamento confirmado: resolucao local antes do registro persistido e materializacao dinamica de path, query, body e headers.

- src/agentic_layer/tools/domain_tools/dynamic_tool_registry_resolver.py
  - Motivo da leitura: confirmar as regras do fallback governado por tenant.
  - Simbolo relevante: resolve_api_dynamic_section.
  - Comportamento confirmado: tenant obrigatorio, operacao ativa, publicada para agentes e restrita a protocol_type rest_json.

- src/agentic_layer/tools/domain_tools/dynamic_api_tools/auth_manager.py
  - Motivo da leitura: confirmar como a autenticacao dinamica e resolvida.
  - Simbolo relevante: AuthenticationManager.
  - Comportamento confirmado: cache de token, resolucao de placeholders e dependencia de Redis.

- src/api/services/admin/integrations_service.py
  - Motivo da leitura: confirmar o bounded context administrativo que publica operacoes HTTP e auth profiles.
  - Simbolo relevante: create_api_operation e create_auth_profile.
  - Comportamento confirmado: o catalogo governado separa operacao HTTP e autenticacao reutilizavel.

- src/integrations/schema.py
  - Motivo da leitura: confirmar as tabelas reais do slice dyn_api.
  - Simbolo relevante: api_operation_registry e api_swagger_source_registry.
  - Comportamento confirmado: as operacoes HTTP governadas e suas origens Swagger possuem tabelas dedicadas e relacionadas.

- src/integrations/swagger_import_service.py
  - Motivo da leitura: confirmar o comportamento da importacao Swagger.
  - Simbolo relevante: SwaggerImportApplicationService.execute.
  - Comportamento confirmado: a importacao cria ou atualiza origem e operacoes de forma idempotente, mas grava publish_to_agents=false nas operacoes importadas.

## 17. Lacunas reais encontradas

- Nao foi confirmada no codigo lido uma camada unica de observabilidade administrativa que correlacione, na mesma tela, endpoint, auth profile, teste operacional e status de publicacao.
- Nao foi confirmado no codigo lido um inventario consolidado pronto para negocio mostrando, por tenant, todas as capacidades dyn_api ja publicadas para agentes.
