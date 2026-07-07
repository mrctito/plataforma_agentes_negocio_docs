# Capítulo 1 — A Norma da Fábrica (`CLAUDE.md`)

> 🏭 [↩ Doc-mãe da lente](README.md) · [↩ Índice da metodologia](../README.md)

> **Parte da fábrica:** o `CLAUDE.md` não é uma estação. Ele é a **Norma da Fábrica** — o Manual de
> Qualidade mestre que todo operador lê antes de assumir o posto. É carregado em **100% das sessões**,
> antes de qualquer trabalho. Se um agente é uma estação e uma skill é a ordem de produção, o
> `CLAUDE.md` é o que faz a linha inteira produzir do mesmo jeito, em qualquer turno, sem combinar.

> 🧑‍💼 **RESUMO EXECUTIVO.** Este arquivo é a nossa "norma ISO de engenharia". Ele transforma
> preferências dispersas ("seria bom testar", "tenta não duplicar") em **especificação obrigatória,
> aplicada automaticamente em toda sessão**. O valor de negócio: a qualidade deixa de depender do humor
> ou da pressa do dia e vira um **piso garantido**. É o que permite escalar a produção sem escalar o
> refugo.

---

## 1.1 Por que existe um padrão lido sempre

Uma fábrica sem norma comum vira um amontoado de artesãos, cada um com seu critério. Cada sessão de IA,
sem um padrão, recomeçaria do zero. O `CLAUDE.md` resolve isso sendo **carregado em toda sessão** e tendo
precedência sobre o comportamento padrão. Ele é curto de propósito e **aponta para os procedimentos
detalhados** (os SOPs em `.claude/rules/`) sob demanda — assim o padrão cabe na cabeça, e a instrução de
trabalho específica é consultada quando a operação acontece.

Isso já é decisão de engenharia: **separar o "o que acreditamos" (sempre presente) do "como executar em
detalhe" (carregado quando necessário)**. É o mesmo princípio de uma boa arquitetura: núcleo estável e
estreito, detalhe sob demanda — e o mesmo princípio de uma fábrica enxuta: a norma na parede, a
instrução de trabalho na estação.

---

## 1.2 A Norma não é um documento único: 6 arquivos, um por área da planta

Detalhe que muda a leitura de tudo: a Norma são **6 arquivos `CLAUDE.md`** — o **raiz** (lido em 100%
das sessões) e **5 aninhados por pasta** (`src/`, `tests/`, `app/yaml/`, `app/ui/`, `docs/`), carregados
**por caminho**: quando o trabalho entra numa dessas áreas, a instrução daquela área entra junto. Na
fábrica, é a **norma corporativa na parede** mais a **instrução de trabalho afixada em cada área da
planta**: o soldador não carrega o manual da pintura.

A precedência é declarada nos próprios arquivos: os aninhados **complementam o raiz; em sobreposição, o
raiz é a base**. Não há duas normas disputando — há um padrão e seus detalhamentos por área.

## 1.3 As seções da Norma raiz, agrupadas por função na fábrica

O `CLAUDE.md` raiz tem **12 seções numeradas (§1–§12)**, duas seções de abertura não numeradas
("Disciplina de execução" e "Diretrizes de investigação") e o bloco "Comandos essenciais". Em vez de
listá-las cruamente (isso é índice, não engenharia), vamos agrupá-las pelo **papel que cumprem na
produção**.

### Grupo A — A cultura de qualidade (aberturas, §1, §2, §12)

Definem *como o operador pensa e comunica*.

- **Aberturas não numeradas** — "Disciplina de execução" (todo trabalho registrado em pequenas tarefas
  sequenciais, com lista de TODO no nível de orquestração) e "Diretrizes de investigação" (rigor
  proporcional à complexidade; separar fato, hipótese e conclusão; isolar variáveis antes de mexer;
  preferir a menor correção robusta).
  - **Problema que evita:** a correção por palpite — mexer em três coisas ao mesmo tempo sem causa raiz.
  - **Valor:** método antes de movimento.

- **§1 Princípios inegociáveis** — o coração da norma. Força "evidência, não invenção", "leitura direta
  acima de comparação rasa", "qualidade acima de pressa", "escopo mínimo". Proíbe gambiarra, código
  legado, fluxo duplicado, reinvenção. Inclui a **regra de ouro da fonte de verdade real**: afirmação
  sobre estado de recurso (banco, Qdrant, Redis, fila) só vale depois de **tocar o store físico** —
  proibido concluir por proxy (grep, código, doc, log). E declara os **dois loops estratégicos**
  (correção por log e aperfeiçoamento) como comportamento default, não opção.
  - **O que força:** que toda afirmação sobre o sistema seja **medida** — comentário e doc são "pista,
    nunca prova; código executável vence sempre".
  - **Problema de engenharia que evita:** **alucinação** (a IA "lembrar" de uma peça que não existe) e
    **decisão por conveniência** (escolher o caminho fácil que piora o produto).
  - **Valor:** rigor empírico. É a base de tudo.

- **§2 Perfil e postura** — manda agir como engenheiro sênior: apontar erro/risco com causa e impacto
  práticos, perguntar com recomendação clara em vez de pergunta vaga, e **proíbe expressamente o
  whack-a-mole** (corrigir sintoma isolado sem causa raiz nem mapa de impacto).
  - **Problema que evita:** aceitar especificação ruim sem crítica e "consertos" que só empurram o
    defeito de lugar.
  - **Valor:** honestidade técnica. Um inspetor que reprova quando precisa vale mais que um que aprova
    sempre.

- **§12 Qualidade de texto (anti-linguiça e anti-lacuna)** — todo texto (inclusive resposta ao usuário)
  é medido por **densidade de valor por linha**: proibido inflar com paráfrase, e igualmente proibido
  omitir o que o leitor precisa.
  - **Problema que evita:** relatório opaco que ninguém audita — por excesso ou por falta.
  - **Valor:** clareza e auditabilidade.

### Grupo B — Os pontos de controle de qualidade (§5, §8, §9, §10)

São os **portões de inspeção** — a peça não avança sem passar. Aqui mora a maior parte do valor
anti-refugo.

- **§5 Portões obrigatórios antes de agir** — nove portões nomeados: entrada canônica pela estação
  `implementar`, reuso, banco/schema, execução centralizada de SQL, **proibição absoluta de DDL em
  runtime** (todo DDL é manual, em janela controlada), testes, log, navegação em repositório grande e
  **tier de modelo de subagente**. É o "antes de operar a máquina, leia a instrução de trabalho".
  - **Problema que evita:** improvisar sobre um subsistema com contrato rígido (inventar uma coluna de
    banco, criar um campo YAML que não existe, travar a tabela com um `ALTER` embutido no boot).
  - **Valor:** conformidade a contrato; falha cedo e explícita em vez de defeito silencioso.

- **§8 correlation_id e logs canônicos** — o coração do registro de processo. Todo processo nasce com um
  `correlation_id` único no boundary oficial; **nenhuma camada abaixo cria, deriva ou substitui — só
  propaga**; objeto cacheado/compartilhado nunca "congela" o número de série de outra execução; proibido
  varrer a pasta `/logs` sem um `correlation_id` em mãos.
  - **Problema que evita:** o **defeito invisível que só aparece no cliente** — aquele que você não
    diagnostica porque o registro de processo não conta a história.
  - **Valor:** rastreabilidade por construção. *(Aprofundado no [Cap. 2](capitulo-02-especificacoes-processo.md).)*

- **§9 Observabilidade, resiliência e erros** — todo fluxo, decisão e resultado relevantes gravam log
  canônico; recurso externo tem retry explícito com backoff; dentro de `except`, `logger.exception`
  (nunca `logger.error`). E a máxima: **"sucesso ≠ ausência de exceção"** — procurar também a falha
  silenciosa e o fallback indevido.
  - **Problema que evita:** logger paralelo, dado sensível em log e o "verde" que esconde degradação.
  - **Valor:** o registro conta a história inteira, inclusive a das quase-falhas.

- **§10 Testes e suíte oficial** — cobertura proporcional ao risco; implementação nova exige teste
  unitário; refatoração exige proteção **antes** da mudança; proibido "ajustar código para passar teste
  errado"; proibido mock que recria framework real.
  - **Problema que evita:** o ensaio mentiroso — verde porque foi maquiado, não porque a peça funciona.
  - **Valor:** ensaio como evidência real, não teatro. *(O portão de saída anti-falso-verde — a
    "definição de pronto" — saiu da Norma raiz e virou SOP dedicado: ver [Cap. 2](capitulo-02-especificacoes-processo.md).)*

### Grupo C — A engenharia do produto (§3, §4, §6, §7)

Definem *como o sistema deve ser construído* — o projeto da peça.

- **§3 Stack e ambiente** — Python 3.11/FastAPI; PostgreSQL/MySQL/MSSQL/Azure Search/Qdrant, Redis,
  RabbitMQ, LangChain/LangGraph; 3 containers na cloud, **sem filesystem persistente**. Define as quatro
  partes (API, Worker, Scheduler, UI) e a **segregação por ambiente**: todo recurso compartilhado
  (fila, chave, tabela auxiliar, cache) embute a variável `ENVIRONMENT` no nome.
  - **Valor:** contexto compartilhado. Ninguém propõe gravar em disco local porque sabe que a planta não
    tem esse depósito — e dev nunca pisa no dado de produção.

- **§4 Produto, YAML-First e arquitetura agentic** — a plataforma é 100% configurada por YAML,
  multi-tenant e multi-user; o runtime oficial é **stateless por requisição** (tudo que precisa
  sobreviver entre requests vive em store persistente, nunca só em RAM); um **cache misto**
  (processo + Redis + aquecimento no boot) amortece a remontagem sem quebrar o contrato stateless.
  - **Problema que evita:** desenho que só funciona com baixa concorrência, processo único ou memória
    local "que sempre esteve lá".
  - **Valor:** a peça nasce pronta para alta carga e paralelismo real.

- **§6 Reuso e arquitetura de código** — OO clássico como padrão; **arquitetura hexagonal** (domínio
  isolado da infraestrutura, infra entra por adapters); banco sempre pela classe global com retry;
  controle otimista antes de lock pessimista; anti-refatoração cosmética e anti-drift.
  - **Problema que evita:** peça gigante (god class), duplicação, acoplamento a SDK concreto (trocar de
    fornecedor exigindo mexer na regra de negócio).
  - **Valor:** SOLID, baixo acoplamento, alta coesão.

- **§7 YAML estratégico e AST agentic** — YAML é contrato, tratado com o rigor do código; proibido
  `.get()` para mascarar chave obrigatória ausente; chave não vira diretiva universal só por existir
  (cada contexto consome seu subconjunto); a mesma chave vive no mesmo caminho estrutural em todos os
  YAMLs.
  - **Problema que evita:** configuração que mente (chave que existe mas ninguém usa; ausência mascarada
    por default silencioso).
  - **Valor:** rastreabilidade da configuração.

### Grupo D — A disciplina de chão de fábrica ("Comandos essenciais", §11)

Mantêm a planta organizada e operável.

- **"Comandos essenciais"** — como ligar cada parte da planta (`./run.sh +a/+w/+s` para API, Worker e
  Scheduler), o entrypoint único da suíte oficial (`python suite_de_testes_padrao.py <modo> --run-id`),
  lint e type-check (`ruff`, `mypy`, `black`).
  - **Valor:** qualquer operador liga a máquina certa sem adivinhar o botão.

- **§11 Arquivos temporários e scripts shell** — todo artefato temporário vai para `./.sandbox/tmp/`,
  nunca em `/tmp` ou na raiz; scripts `.sh` pequenos, coesos e com bit de execução. Inclui o validador
  de descrições de subagentes (`./validar_descricao_subagentes.sh`).
  - **Valor:** higiene; a planta não vira depósito de sucata. *(Aplicado também por um dispositivo — ver
    [Cap. 5](capitulo-05-pokayoke-e-andon.md).)*

---

## 1.4 O detalhe de engenharia mais subestimado: "100%"

Há uma definição que parece banal e é uma das mais importantes:

> **"Implementação 100%"** = executar o escopo pedido com qualidade total, **não** ampliar escopo.

Por que isso é engenharia de alto nível? Porque o erro clássico de um operador zeloso (humano ou IA) é
ouvir "faça 100%" e **inflar o lote**: reprojetar o subsistema inteiro, adicionar peças para um futuro
hipotético, "já que a máquina está ligada, melhoro tudo". Isso gera over-engineering, risco e estoque
parado de código.

A norma corta isso na raiz: o número **qualifica a execução** (faça o lote certo com excelência), não
**autoriza o escopo** (não invente encomenda nova). É a diferença entre produzir a peça especificada com
perfeição e refazer a planta toda a cada pedido.

> 🧑‍💼 **RESUMO EXECUTIVO.** Esta regra é controle de custo puro. Impede que cada tarefa pequena vire um
> projeto grande "por iniciativa". Previsibilidade de prazo e orçamento começa aqui.

---

## 1.5 Como o padrão chega à estação (e por que é barato)

O `CLAUDE.md` é deliberadamente **enxuto e referencial**: afirma o princípio e **aponta** o procedimento
detalhado (`.claude/rules/...`) para quando a operação exigir. É *economia de contexto*: o padrão inteiro
está sempre presente sem pesar; a instrução densa (centenas de linhas do contrato de log, por exemplo) só
entra **quando o trabalho toca aquele tema**. É o mesmo "carregamento sob demanda" do bom software,
aplicado à própria configuração. A norma pratica o que prega.

---

## 1.6 O que levar deste capítulo

- O `CLAUDE.md` é a **Norma da Fábrica** — não uma estação, mas o que faz a linha produzir junto.
- A Norma são **6 arquivos**: o raiz (sempre presente) + 5 por área da planta, carregados por caminho;
  em sobreposição, **o raiz é a base**.
- Converte **cultura de engenharia em especificação aplicada**, presente em toda sessão.
- Seus cinco eixos mais valiosos: **evidência > suposição**, **rastreabilidade de lote**,
  **anti-falso-verde**, **reuso > fabricar**, **escopo mínimo com qualidade máxima**.
- A genialidade está em ser **curto e referencial**: padrão sempre presente, instrução de trabalho sob
  demanda.

**Próximo:** [Capítulo 2 — As Especificações de Processo (`.claude/rules/`)](capitulo-02-especificacoes-processo.md).
