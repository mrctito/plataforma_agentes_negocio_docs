# Estudos de caso de referência para comercial e liderança

## 1. Objetivo deste documento

Este documento existe para dar ao time comercial, à pré-venda e à
liderança um conjunto de estudos de caso de referência baseados no que o
repositório já comprova.

O ponto mais importante é este: os cenários abaixo não devem ser
vendidos como clientes reais já implantados, nem como números de retorno
financeiro já medidos, porque essa evidência não aparece no material
lido. Eles devem ser usados como estudos de caso de referência, ou seja,
como narrativas comerciais plausíveis e tecnicamente sustentadas pela
arquitetura e pelas capacidades já documentadas.

Em linguagem simples, este manual responde à pergunta:

"Se eu precisar explicar rapidamente onde essa plataforma gera valor em
varejo, PDV e ERP, que história comercial eu posso contar sem inventar
nada?"

## 2. Como usar este material corretamente

Use este documento para:

1. estruturar apresentação comercial;
2. guiar demonstração de produto;
3. organizar conversa com diretoria, operações e tecnologia;
4. mostrar aderência por domínio sem cair em promessa vaga.

Não use este documento para:

1. afirmar que um cliente nominal já obteve os resultados descritos;
2. prometer ROI numérico sem projeto medido;
3. vender o produto como substituto do ERP ou do PDV;
4. esconder que alguns ganhos aqui são potenciais esperados, não métricas
   históricas comprovadas pelo repositório.

## 3. Leitura executiva rápida

Os três estudos de caso mais fortes hoje são estes.

1. varejo omnichannel com operação comercial conectada;
2. PDV e analytics governada com AG-UI e consultas aprovadas;
3. ERP com investigação operacional e automação durável governada.

Eles são fortes porque cobrem três dores grandes e fáceis de entender.

1. fragmentação de canais e operação de varejo;
2. dificuldade de consultar dado interno sem abrir SQL livre;
3. dificuldade de automatizar processo complexo de ERP com governança.

## 4. Estudo de caso 1: varejo omnichannel com catálogo, pedidos e operação conectada

### 4.1. Perfil do cenário

Empresa com operação de varejo que vende em múltiplos canais, como
ecommerce, marketplace e food delivery, e que sofre com excesso de tela,
excesso de integração pontual e baixa visão consolidada da operação.

### 4.2. Problema anterior

Antes da plataforma, o cenário típico é este.

1. pedido fica espalhado em vários ambientes;
2. catálogo vive desconectado entre canais;
3. o operador precisa alternar entre painéis diferentes para entender a
   operação;
4. cada novo canal tende a virar uma integração sob medida;
5. analytics e ação operacional acabam nascendo separadas.

Na prática, isso custa tempo, aumenta atrito de suporte e reduz a
capacidade de escalar a mesma solução entre clientes e verticais.

### 4.3. Como a plataforma entra

Com base na documentação atual, a plataforma pode entrar como camada
agentic e operacional por cima da operação de varejo já existente.

O desenho sustentado pelo repositório combina:

1. catálogo de tools de varejo e ecommerce para consulta e operação de
   catálogo, pedidos, clientes e estoque;
2. suporte a canais de food delivery e varejo omnichannel;
3. governança para separar integração externa de analytics interna;
4. possibilidade de expor a mesma capacidade em agente, interface
   agentic e fluxos de operação.

As âncoras principais dessa leitura estão em
[docs/tools/varejo.md](../tecnico/tools/varejo.md) e em
[docs/GUIA-COMERCIAL-PLATAFORMA.md](GUIA-COMERCIAL-PLATAFORMA.md).

### 4.4. Fluxo de solução em linguagem simples

1. a plataforma conecta ou reutiliza as trilhas de varejo já suportadas;
2. o operador passa a consultar e operar partes do ecossistema por uma
   camada comum;
3. o time reduz a necessidade de criar integração isolada a cada caso;
4. consultas internas mais sensíveis podem migrar para mecanismos
   governados em vez de acesso solto.

### 4.5. Valor entregue

O valor mais forte deste caso não é "ter um chat para varejo". O valor é
reduzir fragmentação operacional.

Ganho esperado e defensável:

1. menos retrabalho de integração por canal;
2. mais reaproveitamento entre clientes e projetos;
3. mais velocidade para demonstrar e operar novos fluxos comerciais;
4. melhor conexão entre consulta, operação e governança.

### 4.6. Como vender este caso em reunião

Use uma frase como esta:

> Hoje sua operação de varejo vive repartida entre canais, pedidos,
> estoque, catálogo e backoffice. A plataforma entra para transformar
> isso em uma camada agentic comum, com reuso, integração e governança,
> sem exigir que cada novo canal vire um projeto do zero.

### 4.7. Melhor encaixe

Este caso encaixa melhor quando a empresa:

1. opera múltiplos canais comerciais;
2. tem dor de integração recorrente;
3. precisa acelerar time-to-value em varejo ou food service;
4. quer vender plataforma de operação inteligente, não só chatbot.

### 4.8. Limite honesto do caso

Este caso não deve ser vendido como se o repositório já provasse uma
visão única perfeita de todos os canais em qualquer cliente. O que ele
prova é uma base forte de capacidades reutilizáveis para construir isso
de forma governada.

## 5. Estudo de caso 2: PDV e analytics governada com AG-UI

### 5.1. Perfil do cenário

Rede ou operação que precisa mostrar indicadores e leituras de negócio
de PDV sem abrir SQL livre, sem expor DSN e sem transformar dashboard em
acesso improvisado ao banco.

### 5.2. Problema anterior

O padrão ruim mais comum é este.

1. a área de negócio quer KPI rápido;
2. a área técnica teme abrir acesso demais;
3. dashboards especiais viram código artesanal;
4. perguntas simples sobre vendas e checkout viram fila de atendimento;
5. cada nova demanda analítica abre risco de governança.

### 5.3. Como a plataforma entra

O slice AG-UI de varejo demo comprova um caminho importante: a interface
não fala com SQL livre. Ela fala com capabilities aprovadas de negócio,
e o backend resolve a execução governada.

Pelo material lido, o runtime confirma:

1. catálogo fechado de capabilities para consultas aprovadas;
2. bloqueio explícito de SQL livre no payload;
3. validação estrita de parâmetros;
4. execução read-only por trilha governada;
5. caminho separado para materialização segura de dashboard.

As âncoras principais estão em
[docs/README-CONCEITUAL-AG-UI-DOMINIO-VAREJO-DEMO.md](../conceitual/README-CONCEITUAL-AG-UI-DOMINIO-VAREJO-DEMO.md)
e em [docs/tools/sql_dinamico.md](../tecnico/tools/sql_dinamico.md).

### 5.4. Fluxo de solução em linguagem simples

1. o usuário pede um KPI ou radar de operação;
2. a interface envia uma capability de negócio, não uma SQL arbitrária;
3. o backend escolhe a consulta aprovada ou materializa o dashboard por
   caminho governado;
4. a resposta volta como experiência AG-UI com eventos, snapshot e
   resultado final;
5. a empresa ganha leitura operacional sem abrir o banco de forma solta.

### 5.5. Valor entregue

Este caso é forte porque resolve um conflito clássico entre velocidade e
controle.

Ganho esperado e defensável:

1. mais autonomia para liderança e operação consultarem dados;
2. menos dependência de pedido técnico para leitura simples;
3. melhor governança sobre queries internas;
4. uso de interface agentic em um problema de negócio concreto.

### 5.6. Como vender este caso em reunião

Use uma frase como esta:

> A plataforma permite levar analytics assistido para PDV e operação
> comercial sem abrir SQL livre. O usuário conversa com capacidades de
> negócio aprovadas, e o backend garante governança, leitura controlada
> e experiência executiva mais simples.

### 5.7. Melhor encaixe

Este caso encaixa melhor quando a empresa:

1. tem base interna rica, mas pouco acessível ao negócio;
2. precisa de cockpit, radar ou KPI assistido;
3. quer demonstrar IA aplicada com segurança;
4. precisa convencer diretoria sem assustar segurança e TI.

### 5.8. Limite honesto do caso

Este caso não deve ser vendido como BI universal pronto para qualquer
consulta. O que o repositório prova é um caminho governado e fechado
para analytics aprovada e dashboards materializados com segurança.

## 6. Estudo de caso 3: ERP com investigação operacional e automação durável

### 6.1. Perfil do cenário

Empresa com ERP complexo, processos longos, exceções operacionais,
aprovações humanas e bases de dados grandes demais para serem navegadas
manualmente com agilidade.

### 6.2. Problema anterior

Em ERP, a dor raramente é uma pergunta simples. A dor costuma ser esta.

1. divergência de faturamento, estoque, compra ou fechamento;
2. processo que cruza múltiplas tabelas, anexos e histórico;
3. necessidade de aprovação em alguns pontos críticos;
4. dependência de especialista que conhece a base de memória;
5. automação limitada porque o fluxo não cabe em request curto.

### 6.3. Como a plataforma entra

O material atual sustenta uma combinação muito forte para esse cenário.

1. NL2SQL para tornar bases grandes de ERP mais consultáveis com
   guardrails de somente leitura;
2. DeepAgent Supervisor para tarefas duráveis, iterativas e com memória;
3. HIL e background execution para pausar, aprovar e continuar sem perder
   contexto;
4. observabilidade e governança como parte do desenho, não como acessório.

As âncoras principais estão em
[docs/README-CONCEITUAL-NL2SQL-COMPLETO.md](../conceitual/README-CONCEITUAL-NL2SQL-COMPLETO.md)
e em
[docs/README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md](../conceitual/README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md).

### 6.4. Fluxo de solução em linguagem simples

1. o operador ou gestor dispara uma investigação ou processo assistido;
2. o runtime consulta o recorte de dados relevante com governança;
3. o DeepAgent coordena subtarefas, memória e execução mais longa;
4. quando surge decisão sensível, o fluxo pode parar para aprovação
   humana;
5. o processo continua depois, com rastreabilidade e contexto preservado.

### 6.5. Valor entregue

Este é o estudo de caso mais forte para diretoria de operações e TI,
porque mostra IA como camada de trabalho real em processo complexo.

Ganho esperado e defensável:

1. menos dependência de especialistas raros para investigação inicial;
2. mais velocidade para triagem e diagnóstico operacional;
3. mais aderência entre autonomia e governança;
4. base mais forte para automação de processos longos de ERP.

### 6.6. Como vender este caso em reunião

Use uma frase como esta:

> O ERP não precisa de mais um chatbot. Precisa de uma camada agentic
> capaz de investigar exceções, consultar dados complexos com segurança,
> trabalhar em segundo plano e pedir validação humana quando o processo
> exigir. É exatamente esse tipo de desenho que a plataforma já sustenta.

### 6.7. Melhor encaixe

Este caso encaixa melhor quando a empresa:

1. tem ERP com muita exceção e muito processo;
2. sofre com gargalo de análise manual;
3. precisa combinar autonomia com controle;
4. quer um discurso de IA mais maduro do que atendimento conversacional.

### 6.8. Limite honesto do caso

Este caso não deve ser vendido como se o produto já viesse pronto com
todo o conhecimento de negócio do ERP do cliente. O que o repositório
prova é a base arquitetural para publicar tools, dados, governança,
execução durável e trilha humana nesse contexto.

## 7. Como escolher qual estudo de caso apresentar primeiro

Se a conversa estiver mais comercial e próxima da operação de loja,
comece pelo caso de varejo omnichannel.

Se a conversa estiver com diretoria, coordenação comercial ou liderança
que precisa ver KPI e governança, comece pelo caso de PDV e analytics
governada.

Se a conversa estiver com TI, operações, backoffice, ERP ou investidor
mais atento à profundidade do produto, comece pelo caso de ERP.

## 8. Roteiro curto de apresentação

Uma forma prática de usar este documento em reunião é esta.

1. comece pela dor do cliente, não pela arquitetura;
2. escolha um dos três casos mais parecidos com a operação dele;
3. explique como a plataforma entra por cima do sistema existente;
4. mostre o valor esperado em linguagem simples;
5. termine deixando claro o que já está provado e o que depende da
   implantação do cliente.

## 9. Explicação 101

Pense nesses estudos de caso como maquetes comerciais fiéis ao desenho
do produto. Eles não são propaganda solta. Também não são prova de que
tudo isso já está rodando em um cliente específico com número fechado de
resultado. Eles existem para mostrar, de forma prática e honesta, onde a
plataforma já é forte e como essa força pode ser aplicada em problemas
reais de varejo, PDV e ERP.

## 10. Leituras de apoio

Para aprofundar cada caso, a sequência mais útil é esta.

1. [docs/GUIA-COMERCIAL-PLATAFORMA.md](GUIA-COMERCIAL-PLATAFORMA.md)
2. [docs/tools/varejo.md](../tecnico/tools/varejo.md)
3. [docs/README-CONCEITUAL-AG-UI-DOMINIO-VAREJO-DEMO.md](../conceitual/README-CONCEITUAL-AG-UI-DOMINIO-VAREJO-DEMO.md)
4. [docs/README-CONCEITUAL-NL2SQL-COMPLETO.md](../conceitual/README-CONCEITUAL-NL2SQL-COMPLETO.md)
5. [docs/README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md](../conceitual/README-CONCEITUAL-DEEPAGENT-SUPERVISOR-COMPLETO.md)
