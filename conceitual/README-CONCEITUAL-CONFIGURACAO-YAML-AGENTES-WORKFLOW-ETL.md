# Manual conceitual, executivo, comercial e estratégico: configuração YAML total de agentes, workflows e ETL sem necessidade de programação

## 1. O que é esta feature

Esta feature é a capacidade da plataforma de ser montada, escolhida, validada e publicada por configuração declarativa, sem exigir edição de Python para cada novo caso de uso. Em vez de tratar YAML como simples arquivo de parâmetros, o produto o trata como contrato de montagem do sistema.

O ponto central não é apenas “usar YAML”. O ponto central é que o mesmo documento declarativo pode definir:

1. Qual workflow deve ser o fluxo ativo.
2. Qual DeepAgent deve ser o orquestrador ativo quando o caso nao for WorkflowAgent.
3. Quais tools builtin entram no runtime.
4. Quais pipelines ETL ficam ligados ou desligados.
5. Quais credenciais, tenant, índices, caches e políticas operacionais entram na execução.

Em linguagem simples, a plataforma foi desenhada para que grande parte da personalização aconteça por montagem declarativa do YAML e não por bifurcação contínua do código.

## 2. Que problema ela resolve

Sem esse desenho, todo ajuste importante de agentes, workflows e ETL cairia em uma rotina cara e ruim.

1. O cliente pediria uma nova combinação de fluxo.
2. O time teria de alterar Python para trocar seleção, parâmetros ou topologia.
3. O runtime acumularia caminhos paralelos, ifs por cliente e comportamento escondido.
4. A governança cairia, porque configuração e comportamento deixariam de conversar de forma transparente.

O contrato YAML resolve isso ao deslocar a maior parte da customização para uma camada declarativa governada. Isso permite operar em um nível mais alto: escolher capacidades prontas, variar comportamento, ativar ou desativar partes do produto e publicar um runtime diferente sem reescrever o core.

## 3. Visão executiva

Executivamente, essa feature importa porque reduz custo marginal de personalização. O produto fica mais escalável quando uma equipe consegue configurar a solução para um novo cliente, uma nova automação ou um novo fluxo operacional sem depender de sprint de desenvolvimento para cada variação.

Isso produz quatro ganhos diretos.

1. Menor dependência de desenvolvimento para ajustes frequentes.
2. Implantação mais rápida entre tenants e cenários.
3. Menor risco de divergência entre clientes, porque o core continua reaproveitado.
4. Governança melhor, porque a plataforma valida e compila a configuração antes da execução.

O valor executivo não está em “eliminar código”. Está em reduzir a quantidade de mudanças que realmente precisam de código.

## 4. Visão comercial

Comercialmente, essa capacidade sustenta uma proposta forte: a plataforma pode ser adaptada por configuração para diferentes cenários de agentes, workflows e ETL, sem transformar cada cliente em um projeto de software quase novo.

Isso ajuda em conversas de pré-venda e expansão porque permite dizer, com base no código real, que o produto já suporta:

1. Escolha declarativa do workflow ativo.
2. Escolha declarativa do DeepAgent ativo.
3. Montagem declarativa de ferramentas builtin no runtime.
4. Ativação declarativa de pipelines ETL já existentes.
5. Parametrização de tenant, segredos e recursos operacionais por YAML.

O que não pode ser prometido é que qualquer capacidade nova nasce sem programação. Se o cliente pedir uma tool inédita, um tipo novo de node ou um pipeline ETL que o produto ainda não possui, isso continua exigindo desenvolvimento.

## 5. Visão estratégica

Estratégicamente, essa feature fortalece a plataforma por cinco razões.

1. Reforça a arquitetura YAML-first sem cair em texto livre sem governança.
2. Permite que consultoria, operação e produto trabalhem em uma camada mais alta do que Python.
3. Preserva o core técnico ao deslocar variação para configuração validada.
4. Conecta AST, validadores e runtime em uma cadeia única de verdade.
5. Prepara o produto para escalar casos de uso sem multiplicar ramificações de código por cliente.

Em linguagem simples, essa feature aumenta a capacidade de montar soluções com mais velocidade e menos retrabalho estrutural.

## 6. Conceitos necessários para entender

### 6.1. YAML-first

YAML-first significa que a plataforma nasce de configuração declarativa. Não significa que qualquer YAML seja aceito. O documento só vira runtime depois de passar por carga, normalização, validação e, no escopo agentic, por AST tipada.

### 6.2. Configuração total sem programação

Aqui, “sem programação” significa montar o comportamento usando capacidades que ja existem no produto. Isso inclui escolher WorkflowAgent, DeepAgent, tools e pipelines ETL ja implementados. Nao significa inventar uma nova capacidade do zero sem codigo.

### 6.3. selected_workflow

É a chave que escolhe qual workflow habilitado do documento será o ativo em runtime. Ela existe para evitar ambiguidade quando o YAML traz mais de um fluxo possível.

### 6.4. selected_supervisor

E a chave historica que escolhe qual entrada DeepAgent habilitada sera a ativa em runtime. Ela existe para evitar que o sistema “adivinhe” o orquestrador correto.

### 6.5. multi_agents

E a espinha dorsal declarativa usada pelo DeepAgent no caminho oficial. Se um documento antigo ainda trouxer a trilha anterior sem `execution.type=deepagent`, isso deve ser tratado como compatibilidade legada e nao como modelagem nova.

### 6.6. workflows

É a coleção declarativa de workflows. Cada item descreve fluxo, nodes, edges, settings e outras partes do contrato executável.

### 6.7. tools_library

É a biblioteca de tools visível para o runtime agentic. O detalhe crítico é que ela não deve ser preenchida manualmente com builtin tools no YAML recebido. O sistema injeta esse catálogo automaticamente.

### 6.8. extract_transform_load

É o bloco declarativo que governa o domínio de ETL. Ele define se o ETL está ligado, quais famílias entram em execução e quais pipelines concretos já implementados ficam ativos.

### 6.9. AST governada

No escopo agentic, o produto não trata o YAML apenas como texto. Ele o converte para AST tipada, valida semanticamente e só depois confirma a versão publicável. Isso protege a plataforma contra YAML sintaticamente bonito, mas semanticamente errado.

## 7. Como a feature funciona por dentro

O fluxo começa quando um operador, consultor ou processo administrativo fornece um YAML base ou escolhe um template. Esse documento passa pela fábrica de configuração, que injeta correlation_id, tenta completar contexto de sessão, garante security_keys como store lógico e injeta tools_library builtin quando o contrato agentic exige isso.

Se o caso envolver workflows ou deepagents, entra a camada de assembly agentic. Ela resolve o alvo, faz parse para AST, valida o documento e pode ainda rodar draft, validate e confirm por API administrativa. Isso significa que a configuracao agentic nao depende de editar Python para escolher o fluxo ou o DeepAgent ativo. Ela depende de preencher corretamente o contrato declarativo.

Se o caso envolver ETL, o runtime procura o bloco extract_transform_load. Se enabled estiver verdadeiro e ao menos um subsistema estiver ativo, o serviço chama o orquestrador dedicado. O orquestrador só executa pipelines explicitamente ligados no YAML.

O resultado prático é que o mesmo produto consegue ser remontado por configuração em três domínios diferentes.

1. Fluxo determinístico por workflow.
2. Orquestracao agentic por DeepAgent.
3. Transformação estruturada por ETL.

## 8. Divisão por capacidades configuráveis

### 8.1. Configuração total de workflows

O YAML pode escolher o workflow ativo, transportar defaults da coleção, declarar nodes, edges e settings, e publicar esse contrato para o runtime sem reescrever Python. O ganho é controlar automação determinística por configuração.

### 8.2. Configuracao total de DeepAgent

O YAML pode escolher o DeepAgent ativo, declarar multi_agents, controlar execucao deepagent e expor ferramentas e politicas ja existentes no runtime. O ganho e variar topologia agentic sem criar um orquestrador novo em codigo para cada caso.

### 8.3. Configuração total de ETL existente

O YAML pode ligar ou desligar extract_transform_load, ativar famílias como Apify e schema_metadata e parametrizar pipelines concretos já disponíveis. O ganho é controlar ingestão estruturada e preparação técnica por configuração.

### 8.4. Geração e validação assistidas

Além do arquivo YAML em si, a plataforma já tem endpoints para gerar configuração a partir de template, fazer draft de AST, validar, confirmar e até converter objetivo em YAML governado. Isso reduz ainda mais a necessidade de programação para quem opera dentro das capacidades já existentes.

## 9. O que realmente pode ser feito sem programação

Com base no código lido, estas mudanças podem ser feitas sem editar Python, desde que a capacidade já exista no produto.

1. Selecionar o workflow ativo.
2. Selecionar o DeepAgent ativo.
3. Ativar ou desativar workflows, DeepAgents e pipelines ETL ja implementados.
4. Ajustar parâmetros de execução, credenciais e tenant via YAML.
5. Trocar o target vetorial, segredos e recursos do runtime já suportados.
6. Publicar fragmento agentic governado via validate e confirm.
7. Gerar configuração final a partir de template e overrides.

## 10. O que ainda exige programação

O código também deixa claro onde a promessa “sem programação” termina.

1. Criar uma tool nova continua exigindo Python e sincronização do catálogo builtin.
2. Criar um tipo novo de node de workflow continua exigindo Python no runtime e na AST.
3. Criar um pipeline ETL totalmente novo continua exigindo implementação concreta.
4. Criar lógica de negócio inédita fora das capacidades existentes continua exigindo código.

Esse limite não é defeito. É um guardrail arquitetural. Ele impede que a plataforma prometa configuração onde ainda não existe engine real por trás.

## 11. Decisões técnicas e trade-offs

### 11.1. Configurar o que já existe, não o que ainda não existe

Ganho: evita YAML enganoso que promete uma engine inexistente.

Custo: reduz a liberdade de quem gostaria de “inventar” capacidade nova só por configuração.

### 11.2. Injetar tools builtin automaticamente

Ganho: evita duplicidade, drift e declaração manual do catálogo.

Custo: exige disciplina para enviar tools_library vazia e deixar o runtime completar o catálogo.

### 11.3. Governar o escopo agentic por AST

Ganho: reduz erro semântico e protege a execução.

Custo: a configuração agentic fica mais rígida e menos tolerante a atalhos textuais.

### 11.4. Falhar cedo no ETL

Ganho: evita pipeline opaco rodando com bloco vazio ou mal configurado.

Custo: o operador precisa completar o contrato de extract_transform_load antes de testar.

## 12. O que acontece em caso de sucesso

No caminho feliz, o YAML e carregado, normalizado, enriquecido com contexto de sessao, passa pela validacao adequada ao dominio e vira runtime efetivo sem exigir mudanca no codigo Python. O workflow selecionado passa a ser o fluxo ativo, o DeepAgent selecionado passa a ser o orquestrador ativo e o ETL passa a executar apenas os pipelines habilitados.

## 13. O que acontece em caso de erro

Os erros mais relevantes confirmados no código lido são estes.

1. YAML top-level que não é objeto.
2. Falta ou ambiguidade de workflow ativo.
3. selected_supervisor incompatível com os DeepAgents disponíveis.
4. tools_library ausente ou preenchida indevidamente quando o contrato agentic espera auto-injeção.
5. extract_transform_load ausente, disabled ou sem subsistemas ativos.
6. Target agentic inválido ou feature de AST desabilitada no endpoint administrativo.

O ponto conceitual importante é este: a plataforma prefere erro explícito a adivinhação silenciosa.

## 14. Observabilidade e diagnóstico

Para investigar um problema nesse domínio, a primeira pergunta correta não é “qual arquivo Python eu tenho de editar”. A pergunta correta é “qual parte do contrato YAML não fechou?”.

A ordem mais segura de diagnóstico é esta.

1. Confirmar o target da montagem, como workflow ou deepagent.
2. Confirmar se selected_workflow ou selected_supervisor apontam para um item real e habilitado.
3. Confirmar se tools_library chegou no formato esperado.
4. Confirmar se extract_transform_load.enabled e os subsistemas corretos estão ativos.
5. Só depois investigar engine, provider externo ou runtime específico.

## 15. Impacto técnico

Tecnicamente, essa feature reduz acoplamento entre customização e código, centraliza a verdade operacional no YAML governado e melhora a consistência entre geração, validação e execução.

## 16. Impacto executivo

Executivamente, essa feature acelera implantação, reduz dependência de desenvolvimento para ajustes frequentes e torna mais previsível o custo de personalização.

## 17. Impacto comercial

Comercialmente, ela torna mais crível a proposta de configurar soluções diferentes sobre a mesma plataforma, sem transformar cada cliente em um fork de produto.

## 18. Impacto estratégico

Estratégicamente, ela prepara a plataforma para crescer com mais variações operacionais sem crescer na mesma proporção em complexidade de código de manutenção.

## 19. Exemplos práticos guiados

### 19.1. Cliente que precisa trocar o fluxo ativo

Cenário: o cliente já possui mais de um workflow no documento, mas quer promover outro fluxo como principal.

A mudança fica restrita ao contrato YAML, pela seleção do workflow ativo, desde que os workflows já existam e sejam válidos.

### 19.2. Operação que precisa trocar o orquestrador agentic

Cenario: um tenant quer ativar um DeepAgent ja implementado no lugar de uma configuracao antiga.

Essa troca pode acontecer por seleção declarativa e compatibilidade estrutural do bloco multi_agents, sem necessidade de editar Python, desde que o deepagent já exista no documento e no runtime.

### 19.3. Time de dados que precisa ativar ETL de schema metadata

Cenário: o cliente quer preparar metadados de schema para uso posterior em NL2SQL.

A ativação ocorre no bloco extract_transform_load, ligando schema_metadata e preenchendo origem e destino. O ganho é configurar a esteira pronta sem criar pipeline novo em código.

## 20. Explicação 101

Pense na plataforma como uma fábrica com máquinas já instaladas. O YAML é o painel de configuração que decide quais máquinas entram na linha, em que ordem e com quais parâmetros. O painel não constrói uma máquina nova do nada. Mas, para as máquinas já existentes, ele consegue mudar bastante o comportamento sem chamar a equipe de engenharia para reescrever o chão de fábrica.

## 21. Limites e pegadinhas

1. “Sem programação” não significa “sem validação”.
2. “Sem programação” não significa “capacidade ilimitada”.
3. Tools builtin não devem ser declaradas manualmente no YAML recebido.
4. selected_workflow e selected_supervisor evitam ambiguidade, não são enfeite opcional em todo cenário.
5. ETL declarativo só cobre pipelines que já existem no código.
6. AST governada é proteção, não burocracia extra.

## 22. Troubleshooting

### 22.1. Sintoma: o YAML parece válido, mas o runtime agentic rejeita

Causa provável: o documento é sintaticamente válido, mas semanticamente inválido para o alvo escolhido.

### 22.2. Sintoma: a plataforma não executa o ETL mesmo com o bloco presente

Causa provável: extract_transform_load.enabled está falso ou nenhum subsistema real foi ativado.

### 22.3. Sintoma: o supervisor errado entra em execução

Causa provável: selected_supervisor ausente, incompatível ou ambíguo diante dos DeepAgents habilitados.

### 22.4. Sintoma: a configuração agentic não enxerga as tools esperadas

Causa provável: tools_library não chegou vazia para auto-injeção ou o catálogo builtin ativo não contém as tools desejadas.

## 23. Checklist de entendimento

- Entendi o que a plataforma realmente configura por YAML.
- Entendi o que ainda exige programação.
- Entendi a diferenca entre WorkflowAgent, DeepAgent e ETL.
- Entendi por que tools_library é injetada e não preenchida manualmente.
- Entendi como selected_workflow e selected_supervisor controlam a escolha ativa.
- Entendi por que extract_transform_load governa o ETL.
- Entendi por que AST e validadores existem.

## 24. Evidências no código

- src/config/config_cli/configuration_factory.py
  - Motivo da leitura: confirmar carga do YAML, injeção de sessão e tools_library.
  - Comportamento confirmado: a configuração é finalizada com contexto de sessão, security_keys e auto-injeção do catálogo builtin.

- src/config/agentic_assembly/assembly_service.py
  - Motivo da leitura: confirmar draft, validate, confirm e objetivo para YAML no escopo agentic.
  - Comportamento confirmado: a plataforma já possui fluxo assistido para montar e publicar YAML governado.

- src/config/agentic_assembly/target_scope_resolver.py
  - Motivo da leitura: confirmar escolha do workflow ativo e do membro ativo da familia supervisor.
  - Comportamento confirmado: selected_workflow e selected_supervisor controlam trilhos diferentes do runtime agentic, com fallback restrito ao que está habilitado e compatível.

- src/config/agentic_assembly/validators/document_validator.py
  - Motivo da leitura: confirmar validação semântica por alvo.
  - Comportamento confirmado: workflow e deepagent passam por validadores especificos no caminho oficial antes da confirmacao. Workflow permanece backbone separado e configuracoes antigas sem deepagent devem ser tratadas como migracao.

- src/services/etl_service.py
  - Motivo da leitura: confirmar ativação e falha cedo do ETL declarativo.
  - Comportamento confirmado: o ETL só executa quando extract_transform_load está presente, ativo e com subsistema habilitado.

- src/etl_layer/orchestrator.py
  - Motivo da leitura: confirmar que o ETL executa apenas pipelines explicitamente ligados no YAML.
  - Comportamento confirmado: Booking, Hotels.com, TripAdvisor e schema metadata dependem de flags enabled no bloco extract_transform_load.
