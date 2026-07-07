# Capítulo 1 — A Filosofia do Clube (`CLAUDE.md`)

> **Posição em campo:** o `CLAUDE.md` não é um jogador. Ele é o **DNA do clube** — a filosofia de
> jogo que todo jogador carrega ao entrar em campo. O raiz é lido em **100% das sessões**, antes de
> qualquer trabalho. Se um agente é um jogador e uma skill é a convocação, o `CLAUDE.md` é aquilo
> que faz o time inteiro jogar do mesmo jeito, mesmo sem combinar.

> 🧑‍💼 **RESUMO EXECUTIVO.** Este arquivo é a nossa "constituição de engenharia". Ele transforma
> preferências dispersas ("ah, seria bom testar", "tenta não duplicar código") em **lei inegociável
> aplicada automaticamente em toda sessão**. O valor de negócio: a qualidade deixa de depender do
> humor ou da pressa do dia e passa a ser um piso garantido. É o que permite escalar a IA sem
> escalar o caos.

---

## 1.1 Por que existe um arquivo lido sempre

Um time sem filosofia comum vira onze individualidades. Cada sessão de IA, sem um DNA, recomeçaria do
zero, com critérios diferentes. O `CLAUDE.md` resolve isso sendo **carregado em toda sessão** e tendo
precedência sobre o comportamento padrão. Ele é curto de propósito e **aponta para os contratos
profundos** (as jogadas ensaiadas em `.claude/rules/`) sob demanda — assim a filosofia cabe na cabeça,
e o detalhe é consultado quando a jogada acontece.

Isso já é uma decisão de engenharia: **separar o "o quê acreditamos" (sempre presente) do "como
executar em detalhe" (carregado quando necessário)**. É o mesmo princípio de uma boa arquitetura:
núcleo estável e estreito, detalhe sob demanda.

---

## 1.2 O DNA não é um arquivo só: o raiz e as instruções por setor do campo

Uma correção de imagem importante: quando dizemos "o `CLAUDE.md`", estamos falando de **seis
arquivos**, não de um. O raiz carrega a filosofia do clube; cada setor do campo tem, além dela,
**as instruções fixadas no vestiário daquele setor** — um `CLAUDE.md` aninhado na pasta, carregado
automaticamente quando o trabalho toca aquele caminho:

| Arquivo | Setor do campo | Quando entra em campo |
|---|---|---|
| `CLAUDE.md` (raiz, ~180 linhas) | O DNA do clube inteiro | **Sempre** — 100% das sessões |
| `src/CLAUDE.md` (~484 linhas) | O motor: API, motor agentic, ingestão, Job Core | Quando o trabalho toca `src/` |
| `app/ui/CLAUDE.md` (~508 linhas) | A vitrine: interface e webchat | Quando toca `app/ui/` |
| `tests/CLAUDE.md` (~196 linhas) | O centro de treinamento | Quando toca `tests/` |
| `app/yaml/CLAUDE.md` (~59 linhas) | A prancheta de configuração | Quando toca `app/yaml/` |
| `docs/CLAUDE.md` (~27 linhas) | A sala de imprensa | Quando toca `docs/` |

Duas regras de convivência entre eles:

- **Precedência declarada:** os aninhados "complementam o raiz; em sobreposição, **o raiz é a base**".
  A instrução do setor detalha o jogo naquele setor; não reescreve a filosofia do clube.
- **Economia real de contexto:** o `src/CLAUDE.md` sozinho tem mais que o dobro do raiz — e quem está
  trabalhando na UI **nunca paga por ele**. O detalhe pesado de cada setor só entra em campo quando a
  bola chega lá. É a mesma decisão do §1.1 (núcleo estreito + detalhe sob demanda), aplicada uma
  camada acima.

---

## 1.3 As seções do raiz, agrupadas por função tática

O `CLAUDE.md` raiz tem **12 seções numeradas (§1–§12)**, duas seções de abertura não numeradas
("Disciplina de execução" e as "Diretrizes de investigação, resolução de problemas e criação de
soluções") e o bloco prático **"Comandos essenciais"**. Em vez de listá-las cruamente (isso é índice,
não engenharia), vamos agrupá-las pelo **papel que cumprem no time**.

### Grupo A — O caráter do time (aberturas, §1, §2)

São as seções que definem *como o jogador pensa e se posiciona*.

- **As aberturas não numeradas** — "Disciplina de execução" (todo trabalho vira pequenas tarefas
  sequenciais, com lista de TODO mantida no nível de orquestração) e as "Diretrizes de investigação"
  (rigor proporcional à complexidade; separar **fatos comprovados, hipóteses e conclusões**; isolar as
  variáveis antes de mexer; preferir a menor correção robusta).
  - **Problema que evita:** aceitar a primeira explicação plausível e mexer em vários pontos ao mesmo
    tempo sem causa raiz.
  - **Valor:** método antes de movimento.

- **§1 Princípios inegociáveis** — o coração do DNA. Força "evidência, não invenção", a **regra de
  ouro da fonte de verdade real** (proibido afirmar "não existe / vazio" sobre banco, Qdrant, Redis ou
  fila sem tocar o store físico — grep, código, doc e log são proxy, não prova), "leitura direta acima
  de comparação rasa", "qualidade acima de pressa", "escopo mínimo" sem fragilidade. Proíbe gambiarra,
  código legado, fluxo duplicado, reinvenção. E declara os **dois loops estratégicos** (auto-correção
  por log + aperfeiçoamento) como comportamento default, não opcional (ver [Cap. 2](capitulo-02-rules-playbook.md)).
  - **Problema de engenharia que evita:** **alucinação** (a IA "lembrar" de uma API que não existe),
    **conclusão por proxy** ("a tabela-registro está vazia, logo não foi ingerido") e **decisão por
    conveniência**.
  - **Valor que incentiva:** rigor empírico. É a base de tudo.

- **§2 Perfil e postura** — manda o agente agir como engenheiro sênior: ao apontar erro, risco ou
  problema arquitetural, **explicar causa e impacto prático**, de forma direta e proporcional ao
  risco; ao perguntar, ser inequívoco e **trazer a recomendação** quando houver uma clara. E proíbe
  expressamente o estilo **whack-a-mole**: corrigir sintoma isolado sem reconstruir a causa raiz.
  - **Problema que evita:** a pergunta vaga sem posicionamento técnico e a "correção" que só empurra o
    bug de lugar.
  - **Valor:** honestidade técnica com responsabilidade — um sênior que se posiciona vale mais que um
    executor que só obedece.

### Grupo B — O livro de regras do campo (§5, §8, §9, §10)

São os **portões e as provas** — o jogador não pode avançar sem cumpri-los. Aqui mora a maior parte do
valor anti-bug.

- **§5 Portões obrigatórios antes de agir** — **nove portões nomeados**: entrada canônica de
  implementação (tudo passa pela skill/agente `implementar`), reuso, banco/schema, execução
  centralizada de SQL, **proibição absoluta de DDL em runtime** (todo DDL é manual, em janela
  controlada — DDL embutido em código é defeito, não "melhoria"), testes, log, navegação em
  repositório grande e tier de modelo de subagente.
  - **Problema que evita:** o agente improvisar sobre um subsistema que tem contrato rígido (ex.:
    inventar uma coluna de banco, criar um campo YAML que não existe, travar uma tabela de produção
    com um `CREATE INDEX` "inofensivo").
  - **Valor:** conformidade a contrato; falha cedo e explícita em vez de bug silencioso.

- **§8 correlation_id e logs canônicos** — o coração do "raio-X". Todo processo nasce com um
  `correlation_id` único no boundary oficial; **nenhuma camada abaixo cria, deriva ou substitui — só
  propaga**; objeto cacheado/compartilhado nunca congela o id de outra execução; data/hora operacional
  nasce do clock canônico (`America/Sao_Paulo`); e é proibido varrer a pasta `/logs` sem um id em mãos.
  - **Problema que evita:** o **bug invisível em produção** — aquele que você não consegue diagnosticar
    porque o log não conta a história (ou conta a história da execução errada).
  - **Valor:** observabilidade por construção. (Aprofundado no [Cap. 2](capitulo-02-rules-playbook.md).)

- **§9 Observabilidade, resiliência e erros** — todo fluxo, decisão e resultado relevantes gravam log
  canônico no ponto certo; recurso externo (banco, Redis, filas, HTTP) tem retry explícito com backoff
  pelo helper central; dentro de `except`, **`logger.exception`**, nunca `logger.error`.
  - **Problema que evita:** falha silenciosa, fallback indevido e a queda transitória que vira
    incidente por falta de retry.
  - **Valor:** "sucesso ≠ ausência de exceção" — saúde provada, não presumida.

- **§10 Testes e suíte oficial** — cobertura proporcional ao risco; proibido "ajustar código para
  passar teste errado"; proibido mock que recria framework real (LangChain, FastAPI…); e a **trava
  anti-teste-raso**: proibido rotular causa ("ambiente", "sem dado") por inferência, sem esgotar as
  hipóteses na fonte de verdade real.
  - **Problema que evita:** a suíte mentirosa — verde porque foi maquiada, não porque o sistema
    funciona.
  - **Valor:** teste como evidência real, não como teatro.

> 📌 **Onde foi parar a "definição de pronto"?** O contrato anti-falso-verde não vive mais no raiz:
> virou a jogada ensaiada `definicao-de-pronto.md`, carregada por caminho ([Cap. 2](capitulo-02-rules-playbook.md)).
> É o DNA praticando a própria dieta: o raiz emagrece delegando o detalhe ao playbook.

### Grupo C — A arquitetura do jogo (§3, §4, §6, §7)

Definem *como o sistema deve ser construído* — o estilo de jogo posicional.

- **§3 Stack e ambiente** — Python 3.11/FastAPI; bancos, Redis, RabbitMQ, LangChain/LangGraph; 3
  containers na cloud, **sem filesystem persistente**; as quatro partes (API, Worker, Scheduler, UI).
  E a **segregação por `ENVIRONMENT`**: todo nome de fila, cache key, tabela auxiliar ou vector store
  compartilhado entre ambientes carrega o ambiente no identificador — proibido identificador fixo.
  - **Valor:** contexto compartilhado. Ninguém propõe gravar em disco local porque sabe que não
    existe; ninguém mistura dado de dev com produção porque o nome não deixa.

- **§4 Produto, YAML-First e arquitetura agentic** — a plataforma é 100% configurada por YAML;
  **multi-tenant, multi-user e stateless por requisição**: pipeline, agentes e grafo são remontados a
  cada request; memória que precisa sobreviver vai para store persistente (nunca só RAM); cache misto
  (processo + Redis + aquecimento no boot) amortece o custo de montagem sem quebrar o contrato
  stateless.
  - **Problema que evita:** desenho que só funciona com baixa concorrência ou processo único; estado
    que "some" entre requests.
  - **Valor:** arquitetura nascida para alta carga e isolamento entre tenants.

- **§6 Reuso e arquitetura de código** — OO clássico como padrão; **arquitetura hexagonal** (domínio
  isolado da infraestrutura, provedores atrás de ports/adapters); banco sempre pela classe global com
  retry; lock pessimista só como última opção justificada; **anti-refatoração cosmética** e
  **anti-drift** (o código lido é a verdade; doc externa é referencial).
  - **Problema que evita:** god class, duplicação, acoplamento a SDK concreto (trocar de provider
    exigindo mexer em regra de negócio).
  - **Valor:** SOLID, baixo acoplamento, alta coesão — como regra operante, não slogan.

- **§7 YAML estratégico e AST agentic** — YAML é contrato, tratado com o rigor do código; proibido
  `.get()` para mascarar chave obrigatória ausente (chave obrigatória ausente **falha explícito**);
  a mesma chave vive no mesmo caminho estrutural em todos os YAMLs; cada contexto consome só o
  subconjunto que lhe cabe.
  - **Problema que evita:** configuração que mente (chave que existe mas ninguém usa; ausência
    mascarada por default silencioso).
  - **Valor:** rastreabilidade da configuração.

### Grupo D — A disciplina de bastidores (Comandos essenciais, §11, §12)

Mantêm o vestiário organizado.

- **"Comandos essenciais"** (bloco não numerado) — como rodar API/Worker/Scheduler (`./run.sh`), a
  suíte oficial (`--run-id` obrigatório), lint e type-check. A fonte de cada comando é o próprio repo —
  o DNA aponta, não replica help.
  - **Valor:** operação previsível; ninguém inventa jeito próprio de subir o sistema.

- **§11 Arquivos temporários e scripts shell** — todo artefato temporário vai para `./.sandbox/tmp/`,
  nunca em `/tmp` ou na raiz. *(Este é, inclusive, vigiado pelo árbitro: o `bash-guard` avisa quando
  alguém grava em `/tmp` — ver [Cap. 5](capitulo-05-hooks-arbitragem.md).)*
  - **Valor:** higiene; o repositório não vira lixeira.

- **§12 Qualidade de texto (anti-linguiça)** — todo texto (documentação, comentário e até resposta ao
  usuário) é medido por **densidade de valor por linha**, com o contrapeso anti-lacuna: conteúdo
  necessário entra mesmo longo, redundante sai mesmo curto. Aponta a jogada `qualidade-texto.md`.
  - **Problema que evita:** resposta inflada que esconde o essencial — e doc rasa que omite o que o
    leitor precisa.
  - **Valor:** comunicação auditável.

---

## 1.4 O detalhe de engenharia mais subestimado: "100%"

Há uma definição que parece banal e é, na verdade, uma das mais importantes:

> **"Implementação 100%"** = executar o escopo pedido com qualidade total, **não** ampliar escopo.

Por que isso é engenharia de alto nível? Porque o erro clássico de um executor zeloso (humano ou IA) é
ouvir "faça 100%" e **inflar o escopo**: refatorar o subsistema inteiro, adicionar abstração para um
futuro hipotético, "já que estou aqui, melhoro tudo". Isso gera over-engineering, risco e dívida.

O `CLAUDE.md` corta isso na raiz: o número **qualifica a execução** (faça o recorte certo com
excelência), não **autoriza o escopo** (não invente trabalho novo). É a diferença entre um atacante
que finaliza a jogada combinada e um que tenta driblar o time inteiro toda vez.

> 🧑‍💼 **RESUMO EXECUTIVO.** Esta regra é controle de custo puro. Ela impede que cada tarefa pequena
> vire um projeto grande "por iniciativa". Previsibilidade de prazo e orçamento começa aqui.

---

## 1.5 Como o DNA chega ao jogador (e por que isso é barato)

O `CLAUDE.md` raiz é deliberadamente **enxuto e referencial**: ele afirma o princípio e aponta o
contrato profundo (`.claude/rules/...`) para quando a jogada exigir. Os `CLAUDE.md` de setor fazem o
mesmo uma camada abaixo: só entram quando a bola chega no setor. Isso é uma decisão de *economia de
contexto*: a filosofia inteira está sempre presente sem custar muito; o detalhe pesado (centenas de
linhas de contrato de log, por exemplo) só é carregado quando o trabalho toca aquele tema.

É o mesmo princípio de "carregamento sob demanda" que usamos no código. A configuração pratica o que
prega.

---

## 1.6 O que levar desta posição para a aula

- O `CLAUDE.md` é o **DNA** — não um jogador, mas o que faz o time jogar junto. E são **6 arquivos**:
  o raiz (sempre presente) + 5 instruções por setor do campo, com o **raiz como base** em qualquer
  sobreposição.
- Ele converte **cultura de engenharia em lei aplicada**, presente em toda sessão.
- Seus cinco eixos mais valiosos: **evidência > suposição**, **rastreabilidade**, **anti-falso-verde**,
  **reuso > criar**, **escopo mínimo com qualidade máxima**.
- A genialidade está em ser **curto e referencial**: filosofia sempre presente, detalhe sob demanda —
  em duas camadas (rules por tema; CLAUDE.md por setor).

**Próximo:** [Capítulo 2 — As Jogadas Ensaiadas (`.claude/rules/`)](capitulo-02-rules-playbook.md), onde
os contratos profundos que o DNA aponta ganham detalhe.
