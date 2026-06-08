# Metodologia de Engenharia de Agentes do Prometeu — Índice

> **O que é este material.** Uma explicação aprofundada da **engenharia** por trás da nossa configuração
> do Claude Code: por que cada arquivo de instrução existe, que problema real de engenharia de software
> ele evita e como tudo se encaixa no processo de desenvolvimento. **Não** é um "o que é um agente" —
> isso está no site da Anthropic. Aqui o foco é o **conteúdo** dos nossos arquivos.

> 🧑‍💼 **A tese central, em uma frase.** Nós não "usamos uma IA para programar". Nós **montamos um time
> de engenharia inteiro dentro de arquivos de texto** — com filosofia, papéis especializados, jogadas
> ensaiadas e um árbitro automático — de forma que a IA seja obrigada a trabalhar como um engenheiro
> sênior disciplinado. O valor não está na ferramenta; está na **engenharia da configuração**.

---

## 1. Uma metodologia, várias lentes

A metodologia é **uma só**: os mesmos agentes, as mesmas regras, os mesmos hooks e os mesmos **dois
ciclos** (correção e aperfeiçoamento). O que muda é a **lente** — a metáfora didática usada para
ensiná-la. Cada lente é uma **apresentação completa e autônoma** (doc-mãe + capítulos + deck + roteiro
de aula). Escolha a que conversa melhor com o seu público.

| Lente | Metáfora condutora | Melhor para | Entrar |
|---|---|---|---|
| ⚽ **Futebol** | Um time profissional: DNA do clube, posições, jogadas ensaiadas, árbitro/VAR | Público amplo; abertura lúdica e memorável | [Abrir a lente Futebol →](lente-futebol/README.md) |
| 🏭 **Fábrica de Software** | Uma linha de montagem lean: norma da fábrica, estações, poka-yoke/Andon, Kaizen | Quem pensa em processo de engenharia, qualidade industrial, SDLC | [Abrir a lente Fábrica →](lente-engenharia-software/README.md) |

> 💡 **Espaço para futuras lentes.** A estrutura foi feita para crescer: basta criar uma nova pasta
> `lente-<nome>/` com a sua apresentação e reusar a base conceitual abaixo. Nada de duplicar substância.

---

## 2. A base conceitual (compartilhada por todas as lentes)

Os conceitos **transversais e neutros** — verdadeiros independentemente da metáfora — vivem **uma única
vez** em [`base-conceitual/`](base-conceitual/README.md) e são reusados por todas as lentes. É a nossa
higiene de "fonte única de verdade" aplicada à própria documentação.

| Documento | Do que trata |
|---|---|
| [Operar a Metodologia](base-conceitual/operar-a-metodologia.md) | O manual de quem dirige: fórmula do bom pedido, acompanhar/corrigir a execução, quando não implementar, checklist de aceite |
| [A Suíte como Prova](base-conceitual/suite-como-prova.md) | A suíte oficial como evidência auditável (artefatos, `run_id`, foco × fechamento) |
| [Os Dois Loops](base-conceitual/os-dois-loops.md) | Auto-correção (dentro da tarefa) × auto-aperfeiçoamento (entre tarefas) |
| [Ciclo de Vida do Software (SDLC)](base-conceitual/ciclo-de-vida-software.md) | Mapa fase-a-fase: dono, contrato e prova |
| [Travas para Projetos Grandes](base-conceitual/travas-projetos-grandes.md) | As 6 travas de escala |
| [Sem Digitar Código](base-conceitual/sem-digitar-codigo.md) | Por que o code review tradicional já está embutido |
| [Diagramas](base-conceitual/diagramas.md) · [FAQ](base-conceitual/faq-objecoes.md) · [Cartão de Referência](base-conceitual/cartao-de-referencia.md) · [Indicadores](base-conceitual/indicadores.md) | Artefatos de apoio neutros |

---

## 3. Os cinco valores que a metodologia inteira protege

Independentemente da lente, todos os arquivos existem para forçar cinco valores de engenharia:

1. **Evidência acima de suposição.** Proibido inventar API, endpoint, schema, comportamento. Só vale o
   que foi lido no código. *(Mata a alucinação — o erro nº 1 de IA em código.)*
2. **Rastreabilidade ponta a ponta.** Todo processo real tem um `correlation_id` que nasce uma vez e
   atravessa tudo; todo log é "raio-X" reconstruível. *(Mata o bug invisível em produção.)*
3. **Anti-falso-verde.** Teste passar não prova nada se o caminho oficial em runtime não usa o código
   novo. A entrega só vale ligada no boundary real. *(Mata a falsa sensação de "pronto".)*
4. **Reuso antes de criar.** Em repositório grande, solução nova é rara — quase sempre já existe.
   *(Mata a duplicação, a dívida técnica e a divergência de comportamento.)*
5. **Escopo mínimo, qualidade máxima.** "100%" qualifica a *execução* do recorte pedido, não autoriza
   inflar o escopo. *(Mata o over-engineering e a refatoração cosmética.)*

---

## 4. Glossário rápido (para a plateia não-técnica)

- **Agente (subagente):** um especialista, com cérebro e contexto próprios, que recebe uma tarefa,
  trabalha isolado e devolve só o resultado.
- **Skill / comando `/`:** o gatilho que coloca um agente específico para trabalhar. Geralmente um
  atalho fino que delega ao agente.
- **Hook:** automação do sistema que dispara sozinha em certos momentos (antes de um comando, depois de
  editar um arquivo, no fim da rodada). Não é a IA decidindo — é regra automática.
- **Nudge ("cutucada"):** um hook que **avisa, mas não bloqueia**.
- **Guard ("trava"):** um hook que **bloqueia de fato**.
- **`correlation_id`:** o número de rastreio de uma execução. Nasce uma vez e segue o processo inteiro,
  permitindo reconstruir o que aconteceu (o "raio-X").
- **Falso-verde:** quando parece pronto (teste passou, doc atualizada) mas o sistema real ainda não usa
  o código novo. O inimigo público nº 1 da metodologia.

---

## 5. Estrutura desta pasta

```
metodologia-engenharia-claude/
├── README.md                      ← você está aqui (índice-raiz)
├── base-conceitual/               ← substância neutra, compartilhada
├── lente-futebol/                 ← apresentação com metáfora de time de futebol
└── lente-engenharia-software/     ← apresentação com metáfora de fábrica de software
```

> **Sugestão de uso:** escolha a lente pelo público, leia a doc-mãe dela, e use a base conceitual para
> aprofundar os temas transversais e projetar os diagramas.
