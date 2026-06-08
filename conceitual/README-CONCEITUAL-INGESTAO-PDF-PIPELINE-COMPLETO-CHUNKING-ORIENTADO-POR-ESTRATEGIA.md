# Manual detalhado da etapa: Chunking orientado por estrategia do pipeline PDF

## 1. O que esta etapa faz

Esta etapa decide como quebrar o conteudo final do PDF em chunks utilizaveis para retrieval e indexacao. O objetivo nao e apenas cortar texto em tamanhos fixos. O objetivo e escolher a melhor estrategia disponivel para o tipo de conteudo que saiu das etapas anteriores.

Em linguagem simples: e a etapa que transforma texto processado em unidades consultaveis com semantica melhor do que um recorte cego por caracteres.

## 2. Onde ela entra no fluxo

No codigo lido, esta etapa esta centralizada em PdfChunkingService.create_chunks. Ela roda depois de extracao, pos-processamento, OCR complementar e possivel trilha multimodal textual. Quando a trilha multimodal nao assume a entrega final, e o chunking que produz a saida padrao do processor PDF.

## 3. O que entra e o que sai

Entradas confirmadas:

- documento PDF
- processed_content final
- parametros adaptativos de chunk_size e chunk_overlap
- page_info extraido da metadata
- lista ordenada de estrategias de chunking

Saidas confirmadas:

- lista de ContentChunk valida
- strategy_used registrado no resumo operacional
- metadata de secao anexada aos chunks quando aplicavel
- enriquecimento de dominio aplicado depois do chunking

## 4. Como o codigo implementa a etapa

O fluxo real observado em create_chunks segue esta ordem.

1. calcula parametros adaptativos de chunking;
2. registra log inicial com os parametros efetivos;
3. se o texto estiver vazio, devolve lista vazia e registra no_chunks_created;
4. resolve o runtime do chunking com estrategias, content_type detectado e page_info;
5. entra em um laco ordenado de estrategias;
6. para cada estrategia, verifica can_handle;
7. se a estrategia aceitar o conteudo, tenta create_chunks;
8. converte chunks brutos para ContentChunk canonico;
9. a primeira estrategia que produzir chunks validos vence;
10. se nenhuma estrategia funcionar, usa fallback simples de text splitting;
11. finaliza os chunks aplicando domain processing, section metadata e resumo operacional.

O processor hospedeiro monta as estrategias em ordem. No slice lido, a ordem padrao inclui page based, section based, paragraph based e sentence based.

## 5. Decisoes tecnicas importantes

### 5.1. Primeira estrategia valida vence

O laco para na primeira estrategia que gera chunks validos. Isso e uma decisao clara de engenharia. Em vez de combinar varias estrategias e depois decidir, o sistema assume uma fila ordenada e governada pelo host.

O ganho pratico e previsibilidade. O custo e que a qualidade da ordem de estrategias importa muito.

### 5.2. Fallback do chunking e local e explicito

Se nenhuma estrategia gerar chunks validos, o servico usa _create_fallback_chunks e registra strategy_used como fallback. Isso e importante porque nao esconde a degradacao. O operador sabe que o chunking estruturado falhou e que o pipeline caiu em uma alternativa simples.

### 5.3. Domain processing entra depois do chunking

A finalizacao do chunking chama primeiro o enriquecimento por dominio e depois anexa metadata de secao. Isso mostra uma escolha arquitetural importante: o dominio nao define a quebra inicial do texto. Ele enriquece os chunks ja criados.

Na pratica, isso mantem o problema de segmentacao separado do problema de semantica de negocio.

## 6. O que pode dar errado

Problemas confirmados ou diretamente implicados pelo codigo:

- processed_content vazio gera zero chunks;
- uma estrategia pode rejeitar o conteudo via can_handle;
- uma estrategia pode lancar excecao durante create_chunks, mas o laco continua;
- uma estrategia pode gerar chunks brutos que nao viram ContentChunk valido;
- nenhuma estrategia pode vencer, forçando fallback.

Isso e importante porque mostra que erro de estrategia nao implica necessariamente erro do documento inteiro.

## 7. Como diagnosticar

Os sinais mais uteis confirmados no codigo sao:

- log PDF CHUNKING: iniciando chunking com Strategy Pattern;
- content_type_detected e page_info no log de analise;
- available_strategies registradas;
- logs STRATEGY de accepted, rejected, erro ou success;
- event_name ingestion.pdf.chunking.completed com strategy_used;
- resumo operacional do PDF apos a finalizacao.

Em linguagem simples: se a busca posterior ficou ruim, a pergunta pratica e se o problema nasceu do texto ou da estrategia de chunking. Esses logs ajudam a separar as duas coisas.

## 8. Exemplo pratico guiado

Cenario: um PDF com forte estrutura por pagina e secoes entra no chunking.

1. O servico calcula chunk_size e overlap adaptativos.
2. Detecta o content_type predominante.
3. Tenta a estrategia page based.
4. Se ela aceitar e gerar chunks validos, vence imediatamente.
5. Os chunks passam por domain processing e recebem metadata de secao.
6. O resumo final registra a strategy_used.

Se a estrategia page based nao funcionar, o fluxo segue para section, paragraph e sentence. Se nenhuma funcionar, o fallback simples ainda preserva uma saida indexavel, mas com observabilidade explicita dessa perda de sofisticao.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/pdf_chunking_service.py
  - Simbolo relevante: PdfChunkingService.create_chunks
  - Comportamento confirmado: parametros adaptativos, laco de estrategias, fallback local e fechamento do chunking.
- src/ingestion_layer/processors/pdf_chunking_service.py
  - Simbolo relevante: os metodos internos de execucao, finalizacao e fallback do chunking PDF
  - Comportamento confirmado: primeira estrategia valida vence, domain processing entra no final e fallback simples fica explicitamente marcado.
