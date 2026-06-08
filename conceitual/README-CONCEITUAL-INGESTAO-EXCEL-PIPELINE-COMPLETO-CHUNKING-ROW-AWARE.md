# Manual detalhado da etapa: Chunking row-aware do pipeline Excel

## 1. O que esta etapa faz

Esta etapa quebra o Excel em chunks por linha util, mantendo o contexto da aba e o retrato estrutural da linha. Ela existe porque planilha nao deve ser segmentada como texto corrido quando o valor principal esta na relacao entre colunas e valores da mesma linha.

Em linguagem simples: e a etapa que faz cada chunk continuar parecendo uma linha de planilha.

## 2. Onde ela entra no fluxo

No codigo lido, o caminho canonico da ingestao assíncrona esta na etapa assíncrona de chunking, que chama diretamente a fabrica row-aware de chunks.

## 3. O que entra e o que sai

Entradas confirmadas:

- ExcelDocument ja montado
- metadata.sheets
- excel_schema_summary indexado por sheet
- tables_data ou fallback textual

Saidas confirmadas:

- ContentChunk por linha util
- metadata por chunk com sheet_name, row_index, row_data, column_types, column_roles e numeric_columns
- serializacao textual da linha em JSON, markdown ou pares chave valor

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. _split_into_chunks recebe o documento e ignora o conteudo textual bruto para usar a forma especializada do Excel;
2. _create_row_aware_chunks percorre as sheets registradas em metadata;
3. monta um indice de schema por aba;
4. tenta extrair rows estruturadas primeiro de tables_data;
5. se nao houver dados estruturados, registra fallback e reconstrói linhas a partir do texto da aba;
6. para cada row_payload, serializa a linha conforme embed_as_json, markdown_tables ou pares chave valor;
7. monta metadata rica do chunk;
8. respeita o limite max_chunks_per_document.

O detalhe mais importante e que o valor do chunk esta tanto no content quanto na metadata. row_data, column_types e column_roles sao parte central do contrato do slice.

## 5. Decisoes tecnicas importantes

### 5.1. tables_data tem precedencia sobre fallback textual

Isso e importante porque a estrutura tabular montada na extracao e mais confiavel do que reconstruir linhas a partir do texto formatado.

### 5.2. Cada chunk carrega semantica de planilha, nao apenas texto

sheet_name, row_index, total_rows, column_names, column_roles e numeric_columns viajam junto com o chunk. O ganho pratico e permitir retrieval e consumo posterior muito mais ricos.

### 5.3. O formato textual do chunk e configuravel, mas o contrato estrutural permanece

Mesmo quando o content muda entre JSON, markdown e pares chave valor, a metadata estruturada continua existindo. Isso protege a utilidade do chunk contra variacoes de serializacao.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- ausencia de tables_data leva ao fallback textual, que e menos fiel;
- max_chunks_per_document pode truncar planilhas grandes;
- embed_as_json, markdown_tables e fallback textual mudam a forma do content e podem impactar consumidores downstream;
- se metadata.sheets estiver incompleta, parte do workbook pode nem chegar ao chunking.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- logs de fallback para extracao textual por sheet;
- metadata dos chunks com row_data e column_roles;
- discrepancia entre row_data e content serializado;
- quantidade final de chunks em relacao ao numero de linhas uteis.

Em linguagem simples: se o Excel entrou, mas a recuperacao posterior ficou pobre, esta etapa e uma das primeiras que merecem auditoria.

## 8. Exemplo pratico guiado

Cenario: uma aba Produtos tem 500 linhas uteis e tables_data estruturado disponivel.

1. _create_row_aware_chunks localiza a aba em metadata.sheets;
2. extrai as rows de tables_data sem precisar fallback textual;
3. para cada linha, serializa o payload conforme a configuracao;
4. anexa metadata estrutural da linha;
5. gera um chunk por linha ate o limite configurado.

O valor desta etapa e manter a unidade logica da planilha, que e a linha com seus campos, e nao um bloco arbitrario de texto.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/excel_processor.py
  - Simbolo relevante: etapa assíncrona de chunking e fabrica row-aware de chunks
  - Comportamento confirmado: caminho canonico row-aware da ingestao assíncrona.
- src/ingestion_layer/processors/excel_processor.py
  - Simbolo relevante: _extract_rows_for_sheet, _extract_rows_from_text, _serialize_row e _build_chunk_metadata
  - Comportamento confirmado: precedencia de dados estruturados, fallback textual e metadata rica por chunk.
