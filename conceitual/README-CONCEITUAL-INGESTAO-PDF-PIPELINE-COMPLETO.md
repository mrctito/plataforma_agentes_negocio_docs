# Manual técnico, executivo, comercial e estratégico: Pipeline de Ingestão de PDF

## 1. O que é esta feature

O pipeline de ingestão de PDF é a capacidade da plataforma de transformar um PDF bruto em acervo consultável com rastreabilidade operacional. Ele não existe só para “tirar texto do arquivo”. Ele existe para decidir quando confiar no texto nativo, quando ativar OCR, quando trocar de engine, quando enriquecer com leitura visual, como preservar sinais de página e como devolver chunks úteis para busca, auditoria e RAG.

Na prática, este pipeline é uma cadeia de decisão. O código mostra que o PDF não é tratado como arquivo simples. Ele passa por um runtime especializado com bootstrap próprio, builder de engine, pipeline de extração, pipeline textual, fluxo rico, trilha multimodal, manifesto operacional e integração com a esteira comum de persistência e indexação.

## 2. Que problema ela resolve

PDF é o tipo documental mais traiçoeiro do produto. Dois arquivos com a mesma extensão podem representar problemas completamente diferentes.

- Um PDF pode ser nascido digital e ter texto limpo.
- Um PDF pode ser escaneado e não ter texto útil algum.
- Um PDF pode ter texto parcial, tabelas, imagens, anexos e páginas com qualidade desigual.
- Um PDF pode parecer textual, mas carregar texto corrompido ou pouco denso.
- Um PDF pode exigir leitura visual complementar para imagens relevantes, diagramas e trechos não capturados pela trilha textual.

Sem um pipeline especializado, o produto sofreria com três perdas graves.

- Perda de qualidade do acervo, porque PDF ruim entraria como texto quebrado.
- Perda de governança, porque OCR, parsing e chunking aconteceriam de forma opaca.
- Perda comercial, porque clientes corporativos medem a maturidade da plataforma pela capacidade real de lidar com PDF complexo.

## 2-A. Capítulo introdutório — por que a ingestão "ingênua" não funciona (e por que isto NÃO é overengineering)

> Este capítulo existe porque 99% das pessoas que olham este pipeline pela primeira vez pensam:
> *"isso tudo é trabalho desnecessário; dá pra fazer com 10 linhas de LangChain ou 3 prompts numa
> ferramenta de IA"*. A intenção aqui é mostrar, com profundidade técnica e exemplos concretos de
> documentos como os do **DNIT** (normas, especificações de serviço — ES, métodos de ensaio — ME,
> instruções, projetos de engenharia rodoviária), **exatamente onde** essa abordagem rasa quebra,
> **listando todos os problemas**, e depois fazer o **cara-crachá**: cada etapa que implementamos aqui
> contra a dor real que ela resolve. Quem ler até o fim entende que cada estágio não é enfeite — é a
> resposta a uma falha que aparece **no primeiro documento técnico de verdade**.

### 2-A.1. O que é a "ingestão ingênua" (o pipeline de 10 linhas)

O pipeline ingênuo — o que está em quase todo tutorial e em quase todo serviço online de "RAG em 5
minutos" — tem exatamente quatro passos:

1. **Carregar o PDF** (um loader genérico, ex.: `PyPDFLoader`).
2. **Extrair o texto** (uma única engine, leitura linear página a página).
3. **Quebrar em pedaços** (chunking por tamanho fixo, ex.: 1000 caracteres com 200 de sobreposição).
4. **Gerar embeddings e jogar num banco vetorial**; na pergunta, busca por similaridade de cosseno e
   manda os top-k pedaços para o LLM responder.

Isso **funciona numa demo**: um PDF nascido digital, de uma coluna, texto limpo, sem tabela, sem
imagem, pergunta simples. O problema é que **documento técnico real não é assim** — e o pior é que a
ingestão ingênua **não falha com erro**: ela falha **em silêncio**. O lixo entra, a busca traz o pedaço
errado, e o LLM responde com confiança uma informação **incorreta**. Em engenharia, norma e contrato,
**resposta errada com aparência de certa é o pior resultado possível** — e é o resultado padrão da
abordagem rasa.

### 2-A.2. Por que documentos técnicos (DNIT) são tão difíceis — todos os problemas

Os problemas se acumulam em **seis camadas**. Nenhuma sozinha, mas todas juntas, é o que mata o pipeline
ingênuo.

#### (1) Problemas do arquivo — o PDF como "container", não como documento

- **Nascido digital × escaneado × híbrido.** O mesmo `.pdf` pode ter texto nativo, ser **imagem pura de
  papel escaneado** (zero texto extraível), ou ser **híbrido** (corpo digital + anexos, carimbos,
  ARTs e assinaturas escaneados). Um loader único trata os três igual e perde o escaneado inteiro.
- **Camada de texto "fantasma".** Muitos PDFs trazem uma camada de OCR antiga e ruim embutida. O
  extrator lê esse lixo como se fosse texto bom — parece que extraiu, mas o conteúdo é gibberish.
- **Ordem de leitura quebrada.** O PDF **não garante** ordem lógica de leitura. A extração linear
  embaralha blocos; em página de **duas colunas** (comum em normas), mistura a coluna da esquerda com a
  da direita, gerando frases Frankenstein.
- **Cabeçalho, rodapé e numeração repetidos** em todas as páginas viram ruído que se infiltra no meio
  das frases e polui os chunks ("...DNIT 031/2006-ES 12 o teor de ligante...").
- **Hifenização de quebra de linha.** Palavras cortadas no fim da linha ("compacta-\nção") viram dois
  tokens quebrados; sem correção, "compactação" some do índice.
- **Fontes embutidas sem mapeamento Unicode.** O texto extraído vira símbolos errados (caracteres
  trocados, acentos perdidos), corrompendo termos técnicos.
- **Rotação de página.** Tabelões e plantas entram em paisagem dentro de um documento retrato; a
  extração ingênua lê torto ou não lê.
- **Watermarks, carimbos e números de processo** sobrepostos ao conteúdo confundem tanto OCR quanto
  parsing.

#### (2) Problemas de estrutura técnica — o que esses documentos de fato carregam

- **Tabelas — o problema número um.** Especificação DNIT é feita de tabela: faixas granulométricas,
  traços, limites de ensaio, tolerâncias. Essas tabelas têm **células mescladas, cabeçalhos
  hierárquicos e linhas com múltiplos valores**. A extração linear **achata** a tabela: o número
  "0,5" perde a amarração "limite superior de % passante na peneira X para a faixa C". Uma tabela
  achatada vira uma sopa de números **semanticamente inútil** — e o embedding dela não responde nada.
- **Numeração normativa hierárquica.** Cláusulas "5.3.2.1", "6.2", subitens "a) b) c)". O sentido de um
  parágrafo **depende do número da cláusula** a que pertence. Cortar o texto longe do seu número perde a
  âncora normativa.
- **Referências cruzadas.** Uma norma remete a outra ("conforme item 7.4", "ver Tabela 3", "atender à
  DNIT 005/2003-TER"). A resposta correta muitas vezes exige **seguir a referência**; um pedaço isolado
  não tem essa ligação.
- **Unidades, grandezas e faixas.** "MPa", "kg/m³", "%", tolerâncias "± 0,5", intervalos "entre 4% e
  7%". Pergunta factual de engenharia é quase sempre **numérica e condicional** ("qual o teor de ligante
  da faixa C?"). Similaridade de cosseno **não entende faixa nem "está dentro de"**.
- **Fórmulas e equações.** Índices, símbolos, frações. A extração de texto destrói a notação; o
  embedding não a representa.
- **Figuras técnicas: seções transversais, perfis, ábacos, plantas, detalhes.** Em projeto de
  engenharia, **a informação está no desenho**, não no texto corrido. Extrair só texto perde 100% dessa
  evidência.
- **Notas de rodapé, legendas e observações** que **modificam** o sentido da tabela/figura, mas ficam
  fisicamente distantes dela na página.
- **Listas de requisitos.** "Deverá atender: a)...; b)...; c)...". Cortar no meio perde requisitos —
  numa norma, perder um requisito é perder a conformidade.
- **Terminologia e siglas densas.** CBUQ, ISC/CBR, granulometria, ES/ME/TER, número da norma. Modelos de
  embedding genéricos têm **baixa densidade semântica** nesses termos, e o número da norma é um
  **identificador exato** que vetor nenhum casa bem.
- **Documentos longos e versionados.** Normas com dezenas de páginas; a resposta está **espalhada**. E a
  **versão importa** (a revisão de 2006 difere da anterior) — misturar versões gera resposta errada.

#### (3) Por que a EXTRAÇÃO DE TEXTO ingênua falha (o passo "extrair texto")

- Uma **única engine** não cobre digital + escaneado + tabela ao mesmo tempo. A que é boa em texto
  nativo é ruim em tabela; a que é boa em layout pode ser cara ou indisponível.
- **Sem OCR**, o escaneado entra vazio. **Com OCR em tudo**, você degrada o PDF digital bom (rasteriza
  texto perfeito) e **explode o custo**.
- **Sem detecção de tabela**, toda tabela vira texto corrido sem relação linha×coluna.
- **Sem preservar página e estrutura**, você perde a âncora de auditoria — não consegue dizer "isto está
  na página 12, item 5.3", que é **obrigatório** em contexto normativo.

#### (4) Por que o CHUNKING ingênuo falha (o passo "quebrar em pedaços")

- **Tamanho fixo corta no lugar errado:** parte uma tabela no meio, separa a cláusula do seu número,
  separa o requisito da sua condição, e gruda o fim de uma seção com o começo de outra **não
  relacionada**. Cada um desses cortes gera um chunk que **mente** sobre o conteúdo.
- **Sem metadata** (página, seção, tipo de conteúdo), o pedaço **não pode ser filtrado nem citado**: a
  resposta fica sem fonte auditável.
- **Sem estratégia por tipo**, tabela, texto corrido e lista são cortados do mesmo jeito — o pior jeito
  para os três.

#### (5) Por que EMBEDDING + busca vetorial ingênua falha (o passo "embeddings + banco vetorial")

- **Embedding genérico representa mal** número, faixa, unidade e sigla técnica — justamente o que a
  pergunta de engenharia precisa.
- **Cosseno traz "parecido no assunto", não "responde à pergunta".** Para "qual o teor de ligante da
  faixa C?", o trecho certo pode **não** ser o mais próximo da pergunta em linguagem natural.
- **Gap de vocabulário.** O usuário pergunta com as palavras dele; a norma usa o termo técnico. Os
  vetores não casam. E há identificadores **exatos** (número da norma, código de ensaio, sigla) que
  **só** correspondência de palavra-chave (BM25/FTS) acerta — busca puramente vetorial erra.
- **Sem rerank**, o top-k vem com ruído e o trecho certo fica em 8º lugar — **fora** da janela de
  contexto que vai pro LLM.
- **Só vetor** perde o casamento exato; **só palavra-chave** perde a paráfrase. Sem **busca híbrida**,
  você perde um dos dois sempre.

#### (6) Por que a RESPOSTA (RAG) ingênua falha mesmo com retrieval razoável

- **Sem reescrita/expansão da pergunta**, pergunta curta ou ambígua recupera mal.
- **Sem roteamento**, toda pergunta é tratada igual (factual × analítica × tabular pedem caminhos
  diferentes).
- **Sem citação/âncora**, a resposta **não é auditável** — inaceitável em norma, contrato e laudo.
- **Contexto poluído** por chunks ruins faz o LLM **alucinar** ou **misturar normas/versões**.
- **Sem ACL/escopo pós-retrieval**, mistura documentos de projetos ou clientes diferentes — vazamento
  de informação.

### 2-A.3. Cara-crachá — cada etapa do NOSSO pipeline × a dor que ela resolve

Aqui está o ponto central: **cada estágio deste projeto existe porque um dos problemas acima é real e
aparece cedo**. A coluna "dor que resolve" remete às camadas da seção 2-A.2.

#### Lado da INGESTÃO (este manual)

| Etapa implementada | Dor que resolve (seção 2-A.2) | Como e por quê |
| --- | --- | --- |
| **OCR document-level seletivo** (heurística antes de rasterizar) — §8.2, `pdf_document_ocr_service` | (1) escaneado/híbrido; texto fantasma | Detecta que o PDF "parece escaneado/vazio" e **só então** roda OCR no documento — recupera o escaneado **sem** degradar o digital bom nem pagar OCR à toa. |
| **Fila determinística de engines plug-and-play** (Docling, PyMuPDF4LLM, Unstructured, GMFT, OCRmyPDF…) — §8.2, `pdf_parsing_engine_resolver`, `deterministic_lego_pdf_parsing_engine` | (3) engine única não cobre tudo | Em vez de um parser só, uma **bancada ordenada por YAML**: a engine boa em layout/tabela entra quando faz sentido; a próxima só é acionada por **sinal objetivo** de falha/insuficiência. Sem hardcode, sem fallback implícito. |
| **Extração estruturada de tabelas** (engines de tabela na fila) — §8.2, `pdf_parsing_runtime_builder` | (2) tabelas achatadas | Preserva a relação **linha×coluna**, mantendo o número amarrado ao seu cabeçalho e à sua faixa — a tabela deixa de virar "sopa de números". |
| **Pós-processamento textual** (hifenização, ruído, cabeçalho/rodapé) — §8.3 | (1) hifenização, rodapé, ordem | Limpa **sem destruir** estrutura: junta palavras quebradas, tira o ruído repetido de página, melhora o texto para chunking e busca. |
| **OCR complementar** (só nas páginas que ficaram vazias) — §8.4 | (1)/(3) lacunas pontuais | Completa o que faltou **sem** sobrescrever o texto já bom — equilíbrio entre cobertura e custo/qualidade. |
| **Trilha multimodal** (imagens/figuras → OCR visual + descrição + chunk visual) — §8.5, `pdf_multimodal_application_service` | (2) figuras, seções, plantas, ábacos | Recupera a evidência que está **no desenho**, não no texto corrido — com `strict_mode` explícito para quando a leitura visual é obrigatória. |
| **Chunking por estratégia** (Strategy Pattern por tipo de conteúdo, com metadata de página/seção) — §8.6, `pdf_chunking_service` | (4) corte no lugar errado; falta de metadata | Não corta tabela/cláusula/requisito no meio e **carimba página, seção e tipo** em cada chunk — o pedaço fica **citável e filtrável**. |
| **Enriquecimento de domínio** (cadeia de processadores por prioridade) — §8.6 | (2)/(6) terminologia e filtro de negócio | Acrescenta metadata especializada que melhora relevância e permite filtrar por contexto de negócio no retrieval. |
| **Manifesto operacional + checkpoint + artefatos** — §8.7, §6.8 | auditabilidade e custo de reprocesso | Conta a história por etapa (engine usada, decisão de OCR, status multimodal), permite **retomar** e **auditar** — em vez de uma caixa-preta. |
| **Persistência + indexação na esteira comum** — §8.7, `DocumentIndexingExecutor.finalize` | fechar o ciclo | Liga o PDF processado ao acervo consultável com metadata canônica — o chunk vira **valor de produto**, não arquivo solto. |

#### Lado da RECUPERAÇÃO/RESPOSTA (RAG) — ver `README-CONCEITUAL-RAG-PIPELINE-COMPLETO.md`

| Etapa implementada | Dor que resolve (seção 2-A.2) | Como e por quê |
| --- | --- | --- |
| **Reescrita da consulta** (query rewrite) — RAG §9.4 | (6) pergunta curta/ambígua; (5) gap de vocabulário | Reescreve/expande a pergunta do usuário para o vocabulário do acervo antes de buscar. |
| **Análise + roteamento** (query analysis/router) — RAG §9.5 | (6) toda pergunta tratada igual | Distingue factual × analítica × tabular e manda cada uma pelo caminho certo. |
| **Self-query / multi-query** — RAG §7.8/§7.9 | (5) um ângulo de busca não basta | Recupera por **vários ângulos**/filtros, aumentando a chance de achar o trecho certo. |
| **Busca híbrida (semântica + BM25/FTS)** — RAG §7.6/§7.7, §9.6 | (5) cosseno perde o exato; keyword perde a paráfrase | Casa **paráfrase** (vetor) **e** identificador exato (número da norma, código de ensaio, sigla) ao mesmo tempo. |
| **Rerank pós-retrieval** — RAG §7.11, §9.7 | (5) trecho certo fora do top-k | Reordena os candidatos e **puxa o trecho certo para dentro** da janela de contexto. |
| **Cache semântico** — RAG §7.10 | custo e latência | Reaproveita respostas de perguntas semanticamente equivalentes. |
| **ACL pós-retrieval** — RAG §7.12 | (6) vazamento entre escopos | Garante que cada resposta só use documentos que aquele usuário/projeto pode ver. |
| **Metadata de domínio gerada na ingestão, usada no retrieval** — RAG §7.13 | (2)/(6) filtro e prioridade | Usa o que foi enriquecido na ingestão para filtrar e priorizar na busca. |
| **Geração final com citação/âncora** — RAG §9.8 | (6) resposta não auditável | Responde **citando a fonte** (documento, página, seção) — auditável, requisito de norma/contrato. |

### 2-A.4. Por que isto NÃO é overengineering

A sensação de "trabalho desnecessário" vem de medir o pipeline contra a **demo**, não contra o
**documento real**. Cada etapa acima **só existe porque o caminho ingênuo falha exatamente ali** — e
falha **calado**, devolvendo resposta errada com cara de certa. Tirar qualquer estágio reabre uma falha
concreta:

- Sem OCR seletivo → o edital escaneado entra **vazio** e o sistema "não acha" o que está no acervo.
- Sem fila de engines / tabelas → a faixa granulométrica vira sopa de números e a resposta numérica é
  **inventada**.
- Sem chunking com metadata → a resposta **não cita a fonte**, e ninguém aprova uma norma sem fonte.
- Sem híbrida + rerank → a pergunta com número de norma traz o documento **errado**.
- Sem ACL → um projeto **vê o documento de outro**.

Em uma frase: o pipeline raso é barato de escrever e **caríssimo de operar** — a conta vem em forma de
resposta errada, retrabalho de investigação e perda de confiança do cliente. O que parece
"overengineering" é, na verdade, a **diferença entre uma demo e um produto** que aguenta documento
técnico de verdade. O resto deste manual detalha, etapa por etapa, **como** cada um desses estágios foi
implementado.

## 3. Visão executiva

Para liderança, este pipeline importa porque ele protege a parte mais frágil da cadeia de valor: a entrada de conhecimento documental. Se a ingestão do PDF falha ou empobrece o conteúdo, todo o resto fica comprometido. O RAG responde pior, a auditoria perde explicabilidade, o suporte gasta mais tempo em investigação e o cliente percebe baixa confiabilidade.

O código confirma que a plataforma tenta reduzir esse risco com uma abordagem em camadas.

- Pré-processamento documental opcional para PDFs suspeitos.
- Parsing determinístico por fila ordenada de engines.
- OCR básico apenas quando o texto não basta.
- Pipeline multimodal com fallback textual explícito.
- Manifesto operacional por etapa para facilitar retomada e troubleshooting.

Em linguagem simples: a plataforma não trata PDF como upload passivo. Ela trata PDF como um processo de engenharia de qualidade de conteúdo.

## 4. Visão comercial

Comercialmente, esta feature sustenta uma promessa importante: o cliente pode trazer documentos reais, inclusive PDFs difíceis, e a plataforma tem uma esteira séria para transformá-los em base consultável.

Isso resolve dores muito concretas.

- Contratos e editais que chegam escaneados.
- Relatórios técnicos longos com tabelas e imagens.
- Normas e laudos com leitura por página.
- Catálogos e manuais com mistura de texto, diagramas e anexos.

O diferencial suportado pelo código não é apenas ter OCR. O diferencial é a combinação entre parsing estruturado, OCR seletivo, filas determinísticas de engine, multimodalidade opcional e telemetria por etapa. O que não deve ser prometido é milagre. O próprio código deixa claro que qualidade depende do documento, do ambiente e das engines disponíveis.

## 5. Visão estratégica

Estratégicamente, o pipeline PDF fortalece a plataforma em cinco frentes.

- Reduz acoplamento entre parsing, OCR, chunking e persistência.
- Sustenta evolução incremental de engines sem reescrever o core.
- Reforça a arquitetura YAML-first, porque seleção de comportamento vem de contrato canônico.
- Cria base operacional para retomada e observabilidade fina.
- Prepara o produto para cenários multimodais e document intelligence mais ricos.

Isso é relevante porque PDF costuma virar atalho perigoso em produtos de IA. Quando o produto resolve PDF com heurística rasa, a dívida técnica nasce na entrada e contamina todo o restante. Aqui o desenho tenta impedir isso explicitamente.

## 6. Conceitos necessários para entender

### 6.1. Parsing nativo

Parsing nativo é a leitura do que já existe no PDF como texto, tabela, imagem incorporada, metadado e estrutura de página. Ele é mais barato e, quando funciona bem, preserva melhor o conteúdo original do que rasterizar tudo.

### 6.2. OCR document-level

OCR document-level é o pré-processamento do PDF inteiro antes do parsing principal. No código, ele existe para casos em que o documento parece escaneado, vazio demais ou suspeito demais para confiar no texto nativo.

### 6.3. OCR básico complementar

OCR básico complementar é um passo posterior, acionado no fluxo rico quando o conteúdo processado ainda está vazio ou quando a configuração manda tentar OCR em páginas vazias. Ele não substitui o parsing; ele tenta completar o que faltou.

### 6.4. Engine determinística

Engine determinística significa que a ordem de tentativa de parsing vem do YAML, e não de ifs escondidos no core. O resolvedor monta uma fila ordenada e cada engine só leva à próxima quando falha, devolve resultado insuficiente ou está indisponível conforme a política configurada.

### 6.4.1. Arquitetura plug-and-play de engines

No contexto deste projeto, plug-and-play não significa plugar qualquer biblioteca arbitrária e esperar que o core descubra sozinho como usá-la. Significa outra coisa, mais disciplinada: o runtime PDF foi desenhado para compor engines compatíveis dentro de uma fila ordenada por YAML, usando um contrato comum de options, modes, gatilhos e política de falha.

Na prática, isso cria uma arquitetura de composição. O parser comum lê a fila declarada, o resolvedor transforma cada item em uma engine suportada, a engine determinística decide quando a próxima deve entrar e o restante do pipeline continua neutro em relação ao nome da engine escolhida. O ganho operacional é importante: a plataforma pode reordenar, combinar, desativar ou tornar obrigatória uma engine sem reescrever o fluxo principal de PDF.

Esse desenho também protege contra dois erros comuns. O primeiro erro seria hardcode no orquestrador, do tipo se a engine anterior for X, rode Y. O segundo erro seria fallback implícito por conveniência. O código observado evita os dois: a fila vem do contrato canônico, a passagem para a próxima engine depende de sinais objetivos e a política de falha fica explícita.

O limite importante é este: a composição plug-and-play só vale para engines compatíveis com o contrato do runtime e efetivamente suportadas no resolvedor. Não existe promessa honesta de que qualquer engine externa entra sem implementação dedicada.

### 6.5. Failure policy

Failure policy é a regra que decide o que fazer quando nenhuma engine alcança sucesso formal.

- `strict_first_success` aborta quando ninguém entrega sucesso aceitável.
- `best_effort` permite devolver o melhor resultado parcial disponível.

### 6.6. Fluxo rico

Fluxo rico é a orquestração que junta resolução do texto-base, pós-processamento textual, OCR complementar, multimodalidade opcional e chunking final.

### 6.7. Multimodalidade

Multimodalidade, neste contexto, é a capacidade de tratar imagens do PDF como fonte útil de evidência. O runtime pode extrair imagens, aplicar OCR visual, descrever imagens e produzir chunks enriquecidos com metadata visual.

### 6.8. Manifesto operacional

Manifesto operacional é o registro por etapa persistido em `metadata.operational_controls.execution_manifest`. Ele existe para contar a história do processamento, manter checkpoints e permitir retomada coerente.

## 7. Como a feature funciona por dentro

O entrypoint lógico do PDF é o `PDFContentProcessor`, mas a decisão real fica distribuída em serviços especializados. O bootstrap inicializa o runtime PDF, resolve configurações canônicas, monta as engines e constrói dois pipelines explícitos: extração e limpeza textual. Depois disso, o boundary oficial do `process_document` delega para um fluxo rico que pode passar por OCR básico, multimodalidade e chunking.

O ponto arquitetural mais relevante para este pedido é que esse bootstrap não escolhe uma engine fixa. Ele monta uma composição plug-and-play governada pelo YAML. Isso permite que o parsing PDF evolua por troca de opções e combinação de engines suportadas, em vez de forçar uma única biblioteca a resolver todos os tipos de documento.

O ponto importante é que o pipeline não é monolítico. O código separa claramente:

- bootstrap e wiring do runtime;
- extração do conteúdo bruto do PDF;
- limpeza e normalização textual;
- decisão de OCR complementar;
- decisão multimodal;
- overrides de domínio aplicados sobre a configuração PDF quando um domínio ativo assim determina;
- chunking por Strategy Pattern;
- enriquecimento de chunks por cadeia de processadores de domínio;
- checkpoint e manifesto operacional;
- reentrada na esteira comum de indexação e persistência.

Essa separação tem valor prático. Ela permite responder perguntas diferentes com precisão.

- O problema aconteceu antes ou depois do parsing?
- O PDF precisava de OCR document-level?
- A engine falhou ou só não entregou texto suficiente?
- O multimodal abortou ou caiu em fallback textual?
- O chunking falhou por estratégia ou por falta de conteúdo útil?

### 7.1. O que acontece antes de o PDF chegar ao processor

No produto real, o PDF não entra primeiro no `PDFContentProcessor`. Antes disso, a API pública recebe `POST /rag/ingest`, compõe o YAML com a sessão do usuário, resolve o paralelismo documental e agenda um job pai na fila assíncrona.

Em linguagem simples: a API organiza o pedido e entrega o trabalho para o worker. Ela não fica esperando OCR, parsing ou chunking terminarem dentro da resposta HTTP.

### 7.2. O que o job pai faz

O job pai é o coordenador do lote. Ele decide se a ingestão segue como lote simples ou se vale quebrar o trabalho em documentos individuais.

Quando o fan-out documental é elegível, esse job inventaria os documentos, grava o plano no estado durável e publica envelopes filhos para a fila documental. Isso quer dizer que o job pai prepara a execução, mas não é ele quem faz o parsing final de cada PDF.

### 7.3. O que o job filho faz

O job filho executa uma unidade documental real. Ele recebe a referência do documento, valida se o pai ainda autoriza execução, respeita cancelamento cooperativo e só então chama a esteira especializada do PDF.

Na prática, quando alguém fala que o sistema está processando PDFs em paralelo, o código atual quer dizer isto: vários jobs filhos podem estar executando documentos diferentes ao mesmo tempo, cada um passando pela mesma esteira PDF.

### 7.4. Quando existe paralelismo de verdade

O paralelismo por documento não vale para qualquer origem. O código lido só considera fan-out quando a fonte é remota, replayable e compartilhável entre processos. Se o arquivo depende de filesystem local não compartilhado, o comportamento correto é não abrir jobs filhos paralelos.

Essa regra existe para evitar um erro comum: imaginar que basta pedir `document_parallelism=8` para qualquer upload local virar oito workers úteis. Sem fonte compartilhável, isso seria só aparência de paralelismo.

## 8. Divisão em etapas ou submódulos

Detalhamento aprofundado por etapa:

1. [Bootstrap do runtime PDF](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-BOOTSTRAP-DO-RUNTIME-PDF.md)
2. [Extracao documental principal](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-EXTRACAO-DOCUMENTAL-PRINCIPAL.md)
3. [Pos-processamento textual](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-POS-PROCESSAMENTO-TEXTUAL.md)
4. [OCR complementar](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-OCR-COMPLEMENTAR.md)
5. [Trilha multimodal](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-TRILHA-MULTIMODAL.md)
6. [Chunking orientado por estrategia](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-CHUNKING-ORIENTADO-POR-ESTRATEGIA.md)
7. [Persistencia operacional e indexacao](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-PERSISTENCIA-OPERACIONAL-E-INDEXACAO.md)

### 8.1. [Bootstrap do runtime PDF](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-BOOTSTRAP-DO-RUNTIME-PDF.md)

Esta etapa existe para consolidar tudo o que o processor precisa saber antes de tocar no documento. Ela resolve o contrato YAML, inicializa parâmetros de OCR, tabelas, filtros de qualidade, metadata, multimodalidade e só então instancia os serviços de suporte.

O que recebe: configuração YAML já carregada.

O que faz: traduz a configuração canônica em runtime executável.

O que entrega: coordinator, builder, pipelines, flags de multimodalidade e bundle de serviços.

Por que isso importa: evita que cada documento tenha de descobrir sua configuração em vários lugares diferentes.

### 8.2. [Extracao documental principal](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-EXTRACAO-DOCUMENTAL-PRINCIPAL.md)

Esta etapa existe para transformar bytes de PDF em um resultado estruturado de parsing. Ela valida bytes, pode aplicar OCR documental antes do parsing, executa a engine resolvida e prepara o payload final com texto, tabelas, imagens, anexos e metadata derivada.

O que recebe: `StorageDocument` com bytes do PDF.

O que faz: produz a base técnica do documento.

O que entrega: texto extraído, resumo de OCR documental, metadata e artefato de extração.

Por que isso importa: é a etapa que define se o sistema está trabalhando sobre um PDF real e legível ou sobre uma ilusão de conteúdo.

### 8.3. [Pos-processamento textual](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-POS-PROCESSAMENTO-TEXTUAL.md)

Esta etapa existe para limpar sem destruir. Ela preserva a estrutura do PDF, remove artefatos básicos e corrige alguns ruídos simples de OCR.

O que recebe: texto extraído do parsing.

O que faz: normaliza o texto para uso posterior.

O que entrega: texto processado e checkpoint do pipeline textual.

Por que isso importa: texto bruto de PDF costuma ser tecnicamente extraído, mas semanticamente ruim para chunking e recuperação.

### 8.4. [OCR complementar](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-OCR-COMPLEMENTAR.md)

Esta etapa existe para evitar dois extremos ruins.

- Confiar cegamente em texto insuficiente.
- Aplicar OCR pesado em tudo e degradar conteúdo bom.

O que recebe: texto já processado e acesso à fonte visual do PDF.

O que faz: decide se vale rodar OCR básico e, se rodar, mescla o novo texto sem sobrescrever o que já era útil.

O que entrega: texto complementado ou a confirmação de que o parsing inicial era suficiente.

Por que isso importa: reduz custo e reduz risco de degradar PDFs já bons.

### 8.5. [Trilha multimodal](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-TRILHA-MULTIMODAL.md)

Esta etapa existe para quando o PDF é visual o suficiente para merecer leitura por imagem. Ela tenta extrair imagens relevantes, processá-las e transformar esse material em enriquecimento do texto e dos chunks.

O que recebe: documento PDF, conteúdo textual já processado e fonte visual resolvível.

O que faz: executa OCR multimodal, descrição de imagem, possível embedding visual e montagem de chunks multimodais.

O que entrega: texto enriquecido, relatórios por etapa, artefatos de execução e chunks multimodais.

Por que isso importa: alguns PDFs falham não porque não têm texto, mas porque a evidência relevante está em imagem, figura, diagrama ou bloco visual.

### 8.6. [Chunking orientado por estrategia](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-CHUNKING-ORIENTADO-POR-ESTRATEGIA.md)

Esta etapa existe para não cortar todo PDF do mesmo jeito. O serviço avalia o tipo de conteúdo, informações de página e a ordem de estratégias disponíveis para decidir como quebrar o documento. Quando uma estratégia finalmente produz chunks válidos, o fluxo ainda não terminou: o processor pode passar esses chunks pela capability de domain processing para acrescentar metadata especializada de negócio.

O que recebe: texto final processado e metadata do documento.

O que faz: tenta estratégias ordenadas, cai em chunking simples se nenhuma gerar resultado e, quando `domain_specific_processing` está ativo, aplica a cadeia de plugins configurados por prioridade.

O que entrega: `ContentChunk` com metadata de página, estratégia, seção e, quando aplicável, sinais de domínio enriquecidos para retrieval posterior.

Por que isso importa: chunk errado compromete recuperação mesmo quando a extração textual foi boa. E chunk sem metadata de domínio pode continuar "legível", mas perder muito valor em cenários como food service, ERP, catálogos, cupons ou outros domínios suportados.

### 8.7. [Persistencia operacional e indexacao](README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO-PERSISTENCIA-OPERACIONAL-E-INDEXACAO.md)

Esta etapa existe para ligar o PDF processado à esteira comum da ingestão. Depois que o processor devolve chunks, o executor genérico completa metadata canônica, indexa no vector store e persiste o documento processado.

O que recebe: documento processado e chunks.

O que faz: indexa, persiste e registra telemetria final.

O que entrega: documento efetivamente incorporado ao acervo.

Por que isso importa: o pipeline de PDF não termina no chunk. Ele só vira valor de produto quando entra no acervo consultável.

## 9. Pipeline ou fluxo principal

![9. Pipeline ou fluxo principal](../assets/diagrams/docs-readme-conceitual-ingestao-pdf-pipeline-completo-diagrama-01.svg)

O diagrama mostra o fluxo ponta a ponta observado no código. A mensagem principal é simples: a ingestão PDF começa na API, passa por fila e worker e só depois entra no slice especializado do documento.

### 9.1. Subpipeline interno do documento PDF

![9.1. Subpipeline interno do documento PDF](../assets/diagrams/docs-readme-conceitual-ingestao-pdf-pipeline-completo-diagrama-02.svg)

Esse segundo diagrama é o que acontece dentro da unidade documental, seja ela executada pelo lote simples, seja por um worker filho do fan-out.

## 10. Decisões técnicas e trade-offs

### 10.1. OCR documental separado do OCR complementar

Ganho: o sistema diferencia o PDF inteiro parece problemático de faltou texto em partes do fluxo.

Custo: mais complexidade operacional e mais telemetria para manter.

Impacto: melhora o controle sobre quando rasterizar ou não, evitando OCR indiscriminado.

### 10.2. Fila determinística de engines em vez de engine única

Ganho: o produto pode combinar engines com perfis diferentes sem hardcode no core.

Custo: mais cuidado de configuração e mais cenários de indisponibilidade para tratar.

Impacto: reduz dependência de uma única engine e aproxima o produto de um runtime evolutivo.

Em linguagem direta, essa decisão é o que materializa a arquitetura plug-and-play do parsing PDF. O produto deixa de ter um parser único e passa a ter uma bancada ordenada de engines compostas, governadas por contrato e trocáveis sem refatorar o fluxo principal.

### 10.3. Falha explícita para contrato legado de parsing

Ganho: evita ambiguidade entre contrato antigo e contrato novo.

Custo: quebra configurações velhas em vez de mascará-las.

Impacto: melhora governança do YAML e reduz comportamento escondido.

### 10.4. Multimodalidade com `strict_mode` explícito

Ganho: a plataforma distingue um enriquecimento opcional de um enriquecimento obrigatório.

Custo: exige que o operador saiba qual tolerância quer para falha visual.

Impacto: evita fallback silencioso quando o caso exige leitura visual obrigatória.

### 10.5. Manifesto operacional e artefatos de retomada

Ganho: observabilidade forte e possibilidade de retomar a partir de estágios posteriores.

Custo: maior volume de metadata e necessidade de manter coerência entre manifesto e artefatos.

Impacto: reduz custo de reprocessamento e facilita troubleshooting forense.

## 11. Comparação com estado da arte

O estado da arte atual em ingestão de PDF, olhando para as referências normativas oficiais das stacks relevantes, converge em alguns princípios.

- OCR deve ser seletivo, não indiscriminado.
- PDFs digitais e PDFs escaneados não devem seguir o mesmo custo por padrão.
- Tabelas exigem tratamento especializado.
- Layout e imagens importam para RAG em documentos complexos.
- A ordem de leitura e a estrutura de página influenciam a qualidade final do chunking.

Comparando o pipeline do projeto com essas práticas:

### 11.1. Convergências fortes

- O OCR document-level usa análise heurística antes de aplicar OCR, o que converge com a ideia de OCR seletivo defendida por OCRmyPDF e PyMuPDF4LLM.
- A fila determinística de parsing aproxima o runtime de uma estratégia auto, mas com governança local explícita pelo YAML.
- O uso opcional de Docling, PyMuPDF4LLM, Unstructured, PyMuPDF, GMFT e OCRmyPDF mostra alinhamento com o ecossistema contemporâneo de document intelligence.
- A trilha multimodal reconhece que imagem e figura podem carregar evidência útil, o que está alinhado com a evolução recente de pipelines documentais para RAG.

### 11.2. Diferenciais práticos do projeto

- O projeto separa claramente manifesto operacional, checkpoint, artefato de retomada e fallback multimodal textual. Isso não é só parsing; é engenharia operacional do documento.
- A seleção de engine é governada por contrato YAML canônico, em vez de heurística escondida no core.
- O pipeline liga parsing e multimodalidade diretamente à esteira comum de indexação do produto, o que reduz trabalho duplicado entre extração documental e ingestão de fato.

### 11.3. Limites frente ao estado da arte

- O projeto ainda depende da disponibilidade local das dependências de engine. Quando elas faltam, algumas opções são desabilitadas e o resultado passa a depender da fila remanescente.
- O OCR document-level usa heurísticas próprias e OCRmyPDF com uma única família de engine permitida. Isso é robusto para governança, mas menos plural do que alguns stacks de document intelligence mais recentes.
- A documentação oficial das stacks como Docling e PyMuPDF4LLM já expõe recursos mais ricos para layout, VLM e exportações estruturadas. O projeto usa parte desse potencial, mas não todo ele de forma confirmada no slice lido.

## 12. O que acontece em caso de sucesso

No caminho feliz, o PDF entra com bytes válidos, o runtime é inicializado, a engine adequada extrai conteúdo útil, o texto é limpo, o multimodal só roda quando faz sentido, o chunking encontra uma estratégia adequada e a esteira comum indexa e persiste o resultado.

Para o usuário do produto, isso aparece como um documento que passa a ser recuperável e consultável. Para operação, isso aparece como manifesto operacional coerente, métricas de página, engine usada, estratégia de chunking, status multimodal e chunks indexados.

## 13. O que acontece em caso de erro

Os principais cenários confirmados no código são estes.

- PDF sem bytes ou com assinatura inválida.
- Runtime de OCR documental indisponível.
- Engine de parsing indisponível em modo obrigatório.
- Falha de parsing sem resultado utilizável.
- Ausência de conteúdo base para o fluxo rico.
- Falha na origem visual para OCR básico ou multimodal.
- Exceção no processador multimodal com `strict_mode=true`, abortando o fluxo.
- Chunking sem estratégia útil, levando ao fallback simples.
- Retomada solicitada sem artefato de extração obrigatório.

O ponto mais importante é este: vários erros são tratados com decisão explícita, não com silêncio. O pipeline registra status, motivo e, quando aplicável, decide entre abortar e cair para texto.

## 14. Observabilidade e diagnóstico

O pipeline PDF foi desenhado para contar a história do documento. Os pontos de observabilidade mais relevantes são:

- logs de início e fim do `process_document`;
- logs do OCR document-level com análise, decisão e runtime preflight;
- logs da fila determinística de parsing e da engine efetivamente usada;
- `metadata.operational_controls.execution_manifest`;
- `metadata.operational_controls.execution_artifacts`;
- `multimodal_status_details`;
- métricas de OCR básico, páginas, chunking e qualidade;
- resumo operacional final do PDF antes da indexação.

Na prática, o diagnóstico eficiente sempre segue esta ordem.

1. Confirmar se o documento entrou como PDF válido.
2. Descobrir qual engine foi tentada e qual venceu.
3. Ver se houve OCR documental e por quê.
4. Ver se o texto continuou vazio ou insuficiente depois do parsing.
5. Conferir se houve multimodal e qual status final ele registrou.
6. Confirmar qual estratégia de chunking gerou os chunks finais.
7. Só então investigar persistência e indexação.

## 15. Impacto técnico

O impacto técnico principal é aumentar a separação de responsabilidades sem perder encadeamento operacional. O pipeline reforça:

- baixo acoplamento entre decisão de OCR, decisão de parsing e chunking;
- capacidade de trocar ou reordenar engines por contrato;
- maior testabilidade de etapas específicas;
- melhor diagnóstico de falha por slice;
- preparação concreta para documentos multimodais;
- integração limpa com a esteira comum de persistência do produto.

## 16. Impacto executivo

Para liderança, esta feature reduz risco de conhecimento mal ingerido e aumenta previsibilidade de operação documental. Também reduz a dependência de diagnósticos manuais ad hoc, porque o pipeline produz uma trilha operacional mais explicável.

## 17. Impacto comercial

Para venda e pré-venda, o pipeline PDF melhora a história em contas corporativas onde documento regulatório, manual técnico, contrato e laudo são fontes centrais de conhecimento. O diferencial não é aceita PDF, e sim tem uma esteira governada para PDF difícil.

## 18. Impacto estratégico

Estratégicamente, esta feature prepara a plataforma para ampliar document intelligence sem refazer a ingestão do zero. O desenho atual já suporta evolução por engine, por multimodalidade e por contratos YAML mais ricos, o que é muito mais valioso do que um parser único difícil de evoluir.

## 19. Exemplos práticos guiados

### 19.1. PDF digital bem formado

Cenário: o cliente envia um manual técnico em PDF com texto nativo e algumas tabelas.

Processamento esperado: o pipeline usa a trilha de parsing sem precisar de OCR documental, limpa o texto, detecta tabelas, gera chunks por estratégia e indexa normalmente.

Impacto prático: custo menor e preservação melhor do texto original.

### 19.2. PDF escaneado com pouco texto

Cenário: o cliente envia um relatório escaneado com páginas quase sem texto nativo.

Processamento esperado: o OCR document-level detecta sinais de documento escaneado, tenta pré-processamento, a fila de parsing trabalha sobre o novo material e, se ainda faltar conteúdo, o fluxo rico pode complementar com OCR básico.

Impacto prático: o documento deixa de entrar vazio no acervo.

### 19.3. PDF com imagens relevantes

Cenário: um laudo contém diagramas, plantas ou imagens que carregam evidência.

Processamento esperado: a trilha multimodal só roda se o documento for visual, o recurso estiver habilitado e a fonte visual estiver disponível. Se tudo der certo, o texto é enriquecido e os chunks podem ganhar metadata visual.

Impacto prático: o produto reduz o risco de perder conhecimento que não está só no texto corrido.

## 20. Explicação 101

Pense no pipeline PDF como uma triagem hospitalar para documento.

Primeiro ele pergunta: isso aqui é mesmo um PDF válido?
Depois pergunta: dá para confiar no texto que já existe ou preciso de OCR?
Depois pergunta: qual ferramenta faz mais sentido para este caso?
Depois pergunta: o texto final está bom o suficiente ou ainda falta algo visual?
Só no final ele pergunta: como vou quebrar isso em pedaços úteis para a busca?

Esse encadeamento é o que impede o sistema de tratar todo PDF como se fosse igual.

## 21. Limites e pegadinhas

- PDF válido não significa PDF útil. Ele ainda pode ter texto ruim.
- OCR melhora cobertura, mas não cria qualidade onde o scan é ruim demais.
- Engine mais rica não é automaticamente melhor para todos os documentos.
- Multimodalidade não é gratuita. Ela aumenta custo e superfície de falha.
- Chunking correto depende do texto final. Se a extração veio ruim, o chunking só organiza um problema já existente.
- Parte do estado da arte atual já oferece recursos ainda mais ricos de layout e VLM. O projeto está alinhado em direção, mas não usa tudo isso de forma comprovada no slice lido.

## 22. Checklist de entendimento

- Entendi por que PDF precisa de pipeline próprio.
- Entendi a diferença entre OCR document-level e OCR complementar.
- Entendi por que a fila de engines é determinística.
- Entendi quando o multimodal entra.
- Entendi que o chunking é a etapa final do processor, não o pipeline inteiro.
- Entendi como o manifesto operacional ajuda no diagnóstico.
- Entendi o valor executivo, comercial e estratégico da feature.
- Entendi os limites e as promessas que não devem ser exageradas.

## 23. Evidências no código

- `src/api/routers/rag_ingestion_router.py`
  - Motivo da leitura: localizar a borda HTTP pública da ingestão.
  - Símbolo relevante: `build_router`.
  - Comportamento confirmado: registra `POST /rag/ingest` como entrada oficial da ingestão documental.

- `src/api/routers/rag_runtime_ingestion_compat.py`
  - Motivo da leitura: entender a preparação do pedido antes da fila.
  - Símbolo relevante: `PreparedAsyncIngestionExecutionService.__call__`.
  - Comportamento confirmado: compõe YAML, resolve paralelismo, agenda o job pai e devolve contrato HTTP de acompanhamento.

- `src/api/services/ingestion_http_prepared_async_service.py`
  - Motivo da leitura: confirmar o agendamento do job pai preparado.
  - Símbolo relevante: `schedule_prepared_ingestion_worker_job`.
  - Comportamento confirmado: registra run pai, valida telemetria durável e publica o envelope `prepared_yaml`.

- `src/api/services/async_job_dramatiq.py`
  - Motivo da leitura: entender como o worker separa pai e filho.
  - Símbolo relevante: `DramatiqAsyncJobWorkerRuntime.start`.
  - Comportamento confirmado: sobe consumidores separados para filas pai e filha com contrato versionado.

- `src/api/services/worker_process_runtime.py`
  - Motivo da leitura: confirmar o runtime oficial do worker.
  - Símbolo relevante: `build_worker_process_runtime` e `WorkerProcessRuntime.start`.
  - Comportamento confirmado: exige Dramatiq + RabbitMQ e sobe o runtime unificado do processo worker.

- `app/runners/worker_runner.py`
  - Motivo da leitura: entender o bootstrap do processo worker.
  - Símbolo relevante: `run_worker_process`.
  - Comportamento confirmado: prepara ambiente, valida infraestrutura e inicia o runtime oficial do worker.

- `src/services/ingestion_service.py`
  - Motivo da leitura: localizar a decisão entre lote simples e fan-out.
  - Símbolo relevante: `_build_document_fanout_plan`.
  - Comportamento confirmado: delega o planejamento paralelo ao coordenador especializado.

- `src/services/document_fanout_coordinator.py`
  - Motivo da leitura: entender o que significa paralelismo por documento.
  - Símbolo relevante: `build_plan`.
  - Comportamento confirmado: só habilita fan-out para fontes remotas elegíveis e publica envelopes filhos até o limite operacional.

- `src/services/document_fanout_child_executor_service.py`
  - Motivo da leitura: entender a responsabilidade do filho.
  - Símbolo relevante: `DocumentFanoutChildExecutorService.execute`.
  - Comportamento confirmado: executa um documento por vez, consulta a gate canônica e persiste estado terminal antes do ACK.

- `tests/integration/test_03-01-08_async_job_dramatiq_real_flow.py`
  - Motivo da leitura: confirmar o fluxo assíncrono com evidência executável.
  - Símbolo relevante: `test_runtime_real_consumes_parent_and_child_queues`.
  - Comportamento confirmado: valida consumo real de envelopes pai e filho em filas distintas.

- `src/ingestion_layer/processors/pdf_processor.py`
  - Motivo da leitura: entrypoint real, bootstrap, decisões de OCR básico, multimodalidade, chunking e checkpoint.
  - Símbolo relevante: `PDFContentProcessor`.
  - Comportamento confirmado: coordena runtime PDF, fluxo rico, multimodal e integração com a esteira comum.

- `src/ingestion_layer/processors/pdf_document_processing_application_service.py`
  - Motivo da leitura: boundary oficial de `process_document`.
  - Símbolo relevante: `PdfDocumentProcessingApplicationService.process_document`.
  - Comportamento confirmado: valida o documento, executa pre e post hooks, chama o fluxo rico e faz cleanup final.

- `src/ingestion_layer/processors/pdf_extraction_application_service.py`
  - Motivo da leitura: extração principal, manifesto e retomada.
  - Símbolo relevante: `PdfExtractionApplicationService.extract_pdf_text`.
  - Comportamento confirmado: executa pipeline de extração, persiste manifesto e cria artefato de extração reutilizável.

- `src/ingestion_layer/processors/pdf_document_ocr_service.py`
  - Motivo da leitura: decisão de OCR documental.
  - Símbolo relevante: `PdfDocumentOcrService.maybe_preprocess_pdf`.
  - Comportamento confirmado: analisa heurísticas do PDF e só aplica OCR documental quando a decisão justificar.

- `src/ingestion_layer/processors/pdf_parsing_runtime_builder.py`
  - Motivo da leitura: wiring do runtime de parsing.
  - Símbolo relevante: `PdfParsingRuntimeBuilder.build`.
  - Comportamento confirmado: monta OCR, tabelas, metadata, pages info e a engine final de parsing.

- `src/ingestion_layer/processors/pdf_parsing_engine_resolver.py`
  - Motivo da leitura: fila determinística de engines.
  - Símbolo relevante: `PdfParsingEngineResolver.resolve`.
  - Comportamento confirmado: rejeita contrato legado e monta options ordenadas por YAML com policy explícita.

- `src/ingestion_layer/pdf_tools/deterministic_lego_pdf_parsing_engine.py`
  - Motivo da leitura: semântica de tentativa e sucesso das engines.
  - Símbolo relevante: `DeterministicLegoPdfParsingEngine`.
  - Comportamento confirmado: tenta engines em ordem, decide handoff e aplica `strict_first_success` ou `best_effort`.

- `src/ingestion_layer/processors/pdf_multimodal_application_service.py`
  - Motivo da leitura: trilha multimodal do PDF.
  - Símbolo relevante: `PdfMultimodalApplicationService.process_multimodal_document`.
  - Comportamento confirmado: decide fallback textual, persiste stage reports, registra status multimodal e monta chunks multimodais.

- `src/ingestion_layer/processors/pdf_chunking_service.py`
  - Motivo da leitura: chunking final do PDF.
  - Símbolo relevante: `PdfChunkingService.create_chunks`.
  - Comportamento confirmado: aplica Strategy Pattern e fallback simples quando nenhuma estratégia gera chunks.

- `src/ingestion_layer/file_pipeline_services.py`
  - Motivo da leitura: fechamento na esteira comum.
  - Símbolo relevante: `DocumentIndexingExecutor.finalize`.
  - Comportamento confirmado: adiciona metadata canônica, indexa os chunks e persiste o documento processado.
