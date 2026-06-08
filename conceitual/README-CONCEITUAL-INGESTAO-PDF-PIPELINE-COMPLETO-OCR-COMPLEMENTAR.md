# Manual detalhado da etapa: OCR complementar do pipeline PDF

## 1. O que esta etapa faz

Esta etapa decide se vale complementar o texto atual com OCR basico depois da extracao e do pos-processamento textual. Ela nao tenta substituir a extracao principal. O objetivo dela e recuperar texto faltante quando o conteudo ainda esta vazio ou quando a configuracao manda olhar paginas vazias.

Em linguagem simples: e o reforco pontual, nao a reescrita completa do documento.

## 2. Onde ela entra no fluxo

No codigo lido, esta etapa aparece dentro de PdfRichProcessingApplicationService.run. Ela acontece depois de process_content(document) e antes da decisao multimodal final. Portanto, fica no meio do fluxo rico: tarde o bastante para saber se o texto ja esta bom, cedo o bastante para ainda influenciar o restante do pipeline.

## 3. O que entra e o que sai

Entradas confirmadas:

- texto processado da etapa anterior
- documento PDF com origem visual resolvivel
- flag _ocr_on_empty_pages

Saidas confirmadas:

- texto complementar vindo do OCR basico quando houver
- metricas ocr_basic_metrics registradas na metadata
- texto mesclado sem sobrescrever conteudo util
- reprocessamento textual do conteudo mesclado quando houve retorno do OCR

## 4. Como o codigo implementa a etapa

O fluxo observado em PdfRichProcessingApplicationService.run e no processor PDF segue esta ordem.

1. calcula should_run_basic_ocr com base no texto processado;
2. se o texto ja existe e _ocr_on_empty_pages esta desligado, registra que o OCR basico sera ignorado;
3. se a etapa deve rodar, chama _apply_minimal_ocr(document);
4. _apply_minimal_ocr garante cancelamento cooperativo, resolve a origem visual e monta extrator de imagens e OCR processor;
5. a etapa coleta metricas de imagens encontradas, tentadas e com texto;
6. se houver texto novo, _merge_basic_ocr_content combina base_content e ocr_text em modo append_unique;
7. o documento recebe o texto mesclado e passa por process_content novamente.

O detalhe mais importante esta em _merge_basic_ocr_content. O codigo nao sobrescreve o texto base. Ele evita duplicacao quando o OCR ja esta contido no texto e, quando ha material novo, apenas anexa esse material.

## 5. Decisoes tecnicas importantes

### 5.1. OCR basico so entra quando ha motivo claro

O metodo que decide o OCR complementar devolve True quando o processed_content esta vazio ou quando a politica de OCR em paginas vazias permite complementar mesmo havendo texto parcial. Isso reduz OCR desnecessario em PDFs que ja sairam bons da extracao.

### 5.2. Sem origem visual valida, a etapa nao inventa caminho alternativo

Se _resolve_pdf_visual_source(document) devolve None, o codigo registra warning e retorna vazio. Isso e importante porque evita OCR ficticio sobre documento sem base visual disponivel.

### 5.3. O texto complementar passa de novo pela limpeza oficial

Quando o OCR basico devolve algo, o fluxo reexecuta process_content(document). Isso impede que o texto OCR entre cru no restante da esteira.

## 6. O que pode dar errado

Falhas confirmadas ou implicadas pelo codigo:

- documento sem origem visual valida impede o OCR basico;
- extrator de imagens indisponivel faz a etapa abortar com excecao registrada;
- OCR processor indisponivel tambem vira falha explicita;
- OCR pode nao retornar texto util, caso em que o fluxo preserva a extracao inicial.

Na pratica, a etapa distingue bem tres estados: nao precisava rodar, rodou mas nao encontrou nada util, ou falhou estruturalmente.

## 7. Como diagnosticar

Os sinais uteis confirmados no codigo sao estes.

- log de decision_summary do fluxo rico com basic_ocr_selected;
- log "Fluxo PDF: OCR basico ignorado para preservar texto extraido";
- log "Fluxo PDF: executando OCR basico apos extracao inicial";
- log de telemetria OCR basico consolidada;
- log "OCR basico ja contido no texto base; evitando duplicacao" quando o merge detecta repeticao;
- campo ocr_basic_metrics na metadata.

Em linguagem simples: se voce quer saber se o OCR complementar ajudou de fato, o diagnostico esta nesses logs e nessas metricas.

## 8. Exemplo pratico guiado

Cenario: o parser principal extraiu quase nada de um PDF escaneado, mas o documento tem origem visual valida.

1. O fluxo rico processa o texto e constata que ele esta vazio.
2. _should_run_basic_ocr escolhe True.
3. _apply_minimal_ocr resolve o PDF visual, extrai imagens e roda OCR leve.
4. O texto OCR volta.
5. _merge_basic_ocr_content anexa esse texto ao conteudo base.
6. process_content roda novamente sobre o conteudo mesclado.
7. O documento segue adiante com base textual melhor para multimodal e chunking.

O valor desta etapa e melhorar documentos ruins sem destruir documentos que ja estavam bons.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/pdf_rich_processing_application_service.py
  - Simbolo relevante: PdfRichProcessingApplicationService.run
  - Comportamento confirmado: decisao consolidada do pipeline rico, execucao opcional do OCR basico e reprocessamento do conteudo quando ha retorno util.
- src/ingestion_layer/processors/pdf_processor.py
  - Simbolo relevante: os metodos que decidem, mesclam e executam o OCR basico complementar
  - Comportamento confirmado: criterio de ativacao, merge sem sobrescrita e dependencia de origem visual valida.
