# O pipeline de ingestão e RAG sobre o acervo DNIT — arquitetura, medição e limites

> **O que é este documento.** A explicação completa, em camadas, de **como** esta plataforma
> transforma 564 manuais técnicos e ~21 mil páginas em conhecimento consultável, **por que** cada
> camada de sofisticação foi acrescentada, **o que foi medido** em cada uma delas, e — a parte que
> quase nenhum documento de engenharia tem a coragem de escrever — **onde estão os limites reais e
> por que eles não são falha de execução**.
>
> **Para quem.** Gerente de desenvolvimento, líder técnico, arquiteto. Público técnico, não
> especialista em RAG. Todo termo é explicado na primeira vez que aparece; não há despejo de código.
>
> **A regra de leitura, e ela vale para o documento inteiro.** Toda afirmação forte vem com o número
> que a prova ou com a decisão de arquitetura que a sustenta. Onde não há medição, está escrito que
> não há. Onde uma capacidade é projeto e não produto, está dito explicitamente — e o capítulo 12
> consolida o **roadmap** num lugar só. Essa disciplina não é modéstia: é o que separa um relatório
> de engenharia de uma apresentação comercial.

---

## Como este documento está organizado

O documento sobe em **camadas de problema**. Cada camada apresenta uma dificuldade que a camada
anterior não tinha como prever, e em seguida a resposta de engenharia que o pipeline dá para ela.

| Parte | O que ela resolve |
|---|---|
| **I — As camadas do problema** | Por que "indexar documentos" fica mais difícil a cada passo: arquivo → PDF → PDF técnico → acervo DNIT real |
| **II — O pipeline de ingestão** | A resposta em camadas: multi-engine, OCR, chunking, dupla indexação, execução assíncrona governada, gates fail-closed, proveniência |
| **III — O pipeline de resposta (RAG)** | Escolher ~20 trechos entre dezenas de milhares, citar com endereço correto e provar depois o que foi usado |
| **IV — A medição** | O harness de avaliação: gabarito validado, juiz LLM, N=3 e a incerteza do próprio instrumento |
| **V — A camada agentic** | DeepAgent sobre o acervo: supervisor, tools governadas, busca iterativa, AG-UI |
| **VI — O fechamento** | Por que 100% do benchmark é estruturalmente difícil, por que o ChatGPT "consegue" e não escala, e para onde as duas abordagens convergem |

**Documentos irmãos — este aqui os referencia e não os repete:**

- [README-PLAYBOOK-PIPELINE-INGESTAO-RAG-ROBUSTO.md](README-PLAYBOOK-PIPELINE-INGESTAO-RAG-ROBUSTO.md)
  — as **lições de engenharia** extraídas da campanha, em forma de 17 regras transferíveis para
  qualquer projeto. Aqui explicamos o que a plataforma **é**; lá está o que a campanha **ensinou**.
- [README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md)
  e [README-TECNICO-RAG-PIPELINE-COMPLETO.md](README-TECNICO-RAG-PIPELINE-COMPLETO.md) — a
  referência técnica de implementação, com o detalhe que este documento deliberadamente não traz.
- [README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md](README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md)
  — o Job Core, motor de execução assíncrona citado na Parte II.

---

## A tese, em uma página

Quatro afirmações sustentam este documento. Cada uma delas é defendida com número no capítulo
indicado.

**1. O trabalho difícil não está na busca. Está em transformar o documento em algo pesquisável.**
A cadeia tem cinco elos, e o que o primeiro perde nenhum dos outros quatro recupera. A prova é
direta: aumentar em **cinco vezes** o número de candidatos da busca comprou **1 alvo de 8**; uma
variante mais sofisticada do prompt do agente **empatou custando 41% a mais**. No mesmo período,
consertar a extração de um único manual levou o conteúdo indexado de **3,4 para 2.201,8 caracteres
por página — 646 vezes mais**. *(Capítulos 1 e 5.)*

**2. A plataforma aplica o estado da arte, e aplica por classe de documento, não por preferência.**
Múltiplas engines de extração escolhidas por classe medida, OCR, chunking estrutural, indexação
dupla (léxica e semântica) com um terceiro vetor de reranking, fusão de resultados, execução
assíncrona com ledger e cancelamento real, gates de qualidade que falham em vez de fingir sucesso,
impressão digital de política de extração para reprocesso cirúrgico, proveniência de página física
**e** impressa por trecho, e isolamento por cliente embutido no nome do recurso físico.
*(Capítulos 5, 6 e 7.)*

**3. A plataforma mede a si mesma — inclusive contra o próprio otimismo.** Existe um *harness* de
avaliação — um conjunto automatizado de perguntas, respostas de referência e correção — com gabarito
validado por especialista, juiz LLM, três execuções por pergunta e medição da
**incerteza do instrumento**: 10 de 18 perguntas dão nota diferente entre execuções idênticas, e a
incerteza do placar é de **±2 pontos**. Foi essa disciplina que permitiu descobrir que a etapa de
reranking — a peça "avançada" que todo projeto de RAG liga — estava **apagando** a ordenação que a
busca já produzia: dispersão de score de **56,38% cai para 1,94%** com ela ligada, e **281 de 281**
buscas terminam empatadas. Desligar o que não ajuda, com medição na mão, é engenharia; manter
ligado porque "é o estado da arte" é moda. *(Capítulos 7 e 8.)*

**4. O que não é respondido não é incompetência do pipeline — são três limites estruturais, e eles
têm nome.** *(Capítulos 10 e 11.)*

- Conteúdo que **não sobrevive à extração** porque no PDF ele é uma imagem, não um texto — a fórmula
  de Rehbock foi lida diretamente no acervo: o texto anuncia *"tem a seguinte expressão:"* e **não
  há expressão nenhuma depois**. Nenhuma busca recupera o que não foi indexado.
- Perguntas que exigem **oito evidências simultâneas** em quatro manuais diferentes, dentro de um
  lote de ~20 trechos. O número que resume a campanha: **71% de acerto quando a pergunta pede uma
  coisa, 20% quando pede sete a nove**.
- **Provar uma ausência.** Quem vê 20 trechos sabe o que veio, nunca o que não veio. É um limite
  lógico, não técnico.

E a afirmação que o placar não mostra: **10 das 17 perguntas do benchmark já foram respondidas com
qualidade adequada por esta plataforma**, numa mesma rodada; as 7 restantes nunca foram, por
nenhuma configuração, em **11 versões medidas**. A distância entre "7 hoje" e "10 provado" é
**estabilidade**; a distância de 10 para 15 é **capacidade** — e a capacidade que falta está
identificada, orçada e com experimento de validação desenhado. *(Capítulos 8 e 12.)*

---

# Parte I — As camadas do problema

## 1. Camada 1 — transformar conhecimento em algo pesquisável

### 1.1 O problema, formulado sem eufemismo

Um engenheiro pergunta: *"qual o recobrimento mínimo para tubo de concreto em rodovia?"*. A resposta
está em algum lugar de 564 manuais e 20.905 páginas — provavelmente em dois ou três deles, em
critérios que competem entre si, com item, quadro e página que precisam ser citados.

A intuição diz que basta "dar os documentos para a IA". Essa intuição quebra em dois pontos, e os
dois são de engenharia, não de inteligência do modelo:

- **Volume.** Nenhum modelo lê 20.905 páginas por pergunta. Mesmo onde a janela de contexto
  coubesse, o custo e o tempo seriam pagos **de novo a cada pergunta** — e o produto aqui não é
  responder uma pergunta, é responder milhares por dia sobre um acervo vivo.
- **Seleção.** Para "ler só o que interessa", alguém precisa saber **qual** manual abrir. Essa
  decisão já é um índice. Ou o sistema tem um, ou o usuário faz o trabalho do índice à mão.

Daí a arquitetura: **pagar o trabalho pesado uma vez, na ingestão, e transformá-lo em estrutura
consultável**. Isso é o que se chama de RAG — *Retrieval-Augmented Generation*, "geração aumentada
por recuperação": em vez de o modelo responder de memória, ele recebe trechos recuperados de uma
base e responde **com base neles**.

### 1.2 A cadeia de cinco elos

Um pipeline de ingestão + RAG é uma cadeia. Documentos entram numa ponta, respostas saem na outra, e
**cada elo pode perder alguma coisa**.

1. **Extração** — abrir o arquivo e virar texto.
2. **Recorte (*chunking*)** — partir o texto em pedaços de mais ou menos meia página.
3. **Indexação** — guardar cada pedaço de dois jeitos: por **palavra** (achar "Bueiro Tubular"
   literalmente) e por **significado** (achar "bueiro" quando alguém escreveu "galeria").
4. **Busca (*retrieval*)** — dada a pergunta, escolher os ~20 melhores pedaços entre dezenas de
   milhares.
5. **Resposta** — o modelo lê esses ~20 pedaços e escreve. Nesta plataforma, ele pode buscar de novo
   se achar que falta coisa.

### 1.3 A regra-mãe, e por que ela organiza o documento inteiro

> **O que o Elo 1 perde, os Elos 2 a 5 não recuperam. Não se busca o que não foi indexado.**

Escrita assim parece óbvia. Na prática, quase todo o esforço de melhoria de RAG no mercado vai para
os elos 4 e 5 — "buscar melhor", "reordenar melhor", "prompt melhor". A campanha mediu quanto isso
rende quando o problema nasceu no elo 1:

- Subir os candidatos da busca de 100 para 500 — **cinco vezes mais** — em 8 consultas críticas
  comprou **1 alvo de 8**.
- Uma variante mais sofisticada do prompt do agente **empatou, custando 41% a mais**.
- Consertar a extração de um único manual levou o conteúdo indexado de **3,4 para 2.201,8 caracteres
  por página**.

Duas ordens de grandeza separam o elo 1 dos elos 4 e 5. **É por isso que a maior parte deste
documento fala de ingestão** — e é por isso que a Parte I sobe camada por camada dentro do elo 1
antes de chegar à busca.

### 1.4 O que esta camada compra, e o que ela cobra

Pagar o custo uma vez compra três coisas que o modelo "ler na hora" não tem: **escala** (responder
sobre 564 manuais custa o mesmo que sobre 5), **governança** (cada trecho tem endereço, e dá para
dizer quem enxerga o quê) e **auditoria** (dá para provar depois o que foi usado).

E cobra uma: **o que se perdeu na leitura antecipada está perdido para sempre**, e — este é o ponto
grave — **some em silêncio**. O sistema não sabe que perdeu.

> Toda a Parte II existe para reduzir essa cobrança ao mínimo e, onde não der para eliminá-la,
> **torná-la barulhenta em vez de silenciosa**.

---

## 2. Camada 2 — PDF é um desenho de página, não um texto

### 2.1 O problema

Um arquivo `.txt` guarda texto. Um PDF guarda **instruções de desenho**: coloque este glifo nesta
coordenada, com esta fonte, neste tamanho. Não existe no arquivo o conceito de parágrafo, de ordem
de leitura, de coluna ou de tabela. Tudo isso é **percepção humana sobre um layout** — e é o
extrator que precisa reconstruir.

Quatro dificuldades concretas nascem daí:

- **Ordem de leitura.** Num documento de duas colunas, a ordem física dos glifos pode intercalar as
  colunas. O texto extraído fica embaralhado, e o embaralhamento não gera erro nenhum.
- **Tabela.** Uma tabela é um conjunto de textos posicionados e, às vezes, linhas desenhadas.
  Reconstituir "esta célula pertence a esta linha e a esta coluna" é inferência sobre geometria. Um
  erro de poucos pixels cola duas células numa só.
- **Imagem.** Fórmulas, ábacos, desenhos e boa parte dos diagramas normativos são **imagens**. Não
  há texto para extrair. O extrator não falha: ele simplesmente não encontra nada e segue.
- **Página escaneada.** Manuais antigos não têm camada de texto nenhuma — são fotografias de papel.
  Só há pixels, e a única saída é **OCR** (*Optical Character Recognition*): um programa que "lê" a
  imagem e chuta as letras. OCR erra, e erra em silêncio.

### 2.2 Como isso aparece no acervo real — dois exemplos lidos diretamente

Não é teoria. Estes dois trechos foram lidos no índice de produção, não deduzidos:

O Quadro 5.7.2.1 do manual de projeto geométrico, que dá a largura dos acostamentos, está indexado
assim:

```
|Classedorojeto||Relevo||
|ClasseO|3,50|3,00*|3,00*|
|ClasseIII|2,50|2,00|1,50|
```

Duas coisas aconteceram na mesma tabela. **Colagem de célula:** `ClasseIII` perdeu os espaços, e
quem procura "Classe III" não casa com ele — nem por busca exata, nem por busca lexical.
**Corrupção de caractere:** `Classedorojeto` é "Classe do Projeto" com letras perdidas, e num outro
quadro do mesmo manual o `III` virou `H` (`ClasseHI`).

O extrator não reportou nenhum erro. Do ponto de vista do pipeline, aquele documento foi ingerido
com sucesso.

### 2.3 A resposta desta camada: a pergunta certa não é "qual é a melhor engine"

*Engine* de extração é o programa que abre o PDF e devolve texto. Existem várias, boas em coisas
diferentes, e a tentação é escolher "a melhor". A campanha mediu que essa é a pergunta errada.

A pergunta certa é: **qual engine minimiza o custo total = primeira passada + reprocessos?** A
métrica que responde isso é a **taxa de escalada** — quantos documentos aquela engine erra a ponto
de exigir uma segunda passada por outra. Medida no acervo, contando documentos que saíram abaixo de
1.000 caracteres por página (o piso abaixo do qual a extração está claramente incompleta):

| Engine no acervo | Documentos | Caracteres por página | Documentos abaixo de 1.000 | Taxa de escalada |
|---|---:|---:|---:|---:|
| `pymupdf4llm` (com e sem OCR) | 508 | **3.860 / 3.856** | **6** | **~1,2%** |
| `docling` (com e sem OCR) | 56 | 1.698 / 747 | **12** | **~21%** |

O detalhe que decide o desenho: **a melhor extração de tabela, `docling` em modo preciso, custa
11,70 segundos por página; o `opendataloader` entrega a mesma célula íntegra a 0,09 segundos por
página — 130 vezes mais barato**, e ainda **preserva a numeração impressa que o `docling`
descarta**. Para a classe inteira de documentos com tabela degradada, a diferença é **41 minutos
contra 17,5 horas**.

E o inverso também é verdade, medido no mesmo dia: no manual escaneado de 1981, o `opendataloader`
devolveu **18 caracteres por página com código de saída zero**. Sucesso mentiroso.

> **A decisão desta camada, e ela é a espinha da Parte II: multi-engine por classe de documento,
> agregando, nunca substituindo.** Cada engine entra na classe em que ela ganha, e a escolha é
> medida por classe — não escolhida por preferência nem por *benchmark* de folheto.

**E há uma segunda decisão, menos óbvia e mais importante:** a engine **padrão** tem que ser aquela
cuja falha é **contável**, mesmo que ela não seja a melhor em nenhuma categoria isolada. Os 18
caracteres por página com código de saída zero do parágrafo anterior não são um erro que dá para
tratar: são um sucesso mentiroso, e nenhum gate de qualidade consegue julgar o que a engine não
confessa.

O mesmo vale para "modos automáticos". Um modo de triagem que deveria detectar página escaneada e
acionar o processamento pesado foi medido num documento em que **100% das páginas precisam de OCR**:
ele decidiu que **nenhuma** era complexa, não chamou o processamento uma única vez, e devolveu
código zero. **O defeito estava na triagem, não no processamento** — e um pipeline que confia em
triagem automática não tem como saber disso.

### 2.4 O que ainda falta

Resolvida a leitura genérica de PDF, o documento **técnico** acrescenta quatro problemas que um PDF
comum não tem — e é neles que o acervo DNIT vive.

---

## 3. Camada 3 — PDF técnico: fórmula, tabela normativa e o endereço da resposta

### 3.1 A fórmula que é uma imagem, e a perda silenciosa

Numa norma de engenharia a fórmula **é** a resposta. E, em boa parte dos manuais, ela está gravada
como imagem dentro do PDF.

Lido diretamente no acervo, no manual de drenagem, exatamente na página que o gabarito cobra:

> *"A fórmula de Rehbock, aplicável principalmente para o escoamento em regime subcrítico, tem a
> seguinte expressão:"* — **e não há expressão nenhuma depois.**

O mesmo padrão se repete três vezes na mesma página com as equações das aproximações sucessivas:
*"calcula-se o valor de y1:"* seguido de nada.

O que torna isso grave não é a perda — é **o silêncio**. Uma varredura completa contando os
marcadores que o próprio extrator deixa mostrou:

| Documento | Chunks | `formula-not-decoded` | Equação anunciada e vazia | Sinais de OCR corrompido |
|---|---:|---:|---:|---:|
| Manual de drenagem (IPR-724) | 1.411 | **0** | **3** | 0 |
| Manual de hidrologia (IPR-715) | 358 | **0** | **3** | 0 |
| Método de pavimentos flexíveis (IPR-667) | 62 | **8** | **4** | **22** |

Leia a primeira linha com atenção: **o contador de "fórmula não decodificada" é zero, e a fórmula
não está lá.** O extrator não marcou nada. Quem lê o trecho — modelo ou humano — não tem como saber
que faltou alguma coisa.

> **A perda é majoritariamente silenciosa.** Essa é a frase mais cara da campanha, e a razão pela
> qual "o pipeline não respondeu" e "o pipeline não tinha o que responder" são diagnósticos
> completamente diferentes.

### 3.2 A tabela normativa densa

Numa norma, a tabela não ilustra: ela **é** o critério. E ela costuma ser grande, com cabeçalho de
dois níveis, notas de rodapé e células que dependem de alinhamento visual para significar alguma
coisa.

Duas medições do acervo mostram o tamanho do problema:

- O "Quadro 2" da IS-203 — a tabela que fixa o tempo de recorrência por tipo de bueiro — tem **zero
  ocorrência** nos 1.442 trechos indexados do manual que a publica. Não é ranking ruim: a tabela
  **não foi extraída**.
- Na coleção inteira: **641 de 28.263 tabelas degradadas, em 145 de 438 documentos com tabela**. O
  pior caso é o manual de projeto geométrico, com **16 tabelas degradadas de 49**.

### 3.3 A numeração dupla — o problema que ninguém prevê e que custou mais nota que qualquer bug

Todo manual técnico tem **duas** numerações de página: a do **arquivo PDF**, que conta desde a capa,
e a **impressa no rodapé**, que é a que o engenheiro procura. Elas divergem porque capa, ficha
catalográfica e sumário criam uma defasagem — tipicamente de 1 a 4 páginas, e num caso medido, 24.

Medido documento a documento, o deslocamento é constante e altíssimamente consistente:

| Documento | Pares medidos | Deslocamento (física − impressa) | Consistência |
|---|---:|---:|---|
| Manual de drenagem (IPR-724) | 324 | **+4** | 322/324 (99,4%) |
| Manual de hidrologia (IPR-715) | 122 | **+4** | 122/122 (100%) |
| IPR-726 (3ª edição, 2006) | 465 | **+3** | 465/465 (100%) |
| Manual de pavimentação (IPR-719) | 20 | **+3** | 20/20 (100%) |
| Manual de projeto geométrico (IPR-706) | 3 | — | **inconclusivo** (quase não tem rodapé legível) |

O efeito prático era exatamente este: a resposta trazia **o valor certo, a norma certa e o critério
certo — e errava o endereço**. Na prova mais limpa, o gabarito pedia p. 256 e a resposta citou
p. 259, que é 256 + 3.

> **A maior fonte de perda de nota da campanha não foi erro técnico. Foi convenção de numeração.**
> **31,2% das execuções** saíam com referência inadequada.

### 3.4 A hierarquia e as referências cruzadas

Um manual técnico não é texto corrido: é uma árvore (`8.5.6.2`), com quadros e tabelas numerados
independentemente, e com normas que **se citam entre si** — uma instrução de serviço publicada
dentro de um manual, um manual que remete a um desenho-padrão de outro. Uma resposta correta precisa
citar documento, item, quadro **e** página; e uma pergunta correta frequentemente exige juntar dois
ou três documentos que se referenciam.

Isso impõe duas exigências ao pipeline que um RAG genérico não tem: **recortar respeitando a
estrutura** (não cortar no meio de uma tabela nem colar o fim de uma seção no começo da próxima) e
**carregar identidade normativa em cada trecho** — que norma é essa, de que ano, de que tipo.

### 3.5 A resposta desta camada

Para cada um dos quatro problemas acima, o pipeline acrescenta uma peça específica. Todas estão
detalhadas na Parte II; aqui fica o mapa:

- **Fórmula-imagem** → enriquecimento de fórmula na extração, que converte a imagem em notação
  matemática textual (LaTeX), ligável **por documento** — porque ligar para o acervo inteiro é
  proibitivo (§5.2).
- **Tabela normativa** → modo de extração de tabela preciso, que separa linhas coladas, e uma engine
  alternativa duas ordens de grandeza mais barata para a mesma classe (§5.1).
- **Numeração dupla** → proveniência de página **física e impressa** gravada em cada trecho, e
  citação que apresenta as duas (§6.4 e §7.5).
- **Hierarquia e referência cruzada** → chunking estrutural e metadados normativos por trecho —
  código do documento, ano de publicação, tipo e subtipo, disciplina, órgão emissor —, presentes hoje
  em **535 de 541 documentos amostrados (98,9%)** (§6.4).

### 3.6 O que ainda falta

Nada disso ainda encostou no acervo **real**. Um PDF técnico bem produzido já é difícil; um acervo
com quatro décadas de normas sobrepostas — de 1981 a 2026 — é outra categoria de problema.

---

## 4. Camada 4 — o acervo DNIT real

### 4.1 O censo: o que está de fato lá dentro

Tudo abaixo foi medido na fonte física — a coleção no banco vetorial e o registro de ingestão —,
nunca por documentação ou log.

- **564 documentos ativos, 20.905 páginas.**
- **65.324 trechos indexados**, com integridade provada ponto a ponto entre o registro de ingestão e
  o armazenamento físico.
- Composição por tipo, em amostra que alcançou 541 documentos distintos: **429 normas, 67 manuais,
  30 referências técnicas, 9 atos normativos**.

Não é uma biblioteca homogênea. É um acervo normativo acumulado.

### 4.2 Idade e origem: os manuais antigos são escaneados

O documento que responde uma das perguntas do benchmark é de **1981**. Ele está no acervo como
imagem digitalizada, e o resultado da extração fala por si: **62 trechos**, concentrando **8 fórmulas
explicitamente não decodificadas**, 4 equações anunciadas e vazias e **22 sinais de OCR corrompido**
— `I.G.` virando `1.6.`, `I.S.` virando `1.5.` ou `|.S.`.

Isso degrada duas coisas ao mesmo tempo: a leitura do modelo **e** o casamento lexical da busca por
palavra. Quem procura "I.S." não acha "1.5.".

E o censo mostra que isso não é exceção. Classificando os 564 documentos pela decisão de extração
gravada na própria ingestão:

| Classe | Documentos | Páginas |
|---|---:|---:|
| Texto nativo suficiente | 462 | 8.542 |
| **Dependente de OCR** | **102 (18,1%)** | **12.363 (59,1%)** |

> **18,1% dos documentos dependem de OCR — e eles são 59,1% das páginas.**

A leitura correta desse par de números é o que decide a arquitetura: em número de arquivos o OCR é
minoria; em volume de conteúdo, é **a maior parte do acervo**. Uma política de extração que trate
OCR como caso de borda erra na maioria das páginas.

### 4.3 Qualidade heterogênea: o mesmo pipeline, documentos incomparáveis

O manual de projeto geométrico é, ao mesmo tempo, **o pior documento do acervo em célula colada**
(16 tabelas degradadas de 49) **e** um dos documentos que o mapa de página impressa **recusa** — ele
quase não tem rodapé legível, e a derivação do deslocamento fica inconclusiva com apenas 3 pares
dispersos.

Ou seja: o mesmo documento pede tratamento especial em duas dimensões independentes, e uma delas não
tem solução disponível. **Uma política única para o acervo inteiro é matematicamente incapaz de
servir bem a ele.**

### 4.4 Vigência: normas que se sucedem no tempo

Este é o problema que separa um acervo normativo de uma base de conhecimento qualquer. Um manual de
1981 e uma instrução de serviço de 2021 podem tratar do mesmo assunto, e a resposta correta **não é
necessariamente a mais nova** — depende de haver ou não revogação expressa.

Um caso real do benchmark: uma pergunta exige apresentar **os dois** critérios concorrentes — o da
instrução de serviço antiga e o da instrução normativa de 2026 — **sem hierarquizar entre eles**,
justamente porque não há revogação expressa. A resposta que "resolve" escolhendo um dos dois erra.

A boa notícia, medida: **os metadados de vigência já existem no acervo**. Cada trecho carrega código
do documento, ano de publicação, tipo, subtipo, disciplina e órgão emissor, vindos do registro
paralelo de catalogação — presentes em **535 de 541 documentos amostrados (98,9%)**, e conferidos um
a um em **13 de 13** documentos-chave do gabarito.

### 4.5 Duplicatas: o acervo servindo duas verdades ao mesmo tempo

Quando um documento é reprocessado, a geração anterior precisa sair do índice. Um defeito fazia essa
remoção **não casar com nada** — e ainda registrar "limpeza concluída". Medido no acervo inteiro:

- **8.703 trechos mortos em 77.441 (11,24%)**, distribuídos em **57 documentos** — medido no acervo
  **antes** da janela de reprocesso, que é onde esse número existe (§6.7).
- E a reviravolta: **em 33 desses 57, a geração "viva" era MENOR que a "morta"**. Num documento, 11
  trechos vivos contra 338 mortos.

Duas lições saíram daí ao mesmo tempo: o defeito era real, **e limpar às cegas teria apagado a
versão boa**. A correção foi feita na origem, e a prova veio depois — **84 de 84 documentos
reprocessados, 564 linhas no acervo, sem duplicar nada**.

### 4.6 O que este acervo exige do pipeline

Somando as quatro camadas, o acervo DNIT impõe seis requisitos que nenhum pipeline genérico atende
por acaso:

1. **Política de extração por classe de documento**, porque nenhuma engine serve a todos.
2. **OCR tratado como caminho principal e não como caso de borda**, porque 59,1% das páginas
   dependem dele.
3. **Recuperação de fórmula e de tabela** como capacidades separadas e acionáveis por documento.
4. **Proveniência de página física e impressa**, porque a resposta certa no endereço errado perde.
5. **Identidade normativa e vigência por trecho**, porque a pergunta é sobre qual norma vale.
6. **Substituição de geração confiável e auditável**, porque um acervo vivo é reprocessado.

A Parte II mostra como cada um deles foi respondido, o que custou e o que ficou de fora.

---

# Parte II — O pipeline de ingestão da plataforma

> A Parte I terminou com seis requisitos. Esta parte mostra a peça que responde a cada um, o que ela
> custa e o que ela **não** faz. A ordem aqui é a ordem real de execução.

## 5. Da página ao índice: extração, OCR, recorte e indexação

### 5.1 Multi-engine por classe — a decisão estrutural

O pipeline não tem "a engine". Ele tem um **catálogo de engines** e uma **política de seleção por
documento**, declarada em configuração e resolvida em tempo de ingestão. As engines efetivamente
executáveis neste ambiente são `pymupdf4llm`, `docling` e `opendataloader`, todas por trás do
**mesmo contrato de extração** — trocar de engine não muda uma linha do resto do pipeline. É a
arquitetura hexagonal aplicada onde ela paga: a decisão de qual programa lê o PDF virou uma chave de
configuração, e não uma reescrita.

A seleção é declarativa e **por documento nomeado**, não global:

```yaml
processing:
  parsing:
    engine_overrides:
      - engine: docling
        config:
          ocr: true                          # liga o reconhecimento de imagem
          tables: true                       # liga a extração de tabela
          structured_chunking: true          # recorta respeitando a estrutura do documento
          skip_scanned_pdf: false            # NÃO pule PDF escaneado — é justamente o caso
          document_timeout_seconds: 7200.0   # teto por documento, folgado de propósito
          subprocess_timeout_seconds: 14401.0# teto do processo externo: SEMPRE maior que o do documento
          table_mode: accurate               # modo preciso de tabela: separa linhas coladas
        documents:
          - "manual_com_tabela_critica.pdf"
```

O que está atrás dessa configuração é o resultado de uma bancada que avaliou **dez configurações de
extração** — as engines locais e suas variações de OCR, enriquecimento de fórmula e modo de tabela,
mais alternativas externas cujos preços foram levantados e que **não foram medidas** — sobre fatias
reais do acervo, com os alvos exatos das perguntas que falhavam. A conclusão foi uma **matriz de
classes**, não um vencedor:

| Classe de documento | Como é identificada | Engine que ganha | Por quê, com número |
|---|---|---|---|
| **1. Com fórmula** | fórmula anunciada e vazia, ou marcador de "não decodificada" | `docling` + enriquecimento de fórmula | única configuração que recuperou os alvos medidos |
| **2. Tabela com célula colada** | célula com palavra sem espaço; termo-chave ausente do trecho | `opendataloader` | mesma célula íntegra a **0,09 s/página** contra **11,70 s/página**, e preserva a página impressa |
| **3. Escaneado puro** (0 caractere nativo) | contagem de caractere nativo por página = 0 | **sem vencedor único** — ver abaixo | trade-off medido, declarado como tal |
| **4. Escaneado com camada de texto** | 200 a 1.500 caracteres nativos por página | `pymupdf4llm` | 3.860 caracteres/página no acervo, **zero colapsos** |
| **5. Todo o resto** (~460 documentos) | não se enquadra acima | `pymupdf4llm`, como está | 3.856 caracteres/página, sem dívida provada |

A classe 3 merece atenção porque é onde a honestidade da medição aparece. Rodando o **mesmo manual
escaneado inteiro** nas duas engines:

| Critério | `docling` (teto folgado) | `pymupdf4llm` |
|---|---:|---:|
| Linhas de tabela recuperadas | **304** | 165 |
| Fórmulas auditadas (marcadas como não decodificadas) | **52** | 0 — perda **silenciosa** |
| Marcador de página física | 0 | **34** |
| Títulos / hierarquia | 0 | **28** |
| Siglas técnicas corretas / erradas | 83 / 2 | **89 / 0** |
| Segundos por página | 10,06 | **6,05** |

Não existe resposta única: `docling` ganha tabela e fórmula, `pymupdf4llm` ganha proveniência,
hierarquia, fidelidade de sigla e velocidade. **Declarar o empate é mais útil do que fabricar um
vencedor** — e é por isso que a política é por classe e por documento.

> **A regra de ouro do multi-engine: agregar, nunca substituir.** E ela tem preço declarado: migrar
> um documento para `docling` **apaga a numeração de página impressa** — medido em **4 de 4
> documentos**. Ganhar tabela custou perder proveniência, e essa troca precisa ser decidida
> documento a documento, não por padrão.

### 5.2 As duas capacidades avançadas de extração, e o que elas custam

Ambas existem no produto, ligáveis **por documento** dentro da configuração da engine.

**Enriquecimento de fórmula** — converte a fórmula que é imagem em notação matemática textual
(LaTeX), que passa a ser texto indexável e pesquisável. É a única técnica medida que recuperou o
conteúdo perdido:

| Documento / fatia | Marcadores de fórmula não decodificada | Depois | Resultado |
|---|---:|---:|---|
| Manual de drenagem, pp. 149-154 | **10** | **0** | fórmula de Rehbock recuperada em LaTeX |
| Manual de hidrologia, pp. 125-135 | **8** | **0** | 16 blocos LaTeX; **as duas** fórmulas-alvo recuperadas |
| Método de pavimentos flexíveis, pp. 1-10 | — | — | 18 blocos LaTeX onde antes havia zero |

**A ressalva, e ela é obrigatória:** recuperar o alvo no índice é condição **necessária, não
suficiente**. A transcrição volta com defeito de OCR — na fórmula de Rehbock **o sinal de igualdade
não saiu**, e num outro documento o OCR escreveu `1.5.` onde estava `I.S.`. A estrutura da fração
está correta; a sigla, não. Uma correção posterior tratou a restauração de sigla dentro do bloco
LaTeX; o sinal de igualdade segue registrado como defeito aberto.

**O custo é o motivo de a chave ser por documento:** o enriquecimento mede **85,9 a 134,1 segundos
por página**. Para o acervo inteiro seriam cerca de **20 dias de CPU** — proibitivo. Para os **4
documentos que carregam a dívida de fórmula, 995 páginas = 4,8% do acervo**, é da ordem de **24 a
75 horas sequenciais, ou 8 a 25 horas com paralelismo 3**, sem custo de API porque a extração roda
local. **Uma a três noites de janela, não vinte dias.** É a diferença entre uma capacidade
proibitiva e uma capacidade dirigida.

**Modo preciso de tabela** — separa linhas coladas na reconstrução da grade. Medido na fatia que
contém o quadro normativo perdido:

| Configuração | s/página | Linhas de grade | O alvo saiu? |
|---|---:|---:|---|
| `pymupdf4llm` | 3,49 | 46 | parcial — termo-chave destruído |
| `docling` modo rápido | 14,58 | 49 | sim, com 2 linhas fundidas |
| `docling` modo preciso | **11,70** | 42 | **sim, íntegro** |
| `opendataloader` | **0,09** | 39 | **sim, íntegro** |

O efeito no índice é rastreável e é a prova mais bonita desta seção: depois do modo preciso, o
trecho indexado passou a trazer literalmente `| Bueiros Tubulares | 15 (como canal) |`, e a palavra
"Bueiro" — que **não existia no índice** desse manual — passou a casar **114 trechos**.

### 5.3 OCR: a maioria do acervo passa por aqui

Como 59,1% das páginas dependem de OCR, ele não é um caso de borda: é o caminho principal. O
pipeline trata OCR como um **estágio próprio**, anterior ao parse, com dois modos:

- **OCR embutido na engine de extração**, resolvido automaticamente pela biblioteca (RapidOCR é o
  motor efetivamente usado nesta instalação; Tesseract está disponível pela linha de comando).
- **Pré-OCR do documento inteiro** (`ocrmypdf`), que rasteriza a página e recria a camada de texto,
  com `oversample: 300` — a resolução de amostragem que torna a página legível ao custo de torná-la
  cara.

E aqui está uma das medições mais úteis da bancada, porque contraria a intuição: **pré-OCR forçado
não é melhoria automática**. No manual escaneado, forçar a re-OCR-ização degradou a fidelidade de
sigla (de 89 corretas / 0 erradas para 61 / 6) e colou palavras do documento inteiro
(`Drenagemsuperficial`, `Períododerecorrência(anos)`). No manual de drenagem, produziu exatamente a
colagem que se queria evitar.

> **Conclusão registrada: OCR mais agressivo troca um defeito por outro.** A decisão certa é por
> classe de documento, com medição, e não "ligar tudo".

### 5.4 O teto de tempo — a variável que sozinha valia 21 vezes mais conteúdo

Antes de qualquer sofisticação de engine, havia um número trivial governando o resultado: o teto de
tempo por documento. Mudando **uma única variável** num manual escaneado:

| | teto de 120 s | teto de 7.200 s | ganho |
|---|---:|---:|---:|
| Caracteres extraídos | 2.093 | **44.683** | **21,3×** |
| Caracteres por página | 62 | **1.314** | 21× |
| Tabelas detectadas | 1 | **24** | 24× |
| Fórmulas identificadas | 0 | **52** | — |
| Tempo de parede | 167,8 s | 342,1 s | 2,0× |

Dobrar o tempo comprou 21 vezes mais conteúdo. **E teto folgado não custa nada:** um manual que
gastou 5.597 segundos de um teto de 43.200 terminou antes e não desperdiçou nada. Já um teto curto
custou 5 horas de processamento reprovadas no fim.

### 5.5 Recorte (*chunking*)

O texto extraído é partido em trechos de mais ou menos meia página. Duas decisões importam aqui:

- **Recorte estrutural** (`structured_chunking`) — respeitar a estrutura do documento em vez de
  cortar a cada N caracteres. Se o corte cai no meio de uma tabela, a tabela quebra em dois trechos
  e nenhum deles responde sozinho.
- **Marcador de página injetado no texto** — o pipeline insere um separador declarado em
  configuração (`\n\n--- Página {} ---\n\n`) antes de recortar. Esse marcador aparentemente
  burocrático é o que torna possível, mais tarde e **sem reprocessar nada**, derivar a numeração
  impressa (§6.4).

O resultado é um acervo com dois tipos de trecho, e a proporção mostra o peso das tabelas neste
domínio: numa varredura de 42.496 trechos, **27.030 de texto e 15.465 de tabela**.

### 5.6 Indexação: três representações do mesmo trecho

Cada trecho é indexado de **três formas simultâneas** na mesma coleção vetorial. Esta é a peça que
sustenta a busca híbrida da Parte III, e vale entender o que cada uma faz:

| Representação | O que é | Para que serve |
|---|---|---|
| **Vetor denso** — 1.536 dimensões, distância cosseno | o *significado* do trecho, transformado em números | achar "bueiro" quando a pergunta disse "galeria" |
| **Vetor esparso** — com modificador `idf` | a *estatística de palavras* do trecho, no estilo BM25 | achar "Bueiro Tubular" literalmente, e dar peso a termos raros |
| **Vetor de interação tardia** — 96 dimensões, multivetor com `max_sim`, quantização binária, em disco | uma representação por *token*, comparada termo a termo | reordenar os candidatos com precisão maior (ColBERT / MaxSim) |

Um detalhe de rigor que vale citar porque ele quase invalidou uma medição inteira: **o modelo de
vetorização declarado na configuração não era o realmente usado**. Quem confiou no arquivo mediu
similaridade `−0,02` contra o vetor armazenado; quem descobriu o modelo real mediu **0,999935**.
A configuração declara a intenção; **o runtime diz o fato** — e este pipeline foi auditado contra o
runtime.

### 5.7 O que está ligado hoje, e o que é decisão tomada mas ainda não aplicada

Esta distinção é obrigatória, e ela é a diferença entre um documento técnico e um folheto.

- **Ligado e em produção:** seleção de engine por documento, `pymupdf4llm` como padrão com OCR,
  `docling` por override, tetos de tempo corrigidos, marcador de página, recorte estrutural,
  indexação nas três representações.
- **Existe no produto, ligado em poucos documentos:** o modo preciso de tabela. Do acervo
  reprocessado na última janela, **83 documentos, 82 deles sem o modo preciso** — só um o tem. A
  entrada de configuração que cobre 76 documentos não declara a chave.
- **Existe no produto, medido em bancada, ainda não aplicado ao acervo:** o enriquecimento de
  fórmula nos 4 documentos que carregam a dívida. Está orçado (§5.2) e depende de janela.
- **Engine registrada no catálogo, migração da classe ainda não executada:** o `opendataloader` para
  a classe de tabela colada. A decisão de arquitetura está tomada e registrada — "terceira engine só
  para tabela, como trabalho separado" —, a execução é trabalho planejado.

> **Nada acima é apresentado como pronto por estar planejado.** O capítulo 12 consolida o que é
> roadmap, com o critério que decide cada item.

### 5.8 O que ainda falta nesta parte

Extrair, recortar e indexar bem resolve o conteúdo. Não resolve **executar isso sobre 20.905 páginas
sem que a operação vire uma caixa-preta** — nem garante que um resultado ruim seja detectado em vez
de publicado. É o assunto do capítulo 6.

---

## 6. A máquina por baixo: executar 20.905 páginas sem virar caixa-preta

### 6.1 Job Core — execução assíncrona com registro contábil

Ingerir o acervo é uma operação de **dezenas de horas**, distribuída em centenas de documentos, que
precisa sobreviver a reinício de servidor, ser cancelável no meio e ser auditável depois. Isso não é
"rodar um script".

A plataforma tem para isso um componente próprio e **independente de domínio**, o **Job Core**:
qualquer trabalho assíncrono — ingestão, ETL, tarefas agendadas, processamento em segundo plano —
entra por um envelope opaco e um manipulador registrado. A ingestão é **usuária** do Job Core, não
dona dele.

O que isso entrega, e por que cada peça importa aqui:

- **A fila é o próprio ledger.** Não há um serviço de fila separado no caminho: o estado durável
  vive em PostgreSQL (`job_core.job_runs` e `job_run_events`) e os workers disputam trabalho por
  **reivindicação atômica no banco**, com bloqueio que pula registros já tomados. Isso elimina a
  classe inteira de problemas de "a fila e o banco discordam sobre o que aconteceu" — há uma fonte
  de verdade só. Se o servidor cai, o trabalho continua existindo, e o histórico também.
- **Estado explícito e finito.** Uma execução transita por **13 estados**, dos quais **5 são
  terminais**. Não existe "acho que terminou": a transição é gravada.
- **Paralelismo governado em camadas** — vagas locais do worker, admissão global no ledger e um teto
  de execuções filhas por pai. Para ingestão, o número de documentos simultâneos é declarado por
  requisição ou por configuração e limitado pelo servidor. A janela de reprocesso do acervo é
  planejada com **paralelismo 3**: é o que transforma "39 horas sequenciais" em "13 horas de
  janela".
- **Cancelamento que cancela de verdade.** Esta é menos óbvia do que parece, e é onde a maioria das
  implementações para. Aqui o cancelamento tem três alcances: (a) o processo em execução observa o
  pedido em pontos de checagem; (b) o subprocesso pesado de extração recebe escalada de sinal, com
  **encerramento do grupo de processos inteiro** — não só do processo filho, mas dos "netos" que as
  bibliotecas de OCR e visão criam; (c) um portão **antes de nascer** impede que um subprocesso seja
  criado depois de o cancelamento já ter sido pedido. Os dois primeiros existem porque a campanha
  mediu o que acontece sem eles: **3 processos órfãos segurando 14,7 GB de memória e metade da vazão
  da janela**, e subprocessos nascendo dez minutos **depois** do cancelamento registrado.
- **Detecção de execução morta** por sinal de vida periódico (a cada **30 segundos**), com corte de
  obsolescência em **180 segundos** e um reconciliador cujo **único desfecho possível é cancelar** —
  ele nunca ressuscita, nunca reenfileira, nunca reexecuta. Reiniciar o servidor não pode fazer
  voltar à vida uma tarefa de uma rodada já encerrada, com a configuração errada. E ele nasce
  **vencido no boot**, de propósito: a varredura acontece antes da primeira reivindicação de
  trabalho.
- **Acompanhamento por estados, eventos e logs correlacionados**, sem retorno de progresso por
  chamada de função. Progresso empurrado por callback não sobrevive a processo que morre; evento
  gravado no ledger, sim. São **12 eventos duráveis** no ledger, com ordem de precedência causal
  declarada para desempatar registros do mesmo instante.
- **O identificador de correlação é herdado, nunca derivado.** Um job filho recebe exatamente a
  correlação do pai, e o próprio armazenamento recusa a gravação se ela não for preservada. É isso
  que faz uma janela de ingestão inteira — API, worker, subprocessos — caber sob um identificador só.

**E a fronteira é testada, não confiada.** Existe um teste de arquitetura que reprova a build se
código de domínio chamar operações de ciclo de vida de job, implementar transporte próprio de
filhos, ou usar controle de execução por conta própria. Uma varredura no código de ingestão inteiro
não encontra **nenhuma** chamada de ciclo de vida do Job Core — o domínio *pede* capacidade especial
de execução; ele não a implementa.

**O que o Job Core deliberadamente não faz**, e isso está declarado: não há retentativa automática
de job, reenfileiramento, fila de mensagens mortas nem reabertura de estado terminal. Um job que
falha, falhou — e a decisão de rodar de novo é de quem opera, com registro.

A referência técnica completa está em
[README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md](README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md).

### 6.2 Gates fail-closed: resultado cortado nunca é sucesso

*Fail-closed* significa "na dúvida, falhe e avise". O contrário — *fail-open*, "na dúvida, siga em
frente" — é o comportamento que produziu o pior defeito da campanha, e vale contar porque **o
compromisso de engenharia deste pipeline não é nunca errar; é ser construído para tornar o
erro visível**.

O que acontecia: a biblioteca de extração tinha um teto de tempo por documento. Documento grande
estourava o teto, a extração parava no meio, a biblioteca devolvia um status de **sucesso parcial**
— e o código aceitava isso como válido e **publicava**.

O tamanho do estrago, medido documento a documento na fonte de verdade:

| Status devolvido pela extração | Documentos | Páginas | Páginas com erro de tempo |
|---|---:|---:|---:|
| Sucesso pleno | 32 | 315 | 0 |
| **Sucesso parcial — publicado assim mesmo** | **24** | **2.127** | **1.888** |

O pior caso: um manual de 568 páginas com **3,4 caracteres de texto por página**, publicado como
íntegro. E o sistema ainda reportava `páginas processadas: 34`, `páginas vazias: 0`, `páginas com
falha: []` — **as três informações eram falsas**.

**A correção não foi "aumentar o teto".** Foi mudar a regra: extração interrompida **falha
explicitamente, na origem**. Aumentar o teto sem essa regra teria escondido o problema seguinte.

E a ordem importou: **o teto folgado foi ligado antes do fail-closed**. Na ordem inversa, a próxima
ingestão teria reprovado em massa justamente os documentos que hoje passam — trocando um problema
silencioso por uma parada geral.

O resultado depois do conserto, no conjunto reprocessado: **mediana de 5,3 vezes mais conteúdo,
máximo de 646 vezes, 24 de 24 documentos melhoraram e nenhum piorou**.

**O limite honesto desta seção.** Não existe hoje detector de *colapso* de extração — o corte é
binário entre "zero caractere" e "íntegro", e é exatamente na faixa intermediária que moram os
documentos truncados. Um detector de "muito menos texto do que o esperado para esta quantidade de
páginas" é a lacuna mais valiosa que este pipeline conhece sobre si mesmo, e ela está registrada
como tal.

### 6.3 Impressão digital de política: reprocesso cirúrgico em vez de reingestão total

Reprocessar 20.905 páginas custa dezenas de horas. Reprocessar 995 páginas custa uma noite. A peça
que permite escolher é o **`extraction_policy_fingerprint`** — um número calculado a partir das
configurações de extração de cada documento. Se ele muda, aquele documento precisa ser reprocessado;
se não muda, ele é pulado.

Três consequências práticas, e as três já foram fonte de erro real:

1. **Onde a chave mora muda o escopo do reprocesso.** A mesma chave `table_mode: accurate` declarada
   no bloco global forçaria **564 documentos / 20.905 páginas**; declarada na configuração por
   documento, força **só os documentos nomeados**. A diferença entre as duas é uma janela de horas
   contra uma janela de dias.
2. **A impressão digital não é o portão que parece ser.** Ao preparar uma janela, o valor vigente
   divergia em **564 de 564 documentos** — sozinho, ele mandaria o acervo inteiro para reprocesso.
   Quem realmente protegia os outros 484 era o **filtro de listagem de arquivos**: o que não é
   listado não é baixado, e o que não é baixado não é reprocessado.
3. **A impressão digital tem que ser calculada com as variáveis de ambiente já resolvidas.** Calcular
   sobre a variável crua produz um número diferente do real — foi exatamente a origem de um falso
   alarme de reprocesso total. **Isso hoje é um portão no código, não uma recomendação:** o cálculo
   **falha explicitamente** se sobrar qualquer variável não expandida. Um arquivo de ambiente não
   carregado não consegue mais mandar 21 mil páginas para reprocesso em silêncio.

### 6.4 Proveniência por trecho: as duas numerações de página

Cada trecho guarda de onde veio. E, por causa do problema da §3.3, guarda **duas** páginas: a física
do arquivo e a impressa no rodapé.

O que torna essa solução elegante é que ela **não exigiu reprocessar nada**. A numeração impressa foi
**derivada do texto já indexado**: o marcador de página injetado no recorte (§5.5) fica ao lado do
número que o extrator capturou do rodapé; medir a diferença entre os dois, trecho a trecho, dá o
deslocamento do documento. Uma única varredura de 77.441 trechos, em **11 minutos**.

E — este é o ponto de qualidade — **a derivação recusa quando não tem confiança**:

| Confiança | Regra | Documentos |
|---|---|---:|
| Alta | ≥ 10 pares **e** ≥ 95% no mesmo deslocamento | 36 |
| Média | ≥ 3 pares **e** ≥ 90% no mesmo deslocamento | 49 |
| Baixa | há par, mas não sustenta decisão automática | 95 |
| Sem página impressa | nenhum par legível | 378 |

Aplicável = alta ou média, sem faixa secundária de numeração: **85 documentos de 558**. Parece pouco
até você olhar a cobertura real:

| Métrica | Aplicável | Total | % |
|---|---:|---:|---:|
| Documentos | 85 | 558 | 15,2% |
| **Trechos indexados** | **47.321** | 77.441 | **61,1%** |
| **Páginas físicas** | **10.045** | 15.884 | **63,2%** |

A explicação está no perfil do acervo: **26 dos 34 documentos com mais de 200 páginas (76,5%) têm
deslocamento aplicável**, contra 5 de 355 nos documentos de até 10 páginas. Os manuais grandes — que
é onde as perguntas moram — quase todos têm rodapé.

**E a hipótese alternativa foi testada antes de ser descartada**, não presumida: a regra secundária
"o rodapé está no fim do bloco" resgataria **0 documentos**, porque só **4,1% dos blocos** terminam
em número puro, e onde terminam o deslocamento é incoerente.

**E há uma armadilha desta seção que vale contar, porque ela é instrutiva.** O carimbo da página
impressa era calculado corretamente e **descartado por uma lista de campos permitidos, antes de
gravar, em silêncio** — a ingestão seguia verde e ninguém era avisado. O acervo de produção só tem o
campo porque uma correção posterior o escreveu direto, e qualquer reingestão o teria apagado de
novo. A lição virou regra: **verifique que o campo novo sobrevive à gravação**, com teste de
contrato que falha se ele sumir.

Além da página, cada trecho carrega **identidade normativa** vinda do registro de catalogação:
código do documento, ano de publicação, tipo, subtipo, disciplina e órgão emissor — presentes em
**535 de 541 documentos amostrados (98,9%)**, e conferidos um a um em **13 de 13** documentos-chave.

### 6.5 Substituição de geração: um acervo vivo é um acervo reprocessado

Quando um documento é reingerido, a geração anterior tem que sair. A chave que define uma geração
precisa levar **os dois** componentes que a determinam: o conteúdo do arquivo **e** a política de
extração usada. Quando ela levava só os bytes, reprocessar o mesmo arquivo com outra configuração
produzia a mesma chave — e a limpeza, que dizia "apague tudo deste documento exceto a versão ativa",
casava com tudo e não apagava nada.

Resultado medido: **8.703 trechos mortos em 77.441 (11,24%)**, em **57 documentos**; **507 de 564
documentos (89,9%) limpos** e 57 servindo mais de uma geração ao mesmo tempo.

E o número que mostra por que a correção teve que ser na origem, e não uma faxina: **em 33 dos 57
documentos a geração viva era MENOR que a morta**, e em 15 delas era menos de um terço. Uma limpeza
cega teria removido **72% do conteúdo** desses documentos. A versão "antiga" era a boa.

A correção foi feita na origem e protegida por um **teste de arquitetura** — um teste que lê o
próprio código-fonte e reprova se alguém reintroduzir a fórmula antiga. Sem ele, a correção volta
atrás no próximo refactor.

### 6.6 Segregação: isolamento entre clientes e entre ambientes

A plataforma é multi-cliente e roda em ambientes que compartilham infraestrutura física. Duas
formas de isolamento operam aqui, e **elas têm forças diferentes — o que precisa ser dito com
precisão**:

- **Isolamento por cliente é estrutural e obrigatório.** O escopo de qualquer busca é resolvido a
  partir de um par — o código do cliente e o identificador lógico do acervo declarado na
  configuração governada —, e é esse par que um registro central traduz no **alvo físico**. Não é
  um filtro aplicado sobre uma base comum, que alguém pode esquecer de aplicar: é o endereço do
  recurso. A coleção do acervo DNIT em produção é `qdrant_engenharia_dnit_dnit_producao_v2` — o
  identificador do cliente está no nome.
- **Isolamento por ambiente é provado uma camada acima**, e não no nome do alvo vetorial: ele vem da
  chave de acesso (que é emitida por ambiente), da configuração do cliente e da release ativa, além
  de conexões e credenciais distintas. A plataforma **tem** um helper canônico de ambiente, usado
  nos demais recursos compartilhados — cache, filas, memória, jobs. **Na camada vetorial, o ambiente
  aparece no valor do identificador por convenção, não por composição programática.**

> Essa distinção é feita de propósito. É a diferença entre "o sistema não consegue vazar entre
> clientes" e "os ambientes estão separados porque a infraestrutura e a configuração os separam".
> As duas afirmações são verdadeiras; elas não são a mesma afirmação.

### 6.7 O que a ingestão entrega, medido

Ao fim da esteira, o que existe é um acervo com propriedades verificáveis:

- **564 documentos ativos, 20.905 páginas, 65.324 trechos indexados**, com integridade provada ponto
  a ponto entre o registro de ingestão e o armazenamento físico.
- **Três representações por trecho** (significado, palavra, interação tardia).
- **Proveniência de página física em todos, impressa em 61,1% dos trechos.**
- **Identidade normativa e ano de publicação em 98,9% dos documentos.**
- **Uma geração viva por documento**, com **84 de 84 documentos reprocessados e nenhuma duplicação**.

> **Nota de leitura dos números.** Este documento cita duas contagens de trechos, e elas são
> **fotografias de momentos diferentes**, não uma inconsistência: **77.441** é o acervo antes da
> janela de reprocesso — o número sobre o qual foram feitos o censo de gerações mortas e a derivação
> de página impressa —, e **65.324** é o acervo depois, com as gerações duplicadas eliminadas na
> origem. Sempre que um percentual for citado, ele vale sobre o total do seu próprio momento.

### 6.8 O que ainda falta

Um acervo bem construído não responde nada sozinho. A partir daqui o problema muda de natureza:
**escolher ~20 trechos entre 65 mil, e responder com endereço que o engenheiro consiga conferir.**

---

# Parte III — O pipeline de resposta

## 7. Escolher 20 trechos entre 65 mil — e provar depois o que foi usado

### 7.1 O problema desta camada

Com o acervo indexado, uma pergunta chega e o sistema precisa fazer três coisas em poucos segundos:
**escolher** os trechos certos, **entregar** um lote que caiba no contexto do modelo, e produzir uma
resposta **rastreável** — com documento, item, quadro e página que o engenheiro consiga abrir e
conferir.

Cada uma dessas três tem uma dificuldade própria, e a terceira é a mais subestimada: uma resposta
tecnicamente correta com endereço errado é, na prática de engenharia, uma resposta que não serve.

### 7.2 Busca híbrida com fusão — três representações, um ranking

A busca não é "uma busca". Ela colhe candidatos por **dois caminhos independentes** e depois os
funde:

1. **Caminho semântico** — o vetor denso encontra trechos com o *significado* próximo da pergunta,
   mesmo sem palavra em comum. É o que acha "bueiro" quando a pergunta escreveu "galeria".
2. **Caminho lexical** — o vetor esparso, no estilo BM25, encontra o termo literal e dá peso a
   termos raros. É o que acha "Bueiro Tubular" e "IS-203" exatamente como estão escritos.

Os dois caminhos devolvem listas com escalas de pontuação **incomparáveis entre si** — e é aí que
entra a **fusão DBSF** (*Distribution-Based Score Fusion*): em vez de somar posições, ela normaliza
cada lista pela distribuição das próprias pontuações antes de combinar. O resultado é um ranking
único que respeita a força relativa de cada caminho.

O tamanho da colheita é configurável: o pipeline colhe **100 candidatos** no estágio anterior à
fusão e entrega **20 trechos** ao modelo — e esses dois números são verificados por sonda antes de
cada rodada de medição (§8.2).

**E a largura da colheita foi medida, não estimada.** Subir os candidatos de 100 para 500 em 8
consultas críticas — cinco vezes mais trabalho — **comprou 1 alvo de 8**. Fica registrado como
ajuste barato de ganho marginal, e **não** como alavanca.

### 7.3 Reranking — e o mérito de medir e desligar o estado da arte

Depois da fusão vem, em quase todo pipeline moderno de RAG, uma etapa de **reranking**: reordenar os
candidatos com um modelo mais caro e mais preciso. Aqui ela é feita **dentro do banco vetorial**,
sem chamada externa, usando o terceiro vetor descrito em §5.6 — a representação por token comparada
termo a termo (ColBERT / MaxSim). Fazer isso dentro do banco elimina uma chamada de rede e um modelo
separado para hospedar no caminho quente da resposta — é a forma mais econômica de ter reranking de
interação tardia.

E a medição mostrou o contrário do esperado.

O que foi medido é a **dispersão do lote entregue**: a diferença percentual entre a maior e a menor
pontuação dos 20 trechos. Se ela é alta, o modelo recebe uma ordenação informativa — sabe quais
trechos são claramente melhores. Se é próxima de zero, os 20 estão empatados.

| Configuração | Buscas medidas | Dispersão (mediana) | Buscas com tudo empatado |
|---|--:|--:|--:|
| Com reranking | 281 | **1,94%** | **281 (100%)** |
| Sem reranking | 260 | **56,38%** | **1 (0,4%)** |

> **A busca já separava bem — 56% de dispersão. A etapa seguinte achatava para 2%.** O reranking não
> deixava de criar ordenação: ele **apagava** uma ordenação informativa que já existia.

E a consequência é direta: com 20 trechos empatados dentro de 2%, **qual deles o modelo prioriza
vira sorteio**. O sorteio foi flagrado no ato — uma pergunta feita duas vezes na mesma hora, no mesmo
acervo, citou o quadro certo às 02:51 e o quadro errado às 03:23. E numa pergunta específica o
efeito é limpo: com reranking ligado ela deu `A`, `C` e `D` em três execuções idênticas; desligado,
deu `A`, `A`, `A`.

**O rigor da comparação, porque ele é o que dá peso à conclusão:** os dois braços rodaram com **N=3
nas 18 perguntas — 108 execuções**, contra o mesmo acervo físico (564 documentos, 65.324 trechos), a
mesma configuração exceto a chave em teste, e **no mesmo processo de servidor, sem reinício entre
eles**. A única variável era o reranking.

**O que a medição autoriza dizer, e o que não autoriza — os dois.** Na comparação pareada, desligar
o reranking foi melhor em 7 perguntas, pior em 2 e empatou em 8, com valor-p de **0,090**: direção
consistente, **sem significância estatística**. A mediana do placar **empatou**. O que melhorou foi o
resto — notas `A` de 13 para 18, notas `D` de 11 para 8, referências adequadas de 4 para 7, negativa
prematura de 1 para 0 — ao custo de **~15% de latência**.

> **Esta não é uma alavanca de placar. É uma alavanca de estabilidade** — e estabilidade é
> pré-requisito de qualquer meta de acerto. Um sistema em que 10 de 18 perguntas variam de nota entre
> execuções idênticas não sustenta um resultado alto nem quando o alcança uma vez.

**E há uma armadilha de experimento que vale registrar**, porque ela quase inverteu a conclusão:
com o reranking ligado, quem governa quantos trechos são entregues é um parâmetro; desligado, é
outro. Um teste anterior desligou o reranking sem ajustar o segundo parâmetro, mediu a colheita
colapsada, e concluiu que o reranking ajudava. **A comparação correta exigiu uma correção de código
antes de medir de novo.**

### 7.4 Filtros e escopo: o que a busca consegue restringir

Além da relevância, a busca aceita restrições declaradas:

- **Escopo por acervo e por cliente**, resolvido a partir da configuração governada da requisição: a
  busca só alcança a coleção declarada para aquele cliente, e o identificador do cliente faz parte do
  nome físico dela (§6.6).
- **Filtro por lista de documentos** na própria pergunta: o chamador pode restringir a busca a um
  conjunto de documentos identificados, o que transforma a pergunta genérica numa consulta dirigida.
- **Filtros por metadado** (*self-query*), com um limite honesto e conhecido: **os índices de payload
  hoje existentes cobrem identidade e versão do documento** — chave do documento, identidade, versão,
  hash de conteúdo, página. **Ano de publicação não tem índice**, e por isso filtrar por vigência
  diretamente na busca hoje é recusado pelo banco.
- **Diversidade no lote** (MMR — *Maximal Marginal Relevance*, técnica que penaliza candidatos
  redundantes para que o lote entregue não fique concentrado num documento só). Aqui vem a
  declaração mais desconfortável — e mais importante — desta seção: **as chaves de MMR existem na
  configuração, estão desligadas em produção, e no caminho híbrido usado pelo acervo DNIT elas são
  inertes**, por duas razões independentes verificadas no código: a rota híbrida não repassa o
  parâmetro, e o cliente do banco vetorial não implementa a operação correspondente. **Ligar a chave
  não faria nada.**

Esse último ponto importa mais do que parece, porque **a falta de diversidade no lote é um mecanismo
medido**: numa pergunta, **7 dos 20 lugares vieram do mesmo documento**. Quando a pergunta exige
comparar duas normas, um lote concentrado é a causa direta da falha — e buscar mais fundo no mesmo
documento não ajuda.

> **A consequência prática, dita sem rodeio:** a alavanca de diversidade — que atacaria diretamente
> o limite 2 do capítulo 10 — **não é uma chave de configuração; é desenvolvimento**. Esta
> verificação estava pendente nos documentos da campanha e foi fechada na preparação deste texto.
> Descobrir que uma alavanca barata na verdade é cara **antes** de prometê-la ao cliente é
> exatamente para isso que se lê o código.

### 7.5 Citação: o endereço que o engenheiro consegue conferir

A resposta cita **as duas** numerações, no formato `p. 256 (arquivo p. 259)`: a impressa primeiro,
porque é a que o leitor procura, e a física entre parênteses, porque é a que ele usa para navegar no
PDF.

O efeito dessa mudança foi a maior melhoria de nota da campanha, e ela **não tocou o conteúdo**:

| Indicador | Antes | Depois |
|---|---:|---:|
| Execuções citando a página impressa | **0 de 32** | **52 de 54 (96,3%)** |
| Execuções com referência inadequada | **31,2%** | **3,7%** |

Redução de **82% a 88%** no principal indicador de rastreabilidade, sem reprocessar um único
documento.

### 7.6 Zero alucinação de citação — auditado, não afirmado

A pergunta que todo cliente faz sobre um sistema baseado em modelo de linguagem é: *"ele inventa
fonte?"*.

Aqui ela foi respondida por auditoria, e não por confiança. Em 24 execuções, foram confrontadas
todas as páginas citadas nas respostas contra as páginas dos trechos que a plataforma **efetivamente
entregou** àquela execução:

> **263 de 263 páginas citadas** estavam entre as páginas entregues. Onde o formato da citação
> permitiu casar também o documento, **20 de 20** conferem o par documento + página. **Zero citação
> fabricada.**

O limite dessa afirmação, declarado: parte das citações não expõe o documento em formato casável
automaticamente, então a conclusão "o agente não fabrica citação" é **forte, não absoluta**.

E há um corolário que explica um comportamento que parece derrota e é qualidade: quando o dado não
está no que ela recebeu, **a plataforma diz que não tem, em vez de inventar**. Numa das perguntas do
benchmark ela declara literalmente que não conseguiu decodificar a fórmula — e perde ponto no
gabarito por isso. **Perder ponto por honestidade é o comportamento correto** de um sistema que vai
ser usado para decisão técnica.

### 7.7 Governança e auditoria

Cada requisição carrega três coisas que fazem deste um produto multi-cliente e não uma demonstração:

- **Autenticação e permissão** — chave de API por cliente, com permissão específica para consulta ao
  acervo. Uma chave sem a permissão não consulta.
- **Configuração governada por versão** — o comportamento de runtime (acervo, modelo, busca, agentes,
  tools) vem de uma configuração declarativa publicada e **versionada em releases**. Mudar o
  comportamento em produção é publicar e ativar uma release; **reverter é reativar a anterior, na
  requisição seguinte, sem deploy**. Isso é o que torna uma mudança de configuração de busca uma
  operação reversível em segundos, e não um evento de risco.
- **Identificador de correlação de ponta a ponta** — toda execução nasce com um identificador único
  no boundary de entrada, que atravessa API, agente, tools, busca e worker, e volta na resposta.
  Todos os registros daquela execução ficam sob esse identificador.

**A lacuna, e ela está registrada:** o registro atual guarda, por busca, a consulta usada, a
contagem de resultados, a duração e a faixa de pontuação — **mas não a lista de (documento, página,
pontuação) devolvida**. Auditar uma resposta específica depois do fato exige, hoje, reexecutar a
busca contra o acervo. Fechar essa lacuna é uma recomendação técnica aberta.

### 7.8 O que ainda falta

Um pipeline pode ter todas essas peças e ainda assim não se saber bom. A pergunta que a Parte IV
responde é: **como este pipeline mede a si mesmo, e o que a medição revelou sobre ele.**

---

# Parte IV — A medição

## 8. O harness de avaliação, e o que ele revelou

### 8.1 Por que isto é a parte mais importante do documento

A maioria dos projetos de RAG é avaliada por impressão: alguém faz dez perguntas, gosta das
respostas, e o sistema "está bom". Quando a qualidade cai, ninguém sabe dizer se caiu — e muito
menos onde.

Este projeto tem um instrumento de medição, e é o instrumento que sustenta cada número deste
documento. Ele não é um detalhe de processo: **é a única razão pela qual as afirmações dos capítulos
anteriores são verificáveis em vez de opinativas.**

### 8.2 Como o benchmark funciona

- **18 perguntas técnicas reais**, elaboradas no vocabulário de quem consulta o acervo — 17 delas
  dentro dos critérios de avaliação (uma está fora, com o gabarito em revisão por especialista).
- **Gabarito validado por especialista** para cada pergunta, e — a peça que faz a diferença — uma
  lista explícita de **pontos críticos esperados**: as exigências que a resposta precisa cobrir para
  ser considerada adequada. Não é "a resposta parece boa"; é "a resposta cobre 6 dos 8 pontos".
- **Juiz automatizado por modelo de linguagem**, com temperatura zero, que confronta a resposta com o
  gabarito e os pontos críticos e devolve **nove campos obrigatórios** num formato estrito:

  | Nota | Significado |
  |---|---|
  | **A** | correta e completa — sem erro material |
  | **B** | correta com omissões não críticas — núcleo técnico correto |
  | **C** | parcialmente correta, com omissão relevante para a aplicação |
  | **D** | divergência técnica capaz de alterar a decisão do engenheiro |
  | **E** | incorreta |

  **A+B conta como adequada**; C para baixo, não. Além da nota, o juiz declara os pontos críticos
  cobertos e perdidos, o tipo de falha, se as referências foram adequadas, e dois indicadores
  específicos deste domínio: se houve **afirmação negativa prematura** ("não existe" sem busca
  dirigida) e se a plataforma **executou cálculo indevido** — porque a regra do acervo é que ela não
  faz cálculo de projeto.

  **O juiz é fail-closed:** resposta fora do vocabulário é regerada com o erro realimentado, e três
  falhas seguidas param a corrida. E ele foi **calibrado contra a especialista**: duas sondas contra
  vereditos humanos, ambas batendo na nota. O limite dessa calibração está declarado — a
  classificação reproduziu, mas o *tipo de falha* divergiu num caso, porque o juiz vê só a resposta e
  não o rastro da busca. **Tipo de falha é hipótese diagnóstica, não veredito.**
- **Execuções repetidas com mediana.** O protocolo padrão usa três execuções (N=3) nas perguntas que
  decidem o resultado, e a comparação final rodou **N=3 nas 18 perguntas dos dois braços — 108
  execuções**. A nota é a **mediana**, e **as notas cruas são publicadas junto** — porque `C (ACC)`
  conta uma história que `C` esconde. A razão de existir do N=3 é medida: a mesma versão, na mesma
  pergunta, deu notas diferentes em execuções distintas; com N=1, o resultado do gate decide por
  sorteio.
- **Regras que invalidam uma rodada, declaradas antes de rodar** — e elas nasceram de erros reais:
  configuração velha em cache na tela (medido: 652.262 caracteres na tela contra 657.239 na rota,
  **sem nenhum aviso**), modo de execução errado no seletor, porta ocupada por outra instância,
  interação humana com o navegador durante a corrida, e a proibição de rotular um erro sem abrir o
  registro da correlação.
- **Sondas discriminantes antes de cada rodada cara**: um conjunto pequeno de verificações **em
  tempo de execução** que provam que se está medindo o que se pensa — acervo físico correto,
  composição agentic correta, a busca de fato colhendo 100 candidatos e entregando 20, e a correção
  em teste viva no processo. O critério é declarado por escrito: *"se qualquer uma vier diferente, a
  rodada é inválida"*. A terceira sonda é literal: se a colheita viesse com 20 em vez de 100, a
  rodada seria abortada — foi exatamente essa confusão que produziu um resultado pior numa versão
  anterior. E a quarta nasceu de um achado: **a data de modificação do arquivo não prova qual código
  está rodando**; só a verificação em tempo de execução prova.

### 8.3 Onze versões medidas — o histórico publicado

Onze versões do sistema foram medidas contra o mesmo benchmark. Abaixo, as que rodaram o conjunto
completo de 17 perguntas (uma versão intermediária mediu apenas 6 e é omitida por não ser
comparável):

| Versão | A+B | Distribuição | Referências adequadas | Negativa prematura | Latência mediana |
|---|--:|---|--:|--:|--:|
| V0 (linha de base) | **3/17** | 1A 2B 5C 7D 2E | 0 | 6 | 29,9 s |
| V3 | 4/17 | 3A 1B 9C 4D | 3 | 2 | 180,0 s |
| V4a | 7/17 | 3A 4B 6C 4D | 2 | 1 | 68,0 s |
| V5 | 9/17 | 7A 2B 4C 4D | 7 | 2 | 70,8 s |
| V6 (rerank desligado, colheita colapsada) | 8/17 | 4A 4B 6C 3D | 4 | 0 | 66,9 s |
| **V7 (fusão DBSF, colheita 100)** | **10/17** | 5A 5B 5C 2D | 4 | 0 | 74,0 s |
| Produção pós-janela | 7/17 | 3A 4B 6C 4D | 3 | 2 | 65,2 s |
| A/B final — braço A (rerank ligado) | 7/17 | 4A 3B 7C 3D | 4 | 1 | 67,3 s |
| A/B final — braço B (rerank desligado) | 7/17 | **6A** 1B 8C **2D** | **7** | **0** | 78,5 s |

> **Cuidado ao ler a coluna de latência.** Ela é o tempo que o **cliente** esperou, e **não é
> comparável entre versões medidas por vias diferentes**: uma rodada feita pela API mede até o fluxo
> de eventos *fechar*, e o fluxo só fecha depois de gravar o histórico de replay — depois de a
> resposta já ter sido entregue. O número comparável é o intervalo entre a entrada da requisição e o
> fim da execução do agente, e ele é publicado no relatório de cada rodada. Na medição direta desse
> intervalo, a rodada final marcou **70,9 s com reranking ligado e 81,5 s desligado**.

Duas leituras que o placar sozinho não dá:

- **A trajetória é real:** de **3/17** na linha de base a **10/17** no melhor ponto medido. E três
  versões — a de 10/17 e os dois braços da comparação final — **não pioraram nenhuma pergunta**
  contra a linha de base. Ganho sem regressão é raro em ajuste de RAG, e aqui ele é verificado
  pergunta a pergunta.
- **"Negativa prematura" — a plataforma respondendo "não encontrei" quando o dado estava lá — caiu
  de 6 para 0.** É um indicador de qualidade que o placar não mostra e que importa muito na prática.

### 8.4 A disciplina que quase ninguém aplica: medir a incerteza do próprio instrumento

Antes de dizer "a versão B é melhor que a A", é preciso saber **de quanto é o ruído**. Rodando as 18
perguntas com N=3 na mesma configuração:

> **10 das 18 perguntas dão nota diferente entre execuções idênticas** — mesma pergunta, mesmo
> acervo, mesma configuração. Uma delas variou **três degraus** (`A`, `C`, `D`). A incerteza do
> placar é de **±2 pontos**.

A consequência foi registrada sem suavizar, no próprio diário do projeto: *"7/17, 8/17, 9/17 e 10/17
nunca foram distinguíveis. A campanha vinha comparando versões dentro do próprio ruído."*

Três disciplinas passaram a valer a partir daí, e elas valem para qualquer promessa feita sobre este
sistema: **projeção sai como faixa, nunca como número seco**; **ganhos de alavancas diferentes não se
somam**, porque em boa parte elas disputam as mesmas perguntas; e **delta menor que ±2 não decide
nada**.

> Descobrir que o próprio instrumento tem ±2 de incerteza é um resultado desconfortável e caro. É
> também o resultado que separa medição de teatro de medição.

### 8.5 O achado que reenquadra tudo: o benchmark tem dois blocos que nunca se misturaram

Cruzando as 11 versões medidas e perguntando *"qual foi a melhor nota que cada pergunta já alcançou
alguma vez?"*, o benchmark se parte em dois:

- **Bloco 1 — 10 perguntas** que já chegaram a nota adequada pelo menos uma vez.
- **Bloco 2 — 7 perguntas** que nunca passaram de `C` **em nenhuma das 11 versões** — e duas delas
  nunca passaram nem de `D`.

E o fato que fecha o argumento: **a melhor rodada da campanha, com 10/17, acertou exatamente as 10
perguntas do Bloco 1 — todas elas, e nenhuma do Bloco 2.**

> **10/17 não é "quase o recorde". É o máximo estrutural da arquitetura atual** — o resultado de
> todas as perguntas resolvíveis serem resolvidas ao mesmo tempo. Nenhuma configuração de busca,
> reordenação, engine de extração ou acervo levou uma única pergunta do Bloco 2 à faixa adequada.

Isso separa o problema em duas naturezas completamente diferentes, e tratá-las como a mesma coisa
foi, olhando para trás, o erro de enquadramento da campanha:

- **De 7 para 10** = fazer a plataforma **repetir o que já faz**. Problema de **estabilidade**.
- **De 10 para 15** = fazer a plataforma **fazer o que nunca fez**. Problema de **capacidade**.

### 8.6 O número que explica o teto: alvo único × síntese

Cada pergunta declara quantos pontos críticos a resposta precisa cobrir. Contando-os, o benchmark se
parte de novo — e desta vez o corte explica o Bloco 2:

| Tipo de pergunta | Quantas | Adequadas | Taxa |
|---|--:|--:|--:|
| **Alvo único** (1 ponto crítico) | 7 | 5 | **71%** |
| **Síntese** (7 a 9 pontos críticos) | 10 | 2 | **20%** |

> **A plataforma acerta 71% do que pede uma coisa e 20% do que pede sete a nove coisas de uma vez.**

Das 7 perguntas do Bloco 2, **6 são de síntese**. A única de alvo único é uma fórmula que está dentro
de uma imagem.

E daí sai a aritmética que nenhum otimismo desfaz: chegar a 15 ou 16 de 17, mesmo acertando **todas**
as 7 de alvo único, exige ganhar **8 ou 9 das 10 de síntese** — levar esse grupo de 20% para ~85%.
**Não existe caminho para 90% que não passe pela síntese multi-documento.**

### 8.7 Três causas descartadas — com prova, não com opinião

Antes de aceitar qualquer explicação, a campanha eliminou as três hipóteses óbvias:

- **Não falta documento no acervo.** Das 21 referências exigidas pelos gabaritos, **19 de 19
  documentos estão no acervo**, e **0 de 18 perguntas** tem norma ausente. Numa das perguntas, o
  replay da busca mostra o manual certo em **1º lugar** e o segundo manual certo em **3º** — **o
  endereço é encontrado; o conteúdo é que não está no endereço.**
- **A plataforma não inventa citação.** 263 de 263 páginas conferidas (§7.6).
- **Buscar mais fundo não resolve.** 5× mais candidatos, 1 alvo de 8 (§7.2).

E a trava metodológica que fecha o argumento: na rodada final, **108 de 108 execuções terminaram sem
resposta vazia, sem erro de execução, sem exceção e sem estouro de tempo**.

> **Logo, toda queda de nota medida é limitação de estratégia, de dado ou de régua — nunca defeito de
> implementação.** Essa frase é a conclusão técnica central deste documento, e ela é uma conclusão
> medida, não uma alegação.

### 8.8 O que ainda falta

Medido o teto da busca por trechos, sobra a camada que decide **como** as buscas são feitas: o
agente.

---

# Parte V — A camada agentic

## 9. DeepAgent sobre o acervo

### 9.1 O problema que o agente resolve

Uma consulta de RAG clássica faz **uma** busca e responde. Isso funciona para "qual é o valor de X".
Não funciona para a pergunta que o engenheiro realmente faz: *"para bacia de 4,50 km², qual método
usar, comparando os critérios dos documentos aplicáveis?"* — que exige buscar, olhar o que voltou,
perceber que falta o segundo critério, e **buscar de novo com outro termo**.

Essa é a diferença entre consulta e **agente**: o agente decide o que buscar, lê o resultado e
decide se busca de novo.

### 9.2 A arquitetura, e ela é declarada em configuração

A camada agentic desta plataforma é construída sobre LangGraph/LangChain e é **inteiramente
governada por configuração declarativa** — o que existe no acervo DNIT hoje é:

- **Um supervisor** (`deepagent`) que **não pesquisa e não interpreta**: recebe a pergunta, delega ao
  especialista e repassa a resposta com as fontes intactas. O prompt dele é explícito ao ponto de
  proibir a reescrita: *"não resuma, não reescreva e não reordene as referências — elas são o que
  torna a resposta auditável por quem vai projetar"*.
- **Um subagente especialista**, o pesquisador do acervo, que consulta em camadas, confere se existe
  norma posterior ou mais específica sobre o mesmo assunto, e sintetiza cada critério com seu escopo
  de aplicação e sua fonte. Os limites dele também são declarados: *"responde apenas com o que os
  trechos recuperados sustentam, não executa cálculo numérico de projeto e não inventa documento,
  item ou página"*.
- **Uma tool governada**, `qa_governed_retrieve` — recuperação crua: devolve os trechos e para aí,
  sem gerar texto. É o subagente que decide o que fazer com eles. Note onde ela mora: **o supervisor
  não tem tool nenhuma**. Ferramentas vivem exclusivamente nos subagentes, o que torna a superfície
  de cada camada explícita em vez de herdada.
- **Um teto explícito de buscas por execução** (`max_tool_calls: 10`), que é a fronteira entre "busca
  iterativa" e "loop caro sem fim".

### 9.3 O que a governança por configuração compra aqui

Este é um ponto de arquitetura que costuma passar despercebido e que vale destacar. Todas as
capacidades do agente são **chaves ligáveis por cliente**, e no acervo DNIT elas estão
deliberadamente **desligadas**:

```
filesystem: desligado          shell: desligado
memória: desligada             human-in-the-loop: desligado
lista de tarefas: desligada    interpretador de código: desligado
subagentes: LIGADO
```

Um agente que responde sobre norma técnica **não deve** ter acesso a sistema de arquivos, execução
de comando ou interpretador. Isso não é uma limitação do produto: é **superfície de ataque removida
por decisão declarada**, e ela é auditável lendo uma configuração em vez de lendo código.

E o versionamento segue a mesma lógica: **quatro versões da composição agentic convivem no mesmo
documento de configuração**, e uma única linha (`selected_entrypoint`) escolhe qual está ativa.
**Reverter uma mudança de comportamento do agente é trocar uma linha e ativar a release** — sem
deploy, sem janela, na requisição seguinte.

### 9.4 A evolução medida da composição agentic

As quatro versões não são variações estéticas; cada uma testou uma hipótese, e cada uma foi medida:

| Versão | Hipótese testada | Resultado |
|---|---|---|
| **V2** | orquestrador + 1 subagente que terceiriza pergunta e resposta ao pipeline Q&A em uma chamada | rodada parcial (6 perguntas), não comparável — o objetivo era provar **paridade** com o modo Q&A antes de qualquer sofisticação |
| **V3** | **busca iterativa** — o subagente pesquisa em camadas com a tool de recuperação crua e escreve ele mesmo a resposta | 4/17 |
| **V4a** | **conduta normativa** — o prompt passa a cobrar o que o gabarito cobra: classificar a relação entre documentos, buscar norma posterior em consulta nova, colar o escopo de aplicação em cada critério e auto-auditar a resposta contra os trechos antes de fechar | 7/17 — **a versão ativa hoje** |
| **V4b** | **revisor normativo** — um segundo subagente que não responde e não reescreve: contesta a resposta já redigida pesquisando o acervo por conta própria e devolve um veredito estruturado | **empatou, custando 41% a mais** — medido e **não adotado** |

> A linha da V4b é a mais importante desta tabela. Uma sofisticação agentic real foi construída,
> medida e **descartada por não pagar o próprio custo**. Um projeto que só publica o que deu certo
> não está medindo — está selecionando.

### 9.5 Interface e streaming

A resposta é entregue por streaming pelo protocolo AG-UI, o canal padrão de interação agentic da
plataforma: o usuário vê o texto sendo produzido e, quando o agente aciona uma ferramenta, o evento
correspondente atravessa o mesmo canal. É a mesma superfície usada pelas demais aplicações agentic
do produto — o acervo DNIT não tem um caminho próprio.

### 9.6 O custo real da busca iterativa, e o teto que ele encosta

Buscar de novo custa. Medido na rodada final: **3 execuções encostaram no teto de 10 buscas**, e a
pergunta mais rica do benchmark levou **317,7 e 323,5 segundos** — mais de cinco minutos — em duas
das três execuções.

Isso tem uma consequência que precisa ser dita antes de qualquer promessa: **instruir o agente a
buscar cada norma separadamente exige elevar o teto de buscas, e isso sobe latência e custo por
pergunta.** As alavancas de conduta e as de leitura direta **competem pelo mesmo orçamento**. Somar
ganhos projetados sem revisar esse orçamento é como derrubar as perguntas ricas no meio do caminho.

### 9.7 O que ainda falta

Chegamos ao ponto em que todas as camadas estão descritas e medidas. A pergunta que sobra é a que o
cliente vai fazer: **por que, com tudo isso, o benchmark não vai a 100% — e por que uma ferramenta
genérica parece conseguir?**

---

# Parte VI — O fechamento do arco

## 10. Por que 100% do benchmark é estruturalmente difícil

O capítulo 8 provou que as quedas não são defeito de implementação. Este capítulo explica **o que
elas são**. São três, elas são de naturezas diferentes, e nenhuma se resolve "melhorando a busca" —
porque nenhuma é problema de busca.

### 10.1 Limite 1 — o que a extração perde não existe para ninguém

Este é o limite direto da regra-mãe do capítulo 1. Uma fórmula que é imagem não gera texto; o trecho
sai sem ela; e o sistema não sabe.

O caso mais claro é uma pergunta que exige a fórmula de Rehbock. O que a plataforma faz, medido nas
três execuções:

- acha o manual certo, o item certo e a página certa;
- diz honestamente que não conseguiu decodificar a fórmula — **e não a inventa**;
- perde a pergunta por esse único ponto, em **todas as 11 versões medidas**.

E essa pergunta já sobreviveu ao tratamento mais caro disponível: numa das versões, o manual recebeu
**1.002 trechos de transcrição de figura por visão de modelo**, e a fórmula **continuou** não
chegando íntegra. **Transcrever figura não resolveu.**

> **Nenhuma configuração de busca cria o que não foi indexado.** Este limite não é uma opinião sobre
> a arquitetura: é a consequência aritmética dela.

**O que existe de remédio, e ele é dirigido:** o enriquecimento de fórmula da §5.2 recuperou o alvo
dessa pergunta e de outras duas, em bancada, com a ressalva de qualidade declarada (o sinal de
igualdade não saiu). Recuperar o alvo no índice é condição **necessária, não suficiente** — e por
isso o item aparece no roadmap com experimento de validação antes de investimento, e não como
promessa.

### 10.2 Limite 2 — perguntas que pedem oito evidências ao mesmo tempo

Dez das 17 perguntas do benchmark não pedem um dado. Pedem sete a nove exigências numa resposta só:
comparar duas normas, apresentar dois critérios concorrentes **sem escolher entre eles**, citar item,
quadro e página, recomendar conferir o desenho, e não extrapolar.

Para acertar isso com 20 trechos, **as oito evidências precisam estar entre os 20 ao mesmo tempo**.
Nada garante isso — e há um agravante medido: o lote entregue costuma ser **concentrado**, com **7
dos 20 lugares vindos do mesmo documento** numa pergunta medida. O segundo documento não cabe.

Daí o número que resume a campanha: **71% em alvo único, 20% em síntese**.

**Não é falta de conhecimento — é falta de completude.** E é um teto de arquitetura, não de esforço:
um sistema que responde a partir de ~20 trechos tem um limite natural de quantas exigências
independentes consegue cobrir simultaneamente.

### 10.3 Limite 3 — provar que uma coisa NÃO existe

Uma das perguntas exige reconhecer que um manual **não traz** valor mínimo de declividade em três
páginas específicas.

Isso é impossível por trechos, e a razão é **lógica, não técnica**:

> **Quem vê 20 trechos só sabe o que veio, nunca o que não veio.** A ausência de um trecho pode
> significar "não existe" ou "a busca não achou" — e não há como distinguir os dois de dentro.

Ler as três páginas inteiras resolve. Buscar melhor, não. Nenhuma quantidade de reordenação,
expansão de vocabulário ou ajuste de prompt transforma "não recuperei" em "não existe".

### 10.4 O que isso soma

Das 7 perguntas nunca resolvidas: **duas** caem no limite 1 (conteúdo fora do índice), **quatro**
caem no limite 2 (síntese multi-documento), **uma** cai no limite 3 (prova de ausência) — e há
sobreposição, porque uma pergunta pode ter mais de uma barreira.

Nenhuma delas cai por errar o valor técnico. A rodada de produção registrou o mesmo: *"nenhuma das
quedas foi por perder conteúdo técnico"*. **O que se perde é completude e endereço, não verdade.**

---

## 11. Por que o ChatGPT "consegue" — e por que isso não escala

### 11.1 Aviso de honestidade, antes da comparação

**Não houve comparação direta.** Nenhuma rodada mediu o ChatGPT contra este pipeline sobre as mesmas
17 perguntas. O que vem a seguir é uma comparação **arquitetural** — por que os dois modelos se
comportam de forma diferente —, e não um placar. Qualquer afirmação numérica sobre o desempenho da
ferramenta genérica seria invenção, e este documento não faz isso.

> **E a diferença, quando ela existe, não é inteligência do modelo. É ordem de leitura.** O ChatGPT
> lê o documento **depois** de ouvir a pergunta. Este pipeline leu o documento **antes**, meses
> antes, sem saber que pergunta viria.

### 11.2 O estagiário com o manual aberto na mesa

Imagine um engenheiro júnior muito rápido. Você chega com uma pergunta e com o manual debaixo do
braço. Ele **abre o manual naquele momento**, folheia, lê as páginas que interessam — o texto, a
tabela, o desenho, a fórmula, tudo junto, do jeito que está impresso — e responde.

Três forças decorrem disso, e elas são exatamente os três limites do capítulo 10:

- **Ele vê a página inteira.** A fórmula que é imagem, a tabela com as linhas alinhadas, a legenda
  embaixo do desenho: está tudo lá, porque ele está olhando a página, não uma descrição dela.
  *(Anula o limite 1.)*
- **A pergunta guia a leitura**, e ele pode abrir quantos manuais precisar. *(Anula o limite 2.)*
- **Ele pode dizer "não tem".** Se leu as páginas 122 a 124 inteiras e não encontrou percentual
  nenhum, ele **sabe** que não encontrou — porque leu tudo. *(Anula o limite 3.)*

E as três fraquezas, que são o outro lado exato da mesma moeda:

- **Ele não guarda nada.** Na próxima pergunta, abre o manual de novo, do zero.
- **Não escala.** Com 564 manuais e 21 mil páginas, "ler o que interessa" deixa de ser possível: é
  preciso saber **qual** manual abrir — e essa escolha já é um índice.
- **Não dá para auditar nem governar.** Você não sabe exatamente o que ele leu, não consegue provar
  depois, e não há como dizer "este cliente pode ver estes documentos e não aqueles".

### 11.3 A biblioteca com fichário

Agora imagine uma biblioteca. Meses atrás, alguém pegou os 564 manuais, leu **todos**, e recortou
cada um em milhares de fichas de mais ou menos meia página. Escreveu em cada ficha de onde ela veio —
manual, página impressa, página do arquivo, norma, ano — e guardou tudo num fichário gigante: no
nosso caso, **65.324 fichas**.

Quando a pergunta chega, o bibliotecário não abre manual nenhum. Ele procura no fichário as **~20
fichas mais parecidas** e entrega essas 20 a quem vai responder. **Quem responde nunca vê o manual —
vê 20 fichas.**

As forças são exatamente onde o outro é fraco: **rápido e barato por pergunta**, porque o trabalho
pesado já foi feito uma vez; **escala**, respondendo sobre 564 manuais com o mesmo esforço com que
responderia sobre 5; **auditável e governável**, porque cada ficha tem endereço.

E a fraqueza é uma só, mas é enorme:

> **Se o que você precisa não estiver nas fichas — ou não estiver entre as 20 entregues —, a resposta
> não existe.** E o bibliotecário não sabe disso: ele entrega as 20 melhores que tem, mesmo quando as
> 20 melhores não servem.

### 11.4 O outro lado da conta: o que o modelo "ler na hora" cobra

O preço do estagiário não aparece quando se testa **uma** pergunta:

- **Custo e tempo crescem com o documento, em cada pergunta.** Ler um manual de 500 páginas para
  responder uma pergunta é, em ordem de grandeza, muito mais caro do que ler 20 trechos — e paga-se
  isso **toda vez**. Este pipeline paga o custo pesado **uma vez**, na ingestão. Para um produto que
  responde milhares de perguntas por dia, a diferença não é de percentual: é de viabilidade.
- **Alguém tem que escolher o manual.** Com 564 documentos, "leia o manual" pressupõe saber qual —
  ou é o usuário fazendo o trabalho do índice à mão.
- **Não há governança.** Multi-cliente, controle de acesso por chave e permissão, isolamento entre
  empresas: nada disso existe em "arrastei o PDF para o chat". Aqui, cada busca é escopada por
  cliente e por acervo, e isso é requisito de contrato, não capricho.
- **Não há auditoria.** Aqui, toda resposta tem identificador de execução, e uma auditoria conferiu
  **263 de 263 páginas citadas** contra o que foi entregue ao modelo. Provar isso sobre uma leitura
  livre de PDF é consideravelmente mais difícil.
- **Não há acervo.** A ferramenta genérica responde sobre o PDF que você anexou. Ela não responde
  sobre "o acervo DNIT", não sabe que existe uma norma de 2026 que muda o critério de 2006, e não
  mantém um catálogo vivo com vigência — que aqui existe em **98,9% dos documentos**.

> **Os dois modelos não estão competindo pela mesma vaga. Um é o estagiário; o outro é a
> biblioteca.** Uma consultoria escolhe o estagiário para uma dúvida pontual sobre um PDF que ela já
> tem na mão. Uma organização que precisa responder sobre o próprio acervo normativo, para muitos
> usuários, com rastreabilidade e controle de acesso, não tem essa opção.

### 11.5 Então por que esta plataforma foi feita assim?

Porque o produto não é "responder uma pergunta sobre um PDF". É **manter um acervo técnico vivo,
multi-cliente, governado e auditável, respondendo milhares de perguntas por dia**. Para isso, o
fichário é a arquitetura certa — e é a única que escala.

E o que a campanha mostrou é que **as duas coisas não são excludentes**.

---

## 12. A convergência — e o que este documento promete e não promete

### 12.1 A resposta certa não é trocar de arquitetura

É **deixar o bibliotecário abrir o manual quando as fichas não bastarem**.

E o mais interessante é que **o sinal para acionar isso já existe, medido, no comportamento atual**.
Quando a plataforma responde *"não foi possível decodificar a fórmula"* ou *"o trecho recuperado
estava truncado"*, ela está dizendo literalmente *"as fichas não bastam aqui"*. Esse comportamento
aparece em pelo menos três perguntas do benchmark. **Hoje ela diz isso e para. Poderia dizer isso e
ir abrir a página.**

### 12.2 ROADMAP — o que existe, o que falta, e o que decide cada item

> **Tudo nesta seção é projeto ou decisão pendente. Nada aqui é capacidade entregue.** A ordem é a
> ordem recomendada, e cada item traz o que o valida.

**Escalada para leitura direta do documento** — a peça central, e **não medida**.

- ✅ **Existe:** a plataforma já sabe **baixar o PDF original** de um documento do acervo, ao vivo, da
  origem de onde ele foi ingerido, por um endpoint dedicado (`POST /rag/documents/content`), escopado
  por cliente e por acervo, com autenticação, permissão, limite de taxa e teto de tamanho. O binário
  **não é arquivado pela plataforma**: ele é buscado na origem do cliente — Google Drive ou
  armazenamento de objetos —, o que preserva a governança do dado. Origem não suportada é recusada
  explicitamente, e não silenciosamente.
- ✅ **Existe:** a proveniência de página por trecho (§6.4) — é ela que diz **qual intervalo abrir**.
- ❌ **Falta a ligação:** o agente que responde **não tem tool** para pedir "me dá as páginas 122 a
  124 deste manual". Ele só tem a tool de buscar trechos.

> **Não é um problema de pesquisa. É uma peça que não foi conectada.** Se ela funciona é uma pergunta
> **aberta e honestamente não medida** — e há um precedente que obriga cautela: a pergunta da fórmula
> sobreviveu a 1.002 trechos de transcrição de figura. Se a fórmula estiver ilegível **no próprio
> PDF**, ler a página não a inventa.

**Como decidir isso barato, antes de investir:** dar ao agente as três páginas que a pergunta da
prova de ausência exige e verificar se ele conclui corretamente que ali não há o valor. **Menos de
uma hora de experimento**, e decide se o caminho é investimento ou ilusão. Esse é exatamente o
critério aplicado no resto do roadmap: **experimento barato antes de janela cara.**

**Os demais itens, com o que cada um faz e o que ele não faz:**

| Item | O que muda | Base de evidência | O que NÃO faz |
|---|---|---|---|
| **Estabilidade** — desligar o rerank, com a correção de colheita já pronta | menos variância; recupera o próprio melhor (8 a 10/17) | **MEDIDO** — mediana empata, direção consistente em 5 indicadores, p=0,090 | não abre pergunta nova; é pré-requisito, não ganho |
| **Engine de tabela dirigida** na classe de célula colada | consolida o Bloco 1, torna 10/17 reprodutível | **MECANISMO PROVADO** (célula íntegra no índice); efeito no placar dentro do ruído | **provado que não abre o Bloco 2** |
| **Fórmula dirigida** nos 4 documentos, 995 páginas | única alavanca de ingestão que toca o Bloco 2 | **BANCADA** — três alvos recuperados; efeito na resposta **não medido** | recupera o alvo no índice; não garante a nota |
| **Conduta dirigida** de 2ª geração no prompt | apresentar critérios lado a lado, buscar cada norma | **NÃO MEDIDO** para instruções dirigidas; a variante geral **empatou custando +41%** | não cria conteúdo perdido; **exige mais buscas** |
| **Diversidade no lote** | faz o segundo documento caber | **MECANISMO MEDIDO** (7/20 do mesmo documento); mas **confirmado no código que a chave é inerte no caminho híbrido** — deixou de ser ajuste de configuração e virou desenvolvimento | nada enquanto não for implementada no caminho híbrido |
| **Leitura direta** | única rota compatível com a meta alta; **única que resolve a prova de ausência** | **NÃO MEDIDO** — peças existem, ligação não | nada, se não for construída |

**E a regra que impede a aritmética errada:** as alavancas **disputam as mesmas perguntas**, e por
isso **não se somam**. Uma pergunta que aparece em três alavancas vale **um** ponto, não três. Além
disso, conduta dirigida e leitura direta **consomem orçamento de busca**, e desligar o rerank já
custa +15% de latência: empilhar sem revisar o teto de buscas pode **derrubar** perguntas ricas no
meio do caminho.

### 12.3 A decisão de régua — separada de propósito

Existe uma discussão que **não é melhoria de produto e não pode ser vendida como tal**, e por isso
ela aparece isolada.

A régua atual mede **completude normativa**, não utilidade. Das 8 perguntas com nota `C`, pelo menos
**4 entregam o valor técnico correto** e caem por não cobrirem todos os pontos críticos — um
engenheiro usaria as quatro. Contar "resposta utilizável com valor técnico correto" como adequada
moveria o placar de **7 para cerca de 11 de 17 sem alterar uma linha de código**.

Há ainda três imprecisões documentadas na própria régua: o gabarito **mistura as duas convenções de
página** (um deles diz literalmente *"p. 174 do PDF (p. 144 impressa)"*, e escolher um lado perde
ponto no outro); o **juiz varia sobre dado idêntico** (a mesma frase foi relevada numa rodada e
penalizada em outra); e **um ponto crítico cobra o que o próprio gabarito validado não mostra** — na
pergunta da fórmula, o gabarito descreve o contexto mas **não contém a fórmula**, de modo que uma
resposta idêntica ao gabarito perderia o ponto.

> **A única forma honesta de tratar isso é declarando a decisão, com os dois placares publicados
> lado a lado, ANTES da próxima medição.** Mudar a régua em silêncio depois de ver o resultado é
> maquiagem. Recusar a discussão também é errado: uma régua que reprova resposta utilizável mede a
> coisa errada.

### 12.4 O que este documento NÃO promete

- **Não promete a meta alta com o que está medido.** Com todas as alavancas conhecidas juntas, a
  faixa honesta é de **11 a 14 de 17**. Chegar a 15 ou 16 depende de uma capacidade que **hoje não
  existe** e cuja eficácia **não foi medida**.
- **Não promete que a leitura direta funciona.** É a rota certa pelo mecanismo, e é **não medida**.
  Um experimento de menos de uma hora decide.
- **Não promete que recuperar a fórmula no índice vira nota.** A bancada provou a recuperação do
  alvo; a transcrição volta com defeito conhecido e a resposta ainda precisa passar por busca e
  conduta.
- **Não afirma nenhum placar do ChatGPT.** Não houve comparação direta; o capítulo 11 é
  arquitetural.
- **Não trata 7 como fracasso nem 10 como vitória.** Com ±2, os dois convivem na mesma faixa.

### 12.5 O que este documento afirma, e sustenta

- **10 das 17 perguntas já foram respondidas com qualidade adequada por esta plataforma**, numa mesma
  rodada; e as 7 restantes nunca foram, por nenhuma configuração, em 11 versões medidas. As duas
  metades dessa frase importam igualmente.
- **A trajetória é medida:** de 3/17 na linha de base para 10/17 no melhor ponto, com negativa
  prematura caindo de 6 para 0.
- **As correções desta campanha entregaram números grandes e provados:** referências inadequadas de
  **31,2% para 3,7%**; citação de página impressa de **0% para 96,3%**; extração dos documentos
  reprocessados com **mediana 5,3× e máximo 646×**, 24 de 24 melhorando e nenhum piorando; **8.703**
  trechos duplicados eliminados na origem; e a palavra "Bueiro", que **não existia no índice** de um
  manual, hoje casando **114 trechos**.
- **O acervo tem integridade provada ponto a ponto** entre registro e armazenamento físico.
- **A plataforma não inventa fonte** — 263 de 263 páginas citadas conferidas.
- **O que sobra está identificado, classificado em três limites estruturais, orçado, e com
  experimento de validação desenhado para cada aposta cara.**

> O placar não mostra nada disso, porque ele mede outra coisa. **Este documento existe exatamente
> para mostrar a diferença entre "o placar é 7" e "o sistema está errado".**

---

## Anexo A — Glossário 101

- **Chunk (trecho)** — pedaço de mais ou menos meia página em que o documento é recortado para ser
  indexado.
- **Embedding (vetorização)** — transformar texto numa lista de números que representa o
  *significado*, permitindo achar "bueiro" quando alguém escreveu "galeria".
- **Busca híbrida** — combinar busca por palavra literal com busca por significado.
- **BM25 / vetor esparso** — método clássico de busca por palavra que dá mais peso a termos raros.
- **Fusão DBSF** — combinar duas listas de resultados com escalas de pontuação diferentes,
  normalizando cada uma pela distribuição das próprias pontuações.
- **Reranking** — etapa que reordena os candidatos da busca com um modelo mais caro e mais preciso.
- **ColBERT / MaxSim (interação tardia)** — reranking que compara pergunta e trecho **termo a termo**
  em vez de comparar dois resumos numéricos.
- **MMR (diversidade)** — técnica que penaliza candidatos redundantes para que o lote entregue não
  fique concentrado num documento só.
- **OCR** — programa que "lê" uma imagem de texto e devolve as letras. Erra, e erra em silêncio.
- **Engine de extração** — o programa que abre o PDF e devolve texto.
- **Taxa de escalada** — porcentagem de documentos que uma engine erra a ponto de exigir reprocesso
  por outra.
- **Fail-closed** — na dúvida, falhe e avise. O contrário de *fail-open*, que segue em frente.
- **Fingerprint de política de extração** — número calculado a partir das configurações; se muda, o
  documento precisa ser reprocessado.
- **Supersessão** — apagar a geração anterior de um documento quando uma nova é indexada.
- **Ledger** — registro contábil de execuções, em banco, usado como fonte de verdade de estado e
  histórico.
- **Página física × página impressa** — a numeração do arquivo PDF (conta desde a capa) × a numerada
  no rodapé (que o leitor procura).
- **Ponto crítico** — cada exigência que uma resposta precisa cobrir para ser considerada adequada
  pelo gabarito.
- **Alvo único × síntese** — pergunta que pede um dado × pergunta que pede sete a nove exigências.
- **Sonda discriminante** — verificação pequena, em tempo de execução, que prova que o experimento
  está medindo o que se pensa.
- **AG-UI** — o protocolo de interação e streaming entre a interface e o agente.
- **DeepAgent / supervisor** — composição agentic em que um orquestrador delega a subagentes
  especializados.

---

## Anexo B — Como verificar cada afirmação

Este documento é uma síntese. Cada família de número tem uma fonte primária no repositório:

| Assunto | Onde conferir |
|---|---|
| Lições de engenharia da campanha, em 17 regras transferíveis | [README-PLAYBOOK-PIPELINE-INGESTAO-RAG-ROBUSTO.md](README-PLAYBOOK-PIPELINE-INGESTAO-RAG-ROBUSTO.md) |
| Implementação do pipeline de ingestão de PDF | [README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md) |
| Implementação do pipeline de resposta | [README-TECNICO-RAG-PIPELINE-COMPLETO.md](README-TECNICO-RAG-PIPELINE-COMPLETO.md) |
| Job Core, worker e paralelismo | [README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md](README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md) |
| Composição agentic, supervisor e subagentes | [README-TECNICO-DEEPAGENT-SUPERVISOR-COMPLETO.md](README-TECNICO-DEEPAGENT-SUPERVISOR-COMPLETO.md) |
| Identificador de correlação e arquitetura de log | [README-TECNICO-ARQUITETURA-LOGGING-CORRELATION-ID.md](README-TECNICO-ARQUITETURA-LOGGING-CORRELATION-ID.md) |
| Configuração declarativa e ciclo de vida por cliente | [README-CONFIGURACAO-YAML.md](README-CONFIGURACAO-YAML.md) |
| Bancada de engines, censo de OCR e custos por página | `docs/.interno/.planos/agentic-rag-dnit/benchmark/04-ENGINES-COMPARATIVO.md` |
| Diagnóstico camada a camada por pergunta reprovada | `docs/.interno/.planos/agentic-rag-dnit/benchmark/03-DIAGNOSTICO-CAMADAS.md` |
| Histórico completo das 11 versões medidas | `docs/.interno/.planos/agentic-rag-dnit/benchmark/99-TABELA-COMPARATIVA-FINAL.md` |
| Mapa de decisão, alavancas, custos e projeções por faixa | `docs/.interno/.planos/agentic-rag-dnit/00-MAPA-DECISAO-90-PORCENTO.md` |
| A comparação arquitetural com o modelo "ler na hora", em 101 | `docs/.interno/.planos/agentic-rag-dnit/00-COMPARATIVO-CHATGPT-X-PIPELINE.md` |

---

## Anexo C — Em dez linhas, para quem só vai ler o fim

1. A diferença entre este pipeline e uma ferramenta genérica não é inteligência do modelo: é **ordem
   de leitura**. Uma lê depois da pergunta; a outra leu antes, sem saber a pergunta.
2. Ler antes é o que permite **escalar** para 564 manuais, **governar** por cliente e **auditar**
   cada resposta — e é requisito do produto, não preferência técnica.
3. O preço de ler antes é que **o que se perde na leitura, nenhuma busca recupera depois**. Por isso
   a maior parte do investimento foi na extração.
4. A extração é **multi-engine por classe medida**, com OCR, enriquecimento de fórmula, modo preciso
   de tabela e tetos de tempo folgados — cada escolha decidida por número, não por preferência.
5. Boa parte do que parecia limite era **defeito nosso, e foi corrigido com números grandes**:
   extração de 3,4 para 2.201,8 caracteres por página no pior caso; referências inadequadas de 31,2%
   para 3,7%; 8.703 trechos duplicados eliminados na origem.
6. O sistema **mede a si mesmo**, inclusive contra o próprio otimismo: descobriu que a etapa de
   reranking apagava a ordenação da busca, e que o próprio placar tem **±2 de incerteza**.
7. O que sobra tem **três formas**: conteúdo que não sobreviveu à extração, perguntas que pedem oito
   evidências de uma vez, e provar uma ausência. **Nenhuma é falha de implementação** — 108 de 108
   execuções terminaram sem erro.
8. O número que mede isso: **71% de acerto quando a pergunta pede uma coisa, 20% quando pede sete a
   nove**. É teto de arquitetura, não de esforço.
9. A saída não é trocar de arquitetura: é **deixar o sistema abrir o documento quando os trechos não
   bastarem** — e ele **já sabe baixar o PDF** e **já sabe qual página abrir**. Falta ligar, e um
   experimento de menos de uma hora decide se vale.
10. Nada aqui foi comparado diretamente com uma ferramenta genérica. Esta é uma explicação
    **arquitetural**, não um placar.


