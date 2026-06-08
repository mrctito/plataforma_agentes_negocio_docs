# Roteiro de Aula — Slides + Falas

> **Como usar este roteiro.** Cada slide tem três blocos:
> - 🖥️ **TELA** — o que projetar (título + bullets curtos; não leia os bullets, eles são âncora visual).
> - 🎤 **FALA** — o que você diz (em linguagem falada, não decore; adapte ao seu jeito).
> - 🙋 **INTERAÇÃO** — quando houver, uma pergunta para a plateia (faça uma pausa real e espere resposta).
>
> **Duração estimada:** 55–70 min. **Divisão em 6 atos.** Os slides marcados com ⏩ são puláveis se o
> tempo apertar. A fonte de cada slide é a apostila em [README.md](README.md), capítulos e apêndices.
>
> **O Ato 5 (A Aposta Estratégica)** carrega o conteúdo dos apêndices — é a parte que mais impacta a
> liderança. Se a plateia for só técnica, você pode encurtá-lo; se houver liderança, é o clímax.
>
> **Material de apoio sugerido:** deixe a [doc-mãe](README.md) aberta como backup para perguntas.

---

## ATO 0 — Abertura (≈5 min)

### Slide 1 — Capa
🖥️ **TELA**
> **A Tática por Trás dos Arquivos**
> Como transformamos a IA em um time de engenharia sênior
> *[seu nome / data]*

🎤 **FALA**
> "Hoje eu não vou ensinar 'o que é o Claude' nem 'o que é um agente' — isso está no site da
> Anthropic e qualquer um lê em dez minutos. Eu vou mostrar uma coisa que **só existe aqui dentro**:
> a engenharia que a gente colocou por trás de cada arquivo de configuração. E vou usar uma metáfora
> que todo mundo entende: **futebol**."

---

### Slide 2 — A pergunta que abre tudo
🖥️ **TELA**
> Qual a diferença entre **"usar IA pra programar"** e **"fazer engenharia com IA"?**

🙋 **INTERAÇÃO**
> Jogue a pergunta e deixe 2–3 pessoas responderem. Não corrija ninguém — só ouça.

🎤 **FALA**
> "Guardem as respostas de vocês. No fim da aula eu quero ver se a gente concorda. Minha resposta,
> em uma frase, é: a diferença é **tática**. Um chute genial ganha um lance. Um time com tática ganha
> o campeonato — de forma previsível, repetível e auditável."

---

### Slide 3 — A tese central
🖥️ **TELA**
> Nós não "usamos uma IA".
> Nós **montamos um time de engenharia inteiro dentro de arquivos de texto.**
> Filosofia · Posições · Jogadas ensaiadas · Árbitro automático

🎤 **FALA**
> "O valor não está na ferramenta. Está na **engenharia da configuração**. A IA é obrigada, pela
> nossa estrutura, a trabalhar como um engenheiro sênior disciplinado — e não como um estagiário
> apressado que faz e reza pra dar certo."

---

## ATO 1 — A Escalação (≈8 min)

### Slide 4 — A formação 4-3-3
🖥️ **TELA**
> *(projete o diagrama da formação da [doc-mãe, seção 2](README.md))*
> CLAUDE.md = DNA · hooks = árbitro · agents = jogadores · skills = convocação · rules = jogadas

🎤 **FALA**
> "Olhem o campo. Cada grupo de arquivos ocupa uma posição. No topo, a **filosofia do clube** — um
> arquivo lido em 100% das sessões. As linhas do campo, sempre ligadas, são o **árbitro automático**.
> Em campo, os **jogadores**. E o caderno de **jogadas ensaiadas** no banco, consultado na hora certa."

---

### Slide 5 — O elenco em números
🖥️ **TELA**
> **22** agentes (jogadores) · **19** skills (convocações) · **12** regras (jogadas) · **7** hooks (arbitragem)
> **0** commands — *e isso é de propósito*

🎤 **FALA**
> "Repare no zero. A gente **escolheu** não usar 'commands' tradicionais, porque as skills já fazem
> esse papel, melhor. Volto nisso no Ato 3. Por ora, segurem essa ideia: **menos mecanismos
> paralelos = menos confusão.** Isso é uma decisão de engenharia, não acaso."

---

### Slide 6 — O eixo central (a jogada que sempre se repete)
🖥️ **TELA**
> `/investigar` → `/planejar` → `/implementar` → `/validar-entrega`
> zaga → meia → ataque → goleiro

🎤 **FALA**
> "Se vocês esquecerem tudo da aula e lembrarem de UMA coisa, lembrem disto. Quase todo trabalho não
> trivial passa por esses quatro. E tem uma regra de ouro: **quem investiga não planeja, quem planeja
> não executa, quem executa não se auto-aprova.** Cada troca de pé é uma barreira contra o erro."

🙋 **INTERAÇÃO**
> "Isso lembra alguma coisa de controle interno de empresa?" *(Espere — alguém vai dizer "segregação
> de funções". Se ninguém disser, você diz.)*

---

### Slide 7 — Os 5 valores que tudo protege
🖥️ **TELA**
> 1. Evidência > suposição
> 2. Rastreabilidade ponta a ponta
> 3. Anti-falso-verde
> 4. Reuso > criar
> 5. Escopo mínimo, qualidade máxima

🎤 **FALA**
> "Todos os arquivos, todas as posições, existem pra forçar esses cinco valores. Cada um mata um erro
> clássico de engenharia. Vou mostrar como, posição por posição."

---

## ATO 2 — As Posições (≈18 min)

### Slide 8 — O DNA: CLAUDE.md
🖥️ **TELA**
> **CLAUDE.md — a Filosofia do Clube**
> Lido em 100% das sessões · curto e referencial · vira **lei**, não sugestão

🎤 **FALA**
> "Esse arquivo não é um jogador — é o DNA que todo jogador carrega. Ele pega coisas que normalmente
> são 'boa intenção' — 'seria bom testar', 'tenta não duplicar' — e transforma em **lei inegociável
> aplicada em toda sessão**. A qualidade deixa de depender do humor do dia."

---

### Slide 9 — O detalhe genial do "100%"
🖥️ **TELA**
> "Implementação 100%" = **execução com qualidade total**, NÃO escopo ampliado
> O número qualifica a *execução*, não autoriza inflar o *recorte*

🎤 **FALA**
> "Esse é o meu favorito. Quando você manda alguém fazer '100%', o zeloso quer refatorar o sistema
> inteiro 'já que está aqui'. Isso é over-engineering, é risco, é estouro de prazo. A nossa regra
> corta isso: faça o **recorte certo** com excelência. Não invente trabalho."

🙋 **INTERAÇÃO**
> "Quem aqui já viu uma tarefa de 2 horas virar um projeto de 2 semanas 'por iniciativa'?"

---

### Slide 10 — As jogadas ensaiadas (rules/)
🖥️ **TELA**
> `.claude/rules/` — conhecimento profundo **carregado sob demanda**
> Contratos: logs · reuso · definição de pronto · suíte · python · large-repo
> Caderno de aprendizado: lessons · error-backlog · regression · bad-instructions

🎤 **FALA**
> "O DNA é curto de propósito. O detalhe pesado fica aqui e é consultado **na hora da jogada** — como
> um manual de posição. É o nosso conhecimento institucional **escrito**, não na cabeça de poucos."

---

### Slide 11 — O contrato mais valioso: o raio-X
🖥️ **TELA**
> **log-instructions.md — a caixa-preta do avião**
> correlation_id nasce uma vez · logs são "raio-X" reconstruível · debug offline

🎤 **FALA**
> "Todo processo ganha um número de rastreio que atravessa tudo. O log não é enfeite — é **evidência
> que permite reconstruir o que aconteceu depois**, como um debug offline. A diferença entre
> 'resolvemos em minutos' e 'passamos três dias no escuro' mora aqui."

🙋 **INTERAÇÃO**
> "Quem já perdeu uma tarde porque o log não dizia nada?"

---

### Slide 12 — Anti-falso-verde
🖥️ **TELA**
> **Falso-verde:** parece pronto (teste passou, doc atualizada)…
> …mas o sistema real **não usa o código novo**
> → A entrega só vale **ativa no caminho oficial em runtime**

🎤 **FALA**
> "Esse é o inimigo número 1 de qualquer automação. Tudo verde, e o sistema não mudou. A gente tem um
> contrato inteiro — a 'definição de pronto' — que obriga a **provar** que o código novo está ligado
> de verdade. É o VAR anulando o gol por impedimento."

---

### Slide 12-bis — A máquina que produz a prova: a suíte
🖥️ **TELA**
> Verde no terminal é opinião que some. A **suíte oficial** deixa **rastro auditável**.
> artefatos: `telemetry.json` · `state.json` · `logs/` · `reports/`
> `run_id`: novo · `--resume` (continua) · `--checkup` (relê) — foco (iterar) × fechamento amplo (concluir)

🎤 **FALA**
> "Anti-falso-verde é a regra; a **suíte** é a máquina que produz a prova dela. Ela não te dá só um
> 'verde' que some — deixa rastro: telemetria, estado, logs por target. E ensina a ordem de leitura:
> resumo primeiro, detalhe depois. O `pytest` direto é a lupa; a suíte é a **trilha auditável** que
> outra pessoa reabre amanhã. Sem artefato auditável, não existe validação confiável."

🔎 **APROFUNDAMENTO**
> [../base-conceitual/suite-como-prova.md](../base-conceitual/suite-como-prova.md)

---

### Slide 13 — Os jogadores: o eixo central
🖥️ **TELA**
> 🛡️ investigar (zagueiro) — levanta a verdade, com evidência
> 🎯 planejar (meia) — transforma evidência em plano seguro
> ⚽ implementar (atacante) — código ativo no runtime
> 🧤 validar-entrega (goleiro) — aprova/reprova com evidência

🎤 **FALA**
> "Quatro especialistas de função única. O zagueiro **não chuta** — o que ele não leu, ele declara
> como desconhecido. O meia **discorda com franqueza** se a ideia for ruim. O atacante prova o gol no
> runtime. E o goleiro pode **anular o gol do próprio time** se for falso-verde — e devolve a bola pro
> meia com a lista do que faltou."

---

### Slide 14 — A defesa de observabilidade ⏩
🖥️ **TELA**
> analisar-log (olheiro) · validar-log (escreveu certo?) · validar-observabilidade (cobriu tudo?)
> corrigir-erros-com-log (cirurgião) · testar-cli-log-analyzer (VAR da ferramenta)

🎤 **FALA**
> "Um setor inteiro só pra garantir que o sistema seja **diagnosticável**. Repare na divisão
> cirúrgica: um valida se o log foi **escrito certo**, outro valida se tem log **suficiente**. Dois
> defeitos diferentes, dois especialistas. E tem até um VAR que valida a própria ferramenta de análise."

---

### Slide 15 — O departamento de futebol (infra)
🖥️ **TELA**
> postgresql · redis · job-core/ingestão · scheduler
> DNA comum: **fotografa antes de mexer · dry-run · mutação auditável · nunca toca dado de terceiro**

🎤 **FALA**
> "Esses quatro tocam o mundo real — dados, filas, agendamentos. Por isso são os mais paranoicos:
> tiram uma foto antes, mexem o mínimo, e registram tudo. É a disciplina de quem mexe em produção com
> checklist e backup, nunca no improviso."

---

### Slide 16 — Treinos e comissão técnica ⏩
🖥️ **TELA**
> **Treinos:** executar-testes · criar-testes · testar-ingestao/cancelamento/webchat/nl2sql/...
> **Bastidores:** documentar · sincronizar-doc · inventário · validar-instructions · analisar-produto

🎤 **FALA**
> "Os treinos provam o time na **vida real** — a ingestão DNIT, por exemplo, exige **três provas
> independentes** pra dar sucesso. E os bastidores cuidam pra que o conhecimento não apodreça: doc
> fiel ao código, catálogo do que já existe, e até um agente que **audita as próprias instruções**."

---

## ATO 3 — A Convocação e a Arbitragem (≈10 min)

### Slide 17 — Skill: o apito que chama o jogador
🖥️ **TELA**
> A **skill** não joga. Ela **convoca** o agente e sai da frente.
> Atalho fino → contexto limpo · custo menor · zero duplicação

🎤 **FALA**
> "Quando você digita `/investigar`, a skill não faz o trabalho — ela chama o jogador isolado, que
> trabalha numa 'sala separada' e volta só com o resultado. Isso mantém a conversa limpa, gasta menos
> e — importante — o 'como trabalhar' fica **só no agente**. Sem duplicação pra dessincronizar."

---

### Slide 18 — A engenharia do "Use when"
🖥️ **TELA**
> `description: "Use when: investigar… ; para análise pura de log, prefira analisar-log"`
> Gatilho de auto-disparo + **desambiguação entre skills parecidas**

🎤 **FALA**
> "Esse cabeçalho parece burocracia, mas é um **contrato semântico**. Ele ensina o sistema a chamar a
> skill certa — e até aponta pra concorrente quando ela é a melhor escolha. Num time de 22
> especialistas, saber **quem chamar** já é metade da qualidade."

---

### Slide 19 — Por que NÃO usamos commands
🖥️ **TELA**
> Skill já é o gatilho `/` — com auto-disparo e delegação a agente
> Adicionar command = **terceiro mecanismo paralelo** pra mesma função → proibido (DRY)

🎤 **FALA**
> "Aqui volta o 'zero commands' do começo. A gente não usa porque seria duplicar mecanismo. A mesma
> regra que aplicamos ao código — não ter dois jeitos de fazer a mesma coisa — a gente aplica na
> própria configuração da IA. A config obedece às regras que ela impõe."

---

### Slide 20 — O conceito-chave: nudge vs. guard
🖥️ **TELA**
> **Guard (trava):** o árbitro apita falta — a jogada NÃO acontece *(bloqueia)*
> **Nudge (cutucada):** o bandeirinha levanta a bandeira — o jogo segue *(avisa)*

🎤 **FALA**
> "Essa é a distinção mais importante da camada de arbitragem. 'Nudge' é 'empurrãozinho' em inglês —
> um lembrete no momento certo. A engenharia está em **escolher o instrumento certo pro risco**:
> **trava** pro que é inegociável; **cutucada** pro que precisa de bom senso."

🙋 **INTERAÇÃO**
> "Qual a diferença entre uma regra que a gente *pede* e uma que o sistema *garante*?"

---

### Slide 21 — Os guards (as travas)
🖥️ **TELA**
> bash-guard: bloqueia `rm -rf`, `drop table`, e **varredura cega de /logs** (trava a sessão!)
> write-guard: impede teste nascer em pasta que a suíte não roda

🎤 **FALA**
> "Coisas que **não podem** acontecer viram trava automática. Um `rm -rf`, um `drop table`, ou uma
> listagem numa pasta com 50 mil arquivos que congela tudo. O agente nem chega a tentar — o árbitro
> apita antes. E ele 'falha aberto': se a própria checagem der pau, ele deixa passar, pra nunca
> atrapalhar indevidamente."

---

### Slide 22 — Os nudges (as cutucadas)
🖥️ **TELA**
> Ao salvar um `.py`: radar automático cutuca…
> `logger.error` no except → use `logger.exception` · f-string no log · print() em src/ · teste sem marker

🎤 **FALA**
> "Toda vez que um arquivo Python é salvo, um radar lê o código e levanta a bandeira pros sinais de
> dívida técnica — no **momento mais barato de consertar**, que é a hora da edição, não num review
> três semanas depois. Mas é cutucada, não trava: porque essas coisas pedem julgamento de contexto."

---

### Slide 23 — O ciclo de aprendizado que se fecha sozinho
🖥️ **TELA**
> `session-start` → injeta as **lições passadas** no começo (preleção)
> `stop-loop` → **cobra** registrar novas lições no fim (vestiário)

🎤 **FALA**
> "E o detalhe mais bonito: no começo da sessão, o sistema **lembra** das lições antigas; no fim, ele
> **cobra** que você registre as novas. O aprendizado da empresa acontece **automaticamente** — o
> time sai de campo mais inteligente do que entrou, por construção, não por disciplina individual."

---

## ATO 4 — A Partida (≈8 min)

### Slide 24 — Vamos ver o time jogando
🖥️ **TELA**
> Cenário real: *"A ingestão de PDFs da DNIT termina com erro intermitente. Conserta."*

🎤 **FALA**
> "Chega de teoria. Vamos seguir a bola num pedido de verdade, do tipo que chega todo dia. Reparem em
> cada troca de pé e em qual árbitro está de olho."

---

### Slide 25 — O jogo, lance a lance
🖥️ **TELA**
> *(projete o mapa da jogada da [Cap. 6, seção 3](capitulo-06-a-partida.md))*
> preleção → olheiro → zagueiro → meia → atacante → jogo real → goleiro → cobrança

🎤 **FALA**
> *(Narre seguindo o Cap. 6.2 — resumido:)*
> "O olheiro lê o log e acha a etapa do erro. O zagueiro vai ao **código real** e levanta a verdade
> com evidência. O meia arma o plano e já mapeia os testes. O atacante conserta, e os bandeirinhas o
> cutucam se ele esquecer um log. Os treinos provam na **ingestão real**, com três provas. E o goleiro
> só valida o gol se estiver ativo no runtime — senão, **devolve a bola pro meia**."

---

### Slide 26 — A bola pode voltar
🖥️ **TELA**
> REPROVADO → volta pro `planejar` com **inventário do que faltou**
> Não é esteira de mão única — é um time que **recupera a posse**

🎤 **FALA**
> "Esse é o ponto que separa nossa abordagem de 'pedir código pra IA'. Quando o lance não sai limpo,
> a bola **volta com a lista do que faltou**. É exatamente isso que impede o falso-verde de chegar na
> produção."

---

### Slide 27 — O placar: problemas neutralizados
🖥️ **TELA**
> alucinação · falso-verde · bug invisível · duplicação · over-engineering
> regressão silenciosa · acidente operacional · conhecimento que apodrece
> *cada um tem um jogador que o neutraliza*

🎤 **FALA**
> "Esse é o placar defensivo. Cada um desses erros clássicos de engenharia tem um responsável no time
> por neutralizar. Não é sorte — é posição."

---

## ATO 5 — A Aposta Estratégica (≈10 min)

> *Este ato condensa os 4 apêndices. É o que diferencia "uma IA que ajuda a programar" de "um processo
> de engenharia que dispensa o gargalo humano". Vá mais devagar aqui — é onde a liderança decide se
> comprou a ideia.*

### Slide 28 — Os dois motores invisíveis
🖥️ **TELA**
> **Auto-correção** — conserta ESTA tarefa até ficar certa *(dentro do jogo, com teto)*
> **Auto-aperfeiçoamento** — deixa O TIME melhor a cada rodada *(memória durável)*
> Um garante o trabalho de hoje · o outro, o de amanhã

🎤 **FALA**
> "Tem dois motores rodando o tempo todo, e gente confunde os dois. Um é a **auto-correção**: o sistema
> insiste numa tarefa — executa, falha, conserta, revalida — até acertar **ou declarar que travou**, com
> limite de tentativas pra não rodar pra sempre. O outro é o **auto-aperfeiçoamento**: no começo de cada
> sessão o sistema lembra das lições passadas, no fim ele cobra registrar as novas. O time sai de campo
> mais inteligente do que entrou. Um conserta o jogo; o outro muda o treino da semana."

🙋 **INTERAÇÃO**
> "Qual desses dois a maioria das empresas **não** tem? *(o segundo — o conhecimento some quando a
> pessoa sai)*"

---

### Slide 29 — Por que funciona num código gigante ⏩
🖥️ **TELA**
> A IA tem memória limitada · nosso código é enorme
> Travas: âncora + fatia · anti-varredura de logs · agentes isolados · carregamento sob demanda
> → "começar num ponto concreto, ler só o necessário, provar por fatia"

🎤 **FALA**
> "Uma objeção honesta: 'isso funciona num exemplinho, mas no nosso sistema real?'. Funciona **porque**
> a gente tem travas de escala. A IA não abraça o código todo — ela é obrigada a começar numa âncora
> concreta, ler só o necessário e provar por fatia. É disciplina de escala. Sem isso, estoura custo e
> ela inventa conclusão."

---

### Slide 30 — A virada de papel: de digitador a maestro
🖥️ **TELA**
> Modelo tradicional: humano **digita** código · humano **revisa** o diff
> Nosso modelo: humano **comanda** e **decide** · agente **executa**
> O humano sobe da **linha de código** para a **evidência**

🎤 **FALA**
> "Agora a parte que muda o jogo. Repare que em nenhum momento desta aula um humano digitou código. O
> humano **conduz**. Ele não confere se a variável tem o nome certo — disso cuidam os contratos e os
> robôs-árbitros. Ele lê **evidência**: o relatório da investigação, o plano, o veredito da validação,
> o log. Ele sobe de altitude: da sintaxe pra decisão."

---

### Slide 30-bis — A preleção do treinador: o bom pedido em 5 partes
🖥️ **TELA**
> Você não entra em campo — **arma a jogada da beira**:
> **Objetivo** · **Escopo** · **Restrições** · **Evidência** · **Critério de aceite**
> Sem **evidência** (parte 4), não peça o gol — peça **investigação** primeiro

🎤 **FALA**
> "Se o humano é o maestro, esta é a batuta. Pedido vago dá resultado vago; pedido com essas cinco
> partes vira trabalho auditável. E a regra de ouro: sem **evidência**, não peça implementação — peça
> investigação. Pedir conserto sem evidência é convidar o chute, justamente o que a metodologia existe
> pra impedir."

🙋 **INTERAÇÃO**
> "Qual dessas cinco partes vocês mais esquecem? *(quase sempre é o critério de aceite)*"

🔎 **APROFUNDAMENTO**
> [../base-conceitual/operar-a-metodologia.md](../base-conceitual/operar-a-metodologia.md)

---

### Slide 31 — A afirmação ousada: o code review fica redundante
🖥️ **TELA**
> O code review não foi **removido** — foi **desmontado e embutido**:
> Design → `planejar` (antes do código) · Estilo → hooks (no ato) · Reuso → `implementar` (durante)
> Veredito cético independente → `validar-entrega` · Critérios → `CLAUDE.md` (lei em toda sessão)

🎤 **FALA**
> "E aqui vem a afirmação que costuma gerar debate: o code review separado, aquele humano lendo o diff
> no fim, fica **redundante**. Não porque a gente abriu mão de qualidade — pelo contrário. A gente pegou
> **cada função** do code review e embutiu no processo: a crítica de arquitetura acontece no
> **planejamento, antes do código existir**; o estilo é pego **no instante da escrita** pelos hooks; o
> reuso é checado **durante** a implementação; e no fim um **revisor independente codificado** — o
> validar-entrega — dá o veredito, e ele **não aceita 'teste verde' como prova**. É shift-left ao
> extremo: cada checagem no momento mais barato, com rigor que não cansa no 40º diff do dia."

🙋 **INTERAÇÃO**
> "Quem aqui já aprovou um PR no automático porque confiava no autor e tava com pressa? *(pausa)* O
> contrato nunca faz isso."

---

### Slide 32 — A leitura honesta (e onde mora a garantia)
🖥️ **TELA**
> O humano **não some** — ele revisa **evidência**, não sintaxe
> A garantia é tão boa quanto **os contratos** (por isso o conteúdo é o ativo, não o agente)
> "Sem digitar código" ≠ "sem engenharia"

🎤 **FALA**
> "Pra não vender ilusão: o humano não desaparece. Quem aprova um plano ruim recebe uma execução fiel a
> um plano ruim. E a garantia toda depende da **qualidade dos contratos** — contrato fraco, revisão
> fraca. Por isso o ativo estratégico da empresa **não é o agente**: é o corpo de contratos. Se a gente
> trocar de modelo de IA amanhã, os contratos continuam valendo e continuam impondo o rigor. **O
> conteúdo é o patrimônio.**"

---

## ATO 6 — Fechamento (≈5 min)

### Slide 33 — A tese, agora completa
🖥️ **TELA**
> Qualquer um pede código pra uma IA.
> Nós construímos um **sistema onde a IA é obrigada a jogar como um time sênior.**
> O valor não está em nenhum arquivo. Está na **tática.**

🎤 **FALA**
> "Voltando à pergunta do começo. A diferença entre 'usar IA' e 'fazer engenharia com IA' é a tática.
> Um jogador genial ganha um lance. **Um time com tática ganha o campeonato** — previsível, repetível,
> auditável. Foi isso que a gente transformou em texto."

---

### Slide 34 — O que levar pra casa
🖥️ **TELA**
> 1. Especialistas de função única > IA faz-tudo
> 2. Quem faz ≠ quem aprova (segregação de funções)
> 3. Pronto = ativo no runtime (anti-falso-verde)
> 4. Trava pro inegociável, cutucada pro bom senso
> 5. O time aprende sozinho a cada rodada
> 6. A garantia mora no **conteúdo dos contratos**, não no agente

🎤 **FALA**
> "Cinco ideias. Se vocês forem montar algo parecido em qualquer time — com ou sem IA — comecem por
> essas."

---

### Slide 35 — Encerramento + perguntas
🖥️ **TELA**
> **Perguntas?**
> Material completo: `docs/metodologia-engenharia-claude/`

🙋 **INTERAÇÃO**
> Reabra a pergunta do Slide 2: *"Agora, o que vocês acham que é a diferença entre usar IA e fazer
> engenharia com IA?"* — compare com as respostas iniciais.

🎤 **FALA**
> "Tudo o que mostrei está documentado em sete capítulos, nessa pasta. Quem quiser se aprofundar numa
> posição específica, é só abrir o capítulo dela. Obrigado."

---

## Apêndice — Cola do apresentador (one-liners de bolso)

Para responder rápido se alguém perguntar "e o que faz X?":

- **CLAUDE.md** → "O DNA. Lido sempre. Vira lei, não sugestão."
- **rules/** → "Os manuais de posição. Conhecimento pesado, consultado na hora da jogada."
- **agents/** → "Os jogadores. 22 especialistas de função única, contexto isolado."
- **skills/** → "O apito que chama o jogador certo. Atalho fino que delega ao agente."
- **hooks/** → "O árbitro automático. Trava (guard) ou cutuca (nudge), sem a IA decidir."
- **correlation_id** → "O número de rastreio. Nasce uma vez, atravessa tudo, permite o raio-X."
- **falso-verde** → "Parece pronto, não está. O sistema real ainda não usa o código novo."
- **nudge** → "Cutucada — avisa, mas não bloqueia. (É 'nudge', não 'nodle'.)"
- **guard** → "Trava — bloqueia de fato. Pro risco inegociável."

---

**Voltar:** [README — A metodologia completa](README.md)
</content>
