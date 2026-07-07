# Capítulo 5 — Poka-yoke e Andon (`.claude/hooks/`)

> 🏭 [↩ Doc-mãe da lente](README.md) · [↩ Índice da metodologia](../README.md)

> **Parte da fábrica:** os hooks são os **dispositivos à prova de erro (poka-yoke), o cordão Andon e os
> sensores automáticos** da linha. Diferente de tudo que vimos, **eles não são a IA decidindo** — são
> mecanismos do sistema que disparam **sozinhos**, em momentos fixos, de forma **determinística**. A
> Norma, os SOPs e as estações dependem de o agente "lembrar" e "querer" cumprir. O dispositivo não pede
> licença: ele bloqueia a montagem errada ou acende o alarme, independentemente do que o operador
> pretendia.

> 🧑‍💼 **RESUMO EXECUTIVO.** Esta é a camada que transforma "boa intenção" em "garantia". Toda regra
> importante o suficiente para não poder falhar nós tiramos da confiança e colocamos no dispositivo. É a
> diferença entre "pedimos para todos seguirem o procedimento" e "a peça não encaixa se montada errado".
> O valor: as regras críticas viram **impossíveis de esquecer**, não importa a pressa do turno.

---

## 5.1 O conceito-chave: poka-yoke (trava) vs. sensor/alarme (aviso)

No léxico lean, **poka-yoke** é o dispositivo à prova de erro — torna o defeito **impossível** (o conector
só entra de um jeito). Já um **sensor/alarme** apenas **avisa** que algo saiu do padrão e deixa o operador
decidir. É a mesma distinção que, na lente futebol, aparece como *guard* (trava) vs. *nudge* (cutucada).

Na nossa linha, isso cria **dois tipos de intervenção**, e a distinção é o coração deste capítulo:

| Tipo | Na fábrica | O que faz | Quando usamos |
|---|---|---|---|
| **Guard (poka-yoke / Andon)** | O dispositivo trava, ou o cordão **para a linha** | **Bloqueia** a ação (`deny`) | Risco grave e claro (destruição, travar a sessão) |
| **Nudge (sensor/alarme)** | A luz amarela acende — a linha **segue**, com aviso | **Avisa**, injeta um lembrete, **não bloqueia** | Disciplina e qualidade onde o operador precisa de julgamento |

**Por que ter os dois?** Porque nem tudo merece parar a linha. Travar demais engessa a produção e gera
falsos positivos irritantes; travar de menos deixa passar o que importa. A engenharia está em **escolher
o instrumento certo para cada risco**: poka-yoke para o inegociável, alarme para o que exige bom senso.

---

## 5.2 Quando cada dispositivo dispara (os momentos da linha)

Os hooks estão amarrados no `settings.json` a **dois momentos** — antes da operação e depois da edição:

```
  PreToolUse   ──►  antes de um comando (Bash)   ──►  bash-guard + worktree-guard   (poka-yoke: pode travar)
               ──►  antes de criar arquivo (Write) ─► write-guard                    (poka-yoke: pode travar)
  PostToolUse  ──►  depois de editar um arquivo  ──►  py-lint / logging-nudge / py-discipline  (sensores)
```

Não há dispositivo de início nem de fim de turno ligado hoje: a pasta contém um sétimo script
(`stop-loop-reminder.sh`) **pronto mas não conectado** — lacuna conhecida, detalhada no §5.5.

---

## 5.3 Os guards (poka-yoke / Andon) — `PreToolUse`

São os que **bloqueiam de fato**. Disparam *antes* da ação e podem dizer "não".

### `bash-guard.sh` — o dispositivo no comando de terminal

Inspeciona todo comando shell antes de rodar. Três níveis:

1. **Trava a varredura cega de `/logs`.** Se o comando usa `ls/find/grep/cat/tail…` na pasta de logs
   **sem** um `correlation_id`, ele **bloqueia** — a pasta pode ter dezenas de milhares de arquivos e
   **travar a sessão inteira**. É a aplicação automática de uma regra do `log-instructions.md`. *(O SOP
   pede; o dispositivo garante.)*
2. **Trava comandos destrutivos.** `rm -rf`, `drop table`, `truncate table`, `git push --force` — tudo
   bloqueado, com orientação de confirmar com o humano e rodar manualmente fora do agente.
3. **Alerta (não bloqueia) desvios menores:** gravar em `/tmp` em vez de `./.sandbox/tmp/`, ou `psql` sem
   desabilitar o pager.

**Detalhe de engenharia importante:** o dispositivo **falha aberto** — se algo der errado na própria
checagem, ele deixa o comando passar. Um poka-yoke que, na dúvida do próprio funcionamento, não trava a
produção indevidamente. Segurança sem virar obstáculo.

**Defeito que evita:** dois acidentes clássicos de automação — **travar a sessão** com uma listagem
gigante e **destruir dados** com um comando irreversível.

### `worktree-guard.sh` — o intertravamento entre células paralelas

O segundo dispositivo no comando de terminal protege um risco específico do trabalho em paralelo: várias
janelas/agentes operam ao mesmo tempo em **worktrees** (células de produção isoladas — `worktree.md`,
Cap. 2), e um `git worktree remove` ou `git branch -D` descuidado pode **destruir o trabalho de outra
sessão de forma irreversível**. Como o naming é fixo (`worktree-codex`/`worktree-claude`), o nome não
prova posse — então o dispositivo **nega sempre**.

A única válvula de escape é um **token nominal**: prefixar o comando com
`CONFIRMO_EXCLUSAO_WT_ALHEIA=<nome-exato-do-worktree>`, obtido só após confirmação expressa do humano
para **aquele alvo exato** — "não existe bypass genérico ou em lote".

**Detalhe de engenharia — o contraste com o bash-guard:** este dispositivo **falha fechado**: na dúvida,
**trava**. A assimetria é deliberada e é a mesma de um intertravamento de prensa (lockout-tagout): o
bash-guard protege contra travar a produção (na dúvida do próprio funcionamento, deixa passar); o
worktree-guard protege contra **destruir produção alheia irreversível** (na dúvida, nega). A política de
falha de cada poka-yoke é escolhida pela **irreversibilidade do dano**, não por padrão único.

### `write-guard.sh` — o dispositivo no arquivo

Impede a criação de testes pytest em diretórios proibidos (`tests/manual|disabled|temp`). É a aplicação
automática do `CLAUDE.md §11` / `tests/CLAUDE.md`: ensaio tem que nascer no lugar coberto pela suíte
oficial, com o marker de família correto.

**Defeito que evita:** ensaio "fantasma" criado num canto que a suíte oficial não roda — cobertura que
parece existir mas não protege nada.

---

## 5.4 Os nudges (sensores/alarmes) — `PostToolUse`

Disparam **depois** de cada `Edit`/`Write` de arquivo Python e **injetam lembretes** sem travar. São
luzes amarelas: acendem, o operador decide.

### `py-lint.sh` — o sensor de acabamento (estilo)

Roda `ruff check` no arquivo recém-editado e, se houver questões, injeta a saída no contexto — com a
ressalva sábia: *"corrija se pertinente ao seu escopo, sem ampliar o recorte"*. Alerta, mas respeita o
**escopo mínimo** (não vira desculpa para reprojetar o arquivo inteiro).

### `logging-nudge.sh` — o sensor de rastreabilidade

O mais sofisticado. Lê o `.py` editado e procura **sinais concretos** de violação da especificação de
logs, alertando quando encontra:
- `logger.error(...)` dentro de `except` → deveria ser `logger.exception(...)` (preserva o stack trace);
- f-string em chamada de log → deveria ser lazy `%` formatting;
- `import logging` / `logging.getLogger` → logger padrão do Python em vez do oficial do projeto;
- `print()` em código de `src/` → deveria ser o logger oficial;
- `extra={...event_name...}` manual → deveria usar o builder canônico;
- usa logging mas não referencia nenhum helper canônico.

Repare: detecta **por leitura estática** o que viraria um defeito de rastreabilidade *antes* de a peça
rodar. É a especificação `log-instructions.md` transformada em sensor automático.

### `py-discipline.sh` — o sensor de disciplina Python

Alerta três coisas:
- `except ImportError`/`ModuleNotFoundError` → import opcional/lazy é proibido (se o pacote falta, deve
  falhar);
- `FOR UPDATE` → lock pessimista, peça justificativa e prefira controle otimista;
- arquivo de teste **sem marker de família** no topo.

**Defeito que os sensores evitam:** a erosão silenciosa da qualidade. Sozinha, nenhuma dessas coisas
derruba o sistema — mas, acumuladas, viram dívida técnica e cegueira de rastreabilidade. O sensor corrige
no **momento exato da edição**, quando é barato consertar, em vez de numa inspeção semanas depois.

> 🛠️ **Por que sensor e não poka-yoke aqui?** Porque essas regras exigem **julgamento de contexto**. Um
> `print()` pode ser legítimo num script de `.sandbox`; um `FOR UPDATE` pode ser realmente necessário.
> Travar seria arrogante e geraria falso positivo. O alarme respeita a inteligência do operador: "olha
> isso, decida". Poka-yoke para o inegociável; alarme para o que pede bom senso.

---

## 5.5 O sensor de fim de turno que existe mas não está ligado (lacuna declarada)

Na pasta de hooks há um sétimo script, `stop-loop-reminder.sh`, projetado para o fim da rodada (`Stop`):
se a rodada editou `src/**.py` **sem tocar em `tests/`**, ele acenderia o alarme de cobertura de ensaios
e observabilidade. Só isso — é um sensor fino de "peça sem ensaio", não um cobrador de lições.

**Estado real: pronto, mas não conectado.** O script não está registrado em nenhum `settings`, então
nunca dispara. É uma **lacuna conhecida e declarada** — na fábrica, um sensor já fabricado aguardando
instalação no painel. Enquanto isso, a checagem que ele automatizaria continua existindo como regra (a
Norma exige teste proporcional ao risco) e como estação (`validar-entrega` reprova entrega sem ensaio) —
mas depende de disciplina, não de dispositivo.

E o Kaizen? **Não mora nos hooks.** O mecanismo de aprendizado da fábrica é o trio contrato dos loops +
memória de rodada + promoção com gate (Cap. 2, §2.2) — acionado pelo **defeito provado**, não por um
lembrete de fim de turno.

---

## 5.6 `settings.json` e `settings.local.json` — o roteiro mestre e as autorizações

- **`settings.json`** é o **roteiro de fabricação**: amarra cada dispositivo ao seu momento
  (`PreToolUse` para os três guards, `PostToolUse` para os três sensores). É o que faz a linha de
  proteção existir — script fora dele é sensor na prateleira.
- **`settings.local.json`** é a lista de **autorizações locais**: pré-autoriza operações seguras e
  repetitivas — a suíte oficial (`python suite_de_testes_padrao.py`), `jest`, `ruff check`, a CLI de
  análise de logs (`python -m src.log_analyzer`) e a consulta PostgreSQL canônica
  (`run_postgresql_query.py`) — para não pedir confirmação a cada vez. É produtividade sem abrir mão de
  segurança — só entra aqui o que é comprovadamente seguro.

---

## 5.7 A síntese dos dispositivos

| Dispositivo | Momento | Tipo | Garante |
|---|---|---|---|
| bash-guard | antes de Bash | **poka-yoke** + alarme (falha aberto) | Não trava a sessão, não destrói dados |
| worktree-guard | antes de Bash | **poka-yoke** (falha fechado, token nominal) | Célula alheia não é destruída |
| write-guard | antes de Write | **poka-yoke** | Ensaio nasce no lugar certo |
| py-lint | após editar .py | alarme | Acabamento limpo no momento da edição |
| logging-nudge | após editar .py | alarme | Rastreabilidade canônica por sensor |
| py-discipline | após editar .py | alarme | Disciplina Python/ensaios |
| stop-loop-reminder | (não ligado) | alarme na prateleira | Lacuna declarada: aviso "src sem tests" |

---

## 5.8 O que levar deste capítulo

- Hooks **não são a IA decidindo** — são dispositivos **determinísticos** que disparam sozinhos. É o
  pilar que converte "intenção" em "garantia".
- A distinção mestra: **poka-yoke/Andon (trava/`deny`)** para o risco inegociável; **sensor/alarme
  (aviso)** para o que exige julgamento. Escolher o instrumento certo é a engenharia.
- A **política de falha** de cada trava é escolhida pela irreversibilidade do dano: bash-guard falha
  aberto; worktree-guard falha fechado, com token nominal e sem bypass em lote.
- Os sensores transformam especificações densas (como a de logs) em **radares automáticos** que corrigem
  no momento mais barato — a hora da edição.
- A honestidade faz parte do desenho: um sensor pronto e não ligado é **declarado como lacuna**, não
  vendido como proteção. E o Kaizen não mora nos hooks — mora nos loops estratégicos (Cap. 2 e 6).

**Próximo:** [Capítulo 6 — Um Lote na Linha (a fábrica inteira em ação)](capitulo-06-um-lote-na-linha.md).
