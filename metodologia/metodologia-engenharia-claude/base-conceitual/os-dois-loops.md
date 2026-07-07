# Os Dois Loops que Sustentam a Metodologia

> **Documento da base conceitual — neutro a qualquer lente.** Cada apresentação (futebol, fábrica de
> software, futuras) enquadra estes dois loops na sua própria linguagem; aqui está a **substância única**,
> sem metáfora.

> **Por que este documento existe.** Ao longo da metodologia, dois mecanismos de "repetição inteligente"
> aparecem em vários pontos. Eles são parecidos no nome e **opostos na função** — e confundi-los é o erro
> mais comum de quem olha o sistema de fora. Este documento isola os dois, lado a lado, porque **o
> contraste entre eles é o verdadeiro insight**.

> 🧑‍💼 **RESUMO EXECUTIVO.** Há dois motores de melhoria rodando o tempo todo:
> - O **loop de auto-correção** conserta **uma tarefa** até ela ficar certa (qualidade *dentro* do trabalho).
> - O **loop de auto-aperfeiçoamento** deixa **o sistema inteiro** mais inteligente a cada rodada
>   (qualidade *entre* os trabalhos).
> Um garante que o trabalho de hoje sai bom; o outro garante que o trabalho de amanhã começa melhor.

---

## A.1 A confusão que este documento desfaz

```
  AUTO-CORREÇÃO                          AUTO-APERFEIÇOAMENTO
  "conserta ESTA tarefa"                 "melhora O SISTEMA para as próximas"
  escopo: uma tarefa/rodada              escopo: todas as tarefas futuras
  vive: dentro do agente em execução     vive: na memória durável (lições + contratos)
  termina: quando a tarefa fica certa    termina: nunca (é contínuo)
  pergunta: "já está correto?"           pergunta: "o que aprendemos para não repetir?"
```

Uma analogia do dia a dia fixa a diferença:
- **Auto-correção** é **revisar e refazer um trabalho enquanto ele ainda está em cima da mesa**, até
  ficar certo.
- **Auto-aperfeiçoamento** é **mudar o procedimento padrão** com base no que deu errado da última vez —
  para o próximo trabalho já começar melhor.

---

## A.2 O Loop de Auto-Correção (convergência dentro da tarefa)

**O que é:** um ciclo fechado de *executar → detectar falha → corrigir a causa raiz → revalidar*, que
se repete até o resultado ficar comprovadamente certo — **com teto** para não rodar para sempre.

**O contrato que o torna obrigatório:** `.claude/rules/loops-estrategicos.md` (Loop 1 — auto-correção
por log). Erro real de backend em qualquer teste/validação exige capturar o `correlation_id`, abrir o
log oficial (`python -m src.log_analyzer`) e confrontar log × código ("cara-crachá") até a causa raiz
**provada** — é proibido rotular "ambiente / fora de escopo / credencial / dado faltando" por inferência.

**Onde ele vive (evidência real no projeto):**

| Onde | Como o loop funciona | Trava contra loop infinito |
|---|---|---|
| `validar-entrega` | REPROVADO → gera inventário de não-entregas → **devolve a bola para o `planejar`** | A bola só volta com lista objetiva do que faltou |
| `executar-testes` | executar → falhar → corrigir → validar local → re-executar (resume) | **Máximo ~20 ciclos** ou impasse técnico comprovado |
| `corrigir-erros-com-log` | matriz de tentativas: erro dominante → hipótese → **checagem discriminante** → ação → resultado | Não repete hipótese **já falsificada** |
| `testar-ingestao-dnit` | executar → capturar correlation_id → monitorar log → validar 3 provas → corrigir → reiniciar | **Máximo ~50 tentativas**; só passa disso com bloqueio sistemático comprovado |
| `testar-cli-log-analyzer` | comparar CLI × verdade manual → corrigir → revalidar em cascata | Loop só fecha quando a **matriz inteira** bate |

**Os três pilares de engenharia que tornam o loop seguro (e não um moto-perpétuo burro):**

1. **Checagem discriminante.** Antes de cada nova tentativa, faz-se "o check mais barato capaz de
   **desconfirmar** a hipótese atual". Isso impede insistir numa teoria errada.
2. **Memória de rodada.** Cada tentativa registra o que foi testado, para **nunca repetir uma hipótese
   já descartada**. É o que diferencia "iterar com inteligência" de "tentar a mesma coisa esperando
   resultado diferente".
3. **Teto explícito + status binário.** Há um limite de ciclos e um veredito sem meio-termo (SUCESSO /
   ERRO, APROVADO / REPROVADO). O loop **converge ou declara bloqueio com evidência** — nunca fica num
   "quase pronto" eterno.

**Problema de engenharia que evita:** a entrega que "quase funciona" e o retrabalho infinito. Em vez de
o humano descobrir o erro depois, o próprio sistema **insiste até acertar ou prova que está travado**.

**Valor:** convergência confiável para o resultado correto, dentro da tarefa, sem supervisão humana
constante.

> 🛠️ **Detalhe fino:** repare que o "retorno" do `validar-entrega` para o `planejar` é um loop de
> auto-correção que atravessa **vários agentes** — não é um agente sozinho insistindo. O sistema se
> corrige no nível do *processo inteiro*, não só do indivíduo.

---

## A.3 O Loop de Auto-Aperfeiçoamento (o sistema fica melhor entre as tarefas)

**O que é:** um ciclo que **atravessa rodadas e tarefas** — o que uma campanha aprendeu de forma
comprovada é promovido para uma memória durável e versionada, que as rodadas seguintes releem antes de
agir.

**Onde ele vive (evidência real no projeto):** a fonte única dos dois loops é o contrato
**`.claude/rules/loops-estrategicos.md`**. Além do Loop 1 (a auto-correção da seção A.2), ele declara o
**Loop 2 — memória de rodada**: dentro de uma mesma campanha, cada tentativa é registrada com evidência
objetiva (erro dominante, hipótese, checagem discriminante, ação, resultado, próximo passo) e **relida
antes de cada nova hipótese**. Essa memória é **local à campanha** — não vira registro global
automaticamente.

O salto do efêmero para o durável acontece pela **promoção com porteiro**, escrita no contrato do
agente `corrigir-erros-com-log`: *"quando surgir regra preventiva durável comprovada, promover para os
`licoes-aprendidas.md`"* — com o contrapeso explícito *"não promover ruído operacional local"*.

```
   DENTRO DA CAMPANHA (efêmero)               ENTRE CAMPANHAS (durável)
   log-rodada.md                              licoes-aprendidas.md (por agente)
   ── cada tentativa, com evidência           ── só regra preventiva COMPROVADA
   ── relido antes de cada nova hipótese      ── relido no início da campanha seguinte
           │                                                ▲
           └──── "regra preventiva durável comprovada?" ────┘
                (porteiro: não promover ruído operacional local)
```

**As memórias duráveis existem e estão em uso real:**

| Memória | O que guarda |
|---|---|
| `.claude/agents/corrigir-erros-com-log/licoes-aprendidas.md` | Lições preventivas comprovadas do loop forense (~22 KB de lições reais) |
| `.claude/agents/testar-ingestao-dnit/licoes-aprendidas.md` | Lições da campanha de ingestão (~24 KB) |
| `log-rodada.md` (ao lado de cada `licoes-aprendidas.md`) | A memória de rodada efêmera (Loop 2) |
| `.claude/scripts/` | Script ad hoc que se provou reutilizável é promovido — deixa de ser recriado a cada rodada |

**A trava de curadoria (por que a memória não vira lixo):** só é promovido o que é **preventivo,
durável e comprovado**; detalhe operacional local fica no `log-rodada.md` e morre com a campanha. Isso
impede o problema clássico da "wiki que ninguém lê porque está cheia de detalhe irrelevante".

**Limite honesto (declarado, não escondido):** o aprendizado **não é disparado por hook de sessão** —
depende do contrato do agente no momento da lição. Existe um hook `stop-loop-reminder.sh` em
`.claude/hooks/`, mas ele **não está plugado** em nenhum settings (wiring pendente — lacuna conhecida),
e seu conteúdo cobre apenas o nudge "editou `src/**.py` sem tocar `tests/`". Hoje, o lado "ler" do loop
é garantido pelos próprios agentes (releem suas lições antes de nova hipótese) e o lado "escrever", pelo
gate de promoção.

**Problema de engenharia que evita:** o sistema repetir o mesmo erro em campanhas diferentes; e o
conhecimento morar só na janela de contexto, que morre com a sessão. É a **capitalização de
conhecimento** com curadoria.

**Valor:** melhoria contínua **institucional** — a lição comprovada vira patrimônio versionado do
sistema, não lembrança de quem estava na sessão.

---

## A.3.1 O rastro da tarefa: os artefatos materializados

As memórias acima capturam **aprendizado**. Falta o destino que captura **a própria entrega**: o eixo
`investigar → planejar → validar-entrega` **materializa** cada etapa em arquivo auditável em
`docs/.interno/.planos/<nome>/` (`investigacao--*.md`, `plano--*.md`, `validacao--*.md`), e toda
execução real deixa seu log por `correlation_id`. Não é "mais um log" — é o que faz uma entrega
**deixar de viver só no chat e virar evidência versionada**: uma regressão futura pode ser cruzada com
a entrega que a introduziu, fechando o elo entre "o que mudou" e "o que voltou a quebrar".

## A.3.2 Gatilho rápido: na dúvida, onde isso fica registrado?

| Aconteceu… | Fica em |
|---|---|
| Tentativa/hipótese testada **nesta campanha** de correção | `log-rodada.md` do agente (memória de rodada) |
| Regra preventiva **durável e comprovada**, que evitaria erro em campanha futura | `licoes-aprendidas.md` do agente (promoção com porteiro) |
| O que uma **execução real** fez, decidiu e onde errou | log por `correlation_id` (`python -m src.log_analyzer`) |
| O que foi **investigado, planejado e validado** numa entrega | `docs/.interno/.planos/<nome>/` (relatórios materializados) |

> Regra anti-filler: promove-se **lição**, não diário. Ajuste trivial sem valor preventivo transversal
> **não** sobe para o `licoes-aprendidas.md`. O loop melhora porque é **curado**, não porque acumula
> tudo.

---

## A.4 A ponte entre os dois loops (onde eles se encontram)

Os dois loops não são ilhas. Há um ponto de contato deliberado:

> Quando o **loop de auto-correção** de uma rodada descobre uma regra preventiva reaproveitável, essa
> lição é **promovida** para o **loop de auto-aperfeiçoamento** (vai para o `licoes-aprendidas.md` do
> agente).

Ou seja: o conserto de **um** problema de hoje pode virar a **prevenção** de uma família de problemas
amanhã. O esforço de auto-correção não se perde quando a tarefa fecha — parte dele é capturada e vira
patrimônio do sistema.

```
   auto-correção (rodada)  ──promove lição transversal──►  auto-aperfeiçoamento (memória durável)
   "consertei este bug"                                     "nunca mais caímos neste tipo de bug"
```

---

## A.5 Resumo de bolso

| | Auto-correção | Auto-aperfeiçoamento |
|---|---|---|
| **Pergunta** | "Já está correto?" | "O que aprendemos para não repetir?" |
| **Escopo** | Uma tarefa/rodada | Todas as tarefas futuras |
| **Mora em** | O agente em execução | A memória durável (`licoes-aprendidas.md` por agente + contratos curados) |
| **Disparado por** | Falha detectada na própria tarefa | Lição preventiva durável **comprovada** na rodada (gate de promoção) |
| **Termina** | Quando converge (com teto) | Nunca (é contínuo) |
| **Mata o problema de** | Entrega "quase pronta", retrabalho infinito | Repetir erro, conhecimento que apodrece |
| **Analogia neutra** | Refazer o trabalho ainda na mesa | Mudar o procedimento padrão |

> **Frase para fechar o tema:** "Um loop garante que o trabalho de **hoje** sai certo. O outro
> garante que o trabalho de **amanhã** começa melhor. Juntos, eles fazem o sistema convergir para a
> qualidade — e ficar mais inteligente — sem ninguém empurrando."

---

**Relacionado (base conceitual):** [Ciclo de Vida do Software](ciclo-de-vida-software.md) ·
[Sem Digitar Código](sem-digitar-codigo.md) · [Diagramas](diagramas.md) ·
[↩ Voltar ao índice da metodologia](../README.md)
</content>
