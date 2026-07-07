---
marp: true
theme: default
paginate: true
size: 16:9
header: 'Engenharia com IA Governada por Contratos'
footer: 'Caso de Negócio — Metodologia de Engenharia de Agentes'
style: |
  section { font-size: 26px; }
  section.lead { text-align: center; }
  h1 { color: #1a4d7a; }
  h2 { color: #1a4d7a; }
  strong { color: #b5651d; }
  section.lead h1 { font-size: 52px; }
  code { background: #eef3f8; }
  table { font-size: 22px; }
---

<!--
COMO USAR ESTE ARQUIVO (Marp)
- Conteúdo visível = o que vai na TELA. Comentários HTML = NOTAS DO APRESENTADOR.
- Público-alvo: CTO / VP de Engenharia / diretoria técnica. Duração-alvo: 30-40 min + perguntas.
- Documento-base com todos os números e fontes: caso-de-negocio-cto.md (mesma pasta).
- Antes de apresentar: leia o FAQ de objeções (../base-conceitual/faq-objecoes.md) — blocos 3, 4 e 5.
-->

# Engenharia com IA Governada por Contratos

## O caso de negócio, com fatos e dados

*Para CTOs e liderança de engenharia*

<!--
FALA: "Não vim apresentar uma ferramenta de IA — vocês já têm acesso às mesmas ferramentas que
todo concorrente. Vim apresentar a única coisa que os dados de 2025 mostram que separa quem ganha
de quem perde com IA: o SISTEMA que governa a ferramenta. E vim com números."
-->

---

<!-- _class: lead -->

# Todo mundo já adotou IA.

## Então por que os resultados não aparecem no P&L de engenharia?

<!--
INTERAÇÃO: pergunte se a percepção interna bate — "os devs dizem que estão mais rápidos.
O lead time melhorou? O change failure rate melhorou?"
Esse desconforto é exatamente o que os dados da indústria capturam. Próximo slide.
-->

---

# Os dados de 2025 (1/2): o paradoxo

| Achado | Fonte |
|---|---|
| **~90% de adoção** de IA entre profissionais de software | DORA 2025 (Google) |
| Mais IA = **mais throughput E mais instabilidade** de entrega | DORA 2025 |
| Devs experientes ficaram **19% mais lentos** com IA — acreditando estar **20% mais rápidos** | METR, ensaio controlado randomizado |
| Só **16,3%** dizem que a IA os tornou muito mais produtivos | Qodo, 2025 |

<!--
FALA: "O dado da METR é o mais importante da apresentação: é um RCT — metade com IA, metade sem.
O grupo com IA foi 19% MAIS LENTO e saiu convencido de que foi 20% mais rápido.
A percepção de produtividade e a produtividade real estão descoladas. Quem mede por percepção
está pilotando no escuro."
-->

---

# Os dados de 2025 (2/2): para onde foi o tempo

**+21%** de tarefas concluídas por dev...

...mas **+98%** de pull requests e **+91%** de tempo de review *(Faros AI)*

## O gargalo mudou: de **escrever** código para **verificar** código.

<!--
FALA: "A IA gera volume. O volume cai na fila de review. O ganho individual evapora na verificação
coletiva. Se a sua estratégia de IA não ataca a VERIFICAÇÃO, ela só move o gargalo para o ponto
mais caro do processo — o tempo dos seus sêniores."
-->

---

# A conclusão do maior estudo do setor

> "O desafio da adoção bem-sucedida de IA **não é um problema de ferramentas —
> é um problema de sistemas.**"

Os maiores retornos vêm de **guardrails, plataformas internas de qualidade
e foco no sistema organizacional** — não da ferramenta.

*— DORA 2025, State of AI-assisted Software Development (Google)*

## IA é um **amplificador**: acelera times com processo, piora times sem.

<!--
FALA: "Este é o slide que resume a tese. O DORA acompanha dezenas de milhares de times há mais de
uma década. A conclusão de 2025: IA amplifica o que você já é. Processo maduro → acelera.
Caos → amplifica o caos. A pergunta certa não é 'qual IA usar', é 'qual sistema governa a IA'."
-->

---

<!-- _class: lead -->

# A proposta:

## um processo de engenharia **inteiro**, codificado em arquivos versionados, que a IA é **obrigada** a seguir

<!--
FALA: "O que vou mostrar agora existe, está rodando em produção num repositório real, e é
inspecionável linha a linha. Não é slideware."
-->

---

# O sistema de referência, em números

| Camada | Volume | Função |
|---|---|---|
| Constituições (`CLAUDE.md`) | 6 arquivos | Princípios sempre em contexto + regras por área |
| Regras profundas | 20 contratos | Carregadas sob demanda, por caminho tocado |
| Agentes especialistas | 20 procedimentos | Papéis segregados, executam isolados |
| Skills de acionamento | 17 gatilhos | Roteiam para o procedimento certo |
| Hooks determinísticos | 7 scripts | **Bloqueiam** o irreversível, auditam o resto |

**~15.900 linhas de contratos** · **9 gates obrigatórios** antes de qualquer ação relevante

<!--
FALA: "Guardem duas coisas: (1) isso é TEXTO VERSIONADO em git — auditável, evolutível, revisável
como código; (2) ninguém escreveu isso de uma vez — cresceu incrementalmente a partir de erro real.
A implantação de vocês começa com 30 linhas, não com 16 mil."
-->

---

# Mecanismo 1 — Pipeline com segregação de funções

```
investigar  →  planejar  →  implementar  →  validar-entrega
(só evidência)  (só plano,     (fiel ao       (veredito independente,
                 crítico)       escopo)         PROIBIDO corrigir)
```

- Cada etapa **materializa um relatório em disco** — auditável e retomável
- Quem valida **não é** quem implementou (e devolve ao planejamento se reprovar)
- O plano é criticado **antes** do código existir

<!--
FALA: "É o mesmo princípio de segregação de funções do financeiro, aplicado à IA.
O validador é instruído a ser advogado do diabo e é PROIBIDO de corrigir — se ele corrige,
vira cúmplice. Isso mata o padrão 'a IA se auto-aprovou'."
-->

---

# Mecanismo 2 — Anti-falso-verde (o "pronto" que mente)

O inimigo público nº 1: *teste passou, doc atualizada...*
**mas o sistema real ainda não usa o código novo.**

A "definição de pronto" contratual exige:

1. Prova de que o **runtime oficial** usa a implementação nova (wiring real)
2. **Teste que falha** se o caminho oficial regredir
3. Log que reconstrói o fluxo sem adivinhação
4. Só **depois** disso, atualizar documentação

<!--
FALA: "Componente novo lindo, testado, documentado — e o endpoint continua chamando o caminho
antigo. Todo CTO já viu isso. Aqui é REPROVAÇÃO automática por contrato: existe uma checklist
de 9 itens que o validador aplica antes de aceitar 'concluído'."
-->

---

# Mecanismo 3 — Rastreabilidade ponta a ponta

- Toda execução real nasce com um **id de correlação** único que atravessa API, workers e jobs
- O log resultante é o **raio-X**: reconstrói o comportamento passo a passo
- Correção de bug é **forense** — confronto log × código até a causa raiz **provada** —
  e é proibido rotular "problema de ambiente" por palpite

## Efeito direto: **MTTR cai** — diagnóstico vira leitura, não arqueologia.

<!--
FALA: "Quando algo quebra em produção, a pergunta 'o que aconteceu?' tem resposta mecânica:
pega o id, abre o log, lê a história. E o mesmo mecanismo é usado pela PRÓPRIA IA para se
corrigir: ela é obrigada a provar a causa raiz pelo log antes de mexer no código."
-->

---

# Mecanismo 4 — O árbitro automático (hooks)

| Tipo | O que faz | Exemplos |
|---|---|---|
| **Guard** (bloqueia) | Impede o irreversível, deterministicamente | `rm -rf`, `DROP TABLE`, `push --force`, exclusão de área alheia |
| **Nudge** (avisa) | Audita cada arquivo editado e injeta o achado | lint, disciplina de log, anti-padrões de Python |

- Não é a IA "se comportando" — é **impossibilidade imposta por script**
- Aplicado no 1º e no 1000º arquivo, **sem fadiga** — o que um revisor humano não sustenta

<!--
FALA: "Calibração importante: só 3 hooks bloqueiam — exatamente o que é irreversível.
O resto é aviso com contexto. Bloquear demais gera fadiga e caça ao bypass; a dosagem é parte
da engenharia."
-->

---

# Mecanismo → ganho (o mapa completo)

| Risco que você paga hoje | Resposta do sistema |
|---|---|
| IA alucina API/schema | Evidência obrigatória: só vale o que foi **lido** do código real |
| Review vira gargalo (+91%) | Humano revisa **evidência** (plano, veredito, prova) — não sintaxe |
| Duplicação/dívida crescem | Gate de reuso: buscar → ler → só então criar |
| Custo de IA descontrolado | Tiering de modelo por risco + contexto carregado sob demanda |
| Time repete o mesmo erro | Lições **comprovadas** promovidas a regra durável (com gate anti-ruído) |
| Processo na cabeça de 2 sêniores | Processo inteiro em git: onboarding = ler; mudança = 1 commit |

<!--
FALA: "Cada linha desta tabela tem um contrato específico por trás, com nome de arquivo.
O detalhamento técnico está no Mapa da Governança — deixo o link no material."
-->

---

# Validação independente: DORA AI Capabilities Model

As **7 capacidades** que os dados do DORA apontam como determinantes do valor de IA —
publicadas **depois** deste desenho:

| Capacidade DORA | Aqui |
|---|---|
| Postura de IA clara e comunicada | ✅ os contratos são a postura, versionada |
| Ecossistema de dados saudável | ✅ log canônico + id de correlação + telemetria |
| Dados internos acessíveis à IA | ✅ ~46 scripts de acesso governado (dry-run default) |
| Controle de versão forte | ✅ 100% da governança e artefatos em git |
| Trabalho em lotes pequenos | ✅ planos fatiados, entrega e prova por fatia |
| Foco no usuário | ✅ fidelidade ao pedido como regra |
| Plataforma interna de qualidade | ✅ suíte como prova + hooks + forense de log |

<!--
FALA: "Isto não é a gente se autoavaliando: é um framework independente, do maior programa de
pesquisa do setor, que mapeia 7 por 7 no que foi construído. Quando o desenho converge com o que
os dados de dezenas de milhares de times apontam, o risco da aposta cai muito."
-->

---

# O que já é mensurável desde o 1º mês

O processo **produz seus próprios indicadores** como subproduto:

- **Vereditos contáveis** por entrega: APROVADO / RESSALVAS / REPROVADO / BLOQUEADO
- **Telemetria de teste** por rodada (`run_id`): passa/falha, duração, histórico
- **Logs por execução**: tempo de diagnóstico, reincidência de erro
- **Trilha git dos contratos**: lições promovidas por período
- **Disparos de bloqueio**: quantas vezes o processo impediu algo destrutivo

3 perguntas com número, não opinião: *qualidade subiu? erra-se menos o mesmo erro? o processo está sendo seguido?*

<!--
FALA: "Transparência: no repositório de referência esses indicadores ainda são agregados
manualmente — a proposta de medição completa está documentada. O ponto é: a matéria-prima nasce
auditável. A maioria das empresas não tem NEM a matéria-prima."
-->

---

# Custos e riscos, sem maquiagem

- **Curadoria é permanente**: contrato sem dono apodrece — 1 sênior, dedicação parcial
- **Tokens/licenças**: custo real e recorrente — contido por desenho (tiering, contexto em camadas), mas entra no TCO
- **Formação**: júnior que só aperta botão sem ler evidência não se forma — responsabilidade de gestão continua
- **Não elimina risco** — reduz classes inteiras dele e torna o resíduo auditável
- **Não decide o que construir** — julgamento de produto e arquitetura segue humano

<!--
FALA: "Prefiro perder o deal a vender 'à prova de falhas'. O material tem um FAQ inteiro de
objeções com respostas honestas — inclusive as desconfortáveis. É de propósito: defesa com exagero
morre na primeira sabatina de engenheiro sênior."
-->

---

# Implantação: 4 fases, sem big-bang

| Fase | Prazo | Entrega | Critério de saída |
|---|---|---|---|
| **0 — Fundação** | semana 1 | constituição ~30 linhas + 2 guards | alucinação pega pelo processo |
| **1 — Eixo central** | semanas 2–4 | 4 papéis segregados + definição de pronto | 1ª entrega ponta a ponta validada |
| **2 — Observabilidade** | mês 2 | id de correlação + log canônico + loop forense | 1º bug corrigido por forense de log |
| **3 — Escala** | mês 3+ | regras sob demanda + tiering + lições + indicadores | 1º ciclo mensal de números à liderança |

**Nenhuma fase começa sem a anterior bater o critério.**

<!--
FALA: "O guia de implantação tem templates prontos, anti-padrões e modelo de maturidade.
O erro clássico é copiar as 16 mil linhas no dia 1 — regra que não nasceu de erro real do SEU
contexto é ruído. Instala-se o MECANISMO de evolução, não o texto pronto."
-->

---

# O ativo é seu — e independe de fornecedor

- O valor **não** está no modelo de IA (commodity, troca-se amanhã)
- Está no **corpo de contratos**: processo de engenharia codificado, versionado, auditável
- Trocar de modelo mantém o rigor: os contratos governam **qualquer executor competente**
- É um ativo que **se valoriza com o uso** (cada lição comprovada vira regra durável)

## Zero lock-in. Patrimônio de processo da empresa.

<!--
FALA: "Para um CTO, este é o argumento estratégico final: todo investimento em prompt individual
evapora quando o dev sai ou o modelo muda. O investimento em contrato fica, compõe e é da empresa."
-->

---

<!-- _class: lead -->

# Proposta de decisão

## Piloto de 4 semanas (Fases 0 + 1), em um repositório real, com critérios de saída objetivos — e números na mesa ao final

<!--
FALA: "Não peço adoção da metodologia inteira. Peço 4 semanas, 1 sênior em dedicação parcial,
1 repositório real. Ao final: entregas validadas ponta a ponta, vereditos contáveis e a decisão
de escalar — ou não — tomada com dado, não com fé. Perguntas?"
-->

---

# Fontes e material de aprofundamento

**Dados externos:** DORA 2025 — State of AI-assisted Software Development (dora.dev) ·
DORA AI Capabilities Model (Google Cloud) · METR 2025 (RCT) · Faros AI — Productivity Paradox ·
Qodo — State of AI Code Quality 2025

**Material interno (links vivos no repositório):**
[Caso de Negócio completo](caso-de-negocio-cto.md) ·
[Guia de Implantação](guia-implantacao.md) ·
[Mapa da Governança](../base-conceitual/mapa-governanca.md) ·
[FAQ de Objeções](../base-conceitual/faq-objecoes.md) ·
[Indicadores](../base-conceitual/indicadores.md)

<!--
Exportar este deck: `marp apresentacao-cto-marp.md --pdf` (ou --pptx).
Todos os números citados têm link direto para a fonte no caso-de-negocio-cto.md.
-->
