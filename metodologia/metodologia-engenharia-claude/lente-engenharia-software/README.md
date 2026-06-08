# A Fábrica de Software — Metodologia de Engenharia de Agentes do Prometeu

> 🏭 **Lente Fábrica de Software** · [↩ Índice da metodologia](../README.md) ·
> [⚽ Lente Futebol](../lente-futebol/README.md) ·
> [📚 Base conceitual](../base-conceitual/README.md)

> **O que é este material.** É a **mesma metodologia** explicada por outra lente. A lente futebol usa um
> time como metáfora; esta usa uma **fábrica enxuta (lean) de software** — uma linha de montagem com
> estações especializadas, dispositivos à prova de erro e melhoria contínua. Aqui a metáfora é quase uma
> **sobreposição**: os conceitos de manufatura de qualidade (poka-yoke, Andon, Jidoka, Kaizen,
> rastreabilidade de lote) mapeiam quase 1:1 nos conceitos de engenharia de software (segregação de
> funções, fail-fast, anti-falso-verde, melhoria contínua, observabilidade).

---

## 0. Como ler este material (duas camadas)

Como a lente futebol, este material tem **duas camadas** — escolha a profundidade:

- 🧑‍💼 **Camada executiva (liderança, produto, comercial):** os blocos `RESUMO EXECUTIVO`. Falam de
  risco, custo, qualidade, previsibilidade e velocidade — sem jargão.
- 🛠️ **Camada técnica (engenharia):** o corpo de cada capítulo. Fala de SOLID, clean code,
  observabilidade, contratos, anti-duplicação, falso-verde.

> 🧑‍💼 **RESUMO EXECUTIVO (a tese central).** Nós não "usamos uma IA para programar". Nós **montamos uma
> linha de produção de software dentro de arquivos de texto** — com norma de qualidade, estações
> especializadas, dispositivos à prova de erro e um sistema de parar-a-linha — de forma que a IA seja
> obrigada a produzir como uma **fábrica madura com controle de qualidade**, não como um artesão
> apressado. O valor não está na máquina; está na **engenharia do processo de produção**.

---

## 1. A virada de chave: instrução não é documentação, é processo de produção

A maioria das empresas usa IA de programação como **produção artesanal**: "faz essa peça pra mim".
Funciona às vezes, falha de forma imprevisível, e o defeito só aparece no cliente.

A nossa abordagem trata a configuração do Claude como **uma fábrica enxuta**: cada arquivo é uma parte
do sistema de produção, com função, padrão e controle. A diferença entre "usar IA" e "fazer engenharia
com IA" é a mesma entre o artesão e a linha: **a fábrica não depende de um lampejo de genialidade;
depende de cada estação cumprir seu padrão.** É isso que troca resultado imprevisível por resultado
confiável, repetível e auditável.

| Na fábrica de software (lean) | Na nossa engenharia | O que isso materializa |
|---|---|---|
| Norma da fábrica / Manual de Qualidade mestre | `CLAUDE.md` — lido em 100% das sessões | Padrão de qualidade como lei aplicada, não preferência |
| Especificações de processo / Instruções de Trabalho (SOPs) | `.claude/rules/` — contratos profundos sob demanda | Procedimento detalhado fora do padrão enxuto |
| Estações da linha (operadores especializados) | `.claude/agents/` — 22 especialistas | SRP, segregação de funções, contexto isolado |
| Ordem de produção / botão que aciona a estação | `.claude/skills/` — gatilhos `/comando` | Disparo fino separado do "cérebro" da estação |
| Poka-yoke + Andon + sensores automáticos | `.claude/hooks/` — guards e nudges | Automação determinística: travar vs. alertar |
| Roteiro de fabricação / layout mestre da linha | `.claude/settings.json` | Amarra tudo: qual dispositivo dispara em cada momento |
| Número de série / rastreabilidade de lote | `correlation_id` | Genealogia do produto, rastreabilidade ponta a ponta |
| Registro de processo (batch record) + inspeção não-destrutiva | logs canônicos + o "raio-X" | Observabilidade por construção, debug offline |

---

## 2. O layout da linha de produção

```
                          ┌─────────────────────────────────┐
                          │   NORMA DA FÁBRICA (padrão)      │
                          │            CLAUDE.md             │
                          │   lida em 100% das sessões       │
                          └─────────────────────────────────┘
                                       │ rege
                                       ▼
  POKA-YOKE + ANDON (hooks) ░░░ dispositivos à prova de erro, sempre ligados ░░░
  bash-guard · write-guard · py-lint · logging-nudge · py-discipline · stop-loop

   INSPEÇÃO        PCP            FABRICAÇÃO        CONTROLE
   DE ENTRADA      (roteiro)      (montagem)        DE QUALIDADE
   investigar      planejar       implementar       validar-entrega
   (mede a         (roteiro de    (produz a peça    (aprova/rejeita
    matéria-prima)  fabricação)    boa, ativa)       o lote)

   ── laboratório de metrologia e qualidade (observabilidade) ──
   analisar-log · validar-log · validar-observabilidade · corrigir-erros-com-log · testar-cli-log-analyzer

   ── manutenção e operação do chão de fábrica (infra real) ──
   gerenciar-postgresql · gerenciar-redis · gerenciar-job-core-ingestao · gerenciar-scheduler

   ── ensaios e lote piloto (testes de fluxo real) ──
   executar-testes · criar-testes · testar-ingestao-dnit · testar-cancelamento · testar-webchat · testar-nl2sql · testar-nl2yaml · testar-paginas-web

   ── engenharia da qualidade e documentação (governança/conhecimento) ──
   documentar · sincronizar-documentacao · inventario-componentes · inventario-yaml · validar-instructions · analisar-produto

   ── especificações de processo / SOPs (rules sob demanda) ──
   log-instructions · reuso-instructions · suite-testes · definicao-de-pronto · estrategia-recomendacoes · python · large-repo · fidelidade-pedido
   + registro de melhoria contínua: lessons · error-backlog · regression-logs · bad-instructions
```

---

## 3. A célula de produção — o fluxo que sempre se repete

Se você guardar **uma só coisa**, guarde esta célula de manufatura. É o fluxo oficial de quase todo
trabalho não-trivial:

```
  /investigar  ──►  /planejar  ──►  /implementar  ──►  /validar-entrega
  INSPEÇÃO DE       PCP              FABRICAÇÃO          CONTROLE DE
  ENTRADA          (roteiro de      (peça boa,          QUALIDADE
  (mede a          fabricação)      ativa no            (aprova ou
   matéria-prima)                   runtime)            rejeita o lote)
```

O que faz disso engenharia, e não enfeite:

1. **Segregação de funções levada ao extremo.** Quem inspeciona a entrada **não** planeja a produção;
   quem planeja **não** fabrica; quem fabrica **não** assina o próprio controle de qualidade. Cada
   passagem de bastão é uma barreira contra o erro mais comum da IA: pular da ideia direto para a peça,
   sem medir e sem inspecionar.
2. **Cada estação entrega à próxima com contrato.** A inspeção de entrada vira a entrada obrigatória do
   PCP; o roteiro vira a entrada obrigatória da fabricação; a peça vira a entrada obrigatória do
   controle de qualidade. Nada "se perde na esteira".
3. **O controle de qualidade pode rejeitar o lote.** `validar-entrega` tem autoridade para **reprovar** e
   devolver ao PCP (`planejar`). É o adversário interno da pressa — o inspetor que não carimba sem ler.

> 🧑‍💼 **RESUMO EXECUTIVO.** Esta célula é o nosso controle de qualidade industrial. Troca "a IA fez,
> deve estar certo" por "quatro estações independentes assinaram, cada uma com evidência". O custo é
> mais passos; o ganho é **previsibilidade** — menos retrabalho, menos defeito no cliente, menos surpresa.

---

## 4. Os cinco valores que a fábrica inteira protege

Todos os arquivos, em todas as estações, existem para forçar cinco valores — a "política de qualidade"
da fábrica:

1. **Evidência acima de suposição.** Proibido inventar API, endpoint, schema, comportamento. Só vale o
   que foi **medido** lendo o código. *(Mata a alucinação — o refugo nº 1 de IA em código.)*
2. **Rastreabilidade de lote ponta a ponta.** Todo processo real tem um `correlation_id` (número de
   série) que nasce uma vez e segue a peça por toda a linha; todo log é registro de processo
   reconstruível. *(Mata o defeito invisível que só aparece no cliente.)*
3. **Anti-falso-verde.** Peça aprovada no ensaio isolado não prova nada se a **linha oficial** em runtime
   não usa a peça nova. A entrega só vale **instalada no produto real**. *(Mata a falsa sensação de
   "pronto".)*
4. **Reuso antes de fabricar.** Em fábrica madura, peça nova é rara — quase sempre o componente já existe
   no almoxarifado. *(Mata a duplicação, o excesso de estoque de código e a divergência de comportamento.)*
5. **Escopo mínimo, qualidade máxima.** "100%" qualifica a *execução* do lote pedido, não autoriza
   produzir o que ninguém encomendou. *(Mata o over-engineering e a refatoração cosmética.)*

---

## 5. Índice dos capítulos (cada um aprofunda uma parte da fábrica)

| Capítulo | Parte da fábrica | O que você vai entender |
|---|---|---|
| [Cap. 1 — A Norma da Fábrica](capitulo-01-norma-da-fabrica.md) | O padrão mestre (`CLAUDE.md`) | Por que cada uma das 17 seções existe e que valor força |
| [Cap. 2 — As Especificações de Processo](capitulo-02-especificacoes-processo.md) | Os SOPs (`rules/`) | Os contratos profundos e o registro de melhoria contínua |
| [⭐ Cap. 3 — As Estações da Linha](capitulo-03-estacoes-da-linha.md) | Os 22 agentes | Função, papel e o defeito que cada estação evita |
| [Cap. 4 — A Ordem de Produção](capitulo-04-ordem-de-producao.md) | As skills (`/comando`) | Por que separar o acionamento do cérebro da estação |
| [Cap. 5 — Poka-yoke e Andon](capitulo-05-pokayoke-e-andon.md) | Hooks e dispositivos | Automação determinística: parar a linha vs. alertar |
| [⭐ Cap. 6 — Um Lote na Linha](capitulo-06-um-lote-na-linha.md) | A fábrica em ação | Um pedido do recebimento à expedição, com os dois ciclos atuando |

> Os capítulos ⭐ concentram o foco pedido nesta lente: **as estações (agentes)** e **os dois ciclos**
> (Jidoka/Andon = correção; Kaizen = aperfeiçoamento).

### Temas transversais (na **base conceitual**, compartilhada por todas as lentes)

> Conteúdo neutro à metáfora — mora em [`../base-conceitual/`](../base-conceitual/README.md) e é reusado
> por todas as lentes. Aqui funciona como os anexos técnicos desta fábrica.

| Tema | O que aprofunda | Enquadramento nesta lente |
|---|---|---|
| [Operar a Metodologia](../base-conceitual/operar-a-metodologia.md) | Como o humano **dirige**: a fórmula do bom pedido, quando não implementar, checklist de aceite | **A ordem de serviço bem especificada** — quem opera a fábrica não aperta botões: emite uma OS clara (objetivo, escopo, restrições, evidência, aceite) |
| [A Suíte como Prova](../base-conceitual/suite-como-prova.md) | A validação como **evidência auditável** (artefatos, `run_id`, foco × fechamento) | **O laudo de ensaio + certificado de rastreabilidade do lote** — QA emite prova que fica, não um "parece conforme" verbal |
| [Os Dois Loops](../base-conceitual/os-dois-loops.md) | Auto-correção (Jidoka/Andon) × auto-aperfeiçoamento (Kaizen); inclui **quando registrar** | **Registro de não-conformidade (RNC)** — cada defeito vira ajuste de procedimento para o próximo turno |
| [As Estratégias no SDLC](../base-conceitual/ciclo-de-vida-software.md) | Cobertura fase-a-fase do ciclo de vida | — |
| [Travas para Projetos Grandes](../base-conceitual/travas-projetos-grandes.md) | As 6 travas que permitem produzir num código enorme sem estourar custo | — |
| [⭐ Sem Digitar Código](../base-conceitual/sem-digitar-codigo.md) | Por que a inspeção final (code review) já está embutida na linha | — |
| [Diagramas](../base-conceitual/diagramas.md) · [FAQ](../base-conceitual/faq-objecoes.md) · [Cartão de Referência](../base-conceitual/cartao-de-referencia.md) · [Indicadores](../base-conceitual/indicadores.md) | Artefatos de apoio neutros | — |

---

## 6. Glossário rápido da fábrica (para a plateia não-técnica)

- **Estação (agente):** um posto de trabalho especialista, com cérebro e contexto próprios, que recebe
  uma peça, trabalha isolado e entrega só o resultado.
- **Ordem de produção (skill / `/comando`):** o documento/botão que aciona uma estação específica.
- **Poka-yoke:** dispositivo à prova de erro — torna fisicamente impossível montar errado. Aqui são os
  **guards** (hooks que bloqueiam).
- **Andon / Jidoka:** o cordão que **para a linha** quando um defeito aparece, para consertar a causa na
  origem antes de produzir mais refugo.
- **Sensor/alarme (nudge):** avisa que algo saiu do padrão, **mas não para a linha** — o operador decide.
- **Rastreabilidade de lote (`correlation_id`):** o número de série que segue a peça por toda a linha,
  permitindo reconstruir o que aconteceu (o "raio-X").
- **Kaizen:** melhoria contínua — cada defeito vira um ajuste de procedimento para o próximo turno.
- **Falso-verde:** quando parece pronto (ensaio passou, doc atualizada) mas o produto real ainda não usa
  a peça nova. O refugo nº 1 da metodologia.

---

> **Próximo passo sugerido:** comece pela [Norma da Fábrica](capitulo-01-norma-da-fabrica.md) — sem
> entender o padrão mestre, as estações parecem arbitrárias; com ele, cada posto faz sentido.
