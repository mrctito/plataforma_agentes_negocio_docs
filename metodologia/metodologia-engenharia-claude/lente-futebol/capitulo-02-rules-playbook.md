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

## 2.0 Por que separar `rules/` do `CLAUDE.md` — e como a jogada certa entra em campo sozinha

Uma decisão de arquitetura: o `CLAUDE.md` precisa ser leve (lido sempre), mas alguns contratos são
densos (o de logs tem centenas de linhas). Misturar tudo tornaria cada sessão cara e difícil de ler.
A solução é a mesma de um bom software: **núcleo estreito + módulos carregados sob demanda.**

O caderno tem **20 jogadas**, e o mecanismo de entrega é tão importante quanto o conteúdo: **19 das 20
têm um frontmatter `paths:`** — a jogada é injetada automaticamente no contexto quando o trabalho toca
os caminhos declarados. E vários desses `paths:` apontam para **os próprios agentes** (ex.:
`estrategia-recomendacoes.md` casa `.claude/agents/implementar.md` — o manual entra em campo junto com
o jogador que o usa). É **injeção de dependência de contexto**: nenhum agente copia a regra para
dentro de si, então não existe cópia para divergir. A única jogada sem `paths:` é
`qualidade-texto.md` — porque ela vale para **todo** texto, em qualquer setor do campo.

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
pesquisar o que já existe lendo o código com `read` (áreas de alta chance: `src/core` e
`src/api/services`; inventário complementar em `docs/tecnico/README-TOOLS-LIB.md`), e só então — com
justificativa — criar algo novo.

**O que força:** uma ordem de **6 passos** — identificar candidatos → buscar implementações
relacionadas → **ler os arquivos direto** → entender a intenção → avaliar se resolve/evolui com
segurança → só então considerar criar algo novo. E uma postura de **advogado do diabo**: "em
repositório grande, solução nova é rara".

**Problema que evita:** duplicação. Em sistema maduro, reimplementar um helper que já existe gera
**divergência de comportamento** (dois jeitos de fazer a mesma coisa, que evoluem diferente), dobra a
manutenção e espalha inconsistência (ex.: dois tratamentos de erro diferentes para a mesma operação).

**Valor:** DRY, consistência, manutenibilidade. E um achado importante: se a busca encontra **duas**
soluções para o mesmo problema, isso é tratado como **inconsistência arquitetural** a ser unificada —
e é proibido adicionar uma terceira.

### `definicao-de-pronto.md` — o manual do VAR (anti-falso-verde)

O contrato que define o que significa "pronto" de forma **falsificável** (morava no DNA; hoje é
jogada ensaiada carregada por caminho — o raiz emagreceu delegando).

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

### `loops-estrategicos.md` — a jogada de recuperação de bola

A fonte única dos **dois loops** que o DNA (§1) declara como comportamento default: auto-correção por
log e memória de rodada. É o coração do aprendizado do time — detalhado na [seção 2.2](#22-o-playbook-real-de-aprendizado-os-dois-loops-e-a-promoção-de-lição)
logo abaixo.

### `regras_uso_subagentes.md` — o teto salarial: modelo por risco

Decide quando um subagente pode rodar no modelo barato (Sonnet 5) e quando exige o forte. Duas travas
têm prioridade sobre tudo: **Trava 1 (falso negativo)** — busca para *coletar evidência* pode ser
barata; busca para **afirmar que algo não existe** ("vazio", "sem reuso", "100% coberto") fica no
modelo forte, porque o falso negativo é o erro de alto impacto e baixa detectabilidade. **Trava 2** —
antes de baixar o *tier*, baixe o *effort*. A pergunta de corte: *"se o subagente errar, o principal
percebe e corrige fácil?"*. (A aplicação real, jogador a jogador, está no [Cap. 3](capitulo-03-agents-jogadores.md).)

### `ferramentas-acesso-dados.md` — o manual do "vá ver com os próprios olhos"

Abre com a instrução "**LEIA ANTES de dizer 'não tenho como verificar'**": inventário dos ~46 scripts
prontos em `.claude/scripts/` (Qdrant, PostgreSQL, Redis, filas, scheduler, logs). Proibido concluir
estado de recurso por proxy (grep, código, doc, log) e proibido reinventar acesso ad-hoc quando existe
script — se faltar ferramenta, cria-se um script reutilizável e atualiza-se o inventário.

### `disciplina-investigacao-teste.md` — a trava anti-teste-raso

Falha, vazio ou erro são o **início** da investigação, não conclusão: enumerar as hipóteses (bug,
config, wiring, dado ausente, credencial, ambiente) → testar **cada uma na fonte de verdade real** →
só então rotular a causa. Com a taxonomia que evita o erro clássico: "tabela-registro vazia" ≠ "store
físico vazio"; "log diz X" ≠ "estado real é X".

### `execucao-plano-resumivel.md` — o revezamento que não perde o jogo por lesão

O padrão de execução de plano longo: **orquestrador quente** (a janela principal, que nunca
re-onboarda e segura o plano) + **worker descartável** (um subagente `implementar` por fase, que morre
ao devolver) + **diário write-ahead** (status gravado por tarefa no arquivo do plano, antes de
avançar). Se o worker cair, a perda máxima é **a tarefa em voo** — nunca o plano. Com o contrapeso:
plano de até ~4 tarefas não se fracciona. (Como a skill `/implementar` carrega essa política:
[Cap. 4](capitulo-04-skills-convocacao.md).)

### `qualidade-texto.md` — o manual do discurso

Anti-linguiça **e** anti-lacuna: todo acréscimo precisa agregar valor real; conteúdo necessário entra
mesmo longo, redundante sai mesmo curto — a régua é **densidade de valor por linha**. A única jogada
sem `paths:`, porque vale para todo texto, inclusive a resposta ao usuário.

### `worktree.md` — o manual do treino em campo separado

Paralelismo com isolamento: trabalho paralelo roda em worktrees de nome fixo
(`worktree-codex`/`worktree-claude`), criados fora do repositório principal, com foco exclusivo no
próprio worktree. É a única jogada com **trava de máquina dedicada**: o hook `worktree-guard`
([Cap. 5](capitulo-05-hooks-arbitragem.md)) impede excluir worktree alheio.

---

## 2.2 O playbook real de aprendizado (os dois loops e a promoção de lição)

O time não aprende por mágica nem por um arquivo central de lições: aprende por **dois loops
declarados como features estratégicas de primeira classe** em `loops-estrategicos.md` — e citados no
próprio DNA (§1) como comportamento padrão obrigatório, não boa intenção.

**Loop 1 — auto-correção por log (o ajuste dentro do jogo).** Gatilho: qualquer **erro real de
backend** durante teste, validação ou execução de plano. É **proibido** declarar "ambiente / fora de
escopo / credencial / dado faltando" por inferência, sem antes provar a causa pelo log. Os 5 passos:

1. capturar o `correlation_id` (a response o devolve no corpo e no header `X-Correlation-Id`);
2. abrir o log oficial via `python -m src.log_analyzer` (nunca varrer `/logs` cegamente);
3. **cara-crachá**: confrontar cada evento do log com o código do caminho do erro, até a causa raiz
   **provada** — não hipótese;
4. decidir pela evidência: causa de escopo → corrigir na origem e voltar ao teste; causa fora do
   escopo → registrar **com o log como prova** ("fora-de-escopo é conclusão provada, não atalho");
5. response sem `correlation_id`? Isso é, em si, uma falha de observabilidade a registrar.

E em camadas, para não usar canhão em mosca: reflexo barato (`log_analyzer` inline) → `analisar-log`
(diagnóstico fundo, sem corrigir) → `corrigir-com-log`/`corrigir-erros-com-log` (correção forense
dedicada).

**Loop 2 — memória de rodada (anti-repetição na mesma campanha).** Dentro de uma campanha de
correção, o agente registra cada tentativa com evidência objetiva (erro dominante, hipótese, checagem
discriminante, ação, resultado, próximo passo) e **relê a memória antes de cada nova hipótese** — para
nunca repetir um beco sem saída já falsificado. É memória local à campanha.

**A promoção durável — o caderno que existe de verdade.** Quando uma rodada revela uma **regra
preventiva durável e comprovada**, ela é promovida ao `licoes-aprendidas.md` do agente — hoje os
cadernos mais grossos são os de `corrigir-erros-com-log` (~22 KB de lições) e `testar-ingestao-dnit`
(~24 KB), cada um acompanhado do seu `log-rodada.md` (a memória efêmera do Loop 2). O gate de
curadoria é explícito: **"não promover ruído operacional local"** — detalhe de uma rodada fica na
memória de rodada; só sobe ao caderno a regra que ainda reduziria erro em rodadas futuras. Até os
scripts ad hoc têm esse ciclo: descartar, manter ou promover para `.claude/scripts/`.

**A engenharia por trás:** separar o **efêmero** (log-rodada, por campanha) do **durável**
(licoes-aprendidas, por agente) com um gate de promoção no meio. Sem o gate, o caderno central viraria
um depósito que ninguém lê; sem a memória de rodada, cada campanha repetiria as próprias tentativas.

> 🧑‍💼 **RESUMO EXECUTIVO.** Este é o mecanismo que faz a nossa engenharia ficar **melhor com o
> tempo**: cada erro real vira, primeiro, uma correção provada por evidência (Loop 1); depois, se a
> lição for durável, vira regra preventiva escrita no caderno do especialista. É capitalização de
> conhecimento — o que aprendemos numa quarta-feira protege as rodadas da quinta em diante.

---

## 2.3 O playbook completo — as 20 jogadas num relance

| Jogada | Em uma linha |
|---|---|
| `log-instructions` | O contrato do raio-X: correlation_id, campos canônicos, builders, anti-varredura |
| `reuso-instructions` | Os 6 passos obrigatórios antes de criar qualquer coisa nova |
| `definicao-de-pronto` | Pronto = ativo no caminho oficial em runtime (anti-falso-verde) |
| `suite-testes-instructions` | Qual modo da suíte rodar, sempre com `--run-id` e auditoria de telemetria |
| `python` | Tipagem estrita, mocks honestos, markers de família |
| `large-repo-navigation` | Âncora + fatia: navegar código enorme sem estourar contexto |
| `estrategia-recomendacoes` | O contrato da bússola entre `planejar` e `implementar` |
| `fidelidade-pedido-usuario` | O pedido do usuário não evapora no pipeline |
| `loops-estrategicos` | Os dois loops: auto-correção por log + memória de rodada (§2.2) |
| `regras_uso_subagentes` | Modelo por risco: as 2 travas contra o falso negativo barato |
| `ferramentas-acesso-dados` | Fonte de verdade real: scripts prontos; proibido concluir por proxy |
| `disciplina-investigacao-teste` | Enumerar hipóteses e testá-las na fonte antes de rotular causa |
| `execucao-plano-resumivel` | Orquestrador quente + worker descartável + diário write-ahead |
| `qualidade-texto` | Anti-linguiça e anti-lacuna; densidade de valor por linha |
| `worktree` | Trabalho paralelo isolado em `worktree-codex`/`worktree-claude` |
| `ambiente-local` | Procedimentos do ambiente local (FastAPI/"porta presa") e automação de navegador |
| `componente-chat-embutivel` | Fonte única do chat embutível (`PrometeuEmbeddableChatRuntime`): gatilho de reuso |
| `dyn-sql-tools-registro` | Contrato das tools `dyn_sql`: inline no YAML vence, registro é fallback |
| `padrao-listas-perguntas-teste` | Formato simétrico das listas de perguntas dos testes de chat |
| `subagentes-descricao-instructions` | Regra arquitetural das descrições de subagentes (validador `./validar_descricao_subagentes.sh`) |

---

## 2.4 O que levar desta posição para a aula

- `rules/` é o **conhecimento profundo carregado sob demanda** — 20 jogadas, 19 delas com `paths:`:
  a jogada entra em campo sozinha quando a bola chega no setor (**injeção de dependência de contexto**,
  inclusive apontando para os próprios agentes).
- Os **contratos profundos** (log, reuso, pronto, suíte, python, large-repo, acesso a dados,
  subagentes…) são os manuais de posição; os **contratos de handoff** (estratégia, fidelidade) vivem
  em arquivo próprio para **não duplicar** entre agentes — DRY na própria config.
- O **aprendizado real** são os dois loops (`loops-estrategicos.md`): auto-correção provada por log +
  memória de rodada, com **promoção curada** de lição durável para o `licoes-aprendidas.md` de cada
  agente.
- A regra de ouro recorrente: **"se divergir do código executável, o código vence"** — a doc nunca se
  acha dona da verdade.

**Próximo:** [Capítulo 3 — Os Jogadores em Campo (`.claude/agents/`)](capitulo-03-agents-jogadores.md).
