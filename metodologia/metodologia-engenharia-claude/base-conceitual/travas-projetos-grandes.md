# As Travas para Projetos Muito Grandes

> **Documento da base conceitual — neutro a qualquer lente.**

> **Por que este documento existe.** Várias decisões do sistema só fazem sentido quando você entende
> **uma restrição física brutal**: a IA tem uma "memória de trabalho" limitada (a janela de contexto),
> e o repositório é gigante. Forçar a IA a "ler tudo" é impossível e, se tentado, trava a sessão e
> estoura o custo. Este documento reúne, num lugar só, todas as **travas de escala** — que nas lentes
> aparecem espalhadas.

> 🧑‍💼 **RESUMO EXECUTIVO.** Imagine pedir a um consultor para "entender nossa empresa inteira" antes
> de resolver um problema pequeno: ele levaria meses e ainda assim entenderia mal. Nossas travas fazem
> o oposto — obrigam a IA a **começar por um ponto concreto, ler só o necessário e provar conclusão por
> fatia**, em vez de fingir que abraçou o sistema todo. É o que permite usar IA num código enorme **sem
> que o custo exploda e sem conclusões inventadas**.

---

## B.0 O problema físico que todas estas travas atacam

```
   Repositório gigante  ✗  Janela de contexto limitada
   ───────────────────────────────────────────────
   Tentar "ler tudo"  →  estoura contexto · custo alto · sessão lenta · trava
   "Achar por busca rasa e concluir"  →  conclusão arquitetural inventada
```

A engenharia aqui é uma só ideia repetida em vários níveis: **trabalhar por âncora e por fatia, com
leitura sob demanda, nunca por abraço global.**

---

## B.1 Trava de navegação — `large-repo-navigation.md` (o manual de leitura de um sistema enorme)

O contrato dedicado ao tema. Suas regras-chave:

- **Âncora obrigatória.** Toda execução começa por um ponto concreto: um arquivo dono, um símbolo, um
  endpoint, um teste falhando, um log com `correlation_id`, um stack trace. *Sem âncora, a primeira
  tarefa é achar uma.*
- **Busca localiza, leitura comprova.** Uma busca textual **não é prova** — só aponta candidatos. A
  conclusão só vem depois de **ler** o trecho relevante. (É o mesmo princípio "evidência > suposição"
  do DNA, aplicado à escala.)
- **Orçamento de exploração inicial.** Ler só o suficiente para formular uma hipótese local
  falsificável — não abrir arquivos não relacionados "por curiosidade".
- **Particionamento obrigatório.** Pedido que "parece repo-wide" é convertido em **fatias** (slices):
  por boundary, módulo, família de teste, fluxo. E o agente deve declarar **qual fatia** está fazendo,
  **o que ficou de fora** e **qual é o gate para mudar de fatia**.
- **Regra de bloqueio.** Se não dá para identificar o dono do comportamento num recorte pequeno, o
  agente **declara a ambiguidade e lista 2–3 candidatos** em vez de expandir para o repositório inteiro
  por reflexo.
- **Aplicação proporcional.** Para tarefa trivial (corrigir um typo), aplica-se só o mínimo. A trava é
  proporcional ao risco — não vira burocracia.

**Problema que evita:** a exploração cega que estoura contexto e custo, e a **conclusão arquitetural
baseada em busca rasa** ("achei a palavra X em 40 arquivos, logo o sistema funciona assim" — falso).

---

## B.2 Trava de logs — anti-varredura cega (a pasta que pode congelar a sessão)

A pasta `/logs` pode ter **dezenas de milhares de arquivos**. Listar ou varrer essa pasta cegamente
**trava a sessão inteira**. Por isso há uma trava em dois níveis:

1. **No contrato** (`log-instructions.md`): proibido listar/iterar `/logs` por curiosidade. Só se busca
   **com um `correlation_id` em mãos**, abrindo direto o arquivo candidato. Para reconstruir a árvore
   de jobs (pai → filhos), usa-se o *ledger*, o *sidecar* e o *manifest* — nunca a varredura.
2. **No árbitro automático** (`bash-guard.sh`): um comando que toca `/logs` com `ls/find/grep/cat/tail…`
   **sem** um `correlation_id` é **bloqueado automaticamente**.

> 🛠️ Note a engenharia em camadas: a mesma regra é **pedida** pelo contrato (o agente deveria saber) e
> **garantida** pelo hook (o sistema impede de qualquer jeito). Cinto e suspensório para um risco que
> derruba a sessão.

---

## B.3 Trava de contexto — o isolamento dos agentes e o atalho fino das skills

Esta é a trava mais sutil e talvez a mais importante para o custo.

- **Agentes rodam isolados ("vão e voltam").** Quando uma skill aciona um agente, o trabalho pesado —
  ler 30 arquivos, montar um relatório de centenas de linhas — acontece numa **sala separada**. A
  conversa principal recebe **só o resultado**.
- **O atalho fino mantém a janela principal limpa.** A própria skill diz: *"o agente roda isolado para
  manter o contexto principal limpo e gastar menos tokens"*.

**Por que isso é uma trava de escala?** Porque sem ela, cada investigação grande entupiria a sessão, e
todo o conteúdo lido seria **pago de novo** em cada interação seguinte (o contexto é reenviado a cada
passo). O isolamento é o que permite encadear `investigar → planejar → implementar` num repositório
enorme **sem acumular o peso de todos eles**.

> 🧑‍💼 Em termos de custo: é a diferença entre contratar um especialista que faz a pesquisa na sala
> dele e te entrega um resumo de uma página, versus um que despeja na sua mesa todos os documentos que
> leu — e te cobra para reler a pilha toda a cada conversa.

---

## B.4 Trava de carregamento — o DNA enxuto e as regras sob demanda

O `CLAUDE.md` é **curto e referencial** de propósito: ele afirma o princípio e **aponta** para o
contrato profundo (`.claude/rules/...`), que só é carregado quando a jogada exige.

- A filosofia inteira está **sempre presente** sem pesar.
- O detalhe denso (o contrato de logs tem centenas de linhas) só entra **quando o trabalho toca aquele
  tema**.

É o mesmo princípio de "lazy loading" do bom software, aplicado à própria configuração da IA: **núcleo
estreito sempre carregado, módulos pesados sob demanda.**

---

## B.5 Trava de complexidade — limites de tamanho e reuso obrigatório

Duas regras do `CLAUDE.md §6` que controlam a complexidade *do código* num repositório que cresce:

- **Limites de tamanho de classe:** >500 linhas = atenção, >800 = avaliação explícita antes de novo
  código, >1000 = exceção justificada. Impede a *god class* que ninguém mais entende.
- **Reuso antes de criar** (`reuso-instructions.md`): "em repositório grande, solução nova é rara". O
  agente é obrigado a procurar o que já existe **antes** de criar — e, se acha **duas** soluções para o
  mesmo problema, trata como inconsistência a unificar (proibido criar a terceira).

**Por que conta como trava de escala?** Porque num código grande, a entropia é o inimigo: duplicação e
classes gigantes multiplicam o custo de manutenção exponencialmente. Estas regras seguram a entropia.

---

## B.6 Trava de handoff — passar o bastão sem perder contexto entre fatias

Quando o trabalho é grande demais para uma fatia, ele atravessa agentes e sessões. O
`large-repo-navigation.md` exige que todo **handoff** declare explicitamente: fatia coberta, boundary
validado, arquivos lidos/alterados, validação executada, risco residual, bloqueios e **próximo gate
esperado**.

**Problema que evita:** a "fofoca corporativa" em escala — informação que se distorce a cada repasse
entre fatias, fazendo a fatia seguinte recomeçar no escuro.

---

## B.7 Resumo — as seis travas e o que cada uma protege

| Trava | Mecanismo | Protege contra |
|---|---|---|
| **Navegação por âncora/fatia** | `large-repo-navigation.md` | Exploração cega, conclusão por busca rasa |
| **Anti-varredura de logs** | contrato + `bash-guard` | Congelar a sessão numa pasta gigante |
| **Isolamento de contexto** | agentes "vão e voltam" + atalho fino | Estouro de contexto e custo |
| **Carregamento sob demanda** | DNA enxuto + regras referenciadas | Peso desnecessário em toda sessão |
| **Limites de complexidade** | tamanho de classe + reuso obrigatório | God class, duplicação, entropia |
| **Handoff explícito** | contrato de passagem entre fatias | Perda de contexto entre etapas |

> **Frase para fechar o tema:** "A IA não tem memória infinita, e o código é enorme. Em vez de fingir
> que ela abraça tudo, a gente a obriga a **começar num ponto concreto, ler só o necessário e provar por
> fatia**. Disciplina de escala é o que separa 'funciona no projetinho de exemplo' de 'funciona no
> sistema real'."

---

**Relacionado (base conceitual):** [Os Dois Loops](os-dois-loops.md) ·
[Ciclo de Vida do Software](ciclo-de-vida-software.md) · [Cartão de Referência](cartao-de-referencia.md) ·
[↩ Voltar ao índice da metodologia](../README.md)
</content>
