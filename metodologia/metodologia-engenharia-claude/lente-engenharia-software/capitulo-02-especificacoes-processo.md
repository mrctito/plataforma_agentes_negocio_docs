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

São **20 especificações** hoje — e há um detalhe de engenharia na forma de entrega: **19 das 20 declaram
no cabeçalho (`paths:`) a quais arquivos se aplicam**, e a instrução é injetada automaticamente quando o
trabalho toca aquele caminho — inclusive quando o "caminho" é o arquivo de uma estação (a regra entra
junto quando a estação opera). É **injeção de dependência de contexto**: nenhuma estação copia a
especificação, então nenhuma cópia desatualiza. A única exceção sem `paths:` é `qualidade-texto.md`, que
vale para todo texto.

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
pesquisar o que já existe, consultar `docs/tecnico/README-TOOLS-LIB.md`, ler os candidatos, e só então —
com justificativa — fabricar algo novo.

**O que força:** uma ordem de **6 passos** (identificar candidatos → buscar no código → ler direto →
entender a intenção → avaliar se resolve/evolui → **só então** considerar criar algo novo), com postura
de **advogado do diabo**: "em fábrica madura, peça nova é rara".

**Problema que evita:** duplicação. Refabricar um componente que já existe gera **divergência de
comportamento** (dois jeitos de fazer a mesma coisa, que evoluem diferente), dobra a manutenção e espalha
inconsistência. E um achado-chave: se a busca encontra **duas** soluções para o mesmo problema, isso é
**inconsistência de processo** a unificar — proibido adicionar uma terceira.

**Valor:** DRY, consistência, manutenibilidade (estoque enxuto de código).

### `definicao-de-pronto.md` — o procedimento de inspeção final (anti-falso-verde)

O portão de saída que já viveu na Norma raiz e hoje é SOP dedicado. Define "pronto" de forma
**falsificável**.

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

### As especificações de governança da própria linha

Além das instruções de operação, um grupo de SOPs governa **como a fábrica trabalha** — investigação,
custo, paralelismo e texto:

- **`loops-estrategicos.md` — o contrato dos dois motores.** Declara os dois loops como features de
  primeira classe: **Loop 1**, auto-correção por log (erro real de backend → capturar `correlation_id` →
  abrir o log oficial → cara-crachá log × código até a causa **provada** — proibido declarar
  "ambiente/credencial/sem dado" por inferência); **Loop 2**, memória de rodada (registrar cada tentativa
  falsificada para não repetir beco sem saída na mesma campanha). É a fonte do Kaizen real (§2.2).
- **`regras_uso_subagentes.md` — a política de custo por estação.** Define quando um subagente pode usar
  o modelo barato, com **duas travas**: (1) **trava do falso negativo** — quem entrega afirmação de
  ausência/completude ("não existe", "está vazio", "cobertura total") fica no modelo forte, porque saída
  verificável prova precisão, nunca recall; (2) **baixe o effort antes de baixar o tier**. Pergunta de
  corte: "se esse subagente errar, o principal percebe e corrige fácil?"
- **`ferramentas-acesso-dados.md` — o inventário de instrumentos de medição.** "LEIA ANTES de dizer 'não
  tenho como verificar'": catálogo dos scripts prontos de `.claude/scripts/` (Qdrant, PostgreSQL, Redis,
  filas, scheduler, logs) que tocam a **fonte de verdade real**; proibido reinventar acesso ad-hoc quando
  há ferramenta pronta.
- **`disciplina-investigacao-teste.md` — a trava anti-teste-raso.** Falha, vazio ou erro são o **início**
  da investigação: enumerar hipóteses → testar cada uma na fonte real → só rotular a causa depois de
  esgotá-las. Distingue camadas que se parecem ("tabela-registro vazia" ≠ "store físico vazio").
- **`execucao-plano-resumivel.md` — a produção com apontamento por operação.** Orquestrador quente na
  janela principal + worker descartável por fase + **diário write-ahead** por tarefa no arquivo do plano;
  perda máxima numa queda = a tarefa em voo. Contrapeso: plano de até ~4 tarefas não se fracionar.
  *(Detalhe no [Cap. 4](capitulo-04-ordem-de-producao.md).)*
- **`qualidade-texto.md` — a especificação do texto.** Anti-linguiça **e** anti-lacuna: mede densidade de
  valor por linha, nunca tamanho. Conteúdo necessário entra mesmo longo; redundante sai mesmo curto.
- **`worktree.md` — o procedimento de células paralelas.** Como criar e operar worktrees isoladas
  (naming fixo `worktree-codex`/`worktree-claude`, setup de `.env`/`.venv`, foco exclusivo na própria
  célula) — imposto por um dispositivo de intertravamento (ver [Cap. 5](capitulo-05-pokayoke-e-andon.md)).

### A prateleira completa — os 20 SOPs

| SOP | Em uma linha |
|---|---|
| `log-instructions.md` | O registro de processo (raio-X): correlation_id, campos canônicos, builders |
| `reuso-instructions.md` | Almoxarifado antes de fabricar: 6 passos, advogado do diabo |
| `definicao-de-pronto.md` | Inspeção final anti-falso-verde: pronto = ativo no runtime |
| `suite-testes-instructions.md` | O procedimento de ensaios: modos, `--run-id`, telemetria |
| `python.md` | A especificação do material: tipagem, mocks, markers, retry |
| `large-repo-navigation.md` | Navegar a planta enorme: âncora, slices, "busca localiza, leitura comprova" |
| `estrategia-recomendacoes.md` | Contrato de passagem de bastão PCP → fabricação |
| `fidelidade-pedido-usuario.md` | O pedido do cliente não evapora no meio da linha |
| `loops-estrategicos.md` | Os dois motores: correção por log + memória de rodada |
| `regras_uso_subagentes.md` | Custo por estação: tier de modelo com as duas travas |
| `ferramentas-acesso-dados.md` | Instrumentos prontos para tocar a fonte de verdade real |
| `disciplina-investigacao-teste.md` | Anti-teste-raso: esgotar hipóteses na fonte real |
| `execucao-plano-resumivel.md` | Ordem de produção com apontamento por operação (write-ahead) |
| `qualidade-texto.md` | Anti-linguiça + anti-lacuna em todo texto |
| `worktree.md` | Células de produção paralelas isoladas |
| `ambiente-local.md` | Operar a planta local: FastAPI, porta presa, navegador |
| `componente-chat-embutivel.md` | Contrato do componente de chat embutível |
| `dyn-sql-tools-registro.md` | Contrato das tools SQL dinâmicas (YAML-first + registro) |
| `padrao-listas-perguntas-teste.md` | Padrão das listas de perguntas usadas nos ensaios |
| `subagentes-descricao-instructions.md` | Regra arquitetural das descrições de subagentes |

---

## 2.2 O mecanismo real de melhoria contínua (o Kaizen da fábrica)

Aqui a fábrica "aprende com a produção" — mas não por um diário central que alguém preenche no fim do
turno. O mecanismo real tem **três camadas**, cada uma com sua vida útil:

1. **O contrato (`loops-estrategicos.md`):** todo erro real dispara o Loop 1 (prova forense por log) — a
   fábrica **não pode** rotular a causa sem evidência. É o que gera matéria-prima de aprendizado
   confiável: causa provada, não achismo.
2. **A memória de rodada (Loop 2):** dentro de uma campanha de correção, cada tentativa é registrada com
   evidência (erro dominante, hipótese, checagem discriminante, ação, resultado) e **relida antes de cada
   nova hipótese** — a campanha não repete beco sem saída. É memória **efêmera, local ao lote**.
3. **A promoção com gate (o Kaizen durável):** quando uma rodada revela uma **regra preventiva durável
   comprovada**, ela é promovida para o arquivo `licoes-aprendidas.md` **do agente** que a aprendeu — com
   o contrapeso explícito de "**não promover ruído operacional local**". As memórias reais existem e têm
   volume: `corrigir-erros-com-log` e `testar-ingestao-dnit` acumulam dezenas de KB de lições cada um.

**A engenharia por trás:** separar o **efêmero** (memória de rodada, morre com o lote) do **durável**
(lição promovida, protege os próximos lotes), com um gate humano-auditável no meio. Antes de promover, a
pergunta é a mesma de sempre: *"se este agente operar amanhã em outro lote, esta regra ainda reduziria
chance real de defeito?"* Se não, fica na rodada. Isso impede que o registro durável vire depósito
inútil. E o fecho do ciclo é **curadoria humana**: lição durável que pede mudança de contrato vira edição
da Norma ou de um SOP — pelo humano, não por auto-escrita.

**Valor:** **Kaizen** — melhoria contínua **institucional** (o know-how fica na empresa, não na pessoa),
com uma honestidade importante: o gatilho é o **defeito provado**, não um lembrete automático de fim de
turno.

> 🧑‍💼 **RESUMO EXECUTIVO.** Este é o mecanismo que faz a produção ficar **melhor com o tempo**. Cada
> defeito provado vira uma regra que previne o próximo — com um filtro que separa lição de ruído. É
> capitalização de conhecimento: o que aprendemos numa quarta protege todos os lotes da quinta em diante.

---

## 2.3 O que levar deste capítulo

- `rules/` são os **20 SOPs carregados sob demanda** — 19 deles injetados por caminho (`paths:`), o
  detalhe que não cabe na Norma entregue no momento exato da operação.
- As **especificações profundas** (log, reuso, pronto, suíte, python, large-repo) são as instruções de
  trabalho de cada operação; as de **governança** (loops, subagentes, acesso a dados, investigação,
  plano resumível, texto, worktree) definem como a própria linha trabalha.
- Os **contratos de passagem de bastão** (estratégia, fidelidade) existem em arquivo próprio para **não
  duplicar** entre estações — DRY na própria config.
- O **Kaizen real** é o trio contrato dos loops + memória de rodada + promoção com gate para o
  `licoes-aprendidas.md` de cada agente — efêmero separado do durável, com curadoria humana no contrato.
- A regra de ouro recorrente: **"se divergir do código executável, o código vence"** — a doc nunca se
  acha dona da verdade.

**Próximo:** [Capítulo 3 — As Estações da Linha (`.claude/agents/`)](capitulo-03-estacoes-da-linha.md).
