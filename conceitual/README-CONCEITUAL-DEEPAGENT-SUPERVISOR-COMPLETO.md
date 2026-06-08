# Manual conceitual, executivo, comercial e estratégico: DeepAgent Supervisor completo

## 1. O que é esta feature

O DeepAgent Supervisor é o runtime agentic mais poderoso do projeto para supervisão governada de agentes com autonomia operacional ampliada. Ele não é apenas um supervisor com prompt melhor, nem apenas um nome diferente para multiagentes. No código, ele aparece como um modo próprio de execução, identificado por execution.type igual a deepagent, com AST dedicada, validação semântica específica, runtime governado, middlewares explícitos, memória durável, pausa humana controlada, subagentes síncronos e assíncronos e integração direta com execução em background.

Em linguagem simples: ele é a peça da plataforma que permite criar agentes que realmente trabalham, e não apenas respondem. Esses agentes podem manipular tools, organizar tarefas, consultar e gravar memória, trabalhar com arquivos sob regras explícitas, rodar shell governado, delegar partes do trabalho a subagentes, pausar para aprovação humana e continuar depois sem perder o fio.

## 2. Que problema ele resolve

O DeepAgent Supervisor existe para resolver um problema que quase sempre aparece quando uma empresa tenta sair do nível “chat com IA” para o nível “agente que executa processos reais”.

Sem ele, normalmente sobram duas opções ruins.

- Ou o sistema usa um agente simples demais, que conversa bem, mas não consegue operar fluxos longos, stateful e governados.
- Ou o sistema solta autonomia demais sem contrato, sem validação semântica forte e sem mecanismos claros de segurança para filesystem, shell, memória, aprovação humana e continuidade.

O DeepAgent Supervisor resolve isso criando um meio-termo disciplinado: autonomia alta com governança alta.

Na prática, isso significa que o sistema consegue operar processos ricos e demorados, mas sem esconder capacidade, sem permissões implícitas e sem deixar a plataforma depender de comportamento mágico de framework.

## 3. Por que ele é tão poderoso

O poder do DeepAgent Supervisor não vem de uma única feature. Ele vem da combinação de várias capacidades que, juntas, formam um runtime de execução muito acima do padrão de supervisor simples.

### 3.1. Ele trabalha com ferramentas reais

O supervisor agrega tools do catálogo efetivo do YAML, deduplica ferramentas repetidas entre subagentes e ainda consegue injetar um subagente automático especializado em execução background quando essa capacidade é habilitada. Isso o torna capaz de operar processos longos, consultar status, reagendar tarefas e agir sobre resultados já produzidos.

### 3.2. Ele sabe planejar execução contínua por tarefas

O middleware todo_list não é detalhe cosmético. Ele dá ao runtime uma lista interna de tarefas, o que melhora controle do trabalho em cenários longos. Em vez de depender apenas de memória textual difusa, o agente pode manter uma noção operacional do que já fez, do que falta fazer e do que precisa ser revisado.

### 3.3. Ele trabalha com arquivos e shell de forma governada

O runtime suporta filesystem e shell persistente, mas não como liberdade total. O contrato exige permissions explícitas, operações limitadas a read e write, caminhos absolutos sem navegação suspeita e políticas declarativas de shell. Isso dá poder operacional sem abrir mão de restrição e rastreabilidade.

### 3.4. Ele mantém memória durável

O contrato atual separa duas coisas. `memory` top-level lista caminhos absolutos de memória operacional carregada no runtime. `backend` top-level habilita persistência durável em Redis com escopos user, agent e org, além de política read_only ou read_write. Na prática, isso transforma o agente em algo capaz de reter contexto de trabalho ao longo do tempo sem misturar memória de prompt com store persistente.

### 3.5. Ele combina autonomia com aprovação humana

Quando a operação exige segurança adicional, o DeepAgent pode pausar em tools específicas via interrupt_on, aguardar decisão humana e continuar depois usando o mesmo thread_id e o mesmo correlation_id lógico. Isso é especialmente importante em processos com impacto financeiro, contratual, operacional ou regulatório.

### 3.6. Ele entende subagentes de verdade

O supervisor consegue trabalhar com subagentes síncronos, que entram no runtime local com a mesma disciplina de middleware e permissões, e com async_subagents, que representam delegação para grafos externos com graph_id e transporte tipado. Isso o aproxima mais de um orquestrador de trabalho do que de um chatbot.

### 3.7. Ele foi pensado para background

Poucos pontos do código deixam isso tão claro quanto o runtime agentic em background e o subagente automático de background execution. O DeepAgent não depende do ciclo HTTP curto para existir. Ele pode ser disparado, pausado, retomado, auditado e acompanhado como execução durável.

## 4. Por que ele é especialmente forte para processos em background

Se o objetivo é rodar agentes em segundo plano, o DeepAgent Supervisor se aproxima do perfil ideal por cinco razões combinadas.

### 4.1. O estado não morre no fim da requisição

Como há thread_id, checkpointer, store e memória Redis, o trabalho pode continuar depois. Isso é decisivo em fluxos que demoram minutos, horas ou até ciclos periódicos inteiros.

### 4.2. O runtime já fala a linguagem de agendamento e retomada

O projeto já expõe tools para agendar, reagendar, cancelar e consultar execuções background. Isso significa que o DeepAgent não é apenas “capaz de continuar”; ele é capaz de operar dentro de um ecossistema que já entende execução fora do request síncrono.

### 4.3. Aprovação humana não quebra o fluxo

Se um processo background cair em Human in the Loop, o runtime consegue transformar essa pausa em uma aprovação assíncrona durável, com canais como email e WhatsApp, token de aprovação, TTL, aprovadores autorizados e continuação posterior. Isso evita a armadilha clássica do job que para e nunca mais sabe como continuar.

### 4.4. O runtime foi desenhado para governança forte

Processos em background são perigosos quando acumulam autonomia invisível. Aqui o desenho força contrato para skills, permissions, interrupt_on, checkpointer, memory backend e canais de aprovação. Esse tipo de rigidez é o que torna o agente confiável para operações reais.

### 4.5. A observabilidade é parte do produto, não acessório

O runtime mantém telemetria de tool, resposta, middleware, lifecycle, correlation_id, thread_id e eventos HIL. Isso é indispensável para operar background com responsabilidade. Um job autônomo sem diagnóstico é só uma fonte lenta de incidente.

## 5. Recursos avançados que tornam o DeepAgent especial

## 5.1. Middlewares governados

O contrato do DeepAgent inclui uma pilha declarativa de middlewares governados.

- filesystem
- shell
- memory
- subagents
- background_execution_subagent
- human_in_the_loop
- summarization
- pii
- todo_list
- skills

Além disso, o runtime registra middlewares oficiais e middlewares do produto para seleção de tool, limites, retry, edição de contexto, pós-processamento e tratamento de erro.

## 5.2. Filesystem com allow e deny

O supervisor aceita permissions com operações read e write, caminhos absolutos e modo allow ou deny. Em outras palavras, o agente pode ganhar poder sobre arquivos, mas dentro de uma cerca declarativa clara.

## 5.3. Shell persistente governado

O middleware de shell suporta políticas host, docker e codex_sandbox, com timeouts, limites de saída, memória, CPU, read_only_rootfs, usuário e outros parâmetros. Isso é muito mais sofisticado do que “rodar um comando”. É uma sessão shell com política operacional definida.

Ao mesmo tempo, o contrato moderno não aceita shell persistente e HIL no mesmo supervisor. Quando os dois aparecem juntos, o runtime falha cedo em vez de esconder uma combinação operacional ambígua.

## 5.4. Memória persistente com escopo organizacional

O DeepAgent pode memorizar contexto por usuário, por agente ou por organização. Em um ERP, isso abre espaço para agentes que aprendem histórico operacional do time, do processo ou do cliente sem confundir escopos.

## 5.5. Skills top-level e por subagente

O runtime suporta skills como fontes de conhecimento e comportamento, tanto no supervisor principal quanto em subagentes. Isso ajuda a especializar agentes sem duplicar prompt gigante e sem transformar o YAML em texto sem estrutura.

## 5.6. Structured output

O contrato aceita response_format em JSON Schema. Isso é extremamente útil quando o processo precisa produzir saídas estruturadas para integração, auditoria, filas de aprovação ou decisão automatizada posterior.

## 5.7. HIL síncrono e assíncrono

O supervisor pode interromper tools específicas, pedir approve, edit, reject ou respond e continuar a mesma thread depois. Quando a aprovação é assíncrona, ainda há canais, aprovadores autorizados, política de expiração e proteção contra resolução por usuário errado.

Na prática, isso significa HIL governado com pausa humana explícita. Se o mesmo supervisor tentar combinar isso com shell persistente, a configuração é rejeitada antes da execução.

## 5.8. Resumo automático de contexto

O middleware summarization serve para processos longos. Ele ajuda a comprimir contexto quando a execução cresce, evitando degradar a qualidade do agente ao longo do tempo.

## 5.9. Proteção de PII

O middleware pii permite bloquear, redigir, mascarar ou hashear dados sensíveis em entrada, saída e resultados de tool. Para ambiente corporativo, isso é um diferencial operacional importante.

## 5.10. Subagentes assíncronos

O suporte a async_subagents permite delegar etapas para grafos externos. Isso é relevante quando o processo é grande demais para um único runtime e precisa distribuir trabalho entre especialistas ou fluxos remotos.

## 6. Todos os recursos mais importantes, em termos práticos

Quando alguém pergunta “o que esse agente realmente consegue fazer?”, a resposta baseada no código é esta.

- Usar catálogo de tools do tenant com deduplicação entre agentes.
- Montar subagentes com prompts, tools, skills, response_format e permissions próprias.
- Manter lista de tarefas interna via todo_list.
- Ler e gravar arquivos quando filesystem e permissions permitem.
- Abrir shell persistente com política governada.
- Persistir memória em Redis por user, agent ou org.
- Pedir aprovação humana em tools específicas.
- Escalar HIL para canais assíncronos com email e WhatsApp.
- Agendar e acompanhar processos em background via subagente especializado.
- Continuar execuções interrompidas com o mesmo thread_id.
- Produzir saídas estruturadas.
- Aplicar regras de PII.
- Resumir contexto em fluxos longos.
- Rodar com cache por hash de YAML e versão de supervisor.

## 7. Por que isso é valioso para um ERP

ERP não vive de perguntas simples. ERP vive de processo, estado, exceção, aprovação, recorrência, rastreabilidade e integração com múltiplos sistemas. É exatamente esse tipo de ambiente em que um agente fraco quebra rápido.

O DeepAgent Supervisor se encaixa bem em ERP porque consegue combinar:

- autonomia para executar trabalho real;
- governança para não extrapolar permissões;
- memória para acompanhar processos longos;
- approval flow para tarefas sensíveis;
- background para não travar a operação em requisições síncronas;
- structured output para integração com o restante da plataforma.

## 8. Casos de uso corporativos reais em ERP

Os exemplos abaixo são casos de uso reais e plausíveis para o runtime, assumindo que o tenant publique as tools corporativas necessárias no catálogo. O que é nativo aqui é a arquitetura agentic de execução, não o domínio específico de cada empresa.

### 8.1. Investigação de divergência de faturamento com execução em segundo plano

Cenário: o ERP detecta diferenças entre pedido, nota fiscal, faturamento e recebimento. Resolver isso exige cruzar dados, abrir anexos, checar logs operacionais, consultar histórico e eventualmente pedir aprovação para reclassificar uma ocorrência.

Como o DeepAgent ajuda:

- o supervisor coordena subagentes de faturamento, financeiro e atendimento;
- usa todo_list para acompanhar o progresso da investigação;
- usa memória para lembrar hipóteses já testadas;
- pode manipular arquivos de evidência quando filesystem está liberado;
- pode abrir shell governado se o processo exigir utilitário operacional interno;
- quando precisa aprovar ação sensível, pausa via HIL;
- se a análise for longa, agenda partes do trabalho em background e consulta os resultados depois.

Por que isso é forte: é o tipo de caso em que um chatbot tradicional só “sugere”. O DeepAgent, ao contrário, consegue coordenar execução durável, revisão humana e devolução estruturada do diagnóstico.

### 8.2. Fechamento financeiro diário com exceções e aprovações assíncronas

Cenário: no fim do expediente, o ERP precisa consolidar contas, revisar pendências, detectar inconsistências, separar exceções e enviar itens críticos para aprovação de gestor fora do horário comercial.

Como o DeepAgent ajuda:

- agenda a execução via background_execution_subagent;
- delega partes do fechamento para subagentes especialistas;
- usa structured output para devolver lotes de exceção em formato previsível;
- resume contexto para manter eficiência durante a longa execução;
- dispara HIL assíncrono por email ou WhatsApp quando a aprovação precisa esperar alguém;
- continua automaticamente o processo depois da decisão humana.

Por que isso é forte: o processo não precisa ficar preso a uma sessão humana ativa, mas também não precisa perder governança quando surge uma decisão financeira crítica.

### 8.3. Auditoria operacional de compras com memória organizacional

Cenário: a empresa quer um agente que rode continuamente em background verificando desvios de compra, padrões de fornecedor, itens fora de política e reincidência por centro de custo.

Como o DeepAgent ajuda:

- mantém memória por escopo org para preservar critérios e contexto entre execuções;
- roda periodicamente em background;
- usa subagentes para análise de fornecedor, compliance e histórico de aprovação;
- protege dados sensíveis com PII middleware;
- produz saídas estruturadas prontas para dashboards, tickets ou filas de revisão;
- pausa apenas os casos que realmente exigem validação humana.

Por que isso é forte: o agente deixa de ser só um assistente consultivo e passa a atuar como camada de vigilância operacional persistente.

### 8.4. Reconciliação complexa de estoque com múltiplas fontes e investigação longa

Cenário: o ERP encontra divergência entre saldo sistêmico, inventário físico, transferências internas e notas de movimentação. A análise precisa cruzar tabelas, arquivos exportados, registros históricos e possivelmente delegar partes da apuração.

Como o DeepAgent ajuda:

- usa filesystem para operar sobre artefatos de apuração dentro de paths permitidos;
- usa shell governado para automações operacionais controladas, quando a política habilita;
- aciona subagentes por área, como logística, fiscal e supply;
- agenda etapas demoradas em background;
- retoma a análise mais tarde na mesma thread quando surgem novas evidências;
- consolida o resultado em structured output para ação posterior.

Por que isso é forte: reconciliação de estoque é exatamente o tipo de caso com muita exceção, muito estado intermediário e baixa tolerância a improviso. O DeepAgent oferece continuidade e disciplina operacional.

## 9. Visão executiva

Para liderança, o DeepAgent Supervisor entrega quatro valores centrais.

- Aumenta a produtividade operacional em processos que antes dependiam de acompanhamento manual constante.
- Reduz risco de autonomia desgovernada porque filesystem, shell, memória e HIL entram por contrato explícito.
- Melhora governança porque cada execução carrega correlation_id, thread_id, timeline e telemetria estruturada.
- Viabiliza automação durável, não apenas atendimento assistido.

Em termos executivos simples: ele é uma base para colocar agentes para trabalhar em processos reais, sem abrir mão de controle.

## 10. Visão comercial

Comercialmente, o DeepAgent Supervisor permite uma proposta muito mais forte do que “temos IA no ERP”. O que o código suporta é algo mais sofisticado: uma camada agentic capaz de executar processos complexos, delegar tarefas, operar em segundo plano e pedir aprovação quando necessário.

Isso responde dores típicas de clientes corporativos.

- automação que não cabe em um request síncrono curto;
- processos que exigem supervisão humana em poucos pontos críticos, não em tudo;
- necessidade de rastreabilidade para auditoria;
- necessidade de usar ferramentas reais e não apenas gerar texto;
- necessidade de memória entre execuções;
- necessidade de operar tarefas rotineiras de ERP com menos esforço humano.

O discurso comercial correto não é “o agente faz tudo sozinho”. O discurso correto é “o agente executa processos duráveis com autonomia governada e pontos formais de intervenção humana”.

## 11. Visão estratégica

Estrategicamente, esta feature fortalece a plataforma porque conecta YAML-first, AST, validação semântica, runtime governado, observabilidade, background execution e HIL assíncrono em um único desenho coerente.

Isso tem impacto de longo prazo.

- Reduz acoplamento entre definição declarativa e execução concreta.
- Facilita evolução segura de middlewares e políticas de autonomia.
- Torna a plataforma mais adequada para agentes especialistas de longa duração.
- Cria base sólida para casos de uso ERP, compliance, operações e automação corporativa.

## 12. Explicação 101

Se um agente comum parece um atendente que responde perguntas, o DeepAgent Supervisor parece um coordenador de operações.

Ele recebe um objetivo, usa ferramentas, organiza tarefas, chama especialistas, consulta memória, mexe em arquivos quando pode, pausa quando precisa de aprovação e volta depois para concluir o trabalho. Por isso ele é muito mais adequado para processos reais e demorados do que um agente que só responde no mesmo request.

## 13. Limites e pegadinhas

- Ele é poderoso porque é governado. Tirar a governança destrói o valor do desenho.
- Filesystem sem permissions explícitas não é aceito.
- HIL sem interrupt_on e sem checkpointer também não é aceito.
- backend.type=store não aceita backend arbitrário; o contrato atual exige Redis via backend.redis.
- background_execution_subagent não funciona se o middleware de subagentes estiver desligado.
- async approval não significa “qualquer pessoa pode aprovar”; há contrato de canais, aprovadores e decisão permitida.
- O runtime é forte para processos longos, mas depende de tools reais do tenant para executar casos de negócio concretos.

## 14. Checklist de entendimento

- Entendi que DeepAgent é um runtime próprio, não um apelido genérico.
- Entendi por que ele é mais poderoso do que um supervisor simples.
- Entendi o papel de filesystem, shell, todo_list, skills, memória e subagentes.
- Entendi por que HIL e async approval são cruciais para operação corporativa.
- Entendi por que ele combina tão bem com background execution.
- Entendi os casos de uso ERP mais adequados para esse runtime.
- Entendi que o poder dele depende de contrato, não de autonomia solta.

## 15. Evidências no código

- src/agentic_layer/supervisor/deep_agent_supervisor.py
  - Motivo da leitura: runtime principal do DeepAgent Supervisor.
  - Símbolos relevantes: initialize, run, _create_agent, _build_background_execution_subagent_spec.
  - Comportamento confirmado: middlewares governados, cache por hash, filesystem, shell, memory, HIL, subagentes, background execution subagent e telemetria.

- src/config/agentic_assembly/ast/deepagent.py
  - Motivo da leitura: contrato declarativo do modo deepagent.
  - Símbolos relevantes: DeepAgentMiddlewaresAST, DeepAgentSupervisorAST, DeepAgentAsyncApprovalAST.
  - Comportamento confirmado: toggles de middlewares, permissions, skills, response_format, interrupt_on e async approval.

- src/config/agentic_assembly/parsers/deepagent_parser.py
  - Motivo da leitura: entrada YAML para AST especializada.
  - Símbolo relevante: DeepAgentParser.parse.
  - Comportamento confirmado: filtragem por execution.type=deepagent, defaults mínimos e rejeição de campos legados.

- src/config/agentic_assembly/validators/deepagent_semantic_validator.py
  - Motivo da leitura: coerência semântica do contrato.
  - Símbolo relevante: validações de permissions, HIL, checkpointer, memory, backend, skills e async_subagents.
  - Comportamento confirmado: fail-fast para combinações inválidas.

- src/agentic_layer/background_execution/runtime.py
  - Motivo da leitura: execução canônica em segundo plano.
  - Símbolo relevante: AgenticBackgroundExecutionRuntime.execute_run.
  - Comportamento confirmado: suporte explícito a deepagent em background, thread_id durável, snapshot obrigatório e integração com HIL assíncrono.

- src/agentic_layer/tools/system_tools/background_execution.py
  - Motivo da leitura: tools especializadas de execução em background.
  - Símbolos relevantes: BACKGROUND_EXECUTION_TOOL_IDS e create_background_execution_tools.
  - Comportamento confirmado: agendar, listar, cancelar, reagendar e consultar execuções background.

- src/api/routers/agent_router.py
  - Motivo da leitura: boundary HTTP oficial.
  - Símbolos relevantes: /agent/execute, /agent/continue, /agent/hil/decisions.
  - Comportamento confirmado: execução deepagent, continuação por thread_id e resolução segura de HIL assíncrono.
