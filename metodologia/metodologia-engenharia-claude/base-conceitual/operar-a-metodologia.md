# Operar a Metodologia — O Manual de Quem Dirige (sem digitar código)

> **Onde este documento se encaixa.** Os outros materiais explicam *como a máquina é construída por
> dentro* (agentes, regras, hooks) e *por que é seguro confiar em código que você não revisou linha a
> linha* — isso está em [Sem Digitar Código](sem-digitar-codigo.md). **Este documento é a outra metade:
> a práxis.** Não é "por que funciona"; é **"como você dirige"** — como formular o pedido, acompanhar a
> execução, corrigir o rumo e saber a hora de parar. É o manual do piloto, não o manual do motor.

> 🧑‍💼 **RESUMO EXECUTIVO.** A qualidade desta metodologia **não vem de confiar cegamente na IA** — vem
> da **governança**. E governança tem uma habilidade central que pode ser aprendida: **dar um bom
> pedido e exigir prova**. Quem aprende isso conduz uma mudança inteira por linguagem natural, com mais
> rigor do que teria digitando à mão. Quem não aprende transforma uma ferramenta poderosa em gerador de
> retrabalho. Este documento ensina essa habilidade.

---

## 1. A virada mental: de "que arquivo eu edito?" para "que agente conduz isto?"

O erro do recém-chegado é abrir um arquivo para mexer. Aqui, o trabalho **não começa no editor** —
começa **formulando a intenção**. Você deixa de ser o editor de código e passa a ser o **coordenador de
uma pequena equipe de especialistas**: você dá a direção, eles executam a parte mecânica (investigar,
planejar, implementar, testar, documentar), e você **exige a prova** a cada passo.

Essa troca de pergunta — *"em que arquivo eu mexo?"* → *"qual agente deve conduzir esta etapa?"* — é o
primeiro hábito a adquirir. Ela força a separação de papéis que protege a qualidade: investigação antes
da ação, plano antes da mudança arriscada, implementação com validação, correção com log, aprendizado
registrado.

---

## 2. A fórmula do bom pedido (5 partes)

Um pedido vago produz um resultado vago. Um pedido com estas **cinco partes** produz trabalho
auditável. Esta é a ferramenta mais prática deste documento — vale a pena decorar.

| Parte | Pergunta que responde | Exemplo conceitual |
|---|---|---|
| **1. Objetivo** | Que resultado concreto eu quero? | "Corrigir a ingestão de PDFs que falha de forma intermitente." |
| **2. Escopo** | Onde a mudança pode acontecer? | "No fluxo de ingestão; não tocar no RAG nem na UI." |
| **3. Restrições** | O que **não** pode acontecer? | "Sem fallback que mascare o erro; sem manter caminho legado." |
| **4. Evidência** | Que fatos eu já tenho? | "O `correlation_id` da última falha é X; o sintoma é Y." |
| **5. Critério de aceite** | Como eu sei que terminou? | "Suíte verde no slice + log prova o fluxo ponta a ponta sem erro." |

**Regra de ouro:** se você **não tem a Evidência (parte 4)**, não peça implementação — peça
**investigação primeiro**. Pedir para consertar sem evidência é convidar o chute, exatamente o que a
metodologia existe para impedir.

> 🛠️ Por que isto é engenharia, e não burocracia: as cinco partes mapeiam diretamente nos contratos
> internos. O **Objetivo** vira o objetivo falsificável do plano; o **Escopo** alimenta o "escopo
> mínimo"; as **Restrições** viram a seção `# Estratégia/Recomendacoes`; a **Evidência** é o que o
> `investigar` exige (path/linha, log); o **Critério de aceite** é o que o `validar-entrega` vai cobrar.
> Um bom pedido já nasce alinhado à máquina.

---

## 3. Acompanhar uma execução — o que observar (sem ler o código)

Durante a execução, você não confere sintaxe. Você confere **disciplina de processo**. Sinais de que
está indo bem:

- leu os arquivos **antes** de editar (não saiu chutando);
- criou uma **lista de tarefas** quando a mudança tem mais de uma etapa;
- **explicou** as edições antes de fazê-las;
- **validou** o resultado e **leu** a saída dos testes (não declarou "feito" pelo exit code);
- **registrou bloqueios** em vez de esconder falhas;
- atualizou documentação e registros quando aplicável.

**Não aceite uma conclusão baseada apenas em "feito". Peça a evidência.** ("Qual boundary foi validado?
Qual teste protege? Cadê o log do fluxo?") A prova é sua âncora — ver [A Suíte como Prova](suite-como-prova.md)
e a [Definição de Pronto](../README.md).

---

## 4. Quando NÃO implementar ainda (sinais de parada)

Tão importante quanto saber pedir é saber **segurar**. Pare e peça **investigação** (não implementação)
quando:

- o comportamento real **não foi lido no código** — só suposto;
- o **log não mostra a causa** do erro;
- a mudança depende de um **campo YAML não confirmado**;
- há dúvida sobre **contrato público** (endpoint, payload, schema);
- a mudança afeta **banco, filas, workers, scheduler, agentes ou runtime crítico**;
- a **documentação e o código parecem divergir**.

Implementar sem evidência quase sempre gera mais trabalho depois. O custo de investigar primeiro é
quase sempre menor que o custo de desfazer uma implementação errada.

---

## 5. Como corrigir o rumo (quando a IA interpreta errado)

Corrigir o agente **não é atrito — é parte do método**. Seja direto e cirúrgico:

1. **Diga qual premissa está errada** ("você assumiu que X existe; não existe").
2. **Aponte a evidência que contradiz** ("o arquivo Y, linha Z, mostra o contrário").
3. Se o erro **pode se repetir**, peça para **registrar uma lição** ([os dois loops](os-dois-loops.md)).
4. Se a causa foi uma **instrução ambígua/contraditória**, peça para registrar em `bad-instructions`.

Quanto mais específica a correção (premissa + evidência), mais rápido o rumo se ajusta — e menos a
mesma confusão volta.

---

## 6. O que "100% IA" significa (e o que não significa)

**Significa** que a IA executa a mudança material: edita arquivos, cria testes, roda comandos, lê logs,
atualiza documentação e registra resultados.

**Não significa** que o humano abandona a responsabilidade técnica. O humano continua dono de: dar
direção, decidir prioridade, fornecer contexto, **validar se o resultado atende ao objetivo**, exigir
provas e **interromper quando o processo sai do trilho**.

> A leitura honesta: "sem digitar código" **não** é "sem engenharia". O esforço migra da digitação para
> a **curadoria do pedido e a leitura crítica da evidência** — um trabalho mais alavancado, não
> inexistente. (Aprofundado em [Sem Digitar Código](sem-digitar-codigo.md).)

---

## 7. Checklist de bolso — antes de aceitar uma entrega

- [ ] o **objetivo original** foi atendido;
- [ ] os **arquivos alterados** fazem sentido para o escopo (nada fora do pedido);
- [ ] **validações** foram executadas — ou o **bloqueio** foi explicado com evidência;
- [ ] **links, diagramas e documentação** foram verificados quando aplicável;
- [ ] os **registros obrigatórios** (lição/erro/regressão/encerramento) foram atualizados quando cabível;
- [ ] os **riscos residuais** foram declarados.

---

## 8. Resumo de bolso

- O trabalho começa na **intenção**, não no editor. Você coordena agentes; não digita código.
- O **bom pedido tem 5 partes**: objetivo, escopo, restrições, evidência, critério de aceite.
- **Sem evidência → peça investigação**, não implementação.
- Você confere **disciplina e prova**, não sintaxe. Nunca aceite "feito" sem evidência.
- Saber **segurar** (não implementar ainda) é tão valioso quanto saber pedir.
- A garantia vem da **governança**, não da confiança cega.

> **Frase para fechar o tema:** "Eu não escrevo o código. Eu escrevo o **pedido certo**, exijo a
> **prova certa** e sei a **hora de parar**. O resto, a máquina executa — sob contrato."

---

**Relacionado:** [Sem Digitar Código](sem-digitar-codigo.md) · [A Suíte como Prova](suite-como-prova.md) ·
[Os Dois Loops](os-dois-loops.md) · [Cartão de Referência](cartao-de-referencia.md) ·
[Ciclo de Vida do Software](ciclo-de-vida-software.md) · [↩ Índice da metodologia](../README.md)
