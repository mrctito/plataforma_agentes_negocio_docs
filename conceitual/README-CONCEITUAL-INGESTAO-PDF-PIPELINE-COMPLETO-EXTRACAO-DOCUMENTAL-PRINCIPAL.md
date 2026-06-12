# Manual detalhado da etapa: Extracao documental principal do pipeline PDF

## 1. O que esta etapa faz

Esta etapa transforma bytes de PDF em um resultado estruturado de extracao. Ela e o primeiro ponto em que o documento deixa de ser um blob binario e passa a produzir texto, metadata e artefatos que o restante do pipeline pode usar.

Em linguagem simples: e a etapa que prova se o PDF e processavel de verdade.

## 2. Onde ela entra no fluxo

No codigo lido, ela esta centralizada em PdfExtractionApplicationService. O metodo build_from_storage chama extract_pdf_text, depois normaliza metadata e monta o PDFDocument resultante. Isso ocorre antes do pos-processamento textual, antes do OCR complementar e antes do multimodal.

## 3. O que entra e o que sai

Entradas confirmadas:

- StorageDocument com raw_bytes do PDF
- metadata inicial do documento
- resume state, quando a retomada esta habilitada

Saidas confirmadas:

- texto extraido
- summary de OCR documental
- resultado da engine de parsing aplicado na metadata
- manifesto operacional por estagio
- artefato de extracao persistido para retomada
- PDFDocument final com metadata normalizada

## 4. Como o codigo implementa a etapa

O fluxo real observado em extract_pdf_text segue esta ordem.

1. coleta raw_bytes e registra inicio da extracao;
2. carrega o estado de retomada;
3. se a retomada puder reutilizar artefato de extracao, restaura metadata_snapshot e extracted_text;
4. se nao houver retomada valida, monta PdfExtractionState com raw_bytes e texto vazio;
5. resolve o pipeline executavel de extracao;
6. executa as etapas ValidatePdfBytesStage, ApplyDocumentOcrStage, ParseViaEngineStage e ApplyEngineResultStage;
7. aplica o estado final de volta na metadata do documento;
8. persiste execution_manifest e artifact extraction_state;
9. grava checkpoint do pipeline;
10. devolve o texto extraido.

Depois disso, build_from_storage ainda faz duas coisas importantes.

- normaliza a metadata via PdfMetadataBuilder;
- tenta construir um domain_summary para prepend no conteudo do PDF quando esse resumo existir.

## 5. Decisoes tecnicas importantes

### 5.1. Retomada por artefato em vez de reprocessar tudo

Quando o resume state permite, o codigo tenta restaurar extraction_state antes de reexecutar o parsing. Isso reduz custo, evita repetir trabalho pesado e preserva coerencia em pipelines longos.

O risco controlado aqui e outro: se o artefato obrigatorio estiver ausente, a etapa falha explicitamente com PdfResumeArtifactMissingError. Isso evita uma retomada falsa.

### 5.2. Falha de extracao nao mata automaticamente a materializacao do documento

Em build_from_storage, se a extracao falhar, o codigo registra a excecao, marca pdf_extraction_failed na metadata, preserva sinais como contagem de paginas e segue com texto vazio.

Isso e uma decisao relevante: o pipeline continua observavel. O operador nao perde a historia do documento so porque a extracao falhou.

### 5.3. OCR documental fica dentro da extracao, nao do fluxo rico

O document-level OCR aparece como parte do pipeline de extracao. Isso faz sentido arquitetural. Ele atua sobre o arquivo inteiro antes do parsing principal, portanto pertence a esta etapa, nao ao OCR complementar posterior.

## 6. O que pode dar errado

Problemas confirmados ou diretamente sustentados pelo codigo:

- raw_bytes ausentes ou assinatura invalida de PDF interrompem o pipeline;
- artefato de retomada obrigatorio ausente dispara erro explicito;
- engine de parsing pode falhar ou devolver None;
- configuracao invalida do runtime de extracao vira ValueError explicito;
- OCR documental pode nao estar pronto ou nao encontrar infraestrutura necessaria.

Na pratica, esta etapa separa com clareza erro de documento, erro de runtime e erro de retomada.

## 7. Como diagnosticar

Os sinais mais uteis confirmados no codigo sao:

- event_name ingestion.pdf.extraction.start;
- event_name ingestion.pdf.extraction.completed;
- event_name ingestion.pdf.extraction.failed;
- event_name ingestion.pdf.extraction.resumed_from_artifact;
- document_ocr_summary no fechamento;
- stages_executed do pipeline de extracao;
- artifact extraction_state persistido na metadata operacional.

Em linguagem simples: se o PDF entrou, mas o texto saiu vazio, e aqui que voce precisa descobrir se o problema foi bytes, OCR documental, engine ou retomada.

## 8. Exemplo pratico guiado

Cenario: um PDF escaneado entra com resume desabilitado.

1. O servico registra inicio da extracao.
2. ValidatePdfBytesStage confirma que os bytes existem e comecam com %PDF.
3. ApplyDocumentOcrStage decide preprocessar o arquivo.
4. ParseViaEngineStage chama a engine final resolvida.
5. ApplyEngineResultStage injeta resultado de parsing e OCR na metadata.
6. O servico persiste o manifesto e o artefato extraction_state.
7. build_from_storage normaliza a metadata e monta o PDFDocument final.

Se a extracao falhar no meio, o documento continua carregando sinal explicito de falha em vez de passar silenciosamente como sucesso.

## 9. Os motores (engines) e as filas — explicação nível 101

Uma confusão comum: o sistema **não usa um único motor** para ler o PDF. Ele tem **filas separadas para tarefas diferentes**, e dentro de cada fila há uma ordem de tentativa. O YAML do tenant é a fonte de verdade dessas filas (ordem, ativação e combinação). São **três tarefas distintas**, e é normal ver vários nomes de motor aparecendo no mesmo processamento:

| Tarefa | O que faz | Quem faz (ex.: YAML DNIT de produção) |
|---|---|---|
| **1. Ler texto e estrutura** | Abre o PDF e tira o texto, as seções e as tabelas | **docling → pymupdf4llm → unstructured** (nessa ordem) |
| **2. OCR (foto → texto)** | Só para PDF escaneado: transforma a imagem de uma página em texto de verdade | ocrmypdf (documento inteiro) / tesseract, rapidocr, gemini (por página) |
| **3. Extrair tabelas (reforço)** | Especialista que pega **só tabelas** quando a etapa 1 não pegou bem | **tabula**, gmft, pdfplumber, camelot |

### O que é cada motor (e o que ele NÃO é)

- **Docling** — motor **completo** de texto/estrutura (texto + seções + tabelas). É o **titular** para os PDFs técnicos do DNIT.
- **pymupdf4llm** — também é um motor **completo** de texto, parecido com o Docling, mas entra como **2º da fila** (o reserva do Docling). Há ainda um primo, o `pymupdf` "nativo", outra variante completa.
- **unstructured** — terceiro motor de texto da fila (último reserva).
- **tabula** — **NÃO** é um motor de texto e **NÃO** faz OCR. É um **especialista em tabelas** (roda o `tabula-py`, baseado em Java) que devolve **apenas tabelas**, no formato `{"columns": [...], "rows": [...]}`. Entra como reforço da tarefa 3, e só quando ligado no YAML.
- **Motores de OCR** (ocrmypdf, tesseract, rapidocr, easyocr, gemini) — fila à parte, usada **só para PDF escaneado**: transformam imagem de página em texto. Não confundir com os motores de texto.

### Por que, "usando Docling", o PyMuPDF aparece?

Porque a fila da tarefa 1 é uma **cadeia de substitutos (fallback)**: o sistema tenta o **Docling primeiro**; se ele **falhar ou estourar o tempo** (acontece com PDFs grandes quando a CPU está sobrecarregada), o sistema **cai automaticamente para o pymupdf4llm**. Por isso, numa **mesma ingestão**, você vê o Docling **ser tentado** e o pymupdf4llm **assumir** — é **titular e reserva**, não competição à toa.

Regra de ouro do orquestrador (`CLAUDE.md §4`): **se um motor devolve resultado OK, o orquestrador NÃO aciona o próximo** — a fila para no primeiro sucesso. Adicionar ou reordenar motor é mudança de YAML; o orquestrador é neutro e não conhece regra por nome de engine.

### Como esses motores rodam isolados (subprocesso)

Os motores pesados que usam bibliotecas nativas perigosas (Docling, Tabula) rodam em **subprocesso** — uma "caixa" separada do processo principal, para que um travamento mate só a caixa, não a ingestão inteira. A supervisão dessa caixa (ligar, vigiar, cancelar, encerrar com segurança, timeout) é centralizada num **runner genérico único** compartilhado pelos motores (`pdf_subprocess_runner`), com a tradução do resultado no `pdf_parsing_result_codec`. Detalhe de reuso desse componente em `docs/tecnico/README-TOOLS-LIB.md`.

### Resumo de uma frase

**Docling e pymupdf4llm** = motores de **texto/estrutura** completos, na **mesma fila**, em ordem titular→reserva (por isso aparecem juntos); **Tabula** = **especialista em tabelas** (não é texto, não é OCR), reforço de outra fila; **OCR** = fila à parte, só para PDF escaneado.

## 10. Evidencias no codigo

- src/ingestion_layer/processors/pdf_extraction_application_service.py
  - Simbolo relevante: PdfExtractionApplicationService.extract_pdf_text
  - Comportamento confirmado: retomada por artefato, execucao do pipeline de extracao, persistencia de manifesto e artifact extraction_state.
- src/ingestion_layer/processors/pdf_extraction_application_service.py
  - Simbolo relevante: PdfExtractionApplicationService.build_from_storage
  - Comportamento confirmado: tolerancia observavel a falha de extracao, normalize_pdf_metadata e injecao opcional de domain_summary no conteudo final.
- src/ingestion_layer/processors/pdf_parsing_engine_resolver.py
  - Simbolo relevante: resolvedor que monta a fila deterministica de engines de parsing (a ordem vem do YAML).
  - Comportamento confirmado: a fila da tarefa 1 (texto/estrutura) e uma cadeia de fallback; para no primeiro motor que devolve OK.
- src/ingestion_layer/processors/pdf_table_service.py
  - Simbolo relevante: fallbacks de extracao de TABELA (tabula, gmft, pdfplumber, camelot), separados da fila de texto.
  - Comportamento confirmado: tabula entra so como reforco de tabela quando ligado no YAML; devolve apenas frames {columns, rows}.
- src/ingestion_layer/pdf_tools/pdf_subprocess_runner.py e src/ingestion_layer/pdf_tools/pdf_parsing_result_codec.py
  - Simbolo relevante: run_pdf_subprocess (supervisor generico de subprocesso) + reconstrucao do resultado.
  - Comportamento confirmado: hoje usado por Docling (one-shot) e Tabula (best-effort); o caminho nativo in-thread (pymupdf4llm/OCR por pagina) ainda NAO usa este runner.
