# Manual conceitual, executivo, comercial e estratégico: exemplos YAML reais de DeepAgent em ERP

## 1. O que e este manual

Este manual explica como o DeepAgent aparece de forma realista em cenarios de ERP, varejo e PDV dentro deste repositorio. O foco aqui nao e descrever o runtime DeepAgent inteiro outra vez. O foco e mostrar como esse runtime vira configuracao YAML aplicada a problemas de negocio reais.

Em vez de falar de um DeepAgent abstrato, este documento parte de quatro artefatos concretos que ja existem no workspace.

1. O modelo canonico em app/yaml/system/rag-config-modelo.yaml.
2. O exemplo corporativo mais amplo em app/yaml/rag-config-linx-food.yaml.
3. O cockpit operacional orientado a APIs em app/yaml/agent-linx-food-cockpit-prd.yaml.
4. O DeepAgent orientado a AG-UI e dyn_sql em app/yaml/rag-config-pdv-vendas-demo.yaml.

Isso importa porque muita documentacao de agentes fica no nivel da promessa. Aqui a pergunta e outra: como o DeepAgent realmente e configurado para operar dominio corporativo.

## 2. Que problema este manual resolve

Quando alguem pergunta como usar DeepAgent em ERP, normalmente aparecem tres respostas ruins.

1. Um exemplo minimo demais, que so prova sintaxe e nao prova valor de negocio.
2. Um exemplo gigante demais, que mistura tudo e nao ensina como adaptar por partes.
3. Um pseudoexemplo inventado, que parece bonito mas nao corresponde ao runtime do projeto.

Este manual resolve isso mostrando exemplos reais com tres recortes de negocio bem diferentes.

1. DeepAgent como coordenador geral de dados, APIs, utilitarios e conhecimento.
2. DeepAgent como cockpit operacional de ERP com APIs aprovadas.
3. DeepAgent como cerebro de UI generativa para vendas e dashboard em PDV.

## 3. Visao executiva

Para lideranca, os YAMLs reais deixam claro que o DeepAgent nao e uma feature de laboratorio. O projeto ja o usa como camada de coordenacao para casos que lembram ERP de verdade: consultas operacionais, leitura de indicadores, composicao de respostas governadas, memoria duravel, especializacao por subdominio e restricao por tool aprovada.

O valor executivo e que a plataforma consegue mudar o comportamento do agente por declaracao, sem reescrever Python para cada novo processo. Isso acelera onboarding de novos tenants, reduz custo de variacao por vertical e preserva governanca.

## 4. Visao comercial

Comercialmente, esses exemplos ajudam a vender o DeepAgent de forma correta. A mensagem nao e "temos IA para ERP". A mensagem suportada pelo codigo e mais forte e mais precisa.

1. O agente pode coordenar especialistas diferentes por subdominio.
2. O acesso a SQL e API pode ser governado por capacidades publicadas, nao por SQL livre.
3. O mesmo runtime serve tanto para operador interno quanto para UI generativa voltada a negocio.
4. Memoria duravel e modo assincrono permitem processos mais longos que um chat comum.

Em demo comercial, isso e muito mais convincente do que mostrar um prompt gigante. O YAML prova que a arquitetura foi pensada para operacao.

## 5. Visao estrategica

Os exemplos reais tambem mostram uma decisao estrategica importante do produto: o DeepAgent e a espinha dorsal reutilizavel, e o dominio ERP entra por configuracao declarativa.

Na pratica, isso prepara a plataforma para crescer em varias direcoes sem trocar de motor.

1. Mais ferramentas governadas por tenant.
2. Mais subagentes por subdominio de negocio.
3. Mais interfaces, como chat, AG-UI ou background execution.
4. Mais politicas de memoria, seguranca e saida estruturada.

O ganho estrategico e separar runtime agentic de conteudo de dominio. O produto fica mais reaproveitavel e menos acoplado a um unico setor.

## 6. Conceitos necessarios para ler os exemplos

### Supervisor DeepAgent

E o item de multi_agents cujo execution.type e deepagent. Ele e o coordenador principal, nao apenas um subagente.

### Subagentes especialistas

Sao agentes filhos declarados dentro de agents. Cada um recebe ferramentas, limites e descricao de uso. Em ERP, isso permite separar vendas, fiscal, lojas, catalogo, clientes ou pesquisa.

### Ferramenta governada

E a tool cujo comportamento ja foi aprovado no catalogo ou no bloco local_tools_configuration. Nos exemplos reais, isso aparece principalmente como dyn_sql<query_id>, dyn_api<endpoint_id> e proc_sql<procedure_id>.

### Memoria duravel

O contrato atual separa `memory` top-level e `backend` top-level. `memory` carrega caminhos absolutos de memoria operacional no runtime. `backend` ativa persistencia em Redis. Em termos praticos, o agente passa a reter contexto de trabalho entre execucoes sem misturar prompt com store duravel.

### AG-UI

E o protocolo de interface generativa usado para montar experiencias visuais mais ricas. No exemplo de PDV, o DeepAgent nao responde so texto: ele monta especificacao estruturada de dashboard.

## 7. Como os exemplos reais se organizam

Os YAMLs lidos deixam claro que ha tres formatos de uso recorrente do DeepAgent em ERP.

### 7.1. DeepAgent ERP generalista

O modelo canonico e o caso Linx Food mostram um supervisor DeepAgent que coordena varios especialistas transversais. Ha analista SQL, especialista de API, validador de documentos, pesquisador web/RAG e outros perfis auxiliares.

Esse formato e bom quando o negocio quer um ponto unico de entrada para duvidas e operacoes de natureza variada.

### 7.2. DeepAgent de cockpit operacional

O caso agent-linx-food-cockpit-prd.yaml mostra um DeepAgent mais estreito e mais operacional. Ele nao tenta resolver tudo. Ele coordena perguntas sobre lojas, saude fiscal e notas fiscais por meio de endpoints dyn_api especificos.

Esse formato e forte para ERP porque se parece com um centro de comando operacional: menos generalidade, mais respostas governadas sobre indicadores e status.

### 7.3. DeepAgent de interface generativa

O caso rag-config-pdv-vendas-demo.yaml mostra o mesmo motor DeepAgent servindo um frontend mais visual. Aqui o supervisor e segmentado por vendas, checkout, catalogo, clientes e dashboard dinamico. O subagente de dashboard ainda responde em JSON Schema restrito, nao em texto livre.

Esse formato e forte para ERP moderno porque aproxima analytics assistido, cockpit visual e governanca de consulta aprovada.

## 8. O exemplo base: template canonico reutilizavel

O modelo em app/yaml/system/rag-config-modelo.yaml e importante porque ele mostra o esqueleto que varios YAMLs reais repetem.

O padrao observado e este.

1. tools_library vazio para permitir injecao governada.
2. multi_agents com um supervisor DeepAgent.
3. middlewares habilitando memoria, subagentes, todo_list e PII.
4. permissions explicitas para filesystem quando necessario.
5. memory top-level e backend persistente com Redis.
6. subagentes com tools e limites bem delimitados.
7. local_tools_configuration descrevendo dyn_sql, dyn_api e configuracoes auxiliares.

Esse template tem valor estrategico porque funciona como lingua franca entre times. Em vez de cada projeto inventar sua propria estrutura, o dominio entra por especializacao sobre uma base recorrente.

## 9. O exemplo de food service e retail como ERP ampliado

Os arquivos app/yaml/rag-config-linx-food.yaml e app/yaml/rag-config-linx-retail.yaml mostram um DeepAgent quase enciclopedico para operacao comercial.

Ele combina:

1. SQL governado para clientes, historico de compras, top produtos e categorias.
2. APIs governadas para integracoes externas.
3. utilitarios e validacoes brasileiras.
4. pesquisa web e consulta RAG.
5. memoria duravel em Redis.

Esse tipo de exemplo e bom para explicar um ERP que precisa mesclar operacao, atendimento, inteligencia comercial e consulta de conhecimento em um mesmo ponto de entrada.

## 10. O exemplo de cockpit ERP por API aprovada

O arquivo app/yaml/agent-linx-food-cockpit-prd.yaml e o melhor exemplo para explicar um DeepAgent de operacao supervisionada.

Ele monta um supervisor DeepAgent com especialistas muito focados.

1. especialista em lojas.
2. especialista fiscal.
3. pesquisador AWS.
4. assistente de compras Agora.

O importante aqui nao e so a lista. O importante e a forma como o YAML delimita o contrato. O especialista fiscal, por exemplo, nao recebe API aberta: ele recebe dyn_api de saude fiscal e dyn_api de consulta de data atual, com instrucoes explicitas para nao assumir a data por conta propria.

Em termos de ERP, isso e valioso porque mostra um agente operando regra de negocio com disciplina. Nao e improvisacao sobre endpoints soltos.

## 11. O exemplo de PDV com AG-UI e dashboard estruturado

O arquivo app/yaml/rag-config-pdv-vendas-demo.yaml e o mais forte para explicar a convergencia entre DeepAgent, ERP e interface generativa.

O supervisor e selecionado explicitamente por selected_supervisor e divide o problema em subdominios de negocio.

1. vendas.
2. checkout e UCP.
3. catalogo.
4. clientes.
5. dashboard dinamico.

O diferencial real aparece no subdominio de dashboard dinamico. Ele nao recebe liberdade total de resposta. Ele recebe response_format com JSON Schema fechado, queries dyn_sql permitidas, parametros permitidos e regras de safety que proíbem HTML, script, SQL livre, segredos e correlation_id no payload.

Esse desenho e muito forte em ERP porque cria uma fronteira clara entre inteligencia e seguranca. O agente ajuda a montar a experiencia visual, mas continua preso ao contrato aprovado.

## 12. O que esses exemplos ensinam sobre modelagem de ERP

O codigo lido sugere cinco licoes praticas.

1. Nao modele um DeepAgent ERP como um unico agente onisciente. Modele por subdominio.
2. Prefira ferramentas governadas e semanticas, como dyn_sql e dyn_api, em vez de acesso livre.
3. Use memory e backend quando o processo exige continuidade real.
4. Use saida estruturada quando o resultado alimenta UI, dashboard ou integracao.
5. Restrinja filesystem e shell ao minimo necessario.

## 13. O que nao esta confirmado nesses exemplos

Os YAMLs lidos nao confirmam tudo que seria teoricamente possivel no DeepAgent.

1. Nao encontrei exemplo ERP real com interrupt_on e HIL habilitados nesses arquivos especificos.
2. Nao encontrei nesses exemplos uso explicito de async_subagents.
3. Nao encontrei shell habilitado nos casos ERP principais lidos.

Isso nao significa que o runtime nao suporte esses recursos. Significa apenas que os exemplos ERP reais lidos priorizam governanca de tools, memoria e especializacao de subdominios.

## 14. Casos praticos que podem ser demonstrados para cliente

### 14.1. Pergunta de operacao comercial

Cenario: "quais lojas estao com problema fiscal hoje e qual o percentual de sucesso de emissao?"

Exemplo real mais proximo: cockpit_supervisor em agent-linx-food-cockpit-prd.yaml.

### 14.2. Analise assistida de vendas

Cenario: "compare ticket medio e volume vendido por loja no periodo".

Exemplo real mais proximo: ag_ui_pdv_vendas_supervisor em rag-config-pdv-vendas-demo.yaml.

### 14.3. Inteligencia comercial com historico

Cenario: "quais clientes deixaram de comprar e quais categorias eles preferiam?"

Exemplo real mais proximo: supervisor_food em rag-config-linx-food.yaml ou rag-config-linx-retail.yaml.

## 15. Explicacao 101

Pense no DeepAgent como um gerente de operacoes configurado por YAML. O YAML diz quem sao os especialistas, quais ferramentas cada um pode usar, quais memorias entram no prompt, qual backend persistente pode gravar e que tipo de resposta e permitida.

Em um ERP, isso e importante porque o problema nunca e so "responder". O problema e decidir quem analisa o que, com qual dado aprovado, com qual limite e em qual formato final. Os exemplos reais do repositorio mostram exatamente isso acontecendo.

## 16. Checklist de entendimento

- Entendi que os exemplos vieram de YAMLs reais do repositorio.
- Entendi a diferenca entre template canonico, exemplo generalista, cockpit e AG-UI PDV.
- Entendi por que dyn_sql e dyn_api aparecem como base de governanca.
- Entendi que o DeepAgent em ERP ganha forca quando e dividido por subdominio.
- Entendi que o exemplo de PDV usa saida estruturada para dashboard, nao texto livre.
- Entendi quais recursos do runtime aparecem nesses exemplos e quais nao aparecem.

## 17. Evidencias no codigo

- app/yaml/system/rag-config-modelo.yaml
  - Motivo da leitura: confirmar o esqueleto canonico do DeepAgent usado como base do produto.
  - Comportamento confirmado: supervisor DeepAgent com middlewares, permissions, memory, backend persistente e subagentes com dyn_sql e dyn_api.
- app/yaml/rag-config-linx-food.yaml
  - Motivo da leitura: confirmar exemplo real de dominio food service com ferramentas governadas e memoria duravel.
  - Comportamento confirmado: DeepAgent generalista para ERP ampliado com SQL, API, RAG e utilitarios.
- app/yaml/rag-config-linx-retail.yaml
  - Motivo da leitura: confirmar variacao varejo/retail do mesmo padrao DeepAgent.
  - Comportamento confirmado: mesma espinha DeepAgent aplicada a retail com dyn_sql e dyn_api.
- app/yaml/agent-linx-food-cockpit-prd.yaml
  - Motivo da leitura: confirmar caso real de cockpit operacional orientado a APIs aprovadas.
  - Comportamento confirmado: especialistas focados em lojas e saude fiscal via dyn_api.
- app/yaml/rag-config-pdv-vendas-demo.yaml
  - Motivo da leitura: confirmar DeepAgent real para AG-UI com saida estruturada de dashboard em contexto PDV.
  - Comportamento confirmado: selected_supervisor explicito, subdominios de negocio e response_format fechado para DashboardSpec.
