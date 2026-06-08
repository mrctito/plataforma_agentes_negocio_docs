# Manual detalhado da etapa: Pos-processamento textual do pipeline PDF

## 1. O que esta etapa faz

Esta etapa pega o texto bruto que saiu da extracao e o transforma em uma base mais limpa para chunking, retrieval e leitura operacional. Ela nao tenta reinventar o documento. O objetivo dela e limpar sem destruir.

Em linguagem simples: e a faxina disciplinada do texto PDF antes de quebrar em chunks.

## 2. Onde ela entra no fluxo

No codigo lido, esta etapa esta centralizada em PdfTextProcessingApplicationService.process. Ela roda logo depois da extracao documental principal e antes do OCR complementar, do multimodal e do chunking.

## 3. O que entra e o que sai

Entradas confirmadas:

- BaseDocument com content textual vindo da etapa de extracao
- metadata do documento
- parametros de preservacao de estrutura e chunking ja resolvidos no runtime

Saidas confirmadas:

- texto processado
- checkpoint do pipeline pdf_text_processing
- metricas de reducao e efetividade do processamento

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. registra um log de inicio com todos os parametros relevantes do processor PDF;
2. valida se document.content existe;
3. analisa o texto inicial contando page markers, quebras de linha e form feeds;
4. resolve ou cria o pipeline de texto;
5. monta PdfTextProcessingState com original_text e processed_text iguais no inicio;
6. executa o pipeline PreserveStructureStage, RemoveBasicArtifactsStage e FixSimpleOcrArtifactsStage;
7. grava checkpoint com stages_executed e stage_runs;
8. calcula reduction_percent e processing_effective;
9. devolve o texto final ja stripado.

O ponto importante e que essa etapa nao mistura limpeza com chunking. O pipeline textual termina antes de a estrategia de chunking ser decidida.

## 5. Decisoes tecnicas importantes

### 5.1. Preservar estrutura antes de limpar artefatos

O primeiro estagio do pipeline textual e PreserveStructureStage. Isso mostra a intencao do design: antes de remover ruido, o sistema tenta manter a estrutura util de paginas e marcadores.

Na pratica, isso protege o pipeline de um erro comum em PDF: limpar demais cedo demais e apagar sinais que depois seriam uteis para chunking e contexto.

### 5.2. Limpeza com checkpoint observavel

O codigo nao trata a limpeza textual como detalhe invisivel. Ele persiste checkpoint do pipeline pdf_text_processing com stage_runs. Isso e importante para diagnostico, porque permite diferenciar problema de extracao de problema de limpeza.

### 5.3. Documento vazio falha de forma segura

Se document.content vier vazio, o servico registra erro e devolve string vazia. Nao tenta forcar limpeza sobre conteudo inexistente.

## 6. O que pode dar errado

Os riscos confirmados ou diretamente implicados sao estes.

- documento chega com content vazio e a etapa devolve vazio;
- cancelamento cooperativo pode interromper antes ou depois do pipeline;
- excesso de limpeza pode reduzir demais o texto, embora o log de reduction_percent ajude a perceber isso.

O ponto pratico e que esta etapa nao deveria esconder degradacao severa do texto. Se o processamento reduziu demais o conteudo, isso precisa aparecer nos logs.

## 7. Como diagnosticar

Os sinais mais uteis confirmados no codigo sao:

- log PDF PROCESSOR: iniciando estrategia de processamento;
- processing_parameters com preserve_page_structure, include_page_markers e outros flags;
- log PDF ANALYSIS com page_markers_detected, newlines_count e form_feeds_count;
- checkpoint do pipeline pdf_text_processing;
- log final com original_size, final_size e reduction_percent.

Em linguagem simples: se a extracao parecia boa, mas o chunking recebeu um texto pobre, a investigacao passa por aqui.

## 8. Exemplo pratico guiado

Cenario: o parser extraiu texto com muitos marcadores quebrados e artefatos simples de OCR.

1. O servico registra os parametros do processor.
2. Conta page markers e form feeds do texto bruto.
3. PreserveStructureStage tenta manter a estrutura util.
4. RemoveBasicArtifactsStage remove ruido basico.
5. FixSimpleOcrArtifactsStage corrige artefatos simples.
6. O checkpoint e gravado.
7. O texto final segue para as decisoes de OCR complementar, multimodalidade e chunking.

O valor desta etapa e preparar o texto sem forcar ainda nenhuma estrategia de segmentacao.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/pdf_text_processing_application_service.py
  - Simbolo relevante: PdfTextProcessingApplicationService.process
  - Comportamento confirmado: analise inicial do texto, execucao do pipeline textual, checkpoint e metricas de reducao.
- src/ingestion_layer/processors/pdf_runtime_coordinator.py
  - Simbolo relevante: build_text_processing_pipeline
  - Comportamento confirmado: composicao explicita das etapas PreserveStructureStage, RemoveBasicArtifactsStage e FixSimpleOcrArtifactsStage.
