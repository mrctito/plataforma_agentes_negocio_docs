# Manual detalhado da etapa: Trilha multimodal do pipeline PDF

## 1. O que esta etapa faz

Esta etapa tenta usar o lado visual do PDF como fonte real de evidencia. Ela existe para casos em que o texto sozinho nao basta: imagens, diagramas, blocos visuais, paginas escaneadas ou trechos que so fazem sentido quando a imagem tambem entra no contexto.

Em linguagem simples: e a parte do pipeline que trata o PDF como documento visual, nao apenas como texto extraido.

## 2. Onde ela entra no fluxo

No codigo lido, a decisao de rodar multimodal aparece em dois lugares.

- PdfMultimodalApplicationService.should_run_multimodal decide se o documento e elegivel.
- PdfRichProcessingApplicationService.run chama process_multimodal_document quando essa decisao e positiva.

Isso significa que a trilha multimodal so entra depois de o documento ja ter texto-base resolvido e, se necessario, OCR complementar aplicado.

## 3. O que entra e o que sai

Entradas confirmadas:

- documento PDF
- texto processado ate aquele momento
- flags _enable_multimodal e _multimodal_strict_mode
- origem visual valida do PDF
- processador multimodal quando disponivel

Saidas confirmadas:

- chunks multimodais ou chunks textuais de fallback
- status multimodal persistido na metadata
- stage reports uniformizados para o pipeline pdf_multimodal
- artefatos operacionais do processamento visual, quando existirem

## 4. Como o codigo implementa a etapa

O fluxo real de process_multimodal_document segue esta ordem.

1. registra inicio com strict_mode e tamanho do texto atual;
2. se multimodal estiver desligado, marca status disabled e volta para chunks textuais;
3. se o documento nao for PDF valido para esse slice, marca status disabled e volta para chunks textuais;
4. se nao houver origem visual valida, marca status ignored e volta para chunks textuais;
5. se o processador multimodal nao existir, marca status skipped_error e decide entre fallback textual ou aborto, conforme strict_mode;
6. se o processador existir, chama process_document(pdf_source, processed_content);
7. persiste stage_reports do pipeline multimodal e artefatos de runtime quando disponiveis;
8. atualiza metadata com status, quantidade de imagens e fallback_to_text;
9. devolve chunks multimodais ou texto de fallback conforme o resultado.

Um detalhe estrutural importante e a ordem canonica de estagios multimodais usada pelo proprio servico.

- image_extraction
- multimodal_ocr
- image_description
- vision_embedding

Isso cria uma narrativa operacional consistente, mesmo quando o processamento falha cedo e os estagios seguintes precisam ser marcados como skipped.

## 5. Decisoes tecnicas importantes

### 5.1. Multimodalidade so entra quando o documento realmente pode sustentar isso

should_run_multimodal exige tres condicoes ao mesmo tempo.

- feature flag habilitada;
- documento reconhecido como PDF visual;
- origem visual resolvivel.

O ganho pratico e impedir que o pipeline multimodal rode em situacoes em que ele seria apenas custo extra e ruina de diagnostico.

### 5.2. Strict mode muda o contrato operacional da etapa

Quando strict_mode esta ativo, certas falhas nao levam a fallback textual. O pipeline aborta. Quando strict_mode esta desligado, a etapa registra a falha, preserva o status multimodal e devolve chunks textuais.

Na pratica, isso separa dois modos de operacao.

- modo tolerante: melhor aproveitar texto do que abortar tudo;
- modo estrito: melhor falhar do que aceitar resposta sem a trilha visual obrigatoria.

### 5.3. Status multimodal nao e cosmetico

O servico grava multimodal_status, multimodal_images_found, multimodal_images_processed, multimodal_fallback_to_text e um payload detalhado em multimodal_status_details. Isso transforma o resultado multimodal em contrato de diagnostico, nao em simples log de console.

## 6. O que pode dar errado

Falhas e comportamentos confirmados:

- multimodal desligado pela flag de feature;
- documento sem elegibilidade visual;
- origem visual ausente;
- processador multimodal indisponivel;
- excecao durante process_document do pipeline multimodal;
- strict_mode impedindo fallback textual e transformando uma falha multimodal em erro terminal.

Em termos prativos, esta etapa deixa muito claro quando o problema e estrutural, quando e apenas nao aplicavel e quando houve fallback voluntario.

## 7. Como diagnosticar

Os sinais mais uteis confirmados no codigo sao estes.

- log de inicio do fluxo PDF multimodal com strict_mode e text_chars;
- metadata multimodal_status e multimodal_status_details;
- contadores de imagens found, attempted, processed, discarded e failed;
- checkpoint do pipeline pdf_multimodal;
- stage reports com estagios skipped, failed ou completed;
- logs finais de conclusao ou abort do multimodal com result_path.

Em linguagem simples: se o PDF tinha imagem relevante, mas a resposta parece puramente textual, a explicacao operacional quase sempre passa por esses sinais.

## 8. Exemplo pratico guiado

Cenario: um PDF com diagrama importante entra com multimodal habilitado e strict_mode desligado.

1. O fluxo rico verifica que o PDF e visual e tem origem valida.
2. O servico multimodal registra inicio.
3. O processador multimodal extrai imagens e tenta OCR visual, descricao e embedding.
4. Se tudo der certo, devolve chunks multimodais.
5. Se o processador falhar, a etapa grava status skipped_error e cai para chunks textuais.
6. O diagnostico final deixa claro se a trilha visual foi usada ou so tentada.

O valor desta etapa e capturar evidencia que simplesmente nao existia no texto extraido puro.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/pdf_multimodal_application_service.py
  - Simbolo relevante: PdfMultimodalApplicationService.should_run_multimodal e process_multimodal_document
  - Comportamento confirmado: gate multimodal, status metadata, strict mode, fallback textual e persistencia de stage reports.
- src/ingestion_layer/processors/pdf_rich_processing_application_service.py
  - Simbolo relevante: PdfRichProcessingApplicationService.run
  - Comportamento confirmado: encaminhamento para a trilha multimodal apenas depois da resolucao do texto-base e do OCR complementar.
