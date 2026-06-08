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
  vive: dentro do agente em execução     vive: na memória durável (rules/)
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

**O que é:** um ciclo que **atravessa sessões e tarefas** — o sistema lembra das lições passadas no
começo e registra as novas no fim, fazendo o conhecimento se acumular na empresa.

**Onde ele vive (evidência real no projeto):**

```
   INÍCIO DA SESSÃO                                FIM DA SESSÃO
   hook: session-start.sh                          hook: stop-loop-reminder.sh
   ── lê e injeta lessons.md no contexto           ── cobra registrar lição/erro/regressão
      (a sessão começa já relembrada)                 (a sessão termina tendo registrado)
            │                                                    │
            └──────────────── lado "LER" ── + ── lado "ESCREVER" ┘
                                       │
                          MEMÓRIA DURÁVEL (.claude/rules/)
                  lessons.md · error-backlog.md · regression-logs.md · bad-instructions.md
```

**Os quatro destinos do aprendizado (cada um com função distinta):**

| Arquivo | Captura | Regra de qualidade |
|---|---|---|
| `lessons.md` | Lições com **valor preventivo transversal** | Só entra se "outro agente, amanhã, em outro slice, ainda evitaria erro com isso" |
| `error-backlog.md` | Erro real de produto | Verifica duplicidade antes de registrar |
| `regression-logs.md` | Erro que **voltou** | Tratado como regressão grave — investiga por que a correção anterior falhou |
| `bad-instructions.md` | Defeito nas **próprias instruções** | Alimentado pelo agente `validar-instructions` |

**A trava de curadoria (por que o `lessons.md` não vira lixo):** existe uma **pergunta-porteiro**
obrigatória antes de promover qualquer lição — *"se outro agente generalista atuar amanhã em outro
slice, esta regra ainda reduziria chance real de erro?"*. Se a resposta é não, a lição fica na memória
local da skill, não polui a memória global. Isso impede o problema clássico de "wiki que ninguém lê
porque está cheia de detalhe irrelevante".

**Problema de engenharia que evita:** o sistema repetir o mesmo erro em slices diferentes; e o
conhecimento morar na cabeça de poucos. É a **capitalização de conhecimento** automatizada.

**Valor:** melhoria contínua **institucional** — a empresa fica mais competente sozinha, por
construção, sem depender de alguém lembrar de documentar.

---

## A.3.1 O quinto destino: o registro de encerramento (o rastro da tarefa)

Os quatro arquivos acima capturam **aprendizado**. Falta um destino que captura **a própria entrega**:
o **registro de encerramento** (exigido pela [Definição de Pronto](../README.md), `§12`). Ele conecta
pedido → alteração → raio de impacto → testes executados → resultado → status final (`SUCESSO` ou
`BLOQUEADA`). Não é "mais um log" — é o que faz uma entrega **deixar de viver só no chat e virar
evidência versionada**, ligando o trabalho de hoje ao histórico que os outros quatro arquivos consultam
amanhã.

> 🛠️ Por que isto importa para o loop: sem o registro de encerramento, o loop de auto-aperfeiçoamento
> fica cego sobre **o que já foi tentado e entregue**. Com ele, uma regressão futura pode ser cruzada
> com a entrega que a introduziu — fechando o elo entre "o que mudou" e "o que voltou a quebrar".

## A.3.2 Gatilho rápido: na dúvida, qual arquivo?

| Aconteceu… | Registre em |
|---|---|
| O usuário corrigiu uma interpretação, ou surgiu um padrão que **pode voltar em outro slice** | `lessons.md` |
| Um **erro real de runtime** (não falha de teste) afetou produto, API, worker, ingestão, RAG… | `error-backlog.md` |
| O **mesmo** erro/sintoma/componente **voltou** | `regression-logs.md` |
| Uma **instrução** se mostrou contraditória, ambígua, ampla demais ou sem critério de validação | `bad-instructions.md` |
| Uma **tarefa fechou** (com ou sem bloqueio) | registro de encerramento |

**Anti-exemplos (o erro × o certo):**

- ❌ Corrigir um erro real e **não** registrar no backlog. ✅ Registrar erro, correção, evidência e validação.
- ❌ Tratar um erro recorrente como **caso novo**. ✅ Registrar regressão e **questionar a abordagem anterior**.
- ❌ Engolir uma instrução contraditória. ✅ Registrar em `bad-instructions` para revisão.
- ❌ Encerrar a tarefa com um "feito" solto. ✅ Registrar encerramento com impacto, testes e status.

> Regra anti-filler: registre **lição** apenas quando houver valor preventivo transversal; ajuste
> trivial sem impacto sistêmico **não** vira lição. O loop melhora porque é **curado**, não porque
> acumula tudo.

---

## A.4 A ponte entre os dois loops (onde eles se encontram)

Os dois loops não são ilhas. Há um ponto de contato deliberado:

> Quando o **loop de auto-correção** de uma rodada descobre uma regra preventiva reaproveitável, essa
> lição é **promovida** para o **loop de auto-aperfeiçoamento** (vai para o `lessons.md`).

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
| **Mora em** | O agente em execução | A memória durável (`rules/`) |
| **Disparado por** | Falha detectada na própria tarefa | Hooks de início e fim de sessão |
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
