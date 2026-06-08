# FAQ de Objeções — As Perguntas Difíceis (e respostas honestas)

> **Para que serve.** Esta não é uma FAQ de "o que é um agente" — isso está na doc da Anthropic e seria
> cosmético. Aqui estão as **objeções reais** que um engenheiro cético ou um líder desconfiado levanta
> contra a metodologia. As respostas são **honestas** (incluem os limites) — porque defender com exagero
> destrói a credibilidade. Use na aula (seção de perguntas), no onboarding e quando precisar **defender
> a abordagem** para quem está de fora.
>
> 🧑‍💼 = objeção mais comum da liderança · 🛠️ = objeção mais comum da engenharia

---

## Bloco 1 — Confiança e qualidade

### 🛠️ "Como eu confio em código que ninguém revisou linha por linha?"
A confiança não vem de alguém ter lido as linhas — vem de o código ter passado por **seis filtros que um
revisor humano sozinho não aplicaria com a mesma consistência**: nasceu de evidência real (sem
alucinação), foi construído sob contrato (gates de reuso/banco/padrão), foi provado **ativo no runtime**
(anti-falso-verde), é auditável por log (raio-X), passou por um **revisor independente** (`validar-entrega`,
que não aceita "teste verde" como prova) e por arbitragem automática (hooks). Você revisa **evidência**
(relatórios, plano, veredito, log), não sintaxe. É revisão numa altitude mais alta. *(Detalhe completo no
[Sem Digitar Código](sem-digitar-codigo.md).)*

### 🛠️ "Então o code review acabou? Isso é perigoso."
O code review **como ritual humano separado no fim** ficou redundante — porque cada uma de suas funções
foi **embutida e distribuída** pelo processo: a crítica de arquitetura no `planejar` (antes do código), o
estilo nos hooks (no ato da escrita), o reuso no `implementar` (durante), o veredito no `validar-entrega`
(revisor independente codificado). Não é "menos revisão" — é **mais revisão, contínua e sem fadiga**. O
que acabou foi o **gargalo**, não o controle.

### 🧑‍💼 "E se o resultado estiver sutilmente errado e ninguém perceber?"
É exatamente o que o **anti-falso-verde** ataca. A `definicao-de-pronto.md` exige um **check
discriminante** (o teste mais barato capaz de *desconfirmar* a entrega) e um **teste de regressão
estrutural** (que falha se o caminho oficial voltar ao comportamento antigo). Além disso, a
observabilidade permite reconstruir o comportamento real pelo log. Nenhum sistema elimina 100% do risco
— mas este o reduz mais do que o "humano lendo o 40º diff do dia com pressa".

---

## Bloco 2 — Alucinação e erro da IA

### 🛠️ "E se o agente alucinar uma API ou um schema que não existe?"
É a primeira regra do `CLAUDE.md §1` e o coração do `investigar`: **proibido afirmar sem ler o código
real** (`read`), com path e linha. Comentário e documentação são "pista, nunca prova; código executável
vence sempre". A alucinação não é eliminada por mágica — ela é **estruturalmente desincentivada**: toda
afirmação precisa de evidência rastreável, e o `validar-entrega` confere contra o código executável, não
contra a "memória" da IA.

### 🛠️ "IA erra. O que impede um erro de virar um loop infinito de tentativas?"
Os loops de auto-correção têm **teto explícito** (ex.: ~20 ciclos no `executar-testes`, ~50 no
`testar-ingestao-dnit`) e **memória de rodada** que impede repetir uma hipótese já falsificada. O sistema
**converge ou declara BLOQUEADA com evidência** — nunca fica num "quase pronto" eterno. *(Ver
[Os Dois Loops](os-dois-loops.md).)*

### 🧑‍💼 "Isso funciona num exemplo pequeno. E no nosso sistema real, gigante?"
Funciona **por causa** das travas de escala: a IA não tenta "abraçar" o código todo (seria impossível e
caro). Ela é obrigada a começar numa **âncora concreta**, ler só o necessário e **provar por fatia**, com
proibição de varredura cega de logs e isolamento de contexto entre agentes. *(Ver
[Travas para Projetos Grandes](travas-projetos-grandes.md).)*

---

## Bloco 3 — Responsabilidade e risco

### 🧑‍💼 "Se quebrar em produção, quem é o responsável?"
A mesma pessoa de sempre: **o engenheiro que conduziu e aprovou**. A metodologia não terceiriza
responsabilidade para a IA — ela **muda o que o humano aprova** (evidência e decisão, em vez de sintaxe).
Quem aprova um plano ruim responde pela execução fiel de um plano ruim. A diferença é que agora há uma
**trilha de evidência auditável** (investigação, plano, validação, log por `correlation_id`) para
entender o que aconteceu — geralmente *melhor* do que num fluxo tradicional.

### 🧑‍💼 / 🛠️ "E segurança? A IA não pode rodar um `rm -rf` ou vazar um segredo?"
Há travas determinísticas (guards) para isso: o `bash-guard` **bloqueia** comandos destrutivos
(`rm -rf`, `drop table`, `git push --force`) e a varredura cega de logs; os agentes de infra operam em
**dry-run** e só executam mutação com flag explícita; o `CLAUDE.md §8` faz o catálogo de tools **falhar
fechado**; e logar dado sensível é proibido (só shape/contagem/decisão). Não é confiança no bom
comportamento da IA — é **impossibilidade imposta pelo árbitro automático**. *(Detalhe dos hooks na
seção de arbitragem de cada lente.)*

---

## Bloco 4 — Dependência e estratégia

### 🧑‍💼 "Isso não cria dependência (lock-in) de um modelo de IA específico?"
Não, e esse é o ponto mais estratégico. **O ativo não é o agente — é o corpo de contratos** (`CLAUDE.md`,
`rules/`, agentes, hooks). Se trocarmos o modelo de IA amanhã, os contratos continuam valendo e continuam
impondo o mesmo rigor a qualquer executor competente. O que construímos é **independente do modelo**: é
um processo de engenharia codificado, não um vínculo com um fornecedor. *(Ver
[Sem Digitar Código, seção D.6](sem-digitar-codigo.md).)*

### 🧑‍💼 "Qual o ROI disso? Por que investir tempo escrevendo contratos?"
Porque o contrato é escrito **uma vez** e aplicado em **toda tarefa, para sempre**. Um padrão de code
review que você cobra em reunião se perde; escrito como contrato + hook, ele é aplicado automaticamente
no 1º e no 1000º arquivo, sem custo marginal. O investimento em contrato é **capitalizado**: cada regra
previne uma classe inteira de erros futuros. *(Métricas concretas para acompanhar isso em
[indicadores.md](indicadores.md).)*

---

## Bloco 5 — Pessoas e cultura

### 🛠️ "Isso não vai atrofiar o time? Devs que não escrevem código viram o quê?"
Viram **engenheiros de mais alavancagem**. O esforço migra de digitar para: **investigar bem, criticar um
plano, ler evidência, curar contratos**. Essas são competências *mais* sêniores, não menos. O risco real
existe e deve ser gerido: um júnior que só aperta `/implementar` sem entender o que lê **não** se forma. A
metodologia ajuda (o nível 101 obrigatório, os relatórios didáticos), mas a formação do time continua
sendo responsabilidade humana.

### 🛠️ "E quando a IA simplesmente faz besteira que um sênior nunca faria?"
Acontece — e por isso existe a **postura crítica obrigatória** (o `planejar` é mandado a discordar de
ideias ruins) e a **segregação de funções** (quem implementa não se auto-aprova). Mas a salvaguarda final
é humana: você lê o plano e o veredito. O sistema é desenhado para **escalar a dúvida** (status BLOQUEADA,
"decisão externa") em vez de esconder, justamente para que a besteira apareça antes de virar produção.

### 🧑‍💼 "Isso substitui meus engenheiros?"
Não. Substitui a **digitação e o code review-gargalo**, não o **julgamento de engenharia**. Decidir *o
que* construir, aprovar arquitetura realmente nova, curar os contratos e ler a evidência criticamente
continuam sendo trabalho humano — e mais valioso do que antes. *(A leitura honesta dos limites está no
[Ciclo de Vida do Software, seção C.5](ciclo-de-vida-software.md) e [seção D.7](sem-digitar-codigo.md).)*

---

## Bloco 6 — Manutenção da própria metodologia

### 🛠️ "E se os contratos ficarem desatualizados ou contraditórios?"
Há um agente dedicado a isso: o `validar-instructions` audita as próprias regras em busca de contradição,
redundância e ambiguidade, e registra em `bad-instructions.md`. E a regra de ouro recorrente — **"se o
contrato divergir do código executável, o código vence"** — impede que uma regra velha mande no
comportamento real. A metodologia tem mecanismo de auto-auditoria.

### 🛠️ "Quem garante que um novo agente/skill seja criado do jeito certo?"
Hoje, a consistência vem das próprias regras (reuso, anti-duplicação) e da revisão humana. Esse é,
honestamente, um ponto que **se beneficia de um checklist formal de extensão** — recomendado como próximo
artefato no [cartao-de-referencia.md](cartao-de-referencia.md). É um exemplo de onde a metodologia ainda
depende de disciplina humana.

---

## Como usar esta FAQ na aula

- Não leia tudo. Deixe-a **de reserva** para a seção de perguntas — quando alguém levantar uma objeção,
  você já tem a resposta honesta estruturada.
- Para liderança, priorize os blocos **3 (risco), 4 (estratégia) e 5 (pessoas)**.
- Para engenharia, priorize os blocos **1 (qualidade), 2 (alucinação) e 6 (manutenção)**.
- **Regra de ouro ao responder:** sempre inclua o limite honesto. "Reduz o risco, não elimina" convence
  mais que "é à prova de falhas" — e protege você quando algo der errado.

---

**Relacionado (base conceitual):** [Sem Digitar Código](sem-digitar-codigo.md) ·
[Os Dois Loops](os-dois-loops.md) · [Indicadores](indicadores.md) ·
[↩ Voltar ao índice da metodologia](../README.md)
</content>
