# Guia de Implantação — Como Levar a Metodologia para Qualquer Empresa de Software

> **O que é este documento.** O roteiro de adoção passo a passo, no formato de uma consultoria: o que
> é portável e o que é específico do repositório de referência, os pré-requisitos, as **4 fases de
> implantação** (cada uma com entregáveis, critério de saída e armadilhas), os templates mínimos para
> começar e o modelo de maturidade para medir progresso. O *porquê* (dados, ganhos, custos) está no
> [Caso de Negócio](caso-de-negocio-cto.md); a anatomia completa do sistema está no
> [Mapa da Governança](../base-conceitual/mapa-governanca.md).

> 🧑‍💼 **RESUMO EXECUTIVO.** A implantação é **incremental e autofinanciada em valor**: a Fase 0 cabe
> em uma semana com ~30 linhas de constituição e 2 bloqueios automáticos, e cada fase seguinte só
> começa quando a anterior provou valor com critério objetivo de saída. Não existe big-bang: copiar as
> ~16 mil linhas do repositório de referência no dia 1 é o **anti-padrão nº 1** — os contratos maduros
> são resultado de evolução guiada por erro real, e é esse mecanismo de evolução (não o texto pronto)
> que a empresa precisa instalar. Papel-chave: **um curador de contratos** (engenheiro sênior,
> dedicação parcial) que transforma cada lição real em regra durável.

---

## 1. O que é portável e o que não é

Antes de copiar qualquer coisa, separar três categorias. Errar essa separação é implantar burocracia
de outro contexto:

| Categoria | O que inclui | Como tratar |
|---|---|---|
| **Núcleo portável** (vale para qualquer empresa) | O modelo de camadas (constituição sempre em contexto + regras profundas sob demanda + agentes-procedimento + skills-gatilho + hooks determinísticos); o pipeline com segregação de funções (investigar → planejar → implementar → validar) e artefatos materializados; os princípios (evidência acima de suposição, anti-falso-verde, reuso antes de criar, escopo mínimo); os dois loops (correção por evidência + promoção de lição); o tiering de modelo por risco | Adotar como está — é o desenho, não o texto |
| **Adaptável** (o conceito vale; a implementação é sua) | O contrato de rastreabilidade (`correlation_id` aqui; pode ser trace-id do seu APM); a "suíte como prova" (aqui um runner próprio com telemetria; na sua empresa, o seu CI com artefatos auditáveis); os scripts de acesso governado a dados (reescrever para os seus stores); os hooks (mesmos eventos, regex adaptado ao seu stack) | Reescrever mantendo a função: *prova auditável, acesso governado, bloqueio determinístico* |
| **Específico do repositório de referência** (não copiar) | Regras de produto (YAML-first, AST agentic, contratos LangChain/LangGraph); regras de domínio dos `CLAUDE.md` aninhados; skills de teste E2E de telas específicas; a proibição de DDL em runtime *como está escrita* (o princípio — mudança de schema fora do caminho quente — vale; o texto é do contexto deles) | Usar como **exemplo de forma**: é assim que uma regra madura se parece quando a sua empresa escrever as dela |

## 2. Pré-requisitos

Sem isso, resolver primeiro (é barato e necessário de qualquer forma):

1. **Git como fonte única** — toda a governança vive versionada; sem isso não há auditoria nem evolução controlada.
2. **Algum teste executável** — o pipeline valida com prova; se hoje não há nenhum teste, a Fase 1 começa criando o esqueleto mínimo (o processo, aliás, força essa dívida a aparecer).
3. **Logging minimamente estruturado** (ou disposição de criá-lo na Fase 2) — o loop de correção forense depende de log que conte a história.
4. **Uma ferramenta de agente com suporte a arquivos de instrução, subagentes e hooks** (Claude Code é a referência; o desenho não depende dele — ver [lock-in no FAQ](../base-conceitual/faq-objecoes.md)).
5. **O curador de contratos**: um engenheiro sênior com autoridade para escrever regra, dedicação parcial (ordem de horas por semana, não full-time). Sem dono, os contratos apodrecem — e contrato podre é regra ruim aplicada com eficiência.

## 3. As quatro fases

> Regra transversal: **nenhuma fase começa sem a anterior ter batido o critério de saída.** O critério
> é objetivo de propósito — é ele que impede a implantação de virar teatro de processo.

### Fase 0 — Fundação (semana 1)

**Objetivo:** parar a sangria dos erros de IA mais caros (alucinação e ação destrutiva) com o mínimo de texto.

**Entregáveis:**
- **Constituição mínima** (~30–60 linhas, template na seção 4): princípios inegociáveis — evidência
  obrigatória (proibido afirmar API/schema/comportamento sem ler o código), escopo mínimo, proibições
  (gambiarra, fluxo duplicado, código sem teste), e onde os artefatos de trabalho moram.
- **2 guards determinísticos**: bloqueio de comandos destrutivos (`rm -rf`, `DROP/TRUNCATE`,
  `push --force`) e do que for irreversível no seu contexto.
- **Convenção de artefatos**: uma pasta versionada para investigações/planos/validações.

**Critério de saída:** uma semana de uso real em que (a) nenhuma afirmação inventada passou sem ser
pega pelo próprio processo ("mostre onde leu isso") e (b) os guards dispararam ou provaram-se
desnecessários — ambos são aprendizado.

**Armadilha típica:** escrever 500 linhas de constituição no dia 1. Regra que não nasceu de erro real
do *seu* contexto é ruído que dilui as regras que importam.

### Fase 1 — O eixo central (semanas 2–4)

**Objetivo:** instalar o pipeline com segregação de funções — a mudança estrutural que ataca o
gargalo de review.

**Entregáveis:**
- **4 agentes** (templates na seção 4): `investigar` (só evidência, proibido propor plano),
  `planejar` (só plano, com postura crítica obrigatória — mandado a discordar de ideia ruim),
  `implementar` (executa o plano, fiel ao escopo), `validar-entrega` (veredito independente,
  **proibido corrigir**; achou não-entrega → devolve ao planejamento).
- **Artefatos materializados obrigatórios**: cada etapa grava relatório em disco ("resposta só no
  chat é inválida") — é o que torna o processo auditável e retomável.
- **Definição de pronto anti-falso-verde**: entrega só fecha com o caminho oficial de runtime usando
  o código novo + teste que falha se regredir (adaptar de `definicao-de-pronto.md`).

**Critério de saída:** primeira entrega real conduzida ponta a ponta pelo pipeline, com relatório de
validação independente — e pelo menos **uma** reprovação legítima no período (validador que nunca
reprova é carimbo, não controle; ver [Indicadores](../base-conceitual/indicadores.md)).

**Armadilha típica:** deixar o mesmo agente (ou a mesma janela de contexto) implementar e se
auto-aprovar "para agilizar". É exatamente o conflito de interesse que a segregação existe para matar.

### Fase 2 — Observabilidade e loops (mês 2)

**Objetivo:** transformar erro de "problema" em "matéria-prima": correção forense por evidência e
prova de teste auditável.

**Entregáveis:**
- **Rastreabilidade por execução**: um id de correlação que nasce no boundary e atravessa tudo
  (adaptar do contrato de `correlation_id`; se já existe tracing no APM, integrar, não duplicar).
- **Log canônico com campos oficiais** nos fluxos principais (entrada, decisões, chamadas externas,
  erro, término) — o suficiente para reconstruir uma execução sem adivinhar.
- **Loop de auto-correção por log** (adaptar de `loops-estrategicos.md`): erro real → capturar o id →
  abrir o log → confrontar log × código até causa raiz **provada** → corrigir na origem → proteger com
  teste. Proibido rotular "problema de ambiente" por inferência.
- **Suíte como prova**: o runner de testes (o seu CI serve) passa a produzir artefato auditável por
  rodada — quem passou, quem falhou, quanto durou — e "verde" passa a ser aferível, não declarado.

**Critério de saída:** primeiro bug de produção/homologação corrigido por leitura forense de log
(com a causa raiz demonstrada no relatório), não por tentativa e erro.

**Armadilha típica:** instrumentar tudo de uma vez. Comece pelos 2–3 fluxos que mais doem; o loop de
correção vai puxando a instrumentação para onde ela falta (toda vez que o log não conta a história, a
lacuna de observabilidade vira item da entrega).

### Fase 3 — Escala, custo e aprendizado (mês 3 em diante)

**Objetivo:** o que transforma um piloto em sistema permanente.

**Entregáveis:**
- **Regras profundas sob demanda** (path-scoped): quando a constituição passar de ~150 linhas,
  extrair contratos por área que só carregam quando o caminho tocado os exige — contexto barato e
  regra sempre presente onde importa.
- **Constituições por área** (equivalente aos `CLAUDE.md` aninhados): regra de domínio junto do
  código que ela governa, com precedência explícita ("a raiz é a base").
- **Tiering de modelo por risco** (adaptar de `regras_uso_subagentes.md`), incluindo a trava do falso
  negativo: conclusão de ausência/completude ("não existe", "está coberto") fica sempre no modelo
  forte.
- **Execução resumível para planos longos**: diário write-ahead por tarefa; queda de sessão perde no
  máximo a tarefa em voo.
- **Memória de lições com gate de promoção**: só lição **comprovada** vira regra durável; ruído
  operacional não sobe. É o mecanismo que faz os contratos melhorarem com o uso em vez de inchar.
- **Indicadores mensais** (adaptar de [Indicadores](../base-conceitual/indicadores.md)): resultado,
  aprendizado e — o mais importante — **integridade** (o processo está sendo seguido ou burlado?).

**Critério de saída:** primeiro ciclo mensal de indicadores apresentado à liderança com números reais
(taxa de aprovação na 1ª validação, reincidência de erro, disparos de guard).

**Armadilha típica:** medir atividade ("tarefas que a IA fez", "linhas geradas") — é vaidade e
incentiva o comportamento errado. Os anti-indicadores estão documentados em
[Indicadores §6](../base-conceitual/indicadores.md).

## 4. Templates mínimos

Esqueletos genéricos para a Fase 0/1 — deliberadamente curtos; eles crescem com erro real, não com
antecipação. (Para ver a forma madura de cada um, use o repositório de referência via
[Mapa da Governança](../base-conceitual/mapa-governanca.md).)

**Constituição mínima (`CLAUDE.md` raiz):**

```markdown
# Regras do repositório

## Princípios inegociáveis
- Evidência, não invenção: proibido afirmar API, schema, endpoint ou comportamento
  sem ter lido o código real nesta sessão. Doc e comentário são pista; código é prova.
- Escopo mínimo: resolver exatamente o pedido, com a menor complexidade que funcione.
  Proibido adicionar features, camadas ou generalizações não pedidas.
- Proibido: gambiarra, fluxo duplicado, código morto, mudança relevante sem teste.
- Na dúvida de escopo ou de fato: perguntar/declarar a lacuna, nunca presumir.

## Definição de pronto
- Entrega só fecha quando o caminho oficial de runtime usa o código novo,
  com teste que falha se regredir. Teste verde isolado não prova conclusão.

## Artefatos de trabalho
- Investigações, planos e validações são gravados em docs/trabalho/<tarefa>/.
- Temporários em .sandbox/tmp/, nunca na raiz nem em /tmp.
```

**Agente (ex.: `validar-entrega`) — o padrão dos 4 do eixo:**

```markdown
---
name: validar-entrega
description: Valida uma entrega contra o plano. Use após o implementar concluir.
---
Você é o fiscal independente da entrega — advogado do diabo, não comparsa do executor.
- Entrada: o plano aprovado + o diff/estado atual. Sem plano, devolva BLOQUEADO.
- Confira cada item do plano contra o CÓDIGO REAL (leia; não aceite o relato do executor).
- Exija a prova anti-falso-verde: o runtime oficial usa o código novo? Há teste que
  falha se regredir? Sem essas provas: REPROVADO ou BLOQUEADO POR FALTA DE EVIDÊNCIA.
- É PROIBIDO corrigir código. Achou não-entrega → descreva-a e devolva ao planejamento.
- Saída: relatório em docs/trabalho/<tarefa>/validacao--<data>.md com veredito
  (APROVADO / APROVADO COM RESSALVAS / REPROVADO / BLOQUEADO) e evidência por item.
```

**Skill dispatcher (gatilho fino — o procedimento vive no agente):**

```markdown
---
name: validar-entrega
description: Use when: validar uma entrega concluída contra o plano aprovado.
---
Esta skill é apenas o atalho de entrada. Todo o procedimento vive no agente
`validar-entrega` — invoque-o com o contexto da tarefa. Nada aqui duplica o agente:
para mudar regra ou comportamento, edite o agente, não esta skill.
```

**Hook guard (bloqueio determinístico de comando destrutivo):**

```bash
#!/usr/bin/env bash
# PreToolUse(Bash): nega comandos destrutivos. Falha aberta (erro do hook não trava o dev).
cmd=$(jq -r '.tool_input.command // empty' 2>/dev/null) || exit 0
if echo "$cmd" | grep -qiE 'rm -rf|drop table|truncate table|push +--force'; then
  echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny",
    "permissionDecisionReason":"Comando destrutivo bloqueado por política. Peça ao humano."}}'
fi
exit 0
```

**Regra profunda path-scoped (Fase 3 — carrega só onde vale):**

```markdown
---
paths:
  - "src/pagamentos/**"
---
# Contrato do módulo de pagamentos
- Toda operação de escrita é idempotente por chave de negócio; retry nunca duplica cobrança.
- Proibido logar PAN/CPF; logar apenas shape, contagem e decisão.
```

## 5. Anti-padrões de implantação (os erros que já vimos)

1. **Big-bang de contratos** — copiar a governança inteira de outro contexto. Resultado: regras que
   ninguém entende, não são cobradas e desmoralizam as que importam.
2. **Contratos sem dono** — sem curador, a primeira regra desatualizada ensina o time a ignorar todas.
3. **Pular a segregação de funções** — "o implementador valida rapidinho" devolve o falso-verde que o
   pipeline existe para matar.
4. **Hook que bloqueia demais** — guard em cima de operação rotineira gera fadiga e caça ao bypass.
   Calibrar como o repositório de referência: **bloqueio só para o irreversível**; disciplina é nudge.
5. **Medir atividade** — volume de tarefas/linhas geradas como KPI corrompe o processo no primeiro mês.
6. **Prometer "sem humanos"** — a metodologia muda *o que* o humano faz (curar, criticar plano, ler
   evidência); vender eliminação de engenheiro destrói a credibilidade e o engajamento de quem opera.
7. **Doc antes da prova** — atualizar documentação/promessa antes de o runtime entregar o
   comportamento. A ordem é sempre: wiring → teste → validação → só então doc.

## 6. Modelo de maturidade

Para diagnóstico inicial (onde a empresa está) e para vender progresso (onde estará por fase):

| Nível | Nome | O que existe | O que a empresa ganha |
|---|---|---|---|
| 0 | **IA ad hoc** | Cada dev com seu prompt; zero contrato | A média dos dados do [Caso de Negócio §1](caso-de-negocio-cto.md): velocidade percebida, instabilidade real |
| 1 | **Governado** (≈ Fase 0) | Constituição mínima + guards | Alucinação estruturalmente desincentivada; destrutivo bloqueado |
| 2 | **Pipeline** (≈ Fase 1) | 4 papéis segregados + artefatos + definição de pronto | Review vira leitura de evidência; falso-verde atacado na raiz |
| 3 | **Auditável** (≈ Fase 2) | Rastreabilidade + log canônico + loop forense + suíte como prova | MTTR cai; "verde" passa a ser aferível; erro vira matéria-prima |
| 4 | **Sistema que aprende** (≈ Fase 3) | Regras sob demanda + tiering de custo + lições promovidas + indicadores | Custo governado, escala em repo grande, processo que melhora com o uso e sobrevive a pessoas |

**Estimativa honesta de esforço** (ordem de grandeza, não cotação): Fases 0–1 cabem em ~2–4 semanas
de um sênior em dedicação parcial conduzindo tarefas reais pelo pipeline; Fase 2 depende do estado da
observabilidade existente (de dias, se há tracing/log estruturado, a semanas, se não há nada); Fase 3
é contínua por natureza — o custo recorrente é a curadoria (horas por semana), e é o melhor dinheiro
gasto do programa inteiro.

---

**Navegação:** [← Caso de Negócio](caso-de-negocio-cto.md) ·
[Mapa da Governança](../base-conceitual/mapa-governanca.md) ·
[Operar a Metodologia](../base-conceitual/operar-a-metodologia.md) ·
[Indicadores](../base-conceitual/indicadores.md) ·
[↩ Voltar ao índice da metodologia](../README.md)
