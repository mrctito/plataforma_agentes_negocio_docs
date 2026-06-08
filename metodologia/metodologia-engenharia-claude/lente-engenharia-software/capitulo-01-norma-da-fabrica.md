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

## 1.2 As seções, agrupadas por função na fábrica

O `CLAUDE.md` tem 17 seções. Em vez de listá-las cruamente (isso é índice, não engenharia), vamos
agrupá-las pelo **papel que cumprem na produção**.

### Grupo A — A cultura de qualidade (§1, §2, §16)

Definem *como o operador pensa e comunica*.

- **§1 Princípios inegociáveis** — o coração da norma. Força "evidência, não invenção", "leitura direta
  acima de comparação rasa", "qualidade acima de pressa", "escopo mínimo". Proíbe gambiarra, código
  legado, fluxo duplicado, reinvenção.
  - **O que força:** que toda afirmação sobre o sistema seja **medida** lendo o código — comentário e
    doc são "pista, nunca prova; código executável vence sempre".
  - **Problema de engenharia que evita:** **alucinação** (a IA "lembrar" de uma peça que não existe) e
    **decisão por conveniência** (escolher o caminho fácil que piora o produto).
  - **Valor:** rigor empírico. É a base de tudo.

- **§2 Perfil e postura** — manda agir como engenheiro sênior e ter **postura crítica**: "não concorde
  com o usuário só para agradar (…) contrarie de forma direta".
  - **Problema que evita:** o "sim, senhor" da IA — aceitar uma especificação ruim e produzir uma peça
    tecnicamente inferior só para não criar atrito.
  - **Valor:** honestidade técnica. Um inspetor que reprova quando precisa vale mais que um que aprova
    sempre.

- **§16 Comunicação** — tudo em Português-BR, nível 101, com prós/contras e recomendação objetiva.
  Define até um "modo de espera" obrigatório.
  - **Problema que evita:** relatório opaco que ninguém audita; falsa sensação de conclusão quando algo
    ainda está rodando.
  - **Valor:** clareza e auditabilidade.

### Grupo B — Os pontos de controle de qualidade (§5, §10, §11, §14)

São os **portões de inspeção** — a peça não avança sem passar. Aqui mora a maior parte do valor
anti-refugo.

- **§5 Portões obrigatórios antes de agir** — antes de tocar banco, YAML, LangChain, testes ou logs, é
  obrigatório ler o procedimento específico. É o "antes de operar a máquina, leia a instrução de
  trabalho".
  - **Problema que evita:** improvisar sobre um subsistema com contrato rígido (inventar uma coluna de
    banco, criar um campo YAML que não existe).
  - **Valor:** conformidade a contrato; falha cedo e explícita em vez de defeito silencioso.

- **§9/§10 correlation_id, logs e observabilidade** — o coração do registro de processo. Todo processo
  nasce com um `correlation_id` único no boundary oficial; toda camada abaixo só propaga. Logs usam
  campos canônicos e builders oficiais.
  - **Problema que evita:** o **defeito invisível que só aparece no cliente** — aquele que você não
    diagnostica porque o registro de processo não conta a história. Evita também logger paralelo e dado
    sensível em log.
  - **Valor:** rastreabilidade por construção. *(Aprofundado no [Cap. 2](capitulo-02-especificacoes-processo.md).)*

- **§11 Testes e suíte oficial** — cobertura proporcional ao risco; proibido "ajustar código para passar
  teste errado"; proibido mock que recria framework real.
  - **Problema que evita:** o ensaio mentiroso — verde porque foi maquiado, não porque a peça funciona.
  - **Valor:** ensaio como evidência real, não teatro.

- **§14 Definição de pronto** — o portão de saída. "A entrega só vale quando o comportamento-alvo está
  ativo no caminho oficial de runtime." Inclui o **check discriminante** e o **teste de regressão
  estrutural**.
  - **Problema que evita:** **falso-verde** — o refugo nº 1. Peça fabricada mas não instalada; schema
    criado mas não consumido; doc atualizada antes da prova.
  - **Valor:** honestidade de entrega. *(Aprofundado no [Cap. 2](capitulo-02-especificacoes-processo.md).)*

### Grupo C — A engenharia do produto (§3, §4, §6, §7, §8)

Definem *como o sistema deve ser construído* — o projeto da peça.

- **§3 Stack e ambiente** — Python 3.11/FastAPI; bancos, Redis, RabbitMQ, LangChain/LangGraph; 3
  containers na cloud, **sem filesystem persistente**. Define as quatro partes (API, Worker, Scheduler,
  UI).
  - **Valor:** contexto compartilhado. Ninguém propõe gravar em disco local porque sabe que a planta não
    tem esse depósito.

- **§4 Produto, YAML-First e arquitetura agentic** — a plataforma é 100% configurada por YAML; há **três
  linhas de montagem isoladas que nunca se misturam** (DeepAgent, Workflow, ETL); o pipeline de ingestão
  de PDF tem um orquestrador neutro.
  - **Problema que evita:** mistura de sintaxes/DSLs (um Frankenstein de YAML), e orquestrador que
    "conhece" regra por nome de engine (acoplamento que trava a evolução).
  - **Valor:** isolamento de responsabilidades no nível do produto.

- **§6 Reuso e arquitetura de código** — OO clássico como padrão; limites de tamanho de classe (>500
  atenção, >800 avaliação, >1000 exceção); arquitetura hexagonal (domínio isolado da infraestrutura);
  banco sempre pela classe global com retry.
  - **Problema que evita:** peça gigante (god class), duplicação, acoplamento a SDK concreto (trocar de
    fornecedor exigindo mexer na regra de negócio).
  - **Valor:** SOLID, baixo acoplamento, alta coesão — com números, não slogan.

- **§7 YAML estratégico e AST agentic** — YAML é contrato, tratado com o rigor do código; proibido
  `.get()` para mascarar chave obrigatória ausente; no escopo agentic, a **AST é a fonte única de verdade**
  e o YAML é artefato compilado.
  - **Problema que evita:** configuração que mente (chave que existe mas ninguém usa; ausência mascarada
    por default silencioso).
  - **Valor:** rastreabilidade da configuração.

- **§8 Tools são a base dos agentes** — tools nascem no código com `@tool`; o catálogo é reconstruído por
  um builder; `tools_library` deve chegar **vazia** nos YAMLs de endpoint, senão o sistema **falha
  fechado**.
  - **Problema que evita:** injeção de tool não autorizada; catálogo manipulado por caminho paralelo.
  - **Valor:** segurança e fonte única de verdade do catálogo.

### Grupo D — A disciplina de chão de fábrica (§12, §13, §15, §17)

Mantêm a planta organizada.

- **§12 Temporários e scripts shell** — todo artefato temporário vai para `./.sandbox/tmp/`, nunca em
  `/tmp` ou na raiz. *(Aplicado também por um dispositivo — ver [Cap. 5](capitulo-05-pokayoke-e-andon.md).)*
  - **Valor:** higiene; a planta não vira depósito de sucata.

- **§13 Execução, planejamento e subagentes** — entra em modo de planejamento para tarefa de mais de um
  passo; usa subagentes para trabalho paralelo; **não para com pendência não bloqueada**.
  - **Problema que evita:** a IA que faz metade e pergunta "quer que eu continue?" a cada passo
    (desperdício), e a que para no meio deixando a peça pela metade.
  - **Valor:** autonomia disciplinada.

- **§15 Aprendizado e auditoria de instruções** — lê `lessons.md` antes de tarefa relevante; promove
  lição transversal ao fechar; registra instrução ruim em `bad-instructions.md`.
  - **Problema que evita:** repetir o mesmo defeito em lotes diferentes; a norma apodrecer com
    contradições.
  - **Valor:** **Kaizen** institucionalizado (a fábrica aprende, não só o operador).

- **§17 Documentação e execução local** — doc nível 101; **documentação só depois da prova executável**
  ("nunca antecipa maturidade que o runtime ainda não entrega").
  - **Problema que evita:** doc que descreve um produto que a linha ainda não fabrica.
  - **Valor:** documentação como retrato fiel do que existe.

---

## 1.3 O detalhe de engenharia mais subestimado: "100%"

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

## 1.4 Como o padrão chega à estação (e por que é barato)

O `CLAUDE.md` é deliberadamente **enxuto e referencial**: afirma o princípio e **aponta** o procedimento
detalhado (`.claude/rules/...`) para quando a operação exigir. É *economia de contexto*: o padrão inteiro
está sempre presente sem pesar; a instrução densa (centenas de linhas do contrato de log, por exemplo) só
entra **quando o trabalho toca aquele tema**. É o mesmo "carregamento sob demanda" do bom software,
aplicado à própria configuração. A norma pratica o que prega.

---

## 1.5 O que levar deste capítulo

- O `CLAUDE.md` é a **Norma da Fábrica** — não uma estação, mas o que faz a linha produzir junto.
- Converte **cultura de engenharia em especificação aplicada**, presente em toda sessão.
- Seus cinco eixos mais valiosos: **evidência > suposição**, **rastreabilidade de lote**,
  **anti-falso-verde**, **reuso > fabricar**, **escopo mínimo com qualidade máxima**.
- A genialidade está em ser **curto e referencial**: padrão sempre presente, instrução de trabalho sob
  demanda.

**Próximo:** [Capítulo 2 — As Especificações de Processo (`.claude/rules/`)](capitulo-02-especificacoes-processo.md).
