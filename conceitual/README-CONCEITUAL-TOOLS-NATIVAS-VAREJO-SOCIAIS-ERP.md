# Manual conceitual, executivo, comercial e estratégico: tools nativas para varejo, sociais e ERP

## 1. O que é esta feature

As tools nativas desta plataforma são o catálogo builtin de capacidades que o runtime já sabe materializar sem depender de cadastro manual no YAML de cada agente.

Em linguagem simples, elas são a prateleira oficial de ações que os agentes e workflows podem usar para operar canais, marketplaces, e-commerce, atendimento, CRM e dados corporativos. O catálogo é global, persistido em banco e injetado automaticamente quando o YAML chega com `tools_library: []`.

O ponto central deste manual é o seguinte: o repositório já contém um número grande e variado de famílias de tools nativas. Elas não se limitam a busca web ou utilidades genéricas. Já existem superfícies concretas para varejo, canais sociais e operação orientada a ERP, o que muda a conversa de produto. A plataforma não começa do zero a cada caso. Ela já nasce com uma base operacional ampla.

## 2. Que problema ela resolve

Sem catálogo nativo, toda iniciativa agentic cairia no mesmo problema.

1. Cada novo agente precisaria redescrever manualmente suas ferramentas.
2. Cada tenant teria tendência a divergir do código real da plataforma.
3. O time perderia governança sobre o que está ativo, desativado ou obsoleto.
4. O produto ficaria parecendo coleção de integrações isoladas, não plataforma.

As tools nativas resolvem isso ao separar três decisões que não devem ficar misturadas.

1. O que a plataforma sabe fazer de forma builtin.
2. O que está operacionalmente ativo no catálogo persistido.
3. O que cada agente ou workflow concreto decide usar.

## 3. Visão executiva

Para liderança, o valor não é apenas “tem muitas tools”. O valor é que a empresa já possui uma base de capacidades reutilizáveis para varejo, canais sociais e ERP, governadas por catálogo central e prontas para compor soluções diferentes com menos código novo.

Isso reduz tempo de implantação, diminui risco de drift entre times, acelera propostas comerciais e fortalece a percepção de produto de plataforma.

## 4. Visão comercial

Comercialmente, essas tools permitem vender casos de uso concretos em vez de promessas vagas de IA.

Exemplos de narrativa comercial suportada pelo código lido:

1. Agente que monitora eventos e pedidos de iFood, Goomer, Shopify Foodservice e Zé Delivery.
2. Agente que responde ou notifica clientes em WhatsApp, Instagram e Chatwoot.
3. Agente que capta lead no canal social e publica a oportunidade no RD Station.
4. Agente que consulta dados governados do ERP por `dyn_sql`, `dyn_api`, `proc_sql` ou `schema_rag_sql`.
5. Workflow que cruza pedido, catálogo, estoque, CRM e atendimento sem reinventar integração a cada fluxo.

O diferencial aqui é concreto: a plataforma já tem conectores tool-first para partes críticas da operação comercial e de varejo, não apenas um chat com documentos.

## 5. Visão estratégica

Estrategicamente, esse catálogo builtin faz a plataforma ganhar densidade operacional.

Uma plataforma agentic madura não vive só de LLM. Ela precisa de superfícies reais de ação e consulta. As tools nativas cumprem esse papel.

1. Varejo: ligam agentes ao mundo transacional e operacional.
2. Sociais: ligam agentes ao mundo conversacional e de relacionamento.
3. ERP e dados governados: ligam agentes ao backoffice, ao banco, às APIs e às consultas corporativas.

Isso fortalece a arquitetura porque o agente deixa de ser só um sintetizador de texto e passa a ser um coordenador governado de capacidades de plataforma.

## 6. O que são tools nativas de forma geral

No código lido, “tool nativa” significa capability builtin publicada pelo próprio produto. Ela pode aparecer de dois jeitos.

1. Tool direta: aponta para uma implementação concreta.
2. Tool por factory: publica uma família e gera uma ou mais tools em runtime.

Essa distinção importa porque boa parte do poder do sistema vem justamente de families parametrizadas. Em vez de uma tool única fixa, a plataforma registra uma entrada canônica e o runtime especializa conforme o contexto.

Isso é muito visível em `dyn_sql`, `dyn_api`, `proc_sql` e `schema_rag_sql`, mas também aparece em toolkits completos, como os de varejo, e-commerce, mensageria e CRM.

## 7. Por que já existe um grande número de tools nativas

O builder oficial percorre recursivamente `src/agentic_layer/tools` e encontra tools distribuídas por múltiplas famílias confirmadas no código.

1. `system_tools`
2. `external_tools`
3. `domain_tools`
4. `vendor_tools`
5. `custom_tools`
6. `integration_tools`

Só esse desenho estrutural já mostra que a plataforma não trabalha com um catálogo curto e artesanal. Ela já foi organizada para suportar muitas famílias especializadas, inclusive por domínio de negócio.

O pedido deste manual faz sentido exatamente por isso: já existe massa crítica suficiente para documentar ferramentas de varejo, sociais e ERP como blocos estratégicos, e não como exceções isoladas.

## 8. O que a tabela de tools representa

O catálogo builtin é persistido na tabela `integrations.builtin_tool_registry`.

Conceitualmente, essa tabela responde quatro perguntas.

1. Que capability nativa existe.
2. Se ela é direta ou gerada por factory.
3. Qual o estado operacional da capability.
4. Qual a trilha administrativa dessa entrada.

Em linguagem simples, essa tabela não guarda “qual agente chamou o quê”. Ela guarda a prateleira oficial de tools da plataforma.

Isso tem impacto prático importante.

1. A plataforma sabe listar o catálogo builtin por API administrativa.
2. A operação pode desativar uma family problemática sem editar YAML de cliente.
3. O runtime consegue injetar automaticamente apenas as tools ativas.
4. O produto evita YAMLs com catálogo manual e divergente.

## 9. Ferramentas nativas para varejo

O bloco de varejo é um dos mais fortes do repositório porque liga agentes a pedido, estoque, catálogo, marketplaces e hubs de venda.

### 9.1. iFood

Existem duas superfícies complementares.

1. Um toolkit operacional completo, com autenticação, fila de eventos, ACK de eventos, consulta de pedido, atualização de status, relatório financeiro e histórico de status.
2. Um toolkit baseado em SDK, mais enxuto, focado em busca e confirmação de eventos.

Na prática, isso permite agentes que:

1. leem a fila de eventos do iFood;
2. confirmam eventos consumidos;
3. consultam detalhes de um pedido específico;
4. atualizam a etapa operacional do pedido;
5. verificam impacto financeiro por período.

Caso real de uso:

Um workflow de conciliação pode capturar eventos de pedido, checar divergência no ERP, solicitar aprovação humana em caso de exceção e só então confirmar o evento para evitar perda operacional.

### 9.2. Goomer

O toolkit de Goomer vai além da leitura de pedidos. Ele expõe operações sobre cardápio, itens, disponibilidade, consulta de pedidos, atualização de status e disponibilidade em lote.

Isso o torna especialmente forte para food service e operação omnichannel.

Caso real de uso:

Um agente pode detectar ruptura de estoque no ERP, disparar um workflow que recalcula disponibilidade, atualizar vários itens no Goomer em lote e avisar o time de loja no canal interno.

### 9.3. Shopify Foodservice

O toolkit especializado de Shopify para food service cobre visão operacional de pedidos, ajuste de estoque, publicação ou ocultação de produto e resumo agregado de vendas.

Isso é relevante porque food service precisa de operação rápida, não apenas catálogo bonito. O toolkit já conversa com pedido, disponibilidade e resumo comercial.

Caso real de uso:

Um agente de operação de delivery pode, ao perceber aumento de fila e falta de insumo, reduzir oferta de determinados produtos, ajustar estoque e devolver um resumo das vendas do dia para apoiar a decisão do gerente.

### 9.4. Zé Delivery

O toolkit confirmado no código é mais enxuto e focado em busca de pedidos. Mesmo assim, ele é útil porque fornece um ponto nativo de leitura operacional para acompanhamento de pedidos por status ou termo.

Caso real de uso:

Um agente de atendimento pode localizar rapidamente pedidos atrasados de determinado status, cruzar com o canal de atendimento e responder ao operador com a situação real do pedido.

### 9.5. Toolkit genérico de e-commerce

Há também um toolkit amplo para Shopify, WooCommerce e VTEX. Ele cobre produto, pedido, cliente, estoque, atualização de status e health check de integração.

Esse toolkit não é apenas “marketplace”. Ele é relevante para varejo porque cobre o coração do comércio digital.

Caso real de uso:

Um workflow pode receber um alerta de baixa cobertura de estoque, consultar catálogo e pedidos em Shopify, WooCommerce ou VTEX, atualizar disponibilidade e registrar a ação em outro sistema corporativo.

### 9.6. Plugg.to e Magalu Hub

Esses toolkits confirmados no código são especializados em busca de produtos por nome, SKU ou categoria. São superfícies menores, mas com uso operacional claro.

Caso real de uso:

Um agente de catálogo pode receber uma pergunta em linguagem natural como “esse SKU já está publicado no hub?” e usar as tools para localizar o produto sem exigir navegação manual do operador.

## 10. Ferramentas nativas para sociais e canais de relacionamento

O bloco social é estratégico porque IA em varejo raramente fica só no backoffice. Ela precisa aparecer em WhatsApp, Instagram, inbox operacional e CRM.

### 10.1. WhatsApp Cloud

O toolkit nativo de WhatsApp confirmado no código cobre envio de mensagem de texto e envio de mensagem por template.

Isso é suficiente para muitos fluxos reais de operação, porque o gargalo frequente não é criatividade de canal; é entrega governada e rastreável da mensagem certa.

Caso real de uso:

Um workflow pode identificar atraso de pedido no ERP ou no hub de delivery, gerar mensagem padronizada com template aprovado e disparar comunicação ao cliente sem depender de operador manual em cada caso.

### 10.2. Instagram Graph

O toolkit de Instagram cobre publicação de mídia, envio de mensagem direta e consulta de insights.

Esse trio é forte porque cobre operação, relacionamento e análise.

Caso real de uso:

Um agente pode publicar uma mídia promocional, acompanhar o desempenho via insights e acionar mensagem direta para leads qualificados em um fluxo supervisionado.

### 10.3. Chatwoot

O toolkit de Chatwoot cobre conversas, mensagens, atualização de conversa, contatos, agentes e health check.

Isso é especialmente relevante em operação omnichannel porque permite acoplar o agente ao inbox onde a equipe humana realmente trabalha.

Caso real de uso:

Um agente de suporte pode listar conversas pendentes, priorizar tickets com base em contexto vindo do ERP, responder no thread certo e marcar a conversa para outro grupo quando detectar necessidade de backoffice.

### 10.4. RD Station

O toolkit de RD Station cobre criação ou atualização de lead, criação de negócio e registro de nota.

Ele transforma o agente de atendimento ou social selling em agente comercial rastreável.

Caso real de uso:

Um fluxo pode começar em Instagram ou WhatsApp, qualificar um lead, criar ou atualizar esse lead no RD Station, abrir negócio e registrar nota com contexto operacional da conversa.

## 11. Ferramentas nativas voltadas ao mundo de ERP e dados corporativos

Quando falamos em ERP neste repositório, o valor não está apenas em um conector com um nome de fornecedor. O núcleo confirmado no código está em tools governadas de consulta, integração e execução controlada sobre os sistemas corporativos.

### 11.1. dyn_sql

`dyn_sql` é a capability para materializar queries governadas por registro ou YAML. Ela usa sintaxe parametrizada como `dyn_sql<query_id>`.

Valor prático:

1. evita SQL solta no prompt;
2. exige query previamente governada;
3. permite reaproveitar consultas de ERP em agentes e workflows.

Caso real de uso:

Um agente financeiro pode usar `dyn_sql<inadimplencia_por_filial>` para responder perguntas recorrentes da operação sem abrir acesso livre ao banco.

### 11.2. dyn_api

`dyn_api` faz papel parecido para endpoints HTTP governados, usando sintaxe como `dyn_api<endpoint_id>`.

Valor prático:

1. o agente chama operação pré-governada;
2. o endpoint continua administrável fora do prompt;
3. o time pode publicar ou retirar operações sem redesenhar cada agente.

Caso real de uso:

Um workflow de pós-venda pode consultar uma API de ERP ou backoffice para status de pedido, regra comercial, limite de crédito ou cadastro de cliente usando uma operação já homologada.

### 11.3. proc_sql

`proc_sql` é a capability parametrizada para procedures governadas, com sintaxe como `proc_sql<procedure_id>`.

Ela é importante em ERP porque muitos processos corporativos críticos continuam encapsulados em procedures.

Caso real de uso:

Um workflow de fechamento de caixa pode preparar contexto, pedir aprovação humana e só então chamar uma procedure governada do ERP para concluir a rotina.

### 11.4. schema_rag_sql

`schema_rag_sql` é a tool canônica de geração de SQL a partir de linguagem natural usando metadados de schema indexados.

Ela não substitui `dyn_sql`. Os dois papéis são diferentes.

1. `dyn_sql`: executa ou materializa consultas previamente governadas.
2. `schema_rag_sql`: ajuda a propor SQL para perguntas novas a partir do schema conhecido.

Caso real de uso:

Um copiloto de ERP pode receber uma pergunta inédita de um consultor, recuperar metadados de schema, propor uma SQL revisável e devolver isso para auditoria humana antes de qualquer execução.

## 12. Como agentes e workflows usam essas tools na vida real

### 12.1. Agente de operação de delivery

Fluxo realista:

1. Lê eventos do iFood.
2. Consulta pedido detalhado.
3. Cruza o contexto com ERP ou backoffice via `dyn_api` ou `dyn_sql`.
4. Se houver ruptura, ajusta disponibilidade no Goomer ou Shopify Foodservice.
5. Envia aviso ao cliente no WhatsApp.
6. Registra ticket ou thread no Chatwoot.

Esse é o tipo de fluxo que mostra que a plataforma não trata varejo só como FAQ. Ela trata varejo como operação.

### 12.2. Workflow de social selling

Fluxo realista:

1. Recebe sinal de interação em Instagram.
2. Um agente qualifica o contexto.
3. Usa RD Station para criar ou atualizar lead.
4. Abre negócio e registra nota.
5. Se necessário, continua a conversa pelo WhatsApp ou Chatwoot.

Esse fluxo une canal social, CRM e governança comercial em cima do mesmo runtime.

### 12.3. Copiloto de ERP para suporte e consultoria

Fluxo realista:

1. Usuário pergunta em linguagem natural sobre indicador ou processo.
2. O agente decide entre `dyn_sql`, `dyn_api` ou `schema_rag_sql`.
3. Se a resposta exigir ação sensível, o workflow pede HIL.
4. A decisão humana aprova ou rejeita a continuação.
5. O run segue com rastreabilidade ponta a ponta.

Esse é o desenho que transforma um ERP tradicional em superfície agentic governada.

## 13. Impacto técnico

Essas tools reduzem acoplamento porque a lógica de integração fica concentrada em toolkits e families governadas, não espalhada em prompts e agentes ad hoc.

Também aumentam reuso, observabilidade e previsibilidade de implantação.

## 14. Impacto executivo

Para liderança, o efeito mais importante é reduzir o custo marginal de novos casos de uso. Em vez de reconstruir integrações e capacidades a cada proposta, a empresa reaproveita catálogo nativo já consolidado.

## 15. Impacto comercial

Para vendas e pré-vendas, o ganho é transformar IA em portfólio repetível. A conversa deixa de ser “podemos estudar uma integração” e passa a ser “já temos famílias nativas para esse domínio e sabemos combiná-las em agentes e workflows”.

## 16. Impacto estratégico

O catálogo builtin de tools para varejo, sociais e ERP fortalece o posicionamento da plataforma como camada industrial de agentes para software houses. Ele aproxima o produto do centro da operação do cliente, e não apenas da borda experimental de IA.

## 17. Explicação 101

Pense na plataforma como um centro operacional de agentes. O LLM é só a parte que entende ou redige linguagem. Quem realmente permite trabalhar com o mundo real são as tools.

As tools de varejo conectam o agente a pedido, estoque e catálogo.

As tools sociais conectam o agente a mensagem, lead e relacionamento.

As tools de ERP conectam o agente a banco, API e processo corporativo governado.

Sem isso, o agente fala. Com isso, ele consulta, coordena, sugere, executa e pede aprovação quando precisa.

## 18. Limites e pegadinhas

1. O fato de a tool existir no catálogo não significa que toda credencial esteja configurada em todo tenant.
2. `schema_rag_sql` não deve ser tratado como execução automática irrestrita de SQL.
3. `dyn_sql`, `dyn_api` e `proc_sql` dependem de governança prévia de query, endpoint ou procedure.
4. Algumas families são amplas, como e-commerce; outras são mais enxutas, como Zé Delivery e hubs de busca de produto.
5. O catálogo builtin é global, mas o uso real continua dependente de contexto, permissões, segredos e configuração resolvida do tenant.

## 19. Checklist de entendimento

- Entendi o que são tools nativas.
- Entendi por que já existe uma base grande de tools builtin na plataforma.
- Entendi o papel da tabela `integrations.builtin_tool_registry`.
- Entendi a diferença entre tools de varejo, sociais e ERP.
- Entendi os principais casos de uso reais dessas families.
- Entendi por que essas tools fortalecem agentes e workflows.
- Entendi os limites de governança e configuração.

## 20. Evidências no código

- `src/agentic_layer/tools/tool_factory_decorator.py`
  - Motivo da leitura: confirmar o contrato oficial de publicação por factory.
  - Símbolo relevante: `tool_factory`.
  - Comportamento confirmado: a plataforma marca factories builtin com metadados para publicação no catálogo.

- `src/agentic_layer/tools/tools_library_builder.py`
  - Motivo da leitura: confirmar descoberta automática e famílias parametrizadas.
  - Símbolo relevante: `ToolsLibraryBuilder`.
  - Comportamento confirmado: o builder descobre tools no diretório oficial e adiciona explicitamente `dyn_sql`, `dyn_api` e `proc_sql`.

- `src/integrations/schema.py`
  - Motivo da leitura: confirmar a tabela de catálogo builtin.
  - Símbolo relevante: `_ddl_builtin_tool_registry`.
  - Comportamento confirmado: a fonte de verdade persistida do catálogo builtin é `integrations.builtin_tool_registry`.

- `src/agentic_layer/tools/domain_tools/food_delivery_tools/ifood_tools.py`
  - Motivo da leitura: confirmar toolkit de iFood legado e suas operações.
  - Símbolo relevante: `create_ifood_tools`.
  - Comportamento confirmado: o toolkit cobre autenticação, eventos, ACK, pedido, status e financeiro.

- `src/agentic_layer/tools/domain_tools/food_delivery_tools/goomer_tools.py`
  - Motivo da leitura: confirmar o toolkit operacional de Goomer.
  - Símbolo relevante: `create_goomer_tools`.
  - Comportamento confirmado: o toolkit cobre catálogo, disponibilidade, pedidos e atualização operacional.

- `src/agentic_layer/tools/vendor_tools/whatsapp_tools/whatsapp_toolkit.py`
  - Motivo da leitura: confirmar o toolkit de WhatsApp.
  - Símbolo relevante: `create_whatsapp_cloud_tools`.
  - Comportamento confirmado: o toolkit publica envio de texto e template.

- `src/agentic_layer/tools/vendor_tools/instagram_tools/instagram_toolkit.py`
  - Motivo da leitura: confirmar o toolkit de Instagram.
  - Símbolo relevante: `create_instagram_tools`.
  - Comportamento confirmado: o toolkit cobre publicação, DM e insights.

- `src/agentic_layer/tools/domain_tools/dynamic_sql_tools/dynamic_sql_factory.py`
  - Motivo da leitura: confirmar a capability ERP de SQL governado.
  - Símbolo relevante: `create_dynamic_sql_tools`.
  - Comportamento confirmado: `dyn_sql` funciona por parametrização inline e resolução governada.

- `src/agentic_layer/tools/domain_tools/dynamic_api_tools/dynamic_api_factory.py`
  - Motivo da leitura: confirmar a capability ERP de API governada.
  - Símbolo relevante: `create_dynamic_api_tools`.
  - Comportamento confirmado: `dyn_api` materializa endpoints governados por parametrização inline.

- `src/agentic_layer/tools/domain_tools/schema_rag_tools/sql_schema_rag_factory.py`
  - Motivo da leitura: confirmar a tool canônica de NL2SQL assistido por schema.
  - Símbolo relevante: `create_sql_schema_rag_tool`.
  - Comportamento confirmado: `schema_rag_sql` é entrada canônica do catálogo para geração de SQL a partir de metadados indexados.
