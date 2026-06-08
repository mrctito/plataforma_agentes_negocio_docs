# Capítulo 1 — A Filosofia do Clube (`CLAUDE.md`)

> **Posição em campo:** o `CLAUDE.md` não é um jogador. Ele é o **DNA do clube** — a filosofia de
> jogo que todo jogador carrega ao entrar em campo. Ele é lido em **100% das sessões**, antes de
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

## 1.2 As seções, agrupadas por função tática

O `CLAUDE.md` tem 17 seções. Em vez de listá-las cruamente (isso é índice, não engenharia), vamos
agrupá-las pelo **papel que cumprem no time**.

### Grupo A — O caráter do time (§1, §2, §16)

São as seções que definem *como o jogador pensa e se comunica*.

- **§1 Princípios inegociáveis** — o coração do DNA. Força "evidência, não invenção", "leitura direta
  acima de comparação rasa", "qualidade acima de pressa", "escopo mínimo". Proíbe gambiarra, código
  legado, fluxo duplicado, reinvenção.
  - **O que força:** que toda afirmação sobre o sistema seja provada lendo o código — comentário e
    doc são "pista, nunca prova; código executável vence sempre".
  - **Problema de engenharia que evita:** **alucinação** (a IA "lembrar" de uma API que não existe) e
    **decisão por conveniência** (escolher o caminho fácil que piora o sistema).
  - **Valor que incentiva:** rigor empírico. É a base de tudo.

- **§2 Perfil e postura** — manda o agente agir como engenheiro sênior e, crucialmente, ter **postura
  crítica**: "não concorde com o usuário só para agradar (…) contrarie de forma direta".
  - **Problema que evita:** o "sim, senhor" da IA — concordar com um pedido ruim e produzir uma
    solução tecnicamente inferior só para não criar atrito.
  - **Valor:** honestidade técnica. Um sênior que discorda quando precisa vale mais que um júnior que
    obedece sempre.

- **§16 Comunicação** — tudo em Português-BR, nível 101 (do básico, sem jargão não explicado), com
  vantagens/desvantagens e recomendação objetiva. Define até um "modo de espera" obrigatório.
  - **Problema que evita:** resposta opaca que o time não consegue auditar; falsa sensação de
    conclusão quando algo ainda está rodando.
  - **Valor:** clareza e auditabilidade da comunicação.

### Grupo B — O livro de regras do campo (§5, §10, §11, §14)

São os **portões** — o jogador não pode avançar sem cumpri-los. Aqui mora a maior parte do valor
anti-bug.

- **§5 Portões obrigatórios antes de agir** — antes de tocar banco, YAML, LangChain, testes ou logs,
  é obrigatório ler o contrato específico. É o "antes de cobrar o pênalti, lembre a regra".
  - **Problema que evita:** o agente improvisar sobre um subsistema que tem contrato rígido (ex.:
    inventar uma coluna de banco, criar um campo YAML que não existe).
  - **Valor:** conformidade a contrato; falha cedo e explícita em vez de bug silencioso.

- **§9/§10 correlation_id, logs e observabilidade** — o coração do "raio-X". Todo processo nasce com um
  `correlation_id` único no boundary oficial; toda camada abaixo só propaga, nunca cria. Logs usam
  campos canônicos e builders oficiais.
  - **Problema que evita:** o **bug invisível em produção** — aquele que você não consegue diagnosticar
    porque o log não conta a história. Evita também logger paralelo e dado sensível em log.
  - **Valor:** observabilidade por construção. (Aprofundado no [Cap. 2](capitulo-02-rules-playbook.md).)

- **§11 Testes e suíte oficial** — cobertura proporcional ao risco; proibido "ajustar código para
  passar teste errado"; proibido mock que recria framework real.
  - **Problema que evita:** a suíte mentirosa — verde porque foi maquiada, não porque o sistema
    funciona.
  - **Valor:** teste como evidência real, não como teatro.

- **§14 Definição de pronto** — o portão de saída. "A entrega só vale quando o comportamento-alvo está
  ativo no caminho oficial de runtime." Inclui o **check discriminante** e o **teste de regressão
  estrutural**.
  - **Problema que evita:** **falso-verde** — o inimigo nº 1. Classe criada mas não ligada; schema
    criado mas não consumido; doc atualizada antes da prova.
  - **Valor:** honestidade de entrega. (Aprofundado no [Cap. 2](capitulo-02-rules-playbook.md).)

### Grupo C — A arquitetura do jogo (§3, §4, §6, §7, §8)

Definem *como o sistema deve ser construído* — o estilo de jogo posicional.

- **§3 Stack e ambiente** — Python 3.11/FastAPI; bancos, Redis, RabbitMQ, LangChain/LangGraph; 3
  containers na cloud, **sem filesystem persistente**. Define as quatro partes (API, Worker,
  Scheduler, UI).
  - **Valor:** contexto compartilhado. Ninguém propõe gravar em disco local porque sabe que não existe.

- **§4 Produto, YAML-First e arquitetura agentic** — a plataforma é 100% configurada por YAML; há
  **três espinhas dorsais isoladas que nunca se misturam** (DeepAgent, Workflow, ETL); o pipeline de
  ingestão de PDF tem um orquestrador neutro.
  - **Problema que evita:** mistura de sintaxes/DSLs (um Frankenstein de YAML), e orquestrador que
    "conhece" regra por nome de engine (acoplamento que trava a evolução).
  - **Valor:** isolamento de responsabilidades no nível do produto.

- **§6 Reuso e arquitetura de código** — OO clássico como padrão; limites de tamanho de classe (>500
  atenção, >800 avaliação, >1000 exceção); arquitetura hexagonal (domínio isolado da infraestrutura);
  banco sempre pela classe global com retry.
  - **Problema que evita:** god class, duplicação, acoplamento a SDK concreto (trocar de provider
    exigindo mexer em regra de negócio).
  - **Valor:** SOLID, baixo acoplamento, alta coesão — não como slogan, como regra com números.

- **§7 YAML estratégico e AST agentic** — YAML é contrato, tratado com o rigor do código; proibido
  `.get()` para mascarar chave obrigatória ausente; no escopo agentic, a **AST é a fonte única de
  verdade** e o YAML é artefato compilado.
  - **Problema que evita:** configuração que mente (chave que existe mas ninguém usa; ausência
    mascarada por default silencioso).
  - **Valor:** rastreabilidade da configuração.

- **§8 Tools são a base dos agentes** — tools nascem no código com `@tool`; o catálogo é reconstruído
  por um builder; `tools_library` deve chegar **vazia** nos YAMLs de endpoint, senão o sistema
  **falha fechado**.
  - **Problema que evita:** injeção de tool não autorizada; catálogo manipulado por caminho paralelo.
  - **Valor:** segurança e fonte única de verdade do catálogo.

### Grupo D — A disciplina de bastidores (§12, §13, §15, §17)

Mantêm o vestiário organizado.

- **§12 Temporários e scripts shell** — todo artefato temporário vai para `./.sandbox/tmp/`, nunca em
  `/tmp` ou na raiz. *(Este é, inclusive, aplicado por um hook — ver [Cap. 5](capitulo-05-hooks-arbitragem.md).)*
  - **Valor:** higiene; o repositório não vira lixeira.

- **§13 Execução, planejamento e subagentes** — entra em modo de planejamento para tarefa de mais de
  um passo; usa subagentes para pesquisa paralela; **não para com pendência não bloqueada** ("ao
  concluir uma tarefa não bloqueada, inicie a próxima sem pedir nova autorização").
  - **Problema que evita:** a IA que faz metade e pergunta "quer que eu continue?" a cada passo
    (desperdício), e a IA que para no meio deixando o trabalho pela metade.
  - **Valor:** autonomia disciplinada.

- **§15 Aprendizado e auditoria de instruções** — lê `lessons.md` antes de tarefa relevante; promove
  lição transversal ao fechar; registra instrução ruim em `bad-instructions.md`.
  - **Problema que evita:** o time repetir o mesmo erro em slices diferentes; a configuração apodrecer
    com contradições.
  - **Valor:** melhoria contínua institucionalizada (o time aprende, não só o indivíduo).

- **§17 Documentação e execução local** — doc em `/docs` nível 101; **documentação só depois da prova
  executável** ("nunca antecipa maturidade que o runtime ainda não entrega").
  - **Problema que evita:** doc que descreve um futuro que o código ainda não tem.
  - **Valor:** documentação como retrato fiel do que existe.

---

## 1.3 O detalhe de engenharia mais subestimado: "100%"

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

## 1.4 Como o DNA chega ao jogador (e por que isso é barato)

O `CLAUDE.md` é deliberadamente **enxuto e referencial**: ele afirma o princípio e aponta o contrato
profundo (`.claude/rules/...`) para quando a jogada exigir. Isso é uma decisão de *economia de
contexto*: a filosofia inteira está sempre presente sem custar muito; o detalhe pesado (centenas de
linhas de contrato de log, por exemplo) só é carregado quando o trabalho toca aquele tema.

É o mesmo princípio de "carregamento sob demanda" que usamos no código. A configuração pratica o que
prega.

---

## 1.5 O que levar desta posição para a aula

- O `CLAUDE.md` é o **DNA** — não um jogador, mas o que faz o time jogar junto.
- Ele converte **cultura de engenharia em lei aplicada**, presente em toda sessão.
- Seus cinco eixos mais valiosos: **evidência > suposição**, **rastreabilidade**, **anti-falso-verde**,
  **reuso > criar**, **escopo mínimo com qualidade máxima**.
- A genialidade está em ser **curto e referencial**: filosofia sempre presente, detalhe sob demanda.

**Próximo:** [Capítulo 2 — As Jogadas Ensaiadas (`.claude/rules/`)](capitulo-02-rules-playbook.md), onde
os contratos profundos que o DNA aponta ganham detalhe.
</content>
