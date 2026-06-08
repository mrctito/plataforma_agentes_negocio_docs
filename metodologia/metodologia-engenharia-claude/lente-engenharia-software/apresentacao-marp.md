---
marp: true
theme: default
paginate: true
size: 16:9
header: 'A Fábrica de Software'
footer: 'Metodologia de Engenharia de Agentes — Prometeu'
style: |
  section { font-size: 26px; }
  section.lead { text-align: center; }
  h1 { color: #1f4e3d; }
  h2 { color: #1f4e3d; }
  strong { color: #b5651d; }
  section.lead h1 { font-size: 52px; }
  code { background: #eef5f1; }
---

<!--
COMO USAR ESTE ARQUIVO (Marp)
- Conteúdo visível = o que vai na TELA.
- Texto dentro de comentário HTML (como este) = NOTAS DO APRESENTADOR (só você vê no modo apresentador).
- Exportar: ver instruções no fim do arquivo ou em README.md.
- Onde houver "[mostrar Diagrama N de ../base-conceitual/diagramas.md]", abra o diagrama Mermaid correspondente.
- Esta é a LENTE FÁBRICA DE SOFTWARE. Existe uma lente paralela (futebol) com o mesmo conteúdo.
-->

# A Fábrica de Software

## Como transformamos a IA em uma linha de produção com controle de qualidade

*Metodologia de Engenharia de Agentes — Prometeu*

<!--
FALA: "Hoje eu NÃO vou ensinar 'o que é o Claude' nem 'o que é um agente' — isso está no site da Anthropic.
Vou mostrar a engenharia por trás de cada arquivo de configuração. E vou usar uma metáfora que mapeia quase
1:1 na engenharia de software de verdade: uma FÁBRICA enxuta — linha de montagem, controle de qualidade,
dispositivos à prova de erro e melhoria contínua."
-->

---

<!-- _class: lead -->

# Qual a diferença entre

## "usar IA pra programar"
## e "fazer engenharia com IA"?

<!--
INTERAÇÃO: jogue a pergunta, deixe 2-3 pessoas responderem. Não corrija — só ouça.
FALA: "Guardem as respostas. Minha resposta em uma frase: é a diferença entre PRODUÇÃO ARTESANAL e uma
LINHA DE PRODUÇÃO. O artesão entrega uma peça boa de vez em quando. A fábrica entrega qualidade previsível,
repetível e auditável — lote após lote."
-->

---

# A tese central

Nós não "usamos uma IA".

Nós **montamos uma linha de produção de software dentro de arquivos de texto.**

> Norma · Estações · Dispositivos à prova de erro · Melhoria contínua

<!--
FALA: "O valor não está na máquina. Está na engenharia do processo de produção. A IA é obrigada, pela nossa
estrutura, a produzir como uma fábrica madura com QA — não como um artesão apressado."
-->

---

<!-- _class: lead -->

# ATO 1
## O Layout da Linha

---

# O chão de fábrica da nossa engenharia

- **CLAUDE.md** = Norma da Fábrica (lida em 100% das sessões)
- **hooks** = poka-yoke + Andon + sensores (dispositivos automáticos)
- **agents** = estações da linha (22 especialistas)
- **skills** = ordem de produção (`/comando`)
- **rules** = especificações de processo / SOPs (sob demanda)

<!--
[mostrar o layout ASCII da doc-mãe (README.md) seção 2]
FALA: "Olhem o chão de fábrica. No topo, a Norma — um arquivo lido sempre. As linhas de proteção são os
dispositivos automáticos. No fluxo, as estações. E as instruções de trabalho na prateleira, consultadas na
hora certa."
-->

---

# O parque em números

# 22 · 19 · 12 · 7 · **0**

estações · ordens · especificações · dispositivos · **commands**

*O zero é de propósito.*

<!--
FALA: "Repare no zero. A gente ESCOLHEU não usar 'commands' tradicionais — as skills já fazem esse papel,
melhor. Volto nisso no Ato 3. Menos mecanismos paralelos = menos confusão. Decisão de engenharia."
-->

---

# A célula central (o fluxo que sempre se repete)

`/investigar` → `/planejar` → `/implementar` → `/validar-entrega`

inspeção de entrada → PCP → fabricação → controle de qualidade

> Quem inspeciona não planeja · quem planeja não fabrica · quem fabrica não aprova a própria peça

<!--
[mostrar Diagrama 1 de ../base-conceitual/diagramas.md — a sequência com retorno]
FALA: "Esta célula resolve quase todo trabalho não-trivial. O segredo é a SEGREGAÇÃO DE FUNÇÕES: cada
passagem de bastão é uma checagem independente. Quem produz não é quem libera para expedição."
-->

---

# Os 5 valores que tudo protege

1. **Evidência > suposição** (mata a alucinação)
2. **Rastreabilidade de lote** (correlation_id, raio-X)
3. **Anti-falso-verde** (pronto = ativo no runtime)
4. **Reuso > fabricar** (almoxarifado antes da peça nova)
5. **Escopo mínimo, qualidade máxima** ("100%" qualifica a execução)

<!--
FALA: "Todos os arquivos existem para forçar estes cinco valores. É a política de qualidade da fábrica.
Guardem eles — vão reaparecer em cada estação."
-->

---

<!-- _class: lead -->

# ATO 2
## As Estações e as Especificações

---

# A Norma: CLAUDE.md

- Lida em **100% das sessões**, antes de tudo
- **Curta e referencial:** princípio sempre presente, SOP sob demanda
- 17 seções: cultura · pontos de controle · projeto da peça · disciplina

<!--
FALA: "A Norma converte cultura de engenharia em ESPECIFICAÇÃO APLICADA. Não é 'seria bom testar' — é
'a peça não passa sem ensaio'. E é enxuta de propósito: a instrução densa entra só quando o trabalho toca
o tema. Mesmo lazy-loading do bom software, aplicado à própria config."
-->

---

# O detalhe genial do "100%"

> "Implementação 100%" = executar o escopo pedido com **qualidade total**,
> **não** ampliar escopo.

O número qualifica a **execução**, não autoriza o **escopo**.

<!--
FALA: "Erro clássico do operador zeloso: ouvir '100%' e refazer a planta inteira. A Norma corta isso.
Faça o LOTE certo com perfeição — não invente encomenda nova. Isso é controle de custo puro."
-->

---

# As especificações de processo (rules/)

- **Profundas (SOPs):** log, reuso, definição-de-pronto, suíte, python, large-repo
- **Contratos de passagem de bastão:** estratégia, fidelidade-do-pedido
- **Diário de bordo (Kaizen):** lessons · error-backlog · regression · bad-instructions

<!--
FALA: "São as instruções de trabalho na estação. O ponto DRY elegante: os contratos de passagem de bastão
vivem em arquivo PRÓPRIO para não duplicar entre estações. A fábrica obedece às mesmas regras que impõe ao
produto."
-->

---

# A especificação mais valiosa: o raio-X

`log-instructions.md` — todo log é **registro de processo**, não enfeite.

- `correlation_id` (número de série) nasce uma vez, só propaga
- campos canônicos + builders oficiais
- `logger.exception` no `except` · anti-varredura de `/logs`

<!--
FALA: "É a caixa-preta da produção. Quando algo dá errado no cliente, a diferença entre 'resolvemos em
minutos' e 'dias no escuro' mora aqui. O log permite reconstruir o que aconteceu DEPOIS — debug offline."
-->

---

# Anti-falso-verde (o defeito nº 1)

**Falso-verde:** parece pronto (ensaio passou, doc atualizada),
mas a **linha oficial em runtime ainda não usa a peça nova**.

- Hierarquia de verdade: código > teste > log > roteiro > doc
- Check discriminante + teste de regressão estrutural

<!--
FALA: "É o golpe mais comum em automação: tudo verde, produto real não mudou. A definição-de-pronto exige
provar a peça INSTALADA no produto, não só fabricada na bancada."
-->

---

# O laudo que prova: a suíte

Verde no terminal é opinião que some. A **suíte oficial** emite **laudo auditável**.

- artefatos: `telemetry.json` · `state.json` · `logs/` · `reports/`
- `run_id`: novo · `--resume` (continua) · `--checkup` (relê)
- **foco** (iterar) × **fechamento amplo** (concluir)

> Laudo de ensaio + certificado de rastreabilidade do lote — não um "parece conforme" verbal

<!--
[aprofundamento: ../base-conceitual/suite-como-prova.md]
FALA: "Anti-falso-verde é a regra; a SUÍTE é a máquina que emite a prova. Não é um 'verde' que some no
terminal — é laudo que fica: telemetria, estado, logs por target. Ordem de leitura: resumo primeiro,
detalhe depois. 'pytest direto' é a lupa de bancada; a suíte é o certificado de ensaio que outro turno
reabre amanhã. Sem artefato auditável, não há validação confiável."
-->

---

# As estações: a célula central

inspeção de entrada → PCP → fabricação → controle de qualidade

> Contratos **opostos de propósito** = checagem independente a cada passo

<!--
[mostrar Diagrama 1 de ../base-conceitual/diagramas.md]
FALA: "Quatro estações, quatro propósitos opostos. A inspeção não pode planejar, o PCP não pode fabricar, o
QA não pode montar. Essa oposição é intencional — é segregação de funções, como nos controles internos de
uma empresa."
-->

---

# O laboratório de metrologia (observabilidade) ⏩

- `analisar-log` — lê os instrumentos (sem corrigir)
- `validar-log` — calibração do registro (escreveu certo?)
- `validar-observabilidade` — cobertura de sensores (há registro suficiente?)
- `corrigir-erros-com-log` — análise de causa raiz (RCA)
- `testar-cli-log-analyzer` — calibra o próprio instrumento

<!--
FALA (rápido): "Cinco estações que garantem que o sistema seja diagnosticável. Detalhe fino do analisar-log:
ele valida PARALELISMO REAL por sobreposição temporal — 'a UI disse 5 workers' não prova que 5 rodaram
juntos."
-->

---

# Manutenção e ensaios ⏩

**Manutenção do chão (infra real):** postgres · redis · job-core · scheduler
→ snapshot antes de mexer, dry-run, lockout-tagout

**Lote piloto (testes de fluxo real):** ingestão-dnit, webchat, nl2sql, nl2yaml…
→ três provas: UI + log + paralelismo real

<!--
FALA (rápido): "A manutenção é a mais paranoica: fotografa antes de mexer, mexe o mínimo, registra tudo. O
lote piloto prova na vida real — acaba com o 'passou na minha bancada'."
-->

---

<!-- _class: lead -->

# ATO 3
## A Ordem de Produção e os Dispositivos

---

# Skill: a ordem que aciona a estação

A skill **não trabalha** — ela aciona a estação certa e sai da frente.

- Acionamento **fino**: contexto limpo + custo menor + zero duplicação
- O *como* vive **só na estação** (fonte única de verdade)

<!--
FALA: "A skill é o botão; o agente é o cérebro. Mudou 'quando acionar'? Toca a skill. Mudou 'como executar'?
Toca o agente. Cada motivo de mudança tem UM lugar. SRP aplicado à config."
-->

---

# A engenharia do "Use when:"

O `description` não é decoração — é **contrato semântico**:

1. **Auto-disparo** (reconhece o pedido na sua fala)
2. **Desambiguação** (aponta para a ordem concorrente quando ela é melhor)
3. **Exclusão** (palavras negativas delimitam a fronteira)

<!--
FALA: "Numa fábrica de 22 postos, saber QUAL acionar é metade da qualidade. O 'Use when' até diz 'para isso
aqui, prefira a outra skill' — evita duas ordens disputando o mesmo pedido."
-->

---

# Por que NÃO usamos commands

| Mecanismo | Inteligência |
|---|---|
| Command | Nenhuma (prompt no mesmo contexto) |
| **Skill** | Acionamento + contrato |
| **Agent** | Toda a regra de execução |

Command seria um **3º mecanismo paralelo** para o mesmo papel.

<!--
FALA: "As skills já fazem o papel do command, melhor: auto-disparo e delegação a estação isolada. Adicionar
commands seria duplicar mecanismo — exatamente o que o CLAUDE.md proíbe."
-->

---

# O conceito-chave: poka-yoke vs. sensor

| Tipo | O que faz | Quando |
|---|---|---|
| **Poka-yoke / Andon (guard)** | **bloqueia** / para a linha | risco grave e claro |
| **Sensor / alarme (nudge)** | **avisa**, não para | exige julgamento |

<!--
FALA: "Poka-yoke é o dispositivo à prova de erro: o conector só entra de um jeito. Andon é o cordão que para
a linha. Sensor é a luz amarela: avisa, mas você decide. A engenharia é escolher o instrumento certo para
cada risco."
-->

---

# Os guards (poka-yoke / Andon)

- `bash-guard` — trava varredura cega de `/logs` e comandos destrutivos (`rm -rf`, `drop table`, `--force`)
- `write-guard` — impede ensaio nascer em pasta proibida

> Detalhe: o bash-guard **falha aberto** — na dúvida do próprio funcionamento, não trava a produção.

<!--
FALA: "Dois acidentes clássicos de automação: travar a sessão com uma listagem gigante e destruir dados com
um comando irreversível. O poka-yoke torna os dois IMPOSSÍVEIS — não depende de a IA 'lembrar'."
-->

---

# Os sensores (alarmes)

A cada `.py` salvo, acendem a luz amarela (não param):

- `logging-nudge` — `logger.error` em `except`? f-string em log? `print()` em src?
- `py-lint` — estilo (ruff) no escopo
- `py-discipline` — import opcional, FOR UPDATE, ensaio sem marker

<!--
FALA: "Transformamos a especificação densa de logs num RADAR automático que corrige no momento mais barato:
a hora da edição. Por que alarme e não trava? Porque pede julgamento — um print() pode ser legítimo num
script de sandbox."
-->

---

# O Kaizen que se fecha sozinho

```
INÍCIO DO TURNO            FIM DO TURNO
session-start              stop-loop-reminder
lê lessons.md              cobra registrar lição/defeito
(começa relembrado)        (termina tendo registrado)
```

<!--
FALA: "session-start LEMBRA das lições no começo; stop-loop COBRA o registro no fim. Juntos, a melhoria
contínua acontece automaticamente, sem depender de alguém lembrar de documentar. Kaizen ligado na tomada."
-->

---

<!-- _class: lead -->

# ATO 4
## Um Lote na Linha

---

# Vamos ver a fábrica produzindo

> *"A ingestão de PDFs da DNIT termina com erro intermitente. Conserta."*

Vamos seguir a peça, estação por estação.

<!--
FALA: "Pedido real, do tipo que chega todo dia. Vou marcar qual estação opera, qual especificação consulta
e qual dispositivo está de olho."
-->

---

# A produção, estação por estação

briefing → **metrologia** → **inspeção** → **PCP** → **fabricação** → **lote piloto** → **controle de qualidade** → registro

<!--
[mostrar Diagrama 1 e Diagrama 2 de ../base-conceitual/diagramas.md]
FALA: "A metrologia lê o log e acha a etapa do defeito. A inspeção vai ao código real e mede a verdade com
evidência. O PCP emite o roteiro e mapeia os ensaios. A fabricação conserta, e os sensores alertam se ela
esquecer um log. O lote piloto prova na ingestão real, com três provas. E o QA só libera a peça se estiver
ativa no runtime — senão, devolve ao PCP."
-->

---

# O lote pode voltar

REPROVADO → volta ao PCP (retrabalho, com inventário)

APROVADO → stop-loop cobra o Kaizen

> A linha **não é de mão única** — ela recupera o trabalho quando a peça não sai conforme.

<!--
FALA: "Essa capacidade de DEVOLVER COM INVENTÁRIO é o que impede o falso-verde de chegar ao cliente. O QA
não devolve dizendo 'tá ruim' — devolve dizendo exatamente o que faltou."
-->

---

# O placar: defeitos neutralizados

| Defeito clássico | Quem neutraliza |
|---|---|
| Alucinação | CLAUDE.md §1 + investigar |
| Falso-verde | definicao-de-pronto + validar-entrega |
| Defeito invisível no cliente | log-instructions + laboratório |
| Duplicação | reuso + inventario |
| Acidente operacional | bash-guard + manutenção (dry-run) |

<!--
FALA: "Cada problema clássico de engenharia tem um dono que o neutraliza. Não é sorte — é cobertura
intencional."
-->

---

<!-- _class: lead -->

# ATO 5
## Os Dois Motores e a Aposta

---

# Os dois motores invisíveis

- **Auto-correção = Jidoka + Andon** → conserta ESTE lote (para a linha, conserta a causa, com teto)
- **Auto-aperfeiçoamento = Kaizen** → melhora a fábrica para os PRÓXIMOS lotes

<!--
[mostrar Diagrama 1 e Diagrama 2 de ../base-conceitual/diagramas.md]
FALA: "Jidoka é 'automação com toque humano': a máquina para sozinha ao detectar anomalia, em vez de
produzir refugo. Kaizen é a melhoria entre turnos. Um garante que o lote de HOJE sai conforme; o outro que o
lote de AMANHÃ começa melhor. E há uma ponte: a lição de um conserto vira prevenção de uma família de
defeitos."
-->

---

# Por que funciona num código gigante ⏩

A IA **não tenta abraçar tudo** (seria impossível e caro):

- começa numa **âncora** concreta
- lê só o necessário, **prova por fatia**
- proibida varredura cega · estações isoladas (vão e voltam)

<!--
FALA (rápido): "São as travas de escala. É o que separa 'funciona no projetinho de exemplo' de 'funciona no
sistema real'. Detalhe na base-conceitual/travas-projetos-grandes."
-->

---

# A virada de papel: de operário a gerente de produção

| Produção artesanal | Nossa linha (orquestração) |
|---|---|
| humano digita o código | humano dá a ordem (`/investigar`…) |
| humano revisa o diff | humano lê **evidência** (laudos, roteiros, vereditos, logs) |
| aprova linha a linha | aprova na **altitude** da decisão |

<!--
FALA: "O humano sobe do nível da linha de código para o nível da evidência e da decisão. Ele não confere se
a variável foi nomeada certo — disso cuidam os SOPs e os dispositivos. Ele confere se a inspeção tem
evidência, se o roteiro é seguro, se a validação provou."
-->

---

# A ordem de serviço bem especificada

Quem opera a fábrica não aperta botões — **emite uma OS clara**:

**Objetivo** · **Escopo** · **Restrições** · **Evidência** · **Critério de aceite**

> Sem **evidência** (parte 4), não se abre OS — pede-se **inspeção** primeiro

<!--
[aprofundamento: ../base-conceitual/operar-a-metodologia.md]
FALA: "Se o humano é o gerente de produção, a OS é a ferramenta dele. Ordem vaga dá lote vago; ordem com
essas cinco partes vira trabalho auditável. Regra de ouro: sem evidência, não se abre ordem de fabricação
— manda-se inspecionar a matéria-prima primeiro. Abrir OS sem evidência é convidar o refugo."
INTERAÇÃO: "Qual dessas cinco partes mais falta nas ordens reais? (quase sempre o critério de aceite)"
-->

---

# A afirmação ousada: a inspeção final (code review) fica redundante

Não acabou — foi **desmontada em dezenas de verificações contínuas**:

- design review → no **PCP** (antes do código)
- estilo → nos **sensores** (no ato da escrita)
- reuso/padrão → na **fabricação** (durante)
- veredito cético → no **controle de qualidade** (independente)

<!--
FALA: "O que era um gargalo humano no fim virou uma malha de garantias do começo ao fim. Não é menos
revisão — é mais revisão, contínua e sem fadiga. Detalhe completo na base-conceitual/sem-digitar-codigo."
-->

---

# A leitura honesta (e onde mora a garantia)

- O humano **não some** — sobe de altitude (aprova evidência, não sintaxe)
- A garantia é tão boa quanto os **contratos** (por isso validar-instructions existe)
- Decisões de produto realmente novas → ainda pedem julgamento humano

> A garantia mora no **conteúdo dos contratos**, não no agente.

<!--
FALA: "Sejamos honestos: 'sem digitar código' não é 'sem engenharia'. O esforço migra da digitação para a
curadoria dos contratos e a leitura crítica da evidência. É mais alavancado, não inexistente."
-->

---

<!-- _class: lead -->

# ATO 6
## Fechamento

---

# A tese, agora completa

Qualquer um "pede código para uma IA".

Nós montamos **uma linha de produção** onde a IA é obrigada a operar como
**fábrica madura com controle de qualidade** — e que **melhora sozinha a cada turno**.

<!--
FALA: "O valor não está em nenhum arquivo isolado. Está no PROCESSO — em como eles operam juntos. Produção
artesanal entrega uma peça boa de vez em quando. Uma fábrica enxuta entrega qualidade previsível, repetível
e auditável, lote após lote."
-->

---

# O que levar pra casa

- **Segregação de funções** (quem fabrica ≠ quem libera)
- **Rastreabilidade de lote** (correlation_id, raio-X)
- **Anti-falso-verde** (pronto = no runtime)
- **Poka-yoke vs. alarme** (travar o inegociável, avisar o resto)
- **Jidoka + Kaizen** (corrige hoje, melhora amanhã)

<!--
FALA: "Se levarem cinco ideias, levem essas. Elas valem para qualquer sistema de software, com ou sem IA."
-->

---

<!-- _class: lead -->

# Obrigado.
## Perguntas?

<!--
INTERAÇÃO: reabra a pergunta do início: "Agora, o que vocês acham que é a diferença entre usar IA e fazer
engenharia com IA?" — compare com as respostas iniciais.
FALA: "Tudo está documentado: capítulos, base conceitual, FAQ de objeções e diagramas. Obrigado."
RESERVA: use ../base-conceitual/faq-objecoes.md para responder objeções difíceis.
-->

---

<!--
=========================================================================
INSTRUÇÕES DE EXPORTAÇÃO (não é slide — é referência para você)
=========================================================================

OPÇÃO A — VS Code (mais fácil):
  1. Instale a extensão "Marp for VS Code".
  2. Abra este arquivo. Clique no ícone de preview (canto superior direito).
  3. Para exportar: paleta de comandos > "Marp: Export Slide Deck..." > PDF / PPTX / HTML.
  4. Modo apresentador (com as notas): exporte para HTML e abra; ou use o preview.

OPÇÃO B — Linha de comando (Marp CLI):
  npx @marp-team/marp-cli@latest apresentacao-marp.md -o apresentacao.html
  npx @marp-team/marp-cli@latest apresentacao-marp.md --pdf  --allow-local-files
  npx @marp-team/marp-cli@latest apresentacao-marp.md --pptx --allow-local-files
  (todos os artefatos de saída devem ir para ./.sandbox/tmp/ se forem temporários)

NOTAS DO APRESENTADOR:
  - O texto dentro destes comentários HTML aparece como "presenter notes" no modo apresentador
    do Marp (HTML) e é ignorado nos slides.

DIAGRAMAS:
  - Marp não renderiza Mermaid nativamente. Onde a nota diz "[mostrar Diagrama N]", abra
    ../base-conceitual/diagramas.md (renderiza no GitHub/VS Code) numa aba ao lado, ou exporte como imagem.
=========================================================================
-->
