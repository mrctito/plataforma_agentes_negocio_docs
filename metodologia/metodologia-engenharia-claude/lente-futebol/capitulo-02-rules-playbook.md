# Capítulo 2 — As Jogadas Ensaiadas (`.claude/rules/`)

> **Posição em campo:** se o `CLAUDE.md` é o DNA do clube, a pasta `.claude/rules/` é o **caderno de
> jogadas ensaiadas e os manuais de posição**. Nenhum desses arquivos joga sozinho — eles são
> **consultados sob demanda** pelos jogadores (agentes) no momento exato da jogada. O zagueiro
> consulta o manual de marcação; o goleiro consulta o manual de saída de bola. Aqui está o
> conhecimento profundo que seria pesado demais para carregar em toda sessão.

> 🧑‍💼 **RESUMO EXECUTIVO.** Estes arquivos são o nosso "conhecimento institucional codificado".
> Eles garantem que decisões difíceis (como logar para diagnosticar, como saber se algo está
> realmente pronto, quando reutilizar em vez de criar) sejam tomadas **do mesmo jeito, sempre, por
> qualquer agente** — em vez de depender de quem está no teclado. É a diferença entre uma empresa
> cujo conhecimento mora na cabeça de poucos e uma cujo conhecimento está escrito e aplicado.

---

## 2.0 Por que separar `rules/` do `CLAUDE.md`

Uma decisão de arquitetura: o `CLAUDE.md` precisa ser leve (lido sempre), mas alguns contratos são
densos (o de logs tem centenas de linhas). Misturar tudo tornaria cada sessão cara e difícil de ler.
A solução é a mesma de um bom software: **núcleo estreito + módulos carregados sob demanda.**

Os arquivos de `rules/` dividem-se em dois tipos:

- **Contratos profundos** (jogadas ensaiadas): a versão completa de um tema. Ex.: `log-instructions.md`.
- **Registros operacionais** (o caderno de aprendizado do time): memória viva que muda a cada rodada.
  Ex.: `lessons.md`, `error-backlog.md`.

---

## 2.1 Os contratos profundos (os manuais de posição)

### `log-instructions.md` — o manual do raio-X

O contrato mais valioso do time. Define que **todo log é evidência operacional**, não enfeite — um
"raio-X" que permite reconstruir um processo *depois* que ele aconteceu, como um debug offline.

**O que força (conceitos de engenharia):**
- **`correlation_id` canônico:** nasce uma única vez no boundary oficial (o endpoint inicial, ou o
  iniciador real em scheduler/job). Nenhuma camada abaixo cria, deriva ou substitui — só propaga. A
  response devolve o mesmo id (corpo + header), com força máxima em erros.
- **Campos canônicos:** fonte única em `src/core/log_canonical_fields.py`; usa **builders oficiais**
  (`build_canonical_log_context`, builders de slice). Proibido inventar campo, alias local
  (`step`/`etapa` no lugar de `stage`) ou montar `extra={"event_name": ...}` na mão.
- **`logger.exception` dentro de `except`:** obrigatório (preserva o stack trace); `logger.error` ali
  é tratado como defeito de observabilidade.
- **Anti-varredura de `/logs`:** proibido listar a pasta cegamente (dezenas de milhares de arquivos
  podem travar a sessão). Só se busca com um `correlation_id` em mãos. *(Esta regra é tão crítica que
  também é aplicada por um hook — ver [Cap. 5](capitulo-05-hooks-arbitragem.md).)*

**Problema de engenharia que evita:** o bug que você não consegue diagnosticar. Em sistemas
distribuídos (3 containers, workers paralelos, filas), sem rastreabilidade você fica cego. Este
contrato garante que o log conte a história inteira: entrada, decisões, delegações, chamadas externas,
paralelismo real, erros e estado final.

**Valor que incentiva:** observabilidade por construção, debug offline, rastreabilidade ponta a ponta.

> 🧑‍💼 Pense neste arquivo como a "caixa-preta de avião" do nosso software. Quando algo dá errado em
> produção, a diferença entre "resolvemos em minutos" e "passamos dias no escuro" mora aqui.

### `reuso-instructions.md` — o manual do "olhe o elenco antes de contratar"

Antes de criar qualquer classe, helper, service, adapter, resolver, validator: é **obrigatório**
pesquisar o que já existe, consultar `docs/README-TOOLS-LIB.md`, ler os candidatos diretamente, e só
então — com justificativa — criar algo novo.

**O que força:** uma ordem de 9 passos que termina em "somente depois disso considerar criar algo
novo". E uma postura de **advogado do diabo**: "em repositório grande, solução nova é rara".

**Problema que evita:** duplicação. Em sistema maduro, reimplementar um helper que já existe gera
**divergência de comportamento** (dois jeitos de fazer a mesma coisa, que evoluem diferente), dobra a
manutenção e espalha inconsistência (ex.: dois tratamentos de erro diferentes para a mesma operação).

**Valor:** DRY, consistência, manutenibilidade. E um achado importante: se a busca encontra **duas**
soluções para o mesmo problema, isso é tratado como **inconsistência arquitetural** a ser unificada —
e é proibido adicionar uma terceira.

### `definicao-de-pronto.md` — o manual do VAR (anti-falso-verde)

A versão profunda do `CLAUDE.md §14`. Define o que significa "pronto" de forma **falsificável**.

**O conceito central — falso-verde:** "quando a entrega parece correta porque testes isolados
passaram, a documentação foi atualizada ou uma classe nova foi criada, mas o caminho oficial do
sistema em runtime ainda não usa a implementação nova".

**O que força:**
- **Hierarquia de verdade:** código executável > teste observado > log real > plano > doc > comentário.
- **Check discriminante:** "o check mais barato capaz de desconfirmar a hipótese da entrega" — verifica
  a implementação concreta usada no wiring final, não só a existência do componente novo.
- **Teste de regressão estrutural:** todo trabalho estrutural inclui um teste que **falha se o caminho
  oficial voltar à implementação antiga**.
- **Ordem obrigatória:** provar o wiring → ligar a dependência → teste de regressão → validar runtime →
  **só então** atualizar documentação.

**Problema que evita:** a falsa sensação de conclusão. É o golpe mais comum em automação: tudo parece
verde, mas o sistema real não mudou. Um gol que o VAR anula por impedimento.

**Valor:** honestidade de entrega; prova no caminho oficial.

### `suite-testes-instructions.md` — o manual de qual treino aplicar

A referência canônica de como rodar a suíte oficial: qual "gate" (modo) usar para cada situação,
sintaxe pública, flags, e a regra de ouro **"se este arquivo divergir do código executável, o código
vence"**.

**O que força:** escolher o modo proporcional (`--focus-tests` para slice pequeno, `--complete` para
fechamento, etc.), sempre com `--run-id`, sempre auditando `telemetry.json`/`state.json` — "não declare
sucesso olhando apenas exit code ou terminal".

**Problema que evita:** validação desproporcional (rodar tudo para uma mudança pequena, ou o contrário)
e o "verde de terminal" sem auditoria real.

**Valor:** validação proporcional ao risco, com evidência auditável.

### `python.md` — o manual técnico da posição (Python)

Complemento especializado para código Python: 100% tipado (mypy --strict), proibido `Any` fora de
boundary, proibido `try/except ImportError` em teste, proibido stub de framework real (LangChain,
FastAPI…), markers de família obrigatórios em pytest.

**Problema que evita:** tipagem frouxa, dependência opcional mascarada, teste que simula framework (e
portanto não testa nada real).

**Valor:** segurança de tipos e testes honestos. *(Vários destes itens viram nudges automáticos — Cap. 5.)*

### `large-repo-navigation.md` — o manual de leitura de jogo em campo grande

Como agir num repositório massivo sem se perder: começar por um **anchor** concreto (arquivo, símbolo,
log, teste falhando), orçamento de exploração inicial, particionar pedido "repo-wide" em slices, e
"busca localiza, leitura comprova".

**Problema que evita:** exploração cega que estoura contexto e custo; conclusão arquitetural baseada em
busca textual rasa; ampliação de escopo por reflexo.

**Valor:** navegação disciplinada, proporcional ao risco.

### `estrategia-recomendacoes.md` e `fidelidade-pedido-usuario.md` — os contratos de passe

Estes dois são **contratos de handoff** — definem como um jogador entrega a bola ao próximo sem que a
informação se perca:

- **`estrategia-recomendacoes.md`:** define a seção `# Estratégia/Recomendacoes` que o `planejar`
  cria e o `implementar` é **obrigado a ler antes de tocar qualquer arquivo**. É a "bússola
  operacional" do plano: o que preservar, mudar, remover; quais fallbacks são proibidos; o gate de
  não-parada. Existe num arquivo único porque é um contrato **entre dois agentes** — se vivesse
  duplicado dentro de cada um, as pontas divergiriam.
- **`fidelidade-pedido-usuario.md`:** obriga a reapresentar o pedido do usuário fielmente (sem reduzir
  a um rótulo genérico) e a destacar as `REGRAS E PREMISSAS IMPERATIVAS DO USUARIO`. Garante que uma
  exigência explícita do usuário não "evapore" no meio do pipeline.

**Problema que evita:** a "fofoca corporativa" — a informação que se distorce a cada repasse.
**Valor:** integridade do contrato entre etapas.

> 🛠️ **Detalhe de engenharia elegante:** estes contratos foram extraídos para arquivos próprios
> exatamente para **eliminar duplicação entre agentes**. É o princípio DRY aplicado à própria
> configuração da IA — os agentes *referenciam* o contrato, não o redefinem. A configuração obedece
> às mesmas regras que impõe ao código.

---

## 2.2 O caderno de aprendizado (a memória viva do time)

Estes quatro arquivos não são contratos fixos — são **memória que muda a cada rodada**. É aqui que o
time "aprende com o jogo".

| Arquivo | Papel no time | O que registra |
|---|---|---|
| `lessons.md` | A lição do treinador, gravada | Apenas lições com **valor preventivo transversal** — regras que mudam como os agentes pensam em mais de um slice |
| `error-backlog.md` | O relatório médico | Todo erro real de produto (com cenário, módulo, evidência) |
| `regression-logs.md` | A reincidência de lesão | Erro já registrado que **voltou** — tratado como regressão grave |
| `bad-instructions.md` | A ouvidoria do regulamento | Contradições, ambiguidades e lacunas nas próprias instruções |

**A engenharia por trás:** `lessons.md` tem uma **regra de curadoria rígida** — não é diário nem
backlog. Antes de promover uma lição, a pergunta obrigatória é: *"se outro agente atuar amanhã em
outro slice, esta regra ainda reduziria chance real de erro?"* Se não, ela fica na memória local da
skill. Isso impede que o arquivo central vire um depósito inútil.

**Problema que evita:** o time repetir o mesmo erro; e, do outro lado, a "poluição de memória" — um
arquivo de lições tão cheio de detalhe local que ninguém lê.

**Valor:** melhoria contínua **institucional** (o conhecimento fica na empresa, não na pessoa). E note
a simetria: este caderno é alimentado automaticamente pelos hooks de início e fim de sessão
(ver [Cap. 5](capitulo-05-hooks-arbitragem.md)) — o time é *lembrado* de aprender.

> 🧑‍💼 **RESUMO EXECUTIVO.** Este é o mecanismo que faz a nossa engenharia ficar **melhor com o
> tempo, sozinha**. Cada erro vira uma regra que previne o próximo. É capitalização de conhecimento:
> o que aprendemos numa quarta-feira protege todos os projetos da quinta em diante.

---

## 2.3 O que levar desta posição para a aula

- `rules/` é o **conhecimento profundo carregado sob demanda** — o detalhe que não cabe no DNA.
- Os **contratos profundos** (log, reuso, pronto, suíte, python, large-repo) são os manuais de posição.
- Os **contratos de handoff** (estratégia, fidelidade) existem em arquivo próprio para **não duplicar**
  entre agentes — DRY na própria config.
- O **caderno de aprendizado** (lessons, error-backlog, regression, bad-instructions) é a memória viva
  que faz o time melhorar com o tempo.
- A regra de ouro recorrente: **"se divergir do código executável, o código vence"** — a doc nunca se
  acha dona da verdade.

**Próximo:** [Capítulo 3 — Os Jogadores em Campo (`.claude/agents/`)](capitulo-03-agents-jogadores.md).
</content>
