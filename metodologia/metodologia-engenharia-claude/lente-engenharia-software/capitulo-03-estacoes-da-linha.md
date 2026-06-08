# Capítulo 3 — As Estações da Linha (`.claude/agents/`) ⭐

> 🏭 [↩ Doc-mãe da lente](README.md) · [↩ Índice da metodologia](../README.md)

> **Parte da fábrica:** os agentes são as **estações de trabalho de verdade**. Cada uma tem cérebro
> próprio, contexto isolado, um conjunto de ferramentas e uma função exclusiva. Entram em operação quando
> acionadas por uma ordem de produção (Cap. 4), produzem segundo a Norma (Cap. 1) consultando as
> especificações de processo (Cap. 2), sob a vigilância dos dispositivos à prova de erro (Cap. 5). São 22
> — e o segredo não é o número, é a **divisão de funções**: cada estação faz uma coisa, e a faz com rigor.

> 🧑‍💼 **RESUMO EXECUTIVO.** Em vez de uma "IA genérica que faz tudo mais ou menos", temos **22 postos
> especializados que fazem uma coisa muito bem**. É exatamente o que uma fábrica madura faz: ninguém é
> "faz-tudo"; há o inspetor de entrada, o engenheiro de processo, o montador, o controle de qualidade, o
> laboratório, a manutenção. A especialização é o que produz qualidade consistente e permite auditar quem
> fez o quê em cada peça.

---

## 3.0 O princípio que rege todas: uma função por estação

Antes das estações, o conceito mais importante: **cada agente tem responsabilidade única (SRP) e contexto
isolado**. Quem inspeciona não monta; quem monta não assina o próprio controle de qualidade. Três ganhos:

1. **Foco:** a estação carrega só as especificações da sua função — não se distrai.
2. **Contexto limpo:** o trabalho pesado roda numa "célula isolada" e volta só com o resultado, sem
   poluir a linha principal (e gastando menos tokens).
3. **Auditabilidade:** dá para saber exatamente qual estação produziu cada parte da peça.

Vamos por setores da fábrica.

---

## 3.1 A célula central — o coração da produção

Estas quatro formam o fluxo oficial de quase todo trabalho não-trivial.

### 🔬 `investigar` — INSPEÇÃO DE ENTRADA + ENGENHARIA DE PROCESSO

- **Função:** medição forense do estado **real** do código, fluxo, YAML, logs e observabilidade.
  Inspeciona, **não** planeja nem fabrica. Entrega um laudo robusto com achados estruturados, níveis de
  confiança e evidência (path + linhas).
- **Quando entra:** no início de tudo, para medir a matéria-prima antes de qualquer decisão de produção.
- **Procedimentos que a definem:**
  - **3 fases obrigatórias** (mapeamento → fechamento de lacunas → auditoria de observabilidade) que não
    podem ser puladas nem mescladas.
  - **Zero alucinação:** proibido especular, presumir ou deduzir de nome/comentário; toda afirmação
    precisa de `read` com filepath e linhas.
  - **Escopo efetivo:** define no início o que é e o que não é o alvo.
  - **Materialização obrigatória:** o laudo precisa existir como arquivo em `.sandbox/`, não só no chat.
- **Defeito que evita:** **medição inventada** (tratar peça que "deveria existir" como existente) e
  **falsa confiança** (laudo "alta confiança" sem ter lido o código-chave).
- **Passe:** entrega ao PCP (`planejar`). Medição ruim = roteiro ruim. Por isso a inspeção de entrada é a
  base — se a matéria-prima entra mal medida, a linha inteira sofre.

### 📋 `planejar` — PCP (PLANEJAMENTO E CONTROLE DA PRODUÇÃO)

- **Função:** a partir da inspeção pronta, **avalia criticamente** a proposta (Sim/Não/Depende +
  prós/contras/riscos) e a transforma em um roteiro de fabricação seguro, incremental e validável, com
  tarefas T1..Tn, critérios de aceite, validação e rollback. Não fabrica nem re-inspeciona.
- **Quando entra:** depois do `investigar`, quando se decide transformar a medição em produção.
- **Procedimentos que a definem:**
  - **T1 e T2 sempre obrigatórias:** T1 cria a seção `# Estratégia/Recomendacoes` (a ordem de serviço);
    T2 mapeia os ensaios existentes e os classifica (preservar/remover/criar/ajustar) **antes** de
    qualquer mudança.
  - **Anti-omissão silenciosa:** todo achado da inspeção **precisa** ter destino — vira tarefa,
    validação, bloqueio ou "fora do escopo com justificativa". Nada some.
  - **Postura crítica obrigatória:** antes de planejar, dizer francamente se a ideia é boa.
- **Defeito que evita:** **escopo inflado** (roteiro que adiciona o que não foi medido) e **tarefa
  genérica** ("ajustar X" sem dizer como comprovar).
- **Passe:** entrega o roteiro à fabricação (`implementar`). Se a inspeção foi incompleta, **falha
  fechado** em vez de inventar — pede inspeção complementar.

### 🛠️ `implementar` — A ESTAÇÃO DE FABRICAÇÃO/MONTAGEM

- **Função:** executa o roteiro, tarefa a tarefa, com qualidade. Lê o código real, fabrica
  incrementalmente, cria ensaios antes da mudança comportamental, instrumenta logs e **prova a peça
  instalada na linha oficial**.
- **Quando entra:** com o roteiro pronto, para produzir a peça boa — que aqui significa **código novo
  realmente ativo no caminho oficial**.
- **Procedimentos que a definem:**
  - **Lê a ordem de serviço (`# Estratégia/Recomendacoes`) antes de tocar qualquer arquivo:** ela vence a
    tarefa isolada; conflito entre as duas = parar e resolver pelo objetivo central.
  - **Procedimento de inspeção final:** antes de marcar CONCLUÍDA, cumpre `definicao-de-pronto.md` (check
    discriminante, teste de regressão, auditoria de observabilidade). Falhou um gate? Vira BLOQUEADA.
  - **Default anti-legado:** não mantém fallback, dual-read ou caminho antigo sem pedido explícito.
  - **Gates de reuso/banco/YAML/LangChain:** lê a especificação antes de fabricar algo novo ou tocar
    subsistema.
- **Defeito que evita:** **regressão silenciosa** (mudança sem ensaio, peça só instalada no ensaio),
  **estoque de legado** e **falso-verde**.
- **Passe:** entrega ao controle de qualidade (`validar-entrega`).

### ✅ `validar-entrega` — CONTROLE DE QUALIDADE DE SAÍDA (inspeção final)

- **Função:** o último a aprovar. Confronta o que foi fabricado contra o roteiro e a inspeção de entrada,
  exige evidência real (código lido, ensaios, logs, linha oficial) e dá um status explícito:
  **APROVADO / APROVADO COM RESSALVAS / REPROVADO / BLOQUEADO**.
- **Quando entra:** ao fim da fabricação.
- **Procedimentos que a definem:**
  - **Hierarquia de verdade absoluta:** código executável > teste > log > roteiro > doc. Ensaio verde
    **não** prova linha real.
  - **Validação ponta a ponta por tarefa:** cada T1..Tn é conferida individualmente.
  - **Pode acionar o laboratório:** chama `validar-log` e `validar-observabilidade` como evidência
    subordinada (não terceiriza a decisão).
  - **Se reprova, gera inventário de não-conformidades e devolve ao PCP (`planejar`)** para um novo
    roteiro focado.
- **Defeito que evita:** **falso-verde** chegando ao cliente; legado não removido; observabilidade
  insuficiente; reuso negligenciado.
- **Valor:** qualidade irreversível; proteção contra o auto-engano da estação que fabricou.

> 🛠️ **Por que essa célula é poderosa.** As quatro têm **contratos opostos de propósito**: a inspeção
> não pode planejar, o PCP não pode fabricar, o controle de qualidade não pode montar. Essa oposição é
> intencional — cria *checagem independente* em cada passagem de bastão. É a **segregação de funções** dos
> controles internos de uma empresa: quem aprova não é quem executa.

---

## 3.2 O laboratório de metrologia e qualidade — observabilidade

Cinco estações que garantem que o sistema seja **diagnosticável**. Giram em torno do `correlation_id`
(número de série) e dos logs canônicos (registro de processo).

### 🔎 `analisar-log` — METROLOGIA (leitura dos instrumentos)

Lê os registros de processo reais por `correlation_id` usando a CLI oficial `python -m src.log_analyzer`
— **sem corrigir nada**. Traduz pergunta em linguagem natural para parâmetros da CLI, consolida a linha
do tempo e **declara o que o registro prova e o que não prova**. Detalhe fino: valida **paralelismo real**
por sobreposição temporal, não por configuração nominal ("a UI disse 5 workers" ≠ "5 workers rodaram
juntos de verdade").
- **Evita:** leitura de instrumento fraca, conclusão por "parece". **Valor:** debug offline com evidência
  reprodutível.

### 📐 `validar-log` — CALIBRAÇÃO DO REGISTRO (audita a escrita do log)

Audita se os logs de um escopo seguem a especificação (logger oficial, `correlation_id` preservado,
`event_name` correto, builders canônicos). Regra de ouro: **falha fechada anti-amostragem** — audita
**100%** do escopo, não para por "já vi o padrão". Caça `logger.error` dentro de `except` e `event_name`
ausente.
- **Evita:** registros "bonitos" que violam a especificação e quebram a análise forense. **Valor:**
  instrumentação correta por construção.

### 📊 `validar-observabilidade` — AUDITORIA DE COBERTURA DE SENSORES

Vai além: valida se **todo o fluxo** está instrumentado — parâmetros, decisões, ifs, chamadas externas,
início/fim de processos e jobs, telemetria. A pergunta que responde: *"um operador consegue reconstruir o
que aconteceu, como e por quê, sem abrir o código?"* Também com falha fechada.
- **Evita:** cobertura parcial com "pontos cegos" no processo. **Valor:** debug offline **completo**.

> 🛠️ A divisão `validar-log` × `validar-observabilidade` é cirúrgica: um valida **se o registro foi
> escrito certo** (especificação), o outro valida **se há registro suficiente** (cobertura). Dois
> defeitos diferentes, duas estações.

### 🔧 `corrigir-erros-com-log` — ENGENHARIA DE FALHAS (análise de causa raiz / RCA)

Diagnostica e corrige defeitos usando o registro real como fonte de verdade. **Corrige definitivamente** —
proibido mascarar com fallback/degraded-mode. Reconcilia "história do registro" com "fluxo do código",
instrumenta logs faltantes, valida com a suíte, e mantém **memória de rodada** (diário + lições) para não
repetir hipótese já falsificada.
- **Evita:** correção superficial que mascara a causa; loop infinito de tentativa-erro.
- **Valor:** correção definitiva, debug determinístico.

### 🧪 `testar-cli-log-analyzer` — CALIBRAÇÃO DO PRÓPRIO INSTRUMENTO

Testa a CLI `src.log_analyzer` contra uma **verdade manual independente** construída lendo o JSONL bruto
**fora** do módulo. Princípio: "o instrumento é réu, não juiz" — se a CLI e a verdade manual divergem, a
CLI está errada até prova em contrário. Loop corretivo até a matriz inteira bater.
- **Evita:** confiar cegamente no instrumento que todas as outras estações usam para medir.
- **Valor:** confiança na ferramenta de observabilidade (sem ela, todo o laboratório seria suspeito).

---

## 3.3 Manutenção e operação do chão de fábrica — infra real

Quatro estações que tocam **infraestrutura real** (banco, cache, fila, scheduler). Compartilham um
procedimento poderoso: **fotografar antes de mexer (snapshot), impacto mínimo, mutação auditável, dry-run
+ flag explícita (lockout-tagout), nunca tocar peça de terceiro.**

| Estação | Cuida de | Trava de segurança característica |
|---|---|---|
| `gerenciar-postgresql` | PostgreSQL real (DSN canônica) | Escrita só quando leitura não basta; nunca expõe credencial; pager off |
| `gerenciar-redis` | Cache Redis real | Leitura antes de mutar; `delete` exige padrão + `--limit` |
| `gerenciar-job-core-ingestao` | Jobs, ingestão e RabbitMQ | Tudo nasce em **dry-run**; só executa com `--execute`/`--apply`; prefere reconciliação auditável à exclusão física |
| `gerenciar-scheduler` | Scheduler universal | Inspeção read-only obrigatória **antes** de cancelar/purgar |

**Defeito que todas evitam:** decisão baseada em suposição sobre o estado real ("acho que o dado está
lá") e mutação destrutiva sem rastro. No chão de fábrica, é a diferença entre manutenção planejada e
acidente de máquina.

> 🧑‍💼 **RESUMO EXECUTIVO.** Estas quatro tocam o "mundo real" (dados, filas, agendamentos). Por isso são
> as mais paranoicas: fotografam antes de mexer, mexem o mínimo, e registram tudo. É a disciplina de
> mexer numa máquina ligada com checklist e lockout — nunca no improviso.

---

## 3.4 Ensaios e lote piloto — testes de fluxo real

Estações que **provam que a linha funciona** antes da produção oficial.

- **`executar-testes` (banco de provas):** estabiliza a suíte oficial até **zero defeito**, auditada por
  telemetria. Não reprojeta nem inventa cobertura — objetivo binário: suíte verde de verdade. Decide
  entre "Hipótese A: defeito no código" vs "Hipótese B: ensaio desatualizado".
- **`criar-testes` (projeto de novos ensaios):** amplia cobertura **só quando reduz risco real**. Gate de
  decisão antes de cada ensaio (contrato estável? sobrevive a refatoração legítima?). Escolhe a família
  (unit/integration/contract/…) pela natureza, não pela pasta. Mock só para I/O externo — nunca simula o
  material (framework).
- **`testar-ingestao-dnit` (lote piloto oficial):** roda a ingestão real pela UI e exige **três provas
  independentes** — UI (correlation_id capturado no clique real), log (raio-X zero erro) e paralelismo
  real (sobreposição temporal por documento). Loop corretivo até SUCESSO ou impedimento comprovado.
  Status **binário**: SUCESSO ou ERRO, sem "parcial".
- **`testar-cancelamento-ingestao-dnit`:** ensaia **exclusivamente** a capacidade de **parar** quando
  pedido — prova no log que o processamento de fato cessou (não só que a UI disse "cancelado").
- **`testar-webchat`, `testar-webchat-dnit`, `testar-nl2sql`, `testar-nl2yaml`, `testar-paginas-web-projeto`:**
  validam interfaces reais (chat, geração de SQL/YAML, páginas), sempre capturando o `correlation_id` da
  response real e corrigindo por log.

**Defeito que todas evitam:** o "passou na minha bancada". Estas estações provam comportamento em
**execução real**, com evidência de três fontes, não em simulação.

---

## 3.5 Engenharia da qualidade e documentação — governança e conhecimento

Seis estações que cuidam do conhecimento e da consistência institucional.

- **`documentar`:** produz documentação profunda (técnica + executiva + comercial), lendo o **código como
  fonte de verdade**, nunca a doc existente. Proíbe documento raso e inventário de arquivos.
- **`sincronizar-documentacao`:** mantém `docs/` sincronizado com o código real ao longo do tempo,
  evitando a "doc que mente". Gera FAQ de onboarding para o consultor júnior.
- **`inventario-componentes-genericos`:** cataloga o que já existe e pode ser reutilizado, alimentando
  `docs/README-TOOLS-LIB.md`. É o "inventário do almoxarifado" — base do gate de reuso.
- **`inventario-yaml`:** caça **só problemas** de YAML × código — chaves órfãs, ausentes, duplicadas,
  desalinhamento com a AST, ensaios ausentes. Não lista o que está certo.
- **`validar-instructions`:** audita as próprias instruções (a Norma e os SOPs) em busca de contradição,
  redundância, ambiguidade — consultando a doc oficial mais recente. Registra em `bad-instructions.md`.
  **É a fábrica auditando o próprio processo.**
- **`analisar-produto`:** investiga a solução sob a ótica técnica, comercial e competitiva; separa
  **documentado × implementado × não encontrado**; produz insumos comerciais.

**Defeito que todas evitam:** o apodrecimento do conhecimento — doc desatualizada, duplicação por
desconhecimento do que existe, instruções contraditórias, promessa comercial que o produto não cumpre.

---

## 3.6 A tabela completa das estações

| Estação | Setor | Função em uma linha |
|---|---|---|
| investigar | Inspeção de entrada | Mede a verdade do estado atual, com evidência |
| planejar | PCP | Transforma medição em roteiro seguro e incremental |
| implementar | Fabricação | Põe peça nova ativa no caminho oficial |
| validar-entrega | Controle de qualidade | Aprova ou rejeita o lote com base em evidência real |
| analisar-log | Metrologia | Lê o registro por correlation_id, sem corrigir |
| validar-log | Calibração do registro | Audita se o log foi escrito segundo a especificação |
| validar-observabilidade | Auditoria de sensores | Audita se há registro suficiente para reconstruir o fluxo |
| corrigir-erros-com-log | Engenharia de falhas (RCA) | Corrige a causa raiz usando o registro como verdade |
| testar-cli-log-analyzer | Calibração do instrumento | Valida a ferramenta de análise de logs |
| gerenciar-postgresql | Manutenção (banco) | Opera o PostgreSQL real, auditável |
| gerenciar-redis | Manutenção (cache) | Opera o cache Redis real, mínimo impacto |
| gerenciar-job-core-ingestao | Manutenção (jobs/fila) | Opera jobs, ingestão e RabbitMQ, dry-run primeiro |
| gerenciar-scheduler | Manutenção (scheduler) | Opera o scheduler, inspeção antes de mutar |
| executar-testes | Banco de provas | Estabiliza a suíte até zero defeito |
| criar-testes | Projeto de ensaios | Amplia cobertura só onde reduz risco |
| testar-ingestao-dnit | Lote piloto oficial | Prova a ingestão real por três fontes |
| testar-cancelamento-ingestao-dnit | Lote piloto (parada) | Prova que o sistema para quando mandado |
| testar-webchat / -dnit | Ensaios de interface | Provam os chats reais por log |
| testar-nl2sql / nl2yaml | Ensaios de interface | Provam geração de SQL/YAML revisável |
| testar-paginas-web-projeto | Ensaio de interface | Prova páginas reais no navegador |
| documentar | Documentação técnica | Documentação profunda a partir do código |
| sincronizar-documentacao | Documentação técnica | Mantém docs fiel ao código |
| inventario-componentes-genericos | Engenharia da qualidade | Cataloga o reutilizável (o almoxarifado) |
| inventario-yaml | Engenharia da qualidade | Caça problemas de YAML × código |
| validar-instructions | Auditoria de processo | Audita as próprias instruções |
| analisar-produto | Inteligência de mercado | Posiciona o produto com evidência |

---

## 3.7 O que levar deste capítulo

- Os agentes são **estações de função única e contexto isolado** — o oposto da "IA faz-tudo".
- A **célula central** (investigar → planejar → implementar → validar) tem **contratos de propósito
  opostos** que criam checagem independente a cada passagem de bastão — segregação de funções.
- O **laboratório** garante diagnóstico; a **manutenção** garante operação real auditável; os **ensaios**
  provam a linha na vida real; a **engenharia da qualidade** mantém o conhecimento fiel.
- O padrão transversal mais valioso: **medir/ler antes de agir, dry-run antes de mutar, três fontes de
  prova, status binário ou explícito.**

**Próximo:** [Capítulo 4 — A Ordem de Produção (`.claude/skills/`)](capitulo-04-ordem-de-producao.md).
