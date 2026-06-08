# Desenvolver e Corrigir Sem Digitar Uma Linha de Código

### (e por que o *code review* tradicional se torna redundante — ele já está embutido no processo)

> **Documento da base conceitual — neutro a qualquer lente. É o tema mais estratégico da metodologia.**

> **Por que este documento é o mais estratégico.** As outras partes explicam as *peças*. Este explica
> a **aposta**: é possível desenvolver e corrigir software de qualidade **sem um humano digitar uma
> linha de código** — e, mais ousado ainda, **sem uma etapa separada de code review** — porque a função
> do code review foi **dissolvida e embutida** ao longo de todo o processo. A garantia não vem de
> confiar no agente; vem do **conteúdo dos contratos** que dão ao agente o poder de fazer certo e
> tornam a revisão humana linha-a-linha desnecessária.

> 🧑‍💼 **RESUMO EXECUTIVO.** A pergunta que tira o sono da liderança é: *"como aprovar código que
> ninguém escreveu e ninguém revisou linha por linha?"*. A resposta deste apêndice: **você não aprova
> código — você aprova evidência**. O controle de qualidade deixou de ser um humano lendo diffs no fim
> (um gargalo que cansa, tem pressa e às vezes carimba sem ler) e virou um **sistema de garantias
> codificadas, contínuas e independentes**, aplicadas a cada passo. Isso não é "abrir mão da revisão" —
> é **substituí-la por algo mais rigoroso, que nunca tem um dia ruim**.

---

## D.1 A virada de papel: de digitador a maestro

No modelo tradicional, o engenheiro **digita** o código e outro engenheiro **revisa** o código. Os dois
gargalos são humanos.

No nosso modelo, o humano **conduz** e **decide**; os agentes **executam**. O humano nunca abre o editor.
O que ele faz, em vez disso:

```
   MODELO TRADICIONAL                    NOSSO MODELO (orquestração)
   humano digita código        ──►       humano dá o comando (/investigar, /implementar...)
   humano revisa o diff        ──►       humano lê EVIDÊNCIA (relatórios, planos, vereditos, logs)
   humano aprova linha a linha ──►       humano aprova na ALTITUDE da decisão, não da sintaxe
```

A mudança é de **altitude**: o humano sobe do nível da linha de código para o nível da **evidência e da
decisão**. Ele não confere se a variável foi nomeada certo — disso cuidam os contratos e os hooks. Ele
confere se a **investigação tem evidência**, se o **plano é seguro**, se a **validação provou** o
comportamento no runtime. É a diferença entre quem **executa** a tarefa e quem **dirige** o trabalho.

---

## D.2 A pergunta de um milhão: por que confiar em código que você não leu?

A confiança **não** é fé no agente. Ela é a soma de seis garantias estruturais — e cada uma tem um
**dono codificado** (um arquivo, não uma boa intenção):

| Garantia | Por que torna a leitura linha-a-linha dispensável | Onde mora (o conteúdo que garante) |
|---|---|---|
| **1. Não há alucinação** | O código nasce de evidência real lida, não de "achismo" | `CLAUDE.md §1` + `investigar` (afirmação só com path/linha) |
| **2. Funções segregadas** | Quem implementa **não** se auto-aprova — há revisor independente | eixo `investigar→planejar→implementar→validar-entrega` |
| **3. Anti-falso-verde** | A entrega só vale **ativa no runtime**, provada | `definicao-de-pronto.md` + check discriminante |
| **4. Observabilidade (raio-X)** | Você audita o **comportamento pelo log**, não pela leitura do código | `log-instructions.md` + `validar-observabilidade` |
| **5. Auto-correção com teto** | O sistema **converge ou declara bloqueio** — não entrega "quase certo" | loops em `validar-entrega`, `executar-testes`, `corrigir-erros-com-log` |
| **6. Arbitragem automática** | Violações de estilo/disciplina são pegas **no ato**, deterministicamente | hooks (`py-lint`, `logging-nudge`, `bash-guard`…) |

> 🛠️ Leia a tabela ao contrário e você tem a tese: **a confiança não depende de eu ler o código —
> depende de o código ter passado por seis filtros que um revisor humano, sozinho e cansado, jamais
> aplicaria com a mesma consistência.**

---

## D.3 O loop completo, comando a comando — com foco no *conteúdo que garante*

Aqui está o ponto central do apêndice. Para cada etapa, três blocos:
- 🎮 **Você comanda** (o prompt que você dá — única coisa que você "digita");
- 🤖 **O agente faz** (o trabalho real, incluindo escrever código);
- 🔒 **O conteúdo que garante** (as regras internas que fazem ele fazer certo **sem você revisar**).

### Etapa 1 — `/investigar` (levantar a verdade)
- 🎮 **Você comanda:** "investigue por que a ingestão falha".
- 🤖 **O agente faz:** lê o código real, mapeia o fluxo, produz um relatório com achados e evidência.
- 🔒 **O conteúdo que garante (sem você ler código):**
  - **Zero alucinação:** proibido afirmar sem `read` com filepath e linhas — então o relatório que você
    recebe é **rastreável até o código**, e você confere a *evidência*, não o código.
  - **3 fases obrigatórias** (mapear → fechar lacunas → auditar observabilidade) que não podem ser
    puladas — garante que a investigação não é superficial.
  - **Materialização obrigatória:** o relatório existe como arquivo auditável, não some no chat.
- 🧪 **Como você confere sem digitar:** lê o relatório e checa se cada achado tem evidência (path/linha).
  Se tiver, está provado; se não tiver, o próprio contrato proíbe que esteja lá.

### Etapa 2 — `/planejar` (desenhar a rota segura)
- 🎮 **Você comanda:** "planeje a correção".
- 🤖 **O agente faz:** avalia criticamente e monta um plano com tarefas, critérios de aceite e rollback.
- 🔒 **O conteúdo que garante:**
  - **Postura crítica obrigatória:** antes de planejar, o agente diz **Sim/Não/Depende** com prós,
    contras e alternativas mais baratas. Isso é **a função de design review** acontecendo *antes* do
    código existir — o momento mais barato de pegar um erro de arquitetura.
  - **T1 cria a `# Estratégia/Recomendacoes`** (a bússola: o que preservar/mudar/remover, quais
    fallbacks são proibidos) e **T2 mapeia os testes de proteção** — antes de qualquer mudança.
  - **Anti-omissão silenciosa:** todo achado da investigação vira tarefa ou justificativa. Nada
    desaparece entre uma etapa e outra.
- 🧪 **Como você confere sem digitar:** lê o plano. Se cada tarefa tem critério de aceite e validação,
  você sabe **como o sucesso será provado** — antes mesmo de uma linha ser escrita.

### Etapa 3 — `/implementar` (escrever o código — o agente, não você)
- 🎮 **Você comanda:** "implemente o plano".
- 🤖 **O agente faz:** escreve o código, cria os testes, instrumenta os logs.
- 🔒 **O conteúdo que garante (este é o coração da dispensa do code review):**
  - **Lê a bússola antes de tocar qualquer arquivo:** conflito entre tarefa e estratégia = parar e
    resolver. O agente não improvisa fora do combinado.
  - **Gate de reuso** (`reuso-instructions.md`): antes de criar algo novo, é obrigado a procurar o que
    já existe. *Isso é a checagem de duplicação do code review — feita sempre, não às vezes.*
  - **Gates de banco/YAML/LangChain:** lê o contrato do subsistema antes de tocá-lo. *Isso é a checagem
    de "você seguiu o padrão?" do code review — automática.*
  - **Default anti-legado:** não deixa fallback, dual-read ou código morto sem pedido explícito. *Isso é
    a checagem de "limpou a sujeira?" do code review.*
  - **Definição de pronto:** antes de declarar CONCLUÍDA, prova que o comportamento está **ativo no
    caminho oficial** e cria **teste de regressão estrutural**. Falhou? Vira BLOQUEADA.
  - **E, em paralelo, os hooks:** a cada `.py` salvo, `py-lint`, `logging-nudge` e `py-discipline`
    levantam a bandeira para `logger.error` em `except`, f-string em log, `print()` em `src/`, teste sem
    marker, import opcional, `FOR UPDATE`… *Isso é a checagem de estilo e disciplina do code review —
    feita no instante da escrita, deterministicamente, sem cansar.*
- 🧪 **Como você confere sem digitar:** você não confere o código aqui — quem confere é a Etapa 4.

### Etapa 4 — `/validar-entrega` (o code review, agora como agente independente)
- 🎮 **Você comanda:** "valide a entrega".
- 🤖 **O agente faz:** confronta o implementado contra o plano e a investigação, e dá um veredito.
- 🔒 **O conteúdo que garante:**
  - **Hierarquia de verdade absoluta:** código executável > teste > log > plano > doc. **"Teste verde
    não prova fluxo real."** É exatamente o ceticismo que um bom revisor sênior aplica — codificado.
  - **Validação ponta a ponta por tarefa:** cada T1..Tn é conferida individualmente.
  - **Pode acionar `validar-log` e `validar-observabilidade`** como evidência subordinada.
  - **Veredito explícito** (APROVADO / RESSALVAS / REPROVADO / BLOQUEADO) e, se reprovado, **devolve a
    bola para o `planejar`** com inventário do que faltou.
- 🧪 **Como você confere sem digitar:** lê o relatório de validação. Ele te diz **qual boundary foi
  validado, qual implementação concreta roda, qual teste protege, e por que não há falso-verde.**

> 🛡️ **O ponto que o `validar-entrega` materializa:** o code review não foi eliminado — ele foi
> **promovido a um agente independente, com checklist codificado, que nunca tem pressa, nunca carimba
> sem ler e nunca deixa passar "porque eu confio no autor".**

---

## D.4 O loop de correção sem digitar — o log no lugar da leitura de código

Corrigir um bug normalmente exige um humano ler o código, formar hipótese e testar. Aqui, o
`corrigir-erros-com-log` faz isso **a partir do log**, não do teclado do humano:

```
   bug em produção
        │
   /analisar-log  ──► lê o log por correlation_id (o "raio-X") e acha a etapa do erro
        │
   /corrigir-com-log ──► reconcilia "história do log" × "fluxo do código",
        │                 acha a causa raiz, corrige DEFINITIVO (proibido mascarar com fallback),
        │                 instrumenta log faltante, valida com a suíte
        │
   (memória de rodada impede repetir hipótese já falsificada)
        │
   converge ──► SUCESSO   ou   declara ──► BLOQUEADA com evidência
```

🔒 **O conteúdo que garante a correção sem você abrir o código:**
- **O log é tratado como evidência operacional reconstruível** (`log-instructions.md`) — então o
  diagnóstico parte de fatos registrados, não de leitura especulativa.
- **Correção na causa raiz, proibido mascarar:** o contrato proíbe "resolver" com fallback que esconde
  o problema. Você recebe o conserto real, não o curativo.
- **Checagem discriminante + memória de rodada:** cada hipótese é testada para ser **desconfirmada**
  primeiro, e nenhuma hipótese já descartada volta. O loop é inteligente, não teimoso.

> O insight estratégico: **o log substitui a leitura de código como fonte de diagnóstico.** É por isso
> que investir pesado em observabilidade (todo o `log-instructions.md`) não é luxo — é o que **viabiliza
> corrigir sem digitar**.

---

## D.5 Por que o Code Review separado se torna redundante

Esta é a afirmação ousada, e ela se sustenta quando você decompõe **o que o code review realmente
faz** e mostra que **cada função já tem um dono contínuo no processo**:

| Função clássica do Code Review | Onde ela já está embutida (e por que é melhor) |
|---|---|
| Pegar **bug de correção** | `validar-entrega` (revisor independente) + auto-correção em **execução real** + testes — pega o que olho humano em diff não pega |
| Conferir **estilo/padrão** | hooks `py-lint`/`py-discipline` — **no instante da escrita**, determinístico, sem cansaço |
| Garantir que **há testes** | `criar-testes` + nudge `stop-loop` (src sem tests) + teste de regressão da `definicao-de-pronto` |
| Checar **arquitetura/design** | `planejar` (postura crítica **antes** do código) + `CLAUDE.md §6` (SOLID, limites de classe) |
| Evitar **duplicação** | `reuso-instructions.md` + `inventario-componentes` — busca obrigatória antes de criar |
| Garantir **observabilidade** | `logging-nudge` + `validar-observabilidade` + `validar-log` |
| Impedir **código morto/legado** | default anti-legado do `implementar` + checklist de fechamento do `CLAUDE.md §14` |
| Barrar **risco de segurança** | `bash-guard` (guard) + `CLAUDE.md §8` (tools falha fechado) |
| Confirmar que **está realmente pronto** | `definicao-de-pronto.md` (anti-falso-verde) + `validar-entrega` |

**Por que isso é superior ao code review tradicional, e não apenas equivalente:**

1. **Shift-left ao extremo.** O code review tradicional acontece **no fim**, quando consertar é caro.
   Aqui, cada função é aplicada **no momento mais barato**: o design é criticado antes do código existir
   (`planejar`); o estilo é pego no ato da escrita (hooks); o reuso é checado antes de criar.
2. **Independência real.** O `validar-entrega` é um agente **separado** do `implementar`, com hierarquia
   de verdade que **não aceita "teste verde" como prova**. É a segregação de funções que muitos times
   humanos não conseguem manter (o revisor é colega, tem pressa, confia no autor).
3. **Consistência sem fadiga.** Um humano revisando o 40º diff do dia relaxa. Os contratos e hooks
   aplicam **o mesmo rigor na primeira e na milésima vez**.

**O papel específico de cada arquivo na dispensa do code review:**
- **`CLAUDE.md`** define os *critérios* que um revisor checaria (evidência, reuso, anti-falso-verde,
  escopo mínimo, SOLID) — e os torna **lei em toda sessão**.
- **`planejar`** executa o *design review* **antes** de existir código para revisar.
- **`implementar`** carrega os *gates* (reuso, banco, YAML, anti-legado, definição de pronto) que um
  revisor cobraria — e os cumpre durante a escrita, não depois.
- **`validar-entrega`** **é** o revisor — só que codificado, independente e cético por contrato.
- **Os hooks** são o *linter + checklist de PR* rodando automaticamente.

> 🧑‍💼 **A frase para a liderança:** "Não removemos o code review. Nós o **desmontamos em dezenas de
> verificações automáticas e contínuas**, distribuídas por todo o processo e fechadas por um revisor
> independente codificado. O que era um gargalo humano no fim virou uma malha de garantias do começo ao
> fim."

---

## D.6 Onde mora a garantia: não é o agente, é o conteúdo

O agente é só o executor. **O poder dele de fazer certo vem do conteúdo que ele carrega.** Tire o
conteúdo e o agente vira um estagiário genérico. Por isso a engenharia de verdade está nos arquivos:

```
   AGENTE (executor)  ◄── recebe poder de ──  CONTEÚDO (a garantia)
   investigar         ◄──  zero-alucinação, 3 fases, materialização
   planejar           ◄──  postura crítica, T1/T2, estratégia, anti-omissão
   implementar        ◄──  gates de reuso/banco/YAML, anti-legado, definição-de-pronto
   validar-entrega    ◄──  hierarquia de verdade, validação por tarefa, anti-falso-verde
   corrigir-com-log   ◄──  log como evidência, causa raiz, checagem discriminante
   TODOS              ◄──  CLAUDE.md (DNA) + log-instructions + hooks (arbitragem)
```

**É por isso que "o conteúdo é estratégico":** se amanhã trocássemos o modelo de IA por outro, **os
contratos continuariam valendo** e continuariam impondo o mesmo rigor. O ativo da empresa não é o
agente — é o **corpo de contratos** que transforma qualquer executor competente em um time sênior
disciplinado.

---

## D.7 A leitura honesta — o que NÃO desaparece (sem isso, vira venda de ilusão)

O `CLAUDE.md` exige postura crítica, então sejamos honestos sobre os limites:

- **O humano não some — ele sobe de altitude.** Você ainda lê e aprova **investigação, plano e
  validação**. Isso *é* revisão — só que no nível da evidência e da decisão, não da sintaxe. Quem
  aprova um plano ruim recebe uma execução fiel a um plano ruim.
- **A garantia é tão boa quanto os contratos.** Contrato fraco → revisão fraca. Por isso existe o
  `validar-instructions` (auditando as próprias regras) e o `bad-instructions.md`. A qualidade dos
  arquivos é o investimento que sustenta tudo.
- **Decisões de produto e arquitetura realmente novas** ainda pedem julgamento humano — e o sistema é
  desenhado para **escalá-las** (status BLOQUEADA, "decisão externa", a postura crítica do `planejar`),
  não para escondê-las.
- **"Sem digitar código" ≠ "sem engenharia".** O esforço de engenharia migra da digitação para a
  **curadoria dos contratos e a leitura crítica da evidência**. É mais alavancado, não inexistente.

> Em uma frase honesta: **o code review linha-a-linha como ritual humano no fim torna-se redundante; o
> julgamento de engenharia, não.** O segundo apenas muda de lugar — sobe para a altitude da evidência.

---

## D.8 Resumo de bolso (para a aula)

- O humano vira **maestro**: comanda e decide; o agente digita o código.
- A confiança vem de **seis garantias codificadas**, não de fé no agente.
- O **code review separado fica redundante** porque cada uma de suas funções já tem **dono contínuo**:
  design review no `planejar` (antes do código), estilo nos hooks (no ato), reuso/padrão no
  `implementar` (durante), e o veredito cético no `validar-entrega` (revisor independente).
- A garantia **mora no conteúdo dos contratos**, não no agente — por isso o conteúdo é o ativo
  estratégico.
- Honestidade: o humano não desaparece; ele **sobe da linha de código para a evidência**.

> **Frase para fechar (e talvez a aula inteira):** "A gente não confia no código porque alguém o leu
> linha por linha. A gente confia porque ele **nasceu de evidência, foi construído sob contrato, provado
> no runtime e julgado por um revisor independente que nunca tem pressa** — e nada disso exigiu um
> humano digitando ou revisando uma única linha."

---

**Relacionado (base conceitual):** [Os Dois Loops](os-dois-loops.md) ·
[Ciclo de Vida do Software](ciclo-de-vida-software.md) · [FAQ de Objeções](faq-objecoes.md) ·
[↩ Voltar ao índice da metodologia](../README.md)
</content>
