# A Tática por Trás dos Arquivos — Metodologia de Engenharia de Agentes do Prometeu

> ⚽ **Lente Futebol** · [↩ Índice da metodologia](../README.md) ·
> [🏭 Lente Fábrica de Software](../lente-engenharia-software/README.md) ·
> [📚 Base conceitual](../base-conceitual/README.md)

> **O que é este material.** Esta é a documentação-mãe de uma aula sobre a *engenharia* da nossa
> configuração do Claude Code. Não é um "o que é um agente" — isso está no site da Anthropic.
> Aqui o foco é o **conteúdo** dos nossos arquivos de instrução: por que cada bloco existe, que
> problema real de engenharia de software ele evita, que valor ele força, e como tudo se encaixa
> no processo de desenvolvimento. A metáfora condutora é o **futebol**: cada arquivo é um jogador
> com posição, papel e função; juntos eles formam um time com tática.

---

## 0. Como ler este material (duas camadas)

Este material foi escrito para a empresa inteira, então ele tem **duas camadas** e você escolhe a profundidade:

- 🧑‍💼 **Camada executiva (liderança, produto, comercial):** os blocos marcados com `RESUMO EXECUTIVO`.
  Falam de risco, custo, qualidade, previsibilidade e velocidade — sem jargão.
- 🛠️ **Camada técnica (engenharia):** o corpo de cada capítulo. Fala de SOLID, clean code,
  observabilidade, contratos, anti-duplicação, falso-verde.

> 🧑‍💼 **RESUMO EXECUTIVO (a tese central, em uma frase).**
> Nós não "usamos uma IA para programar". Nós **montamos um time de engenharia inteiro dentro de
> arquivos de texto** — com filosofia, posições, jogadas ensaiadas e um árbitro automático — de
> forma que a IA seja obrigada a trabalhar como um engenheiro sênior disciplinado, e não como um
> estagiário apressado. O valor não está na ferramenta; está na **engenharia da configuração**.

---

## 1. A virada de chave: instrução não é documentação, é tática

A maioria das empresas usa IA de programação como um chute a gol isolado: "escreve esse código pra mim".
Funciona às vezes, falha imprevisivelmente, e ninguém sabe por quê.

A nossa abordagem é diferente. Tratamos a configuração do Claude como **um time de futebol profissional**:

| No futebol | Na nossa engenharia |
|---|---|
| Filosofia do clube (o DNA, o jeito de jogar) | `CLAUDE.md` — lido em toda sessão, governa todo mundo |
| Jogadas ensaiadas / manuais de posição | `.claude/rules/` — contratos profundos consultados sob demanda |
| Jogadores em campo, cada um numa posição | `.claude/agents/` — 22 especialistas (investigar, implementar, validar…) |
| A convocação / o apito que chama o especialista | `.claude/skills/` — gatilhos `/comando` que colocam o jogador certo em campo |
| O árbitro, as linhas do campo e o VAR | `.claude/hooks/` — automações que bloqueiam falta e levantam a bandeira |
| A súmula oficial | `.claude/settings.json` — registra a escalação e amarra tudo |

A diferença entre "usar IA" e "fazer engenharia com IA" é exatamente esta: **um time com tática não
depende de um lance genial; depende de cada jogador cumprir sua função.** É isso que transforma
resultado imprevisível em resultado confiável.

---

## 2. A escalação — formação 4-3-3 da nossa engenharia

Pense numa formação clássica. Cada grupo de arquivos ocupa um setor do campo:

```
                          ┌─────────────────────────────┐
                          │   FILOSOFIA DO CLUBE (DNA)   │
                          │         CLAUDE.md            │
                          │  lido em 100% das sessões    │
                          └─────────────────────────────┘
                                       │ rege
                                       ▼
  ÁRBITRO + VAR (hooks)  ░░░░░░░░ as linhas do campo, sempre ligadas ░░░░░░░░
  bash-guard · write-guard · py-lint · logging-nudge · py-discipline · stop-loop

      GOLEIRO              ZAGA              MEIO             ATAQUE
   validar-entrega     investigar        planejar        implementar
   (defende a          (lê o jogo)      (arma a jogada)  (faz o gol =
    qualidade)                                            código em runtime)

   ── defesa especializada (observabilidade) ──
   analisar-log · validar-log · validar-observabilidade · corrigir-erros-com-log · testar-cli-log-analyzer

   ── departamento de futebol (operação real) ──
   gerenciar-postgresql · gerenciar-redis · gerenciar-job-core-ingestao · gerenciar-scheduler

   ── treinos e amistosos (testes de fluxo real) ──
   executar-testes · criar-testes · testar-ingestao-dnit · testar-cancelamento · testar-webchat · testar-nl2sql · testar-nl2yaml · testar-paginas-web

   ── comissão técnica e bastidores (governança/conhecimento) ──
   documentar · sincronizar-documentacao · inventario-componentes · inventario-yaml · validar-instructions · analisar-produto

   ── jogadas ensaiadas (rules consultadas sob demanda) ──
   log-instructions · reuso-instructions · suite-testes · definicao-de-pronto · estrategia-recomendacoes · python · large-repo · fidelidade-pedido
   + caderno de aprendizado: lessons · error-backlog · regression-logs · bad-instructions
```

---

## 3. O eixo central do time — a jogada que sempre se repete

Se você guardar **uma única coisa** desta aula, guarde este fluxo. É a "jogada ensaiada" que
estrutura quase todo trabalho não-trivial:

```
  /investigar  ──►  /planejar  ──►  /implementar  ──►  /validar-entrega
   (zaga)           (meia)           (ataque)            (goleiro)

   levanta a        transforma       executa no          confere se o gol
   evidência do     evidência em     código real,        valeu mesmo, ou
   estado real      plano seguro     com testes e        se foi impedimento
   (sem chutar)     e incremental    logs                (falso-verde)
```

Cada um desses quatro é um **agente** (um jogador), acionado por uma **skill** (a convocação `/`).
O que faz isso ser engenharia, e não enfeite:

1. **Separação de responsabilidades levada ao extremo.** Quem investiga **não** planeja. Quem planeja
   **não** executa. Quem executa **não** se auto-aprova. Cada troca de pé é uma barreira contra o
   erro mais comum da IA: pular direto da ideia para o código, sem evidência e sem checagem.
2. **Cada jogador entrega a bola para o próximo com contrato.** A investigação vira a entrada
   obrigatória do plano; o plano vira a entrada obrigatória da implementação; a implementação vira a
   entrada obrigatória da validação. Nada "desaparece no meio do caminho".
3. **O goleiro pode estufar a rede do próprio time.** `validar-entrega` tem autoridade para
   **reprovar** uma entrega e mandar de volta para o `planejar`. É um adversário interno da pressa.

> 🧑‍💼 **RESUMO EXECUTIVO.** Esse eixo é o nosso controle de qualidade industrial. Ele troca
> "a IA fez, deve estar certo" por "quatro especialistas independentes assinaram embaixo, cada um
> com evidência". O custo é mais passos; o ganho é **previsibilidade** — menos retrabalho, menos
> bug em produção, menos surpresa.

---

## 4. Os cinco valores de engenharia que a tática inteira protege

Todos os arquivos, em todas as posições, existem para forçar cinco valores. Esta é a "filosofia de
jogo" do clube. Cada capítulo vai mostrar *como* cada arquivo materializa estes valores:

1. **Evidência acima de suposição.** Proibido inventar API, endpoint, schema, comportamento. Só vale
   o que foi lido no código. *(Mata a alucinação — o erro nº 1 de IA em código.)*
2. **Rastreabilidade ponta a ponta.** Todo processo real tem um `correlation_id` que nasce uma vez e
   atravessa tudo; todo log é "raio-X" reconstruível. *(Mata o bug invisível em produção.)*
3. **Anti-falso-verde.** Teste passar não prova nada se o caminho oficial em runtime não usa o código
   novo. A entrega só vale ligada no boundary real. *(Mata a falsa sensação de "pronto".)*
4. **Reuso antes de criar.** Em repositório grande, solução nova é rara — quase sempre já existe.
   *(Mata a duplicação, a dívida técnica e a divergência de comportamento.)*
5. **Escopo mínimo, qualidade máxima.** "100%" qualifica a *execução* do recorte pedido, não autoriza
   inflar o escopo. *(Mata o over-engineering e a refatoração cosmética.)*

---

## 5. Índice dos capítulos (cada um aprofunda uma posição)

| Capítulo | Setor no campo | O que você vai entender |
|---|---|---|
| [Cap. 1 — A Filosofia do Clube](capitulo-01-filosofia-claude-md.md) | O DNA (`CLAUDE.md`) | Por que cada uma das 17 seções existe e que valor força |
| [Cap. 2 — As Jogadas Ensaiadas](capitulo-02-rules-playbook.md) | Manuais de posição (`rules/`) | Os contratos profundos e o caderno de aprendizado |
| [Cap. 3 — Os Jogadores em Campo](capitulo-03-agents-jogadores.md) | Os 22 agentes | Posição, papel e o problema que cada um evita |
| [Cap. 4 — A Convocação](capitulo-04-skills-convocacao.md) | As skills (`/comando`) | Por que separar o gatilho do cérebro |
| [Cap. 5 — A Arbitragem](capitulo-05-hooks-arbitragem.md) | Hooks e nudges | Automação determinística: bloquear vs. cutucar |
| [Cap. 6 — A Partida](capitulo-06-a-partida.md) | O jogo completo | Um exemplo real atravessando o time inteiro |
| [🎤 Roteiro de Aula](roteiro-aula-slides.md) | Material de apresentação | 35 slides (6 atos) com tela + fala + interação, prontos para ministrar |
| [📽️ Apresentação Marp](apresentacao-marp.md) | Deck projetável | Versão Marp do roteiro: exporta para HTML/PDF/PPTX; falas viram notas do apresentador |

### Temas transversais (na **base conceitual**, compartilhada por todas as lentes)

> Estes conteúdos são neutros à metáfora — moram em [`../base-conceitual/`](../base-conceitual/README.md)
> e são reusados por todas as lentes. Aqui eles funcionam como os "apêndices" desta lente.

| Tema | O que aprofunda | Enquadramento nesta lente |
|---|---|---|
| [Operar a Metodologia](../base-conceitual/operar-a-metodologia.md) | Como o humano **dirige**: a fórmula do bom pedido, quando não implementar, checklist de aceite | **A preleção do treinador** — você não entra em campo; arma a jogada da beira (objetivo, escopo, restrições, evidência, aceite) e exige o resultado |
| [A Suíte como Prova](../base-conceitual/suite-como-prova.md) | A validação como **evidência auditável** (artefatos, `run_id`, foco × fechamento) | **A análise de desempenho com dados** — não é "achei que jogamos bem"; é o relatório que prova o que aconteceu em campo |
| [Os Dois Loops](../base-conceitual/os-dois-loops.md) | Auto-correção × auto-aperfeiçoamento: por que são opostos, onde vivem e onde se encontram; inclui **quando registrar** | **Ajustar no intervalo × mudar o treino da semana** |
| [Travas para Projetos Grandes](../base-conceitual/travas-projetos-grandes.md) | As 6 travas que permitem usar IA num código enorme sem estourar custo | — |
| [As Estratégias no SDLC](../base-conceitual/ciclo-de-vida-software.md) | Ciclo de vida do software: mapa fase-a-fase com dono, contrato e prova | — |
| [⭐ Sem Digitar Código](../base-conceitual/sem-digitar-codigo.md) | **O tema estratégico**: desenvolver/corrigir sem digitar uma linha; por que o code review já está embutido | — |

### Artefatos de apoio (na **base conceitual** — uso prático, não só aula)

| Artefato | Para quê | Quando usar |
|---|---|---|
| [Diagramas](../base-conceitual/diagramas.md) | 4 diagramas Mermaid (loops, decisão de skill, camadas) | Projetar na aula; consultar o fluxo |
| [FAQ de Objeções](../base-conceitual/faq-objecoes.md) | Respostas honestas às perguntas difíceis | Seção de perguntas; defender a abordagem |
| [Cartão de Referência](../base-conceitual/cartao-de-referencia.md) | 1 página: comando → garante → pré-requisito + checklist de extensão | Uso diário; onboarding |
| [Indicadores](../base-conceitual/indicadores.md) | O que medir para provar (e auditar) a metodologia | Reuniões de liderança; melhoria contínua |

---

## 6. Glossário rápido (para a plateia não-técnica)

- **Agente (subagente):** um "jogador" especialista, com cérebro e contexto próprios, que recebe uma
  tarefa, trabalha isolado e devolve só o resultado.
- **Skill / comando `/`:** o "apito" que coloca um jogador específico em campo. Geralmente é um
  atalho fino que delega ao agente.
- **Hook:** uma automação do sistema que dispara sozinha em certos momentos (antes de um comando,
  depois de editar um arquivo, no fim da rodada). Não é a IA decidindo — é regra automática.
- **Nudge ("cutucada"):** um hook que **avisa, mas não bloqueia** — levanta a bandeira como um
  bandeirinha, e a IA decide se acata. (O termo certo é *nudge*, "empurrãozinho", não "nodle".)
- **Guard ("trava"):** um hook que **bloqueia de fato** — apita a falta e a jogada não acontece.
- **`correlation_id`:** o número de rastreio de uma execução. Nasce uma vez e segue o processo
  inteiro, permitindo reconstruir o que aconteceu depois (o "raio-X").
- **Falso-verde:** quando parece pronto (teste passou, doc atualizada) mas o sistema real ainda não
  usa o código novo. O inimigo público nº 1 da nossa metodologia.

---

> **Próximo passo sugerido para a aula:** comece pela [Filosofia do Clube](capitulo-01-filosofia-claude-md.md)
> — sem entender o DNA, as posições parecem arbitrárias; com ele, cada jogador faz sentido.
</content>
</invoke>
