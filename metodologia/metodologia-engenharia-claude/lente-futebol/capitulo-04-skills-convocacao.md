# Capítulo 4 — A Convocação (`.claude/skills/`)

> **Posição em campo:** a skill é o **apito que coloca o jogador certo em campo**. Quando você digita
> `/investigar`, a skill não joga — ela **convoca** o agente `investigar` e sai da frente. É a comissão
> que decide quem entra, não quem dá o passe decisivo. A maioria das nossas skills é deliberadamente
> "fina": só faz a convocação. Algumas poucas carregam, elas mesmas, uma tática de jogo completa.

> 🧑‍💼 **RESUMO EXECUTIVO.** A separação entre "chamar o especialista" (skill) e "o especialista
> trabalhar" (agente) parece um detalhe técnico, mas é uma decisão de eficiência e manutenção que
> economiza dinheiro: mantém a IA focada e barata, e permite mudar *como* um especialista trabalha
> sem mudar *como* ele é chamado. É o mesmo princípio de uma boa interface: você troca o motor sem
> trocar o volante.

---

## 4.1 O padrão dominante: o atalho fino

A grande maioria das nossas skills segue um padrão idêntico e proposital. Veja a skill `investigar`
inteira, na prática:

> *"Esta skill é apenas o atalho de entrada (`/investigar`). Todo o conteúdo, as regras e o
> procedimento completo vivem no agente `investigar`. (…) Despache imediatamente o subagente via a
> ferramenta Agent (…). Não execute o trabalho na janela principal — o agente roda isolado ('vai e
> volta') para manter o contexto principal limpo e gastar menos tokens."*

Três linhas de instrução. Nenhuma regra de negócio. Só a convocação. Esse é o **atalho fino**.

**Por que isso é engenharia, e não preguiça?** Porque resolve três problemas de uma vez:

1. **Contexto limpo (e custo menor).** O trabalho pesado — ler 30 arquivos, montar um relatório de
   centenas de linhas — acontece na "sala separada" do agente. A conversa principal recebe só o
   resultado. Sem isso, cada investigação entupiria a sessão e cada token seria pago de novo nas
   interações seguintes.
2. **Fonte única de verdade.** O *como* trabalhar vive **só no agente**. A skill não duplica nada. Se
   a skill repetisse as regras, teríamos dois lugares para manter sincronizados — e eles divergiriam.
   A própria skill `implementar` deixa isso explícito: *"Nada aqui duplica o agente (…) para mudar as
   regras internas, edite o agente, não esta skill."*
3. **Composição segura.** A janela principal pode encadear `/investigar` → `/planejar` → `/implementar`
   sem que um agente saiba do outro. Cada um é uma peça encaixável.

> 🛠️ **O ponto DRY mais elegante do projeto.** A skill é o *gatilho*; o agente é o *cérebro*.
> Mudança em "quando invocar" toca a skill; mudança em "como executar" toca o agente. Cada motivo de
> mudança tem um único lugar — isso é o Princípio da Responsabilidade Única aplicado à própria
> configuração da IA.

---

## 4.2 A engenharia do `description` (o frontmatter que dispara o jogo)

Cada skill começa com um cabeçalho (`frontmatter`) com um campo `description` escrito num formato
muito específico:

```yaml
description: "Use when: investigar, auditar ou analisar forensemente uma feature, fluxo...
              para leitura isolada de correlation_id ou analise pura de log sem correcao,
              prefira analisar-log; ..."
```

Isso **não é decoração**. É um contrato semântico com três funções de engenharia:

1. **Gatilho de auto-disparo.** O `Use when:` ensina o sistema a reconhecer, na fala do usuário,
   quando aquela skill é a certa — mesmo sem você digitar o comando.
2. **Desambiguação explícita entre skills parecidas.** Repare no trecho *"para leitura isolada de
   correlation_id (…) prefira analisar-log"*. A skill **aponta para a concorrente** nos casos em que
   ela mesma não é a melhor escolha. Isso evita o atrito de duas skills disputando o mesmo pedido.
3. **Exclusão de casos errados.** Palavras-chave negativas ("sem corrigir", "sem planejar") delimitam
   a fronteira da skill.

**Problema que evita:** a ferramenta errada para o trabalho — investigar quando era para corrigir,
planejar quando era para só analisar. Em um time de 22 especialistas, saber **quem chamar** é metade
da qualidade.

---

## 4.3 As skills que fogem do padrão (conteúdo próprio robusto)

Nem toda skill é fina. Algumas — quase todas de **teste/validação de fluxo real** — carregam um
procedimento completo embutido: `corrigir-com-log`, `auditoria-suite-testes`, `testar-webchat`,
`testar-webchat-dnit`, `testar-nl2sql`, `testar-nl2yaml`, `testar-paginas-web-projeto`, e a referência
técnica `playwright-cli`.

**Por que essas são "gordas"?** Porque o procedimento delas é um **ciclo obrigatório e iterativo** que
**não pode ser violado por conveniência**: enviar a pergunta real → capturar o `correlation_id` da
response → ler o log daquela correlação → aplicar o método de correção por log → reexecutar se falhou.

Colocar esse ciclo na **skill** (e não só no agente) é uma decisão deliberada: a skill é o **contrato
imutável** do procedimento. Ela documenta, no nível do sistema, que aquelas etapas são obrigatórias —
um piso que o agente não pode rebaixar para "ir mais rápido".

> 🛠️ **A regra prática:** procedimento que **pode evoluir** → vive no agente, e a skill é fina.
> Procedimento que **não pode ser pulado** (loop de correção, captura de correlation_id, três provas)
> → é fixado na própria skill como contrato. A escolha entre fino e gordo é uma escolha de
> *governança*, não de tamanho.

---

## 4.4 Skill vs. Agent vs. Command — o quadro que tira a dúvida

A pergunta clássica: "isso devia ser uma skill, um agente ou um command?". Como nossa engenharia
respondeu, na prática:

| Conceito | Papel no time | Onde mora a inteligência |
|---|---|---|
| **Command** (`/comando` clássico) | Um carimbo de texto na conversa atual | Nenhuma — é só um prompt pronto, roda no mesmo contexto |
| **Skill** | A convocação `/` (gatilho + contrato de despacho) | Pouca (atalho fino) ou o procedimento imutável (skill gorda) |
| **Agent** | O jogador (cérebro + contexto isolado) | Toda a regra de execução |

Nosso projeto **não usa `.claude/commands/`** — e isso é intencional. As skills cumprem o papel do
command (ser o gatilho `/`) com vantagens: auto-disparo pela `description` e delegação a um agente
isolado. Adicionar commands seria um **terceiro mecanismo paralelo** para a mesma função — exatamente
o tipo de duplicação que o `CLAUDE.md §6` proíbe.

---

## 4.5 O que levar desta posição para a aula

- A skill é o **apito de convocação**, não o jogador. A inteligência mora no agente.
- O **atalho fino** entrega três coisas: contexto limpo, custo menor e fonte única de verdade (zero
  duplicação skill↔agente).
- O `description` em formato `Use when:` é um **contrato semântico** que dispara a skill certa e
  desambigua das parecidas.
- Skills **gordas** existem para **fixar procedimentos que não podem ser pulados** — governança, não
  tamanho.
- Não usamos commands de propósito: as skills já fazem o papel, melhor, sem duplicar mecanismo.

**Próximo:** [Capítulo 5 — A Arbitragem (`.claude/hooks/`)](capitulo-05-hooks-arbitragem.md).
</content>
