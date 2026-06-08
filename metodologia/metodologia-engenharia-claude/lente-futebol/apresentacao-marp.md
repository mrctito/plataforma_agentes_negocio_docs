---
marp: true
theme: default
paginate: true
size: 16:9
header: 'A Tática por Trás dos Arquivos'
footer: 'Metodologia de Engenharia de Agentes — Prometeu'
style: |
  section { font-size: 26px; }
  section.lead { text-align: center; }
  h1 { color: #1a4d7a; }
  h2 { color: #1a4d7a; }
  strong { color: #b5651d; }
  section.lead h1 { font-size: 52px; }
  code { background: #eef3f8; }
---

<!--
COMO USAR ESTE ARQUIVO (Marp)
- Conteúdo visível = o que vai na TELA.
- Texto dentro de comentário HTML (como este) = NOTAS DO APRESENTADOR (só você vê no modo apresentador).
- Exportar: ver instruções no fim do arquivo ou em README.md.
- Onde houver "[mostrar Diagrama N de ../base-conceitual/diagramas.md]", abra o diagrama Mermaid correspondente.
-->

# A Tática por Trás dos Arquivos

## Como transformamos a IA em um time de engenharia sênior

*Metodologia de Engenharia de Agentes — Prometeu*

<!--
FALA: "Hoje eu NÃO vou ensinar 'o que é o Claude' nem 'o que é um agente' — isso está no site da Anthropic.
Vou mostrar uma coisa que só existe aqui dentro: a engenharia por trás de cada arquivo de configuração.
E vou usar uma metáfora que todo mundo entende: futebol."
-->

---

<!-- _class: lead -->

# Qual a diferença entre

## "usar IA pra programar"
## e "fazer engenharia com IA"?

<!--
INTERAÇÃO: jogue a pergunta, deixe 2-3 pessoas responderem. Não corrija — só ouça.
FALA: "Guardem as respostas. No fim eu quero ver se a gente concorda. Minha resposta em uma frase:
a diferença é TÁTICA. Um chute genial ganha um lance. Um time com tática ganha o campeonato —
previsível, repetível, auditável."
-->

---

# A tese central

Nós não "usamos uma IA".

Nós **montamos um time de engenharia inteiro dentro de arquivos de texto.**

> Filosofia · Posições · Jogadas ensaiadas · Árbitro automático

<!--
FALA: "O valor não está na ferramenta. Está na engenharia da configuração. A IA é obrigada, pela nossa
estrutura, a trabalhar como um engenheiro sênior disciplinado — não como um estagiário apressado."
-->

---

<!-- _class: lead -->

# ATO 1
## A Escalação

---

# A formação 4-3-3 da nossa engenharia

- **CLAUDE.md** = DNA (lido em 100% das sessões)
- **hooks** = árbitro automático (linhas do campo)
- **agents** = jogadores (22 especialistas)
- **skills** = convocação (`/comando`)
- **rules** = jogadas ensaiadas (sob demanda)

<!--
[mostrar Diagrama 4 de ../base-conceitual/diagramas.md, ou o diagrama ASCII da doc-mãe seção 2]
FALA: "Olhem o campo. Cada grupo de arquivos ocupa uma posição. No topo, a filosofia do clube — um
arquivo lido sempre. As linhas do campo são o árbitro automático. Em campo, os jogadores. E o caderno
de jogadas no banco, consultado na hora certa."
-->

---

# O elenco em números

# 22 · 19 · 12 · 7 · **0**

agentes · skills · regras · hooks · **commands**

*O zero é de propósito.*

<!--
FALA: "Repare no zero. A gente ESCOLHEU não usar 'commands' tradicionais — as skills já fazem esse papel,
melhor. Volto nisso no Ato 3. Por ora: menos mecanismos paralelos = menos confusão. Decisão de
engenharia, não acaso."
-->

---

# O eixo central (a jogada que sempre se repete)

`/investigar` → `/planejar` → `/implementar` → `/validar-entrega`

zaga → meia → ataque → goleiro

> Quem investiga não planeja · quem planeja não executa · quem executa não se auto-aprova

<!--
[mostrar Diagrama 1 de ../base-conceitual/diagramas.md — a sequência com retorno]
FALA: "Se esquecerem tudo e lembrarem de UMA coisa, lembrem disto. Cada troca de pé é uma barreira
contra o erro."
INTERAÇÃO: "Isso lembra alguma coisa de controle interno de empresa?" (alguém dirá "segregação de funções")
-->

---

# Os 5 valores que tudo protege

1. **Evidência** > suposição
2. **Rastreabilidade** ponta a ponta
3. **Anti-falso-verde**
4. **Reuso** > criar
5. **Escopo mínimo**, qualidade máxima

<!--
FALA: "Todos os arquivos, todas as posições, existem pra forçar esses cinco valores. Cada um mata um
erro clássico de engenharia. Vou mostrar como, posição por posição."
-->

---

<!-- _class: lead -->

# ATO 2
## As Posições

---

# O DNA: CLAUDE.md

A **Filosofia do Clube**

- Lido em **100% das sessões**
- Curto e referencial
- Vira **lei**, não sugestão

<!--
FALA: "Esse arquivo não é um jogador — é o DNA que todo jogador carrega. Ele pega 'boa intenção' —
'seria bom testar', 'tenta não duplicar' — e transforma em lei inegociável em toda sessão. A qualidade
deixa de depender do humor do dia."
-->

---

# O detalhe genial do "100%"

"Implementação 100%" = **execução com qualidade total**

**NÃO** = escopo ampliado

> O número qualifica a *execução*, não autoriza inflar o *recorte*

<!--
FALA: "Meu favorito. Quando você manda fazer '100%', o zeloso quer refatorar o sistema inteiro 'já que
estou aqui'. Isso é over-engineering, risco, estouro de prazo. A regra corta: faça o recorte certo com
excelência. Não invente trabalho."
INTERAÇÃO: "Quem já viu uma tarefa de 2 horas virar um projeto de 2 semanas 'por iniciativa'?"
-->

---

# As jogadas ensaiadas (`rules/`)

Conhecimento profundo **carregado sob demanda**

- **Contratos:** logs · reuso · definição de pronto · suíte · python
- **Caderno de aprendizado:** lessons · error-backlog · regression · bad-instructions

<!--
FALA: "O DNA é curto de propósito. O detalhe pesado fica aqui, consultado na hora da jogada — como um
manual de posição. É o nosso conhecimento institucional ESCRITO, não na cabeça de poucos."
-->

---

# O contrato mais valioso: o raio-X

**log-instructions.md** — a caixa-preta do avião

- `correlation_id` nasce uma vez, atravessa tudo
- O log é **raio-X reconstruível** → debug offline

<!--
FALA: "Todo processo ganha um número de rastreio que atravessa tudo. O log não é enfeite — é evidência
que permite reconstruir o que aconteceu depois. A diferença entre 'resolvemos em minutos' e 'três dias
no escuro' mora aqui."
INTERAÇÃO: "Quem já perdeu uma tarde porque o log não dizia nada?"
-->

---

# Anti-falso-verde

**Falso-verde:** parece pronto (teste passou, doc atualizada)…

…mas o sistema real **não usa o código novo**

> A entrega só vale **ativa no caminho oficial em runtime**

<!--
FALA: "O inimigo número 1 de qualquer automação. Tudo verde, e o sistema não mudou. A gente tem um
contrato inteiro — a 'definição de pronto' — que obriga a PROVAR que o código novo está ligado de
verdade. É o VAR anulando o gol por impedimento."
-->

---

# A máquina que produz a prova: a suíte

Verde no terminal é opinião que some. A **suíte oficial** deixa **rastro auditável**.

- artefatos: `telemetry.json` · `state.json` · `logs/` · `reports/`
- `run_id`: novo · `--resume` (continua) · `--checkup` (relê)
- **foco** (iterar) × **fechamento amplo** (concluir)

> A análise de desempenho com dados: prova o que aconteceu em campo, não "achei que jogamos bem"

<!--
[aprofundamento: ../base-conceitual/suite-como-prova.md]
FALA: "Anti-falso-verde é a regra; a SUÍTE é a máquina que produz a prova dela. Ela não te dá só um
'verde' que some no terminal — ela deixa rastro: telemetria, estado, logs por target. E ensina a ler na
ordem certa: resumo primeiro, detalhe depois. 'pytest direto' é a lupa; a suíte é a trilha auditável que
outra pessoa reabre amanhã. Sem artefato auditável, não existe validação confiável."
-->

---

# Os jogadores: o eixo central

- 🛡️ **investigar** (zagueiro) — levanta a verdade, com evidência
- 🎯 **planejar** (meia) — evidência → plano seguro
- ⚽ **implementar** (atacante) — código ativo no runtime
- 🧤 **validar-entrega** (goleiro) — aprova/reprova com evidência

<!--
FALA: "Quatro especialistas de função única. O zagueiro NÃO chuta — o que não leu, declara desconhecido.
O meia discorda com franqueza se a ideia for ruim. O atacante prova o gol no runtime. E o goleiro pode
anular o gol do próprio time se for falso-verde — e devolve a bola pro meia com a lista do que faltou."
-->

---

# A defesa de observabilidade

- **analisar-log** (olheiro)
- **validar-log** — escreveu certo?
- **validar-observabilidade** — cobriu tudo?
- **corrigir-erros-com-log** (cirurgião)
- **testar-cli-log-analyzer** (VAR da ferramenta)

<!--
FALA: "Um setor inteiro só pra garantir que o sistema seja diagnosticável. Divisão cirúrgica: um valida
se o log foi ESCRITO certo, outro valida se tem log SUFICIENTE. Dois defeitos, dois especialistas."
-->

---

# O departamento de futebol (infra)

**postgresql · redis · job-core/ingestão · scheduler**

> DNA comum: fotografa antes de mexer · dry-run · mutação auditável · nunca toca dado de terceiro

<!--
FALA: "Esses quatro tocam o mundo real — dados, filas, agendamentos. Por isso são os mais paranoicos:
foto antes, mexem o mínimo, registram tudo. Disciplina de quem mexe em produção com checklist e backup."
-->

---

# Treinos e comissão técnica

- **Treinos:** executar-testes · criar-testes · testar-ingestao/webchat/nl2sql…
- **Bastidores:** documentar · sincronizar-doc · inventário · validar-instructions · analisar-produto

<!--
FALA: "Os treinos provam o time na vida real — a ingestão DNIT exige TRÊS provas independentes pra dar
sucesso. E os bastidores cuidam pra que o conhecimento não apodreça: doc fiel ao código, catálogo do que
já existe, e um agente que audita as próprias instruções."
-->

---

<!-- _class: lead -->

# ATO 3
## A Convocação e a Arbitragem

---

# Skill: o apito que chama o jogador

A **skill** não joga. Ela **convoca** o agente e sai da frente.

> Atalho fino → contexto limpo · custo menor · zero duplicação

<!--
FALA: "Quando você digita /investigar, a skill não faz o trabalho — chama o jogador isolado, que trabalha
numa 'sala separada' e volta só com o resultado. Mantém a conversa limpa, gasta menos e o 'como
trabalhar' fica SÓ no agente. Sem duplicação pra dessincronizar."
-->

---

# A engenharia do "Use when"

`description: "Use when: investigar… ; para análise pura de log, prefira analisar-log"`

- Gatilho de **auto-disparo**
- **Desambiguação** entre skills parecidas

<!--
FALA: "Parece burocracia, mas é um contrato semântico. Ensina o sistema a chamar a skill certa — e até
aponta pra concorrente quando ela é a melhor escolha. Num time de 22 especialistas, saber quem chamar já
é metade da qualidade."
-->

---

# Por que NÃO usamos commands

A skill já é o gatilho `/` — com auto-disparo e delegação a agente

> Adicionar command = **terceiro mecanismo paralelo** pra mesma função → proibido (DRY)

<!--
FALA: "Aqui volta o 'zero commands'. A gente não usa porque seria duplicar mecanismo. A mesma regra que
aplicamos ao código — não ter dois jeitos de fazer a mesma coisa — aplicamos na própria configuração da
IA. A config obedece às regras que ela impõe."
-->

---

# O conceito-chave: nudge vs. guard

- **Guard (trava):** o árbitro apita falta — a jogada NÃO acontece *(bloqueia)*
- **Nudge (cutucada):** o bandeirinha levanta a bandeira — o jogo segue *(avisa)*

<!--
FALA: "A distinção mais importante da arbitragem. 'Nudge' é 'empurrãozinho' — um lembrete na hora certa.
A engenharia está em escolher o instrumento certo pro risco: trava pro inegociável, cutucada pro que
precisa de bom senso."
INTERAÇÃO: "Qual a diferença entre uma regra que a gente PEDE e uma que o sistema GARANTE?"
-->

---

# Os guards (as travas)

- **bash-guard:** bloqueia `rm -rf`, `drop table`, e **varredura cega de /logs** (trava a sessão!)
- **write-guard:** impede teste nascer em pasta que a suíte não roda

<!--
FALA: "Coisas que NÃO podem acontecer viram trava automática. Um rm -rf, um drop table, ou uma listagem
numa pasta com 50 mil arquivos que congela tudo. O agente nem chega a tentar. E ele 'falha aberto': se a
própria checagem der pau, deixa passar, pra nunca atrapalhar indevidamente."
-->

---

# Os nudges (as cutucadas)

Ao salvar um `.py`, um radar cutuca:

- `logger.error` no except → use `logger.exception`
- f-string no log · `print()` em src/ · teste sem marker

<!--
FALA: "Toda vez que um .py é salvo, um radar lê o código e levanta a bandeira pros sinais de dívida
técnica — no momento mais barato de consertar, a hora da edição, não num review três semanas depois. Mas
é cutucada, não trava: essas coisas pedem julgamento de contexto."
-->

---

# O ciclo de aprendizado que se fecha sozinho

- `session-start` → injeta as **lições passadas** no começo (preleção)
- `stop-loop` → **cobra** registrar novas lições no fim (vestiário)

<!--
FALA: "O detalhe mais bonito: no começo da sessão o sistema LEMBRA das lições antigas; no fim, COBRA que
você registre as novas. O aprendizado da empresa acontece automaticamente — o time sai de campo mais
inteligente do que entrou, por construção."
-->

---

<!-- _class: lead -->

# ATO 4
## A Partida

---

# Vamos ver o time jogando

Cenário real:

> *"A ingestão de PDFs da DNIT termina com erro intermitente. Conserta."*

<!--
FALA: "Chega de teoria. Vamos seguir a bola num pedido de verdade, do tipo que chega todo dia. Reparem em
cada troca de pé e em qual árbitro está de olho."
-->

---

# O jogo, lance a lance

preleção → **olheiro** → **zagueiro** → **meia** → **atacante** → **jogo real** → **goleiro** → cobrança

<!--
[mostrar Diagrama 1 e Diagrama 2 de ../base-conceitual/diagramas.md]
FALA: "O olheiro lê o log e acha a etapa do erro. O zagueiro vai ao código real e levanta a verdade com
evidência. O meia arma o plano e já mapeia os testes. O atacante conserta, e os bandeirinhas o cutucam se
ele esquecer um log. Os treinos provam na ingestão real, com três provas. E o goleiro só valida o gol se
estiver ativo no runtime — senão, devolve a bola pro meia."
-->

---

# A bola pode voltar

REPROVADO → volta pro `planejar` com **inventário do que faltou**

> Não é esteira de mão única — é um time que **recupera a posse**

<!--
FALA: "O ponto que separa nossa abordagem de 'pedir código pra IA'. Quando o lance não sai limpo, a bola
volta com a lista do que faltou. É isso que impede o falso-verde de chegar na produção."
-->

---

# O placar: problemas neutralizados

alucinação · falso-verde · bug invisível · duplicação · over-engineering · regressão silenciosa · acidente operacional · conhecimento que apodrece

> *cada um tem um jogador que o neutraliza*

<!--
FALA: "O placar defensivo. Cada um desses erros clássicos tem um responsável no time por neutralizar. Não
é sorte — é posição."
-->

---

<!-- _class: lead -->

# ATO 5
## A Aposta Estratégica

*(a parte que mais impacta a liderança)*

---

# Os dois motores invisíveis

- **Auto-correção** — conserta ESTA tarefa até ficar certa *(com teto)*
- **Auto-aperfeiçoamento** — deixa O TIME melhor a cada rodada *(memória durável)*

> Um garante o trabalho de hoje · o outro, o de amanhã

<!--
FALA: "Dois motores rodando o tempo todo, e gente confunde os dois. Auto-correção: executa, falha,
conserta, revalida — até acertar OU declarar que travou, com limite de tentativas. Auto-aperfeiçoamento:
no começo lembra das lições, no fim cobra registrar as novas. Um conserta o jogo; o outro muda o treino
da semana."
INTERAÇÃO: "Qual desses a maioria das empresas NÃO tem?" (o segundo — o conhecimento some quando a pessoa sai)
-->

---

# Por que funciona num código gigante

A IA tem memória limitada · nosso código é enorme

> Travas: âncora + fatia · anti-varredura de logs · agentes isolados · carregamento sob demanda

"Começar num ponto concreto, ler só o necessário, provar por fatia"

<!--
FALA: "Objeção honesta: 'funciona num exemplinho, mas no nosso sistema real?'. Funciona PORQUE temos
travas de escala. A IA não abraça o código todo — começa numa âncora concreta, lê só o necessário, prova
por fatia. Sem isso, estoura custo e ela inventa conclusão."
-->

---

# A virada de papel: de digitador a maestro

- Tradicional: humano **digita** código · humano **revisa** o diff
- Nosso modelo: humano **comanda** e **decide** · agente **executa**

> O humano sobe da **linha de código** para a **evidência**

<!--
FALA: "A parte que muda o jogo. Em nenhum momento desta aula um humano digitou código. O humano CONDUZ.
Não confere se a variável tem o nome certo — disso cuidam contratos e robôs-árbitros. Ele lê evidência:
relatório, plano, veredito, log. Sobe de altitude: da sintaxe pra decisão."
-->

---

# A preleção do treinador: o bom pedido em 5 partes

Você não entra em campo — **arma a jogada da beira**:

**Objetivo** · **Escopo** · **Restrições** · **Evidência** · **Critério de aceite**

> Sem **evidência** (parte 4), não peça o gol — peça **investigação** primeiro

<!--
[aprofundamento: ../base-conceitual/operar-a-metodologia.md]
FALA: "Se o humano é o maestro, ESTA é a batuta. Um pedido vago dá resultado vago; um pedido com essas
cinco partes vira trabalho auditável. E a regra de ouro: se você não tem a EVIDÊNCIA, não peça
implementação — peça investigação. Pedir conserto sem evidência é convidar o chute, justo o que a
metodologia existe pra impedir."
INTERAÇÃO: "Qual dessas cinco partes vocês mais esquecem? (quase sempre é o critério de aceite)"
-->

---

# A afirmação ousada: o code review fica redundante

Não foi **removido** — foi **desmontado e embutido**:

- Design → `planejar` (antes do código)
- Estilo → hooks (no ato)
- Reuso → `implementar` (durante)
- Veredito cético independente → `validar-entrega`
- Critérios → `CLAUDE.md` (lei em toda sessão)

<!--
FALA: "A afirmação que gera debate: o code review separado, o humano lendo o diff no fim, fica redundante.
Não porque abrimos mão de qualidade — pelo contrário. Pegamos CADA função do code review e embutimos: a
crítica de arquitetura no planejamento, ANTES do código existir; o estilo no instante da escrita; o reuso
DURANTE; e no fim um revisor independente codificado que NÃO aceita 'teste verde' como prova. Shift-left
ao extremo, com rigor que não cansa no 40º diff do dia."
INTERAÇÃO: "Quem já aprovou um PR no automático porque confiava no autor e tava com pressa? O contrato nunca faz isso."
-->

---

# A leitura honesta (e onde mora a garantia)

- O humano **não some** — revisa **evidência**, não sintaxe
- A garantia é tão boa quanto **os contratos**
- "Sem digitar código" ≠ "sem engenharia"

> O ativo é o **conteúdo dos contratos**, não o agente

<!--
FALA: "Pra não vender ilusão: o humano não desaparece. Quem aprova um plano ruim recebe execução fiel a um
plano ruim. E a garantia depende da qualidade dos contratos — contrato fraco, revisão fraca. Por isso o
ativo estratégico NÃO é o agente: é o corpo de contratos. Troque o modelo de IA amanhã, os contratos
continuam valendo. O conteúdo é o patrimônio."
-->

---

<!-- _class: lead -->

# ATO 6
## Fechamento

---

# A tese, agora completa

Qualquer um pede código pra uma IA.

Nós construímos um **sistema onde a IA é obrigada a jogar como um time sênior.**

> O valor não está em nenhum arquivo. Está na **tática.**

<!--
FALA: "Voltando à pergunta do começo. A diferença entre 'usar IA' e 'fazer engenharia com IA' é a tática.
Um jogador genial ganha um lance. Um time com tática ganha o campeonato — previsível, repetível,
auditável. Foi isso que transformamos em texto."
-->

---

# O que levar pra casa

1. Especialistas de função única > IA faz-tudo
2. Quem faz ≠ quem aprova (segregação de funções)
3. Pronto = ativo no runtime (anti-falso-verde)
4. Trava pro inegociável, cutucada pro bom senso
5. O time aprende sozinho a cada rodada
6. A garantia mora no **conteúdo dos contratos**

<!--
FALA: "Seis ideias. Se forem montar algo parecido em qualquer time — com ou sem IA — comecem por essas."
-->

---

<!-- _class: lead -->

# Perguntas?

Material completo: `docs/metodologia-engenharia-claude/`

<!--
INTERAÇÃO: reabra a pergunta do início: "Agora, o que vocês acham que é a diferença entre usar IA e fazer
engenharia com IA?" — compare com as respostas iniciais.
FALA: "Tudo está documentado nessa pasta — capítulos, apêndices, FAQ de objeções e diagramas. Obrigado."
RESERVA: use a faq-objecoes.md para responder objeções difíceis.
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
    ../base-conceitual/diagramas.md (renderiza no GitHub/VS Code) numa aba ao lado, ou exporte como imagem
    e cole no slide correspondente.
=========================================================================
-->
</content>
