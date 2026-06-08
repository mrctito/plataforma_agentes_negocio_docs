# Capítulo 4 — A Ordem de Produção (`.claude/skills/`)

> 🏭 [↩ Doc-mãe da lente](README.md) · [↩ Índice da metodologia](../README.md)

> **Parte da fábrica:** a skill é a **ordem de produção / o botão que aciona a estação certa**. Quando
> você digita `/investigar`, a skill não trabalha — ela **aciona** a estação `investigar` e sai da
> frente. É o documento que diz "rode este posto", não o posto em si. A maioria das nossas skills é
> deliberadamente "fina": só faz o acionamento. Algumas poucas carregam, elas mesmas, um procedimento
> obrigatório completo.

> 🧑‍💼 **RESUMO EXECUTIVO.** A separação entre "acionar a estação" (skill) e "a estação trabalhar"
> (agente) parece detalhe técnico, mas é decisão de eficiência e manutenção que economiza dinheiro:
> mantém a IA focada e barata, e permite mudar *como* uma estação trabalha sem mudar *como* ela é
> acionada. É o mesmo princípio de uma boa interface: troca-se o motor sem trocar o botão.

---

## 4.1 O padrão dominante: o acionamento fino

A grande maioria das skills segue um padrão idêntico e proposital. Veja a skill `investigar` inteira, na
prática:

> *"Esta skill é apenas o atalho de entrada (`/investigar`). Todo o conteúdo, as regras e o procedimento
> completo vivem no agente `investigar`. (…) Despache imediatamente o subagente via a ferramenta Agent
> (…). Não execute o trabalho na janela principal — o agente roda isolado ('vai e volta') para manter o
> contexto principal limpo e gastar menos tokens."*

Três linhas de instrução. Nenhuma regra de negócio. Só o acionamento. Esse é o **acionamento fino**.

**Por que é engenharia, e não preguiça?** Resolve três problemas de uma vez:

1. **Contexto limpo (e custo menor).** O trabalho pesado — ler 30 arquivos, montar um laudo de centenas
   de linhas — acontece na "célula isolada" da estação. A linha principal recebe só o resultado. Sem
   isso, cada inspeção entupiria a sessão e cada token seria pago de novo nas interações seguintes.
2. **Fonte única de verdade.** O *como* trabalhar vive **só na estação**. A skill não duplica nada. A
   própria skill `implementar` deixa explícito: *"Nada aqui duplica o agente (…) para mudar as regras
   internas, edite o agente, não esta skill."*
3. **Composição segura.** A linha principal pode encadear `/investigar` → `/planejar` → `/implementar`
   sem que uma estação saiba da outra. Cada uma é uma peça encaixável.

> 🛠️ **O ponto DRY mais elegante do projeto.** A skill é o *acionador*; o agente é o *cérebro*. Mudança
> em "quando acionar" toca a skill; mudança em "como executar" toca o agente. Cada motivo de mudança tem
> um único lugar — Princípio da Responsabilidade Única aplicado à própria configuração.

---

## 4.2 A engenharia do `description` (o cabeçalho que dispara a ordem)

Cada skill começa com um cabeçalho (`frontmatter`) com um campo `description` num formato específico:

```yaml
description: "Use when: investigar, auditar ou analisar forensemente uma feature, fluxo...
              para leitura isolada de correlation_id ou analise pura de log sem correcao,
              prefira analisar-log; ..."
```

Isso **não é decoração**. É um contrato semântico com três funções:

1. **Gatilho de auto-disparo.** O `Use when:` ensina o sistema a reconhecer, na fala do usuário, quando
   aquela ordem é a certa — mesmo sem você digitar o comando.
2. **Desambiguação explícita entre ordens parecidas.** Repare em *"para leitura isolada de correlation_id
   (…) prefira analisar-log"*. A skill **aponta para a concorrente** nos casos em que ela mesma não é a
   melhor escolha. Evita duas ordens disputando o mesmo pedido.
3. **Exclusão de casos errados.** Palavras-chave negativas ("sem corrigir", "sem planejar") delimitam a
   fronteira da ordem.

**Defeito que evita:** a ferramenta errada para o trabalho — inspecionar quando era para corrigir,
planejar quando era para só medir. Numa fábrica de 22 postos, saber **qual acionar** é metade da
qualidade.

---

## 4.3 As skills que fogem do padrão (procedimento próprio robusto)

Nem toda skill é fina. Algumas — quase todas de **ensaio/validação de fluxo real** — carregam um
procedimento completo embutido: `corrigir-com-log`, `auditoria-suite-testes`, `testar-webchat`,
`testar-webchat-dnit`, `testar-nl2sql`, `testar-nl2yaml`, `testar-paginas-web-projeto`, e a referência
técnica `playwright-cli`.

**Por que essas são "robustas"?** Porque o procedimento delas é um **ciclo obrigatório e iterativo** que
**não pode ser violado por conveniência**: enviar o pedido real → capturar o `correlation_id` da response
→ ler o registro daquela correlação → aplicar o método de correção por log → reexecutar se falhou.

Colocar esse ciclo na **skill** (e não só no agente) é deliberado: a skill é o **procedimento imutável**.
Ela documenta, no nível do sistema, que aquelas etapas são obrigatórias — um piso que a estação não pode
rebaixar para "ir mais rápido". É o equivalente a um **procedimento operacional crítico afixado na
máquina**, que o operador não tem autoridade para encurtar.

> 🛠️ **A regra prática:** procedimento que **pode evoluir** → vive na estação, e a skill é fina.
> Procedimento que **não pode ser pulado** (loop de correção, captura de correlation_id, três provas) → é
> fixado na própria skill como contrato. A escolha entre fino e robusto é de *governança*, não de tamanho.

---

## 4.4 Skill vs. Agent vs. Command — o quadro que tira a dúvida

| Conceito | Papel na fábrica | Onde mora a inteligência |
|---|---|---|
| **Command** (`/comando` clássico) | Um carimbo de texto na conversa atual | Nenhuma — prompt pronto, roda no mesmo contexto |
| **Skill** | A ordem de produção `/` (gatilho + contrato de acionamento) | Pouca (acionamento fino) ou o procedimento imutável (skill robusta) |
| **Agent** | A estação (cérebro + contexto isolado) | Toda a regra de execução |

Nosso projeto **não usa `.claude/commands/`** — e é intencional. As skills cumprem o papel do command
(ser o gatilho `/`) com vantagens: auto-disparo pela `description` e delegação a uma estação isolada.
Adicionar commands seria um **terceiro mecanismo paralelo** para a mesma função — exatamente a duplicação
que o `CLAUDE.md §6` proíbe.

---

## 4.5 O que levar deste capítulo

- A skill é a **ordem de produção**, não a estação. A inteligência mora no agente.
- O **acionamento fino** entrega três coisas: contexto limpo, custo menor e fonte única de verdade (zero
  duplicação skill↔agente).
- O `description` em formato `Use when:` é um **contrato semântico** que dispara a ordem certa e
  desambigua das parecidas.
- Skills **robustas** existem para **fixar procedimentos que não podem ser pulados** — governança, não
  tamanho.
- Não usamos commands de propósito: as skills já fazem o papel, melhor, sem duplicar mecanismo.

**Próximo:** [Capítulo 5 — Poka-yoke e Andon (`.claude/hooks/`)](capitulo-05-pokayoke-e-andon.md).
