# As Estratégias no Ciclo de Vida do Software (SDLC)

> **Documento da base conceitual — neutro a qualquer lente.** Cada apresentação (futebol, fábrica de
> software, futuras) conta a mesma cobertura com sua linguagem; aqui está o **mapa neutro** fase-a-fase.

> **Por que este documento existe.** As lentes descrevem o sistema por **papel/posição**. Mas um gestor
> de engenharia pensa por **fase do ciclo de vida do software** (SDLC): levantamento, análise, design,
> implementação, testes, aceite, operação, manutenção, governança. Este documento faz a **tradução**
> entre os dois — mostra que a configuração não é um amontoado de arquivos, mas uma **cobertura completa
> e intencional de todas as fases do desenvolvimento**.

> 🧑‍💼 **RESUMO EXECUTIVO.** A pergunta que este documento responde é: *"isso cobre o processo inteiro
> ou só a parte de escrever código?"*. A resposta: **cobre o ciclo inteiro** — do entendimento do
> pedido até a operação em produção e a melhoria contínua. Cada fase tem dono, contrato e prova. É um
> processo de engenharia completo, não um assistente de digitação de código.

---

## C.1 O mapa: fase do SDLC → quem cobre → com qual contrato

| Fase do SDLC | Agentes responsáveis | Contratos/regras que regem | O que a fase garante |
|---|---|---|---|
| **0. Entendimento do pedido** | *(janela principal)* | `fidelidade-pedido-usuario.md` | O pedido é reapresentado **fiel**, com as "regras imperativas" do usuário destacadas |
| **1. Análise / Discovery** | `investigar`, `analisar-log`, `analisar-produto`, `inventario-componentes`, `inventario-yaml` | `large-repo-navigation.md`, `reuso-instructions.md`, `log-instructions.md` | A **verdade do estado atual** é levantada com evidência (path/linha), sem chute |
| **2. Design / Planejamento** | `planejar` | `estrategia-recomendacoes.md` | A evidência vira **plano seguro, incremental e validável**, com rollback |
| **3. Implementação** | `implementar` | `python.md`, `reuso-instructions.md`, `CLAUDE.md §6/§7/§8` | Código **limpo, tipado, sem duplicação**, no caminho oficial |
| **4. Testes / QA** | `criar-testes`, `executar-testes`, `testar-*` | `suite-testes-instructions.md`, `tests/CLAUDE.md` | Comportamento **provado em execução real**, suíte verde de verdade |
| **5. Verificação / Aceite** | `validar-entrega`, `validar-log`, `validar-observabilidade` | `definicao-de-pronto.md` | Entrega **ativa no runtime** (anti-falso-verde), com observabilidade |
| **6. Operação / Deploy** | `gerenciar-postgresql/redis/job-core/scheduler` | DSNs canônicas, dry-run + `--execute` | Operação real **auditável**, mínimo impacto, nada de improviso |
| **7. Manutenção / Evolução** | `corrigir-erros-com-log`, `implementar` | `definicao-de-pronto.md`, `error-backlog.md`, `regression-logs.md` | Bug corrigido **na causa raiz**, regressão registrada e protegida |
| **8. Documentação** | `documentar`, `sincronizar-documentacao` | `CLAUDE.md §17` | Doc **fiel ao código**, nunca antecipando o que o runtime não entrega |
| **9. Governança / Melhoria contínua** | `validar-instructions` + hooks | `lessons.md`, `bad-instructions.md`, `CLAUDE.md §15` | O **processo e o conhecimento** melhoram a cada rodada |

---

## C.2 As três camadas transversais (presentes em TODAS as fases)

Algumas estratégias não pertencem a uma fase — elas atravessam o ciclo inteiro, valendo do começo ao
fim do trabalho:

### 🧬 Camada 1 — O DNA (`CLAUDE.md`)
Está presente em 100% das sessões, logo em 100% das fases. Os cinco valores (evidência, rastreabilidade,
anti-falso-verde, reuso, escopo mínimo) regem desde o discovery até a operação.

### 🔍 Camada 2 — Observabilidade (`correlation_id` + logs canônicos)
Nasce na fase 3 (implementação instrumenta), é exigida na fase 5 (aceite audita), e é o que torna
possíveis as fases 7 (manutenção lê o log para corrigir) e 6 (operação diagnostica). O **raio-X**
conecta o código de hoje ao diagnóstico de amanhã.

### ⚖️ Camada 3 — A arbitragem automática (hooks)
Os guards e nudges disparam em qualquer fase em que haja edição ou comando — garantindo as regras
independentemente de onde o trabalho esteja no ciclo.

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  DNA (CLAUDE.md)  ── presente em todas as fases ──                     │
   ├──────────────────────────────────────────────────────────────────────┤
   │ 0►1►2►3►4►5►6►7►8►9   (as fases do SDLC, com seus agentes)             │
   ├──────────────────────────────────────────────────────────────────────┤
   │  Observabilidade (correlation_id + logs)  ── atravessa 3→5→6→7 ──      │
   │  Arbitragem (hooks)  ── dispara em qualquer fase com edição/comando ── │
   └──────────────────────────────────────────────────────────────────────┘
```

---

## C.3 O fluxo como o gestor enxerga (a esteira com retorno)

```
  PEDIDO
    │
    ▼
 [0] entender fiel ─► [1] analisar (verdade) ─► [2] planejar (rota) ─► [3] implementar
                                                                              │
                                          ┌───────────────────────────────────┘
                                          ▼
                         [4] testar (prova real) ─► [5] aceitar (anti-falso-verde)
                                          │                       │
                              REPROVADO ──┘   ◄──── devolve ──────┘
                                          │
                                  APROVADO ▼
                         [6] operar ─► [7] manter/evoluir ─► [8] documentar
                                          │
                                          ▼
                         [9] governança + aprendizado  ──► alimenta o próximo ciclo
```

**O ponto que diferencia da cascata clássica:** a fase 5 (aceite) pode **devolver para a 2
(planejamento)** — não é esteira de mão única. E a fase 9 (aprendizado) **alimenta o próximo ciclo
inteiro**, fechando o loop de auto-aperfeiçoamento (ver [Os Dois Loops](os-dois-loops.md)).

---

## C.4 Onde estão os dois loops dentro do ciclo

Conectando com [Os Dois Loops](os-dois-loops.md):

- **Loop de auto-correção** → vive **dentro e entre as fases 3–4–5** (implementar → testar → aceitar →
  devolve → replaneja), convergindo até a entrega ficar certa.
- **Loop de auto-aperfeiçoamento** → vive na **fase 9** e atravessa para o **próximo ciclo**,
  começando-o mais inteligente.

---

## C.5 A leitura honesta: o que esta metodologia cobre bem e o que depende de fora

Para a aula ser honesta (e o `CLAUDE.md` exige postura crítica), vale dizer onde estão os limites:

**Cobre com força:**
- Análise, design, implementação, testes, aceite, manutenção, documentação e governança técnica.
- A disciplina transversal (observabilidade, anti-falso-verde, reuso) em todas elas.

**Depende de processo humano/externo (a metodologia apoia, mas não substitui):**
- **Priorização de produto e roadmap** — o `analisar-produto` informa, mas a decisão é humana.
- **Deploy físico / CI-CD / infraestrutura de release** — a metodologia entrega código pronto e
  validado; a esteira de publicação em si é externa.
- **Requisitos de negócio iniciais** — a fase 0 garante *fidelidade* ao pedido, mas não inventa o
  pedido certo; isso vem do produto/cliente.

> 🧑‍💼 Dito de forma direta para a liderança: esta metodologia transforma "um pedido bem feito" em
> "software de qualidade, provado e observável". Ela **não substitui** decidir *o que* construir — ela
> garante que, decidido o que construir, a construção sai **certa, rastreável e sem falso-verde**.

---

## C.6 Resumo de bolso (para a aula)

- A configuração **não cobre só "escrever código"** — cobre o **ciclo de vida inteiro**, da
  compreensão do pedido à melhoria contínua.
- Cada fase tem **dono (agente), contrato (regra) e prova**.
- Três camadas atravessam tudo: **DNA, observabilidade e arbitragem**.
- O ciclo **tem retorno** (aceite devolve para planejamento) e **se realimenta** (aprendizado alimenta
  o próximo ciclo).
- Honestidade: ela garante a **qualidade da construção**, não a **decisão do que construir** nem a
  esteira física de deploy.

> **Frase para fechar o tema na aula:** "Não é um assistente que digita código mais rápido. É um
> **processo de engenharia completo** — com fase, dono, contrato e prova em cada etapa — que por acaso
> roda dentro de arquivos de texto."

---

**Relacionado (base conceitual):** [Os Dois Loops](os-dois-loops.md) ·
[Sem Digitar Código](sem-digitar-codigo.md) · [Diagramas](diagramas.md) ·
[↩ Voltar ao índice da metodologia](../README.md)
</content>
