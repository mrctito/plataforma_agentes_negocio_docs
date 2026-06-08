# Capítulo 6 — Um Lote na Linha (a fábrica inteira em ação) ⭐

> 🏭 [↩ Doc-mãe da lente](README.md) · [↩ Índice da metodologia](../README.md)

> **O objetivo deste capítulo:** até aqui descrevemos cada parte isolada. Agora vamos ver a **fábrica
> produzindo** — um pedido real percorrendo a linha do **recebimento à expedição** — e, sobretudo, os
> **dois ciclos** atuando: **Jidoka/Andon** (auto-correção, que para a linha e conserta a causa) e
> **Kaizen** (auto-aperfeiçoamento, que melhora o processo para o próximo lote). Use este capítulo como o
> **clímax da aula**.

---

## 6.1 O cenário: um pedido com defeito intermitente

Imagine um pedido concreto, do tipo que chega todo dia:

> *"A ingestão de PDFs da DNIT está terminando com erro intermitente, e ninguém sabe por quê. Conserta."*

Vamos seguir a peça, estação por estação, marcando **qual posto opera, qual especificação consulta e qual
dispositivo está de olho**.

---

## 6.2 A produção, estação por estação

### Início do turno — `session-start` faz o briefing
Antes de qualquer coisa, o dispositivo `session-start` injeta as **lições passadas** (`lessons.md`) e
avisa quantos defeitos há no backlog. A linha começa o turno já sabendo o que deu errado antes. *(Cap. 5 —
lado "ler" do Kaizen)*

### 1ª estação — `analisar-log` (metrologia) lê os instrumentos
Como há um defeito real com rastro, o primeiro a operar é a **metrologia**. Pega o `correlation_id` (número
de série) da execução que falhou e roda a CLI oficial de análise — **sem corrigir nada ainda**. Consolida
a linha do tempo, separa o que o registro **prova** do que não prova. Se alguém tentasse uma listagem
cega de `/logs`, o **poka-yoke `bash-guard` travaria** na hora. *(Cap. 3.2 + Cap. 5.3)*

Resultado: "o defeito ocorre na etapa X, em N dos M documentos, sempre quando a condição Y".

### 2ª estação — `investigar` (inspeção de entrada) mede a verdade estrutural
Com a pista da metrologia, a **inspeção de entrada** faz a medição forense do *código real* daquele
trecho. Consulta as especificações obrigatórias — `large-repo-navigation` (para não se perder),
`reuso-instructions` (o conserto já existe no almoxarifado?), `log-instructions` (a instrumentação está
coberta?). Entrega um **laudo com evidência (path + linha)** e nível de confiança. **Não chuta**: o que
não foi lido é declarado desconhecido. *(Cap. 3.1 + Cap. 2)*

### 3ª estação — `planejar` (PCP) emite o roteiro de fabricação
O **PCP** recebe o laudo e, antes de tudo, **discorda ou concorda com franqueza** (a correção proposta é a
melhor? há alternativa mais barata?). Então monta o roteiro: a tarefa **T1** cria a seção
`# Estratégia/Recomendacoes` (a ordem de serviço), a **T2** mapeia os ensaios que protegem o trecho, e as
tarefas seguintes detalham o conserto — cada uma com critério de aceite e validação. **Nenhum achado da
inspeção fica sem destino.** *(Cap. 3.1 + Cap. 2.1)*

### 4ª estação — `implementar` (fabricação) produz a peça boa
A **fabricação** lê a ordem de serviço **antes de tocar qualquer arquivo**, cria o ensaio de proteção e
faz o conserto incremental. A cada arquivo `.py` salvo, os **sensores (`logging-nudge`, `py-lint`,
`py-discipline`)** acendem o alarme se ela esquecer um `logger.exception` no `except`, usar uma f-string no
log ou deixar um ensaio sem marker. Antes de declarar "CONCLUÍDA", cumpre a especificação
`definicao-de-pronto`: **o conserto está ativo na linha oficial em runtime?** Se não, vira BLOQUEADA, não
peça pronta. *(Cap. 3.1 + Cap. 5.4 + Cap. 2.1)*

### 5ª estação — o lote piloto prova o conserto na vida real
O `testar-ingestao-dnit` (lote piloto oficial) roda a ingestão **de verdade pela UI** e exige as **três
provas**: UI (correlation_id no clique real), log (raio-X zero erro) e **paralelismo real** por
sobreposição temporal. Se algo falha, o ciclo de correção reinicia. Pelos bastidores, o
`gerenciar-job-core-ingestao` garante que cada tentativa começa num **ambiente limpo** (dry-run antes de
qualquer mutação). *(Cap. 3.3 e 3.4)*

### Expedição? Só se o controle de qualidade liberar — `validar-entrega`
O **controle de qualidade** confronta tudo: o fabricado bate com o roteiro? Cada tarefa cumpriu seu
critério? Aciona o laboratório (`validar-log`, `validar-observabilidade`) para conferir o raio-X. A regra
é dura: **ensaio verde não prova linha real**. Veredito explícito: APROVADO, COM RESSALVAS, REPROVADO ou
BLOQUEADO. Se reprovado, **devolve o lote ao PCP** com um inventário do que faltou. *(Cap. 3.1)*

### Fim do turno — `stop-loop` cobra o Kaizen
No fim, o dispositivo `stop-loop` cobra: esse defeito virou uma **lição transversal** em `lessons.md`? Foi
para o `error-backlog`? Se mexeu em `src/` sem tocar em `tests/`, acende o alarme. A linha termina o turno
**mais inteligente do que começou**. *(Cap. 5.5 — lado "escrever" do Kaizen)*

---

## 6.3 O mapa da produção (visão de cima)

```
 session-start ─ briefing (lições passadas)
        │
        ▼
 analisar-log ──► investigar ──► planejar ──► implementar ──► testar-ingestao ──► validar-entrega
 (metrologia)   (inspeção)     (PCP)        (fabricação)     (lote piloto)       (controle qualid.)
        │              │            │             │                 │                  │
        └─ bash-guard ─┴── SOPs ────┴─ ordem de ──┴─ sensores/ ─────┴─ job-core/infra ─┘
                                       serviço       pronto                              │
                                                          ┌─────────────────────────────┘
                                                          ▼
                                            REPROVADO? ──► volta ao PCP (retrabalho)
                                            APROVADO?  ──► stop-loop cobra o Kaizen
```

Repare: **o lote pode voltar.** A linha não é de mão única — é uma célula que **recupera o trabalho**
quando a peça não sai conforme. Essa capacidade de **devolver com inventário** é o que impede o
falso-verde de chegar ao cliente.

---

## 6.4 Os dois ciclos em foco (o coração desta lente)

A produção acima escondia **dois motores de melhoria** rodando o tempo todo. Eles são parecidos no nome e
**opostos na função** — e entender o contraste é o insight central. *(Fonte profunda e neutra:
[Os Dois Loops](../base-conceitual/os-dois-loops.md).)*

### 🛑 Auto-correção = Jidoka + Andon (conserta ESTE lote)

**Jidoka** é o princípio lean de "automação com toque humano": a máquina **para sozinha ao detectar
anomalia** em vez de produzir refugo. O **Andon** é o cordão que para a linha. Na nossa fábrica, isso é o
**loop de auto-correção**: executar → detectar defeito → **parar e consertar a causa raiz** → revalidar —
com **teto** para não rodar para sempre.

Onde ele vive, em evidência real:

| Onde | Como o ciclo funciona | Trava contra loop infinito |
|---|---|---|
| `validar-entrega` | REPROVADO → inventário de não-conformidades → **devolve o lote ao PCP** | Só volta com lista objetiva do que faltou |
| `executar-testes` | executar → falhar → corrigir → validar local → re-executar | **Máximo ~20 ciclos** ou impasse comprovado |
| `corrigir-erros-com-log` | erro dominante → hipótese → **checagem discriminante** → ação → resultado | Não repete hipótese **já falsificada** |
| `testar-ingestao-dnit` | executar → capturar correlation_id → ler registro → 3 provas → corrigir → reiniciar | **Máximo ~50 tentativas**; além disso só com bloqueio comprovado |
| `testar-cli-log-analyzer` | comparar instrumento × verdade manual → corrigir → revalidar | Fecha só quando a **matriz inteira** bate |

Os três pilares que tornam o Andon seguro (e não um botão de pânico burro): **checagem discriminante** (o
check mais barato capaz de *desconfirmar* a hipótese atual), **memória de rodada** (nunca repetir
tentativa já descartada) e **teto + status binário** (converge ou declara bloqueio com evidência — nunca
"quase pronto" eterno).

### 🔁 Auto-aperfeiçoamento = Kaizen (melhora a fábrica para os PRÓXIMOS lotes)

**Kaizen** é a melhoria contínua **entre turnos**. Na nossa fábrica, é o **loop de auto-aperfeiçoamento**:
os dispositivos de início/fim de turno (a **troca de turno**) leem as lições na entrada e cobram o
registro na saída, fazendo o know-how se acumular na empresa.

```
   INÍCIO DO TURNO                                 FIM DO TURNO
   session-start.sh                                stop-loop-reminder.sh
   ── lê e injeta lessons.md                       ── cobra registrar lição/defeito/reincidência
      (o turno começa relembrado)                     (o turno termina tendo registrado)
            │                                                    │
            └──────────── lado "LER" ── + ── lado "ESCREVER" ────┘
                                       │
                          DIÁRIO DE BORDO (.claude/rules/)
                  lessons.md · error-backlog.md · regression-logs.md · bad-instructions.md
```

### A ponte entre os dois (onde se encontram)

> Quando o **Andon** de um lote descobre uma regra preventiva reaproveitável, essa lição é **promovida**
> para o **Kaizen** (vai para o `lessons.md`).

O conserto de **um** defeito de hoje vira a **prevenção** de uma família de defeitos amanhã. O esforço de
auto-correção não se perde quando o lote fecha — parte vira patrimônio da fábrica.

| | Auto-correção (Jidoka/Andon) | Auto-aperfeiçoamento (Kaizen) |
|---|---|---|
| **Pergunta** | "Já está conforme?" | "O que aprendemos para não repetir?" |
| **Escopo** | Um lote/rodada | Todos os lotes futuros |
| **Dispara por** | Defeito detectado no próprio lote | Troca de turno (hooks de início/fim) |
| **Termina** | Quando converge (com teto) | Nunca (é contínuo) |
| **Mata o problema de** | Peça "quase pronta", retrabalho infinito | Repetir defeito, conhecimento que apodrece |

---

## 6.5 Os defeitos de engenharia que a produção inteira evita

| Problema clássico de engenharia | Quem, na fábrica, o neutraliza |
|---|---|
| **Alucinação** (API/schema inventados) | `CLAUDE.md §1` + `investigar` (evidência com path/linha) |
| **Falso-verde** (parece pronto, não está) | `definicao-de-pronto` + `validar-entrega` + teste de regressão estrutural |
| **Defeito invisível no cliente** | `log-instructions` + laboratório de observabilidade + `logging-nudge` |
| **Duplicação / estoque de código** | `reuso-instructions` + `inventario-componentes` |
| **Over-engineering / escopo inflado** | `CLAUDE.md` ("100%" qualifica execução) + `planejar` |
| **Regressão silenciosa** | `criar-testes` + `executar-testes` + `stop-loop` |
| **Acidente operacional** (rm -rf, drop) | `bash-guard` (poka-yoke) + manutenção (dry-run) |
| **Travar a sessão** com varredura de log | `bash-guard` + regra anti-varredura |
| **Conhecimento que apodrece** | `documentar` + `sincronizar-documentacao` + diário de bordo |
| **Repetir o mesmo defeito** | `session-start` + `stop-loop` + `lessons.md` (Kaizen) |

---

## 6.6 Os valores de engenharia que a fábrica cultiva (a Norma em ação)

- **Evidência empírica** acima de opinião e memória.
- **Rastreabilidade de lote ponta a ponta** (correlation_id, logs como raio-X).
- **Segregação de funções** (quem fabrica ≠ quem aprova).
- **Incrementalidade validável** (tarefas pequenas, testáveis, com rollback).
- **Honestidade de entrega** (pronto = ativo no runtime, não "parece pronto").
- **Reuso e DRY** (inclusive na própria configuração da IA).
- **Kaizen institucional** (a fábrica aprende, não a pessoa).

---

## 6.7 O fechamento da aula — a tese, agora completa

> 🧑‍💼 **A mensagem final para a empresa.**
>
> Qualquer um pode "pedir código para uma IA". O que construímos é diferente em natureza: **uma linha de
> produção de software onde a IA é obrigada a operar como uma fábrica madura com controle de qualidade** —
> com norma, estações de função única, especificações de processo, dispositivos à prova de erro e dois
> motores de melhoria contínua que não dormem.
>
> O valor não está em nenhum arquivo isolado. Está no **processo** — em como eles operam juntos. É por
> isso que esta aula não foi sobre "o que é um agente": foi sobre **a engenharia por trás do conteúdo de
> cada arquivo**, porque é aí que mora a vantagem competitiva.
>
> Produção artesanal entrega uma peça boa de vez em quando. **Uma fábrica enxuta entrega qualidade
> previsível, repetível e auditável, lote após lote** — e melhora sozinha a cada turno. É isso que
> transformamos em texto.

---

### Para quem vai ministrar — ganchos de discussão com a plateia

- *"Quem aqui já viu um ensaio passar e o defeito continuar no cliente?"* → entra o **falso-verde**.
- *"Quem já perdeu uma tarde porque o registro de processo não dizia nada?"* → entra o **raio-X**.
- *"Quem já refez um componente que já existia no almoxarifado?"* → entra **reuso > fabricar**.
- *"Qual a diferença entre uma regra que a gente 'pede' e uma que o dispositivo 'garante'?"* → entra
  **poka-yoke vs. alarme**.
- *"Por que separar quem inspeciona de quem fabrica?"* → entra **segregação de funções**.

---

**Voltar para o início:** [A Fábrica de Software](README.md) ·
[📚 Base conceitual](../base-conceitual/README.md) · [↩ Índice da metodologia](../README.md).
