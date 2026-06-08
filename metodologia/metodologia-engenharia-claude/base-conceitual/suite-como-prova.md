# A Suíte como Prova — Validação que Vira Evidência Auditável

> **Onde este documento se encaixa.** Os outros materiais explicam *que a entrega só vale ativa no
> runtime* (anti-falso-verde, na [Definição de Pronto](../README.md)). Este documento explica **o
> mecanismo concreto que produz essa prova**: a suíte oficial de testes. Ele é sobre o **papel
> metodológico** da suíte — *por que ela existe e o que ela garante*. O **guia operacional** (comandos,
> modos, flags) vive em `docs/README-TESTS.MD` e o contrato normativo em
> `.claude/rules/suite-testes-instructions.md`; este documento **não os repete** — aponta para eles.

> 🧑‍💼 **RESUMO EXECUTIVO.** A pergunta perigosa é: *"como sei que passou de verdade, e não que alguém
> rodou um comando solto que por acaso deu verde?"*. A resposta é a suíte oficial: ela transforma
> **validação em evidência auditável** — uma rodada deixa **rastro estruturado** (não só uma tela
> verde) que qualquer pessoa pode reabrir depois e conferir. Sem artefato auditável, **não existe
> validação confiável** — existe só uma afirmação convincente.

---

## 1. O conceito central: prova, não opinião

Um comando de teste solto responde "passou ou não passou" **no seu terminal, naquele instante, e some**.
Isso é frágil: cada pessoa (ou agente) poderia escolher um conjunto diferente de testes, esquecer o
ambiente, pular frontend ou backend por conveniência, e **declarar sucesso sem deixar prova**.

A suíte oficial resolve isso criando um **contrato único de execução e evidência**. Ela não nasce da
pasta onde o teste está nem do nome do arquivo — nasce do **boundary oficial** e dos **markers
explícitos** lidos pelo runtime. Em linguagem simples: um teste só entra no recorte quando o catálogo
oficial pede aquele marker, ou quando você passa o seletor explícito. Isso elimina o erro comum de
"este teste é de unidade porque está na pasta `unit`" — **a pasta não decide nada**.

---

## 2. A anatomia da evidência (o que uma rodada deixa para trás)

Cada rodada oficial persiste artefatos numa pasta isolada por execução. Os que importam:

| Artefato | O que é | Quando você abre |
|---|---|---|
| `telemetry.json` | Resumo estruturado da rodada (duração total e por grupo/target) | **Primeiro** — a visão de cima |
| `state.json` | Plano persistido, progresso e resultados agregados | Para ver **o que** rodou e **como** terminou |
| `logs/` | stdout e stderr **por target** | Para o **detalhe fino** do que falhou |
| `reports/` | Reports estruturados dos targets pytest, quando existem | Para inspecionar itens e tempos |
| `reports/*.live-progress.ndjson` | Telemetria viva por item (quando o target emite) | Para acompanhar item a item |

**A ordem de leitura importa:** primeiro o **resumo** (`telemetry.json`), depois o **estado**
(`state.json`), e **só então** os detalhes finos do target que falhou. Mergulhar direto no log de um
target sem ler o resumo é o caminho mais lento.

---

## 3. A disciplina de `run_id` (a identidade da rodada)

Toda rodada tem um `run_id` — a chave que amarra os artefatos àquela execução. Três regras de bolso:

- **Rodada nova / execução fresca:** `run_id` **novo**.
- **Continuar uma rodada interrompida:** **mesmo** `run_id` com `--resume`.
- **Reler uma rodada já gravada, sem reexecutar nada:** **mesmo** `run_id` com `--checkup`.

> 🛠️ `--resume` é **continuidade**, não atalho para pular etapa: ele economiza tempo numa suíte longa,
> mas não autoriza reciclar o `run_id` numa execução nova e independente. `--checkup` é a **lupa
> read-only** oficial: não chama o runner, não cria artefato novo, só inspeciona o que já foi
> persistido (e `--checkup --json` é a primeira visão resumida para automação/agente).

---

## 4. Dois momentos: foco local × fechamento amplo

A metodologia separa duas intenções, e confundi-las custa caro:

- **Ciclo local (foco):** validar rapidamente **o impacto imediato** da mudança. Serve para **iterar**.
- **Fechamento amplo:** provar que o repositório **não ficou com regressão relevante**. Serve para
  **concluir com segurança**.

Foco ajuda a iterar; fechamento ajuda a fechar. Declarar pronto com base só no foco local é uma porta de
entrada para o falso-verde.

---

## 5. O ciclo correto quando a suíte falha

Falha não é beco sem saída — é um mapa apontando onde olhar:

1. ler `telemetry.json` (resumo);
2. ler `state.json` (estado);
3. abrir `logs/` e `reports/` do **target relevante**;
4. **reproduzir no menor escopo confiável**;
5. corrigir a **causa raiz** (não o sintoma);
6. rodar de novo o **menor gate oficial proporcional**.

**O processo incorreto** (proibido): pular teste; mascarar warning; tratar falha de ambiente como bug
de produto sem evidência; declarar sucesso olhando só o exit code.

> Este ciclo é o **loop de auto-correção** ([os dois loops](os-dois-loops.md)) operando com a suíte como
> fonte de verdade — converge para o conserto ou declara bloqueio com evidência, nunca "quase pronto".

---

## 6. Por que `pytest` direto não basta

`pytest` direto continua útil para **reprodução curta** — é a **lupa**. Mas ele **não substitui** a
suíte oficial, porque sozinho não entrega o `run_id` oficial, a telemetria persistida, o estado
resumível nem a integração com os gates do repositório. Em uma frase:

> **`pytest` direto é lupa; a suíte oficial é trilha auditável.** A lupa ajuda a enxergar um ponto; a
> trilha permite que outra pessoa reconstrua a validação inteira depois.

---

## 7. Resumo de bolso

- A suíte transforma **validação em evidência auditável** — rastro estruturado, não tela verde efêmera.
- O recorte vem de **markers e boundary oficial**, **nunca** da pasta ou do nome do arquivo.
- Leia na ordem: **`telemetry.json` → `state.json` → `logs`/`reports`**.
- `run_id`: novo para rodada nova; `--resume` para continuar; `--checkup` para reler.
- **Foco** para iterar; **fechamento amplo** para concluir.
- **Sem artefato auditável, não há validação confiável.**

> **Frase para fechar o tema:** "Verde no terminal é uma opinião que some. A suíte oficial é uma
> **prova que fica** — e prova é o que separa 'parece pronto' de 'está pronto'."

---

**Relacionado:** [Operar a Metodologia](operar-a-metodologia.md) · [Os Dois Loops](os-dois-loops.md) ·
[Ciclo de Vida do Software](ciclo-de-vida-software.md) · [Indicadores](indicadores.md) ·
[↩ Índice da metodologia](../README.md)
·
*Operacional:* `docs/README-TESTS.MD` · *Contrato normativo:* `.claude/rules/suite-testes-instructions.md`
