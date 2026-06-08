# Capítulo 3 — Os Jogadores em Campo (`.claude/agents/`)

> **Posição em campo:** os agentes são os **jogadores de verdade**. Cada um tem cérebro próprio,
> contexto isolado, um conjunto de ferramentas e uma função exclusiva. Eles entram em campo quando
> convocados por uma skill (Cap. 4), jogam segundo o DNA (Cap. 1) consultando as jogadas ensaiadas
> (Cap. 2), sob o olhar do árbitro (Cap. 5). São 22 — e o segredo não é o número, é a **divisão de
> funções**: cada um faz uma coisa e a faz com rigor.

> 🧑‍💼 **RESUMO EXECUTIVO.** Em vez de uma "IA genérica que faz tudo mais ou menos", temos **22
> especialistas que fazem uma coisa muito bem**. Isso é exatamente o que uma empresa madura faz com
> pessoas: ninguém é "faz-tudo"; há o investigador, o arquiteto, o desenvolvedor, o QA, o DBA. A
> especialização é o que produz qualidade consistente e permite auditar quem fez o quê.

---

## 3.0 O princípio que rege todos eles: uma tarefa por jogador

Antes das posições, o conceito mais importante: **cada agente tem responsabilidade única (SRP) e
contexto isolado**. Quem investiga não corrige; quem corrige não se valida. Isso traz três ganhos:

1. **Foco:** o agente carrega só as regras da sua função — não se distrai.
2. **Contexto limpo:** o trabalho pesado roda numa "sala separada" e volta só com o resultado, sem
   poluir a conversa principal (e gastando menos tokens).
3. **Auditabilidade:** dá para saber exatamente qual especialista produziu cada parte.

Vamos por setores do campo.

---

## 3.1 O eixo central — a coluna vertebral do time

Estes quatro formam a "jogada que sempre se repete" (apresentada na doc-mãe). São o caminho oficial de
quase todo trabalho não-trivial.

### 🛡️ `investigar` — o ZAGUEIRO (lê o jogo, defende contra erro estrutural)

- **Função:** análise forense do estado **real** do código, fluxo, YAML, logs e observabilidade.
  Investiga, **não** planeja nem corrige. Entrega um relatório robusto com achados estruturados,
  níveis de confiança e evidência (path + linhas).
- **Quando entra:** no início de tudo, para levantar a verdade antes de qualquer decisão de mudança.
- **Grupos de instrução que o definem:**
  - **3 fases obrigatórias** (mapeamento → fechamento de lacunas → auditoria de observabilidade) que
    não podem ser puladas nem mescladas.
  - **Zero alucinação:** proibido especular, presumir ou deduzir de nome/comentário; toda afirmação
    precisa de `read` com filepath e linhas.
  - **Escopo efetivo:** define no início o que é e o que não é o alvo.
  - **Materialização obrigatória:** o relatório precisa existir como arquivo em `.sandbox/`, não só no chat.
- **Problema que evita:** **alucinação de estado** (tratar código que "deveria existir" como existente)
  e **falsa confiança** (conclusão "alta" sem ter lido o código-chave).
- **Valor:** evidência sobre suposição; rastreabilidade por path/linha.
- **Passe:** entrega a bola para o `planejar`. Investigação fraca = plano fraco. Por isso o zagueiro é
  a base — se ele erra na saída de bola, o time inteiro sofre.

### 🎯 `planejar` — o MEIA-ARMADOR (transforma leitura em jogada)

- **Função:** a partir da investigação pronta, **avalia criticamente** a proposta (Sim/Não/Depende +
  prós/contras/riscos) e a transforma em um plano seguro, incremental e validável, com tarefas
  T1..Tn, critérios de aceite, validação e rollback. Não executa nem reinvestiga.
- **Quando entra:** depois do `investigar`, quando se decide transformar evidência em ação.
- **Grupos de instrução que o definem:**
  - **T1 e T2 sempre obrigatórias:** T1 cria a seção `# Estratégia/Recomendacoes` (a bússola); T2
    mapeia os testes existentes e os classifica (preservar/remover/criar/ajustar) **antes** de qualquer
    mudança.
  - **Anti-omissão silenciosa:** todo item relevante da investigação **precisa** ter destino — vira
    tarefa, validação, bloqueio ou "fora do escopo com justificativa". Nada desaparece.
  - **Postura crítica obrigatória:** antes de planejar, dizer francamente se a ideia é boa.
- **Problema que evita:** **escopo inflado** (plano que adiciona o que não foi investigado) e **tarefa
  genérica** ("ajustar X" sem dizer como comprovar).
- **Valor:** incrementalidade validável; rastreabilidade de origem (cada tarefa aponta para um achado).
- **Passe:** entrega o plano para o `implementar`. Se a investigação foi incompleta, **falha fechado**
  em vez de inventar — pede investigação complementar.

### ⚽ `implementar` — o ATACANTE (finaliza: código vivo em runtime)

- **Função:** executa o plano, tarefa a tarefa, com qualidade. Lê o código real, implementa
  incrementalmente, cria testes antes da mudança comportamental, instrumenta logs e **prova o
  comportamento no boundary oficial**.
- **Quando entra:** com o plano pronto, para fazer o gol — que aqui significa **código novo realmente
  ativo no caminho oficial**.
- **Grupos de instrução que o definem:**
  - **Lê `# Estratégia/Recomendacoes` antes de tocar qualquer arquivo:** a bússola vence a tarefa
    isolada; conflito entre as duas = parar e resolver pelo objetivo central.
  - **Contrato de definição de pronto:** antes de marcar CONCLUÍDA, cumpre `definicao-de-pronto.md`
    (check discriminante, teste de regressão, auditoria de observabilidade). Falhou um gate? Vira
    BLOQUEADA, não fechamento.
  - **Default anti-legado:** não mantém fallback, dual-read ou caminho antigo sem pedido explícito.
  - **Gates de reuso/banco/YAML/LangChain:** lê o contrato antes de criar algo novo ou tocar subsistema.
- **Problema que evita:** **regressão silenciosa** (mudança sem teste, wiring só no teste), **legado
  acumulado** e **falso-verde**.
- **Valor:** qualidade observável (o log prova que o caminho executou), incrementalismo real.
- **Passe:** entrega para o `validar-entrega`.

### 🧤 `validar-entrega` — o GOLEIRO (defende a porta da qualidade)

- **Função:** o último a aprovar. Confronta o que foi implementado contra o plano e a investigação,
  exige evidência real (código lido, testes, logs, boundary oficial) e dá um status explícito:
  **APROVADO / APROVADO COM RESSALVAS / REPROVADO / BLOQUEADO**.
- **Quando entra:** ao fim da implementação.
- **Grupos de instrução que o definem:**
  - **Hierarquia de verdade absoluta:** código executável > teste > log > plano > doc. Teste verde
    **não** prova fluxo real.
  - **Validação ponta a ponta por tarefa:** cada T1..Tn é conferida individualmente.
  - **Pode acionar especialistas:** chama `validar-log` e `validar-observabilidade` como evidência
    subordinada (não terceiriza a decisão).
  - **Se reprova, gera inventário de não-entregas e aciona o `planejar`** para um novo plano focado.
- **Problema que evita:** **falso-verde** chegando à produção; legado não removido; observabilidade
  insuficiente; reuso negligenciado.
- **Valor:** qualidade irreversível; proteção contra o auto-engano do executor.

> 🛠️ **Por que essa formação é poderosa.** Os quatro têm **contratos opostos de propósito**: o
> investigar não pode planejar, o planejar não pode executar, o validar não pode implementar. Essa
> oposição é intencional — cria *checagem independente* em cada troca de pé. É o equivalente a
> "segregação de funções" em controles internos de uma empresa: quem aprova não é quem executa.

---

## 3.2 A defesa especializada — observabilidade (o setor do raio-X)

Cinco agentes que garantem que o sistema seja **diagnosticável**. Giram em torno do `correlation_id` e
dos logs canônicos.

### 🔍 `analisar-log` — o OLHEIRO/SCOUT (primeiro sensor)

Lê logs reais por `correlation_id` usando a CLI oficial `python -m src.log_analyzer` — **sem corrigir
código**. Traduz pergunta em linguagem natural para parâmetros da CLI, consolida a linha do tempo e
**declara o que o log prova e o que não prova**. Detalhe fino: valida **paralelismo real** por
sobreposição temporal, não por configuração nominal ("a UI disse 5 workers" ≠ "5 workers rodaram
juntos de verdade").
- **Evita:** análise manual de log fraca, conclusão por "parece". **Valor:** debug offline com
  evidência estruturada e reprodutível.

### 🧱 `validar-log` — o ZAGUEIRO DE MARCAÇÃO (audita a escrita do log)

Audita se os logs de um escopo seguem o contrato (logger oficial, `correlation_id` preservado,
`event_name` correto, builders canônicos). Regra de ouro: **falha fechada anti-amostragem** — audita
**100%** do escopo, não para por "já vi o padrão". Caça `logger.error` dentro de `except` e
`event_name` ausente.
- **Evita:** logs "bonitos" que violam o contrato e quebram a análise forense. **Valor:**
  observabilidade garantida por construção.

### 🧱 `validar-observabilidade` — o ZAGUEIRO CENTRAL (audita a cobertura)

Vai além: valida se **todo o fluxo** está coberto — parâmetros, decisões, ifs, chamadas externas,
início/fim de processos e jobs, telemetria. A pergunta que ele responde: *"um operador consegue
reconstruir o que aconteceu, como e por quê, sem abrir o código?"* Também com falha fechada.
- **Evita:** cobertura parcial com "buracos" no fluxo. **Valor:** debug offline **completo**.

> 🛠️ A divisão `validar-log` × `validar-observabilidade` é cirúrgica: um valida **se o log foi
> escrito certo** (contrato), o outro valida **se há log suficiente** (cobertura). Dois defeitos
> diferentes, dois especialistas.

### 🩺 `corrigir-erros-com-log` — o CIRURGIÃO (resolve a causa raiz)

Diagnostica e corrige bugs usando o log real como fonte de verdade. **Corrige definitivamente** — é
proibido mascarar com fallback/degraded-mode. Reconcilia "história do log" com "fluxo do código",
instrumenta logs faltantes, valida com a suíte, e mantém **memória de rodada** (diário + lições
aprendidas) para não repetir hipótese já falsificada.
- **Evita:** correção superficial que mascara a causa; loop infinito de tentativa-erro.
- **Valor:** correção definitiva, debug determinístico.

### 🟨 `testar-cli-log-analyzer` — o VAR (valida a própria ferramenta de análise)

Testa a CLI `src.log_analyzer` contra uma **verdade manual independente** construída lendo o JSONL
bruto **fora** do módulo. Princípio: "a CLI é ré, não juíza" — se a CLI e a verdade manual divergem, a
CLI está errada até prova em contrário. Loop corretivo até a matriz inteira bater.
- **Evita:** confiar cegamente na ferramenta que todos os outros agentes usam para diagnosticar.
- **Valor:** confiança na ferramenta de observabilidade (sem ela, todo o setor de raio-X seria suspeito).

---

## 3.3 O departamento de futebol — operação real (o estádio e o gramado)

Quatro agentes que tocam **infraestrutura real** (banco, cache, fila, scheduler). Compartilham um DNA
comum poderoso: **snapshot antes de mudar, impacto mínimo, mutação auditável, dry-run + flag explícita,
nunca tocar dado de terceiro.**

| Agente | Posição | Cuida de | Trava de segurança característica |
|---|---|---|---|
| `gerenciar-postgresql` | Zagueiro | PostgreSQL real (DSN canônica) | Escrita só quando leitura não basta; nunca expõe credencial; pager off |
| `gerenciar-redis` | Volante | Cache Redis real | Leitura antes de mutar; `delete` exige padrão + `--limit` |
| `gerenciar-job-core-ingestao` | Técnico do setor | Jobs, ingestão e RabbitMQ | Tudo nasce em **dry-run**; só executa com `--execute`/`--apply`; prefere reconciliação auditável à exclusão física |
| `gerenciar-scheduler` | Meia | Scheduler universal | Inspeção read-only obrigatória **antes** de cancelar/purgar |

**Problema que todos evitam:** decisão baseada em suposição sobre o estado real ("acho que o dado está
lá") e mutação destrutiva sem rastro. Em ambiente real, isso é a diferença entre uma operação cirúrgica
e um acidente.

**Valor:** verdade operacional verificável; auditoria de toda mudança; isolamento entre tentativas.

> 🧑‍💼 **RESUMO EXECUTIVO.** Estes quatro são os que tocam o "mundo real" (dados, filas, agendamentos).
> Por isso são os mais paranoicos: fotografam antes de mexer, mexem o mínimo, e registram tudo. É a
> mesma disciplina de um técnico que mexe num servidor de produção com checklist e backup — nunca no
> improviso.

---

## 3.4 Treinos e amistosos — testes de fluxo real

Agentes que **provam que o time funciona** antes do jogo oficial.

- **`executar-testes` (🧤 goleiro do treino):** estabiliza a suíte oficial até **zero erro**, auditada
  por telemetria. Não refatora arquitetura nem inventa cobertura — objetivo binário: suíte verde de
  verdade. Decide entre "Hipótese A: bug no código" vs "Hipótese B: teste desatualizado".
- **`criar-testes` (⚽ atacante do treino):** amplia cobertura **só quando reduz risco real**. Gate de
  decisão antes de cada teste (contrato estável? sobrevive a refatoração legítima?). Escolhe a família
  (unit/integration/contract/…) pela natureza, não pela pasta. Mock só para I/O externo — nunca simula
  framework.
- **`testar-ingestao-dnit` (⚽ atacante de jogo oficial):** roda a ingestão real pela UI e exige **três
  provas independentes** — UI (correlation_id capturado no clique real), log (raio-X zero erro) e
  paralelismo real (sobreposição temporal por documento). Loop corretivo até SUCESSO ou impedimento
  comprovado. Status **binário**: SUCESSO ou ERRO, sem "parcial".
- **`testar-cancelamento-ingestao-dnit`:** testa **exclusivamente** a capacidade de **parar** quando
  pedido — prova no log que o processamento de fato cessou (não só que a UI disse "cancelado").
- **`testar-webchat`, `testar-webchat-dnit`, `testar-nl2sql`, `testar-nl2yaml`, `testar-paginas-web-projeto`:**
  validam interfaces reais (chat, geração de SQL/YAML, páginas), sempre capturando o `correlation_id`
  da response real e corrigindo por log.

**Problema que todos evitam:** o "passou no meu computador". Estes agentes provam comportamento em
**execução real**, com evidência de três fontes, não em simulação.

**Valor:** confiança de que o fluxo crítico funciona ponta a ponta na vida real.

---

## 3.5 Comissão técnica e bastidores — governança e conhecimento

Seis agentes que cuidam do conhecimento e da consistência institucional.

- **`documentar` (🎙️ imprensa/comunicação):** produz documentação profunda (técnica + executiva +
  comercial), lendo o **código como fonte de verdade**, nunca a doc existente. Proíbe documento raso e
  inventário de arquivos; exige proporção mínima de conceito/fluxo.
- **`sincronizar-documentacao` (🧤 goleiro da doc):** mantém `docs/` sincronizado com o código real ao
  longo do tempo, evitando a "doc que mente". Gera FAQ de onboarding para o consultor júnior.
- **`inventario-componentes-genericos` (🎯 meia construtor):** cataloga o que já existe e pode ser
  reutilizado, alimentando `docs/README-TOOLS-LIB.md`. É o "olheiro do próprio elenco" — base do gate
  de reuso.
- **`inventario-yaml` (🛡️ auditor):** caça **só problemas** de YAML × código — chaves órfãs, ausentes,
  duplicadas, desalinhamento com a AST, testes ausentes. Não lista o que está certo.
- **`validar-instructions` (⚖️ ouvidoria do regulamento):** audita as próprias instruções em busca de
  contradição, redundância, ambiguidade — consultando a doc oficial mais recente. Registra em
  `bad-instructions.md`. **É o time se auto-corrigindo.**
- **`analisar-produto` (🔭 scout de mercado):** investiga a solução sob a ótica técnica, comercial e
  competitiva; separa **documentado × implementado × não encontrado**; produz insumos comerciais.

**Problema que todos evitam:** o apodrecimento do conhecimento — doc desatualizada, duplicação por
desconhecimento do que existe, instruções contraditórias, promessa comercial que o produto não cumpre.

**Valor:** o conhecimento da empresa fica **escrito, fiel e consistente** — não na cabeça de poucos.

---

## 3.6 A tabela da escalação completa

| Agente | Posição | Função em uma linha |
|---|---|---|
| investigar | Zagueiro | Levanta a verdade do estado atual, com evidência |
| planejar | Meia-armador | Transforma evidência em plano seguro e incremental |
| implementar | Atacante | Põe código novo ativo no caminho oficial |
| validar-entrega | Goleiro | Aprova ou reprova com base em evidência real |
| analisar-log | Olheiro | Lê log por correlation_id, sem corrigir |
| validar-log | Zagueiro marcação | Audita se o log foi escrito segundo o contrato |
| validar-observabilidade | Zagueiro central | Audita se há log suficiente para reconstruir o fluxo |
| corrigir-erros-com-log | Cirurgião | Corrige a causa raiz usando o log como verdade |
| testar-cli-log-analyzer | VAR | Valida a ferramenta de análise de logs |
| gerenciar-postgresql | Zagueiro infra | Opera o PostgreSQL real, auditável |
| gerenciar-redis | Volante infra | Opera o cache Redis real, mínimo impacto |
| gerenciar-job-core-ingestao | Técnico do setor | Opera jobs, ingestão e RabbitMQ, dry-run primeiro |
| gerenciar-scheduler | Meia infra | Opera o scheduler, inspeção antes de mutar |
| executar-testes | Goleiro do treino | Estabiliza a suíte até zero erro |
| criar-testes | Atacante do treino | Amplia cobertura só onde reduz risco |
| testar-ingestao-dnit | Atacante oficial | Prova a ingestão real por três fontes |
| testar-cancelamento-ingestao-dnit | Atacante especialista | Prova que o sistema para quando mandado |
| testar-webchat / -dnit | Pontas | Provam os chats reais por log |
| testar-nl2sql / nl2yaml | Pontas | Provam geração de SQL/YAML revisável |
| testar-paginas-web-projeto | Ponta | Prova páginas reais no navegador |
| documentar | Imprensa | Documentação profunda a partir do código |
| sincronizar-documentacao | Goleiro da doc | Mantém docs fiel ao código |
| inventario-componentes-genericos | Meia construtor | Cataloga o reutilizável |
| inventario-yaml | Auditor | Caça problemas de YAML × código |
| validar-instructions | Ouvidoria | Audita as próprias instruções |
| analisar-produto | Scout de mercado | Posiciona o produto com evidência |

---

## 3.7 O que levar desta posição para a aula

- Os agentes são **especialistas de função única e contexto isolado** — o oposto da "IA faz-tudo".
- O **eixo central** (investigar → planejar → implementar → validar) tem **contratos de propósito
  opostos** que criam checagem independente a cada passe — segregação de funções.
- A **defesa de observabilidade** garante que o sistema seja diagnosticável; o **departamento de
  infra** garante operação real auditável; os **treinos** provam o fluxo na vida real; a **comissão
  técnica** mantém o conhecimento fiel.
- O padrão transversal mais valioso: **snapshot/leitura antes de agir, dry-run antes de mutar, três
  fontes de prova, status binário ou explícito.**

**Próximo:** [Capítulo 4 — A Convocação (`.claude/skills/`)](capitulo-04-skills-convocacao.md).
</content>
