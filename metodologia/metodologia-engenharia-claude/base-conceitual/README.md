# Base Conceitual — A Substância Neutra (compartilhada por todas as lentes)

> **O que é esta pasta.** Aqui mora o conteúdo **transversal e neutro** da metodologia — aquilo que é
> verdadeiro **independentemente da metáfora** usada para ensinar. As apresentações ("lentes") contam a
> mesma história com linguagens diferentes (⚽ futebol, 🏭 fábrica de software, e futuras); para **não
> duplicar substância**, os conceitos profundos vivem **uma única vez** aqui, e cada lente apenas os
> **enquadra** e aponta para cá.

> 🧑‍💼 **Por que separar assim.** É a mesma higiene que cobramos do código: fonte única de verdade,
> zero duplicação. Se amanhã criarmos uma terceira lente, ela reusa esta base — não recopia. E se um
> conceito mudar, muda em **um** lugar, e todas as lentes ficam corretas de imediato.

---

## Conteúdo da base

| Documento | Do que trata | Quando consultar |
|---|---|---|
| [Operar a Metodologia](operar-a-metodologia.md) | O manual de quem **dirige**: a fórmula do bom pedido, acompanhar/corrigir a execução, quando **não** implementar ainda, checklist de aceite | Para aprender **como conduzir** uma mudança por linguagem natural |
| [A Suíte como Prova](suite-como-prova.md) | A suíte oficial como **evidência auditável**: artefatos, `run_id`, foco × fechamento, o que fazer ao falhar | Para entender **como a validação vira prova**, não opinião |
| [Os Dois Loops](os-dois-loops.md) | Auto-correção (dentro da tarefa) × auto-aperfeiçoamento (entre tarefas) — opostos e complementares; inclui **quando registrar cada aprendizado** | Para entender **como o sistema converge e como aprende** |
| [Ciclo de Vida do Software (SDLC)](ciclo-de-vida-software.md) | Mapa fase-a-fase: dono, contrato e prova de cada etapa | Para responder *"isso cobre o processo inteiro ou só escrever código?"* |
| [Travas para Projetos Muito Grandes](travas-projetos-grandes.md) | As 6 travas de escala (âncora, fatia, anti-varredura, isolamento, carregamento, handoff) | Para entender por que dá para usar IA num repo gigante sem estourar custo |
| [Desenvolver e Corrigir Sem Digitar Código](sem-digitar-codigo.md) | A aposta central: por que o *code review* tradicional já está embutido no processo | O tema **mais estratégico** — para liderança e céticos |
| [Diagramas](diagramas.md) | 4 diagramas Mermaid neutros (loops, correção, decisão de skill, camadas sobre o SDLC) | Para projetar e consultar os fluxos |
| [FAQ de Objeções](faq-objecoes.md) | Respostas honestas às perguntas difíceis (com os limites declarados) | Para defender a abordagem e na seção de perguntas |
| [Cartão de Referência](cartao-de-referencia.md) | 1 página: comando → garante → pré-requisito + sinais de alerta + checklist de extensão | Uso diário e onboarding |
| [Indicadores](indicadores.md) | O que medir (resultado, aprendizado, integridade) para provar e auditar a metodologia | Reuniões de liderança e melhoria contínua |

---

## Como as lentes usam esta base

- A **lente** explica os artefatos (`CLAUDE.md`, `rules/`, `agents/`, `skills/`, `hooks/`) na sua
  metáfora própria e demonstra os dois loops **em ação**.
- A **base** guarda a definição profunda e neutra desses mesmos conceitos.
- Regra prática: se um conceito vale **em qualquer lente**, ele pertence aqui; se é a **forma de contar**
  (a metáfora, o roteiro, o deck), pertence à lente.

> Diagramas puramente decorativos de uma metáfora específica (a formação tática da lente futebol, o
> layout da linha da lente fábrica) ficam **dentro de cada lente** — esta base só guarda os diagramas
> que valem para todas.

---

**Navegação:** [↩ Voltar ao índice da metodologia](../README.md) ·
[⚽ Lente Futebol](../lente-futebol/README.md) ·
[🏭 Lente Fábrica de Software](../lente-engenharia-software/README.md)
