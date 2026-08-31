# Playbook 101 — Como montar um pipeline de ingestão + RAG robusto, seguro, correto e assertivo

> **O que é este documento.** Um conjunto de regras práticas para quem vai montar (ou consertar) um
> pipeline que lê documentos, indexa e responde perguntas sobre eles. **Cada regra aqui foi paga
> caro**: todas nasceram de um defeito real encontrado, medido e corrigido numa campanha de
> engenharia sobre um acervo de **564 manuais técnicos e 21 mil páginas**, com **11 rodadas de
> medição** contra um gabarito validado.
>
> **O que este documento NÃO é.** Não é a descrição de como esta plataforma funciona — para isso,
> veja [README-INGESTAO-INDICE.md](README-INGESTAO-INDICE.md),
> [README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md)
> e [README-TECNICO-RAG-PIPELINE-COMPLETO.md](README-TECNICO-RAG-PIPELINE-COMPLETO.md). Aqui estão as
> **lições de engenharia**, não o manual do produto.
>
> **Nível 101:** escrito para quem está começando. Termos técnicos são explicados na primeira vez que
> aparecem, e há um glossário no fim.
>
> **Regra de leitura:** toda afirmação vem com o número que a provou. Onde não há número, está escrito
> que não há.

---

## Parte 0 — O conceito, em uma página

Um pipeline de ingestão + RAG é uma **cadeia de cinco elos**. Você entrega documentos numa ponta e
recebe respostas na outra.

1. **Extração** — abrir o arquivo e virar texto. Um PDF é um *desenho* de página, não um texto: tabela
   vira texto embaralhado, fórmula muitas vezes é **uma imagem** (não há texto para extrair), e página
   escaneada exige OCR (o programa que "lê" a imagem e chuta as letras).
2. **Recorte (*chunking*)** — partir o texto em pedaços de meia página, mais ou menos.
3. **Indexação** — guardar cada pedaço de dois jeitos: por **palavra** (achar o termo literal) e por
   **significado** (achar "bueiro" quando alguém escreveu "galeria").
4. **Busca (*retrieval*)** — dada a pergunta, escolher os ~20 melhores pedaços entre dezenas de
   milhares.
5. **Resposta** — o modelo de linguagem lê esses ~20 pedaços e escreve.

### A regra-mãe, da qual todas as outras derivam

> **O que o Elo 1 perde, os Elos 2 a 5 não recuperam. Não se busca o que não foi indexado.**

Parece óbvia escrita assim. Na prática, quase todo esforço de melhoria de RAG vai para os elos 4 e 5
— "buscar melhor", "reordenar melhor", "prompt melhor" —, e é lá que ele rende menos quando o problema
nasceu no elo 1. **Duas medições da campanha mostram o tamanho disso:**

- Aumentar os candidatos da busca de 100 para 500 (**cinco vezes mais**) recuperou **1 alvo de 8**.
- Uma variante mais sofisticada do prompt do agente **empatou, custando 41% a mais**.

Enquanto isso, consertar o elo 1 num único manual levou a extração de **3,4 caracteres por página**
para **2.201,8** — **646 vezes mais conteúdo**.

**Conclusão prática: invista no elo 1 primeiro, e desconfie de qualquer plano que comece pelo elo 4.**

---

## Parte 1 — Oito regras para a ingestão

### Regra 1 — Resultado cortado nunca é sucesso (*fail-closed* de truncagem)

**O conceito.** *Fail-closed* significa "na dúvida, falhe e avise" — o contrário de *fail-open*, que é
"na dúvida, siga em frente". Em ingestão, a dúvida mais comum é: a extração terminou ou foi
interrompida?

**O que aconteceu quando isso não existia.** A biblioteca de extração tinha um limite de tempo por
documento (120 segundos, um valor escolhido pelo projeto, não pela biblioteca). Documento grande
estourava o limite, a extração parava no meio, devolvia um status de "sucesso parcial" — **e o código
aceitava isso como válido e publicava**.

Os números do estrago, medidos documento a documento:

- **24 documentos** publicados truncados, somando **2.127 páginas**, das quais **1.888 páginas com
  erro de tempo**.
- O pior caso: um manual de 568 páginas com **3,4 caracteres de texto por página** — publicado como
  íntegro. Depois do conserto, **2.201,8 caracteres por página**.
- O sistema ainda reportava `páginas processadas: 34`, `páginas vazias: 0`, `páginas com falha: []`.
  **As três informações eram falsas.**

**Como fazer.** Extração interrompida **falha explicitamente**, na origem — no ponto onde a
interrupção acontece, não no ponto onde o documento é publicado. Consertar no fim da esteira criaria
registros órfãos e não tocaria na causa.

**A ordem importa, e essa lição custou uma discussão inteira.** Aumente o limite de tempo **antes** de
ligar o *fail-closed*. Ligar o *fail-closed* primeiro faria a próxima ingestão **reprovar em massa**
justamente os documentos que hoje passam (truncados, mas passam) — trocando um problema silencioso por
uma parada geral.

**Como testar.** Não basta testar que documento ruim reprova: teste também que **documento bom
continua passando**. Esse *controle negativo* é o que separa "fail-closed" de "quebrei tudo".

---

### Regra 2 — Contador cego vira sucesso cego

**O conceito.** Toda operação que remove, substitui, pula ou descarta alguma coisa precisa **registrar
quantas coisas foram afetadas**. Um registro que diz "concluído" sem dizer "quantos" não é
observabilidade — é uma opinião.

**O que aconteceu quando isso não existia.** A rotina que apaga a versão antiga de um documento
escrevia no registro `versões substituídas: removidas — status: concluído`. **Sem contar nada.** Um
defeito fazia essa rotina não apagar coisa alguma, e o registro de sucesso escondeu o problema por
duas janelas de ingestão inteiras. Quando finalmente foi medido: **8.703 fichas mortas em 77.441
(11,24% do acervo)**.

**Como fazer.** Todo evento de conclusão de operação com efeito carrega o **número do efeito**
(`pontos_afetados`, `documentos_pulados`, `páginas_perdidas`). E toda perda tem **um único canal
declarado** — um lugar só onde ela é registrada, com o motivo literal. Duas fontes de perda diferentes
escrevendo em dois lugares diferentes garantem que ninguém vai olhar os dois.

**Como testar.** O evento com contador **é o teste de vida da própria correção**: corrigir a remoção
sem o contador é corrigir às cegas — você não tem como provar que passou a remover.

---

### Regra 3 — "Versão do documento" tem que significar geração, não bytes

**O conceito.** Quando você reprocessa um documento, o sistema precisa saber que a nova extração
**substitui** a anterior. Isso normalmente se faz com uma chave de versão.

**O que aconteceu.** A chave de versão era calculada como `identidade do documento + hash dos bytes do
arquivo`. O comando de limpeza dizia: "apague tudo deste documento **exceto** a versão ativa". Só que
reprocessar o **mesmo arquivo** com **outra configuração de extração** produz bytes iguais → chave
igual → o "exceto" casava com tudo → **nada era apagado**.

Resultado: o acervo servia **2 a 3 gerações do mesmo documento ao mesmo tempo**. Nos 24 documentos
mais afetados, **79% dos pedaços indexados eram de uma geração anterior ainda viva e pesquisável**.

**A reviravolta que evitou um desastre.** A reação natural seria apagar as fichas "mortas". A medição
mostrou que isso teria sido catastrófico: **em 33 dos 57 documentos afetados, a geração viva era MENOR
que a morta**, e em 15 delas era menos de um terço. Num documento de 169 páginas: **11 pedaços vivos
contra 338 mortos**. A geração "morta" era a boa — a viva era a truncada.

> **Lição transferível: antes de limpar, meça o que vai ser apagado. "Antigo" não é sinônimo de
> "pior".**

**Como fazer.** A chave de versão precisa levar **os dois** componentes que definem uma geração: o
conteúdo do arquivo **e** a política de extração usada. E — importante — precisa ser calculada pelo
**mesmo código** que decide se o documento vai ser pulado. Duas fórmulas para a mesma pergunta divergem
em silêncio.

**Como testar.** Um *teste de arquitetura* — um teste que lê o próprio código-fonte e reprova se
alguém reintroduzir a fórmula antiga. Sem ele, a correção volta atrás no próximo refactor.

---

### Regra 4 — O *fingerprint* de política não é o portão que você acha que é

**O conceito.** *Fingerprint* (impressão digital) da política de extração é um número calculado a
partir das configurações. Se ele mudar, o documento precisa ser reprocessado. É o mecanismo que evita
reprocessar o acervo inteiro sem necessidade.

**O que a medição mostrou.** Ao preparar uma janela de reprocesso, o fingerprint vigente **divergia em
564 de 564 documentos** — ele sozinho mandaria o acervo inteiro (21 mil páginas) para reprocesso.
Quem realmente protegia os outros 484 documentos era **o filtro de listagem de arquivos**
(`file_patterns`): o que não é listado não é baixado, e o que não é baixado não é reprocessado.

**Duas consequências práticas, e as duas já causaram erro:**

1. **Uma configuração aplicada a um documento que não está no filtro de listagem é inerte — e
   silenciosa.** Você acha que ligou; não ligou.
2. **Onde a chave mora muda o escopo do reprocesso.** A mesma chave `table_mode: accurate` declarada
   no bloco global forçaria **564 documentos / 20.905 páginas**; declarada na configuração por
   documento, força **só os documentos nomeados**. A diferença entre as duas é uma janela de horas
   contra uma janela de dias.
3. **Calcule o fingerprint com as variáveis de ambiente já resolvidas.** Calcular sobre `${VARIAVEL}`
   cru produz um número diferente do real — foi exatamente a origem de um falso alarme de reprocesso
   total.

---

### Regra 5 — Gate de qualidade por classe de documento, com veredito derivado

**O conceito.** Um *gate de qualidade* é uma checagem automática, depois da ingestão, que responde:
"o que eu extraí é bom o suficiente?". Documento reprovado é marcado e reprocessado com outra engine.

**A descoberta que reenquadrou tudo.** Esse loop **já existia e já tinha rodado uma vez** no projeto —
mas à mão. Um relatório de qualidade de tabela apontou 56 documentos ruins, e **o único elo manual do
ciclo foi copiar 56 nomes de arquivo do log para o arquivo de configuração**. O trabalho não era
inventar o gate: era **fechar esse elo**.

**E a curadoria manual erra, com número.** Quando o acervo inteiro foi varrido: **14 dos 32 documentos
com célula de tabela colada estavam FORA** da lista curada; considerando toda tabela degradada,
**77 dos 145 documentos estavam fora contra 68 dentro**. Mais da metade dos casos reais não estava na
lista feita à mão.

**Duas decisões de desenho que valem copiar:**

- **Completude e suficiência são perguntas diferentes.** "Perdi o que eu deveria ter pego?" é uma;
  "o que eu peguei é bom o bastante?" é outra. Misturar as duas no mesmo marcador faz **todo documento
  virar "parcial"**, o marcador vira ruído, e o operador para de olhar.
- **O veredito é derivado, não materializado.** A lista de reprovados é uma **consulta** feita na hora,
  não uma coluna gravada no banco. Coluna gravada é mais uma verdade para manter sincronizada — e duas
  verdades divergem.

**⚠️ O limite honesto, que precisa ser dito.** Não existe hoje detector de *colapso* de extração. O
único corte é binário: "zero caractere" ou "íntegro". **Entre os dois não há nada** — e é exatamente
nessa faixa que moram os documentos truncados. Um detector de "muito menos texto do que o esperado
para esta quantidade de páginas" é a lacuna mais valiosa que este playbook consegue apontar.

**E há um detalhe que engana:** onde o gate lê importa. A tabela de execução guarda tudo; a tabela do
documento vivo passa por uma lista de campos permitidos e **perde** justamente `páginas processadas`,
`páginas vazias` e `problemas de extração`. **Um gate que reavalia lendo a tabela errada julga com
metade dos fatos.**

---

### Regra 6 — Multi-engine: escolha pela **taxa de escalada**, não pela "melhor engine"

**O conceito.** *Engine* é o programa que extrai o texto do PDF. Existem várias, e elas são boas em
coisas diferentes. A pergunta errada é "qual é a melhor?". A pergunta certa é: **"qual minimiza o custo
total = primeira passada + reprocessos?"**

**Os números que decidem.** Taxa de escalada = quantos documentos essa engine erra a ponto de exigir
uma segunda passada:

| Engine | Taxa de escalada | Observação |
|---|---|---|
| `pymupdf4llm` | **~1,2%** (6 de 508) | **zero colapsos** nos 92 documentos que dependem de OCR |
| `docling` | **~21%** (12 de 56) | extrai tabela e fórmula muito melhor |
| `opendataloader` | **59% das páginas** | devolve 19 caracteres por página **com código de saída 0** |

**Por que a mais rápida não é automaticamente a melhor, e vice-versa.** A engine com melhor extração
de tabela (`docling` com modo preciso) custa **11,7 segundos por página**; outra entrega **a mesma
célula íntegra a 0,09 segundos por página** — **130 vezes mais barato**. Para a classe inteira de
documentos com tabela degradada, a diferença é **41 minutos contra 17,5 horas**.

> **O custo da escalada muda em duas ordens de grandeza conforme a engine escolhida. É isso que torna
> a automação viável ou inviável.**

**A regra de ouro do multi-engine: agregar, nunca substituir.** Cada engine entra para a classe em que
ela ganha, e a decisão é medida por classe, não escolhida por preferência. E há trocas explícitas:
migrar um documento para `docling` **apaga a numeração de página impressa** — medido em **4 de 4
documentos**. Ganhar tabela custou perder proveniência.

**Exemplo real de configuração** (a estrutura vem do repositório; os comentários explicam campo a
campo):

```yaml
processing:
  parsing:
    # Cada entrada = uma engine + uma configuração + a lista de documentos que a recebem.
    # Documento não listado usa a engine padrão do projeto.
    engine_overrides:
      # ENTRADA 1 — documentos com tabela ruim ou OCR ilegível.
      - engine: docling
        config:
          ocr: true                          # liga o reconhecimento de imagem
          tables: true                       # liga a extração de tabela
          structured_chunking: true          # recorta respeitando a estrutura do documento
          skip_scanned_pdf: false            # NÃO pule PDF escaneado (é justamente o caso)
          document_timeout_seconds: 7200.0   # teto por documento — folgado de propósito
          subprocess_timeout_seconds: 14401.0# teto do processo externo: SEMPRE maior que o do documento
        documents:
          - "manual_com_tabela_ruim.pdf"

      # ENTRADA 2 — um documento que precisa de política DIFERENTE.
      # Entrada separada porque, na entrada acima, ele herdaria a config errada.
      - engine: docling
        config:
          ocr: true
          tables: true
          structured_chunking: true
          skip_scanned_pdf: false
          document_timeout_seconds: 36000.0
          subprocess_timeout_seconds: 43200.0
          table_mode: accurate               # modo preciso de tabela: separa linhas coladas
          # do_formula_enrichment ausente DE PROPÓSITO: mede 271 s/página e explodiria a janela
        documents:
          - "manual_com_tabela_critica.pdf"
```

**Três armadilhas de configuração que já custaram tempo:**

1. **O mesmo documento em duas entradas é proibido** — seriam duas verdades sobre a mesma decisão.
   Mesma engine e mesma configuração ⇒ **uma entrada só**.
2. **O teto do processo externo tem que ser maior que o teto do documento.** Se o processo morrer
   antes, você perde o resultado e o motivo.
3. **Etapas de enriquecimento podem rodar FORA do cronômetro da biblioteca.** O `document_timeout` do
   `docling` não cobre o enriquecimento — ele acontece depois. Um teto que parece suficiente pode não
   ser.

**E teto folgado não custa nada.** Um manual que gastou 5.597 segundos de um teto de 43.200 não
desperdiçou nada: terminou antes. Já um teto curto custou **5 horas de processamento reprovadas no
fim**.

---

### Regra 7 — Engine cuja falha não é contável não pode ser padrão

Esta é a regra menos óbvia e a mais importante da Parte 1.

**O conceito.** Um gate de qualidade só é tão bom quanto a **honestidade da engine que ele julga**. Se
a engine falha em silêncio — devolvendo pouco texto com código de sucesso —, o gate não tem o que
medir.

**Os casos medidos:**

- Uma engine devolve **19 caracteres por página com código de saída 0**. Não é erro: é sucesso vazio.
- Um modo automático de triagem, que deveria detectar página escaneada e acionar o processamento
  pesado, **devolveu o mesmo texto ruim sem chamar o processamento nenhuma vez, com código 0**. Forçado
  o modo completo, o mesmo documento foi de **18 para 890 caracteres por página**. **O defeito era a
  triagem, não o processamento.**
- Um auditor de "segunda opinião" tinha um comportamento padrão que **substitui o caractere inválido
  por espaço** — apagando exatamente o marcador que as outras engines usam para dizer "aqui falhou".
  O auditor devolvia texto limpo sobre extração suja, e a comparação ficava viesada a favor dele.

**Como fazer.**

- Escolha como padrão a engine **cuja falha é contável**, mesmo que ela não seja a melhor em nenhuma
  categoria isolada.
- Um marcador universal de falha de extração é o caractere de substituição Unicode (`U+FFFD`, o
  losango com interrogação). Ele é **independente de engine**: qualquer uma que o produza está
  dizendo "não consegui ler isto". Nunca configure nada que o apague.
- Desconfie de "modo automático" e de "fallback": um deles inverte um erro honesto ("não consegui,
  saída 1, nenhum arquivo") em um sucesso mentiroso ("documento vazio, saída 0").

**E não confie na nota de confiança da própria engine sem verificar como ela é calculada.** Uma delas
declara a qualidade do documento como **média das páginas**, não como a pior. Num manual de 568
páginas com 30 destruídas, a média dilui o defeito até sumir — **usando a nota do documento, o pior
caso do acervo seria aprovado**. Avalie **por página e por atributo**.

---

### Regra 8 — Guarde as duas numerações de página

**O conceito.** Todo documento técnico tem duas numerações: a do **arquivo** (que conta desde a capa) e
a **impressa no rodapé** (que o leitor procura). Elas diferem porque capa, ficha catalográfica e
sumário criam uma defasagem — tipicamente de 1 a 4 páginas, e num caso medido, **24**.

**O que acontece se você guardar só uma.** A resposta acerta o valor, a norma e o critério, e **erra o
endereço**. Foi a maior fonte de perda de nota da campanha: **31,2% das execuções com referência
inadequada**. Depois de passar a citar **as duas** — no formato `p. 256 (arquivo p. 259)` —, caiu para
**3,7%**, e a citação da página impressa foi de **0 execuções em 32** para **52 em 54 (96,3%)**.

**Como fazer.**

- Derive a numeração impressa **do texto já indexado**, sem reprocessar nada: procure o marcador de
  página e o número no rodapé, e meça a diferença. No acervo, isso levou **11 minutos** para 77 mil
  pedaços.
- **Recuse quando não tiver confiança.** A regra usada: alta confiança exige ≥10 pares e ≥95% no mesmo
  deslocamento; média, ≥3 pares e ≥90%. Abaixo disso, **não chute** — grave "indisponível". Resultado:
  **85 documentos aplicáveis de 558 (15,2%)**, o que parece pouco até você ver que são **61,1% dos
  pedaços** e **63,2% das páginas** — porque os manuais grandes, que é onde as perguntas moram, quase
  todos têm rodapé (**76,5% dos documentos com mais de 200 páginas**).
- **Não invente uma segunda regra de casamento sem medir se ela recupera alguém.** A hipótese "o
  rodapé está no fim do bloco" foi testada: **resgataria 0 documentos**, porque só **4,1% dos blocos**
  terminam em número puro.

**⚠️ E a armadilha que quase apagou o melhor dado do acervo.** O documento **mais bem medido** —
535 pares, deslocamento consistente em 100% — só tinha rodapé **por acidente**: o limite de tempo de
120 segundos cortava a esteira de processamento e o texto nativo do PDF (com rodapé) era entregue no
lugar. **Reprocessá-lo "direito" apagaria os 535 pares.** Antes de migrar um documento de engine,
confira o que ele carrega de bom pelos motivos errados.

**E uma última, que é fácil de esquecer:** verifique que os campos novos **sobrevivem à gravação**.
Nesta campanha, o carimbo da página impressa era calculado corretamente e **descartado por uma lista
de campos permitidos, antes de gravar, em silêncio** — a ingestão seguia verde. O acervo só tinha o
campo porque uma correção posterior o escreveu direto. **Qualquer reingestão o apagaria de novo.**
Teste de contrato que falha se o campo sumir: obrigatório.

---

## Parte 2 — Quatro regras para a consulta

### Regra 9 — O que reordena pode estar destruindo a ordenação

**O conceito.** Depois de achar candidatos, é comum ter uma etapa de *reranking*: reordenar por
relevância com um modelo mais caro e mais preciso. A intuição é que isso só melhora.

**A medição diz o contrário.** Medindo a dispersão dos pedaços entregues — a diferença entre a maior e
a menor pontuação, em porcentagem:

| Configuração | Buscas medidas | Dispersão (mediana) | Buscas com tudo empatado (&lt;15%) |
|---|--:|--:|--:|
| Com reranking | 281 | **1,94%** | **281 (100%)** |
| Sem reranking | 260 | **56,38%** | **1 (0,4%)** |

A busca **já separava bem** (56% de dispersão). A etapa seguinte **achatou para 2%**. Não é que o
reranker deixe de criar ordenação — ele **apaga uma ordenação informativa que já existia**.

**Por que isso importa mais que qualquer placar.** Quando os 20 pedaços entregues estão empatados
dentro de 2%, **qual deles o modelo prioriza vira sorteio**. Uma pergunta feita duas vezes na mesma
hora, no mesmo acervo, achou o quadro certo às 02:51 e o errado às 03:23.

**Como fazer.** **Meça a dispersão do lote entregue** — é uma métrica barata, direta e que quase
ninguém coleta. Se ela estiver perto de zero, sua etapa de reordenação está destruindo informação,
não criando.

**⚠️ E cuidado com o experimento.** Desligar o reranking sem ajustar o tamanho da colheita produz um
resultado **pior** e leva à conclusão errada. Aconteceu: com o reranking ligado, quem manda na entrega
é um parâmetro; desligado, é outro. Um A/B anterior mediu "sem reranking" com a colheita colapsada e
concluiu que o reranking ajudava. **A comparação correta exigiu uma correção de código antes.**

---

### Regra 10 — Buscar mais fundo é a alavanca mais fraca

Aumentar o número de candidatos parece o ajuste mais fácil, e é o que rende menos.

**Medido:** subir os candidatos de 100 para 500 — **cinco vezes mais** — em 8 consultas críticas
comprou **1 alvo de 8**. E a hipótese de que o problema fosse de fragmentação do armazenamento foi
**refutada na infraestrutura física**: a coleção tem um fragmento só, sem replicação.

**Como fazer.** Antes de aumentar a colheita, verifique a **diversidade** do que já vem. Numa pergunta
medida, **7 dos 20 lugares vieram do mesmo documento**, e outros 3 eram fichas duplicadas — sobrava
metade do lote para todo o resto do acervo. Quando a pergunta exige comparar duas normas, um lote
concentrado é a causa direta da falha, e **buscar mais fundo do mesmo documento não ajuda**.

---

### Regra 11 — Perguntas que pedem várias coisas têm um teto natural

**O conceito.** Conte quantas exigências distintas cada pergunta faz. Uma pergunta de "alvo único"
pede um valor. Uma pergunta de "síntese" pede sete a nove coisas: comparar duas normas, apresentar
critérios concorrentes sem escolher, citar item e página, recomendar conferir o desenho.

**O número que resume o problema inteiro:**

| Tipo de pergunta | Taxa de acerto |
|---|--:|
| Alvo único (1 exigência) | **71%** |
| Síntese (7 a 9 exigências) | **20%** |

**Por quê.** Para acertar 8 exigências com 20 pedaços, as 8 evidências precisam estar entre os 20
**ao mesmo tempo**. Nada garante isso, e nenhuma reordenação garante.

**Como usar isso.** Classifique suas perguntas por número de exigências **antes** de prometer uma taxa
de acerto. Um benchmark dominado por perguntas de síntese e um dominado por alvo único medem coisas
diferentes, e o mesmo sistema tira notas muito diferentes nos dois.

**E há um caso que nenhum ajuste de busca resolve: provar uma ausência.** Quando a resposta correta é
"esta norma **não** traz esse valor nestas páginas", um sistema que vê 20 pedaços não pode responder —
ele só sabe o que veio, nunca o que não veio. **Só ler o intervalo inteiro resolve.**

---

### Regra 12 — Preveja a escalada: deixe o sistema abrir o documento

Esta é a conclusão prática das regras 10, 11 e da regra-mãe.

Quando o conteúdo não sobreviveu à extração, ou quando a resposta exige varrer um intervalo, ou provar
uma ausência, **o caminho não é buscar melhor — é ler o documento**.

**E o sinal para acionar isso já existe no seu sistema, de graça.** Quando o modelo responde *"não foi
possível decodificar a fórmula"* ou *"o trecho recuperado estava truncado"*, ele está dizendo
literalmente *"os pedaços não bastam aqui"*. Trate isso como **gatilho de escalada**, não como
resposta final.

**O que a escalada exige:** uma forma de buscar o documento original (nesta plataforma, o endpoint
`POST /rag/documents/content`, que baixa o binário ao vivo da origem), a proveniência de página para
saber **qual** intervalo abrir (Regra 8), e um orçamento de contexto — ler 10 páginas é ordem de
grandeza mais caro que ler 20 pedaços, então a escalada **tem que ser escalada**, acionada por sinal e
nunca por padrão.

> **Nesta plataforma, as duas primeiras peças existem e a ligação com o agente não.** Isso está
> registrado como projeto, com medição pendente — não como capacidade entregue.

---

## Parte 3 — Cinco regras de método (como saber se você melhorou)

### Regra 13 — Meça a incerteza do instrumento ANTES de comparar versões

**O conceito.** Rodar a mesma pergunta duas vezes na mesma configuração pode dar notas diferentes.
Antes de dizer "a versão B é melhor que a A", você precisa saber **de quanto é o ruído**.

**O que aconteceu por não fazer isso.** Com N=3 (três execuções por pergunta) em todas as perguntas,
**10 de 18 deram nota diferente entre execuções idênticas** — uma variou três degraus (`A`, `C`, `D`)
na mesma configuração. A incerteza do placar é de **±2 pontos**.

Consequência declarada no próprio diário do projeto: *"7/17, 8/17, 9/17 e 10/17 nunca foram
distinguíveis. A campanha vinha comparando versões dentro do próprio ruído."*

**Como fazer.**

- Meça a repetibilidade **primeiro**, antes de qualquer comparação.
- Use a **mediana** de N execuções, e publique **as notas cruas junto** — `C (ACC)` conta uma história
  que `C` esconde.
- **Delta menor que o ruído não decide nada.** Mudar configuração de produção com base nele é sorteio,
  não engenharia.
- Quando um resultado tiver direção consistente mas sem significância estatística, **diga isso**. Num
  teste pareado da campanha, a direção era clara (7 melhores contra 2 piores) e o valor-p era **0,090**
  — registrado como "direção consistente, sem significância", em vez de vendido como vitória.

---

### Regra 14 — Prove na fonte física, com dois discriminantes independentes

**O conceito.** Perguntas do tipo "existe?", "está vazio?", "quantos?" não se respondem por código,
documentação, log ou uma única tabela. Respondem-se **tocando o armazenamento real**.

**Como o censo do acervo foi validado, e vale copiar o método:**

- **Dois discriminantes independentes** para o mesmo fato: um pelo carimbo de tempo, outro pela
  identidade física do registro. Deram **8.758** e **8.703**. A diferença foi explicada, e publicou-se
  o número exato: **8.703**.
- **Um teste de controle contra um resultado anterior conhecido:** o método precisou reproduzir três
  números já medidos (9.232 / 1.929 / 7.303) com **zero divergência documento a documento** antes de
  ser aceito.
- **Uma prova de segurança antes de qualquer limpeza:** verificar que a projeção é subconjunto estrito
  do armazenamento — **0 de 68.738 registros da projeção ausentes da coleção**.

**E a armadilha que quase invalidou tudo:** o modelo de vetorização declarado no arquivo de
configuração **não era o realmente usado**. Quem confiou na configuração mediu similaridade `−0,02`;
quem descobriu o modelo real mediu `0,999935`. **A configuração declara a intenção; o runtime diz o
fato.**

---

### Regra 15 — Volume não é qualidade: abra o conteúdo

Duas vezes na campanha, a média mentiu.

- Uma medição de ganho de extração mostrava **dois documentos "piorando"**. Abrindo o conteúdo: um
  deles **perdeu caracteres e ganhou os números decimais** que antes faltavam. A conclusão se inverteu.
- Boa parte do texto que existia num documento não era extração: era **descrição de figura gerada por
  IA** — **78,4% dos caracteres**. A perda real de texto era de **236 vezes**, e estava mascarada por
  um enchimento que parecia conteúdo.

**Como fazer.** Meça, mas **abra os casos extremos** antes de concluir. E separe as fontes de texto na
contagem: extração real e enriquecimento gerado não podem entrar no mesmo número.

**Um padrão útil descoberto assim:** o resultado da extração converge para a faixa de **1.100 a 3.900
caracteres por página**, venha de onde vier. **O ganho é inversamente proporcional ao estado
inicial** — o que permite priorizar uma janela de reprocesso pelo pior primeiro.

---

### Regra 16 — Código correto em disco não é código correto em execução

Três armadilhas medidas, todas capazes de invalidar um experimento inteiro:

1. **`git status` limpo não é prova de reversão.** Aconteceu **duas vezes**: um processo de
   auto-commit capturou o arquivo experimental no histórico, e o comando de reversão devolveu
   "0 arquivos atualizados" com o experimento **ativo**. A profilaxia que funcionou: registrar o
   hash do arquivo original **antes** de editar, e provar a reversão por múltiplas vias.
2. **A data de modificação do arquivo não prova qual código está rodando.** Num experimento, os
   arquivos tinham data posterior ao início do servidor — pela data, a conclusão seria "o servidor
   roda código velho". Só a verificação em tempo de execução desfez isso.
3. **Correção feita não é correção ativa.** Três correções críticas exigiam reiniciar o processo para
   valer. Disparar a janela sem reiniciar teria repetido exatamente o defeito corrigido.

**Como fazer.** Antes de gastar um recurso caro (uma janela de horas, uma rodada de centenas de
execuções), rode uma **sonda discriminante**: um conjunto pequeno de verificações **em tempo de
execução** que provam que você está medindo o que pensa. Na campanha foram quatro (alvo físico
correto, configuração correta, etapa esperada realmente executada, correção viva no processo), com o
critério declarado por escrito: *"se qualquer uma vier diferente, a rodada é inválida"*.

---

### Regra 17 — Artefato de runtime nunca nasce dentro da pasta do projeto

**O conceito.** Em produção, o diretório da aplicação costuma ser **somente leitura**. Qualquer código
que grave arquivo temporário ali funciona na máquina do desenvolvedor e morre no servidor.

**Aconteceu duas vezes, na mesma família, e as duas só apareceram em produção:**

- Um arquivo de credencial gravado na pasta do projeto → erro de permissão → **a ingestão morreu em
  2,5 segundos, sem processar um único documento**.
- A materialização de logs na mesma pasta → erro 500 nas telas de acompanhamento. Sintoma que denuncia:
  em **7 dias, 9 registros de início e zero de fim**.

**Como fazer.** Um **helper canônico único** que resolve o diretório de artefatos de runtime para o
temporário do sistema (com permissões restritas e escrita atômica), e **sem alternativa de reserva** —
um *fallback* aqui esconde o problema em vez de resolvê-lo. Todo consumidor delega a ele; a regra tem
uma fonte só.

**E registre o corpo do erro.** Num dos casos, o log guardava só o código de status HTTP, e foi preciso
reproduzir a chamada à mão para descobrir a causa. Dentro de um bloco de tratamento de exceção,
registre a exceção completa.

---

## Parte 4 — Checklist de arranque

Se você está montando um pipeline do zero, esta é a ordem que a campanha provou:

1. **Meça a repetibilidade do seu benchmark** antes de medir qualquer versão (Regra 13).
2. **Classifique as perguntas** por número de exigências — alvo único × síntese (Regra 11).
3. **Classifique os documentos** por classe: texto nativo, escaneado, com tabela crítica, com fórmula.
   No acervo estudado: **18,1% dos documentos dependiam de OCR, e eles eram 59,1% das páginas**.
4. **Escolha a engine padrão pela taxa de escalada**, não pela melhor extração isolada (Regras 6 e 7).
5. **Ligue tetos de tempo folgados antes de ligar o fail-closed** (Regra 1, a ordem importa).
6. **Ligue o fail-closed de truncagem** e prove com controle negativo (Regra 1).
7. **Garanta contador de efeito em toda operação que remove ou pula** (Regra 2).
8. **Faça a chave de versão significar geração**, e proteja com teste de arquitetura (Regra 3).
9. **Derive e grave as duas numerações de página**, recusando o que não tem confiança (Regra 8).
10. **Meça a dispersão do lote entregue** pela busca (Regra 9).
11. **Faça um censo do acervo na fonte física**, com dois discriminantes (Regra 14).
12. **Só então** invista em reordenação, prompt e ajuste de busca.

---

## Parte 5 — Os erros mais caros, em uma linha cada

- **Extração truncada publicada como sucesso** — o defeito mais caro de todos. Um manual de 568 páginas
  com 3,4 caracteres por página, marcado como íntegro.
- **Limpeza que dizia "concluído" sem contar nada** — escondeu 8.703 registros mortos por duas janelas.
- **Apagar o "antigo" sem medir** — em 33 de 57 documentos, a versão antiga era a boa.
- **Guardar só a numeração do arquivo** — 31,2% das respostas com referência inadequada, tendo o valor
  técnico certo.
- **Confiar no *fingerprint* como portão** — quem protegia o acervo era o filtro de listagem.
- **Curadoria manual de engine por documento** — mais da metade dos casos reais ficou de fora.
- **Engine cuja falha não é contável** — 19 caracteres por página com código de sucesso.
- **Modo automático de triagem** — não detectou página escaneada e não chamou o processamento, com
  código 0.
- **Comparar versões dentro do ruído** — quatro placares diferentes que nunca foram distinguíveis.
- **Confiar em `git status`, na data do arquivo e no modelo declarado no YAML** — os três mentiram.
- **Gravar artefato de runtime na pasta do projeto** — matou a ingestão em produção em 2,5 segundos.
- **Cancelar sem matar o processo filho** — 3 órfãos segurando 14,7 GB e **metade da vazão da janela**.

---

## Parte 6 — Glossário 101

- **Chunk (pedaço)** — trecho de meia página, mais ou menos, em que o documento é recortado para ser
  indexado.
- **Embedding (vetorização)** — transformar texto em uma lista de números que representa o
  *significado*, permitindo achar "bueiro" quando alguém escreveu "galeria".
- **Busca híbrida** — combinar busca por palavra literal com busca por significado.
- **Reranking** — etapa que reordena os candidatos da busca com um modelo mais caro e mais preciso.
- **OCR** — programa que "lê" uma imagem de texto e devolve as letras. Erra, e erra em silêncio.
- **Fail-closed** — na dúvida, falhe e avise. O contrário de *fail-open*, que segue em frente.
- **Fingerprint de política** — número calculado a partir das configurações de extração; se ele muda,
  o documento precisa ser reprocessado.
- **Supersessão** — apagar a versão anterior de um documento quando uma nova é indexada.
- **Gate de qualidade** — checagem automática, depois da ingestão, que decide se a extração é boa o
  bastante.
- **Taxa de escalada** — porcentagem de documentos que uma engine erra a ponto de exigir reprocesso
  por outra.
- **Sonda discriminante** — verificação pequena, em tempo de execução, que prova que o experimento está
  medindo o que você acha que está.
- **Controle negativo** — teste que prova que a correção **não** quebra o caso bom.
- **Página física × página impressa** — a numeração do arquivo PDF (conta desde a capa) × a numerada
  no rodapé (que o leitor procura).
