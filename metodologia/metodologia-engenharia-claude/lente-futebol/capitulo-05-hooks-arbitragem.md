# Capítulo 5 — A Arbitragem (`.claude/hooks/`)

> **Posição em campo:** os hooks são o **árbitro, os bandeirinhas, as linhas do campo e o VAR
> automático**. Diferente de tudo que vimos até aqui, **eles não são a IA decidindo** — são regras do
> sistema que disparam **sozinhas**, em momentos fixos, de forma **determinística**. O DNA, as
> jogadas e os jogadores dependem de o agente "lembrar" e "querer" cumprir. O hook não pede licença:
> ele apita a falta ou levanta a bandeira, independentemente do que o jogador pretendia fazer.

> 🧑‍💼 **RESUMO EXECUTIVO.** Esta é a camada que transforma "boa intenção" em "garantia". Toda regra
> importante o suficiente para não poder falhar nós tiramos da confiança e colocamos na automação. É
> a diferença entre "pedimos para todos lavarem as mãos" e "a porta só abre depois que lavou". O
> valor: as regras críticas passam a ser **impossíveis de esquecer**, não importa a pressa do dia.

---

## 5.1 O conceito que você pediu para explicar: *nudge* (e não "nodle")

Você perguntou sobre "nudge ou nodle". O termo certo é **nudge** — em inglês, "empurrãozinho",
"cutucada". É um conceito de design de comportamento: em vez de **proibir**, você dá um lembrete no
momento certo e deixa a pessoa (ou a IA) decidir.

Na nossa arbitragem, isso cria **dois tipos de intervenção**, e a distinção é o coração deste capítulo:

| Tipo | Analogia no futebol | O que faz | Quando usamos |
|---|---|---|---|
| **Guard (trava)** | O árbitro apita falta — a jogada **não acontece** | **Bloqueia** a ação (`deny`) | Risco grave e claro (destruição, travar a sessão) |
| **Nudge (cutucada)** | O bandeirinha levanta a bandeira — o jogo **segue**, mas com aviso | **Avisa**, injeta um lembrete no contexto, **não bloqueia** | Disciplina e qualidade onde o agente precisa de julgamento |

**Por que ter os dois?** Porque nem tudo merece bloqueio. Bloquear demais engessa o trabalho e gera
falsos positivos irritantes. Bloquear de menos deixa passar o que importa. A engenharia está em
**escolher o instrumento certo para cada risco**: trava para o que é inegociável, cutucada para o que
exige bom senso. Na nossa escalação, só **3 dos 7 hooks bloqueiam** — o resto cutuca.

---

## 5.2 Quando cada hook dispara (os momentos do jogo)

A súmula (`settings.json`) amarra os hooks a **dois momentos**, e só a eles:

```
  PreToolUse (Bash)        ──► antes de um comando de terminal ──► bash-guard + worktree-guard  (árbitro: pode apitar)
  PreToolUse (Write)       ──► antes de criar um arquivo       ──► write-guard                  (árbitro: pode apitar)
  PostToolUse (Edit|Write) ──► depois de editar/criar          ──► py-lint + logging-nudge + py-discipline  (bandeirinhas)
```

**E a honestidade importa aqui:** não existe hook de início nem de fim de sessão (nenhum evento
`SessionStart`/`Stop` configurado). A "preleção antes do jogo" não é um hook — é o **carregamento
automático do DNA**: o `CLAUDE.md` raiz entra em 100% das sessões e as jogadas ensaiadas entram por
`paths:` quando a bola chega no setor ([Cap. 1](capitulo-01-filosofia-claude-md.md) e
[Cap. 2](capitulo-02-rules-playbook.md)). Há ainda um sétimo hook escrito e pronto no vestiário que
**não foi escalado** — falamos dele na [seção 5.5](#55-o-banco-de-reservas--um-hook-pronto-que-não-foi-escalado).

---

## 5.3 Os guards (as travas) — `PreToolUse`

São os que **bloqueiam de fato**. Disparam *antes* da ação acontecer e podem dizer "não".

### `bash-guard.sh` — o árbitro do comando de terminal

Inspeciona todo comando shell antes de rodar. Tem três níveis:

1. **Trava a varredura cega de `/logs`.** Se o comando usa `ls/find/grep/cat/tail…` na pasta de logs
   **sem** um `correlation_id`, ele **bloqueia** — porque a pasta pode ter dezenas de milhares de
   arquivos e **travar a sessão inteira**. Esta é a aplicação automática de uma regra do
   `log-instructions.md`. *(O DNA pede; o árbitro garante.)*
2. **Trava comandos destrutivos.** `rm -rf`, `drop table`, `truncate table`, `git push --force` — tudo
   bloqueado, com a orientação de confirmar com o humano e rodar manualmente fora do agente.
3. **Cutuca (não bloqueia) desvios menores:** gravar em `/tmp` em vez de `./.sandbox/tmp/`, ou `psql`
   sem desabilitar o pager.

**Detalhe de engenharia importante:** o hook **falha aberto** — se algo der errado na própria checagem,
ele deixa o comando passar. Um árbitro que, na dúvida do próprio funcionamento, não trava o jogo
indevidamente. Segurança sem virar obstáculo.

**Problema que evita:** dois acidentes clássicos de automação — **travar a sessão** com uma listagem
gigante e **destruir dados** com um comando irreversível.

### `worktree-guard.sh` — o cartão vermelho direto, sem VAR

O guard mais duro do time, e o melhor material de metáfora da arbitragem. Contexto: o usuário trabalha
com **várias janelas/agentes/worktrees em paralelo** ([Cap. 2](capitulo-02-rules-playbook.md),
`worktree.md`), e os worktrees têm nome fixo (`worktree-codex`/`worktree-claude`) — o nome **não
carrega** sessão nem UUID, então **é impossível provar pela máquina de quem é o worktree**.

A resposta de engenharia: se não dá para provar posse, **nega sempre**. O hook intercepta
`git worktree remove` e `git branch -d/-D` sobre esses nomes e bloqueia — mesmo que o agente "tenha
certeza" de que o worktree é dele. A única válvula de escape é um **token nominal** que exige
confirmação expressa do usuário para **aquele alvo exato**:

```
CONFIRMO_EXCLUSAO_WT_ALHEIA=<nome-exato-do-worktree> git worktree remove ...
```

O próprio hook avisa: "não existe bypass genérico ou em lote". É cartão vermelho direto, sem
consulta ao VAR — porque apagar o worktree de **outra sessão** destrói trabalho alheio de forma
irreversível.

**O contraste que ensina:** o `bash-guard` **falha aberto** (na dúvida sobre si mesmo, deixa passar);
o `worktree-guard` **falha fechado** (na dúvida sobre a posse, nega). A escolha não é gosto — é
**assimetria de risco**: bloquear um comando legítimo custa um pedido de confirmação; deixar passar
uma exclusão errada custa o trabalho de outra sessão. Falha aberta para não atrapalhar; falha fechada
para não destruir.

### `write-guard.sh` — o árbitro do arquivo

Impede a criação de testes pytest em diretórios proibidos (`tests/manual|disabled|temp`). É a aplicação
automática do `CLAUDE.md §10` / `tests/CLAUDE.md`: teste tem que nascer no lugar coberto pela suíte
oficial, com o marker de família correto.

**Problema que evita:** teste "fantasma" criado num canto que a suíte oficial não roda — cobertura que
parece existir mas não protege nada.

---

## 5.4 Os nudges (as cutucadas) — `PostToolUse`

Disparam **depois** de cada `Edit`/`Write` de arquivo Python e **injetam lembretes** sem travar nada.
São bandeirinhas: levantam a bandeira, o agente decide.

### `py-lint.sh` — o bandeirinha do estilo

Roda `ruff check` no arquivo recém-editado e, se houver questões, injeta a saída no contexto — com a
ressalva sábia: *"corrija se pertinente ao seu escopo, sem ampliar o recorte"*. Ou seja, cutuca, mas
respeita a regra de **escopo mínimo** (não vira desculpa para refatorar o arquivo inteiro).

### `logging-nudge.sh` — o bandeirinha da observabilidade

O mais sofisticado. Lê o `.py` editado e procura **sinais concretos** de violação do contrato de logs,
cutucando quando encontra:
- `logger.error(...)` dentro de `except` → deveria ser `logger.exception(...)` (preserva o stack trace);
- f-string em chamada de log → deveria ser lazy `%` formatting;
- `import logging` / `logging.getLogger` → logger padrão do Python em vez do oficial do projeto;
- `print()` em código de `src/` → deveria ser o logger oficial;
- `extra={...event_name...}` manual → deveria usar o builder canônico;
- usa logging mas não referencia nenhum helper canônico.

Repare: ele **detecta por leitura estática** o que viraria um defeito de observabilidade *antes* de o
código rodar. É o contrato `log-instructions.md` transformado em radar automático.

### `py-discipline.sh` — o bandeirinha da disciplina Python

Cutuca três coisas:
- `except ImportError`/`ModuleNotFoundError` → import opcional/lazy é proibido (se o pacote falta, deve
  falhar);
- `FOR UPDATE` → lock pessimista, peça justificativa e prefira controle otimista;
- arquivo de teste **sem marker de família** no topo.

**Problema que os nudges evitam:** a erosão silenciosa da qualidade. Sozinhas, nenhuma dessas coisas
derruba o sistema — mas, acumuladas, viram dívida técnica e cegueira de observabilidade. O nudge
corrige no **momento exato da edição**, quando é barato consertar, em vez de num code review semanas
depois.

> 🛠️ **Por que nudge e não guard aqui?** Porque essas regras exigem **julgamento de contexto**. Um
> `print()` pode ser legítimo num script de `.sandbox`; um `FOR UPDATE` pode ser realmente necessário.
> Travar seria arrogante e geraria falso positivo. A cutucada respeita a inteligência do agente: "olha
> isso aqui, decida". Guard para o inegociável; nudge para o que pede bom senso.

---

## 5.5 O banco de reservas — um hook pronto que não foi escalado

Honestidade de inventário: existe um sétimo script no vestiário, o **`stop-loop-reminder.sh`**, escrito
e funcional — mas **não registrado em nenhum `settings.json`**. Ele não dispara em nenhuma sessão.
É uma **lacuna conhecida e declarada**, não um recurso.

O que ele faria, se escalado: no fim de uma rodada com edição real, se houve mudança em `src/**.py`
**sem** mudança correspondente em `tests/`, injetaria o lembrete de cobertura de testes e
observabilidade. Só isso — um nudge fino de fim de jogo, não um "cobrador de lições".

Duas leituras de engenharia deste fato:

1. **A metáfora continua válida:** um clube pode ter jogador pronto no banco e optar por não escalá-lo.
   O que não pode é a súmula dizer que ele jogou. Documentar o hook como ativo seria exatamente a
   "doc que mente" que o time combate.
2. **O aprendizado do time não depende dele.** O ciclo real de aprendizado não é um hook de fim de
   sessão: são os **dois loops estratégicos** + a **promoção curada** para os `licoes-aprendidas.md`
   dos agentes ([Cap. 2.2](capitulo-02-rules-playbook.md)) — contrato obrigatório, não lembrete
   opcional.

---

## 5.6 `settings.json` e `settings.local.json` — a súmula oficial

- **`settings.json`** é a **súmula**: amarra cada hook ao seu momento (`PreToolUse` para os guards,
  `PostToolUse` para os nudges). É o que faz a arbitragem existir — e é por ele que se prova que o
  `stop-loop-reminder` não está em campo.
- **`settings.local.json`** é a lista de **permissões locais**: pré-autoriza comandos seguros e
  repetitivos para não pedir confirmação a cada vez — hoje: a suíte oficial
  (`python suite_de_testes_padrao.py`), `jest`, `ruff check`, a CLI de logs
  (`python -m src.log_analyzer`) e o script canônico de consulta ao PostgreSQL
  (`run_postgresql_query.py`). É produtividade sem abrir mão de segurança — só entra aqui o que é
  comprovadamente seguro.

---

## 5.7 A síntese da arbitragem

| Hook | Momento | Tipo | Garante |
|---|---|---|---|
| bash-guard | antes de Bash | **guard** (falha aberta) + nudge | Não trava a sessão, não destrói dados |
| worktree-guard | antes de Bash | **guard** (falha fechada) | Não apaga worktree de outra sessão |
| write-guard | antes de Write | **guard** | Teste nasce no lugar certo |
| py-lint | após editar .py | nudge | Estilo limpo no momento da edição |
| logging-nudge | após editar .py | nudge | Observabilidade canônica por radar |
| py-discipline | após editar .py | nudge | Disciplina Python/testes |
| stop-loop-reminder | *(não wired)* | nudge **no banco** | Nada, hoje — lacuna declarada (§5.5) |

---

## 5.8 O que levar desta posição para a aula

- Hooks **não são a IA decidindo** — são regras **determinísticas** que disparam sozinhas. É o pilar
  que converte "intenção" em "garantia".
- A distinção mestra: **guard (trava/`deny`)** para o risco inegociável; **nudge (cutucada/aviso)**
  para o que exige julgamento. E, dentro dos guards, a segunda distinção: **falha aberta** para não
  atrapalhar (bash-guard) vs. **falha fechada** para não destruir (worktree-guard) — escolhida pela
  assimetria de risco, com válvula de escape por **token nominal**, nunca genérico.
- Os nudges transformam contratos densos (como o de logs) em **radares automáticos** que corrigem no
  momento mais barato — a hora da edição.
- Honestidade de súmula: **3 travas + 3 bandeirinhas em campo; 1 hook pronto no banco, sem wiring** —
  e o aprendizado do time vive nos loops estratégicos (Cap. 2), não num hook de fim de sessão.

**Próximo:** [Capítulo 6 — A Partida (o time inteiro em ação)](capitulo-06-a-partida.md).
