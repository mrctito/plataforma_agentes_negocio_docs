---
marp: true
theme: default
paginate: true
size: 16:9
header: 'IA no Desenvolvimento de Software — Decisão Estratégica'
footer: 'Apresentação ao Conselho Consultivo'
style: |
  section { font-size: 27px; }
  section.lead { text-align: center; }
  h1 { color: #1a4d7a; }
  h2 { color: #1a4d7a; }
  strong { color: #b5651d; }
  section.lead h1 { font-size: 50px; }
  table { font-size: 23px; }
---

<!--
COMO USAR ESTE ARQUIVO (Marp)
- Conteúdo visível = TELA. Comentários HTML = NOTAS DO APRESENTADOR.
- Público-alvo: conselho consultivo / board — SEM jargão técnico. Duração-alvo: 15-20 min + perguntas.
- Regra de ouro deste deck: cada conceito técnico entra por analogia de negócio (gestão de qualidade,
  segregação de funções, auditoria) — nunca por vocabulário de engenharia.
- Números e fontes: caso-de-negocio-cto.md (mesma pasta).
-->

# IA no Desenvolvimento de Software

## A decisão não é "usar ou não usar".
## É "governar ou não governar".

*Apresentação ao Conselho Consultivo*

<!--
FALA: "Vou pedir 15 minutos para uma decisão que afeta custo, qualidade e risco da nossa operação
de tecnologia. Começo com uma constatação: a decisão de USAR inteligência artificial no
desenvolvimento já foi tomada — pelos nossos desenvolvedores, com ou sem política. A decisão que
sobrou para nós é se isso acontece de forma governada ou invisível."
-->

---

# O contexto, em três fatos

1. **IA em desenvolvimento virou commodity**: ~90% dos profissionais já usam
   *(DORA 2025 — programa de pesquisa do Google, o maior do setor)*

2. Nossos concorrentes têm acesso **às mesmas ferramentas** que nós

3. Portanto, a ferramenta **não é** diferencial competitivo — o **modo de usar** é

<!--
FALA: "Não há vantagem em comprar a mesma licença que todo mundo compra. A pesquisa mostra que a
diferença entre quem ganha e quem perde com IA não está na ferramenta — está no sistema de gestão
em volta dela. É exatamente onde proponho investir."
-->

---

# O paradoxo que a pesquisa de 2025 revelou

| O que parece | O que os dados mostram |
|---|---|
| "Os times estão mais rápidos" | Em estudo controlado, profissionais experientes ficaram **19% mais lentos** — convencidos de estarem 20% mais rápidos |
| "Produzimos mais" | O volume produzido **dobrou a fila de verificação** (+91% de tempo de revisão) |
| "A IA escreve, nós economizamos" | Mais velocidade de escrita veio com **mais instabilidade** nas entregas |

## Velocidade **percebida** subiu. Resultado **medido**, não.

<!--
FALA: "Este é o coração do problema. A sensação de produtividade é real — o resultado no negócio,
não necessariamente. Empresas que medem por sensação estão tomando decisão no escuro. E o custo
não aparece na conta de licença: aparece em retrabalho, incidente e no tempo dos profissionais
mais caros da casa, que viraram revisores em tempo integral."
-->

---

<!-- _class: lead -->

# Uma analogia

## IA sem governança é contratar mil estagiários brilhantes —
## sem chefia, sem processo e sem controle de qualidade.

<!--
FALA: "Brilhantes, incansáveis, baratos — e capazes de produzir um volume enorme de trabalho que
ninguém consegue conferir. Nenhum conselho aprovaria essa contratação sem estrutura de gestão.
É a situação em que a maioria das empresas de software está hoje, sem perceber."
-->

---

# A resposta: um sistema de gestão da qualidade para a IA

O que propomos é o equivalente a uma **certificação ISO da fábrica de software**:

- **Regras escritas e versionadas** que a IA é obrigada a seguir (não "boa vontade")
- **Segregação de funções**: quem executa **nunca** aprova a própria entrega
- **Prova em cada entrega**: nada é "pronto" sem evidência verificável de funcionamento
- **Trilha de auditoria completa**: cada execução é rastreável do início ao fim
- **Bloqueios automáticos**: ações destrutivas são impedidas por sistema, não por confiança

<!--
FALA: "Cada item desse slide tem implementação concreta e inspecionável — não é intenção, é
mecanismo. O detalhe técnico está no material anexo; para esta mesa, o que importa é que são os
mesmos princípios de controle que o conselho já conhece de gestão financeira e de qualidade
industrial — aplicados à produção de software com IA."
-->

---

# Por que isso é um **ativo**, não uma despesa

- Uma regra escrita **uma vez** é aplicada em **toda tarefa, para sempre** — sem custo marginal
- O sistema **independe do fornecedor de IA**: trocamos de tecnologia amanhã, o processo fica
- Fica **na empresa**, versionado como patrimônio — não na cabeça de quem pode sair
- **Melhora com o uso**: cada erro real, uma vez provado, vira regra que o previne

## Zero dependência de fornecedor. Valor que compõe com o tempo.

<!--
FALA: "Este é o argumento patrimonial. Investimento em treinamento individual evapora com o
turnover. Investimento em ferramenta deprecia a cada lançamento. Investimento em PROCESSO
CODIFICADO fica, compõe e é auditável — é o único dos três que se comporta como ativo."
-->

---

# Validação externa

O maior programa de pesquisa do setor (DORA/Google, 2025) concluiu:

> "O desafio da IA **não é um problema de ferramentas — é um problema de sistemas.**"
> Os maiores retornos vêm de **guardrails e plataformas internas de qualidade.**

E publicou as **7 capacidades organizacionais** que separam quem extrai valor de quem não extrai.

## O sistema proposto implementa as 7. *(mapeamento no material anexo)*

<!--
FALA: "Não pedimos que o conselho acredite em nós: o desenho converge com o que a pesquisa
independente, sobre dezenas de milhares de times, recomenda. Publicada, aliás, DEPOIS de o
sistema de referência existir — não foi engenharia para passar no teste."
-->

---

# Os riscos — colocados na mesa

**De fazer:**
- Custo recorrente real (licenças/consumo de IA + horas de curadoria das regras)
- Exige disciplina de gestão: regra sem dono apodrece
- Formação do time continua sendo responsabilidade nossa — o sistema não forma júnior sozinho

**De não fazer:**
- O uso de IA continua — **invisível, sem auditoria e sem controle de qualidade**
- Pagamos o paradoxo da pesquisa: sensação de velocidade, instabilidade real
- Concorrentes que governarem primeiro ganham margem **e** confiabilidade

<!--
FALA: "Não existe a opção 'risco zero'. Existe risco governado com trilha de auditoria, ou risco
invisível. Proibir também não funciona: a pesquisa mostra que o uso continua por baixo —
só que sem nenhum controle."
-->

---

# O que pedimos ao conselho

**Aval para um programa em fases — não um cheque em branco:**

| Etapa | Prazo | Investimento | Portão de decisão |
|---|---|---|---|
| **Piloto** | 4 semanas | 1 profissional sênior, dedicação parcial | Resultados medidos na mesa |
| **Expansão** | trimestre seguinte | Time piloto ampliado | Indicadores trimestrais |
| **Programa permanente** | contínuo | Curadoria como função formal | Revisão anual pelo conselho |

Cada etapa só avança se a anterior **provar valor com número**.

<!--
FALA: "A estrutura de decisão é a mesma de qualquer investimento em fases com stage-gates.
O piloto é barato de propósito: 4 semanas, uma pessoa em tempo parcial, um projeto real.
Se os números não convencerem, paramos com aprendizado e custo mínimo."
-->

---

# Como mediremos sucesso (número, não opinião)

Três perguntas que passaremos a responder ao conselho, por trimestre:

1. **A qualidade subiu?** — taxa de entregas aprovadas na 1ª verificação; incidentes evitados
2. **Erramos menos o mesmo erro?** — reincidência de defeitos; lições incorporadas às regras
3. **O processo está sendo seguido ou burlado?** — indicadores de integridade
   (o mais importante: resultado sem integridade é número que mente)

**Anti-vaidade:** não mediremos "volume produzido pela IA" — volume sem qualidade é passivo, não ativo.

<!--
FALA: "Detalhe de governança que peço que o conselho guarde: o terceiro indicador — integridade —
é o que impede o teatro. Um time pode mostrar 90% de aprovação burlando o processo. Medimos as
duas coisas juntas, sempre."
-->

---

<!-- _class: lead -->

# Síntese para decisão

## A IA já está na nossa operação.
## A escolha do conselho é entre **risco invisível** e **vantagem governada** — começando por um piloto de 4 semanas com portões de decisão.

<!--
FALA: "Recomendação da diretoria técnica: aprovar o piloto. Trago os resultados, com números,
na próxima reunião. Perguntas?"
-->

---

# Anexos e fontes

**Para aprofundamento da diretoria técnica:**
[Caso de Negócio completo (com todas as fontes)](caso-de-negocio-cto.md) ·
[Guia de Implantação em fases](guia-implantacao.md) ·
[Apresentação técnica para CTO](apresentacao-cto-marp.md)

**Pesquisas citadas:** DORA 2025 — State of AI-assisted Software Development (Google) ·
DORA AI Capabilities Model · METR 2025 (estudo controlado randomizado) ·
Faros AI 2025 · Qodo — State of AI Code Quality 2025

<!--
Exportar este deck: `marp apresentacao-conselho-marp.md --pdf` (ou --pptx).
-->
