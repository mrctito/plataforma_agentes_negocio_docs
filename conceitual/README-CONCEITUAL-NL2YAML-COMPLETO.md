# Manual conceitual, executivo, comercial e estratégico: NL2YAML governado por AST

## 1. O que é esta feature

NL2YAML, neste projeto, é a capacidade de transformar um objetivo descrito em linguagem natural em um YAML agentic governado, validável e publicável. Na prática, isso significa que o operador pode dizer o que quer construir, como um WorkflowAgent ou um DeepAgent, e a plataforma conduz essa intenção até um artefato configurável sem depender de edição textual cega do YAML.

O ponto central é que a feature não gera texto livre para depois "torcer" para que esteja correto. Ela converte a intenção para uma AST, que é uma representação estruturada e tipada do contrato agentic. Só depois dessa etapa o sistema valida, compila, monta preview, aponta lacunas, coleta perguntas obrigatórias e, se tudo estiver consistente, produz o YAML final.

Por isso, NL2YAML não é um prompt bonito para escrever configuração. É uma esteira governada de montagem de agentes avançados por linguagem natural.

## 2. Que problema ela resolve

Sem NL2YAML, a criação de configurações agentic ricas depende de um especialista que conheça a sintaxe YAML, as diferenças entre WorkflowAgent e DeepAgent, o catálogo efetivo de tools, as regras semânticas do assembly e os limites operacionais do runtime. Esse é um gargalo real.

O problema prático não é apenas escrever YAML. O problema é escrever YAML correto, governado, coerente com o alvo escolhido e seguro para publicação. Quando isso é feito manualmente, aparecem riscos previsíveis:

- target errado para o tipo de solução desejada;
- blocos incompatíveis entre si;
- tools faltando, ambíguas ou mal referenciadas;
- publicação de YAML semanticamente inválido;
- reescrita destrutiva de partes do documento fora do escopo alterado;
- drift entre o que a AST governada espera e o que alguém mexeu manualmente depois.

NL2YAML existe para encapsular essa complexidade. Ele reduz a distância entre intenção de negócio e configuração governada, sem abrir mão de validação, rastreabilidade e controle operacional.

## 3. Visão executiva

Para liderança, a importância da feature está em previsibilidade e velocidade com governança.

Ela reduz dependência de especialistas raros para mudanças simples ou médias em configurações agentic. Também diminui o risco de incidentes causados por edição manual em contratos YAML estratégicos. Em vez de um fluxo opaco, a plataforma passa a oferecer uma trilha explícita: prontidão do ambiente, geração do rascunho, validação, confirmação em dry-run, perguntas pendentes e só então publicação.

O ganho executivo é claro: menos improviso operacional, menos retrabalho e mais capacidade de escalar configuração agentic com segurança.

## 4. Visão comercial

Comercialmente, NL2YAML ajuda a explicar a plataforma como algo configurável por intenção, e não apenas por arquivo técnico.

Isso resolve uma dor comum em clientes corporativos: a equipe sabe o fluxo que deseja, mas não domina a sintaxe detalhada da configuração. Em vez de pedir que o cliente ou o consultor escreva YAML na mão, a plataforma recebe o briefing, devolve um preview governado, explica o que escolheu e pede apenas as decisões que realmente faltam.

Na conversa comercial, a proposta de valor fica objetiva:

- acelerar criação de agentes e fluxos complexos;
- reduzir dependência de edição manual especializada;
- manter governança e trilha de auditoria;
- permitir revisão antes de publicar;
- evitar promessas falsas de automação cega.

O que pode ser prometido com segurança: a plataforma transforma briefing em configuração governada com validação e checkpoints.

O que não deve ser prometido: que qualquer objetivo textual vira YAML publicável sem perguntas, sem revisão e sem dependência de catálogo, contexto e prontidão operacional.

## 5. Visão estratégica

Estrategicamente, NL2YAML fortalece a tese de plataforma YAML-first governada por AST.

O valor maior não é apenas "traduzir linguagem natural". O valor maior é usar linguagem natural como entrada de um pipeline tipado e semântico, reaproveitando o mesmo núcleo usado por schema, validação, catálogo, diff e publicação. Isso reduz acoplamento entre frontend, backend e edição manual, porque todos passam a orbitar o mesmo contrato estruturado.

Essa escolha prepara a plataforma para:

- designers visuais mais ricos sem duplicar semântica no frontend;
- montagem assistida de WorkflowAgent e DeepAgent no mesmo núcleo;
- evolução de contratos agentic com menos risco de drift interpretativo;
- maior governança de publicação;
- onboarding de perfis menos técnicos sem rebaixar a qualidade do contrato.

## 6. Conceitos necessários para entender

### 6.1. Linguagem natural para YAML

Neste projeto, linguagem natural não significa gerar texto bruto e aceitar o resultado como pronto. Significa usar a descrição do objetivo como entrada para um processo de montagem assistida. O texto humano serve para classificar o alvo, sugerir estrutura inicial, levantar tools prováveis e apontar lacunas. O produto final continua sendo um contrato validado.

### 6.2. AST

AST é a representação estruturada e tipada do documento agentic. Em termos simples, é o YAML convertido para um objeto que o sistema consegue entender com precisão. Em vez de ver apenas linhas de texto, a plataforma passa a enxergar workflow, nodes, supervisors, agents, tools, memory e demais blocos como partes nomeadas e validáveis.

Na prática, a AST é o motivo pelo qual o sistema consegue:

- saber qual alvo está sendo montado;
- validar semântica por espinha dorsal;
- aplicar merge parcial só na parte governada;
- detectar drift;
- pedir perguntas obrigatórias quando faltam decisões;
- produzir diff preview sem sobrescrever o documento inteiro.

### 6.3. Target agentic

O assembly trabalha com dois alvos oficiais principais: workflow e deepagent. O modo auto tenta resolver qual desses alvos faz mais sentido para o briefing. Isso é importante porque cada alvo tem contrato diferente. Workflow é grafo determinístico com nodes; DeepAgent adiciona recursos avançados de middleware, memória e HIL declarativo.

Regra importante para este repositório: WorkflowAgent continua sendo uma espinha dorsal própria e DeepAgent é o runtime agentic oficial para o trilho multi_agents. Se um YAML antigo ainda cair em uma trilha legada sem deepagent, isso deve ser tratado como migração e não como opção nova de documentação.

### 6.4. Preflight

Preflight é a checagem de prontidão operacional antes da geração final. Ele responde algo simples, mas crítico: o ambiente está pronto para produzir YAML final com segurança? Essa resposta considera, entre outros sinais, provider LLM, suporte a saída estruturada, tamanho do catálogo e modo de geração.

### 6.5. Dry-run

Dry-run é a confirmação sem persistência. O sistema executa a etapa de consolidação final, monta o YAML, gera diff e valida o resultado, mas sem gravar em arquivo. Isso separa revisão de publicação.

### 6.6. Drift governado

Drift é a divergência entre o que o fragmento governado deveria ser e o que alguém alterou manualmente depois no YAML. A feature trata isso como um problema de governança. Em vez de ignorar a divergência, ela detecta a quebra usando fingerprint do bloco governado.

## 7. Como a feature funciona por dentro

O fluxo começa com um briefing textual, um alvo desejado ou automático, um YAML base opcional e o modo de geração. O router administrativo resolve permissões, correlation_id e feature flag. Depois o serviço do assembly conduz a pipeline.

Primeiro, ele resolve o target efetivo. Em seguida roda o preflight. Se o ambiente não estiver pronto, o fluxo para ali com blocking_stage igual a preflight e devolve diagnóstico claro. Se estiver pronto, parte para draft.

No draft, a intenção vira AST inicial e preview mesclado. Esse ponto é importante: o sistema não assume que o rascunho está completo. Se houver perguntas obrigatórias ou invalidade semântica, o fluxo para em draft e devolve questions, diagnostics e o AST parcial para revisão.

Se o draft estiver coerente, o sistema executa validate. Aqui ele verifica semântica do alvo escolhido, tools e eventuais problemas do YAML base governado. Se a validação reprovar, o blocking_stage vira validate.

Se validate passar, o fluxo entra em confirm com apply=false e force=false. Isso quer dizer: consolidar tudo, gerar YAML final e diff preview, mas sem gravar arquivo. Se confirm não conseguir produzir um YAML utilizável, o fluxo bloqueia em confirm.

Só quando todas essas etapas fecham o sistema devolve success=true, final_yaml, final_yaml_text, chosen_tools, decision_trace e diff_preview.

## 8. Divisão em etapas ou submódulos

Detalhamento aprofundado por etapa:

1. [Resolucao de intencao](README-CONCEITUAL-NL2YAML-COMPLETO-RESOLUCAO-DE-INTENCAO.md)
2. [Prontidao operacional](README-CONCEITUAL-NL2YAML-COMPLETO-PRONTIDAO-OPERACIONAL.md)
3. [Draft assistido](README-CONCEITUAL-NL2YAML-COMPLETO-DRAFT-ASSISTIDO.md)
4. [Validacao semantica](README-CONCEITUAL-NL2YAML-COMPLETO-VALIDACAO-SEMANTICA.md)
5. [Consolidacao e preview final](README-CONCEITUAL-NL2YAML-COMPLETO-CONSOLIDACAO-E-PREVIEW-FINAL.md)
6. [Publicacao segura](README-CONCEITUAL-NL2YAML-COMPLETO-PUBLICACAO-SEGURA.md)

### 8.1. [Resolucao de intencao](README-CONCEITUAL-NL2YAML-COMPLETO-RESOLUCAO-DE-INTENCAO.md)

É a etapa que pega o briefing e decide qual contrato agentic faz sentido. Ela existe porque workflow e deepagent não são o mesmo tipo de artefato. Sem essa separação, a plataforma misturaria contratos incompatíveis.

Ela recebe prompt, target solicitado e YAML base. Entrega target resolvido e o começo do rastro de decisão. O valor entregue aqui é enquadramento correto do problema.

### 8.2. [Prontidao operacional](README-CONCEITUAL-NL2YAML-COMPLETO-PRONTIDAO-OPERACIONAL.md)

É a etapa de preflight. Ela existe para evitar gerar preview ou YAML final quando o ambiente ainda não suporta o fluxo com segurança. Ela não produz o YAML em si; produz condições operacionais para seguir.

Ela recebe target, modo de geração, YAML base e contexto do usuário. Entrega checks, diagnostics, llm_provider, suporte a structured output e status ready ou blocked.

### 8.3. [Draft assistido](README-CONCEITUAL-NL2YAML-COMPLETO-DRAFT-ASSISTIDO.md)

É a primeira tradução do objetivo para AST. Aqui o sistema tenta montar a estrutura inicial do documento. O valor dessa etapa é gerar um ponto de partida utilizável, e não fingir completude quando ainda faltam decisões.

Ela pode devolver perguntas obrigatórias quando a intenção ainda está ambígua. Esse comportamento é importante porque evita YAML "meio certo" ser tratado como pronto.

### 8.4. [Validacao semantica](README-CONCEITUAL-NL2YAML-COMPLETO-VALIDACAO-SEMANTICA.md)

Essa etapa garante que o documento AST faz sentido para o alvo. Ela existe porque um YAML sintaticamente válido ainda pode ser semanticamente errado. O validador escolhe a lógica certa para workflow ou deepagent, e também valida tools declaradas.

### 8.5. [Consolidacao e preview final](README-CONCEITUAL-NL2YAML-COMPLETO-CONSOLIDACAO-E-PREVIEW-FINAL.md)

Aqui o sistema compila o fragmento governado, faz merge no YAML base sem reescrever tudo, estampa fingerprint do bloco governado e gera diff preview. Essa etapa transforma o draft validado em um artefato final revisável.

### 8.6. [Publicacao segura](README-CONCEITUAL-NL2YAML-COMPLETO-PUBLICACAO-SEGURA.md)

A publicação só acontece quando apply=true. Mesmo assim, ela não aceita caminho arbitrário. O output_path precisa ser um arquivo YAML dentro de app/yaml. Isso existe para impedir gravação fora do perímetro governado.

## 9. Pipeline principal

1. O operador descreve o objetivo em linguagem natural.
2. O sistema resolve usuário, permissão, correlation_id e target solicitado.
3. O preflight avalia prontidão operacional.
4. O draft gera AST inicial e preview.
5. Se houver perguntas, o fluxo para e devolve somente o que falta decidir.
6. O validate confere semântica do alvo e detecta drift governado do YAML base.
7. O confirm em dry-run consolida YAML final sem persistir.
8. A UI libera revisão e, se aplicável, publicação segura.

O ponto mais importante do pipeline é que ele não salta da intenção para a persistência. Sempre há etapas intermediárias explícitas.

## 10. Decisões técnicas e trade-offs

### 10.1. AST como contrato central

Ganho: elimina ambiguidade e permite validação real.

Custo: aumenta a disciplina do pipeline e exige parsers, validators e compiler dedicados.

Por que vale a pena: porque o problema aqui não é escrever YAML, e sim montar configuração agentic confiável.

### 10.2. Preflight antes do YAML final

Ganho: evita gerar ou publicar configuração em ambiente não pronto.

Custo: adiciona uma etapa extra e pode bloquear cedo.

Por que vale a pena: porque bloquear cedo é melhor do que produzir um YAML aparentemente pronto em ambiente inconsistente.

### 10.3. Perguntas obrigatórias em vez de assumir resposta

Ganho: reduz erro silencioso.

Custo: o fluxo pode não ser totalmente automático.

Por que vale a pena: porque completar lacuna ambígua por inferência fraca é exatamente o tipo de comportamento que gera configuração perigosa.

### 10.4. Confirm em dry-run

Ganho: separa revisão de persistência.

Custo: exige uma etapa explícita a mais.

Por que vale a pena: porque publicação e validação não devem ser a mesma coisa.

### 10.5. Publicação limitada a app/yaml

Ganho: reduz risco operacional e protege o perímetro governado.

Custo: menos liberdade para gravar em qualquer caminho.

Por que vale a pena: porque liberdade irrestrita de output_path seria um bypass claro da governança.

## 11. O que acontece em caso de sucesso

Quando o fluxo fecha corretamente, o operador recebe um YAML final coerente com o alvo resolvido, um texto serializado para revisão humana, diff preview, tools efetivamente escolhidas, rastro de decisão, correlation_id e um relatório de validação limpo ou com avisos não bloqueantes.

Na UI, isso se traduz em preview liberado para publicação. No backend, isso significa que preflight, draft, validate e confirm em dry-run fecharam sem bloqueio.

## 12. O que acontece em caso de erro

Existem quatro pontos principais de bloqueio confirmados no código.

### 12.1. Bloqueio em preflight

O ambiente não está pronto para gerar o YAML final com segurança.

### 12.2. Bloqueio em draft

Ainda existem perguntas obrigatórias ou o rascunho não está semanticamente aceitável.

### 12.3. Bloqueio em validate

O AST até existe, mas o contrato ainda não fecha para o alvo escolhido ou há drift/problema semântico detectado.

### 12.4. Bloqueio em confirm

O sistema não conseguiu consolidar um YAML final utilizável após a confirmação em dry-run.

Esse desenho é intencional. Em vez de retornar apenas "deu erro", a feature devolve em qual estágio o processo parou.

## 13. Impacto técnico

NL2YAML reforça isolamento entre intenção, semântica e persistência. Isso reduz acoplamento com edição textual manual, melhora observabilidade, torna a validação mais confiável e facilita evolução do ecossistema agentic sem espalhar regra de negócio em múltiplas superfícies.

## 14. Impacto executivo

O impacto executivo é redução de risco operacional na configuração agentic e aumento de previsibilidade do fluxo de mudança. O negócio ganha velocidade sem aceitar publicação cega.

## 15. Impacto comercial

O impacto comercial é permitir demonstrar configuração avançada por briefing com governança. Isso ajuda especialmente em clientes com muitos fluxos e pouca tolerância a erro manual.

## 16. Impacto estratégico

O impacto estratégico é consolidar a AST como plataforma de montagem agentic. Isso sustenta o roadmap de studio visual, montagem assistida, tooling recomendada e publicação governada no mesmo núcleo semântico.

## 17. Explicação 101

Pense assim: o usuário descreve o tipo de agente ou fluxo que quer montar. Em vez de a plataforma escrever um texto YAML qualquer e dizer "se vira", ela transforma essa ideia em uma versão estruturada que consegue revisar. Se faltar informação, pergunta. Se houver erro, bloqueia. Se estiver tudo consistente, entrega o YAML final pronto para revisão e publicação segura.

Ou seja, a linguagem natural aqui é a porta de entrada. A AST é o mecanismo que impede essa porta de virar bagunça.

## 18. Limites e pegadinhas

- NL2YAML não elimina a necessidade de revisão humana.
- Linguagem natural não substitui catálogo, validação e preflight.
- YAML final não é persistido automaticamente pelo fluxo objective-to-yaml.
- Workflow e deepagent continuam com contratos diferentes; o modo auto não apaga essa diferença.
- Ter preview não significa que já é seguro publicar sem olhar diagnostics e diff.
- Fallback heurístico não é implícito por conveniência; no modo auto ele depende de opt-in explícito.

## 19. Checklist de entendimento

- Entendi que NL2YAML é uma esteira governada, não geração textual livre.
- Entendi o papel da AST como contrato tipado.
- Entendi por que existem preflight, draft, validate e confirm.
- Entendi a diferença entre preview e publicação.
- Entendi por que a feature pode devolver perguntas obrigatórias.
- Entendi o valor para workflows, supervisores e deepagents.
- Entendi por que drift e publicação segura são partes do problema.
- Entendi o impacto executivo, comercial e estratégico.

## 20. Evidências no código

- src/api/routers/config_assembly_router.py
  - Motivo da leitura: confirmar o endpoint HTTP canônico de NL2YAML.
  - Símbolo relevante: objective_to_yaml_agentic_ast.
  - Comportamento confirmado: publicação de POST /config/assembly/objective-to-yaml com permissionamento, correlation_id e logging estruturado.

- src/config/agentic_assembly/assembly_service.py
  - Motivo da leitura: confirmar a pipeline real do objetivo até YAML.
  - Símbolo relevante: objective_to_yaml.
  - Comportamento confirmado: preflight -> draft -> validate -> confirm dry-run, com blocking_stage explícito.

- src/config/agentic_assembly/ast/document.py
  - Motivo da leitura: confirmar o envelope AST canônico.
  - Símbolo relevante: AgenticDocumentAST.
  - Comportamento confirmado: AST centraliza target, workflows, multi_agents, tools_library e recorte por alvo.

- src/config/agentic_assembly/ast/workflow.py
  - Motivo da leitura: confirmar a riqueza do contrato workflow.
  - Símbolo relevante: WorkflowAST e WorkflowNodeAST.
  - Comportamento confirmado: nodes discriminados por mode, edges declarativas e settings governados.

- src/config/agentic_assembly/ast/deepagent.py
  - Motivo da leitura: confirmar a especialização DeepAgent.
  - Símbolo relevante: DeepAgentSupervisorAST.
  - Comportamento confirmado: middlewares, skills, interrupt_on, permissions e execução forçada em deepagent.

- tests/unit/test_02-01-78_agentic_assembly_service.py
  - Motivo da leitura: confirmar o comportamento esperado do fluxo.
  - Símbolo relevante: testes objective_to_yaml.
  - Comportamento confirmado: fluxo encadeado, perguntas pendentes, modo heurístico e modo auto com opt-in de fallback.

- app/ui/static/js/objective-yaml-studio.js
  - Motivo da leitura: confirmar a experiência standalone orientada ao usuário final.
  - Símbolo relevante: objectiveYamlStudioPage.
  - Comportamento confirmado: consumo do endpoint, exibição de diagnostics, questions, diff, correlation_id e publicação assistida.
