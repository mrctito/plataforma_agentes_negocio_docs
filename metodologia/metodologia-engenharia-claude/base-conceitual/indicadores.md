# Indicadores — Como Provar que a Metodologia Funciona

> **Por que este artefato existe.** Uma metodologia que não se mede não se defende, não se melhora e não
> se justifica para a liderança. Este documento define **o que medir** para saber se o processo está
> realmente entregando qualidade — e, igualmente importante, para **detectar quando ele está sendo
> burlado** (falso-verde, atalho, erosão).
>
> ⚠️ **Aviso honesto de engenharia.** A maioria destes indicadores **ainda não é coletada
> automaticamente** hoje — este documento é a **proposta de medição**, não um relatório de números
> existentes. Vários sinais já existem na forma bruta (telemetria da suíte, `error-backlog.md`,
> `regression-logs.md`, logs por `correlation_id`); transformá-los em indicador é um trabalho a fazer.
> Não apresente estes números como se já estivessem sendo medidos.

> 🧑‍💼 **RESUMO EXECUTIVO.** Três perguntas que a liderança deve conseguir responder com número, não
> com opinião: *(1) a qualidade está subindo?* *(2) o time está aprendendo (errando menos o mesmo
> erro)?* *(3) o processo está sendo seguido ou burlado?* As seções abaixo dão o indicador para cada uma.

---

## 1. Princípio: medir resultado e integridade, nunca atividade

Um erro clássico é medir **atividade** ("quantas tarefas a IA fez", "quantas linhas geradas"). Isso é
**vaidade** — não diz nada sobre qualidade e ainda incentiva o comportamento errado. Medimos duas coisas:

- **Resultado:** a qualidade do que sai (entregas aprovadas, bugs evitados).
- **Integridade do processo:** se o processo está sendo **seguido de verdade** ou burlado (o anti-vaidade).

> Regra de ouro: **se um indicador pode ser melhorado burlando o processo, ele precisa de um indicador
> de integridade ao lado.** Ex.: "mais entregas aprovadas" é ótimo — a menos que estejam aprovando com
> ressalvas demais. Por isso os dois andam juntos.

---

## 2. Indicadores de RESULTADO (a qualidade está subindo?)

| Indicador | Como calcular | Fonte de dados | Lê como |
|---|---|---|---|
| **Taxa de aprovação na 1ª validação** | entregas APROVADAS sem retorno ÷ total validado | relatórios de `validar-entrega` em `.sandbox` | ↑ = plano e execução estão maduros |
| **Taxa de retorno (bola que volta)** | entregas REPROVADAS que voltaram ao `planejar` ÷ total | relatórios de `validar-entrega` | Saudável **não é zero** — zero pode ser validação frouxa |
| **Densidade de regressão** | itens novos em `regression-logs.md` ÷ período | `regression-logs.md` | ↑ = correções não estão atacando a causa raiz |
| **Idade do bug ao corrigir** | tempo entre 1º registro no log e correção | `correlation_id` + `error-backlog.md` | ↓ = observabilidade está ajudando o diagnóstico |
| **Cobertura de observabilidade** | fluxos auditados reconstruíveis por log ÷ auditados | relatórios de `validar-observabilidade` | ↑ = debug offline está viável |

> **O contraintuitivo:** a "taxa de retorno" **não deve ser zero**. Se o `validar-entrega` nunca reprova
> nada, desconfie — ou o time está perfeito (improvável) ou a validação virou carimbo. Um sistema
> saudável **rejeita** uma fração das entregas; é o sinal de que o controle de qualidade está atuando.

---

## 3. Indicadores de APRENDIZADO (o time está ficando melhor?)

| Indicador | Como calcular | Fonte | Lê como |
|---|---|---|---|
| **Lições transversais promovidas** | nº de lições novas em `lessons.md` ÷ período | `lessons.md` (git log) | ↑ controlado = o time captura aprendizado |
| **Reincidência** | erros do `error-backlog` que reaparecem em `regression-logs` | cruzamento dos dois arquivos | ↓ = lições estão de fato prevenindo |
| **Razão de reuso** | quantas vezes um componente do inventário foi reutilizado | `README-TOOLS-LIB.md` + buscas | ↑ = duplicação está sendo evitada |
| **Saúde das instruções** | nº de itens abertos em `bad-instructions.md` | `bad-instructions.md` | ↓ ao longo do tempo = contratos amadurecendo |

> **Cuidado com o `lessons.md`:** "mais lições" **não é sempre melhor**. A regra de curadoria existe
> justamente para o arquivo não virar lixão. O indicador saudável é **lições promovidas com qualidade**
> (passaram no teste de promoção), não volume bruto.

---

## 4. Indicadores de INTEGRIDADE (o processo está sendo seguido ou burlado?)

Estes são os mais importantes e os mais esquecidos. Eles detectam quando a equipe **fura o processo**.

| Sinal de burla | Como detectar | O que significa |
|---|---|---|
| **Implementação sem investigação/plano** | tarefas que foram direto ao `/implementar` | Pulou o eixo — risco de escopo inflado e falso-verde |
| **`src/` editado sem `tests/`** | o nudge `stop-loop` já sinaliza; contar a frequência | Cobertura sendo negligenciada |
| **Disparos de guard** | quantas vezes o `bash-guard` bloqueou (destrutivo/varredura) | Alto e recorrente = falta treino ou há automação perigosa |
| **Nudges ignorados repetidamente** | mesmos sinais do `logging-nudge` reaparecendo | Disciplina de log/Python erodindo |
| **Excesso de "APROVADO COM RESSALVAS"** | proporção sobre o total aprovado | Qualidade entrando por baixo do radar |
| **Fallback sem justificativa** | aparições de fallback/dual-read em diffs sem pedido | Anti-legado sendo burlado |

> **Por que integridade importa mais que resultado:** um time pode mostrar "90% de aprovação" enquanto
> pula o `investigar`, ignora nudges e aprova tudo com ressalvas. O número de resultado parece bom; o
> processo está podre. **Sem os indicadores de integridade, você não enxerga a podridão.**

---

## 5. Como começar a medir (proposta incremental, barata)

Não tente instrumentar tudo de uma vez. Ordem sugerida, do mais barato ao mais elaborado:

1. **Manual / mensal (custo quase zero):** contar itens em `error-backlog.md`, `regression-logs.md`,
   `lessons.md`, `bad-instructions.md` por período (um `git log` resolve). Já responde "estamos
   aprendendo?".
2. **Semi-automático:** um script que varre os relatórios em `.sandbox` (`validacao--*.md`) e conta os
   status (APROVADO / RESSALVAS / REPROVADO / BLOQUEADO). Responde "a qualidade está subindo?".
3. **Automático:** agregar a telemetria da suíte (`telemetry.json`) e eventos de log por `correlation_id`
   num painel. Responde "o processo está sendo seguido?".

> Comece pelo passo 1 nesta semana. É grátis e já dá três meses de tendência para a próxima reunião.

---

## 6. Os anti-indicadores — o que NÃO medir (porque distorce)

- ❌ **Linhas de código geradas** — incentiva volume, não qualidade.
- ❌ **Número de tarefas concluídas pela IA** — vaidade; não diz se ficaram certas.
- ❌ **Velocidade isolada** — sem o par de qualidade, premia o atalho (exatamente o que o `CLAUDE.md`
  proíbe: "qualidade acima de pressa").
- ❌ **Zero reprovações no `validar-entrega`** — não é meta; é sinal de alerta de validação frouxa.

---

## 7. Resumo de bolso

- Meça **resultado** (qualidade sobe?), **aprendizado** (erra menos o mesmo?) e **integridade**
  (processo seguido?).
- **Integridade é o indicador mais negligenciado e o mais importante** — sem ele, o resultado mente.
- Vários sinais já existem em bruto (`error-backlog`, `regression-logs`, relatórios de validação,
  telemetria). Falta **agregá-los** — e isso é um trabalho a fazer, não um número pronto.
- Cuidado com vaidade: linhas geradas e tarefas concluídas **não** são qualidade.

> **Frase para a liderança:** "A pergunta não é 'a IA fez muito?'. É 'a qualidade subiu, o time errou
> menos o mesmo erro, e o processo foi seguido?'. Se a gente não mede isso, está pilotando no escuro —
> exatamente o que esta metodologia foi feita para evitar."

---

**Relacionado (base conceitual):** [Os Dois Loops](os-dois-loops.md) ·
[Cartão de Referência](cartao-de-referencia.md) · [FAQ de Objeções](faq-objecoes.md) ·
[↩ Voltar ao índice da metodologia](../README.md)
</content>
