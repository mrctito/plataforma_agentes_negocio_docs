# Capítulo 6 — A Partida (o time inteiro em ação)

> **O objetivo deste capítulo:** até aqui descrevemos cada posição isolada. Agora vamos ver o **time
> jogando junto** — uma mudança real atravessando o campo inteiro, do apito inicial ao gol validado.
> É aqui que a tática deixa de ser teoria e vira resultado. Use este capítulo como o **clímax da
> aula**: a narrativa que amarra tudo.

---

## 6.1 O cenário: um bug real chega ao time

Imagine um pedido concreto, do tipo que chega todo dia:

> *"A ingestão de PDFs da DNIT está terminando com erro intermitente, e ninguém sabe por quê. Conserta."*

Vamos seguir a bola, passe a passe, e marcar **qual jogador toca, qual jogada ensaiada ele consulta e
qual árbitro está de olho**.

---

## 6.2 O jogo, lance a lance

### Apito inicial — `session-start` faz a preleção
Antes de qualquer coisa, o hook `session-start` injeta as **lições passadas** (`lessons.md`) e avisa
quantos erros há no backlog. O time entra em campo já sabendo o que deu errado antes. *(Cap. 5)*

### 1º toque — `analisar-log` (o olheiro) lê o jogo
Como há um erro real com rastro, o primeiro a tocar é o **olheiro**. Ele pega o `correlation_id` da
execução que falhou e roda a CLI oficial de análise — **sem corrigir nada ainda**. Consolida a linha do
tempo, separa o que o log **prova** do que não prova. Se alguém tentasse uma listagem cega de `/logs`,
o **árbitro `bash-guard` apitaria a falta** na hora. *(Cap. 3.2 + Cap. 5.3)*

Resultado: "o erro ocorre na etapa X, em N dos M documentos, sempre quando a condição Y".

### 2º toque — `investigar` (o zagueiro) levanta a verdade estrutural
Com a pista do olheiro, o **zagueiro** faz a análise forense do *código real* daquele trecho. Consulta
as jogadas ensaiadas obrigatórias — `large-repo-navigation` (para não se perder), `reuso-instructions`
(o conserto já existe?), `log-instructions` (a observabilidade está coberta?). Entrega um **relatório
com evidência (path + linha)** e nível de confiança. **Não chuta**: o que não foi lido, é declarado
como desconhecido. *(Cap. 3.1 + Cap. 2)*

### 3º toque — `planejar` (o meia) arma a jogada
O **meia** recebe o relatório e, antes de tudo, **discorda ou concorda com franqueza** (a correção
proposta é a melhor? há alternativa mais barata?). Então monta o plano: a tarefa **T1** cria a seção
`# Estratégia/Recomendacoes` (a bússola), a **T2** mapeia os testes que protegem o trecho, e as tarefas
seguintes detalham o conserto — cada uma com critério de aceite e validação. **Nenhum achado do
zagueiro fica sem destino.** *(Cap. 3.1 + Cap. 2.1)*

### 4º toque — `implementar` (o atacante) finaliza
O **atacante** lê a bússola **antes de tocar qualquer arquivo**, cria o teste de proteção, e faz o
conserto incremental. A cada arquivo `.py` salvo, os **bandeirinhas (`logging-nudge`, `py-lint`,
`py-discipline`)** levantam a bandeira se ele esquecer um `logger.exception` no `except`, usar uma
f-string no log ou deixar um teste sem marker. Antes de cantar "CONCLUÍDA", ele cumpre a jogada ensaiada
`definicao-de-pronto`: **o conserto está ativo no caminho oficial em runtime?** Se não, vira BLOQUEADA,
não gol. *(Cap. 3.1 + Cap. 5.4 + Cap. 2.1)*

### 5º toque — os treinos provam o conserto na vida real
O `testar-ingestao-dnit` (atacante de jogo oficial) roda a ingestão **de verdade pela UI** e exige as
**três provas**: UI (correlation_id no clique real), log (raio-X zero erro) e **paralelismo real** por
sobreposição temporal. Se algo falha, o loop corretivo reinicia. Pelos bastidores, o
`gerenciar-job-core-ingestao` garante que cada tentativa começa num **ambiente limpo** (dry-run antes de
qualquer mutação). *(Cap. 3.3 e 3.4)*

### Gol? Só se o goleiro validar — `validar-entrega`
O **goleiro** confronta tudo: o implementado bate com o plano? Cada tarefa cumpriu seu critério? Aciona
os especialistas `validar-log` e `validar-observabilidade` para conferir o raio-X. A regra é dura:
**teste verde não prova fluxo real**. Veredito explícito: APROVADO, COM RESSALVAS, REPROVADO ou
BLOQUEADO. Se reprovado, ele **devolve a bola para o meia** com um inventário do que faltou. *(Cap. 3.1)*

### Apito final — `stop-loop` cobra o aprendizado
No fim, o hook `stop-loop` cobra: esse bug virou uma **lição transversal** em `lessons.md`? Foi para o
`error-backlog`? Se mexeu em `src/` sem tocar em `tests/`, levanta a bandeira. O time sai de campo **mais
inteligente do que entrou**. *(Cap. 5.5)*

---

## 6.3 O mapa da jogada (visão de cima)

```
 session-start ─ preleção (lições passadas)
        │
        ▼
 analisar-log ──► investigar ──► planejar ──► implementar ──► testar-ingestao ──► validar-entrega
   (olheiro)      (zagueiro)     (meia)       (atacante)       (jogo real)         (goleiro)
        │              │            │             │                 │                  │
        └─ bash-guard ─┴── rules ───┴─ estratégia ┴─ nudges/pronto ─┴─ job-core/infra ─┘
                                                                                        │
                                                          ┌─────────────────────────────┘
                                                          ▼
                                            REPROVADO? ──► volta para planejar
                                            APROVADO?  ──► stop-loop cobra aprendizado
```

Repare: **a bola pode voltar.** O sistema não é uma esteira de mão única — é um time que recupera a
posse quando o lance não sai limpo. Essa capacidade de **devolver para trás com inventário** é o que
impede o falso-verde de chegar à produção.

---

## 6.4 Os problemas de engenharia que a partida inteira evita

Vale fechar com o placar — o que esta tática, no conjunto, impede que aconteça:

| Problema clássico de engenharia | Quem, no time, o neutraliza |
|---|---|
| **Alucinação** (API/schema inventados) | `CLAUDE.md §1` + `investigar` (evidência com path/linha) |
| **Falso-verde** (parece pronto, não está) | `definicao-de-pronto` + `validar-entrega` + teste de regressão estrutural |
| **Bug invisível em produção** | `log-instructions` + setor de observabilidade + `logging-nudge` |
| **Duplicação / dívida técnica** | `reuso-instructions` + `inventario-componentes` |
| **Over-engineering / escopo inflado** | `CLAUDE.md` ("100%" qualifica execução) + `planejar` |
| **Regressão silenciosa** | `criar-testes` + `executar-testes` + `stop-loop` |
| **Acidente operacional** (rm -rf, drop) | `bash-guard` (guard) + agentes de infra (dry-run) |
| **Travar a sessão** com varredura de log | `bash-guard` + regra anti-varredura |
| **Conhecimento que apodrece** | `documentar` + `sincronizar-documentacao` + caderno de aprendizado |
| **Repetir o mesmo erro** | `session-start` + `stop-loop` + `lessons.md` |

---

## 6.5 Os valores de engenharia que a tática incentiva (o DNA em ação)

E o lado positivo do placar — o que ela **cultiva**:

- **Evidência empírica** acima de opinião e memória.
- **Rastreabilidade ponta a ponta** (correlation_id, logs como raio-X).
- **Separação de responsabilidades / segregação de funções** (quem faz ≠ quem aprova).
- **Incrementalidade validável** (tarefas pequenas, testáveis, com rollback).
- **Honestidade de entrega** (pronto = ativo no runtime, não "parece pronto").
- **Reuso e DRY** (inclusive na própria configuração da IA).
- **Melhoria contínua institucional** (o time aprende, não a pessoa).

---

## 6.6 O fechamento da aula — a tese, agora completa

> 🧑‍💼 **A mensagem final para a empresa.**
>
> Qualquer um pode "pedir código para uma IA". O que construímos é diferente em natureza: **um sistema
> de engenharia onde a IA é obrigada a trabalhar como um time sênior disciplinado** — com filosofia,
> especialistas de função única, jogadas ensaiadas, contratos de passe e um árbitro automático que não
> dorme.
>
> O valor não está em nenhum arquivo isolado. Está na **tática** — em como eles jogam juntos. É por
> isso que esta aula não foi sobre "o que é um agente": foi sobre **a engenharia por trás do conteúdo
> de cada arquivo**, porque é exatamente aí que mora a vantagem competitiva.
>
> Um jogador genial ganha um lance. **Um time com tática ganha o campeonato** — e ganha de forma
> previsível, repetível e auditável. É isso que transformamos em texto.

---

### Para quem vai ministrar — ganchos de discussão com a plateia

- *"Quem aqui já viu um teste passar e o bug continuar em produção?"* → entra o conceito de **falso-verde**.
- *"Quem já perdeu uma tarde porque o log não dizia nada?"* → entra o **raio-X / observabilidade**.
- *"Quem já reescreveu um helper que já existia?"* → entra **reuso > criar**.
- *"Qual a diferença entre uma regra que a gente 'pede' e uma que o sistema 'garante'?"* → entra
  **nudge vs. guard**.
- *"Por que separar quem investiga de quem implementa?"* → entra **segregação de funções**.

---

**Voltar para o início:** [README — A Tática por Trás dos Arquivos](README.md).
</content>
