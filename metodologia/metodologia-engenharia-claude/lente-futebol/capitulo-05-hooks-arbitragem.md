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
exige bom senso.

---

## 5.2 Quando cada hook dispara (os momentos do jogo)

Os hooks estão amarrados no `settings.json` a quatro momentos do "jogo":

```
  SessionStart   ──►  antes do jogo começar      ──►  session-start.sh        (preleção)
  PreToolUse     ──►  antes de uma jogada (Bash/Write) ─► bash-guard / write-guard  (árbitro: pode apitar falta)
  PostToolUse    ──►  depois de editar um arquivo ──►  py-lint / logging-nudge / py-discipline  (bandeirinhas)
  Stop           ──►  no apito final              ──►  stop-loop-reminder      (cobrança pós-jogo)
```

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

### `write-guard.sh` — o árbitro do arquivo

Impede a criação de testes pytest em diretórios proibidos (`tests/manual|disabled|temp`). É a aplicação
automática do `CLAUDE.md §11` / `tests/CLAUDE.md`: teste tem que nascer no lugar coberto pela suíte
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

## 5.5 Os hooks de início e fim — fechando o loop de aprendizado

### `session-start.sh` — a preleção antes do jogo

No começo de **toda sessão**, injeta automaticamente no contexto o conteúdo de `lessons.md` (as lições
globais) e um resumo dos registros operacionais (quantos erros no backlog, quantas regressões). É o
**lado "ler"** do loop de aprendizado: o time entra em campo já lembrado das lições passadas.

### `stop-loop-reminder.sh` — a cobrança no vestiário

No fim da rodada (`Stop`), se houve edição real, lembra o **lado "escrever"** do loop: registrar erro
real em `error-backlog.md`, reincidência em `regression-logs.md`, promover lição transversal para
`lessons.md`. E faz uma checagem fina: **se editou `src/**.py` mas não tocou em `tests/`**, levanta a
bandeira sobre cobertura de testes e observabilidade.

> 🧑‍💼 **RESUMO EXECUTIVO — o ciclo que se fecha sozinho.** O `session-start` *lembra* das lições no
> começo; o `stop-loop` *cobra* o registro de novas lições no fim. Juntos, eles fazem o aprendizado da
> empresa acontecer **automaticamente**, sem depender de alguém lembrar de documentar. É a máquina de
> melhoria contínua ligada à tomada — o time fica mais inteligente a cada rodada, por construção.

---

## 5.6 `settings.json` e `settings.local.json` — a súmula oficial

- **`settings.json`** é a **súmula**: amarra cada hook ao seu momento (`SessionStart`, `PreToolUse`,
  `PostToolUse`, `Stop`). É o que faz a arbitragem existir.
- **`settings.local.json`** é a lista de **permissões locais**: pré-autoriza comandos seguros e
  repetitivos (rodar a suíte, `ruff`, `mypy`, `pytest`, `npm run test:*`) para não pedir confirmação a
  cada vez. É produtividade sem abrir mão de segurança — só entra aqui o que é comprovadamente seguro.

---

## 5.7 A síntese da arbitragem

| Hook | Momento | Tipo | Garante |
|---|---|---|---|
| session-start | início | injeção | Lições passadas presentes desde o 1º minuto |
| bash-guard | antes de Bash | **guard** + nudge | Não trava a sessão, não destrói dados |
| write-guard | antes de Write | **guard** | Teste nasce no lugar certo |
| py-lint | após editar .py | nudge | Estilo limpo no momento da edição |
| logging-nudge | após editar .py | nudge | Observabilidade canônica por radar |
| py-discipline | após editar .py | nudge | Disciplina Python/testes |
| stop-loop-reminder | fim | nudge | Aprendizado registrado + cobertura lembrada |

---

## 5.8 O que levar desta posição para a aula

- Hooks **não são a IA decidindo** — são regras **determinísticas** que disparam sozinhas. É o pilar
  que converte "intenção" em "garantia".
- A distinção mestra: **guard (trava/`deny`)** para o risco inegociável; **nudge (cutucada/aviso)**
  para o que exige julgamento. Escolher o instrumento certo é a engenharia.
- Os nudges transformam contratos densos (como o de logs) em **radares automáticos** que corrigem no
  momento mais barato — a hora da edição.
- `session-start` + `stop-loop` **fecham o ciclo de aprendizado automaticamente** — a empresa aprende
  por construção, não por disciplina individual.

**Próximo:** [Capítulo 6 — A Partida (o time inteiro em ação)](capitulo-06-a-partida.md).
</content>
