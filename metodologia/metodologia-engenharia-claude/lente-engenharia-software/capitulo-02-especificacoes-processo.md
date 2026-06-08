# Capítulo 2 — As Especificações de Processo (`.claude/rules/`)

> 🏭 [↩ Doc-mãe da lente](README.md) · [↩ Índice da metodologia](../README.md)

> **Parte da fábrica:** se o `CLAUDE.md` é a Norma da Fábrica, a pasta `.claude/rules/` é o conjunto de
> **Especificações de Processo e Instruções de Trabalho (SOPs)**. Nenhuma delas opera sozinha — são
> **consultadas sob demanda** pelas estações (agentes) no momento exato da operação. A estação de
> soldagem consulta a especificação de solda; o controle de qualidade consulta o procedimento de
> inspeção. Aqui está o conhecimento profundo que seria pesado demais para carregar em todo turno.

> 🧑‍💼 **RESUMO EXECUTIVO.** Estes arquivos são o nosso "conhecimento de processo codificado". Garantem
> que decisões difíceis (como registrar para diagnosticar, como saber se algo está realmente pronto,
> quando reusar em vez de fabricar) sejam tomadas **do mesmo jeito, sempre, por qualquer estação** — em
> vez de depender de quem está no posto. É a diferença entre uma empresa cujo know-how mora na cabeça de
> poucos e uma cujo know-how está escrito e aplicado.

---

## 2.0 Por que separar `rules/` da Norma

Decisão de arquitetura: a Norma (`CLAUDE.md`) precisa ser leve (lida sempre), mas alguns procedimentos
são densos (o de logs tem centenas de linhas). Misturar tudo tornaria cada turno caro e ilegível. A
solução é a de uma fábrica enxuta (e de um bom software): **norma estreita na parede + instruções de
trabalho carregadas na estação, sob demanda.**

As `rules/` dividem-se em dois tipos:

- **Especificações profundas** (instruções de trabalho): a versão completa de um procedimento. Ex.:
  `log-instructions.md`.
- **Registros de melhoria contínua** (o diário de bordo da fábrica): memória viva que muda a cada turno.
  Ex.: `lessons.md`, `error-backlog.md`.

---

## 2.1 As especificações profundas (as instruções de trabalho)

### `log-instructions.md` — a especificação do registro de processo (o raio-X)

A especificação mais valiosa da fábrica. Define que **todo log é registro de processo (batch record)**,
não enfeite — um "raio-X" / inspeção não-destrutiva que permite reconstruir uma produção *depois* que
ela aconteceu, como um debug offline.

**O que força:**
- **`correlation_id` canônico (número de série):** nasce uma única vez no boundary oficial (o endpoint
  inicial, ou o iniciador real em scheduler/job). Nenhuma camada abaixo cria, deriva ou substitui — só
  propaga. A response devolve o mesmo número (corpo + header), com força máxima em erros.
- **Campos canônicos:** fonte única em `src/core/log_canonical_fields.py`; usa **builders oficiais**
  (`build_canonical_log_context`, builders de slice). Proibido inventar campo, alias local
  (`step`/`etapa` no lugar de `stage`) ou montar `extra={"event_name": ...}` na mão.
- **`logger.exception` dentro de `except`:** obrigatório (preserva o stack trace); `logger.error` ali é
  tratado como defeito de rastreabilidade.
- **Anti-varredura de `/logs`:** proibido listar a pasta cegamente (dezenas de milhares de arquivos
  podem travar a sessão). Só se busca com um `correlation_id` em mãos. *(Tão crítica que também é um
  dispositivo — ver [Cap. 5](capitulo-05-pokayoke-e-andon.md).)*

**Problema que evita:** o defeito que você não consegue diagnosticar. Em sistema distribuído (3
containers, workers paralelos, filas), sem rastreabilidade de lote você fica cego. Esta especificação
garante que o registro conte a história inteira: entrada, decisões, delegações, chamadas externas,
paralelismo real, erros e estado final.

> 🧑‍💼 Pense neste arquivo como a "caixa-preta" da produção. Quando algo dá errado no cliente, a
> diferença entre "resolvemos em minutos" e "passamos dias no escuro" mora aqui.

### `reuso-instructions.md` — a especificação do "consulte o almoxarifado antes de fabricar"

Antes de criar qualquer classe, helper, service, adapter, resolver, validator: é **obrigatório**
pesquisar o que já existe, consultar `docs/README-TOOLS-LIB.md`, ler os candidatos, e só então — com
justificativa — fabricar algo novo.

**O que força:** uma ordem de 9 passos que termina em "somente depois disso considerar criar algo novo",
com postura de **advogado do diabo**: "em fábrica madura, peça nova é rara".

**Problema que evita:** duplicação. Refabricar um componente que já existe gera **divergência de
comportamento** (dois jeitos de fazer a mesma coisa, que evoluem diferente), dobra a manutenção e espalha
inconsistência. E um achado-chave: se a busca encontra **duas** soluções para o mesmo problema, isso é
**inconsistência de processo** a unificar — proibido adicionar uma terceira.

**Valor:** DRY, consistência, manutenibilidade (estoque enxuto de código).

### `definicao-de-pronto.md` — o procedimento de inspeção final (anti-falso-verde)

A versão profunda do `CLAUDE.md §14`. Define "pronto" de forma **falsificável**.

**O conceito central — falso-verde:** "quando a entrega parece correta porque ensaios isolados passaram,
a documentação foi atualizada ou uma peça nova foi criada, mas a linha oficial em runtime ainda não usa a
peça nova".

**O que força:**
- **Hierarquia de verdade:** código executável > teste observado > log real > plano > doc > comentário.
- **Check discriminante:** "o check mais barato capaz de desconfirmar a hipótese da entrega" — verifica a
  peça concreta instalada no produto final, não só a existência do componente novo.
- **Teste de regressão estrutural:** todo trabalho estrutural inclui um teste que **falha se a linha
  oficial voltar à peça antiga**.
- **Ordem obrigatória:** provar a instalação → ligar a dependência → teste de regressão → validar runtime
  → **só então** atualizar documentação.

**Problema que evita:** a falsa sensação de conclusão — o golpe mais comum em automação. Tudo parece
verde, mas o produto real não mudou.

### `suite-testes-instructions.md` — o procedimento de ensaios

A referência de como rodar a suíte oficial: qual "gate" (modo) usar para cada situação, sintaxe, flags, e
a regra de ouro **"se este arquivo divergir do código executável, o código vence"**.

**O que força:** escolher o ensaio proporcional (`--focus-tests` para slice pequeno, `--complete` para
fechamento), sempre com `--run-id`, sempre auditando `telemetry.json`/`state.json` — "não declare sucesso
olhando apenas exit code ou terminal".

**Valor:** validação proporcional ao risco, com evidência auditável.

### `python.md` — a especificação técnica do material (Python)

Complemento especializado: 100% tipado (mypy --strict), proibido `Any` fora de boundary, proibido
`try/except ImportError` em teste, proibido stub de framework real (LangChain, FastAPI…), markers de
família obrigatórios em pytest.

**Problema que evita:** tipagem frouxa, dependência opcional mascarada, ensaio que simula o material (e
portanto não ensaia nada real). *(Vários itens viram alarmes automáticos — Cap. 5.)*

### `large-repo-navigation.md` — o procedimento de navegação numa planta enorme

Como operar num repositório massivo sem se perder: começar por um **anchor** concreto (arquivo, símbolo,
log, teste falhando), orçamento de exploração inicial, particionar pedido "repo-wide" em slices, e "busca
localiza, leitura comprova". *(Detalhe na base: [Travas para Projetos Grandes](../base-conceitual/travas-projetos-grandes.md).)*

**Valor:** navegação disciplinada, proporcional ao risco.

### `estrategia-recomendacoes.md` e `fidelidade-pedido-usuario.md` — os contratos de passagem de bastão

Definem como uma estação entrega à próxima sem perder informação:

- **`estrategia-recomendacoes.md`:** define a seção `# Estratégia/Recomendacoes` que o PCP (`planejar`)
  cria e a fabricação (`implementar`) é **obrigada a ler antes de tocar qualquer arquivo**. É a "ordem de
  serviço" do lote: o que preservar, mudar, remover; quais fallbacks são proibidos; o gate de
  não-parada. Vive em arquivo único porque é um contrato **entre duas estações** — se vivesse duplicado,
  as pontas divergiriam.
- **`fidelidade-pedido-usuario.md`:** obriga a reapresentar o pedido do cliente fielmente (sem reduzir a
  um rótulo genérico) e a destacar as `REGRAS E PREMISSAS IMPERATIVAS DO USUARIO`. Garante que uma
  exigência explícita não "evapore" no meio da linha.

> 🛠️ **Detalhe de engenharia elegante:** estes contratos foram extraídos para arquivos próprios
> exatamente para **eliminar duplicação entre estações**. É o princípio DRY aplicado à própria
> configuração — as estações *referenciam* o procedimento, não o redefinem. A fábrica obedece às mesmas
> regras que impõe ao produto.

---

## 2.2 O registro de melhoria contínua (o diário de bordo / Kaizen)

Estes quatro arquivos não são especificações fixas — são **memória que muda a cada turno**. É aqui que a
fábrica "aprende com a produção".

| Arquivo | Papel na fábrica | O que registra |
|---|---|---|
| `lessons.md` | A lição de processo, gravada | Apenas lições com **valor preventivo transversal** — regras que mudam como as estações pensam em mais de um slice |
| `error-backlog.md` | O relatório de não-conformidade | Todo defeito real de produto (com cenário, módulo, evidência) |
| `regression-logs.md` | A reincidência de defeito | Defeito já registrado que **voltou** — tratado como falha grave da ação corretiva |
| `bad-instructions.md` | A auditoria da própria norma | Contradições, ambiguidades e lacunas nas próprias instruções |

**A engenharia por trás:** `lessons.md` tem **curadoria rígida** — não é diário nem backlog. Antes de
promover uma lição, a pergunta obrigatória é: *"se outra estação operar amanhã em outro slice, esta regra
ainda reduziria chance real de defeito?"* Se não, fica na memória local. Isso impede que o registro
central vire um depósito inútil.

**Valor:** **Kaizen** — melhoria contínua **institucional** (o know-how fica na empresa, não na pessoa).
E note a simetria: este diário é alimentado automaticamente pelos dispositivos de início e fim de turno
(ver [Cap. 5](capitulo-05-pokayoke-e-andon.md)) — a fábrica é *lembrada* de aprender.

> 🧑‍💼 **RESUMO EXECUTIVO.** Este é o mecanismo que faz a produção ficar **melhor com o tempo, sozinha**.
> Cada defeito vira uma regra que previne o próximo. É capitalização de conhecimento: o que aprendemos
> numa quarta protege todos os lotes da quinta em diante.

---

## 2.3 O que levar deste capítulo

- `rules/` é o **conhecimento de processo carregado sob demanda** — o detalhe que não cabe na Norma.
- As **especificações profundas** (log, reuso, pronto, suíte, python, large-repo) são as instruções de
  trabalho de cada operação.
- Os **contratos de passagem de bastão** (estratégia, fidelidade) existem em arquivo próprio para **não
  duplicar** entre estações — DRY na própria config.
- O **registro de melhoria contínua** (lessons, error-backlog, regression, bad-instructions) é o Kaizen
  que faz a fábrica melhorar com o tempo.
- A regra de ouro recorrente: **"se divergir do código executável, o código vence"** — a doc nunca se
  acha dona da verdade.

**Próximo:** [Capítulo 3 — As Estações da Linha (`.claude/agents/`)](capitulo-03-estacoes-da-linha.md).
