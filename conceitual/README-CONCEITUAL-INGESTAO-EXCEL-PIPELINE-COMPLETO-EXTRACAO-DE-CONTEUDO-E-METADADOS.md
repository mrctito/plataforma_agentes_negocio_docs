# Manual detalhado da etapa: Extracao de conteudo e metadados do pipeline Excel

## 1. O que esta etapa faz

Esta etapa transforma workbook e retrato estrutural por aba em um ExcelDocument rico o suficiente para leitura humana, chunking especializado e observabilidade posterior. Ela existe porque o pipeline Excel precisa preservar duas coisas ao mesmo tempo: legibilidade textual e estrutura tabular util.

Em linguagem simples: e a etapa que converte planilha em documento indexavel sem esmagar sua semantica principal.

## 2. Onde ela entra no fluxo

No codigo lido, a etapa principal acontece em build_from_storage e em _extract_text_and_metadata para .xlsx, com equivalente em _extract_xls_text_and_metadata para o caminho legado.

## 3. O que entra e o que sai

Entradas confirmadas:

- workbook carregado
- metadata inicial do StorageDocument
- sheet_info resolvido por aba
- configuracoes de extracao, output e schema

Saidas confirmadas:

- text_content do workbook por sheets
- metadata com sheets, schema summary, numeric stats e tables_data
- totals com sheet_names, total_sheets, total_rows e total_columns
- ExcelDocument final com tables_data e raw_data opcionais

## 4. Como o codigo implementa a etapa

O fluxo moderno observado segue esta ordem.

1. se o workbook for nulo, devolve conteudo e metadata minimos;
2. se o workbook for .xls, desvia para a trilha legada dedicada;
3. no caminho .xlsx, carrega tambem um workbook auxiliar para mapear tabelas nativas quando possivel;
4. percorre sheet por sheet respeitando max_sheets_total e excluded_sheet_names;
5. resolve sheet_info;
6. varre linhas ate max_rows_per_sheet;
7. detecta ou assume headers;
8. processa cada linha, preservando row_payload estruturado e linha textual formatada;
9. calcula schema summary e numeric stats por aba;
10. monta sheet_block textual com titulo da aba;
11. opcionalmente preserva tables_data e raw_data;
12. agrega metadata global e constroi o ExcelDocument final.

O caminho .xls tenta manter a mesma intencao funcional, mas com contrato estrutural mais pobre e sem leitura nativa de tabelas do workbook.

## 5. Decisoes tecnicas importantes

### 5.1. A extracao monta texto e estrutura ao mesmo tempo

O documento final nao nasce apenas como texto nem apenas como payload tabular. O slice preserva os dois lados para atender leitura, retrieval e futuros consumidores estruturados.

### 5.2. Cabecalho automatico e controlado por heuristica simples

Se auto_detect_headers estiver ativo, a primeira linha e testada por predominancia de texto. Quando nao parece cabecalho, o runtime gera nomes sinteticos e trata a primeira linha como dado. Isso evita perder informacao so porque a planilha nao veio perfeita.

### 5.3. Schema resumido e papis analiticos ja nascem aqui

O processor infere tipo dominante por coluna e ainda classifica o papel da coluna como identifier, metric, dimension ou time_dimension. O ganho pratico e enriquecer o acervo antes mesmo do chunking.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- auto-detect de cabecalho pode errar em planilhas hibridas;
- sheet de baixa densidade pode ser ignorada;
- o caminho .xls preserva menos estrutura do que .xlsx;
- tables_data e raw_data dependem de flags de output e podem nao existir.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- metadata.sheets por aba;
- excel_schema_summary e excel_numeric_stats;
- headers resolvidos em cada sheet;
- tables_data e raw_data quando habilitados;
- metadata de contrato adicionada depois, como excel_engine e excel_processing_mode.

Em linguagem simples: se o workbook entrou, mas o documento ficou pobre ou confuso, a explicacao quase sempre aparece nesta etapa.

## 8. Exemplo pratico guiado

Cenario: um workbook .xlsx com tres abas, sendo duas operacionais e uma aba auxiliar vazia.

1. o processor entra em _extract_text_and_metadata;
2. percorre as tres abas;
3. a aba vazia ou de baixa densidade e descartada;
4. nas abas operacionais, resolve cabecalhos e rows;
5. produz schema summary, numeric stats e, se habilitado, tables_data;
6. devolve um ExcelDocument contendo apenas as sheets relevantes.

O valor desta etapa e transformar o workbook em um documento fiel ao uso analitico, nao apenas a aparencia visual da planilha.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/excel_processor.py
  - Simbolo relevante: build_from_storage e _extract_text_and_metadata
  - Comportamento confirmado: montagem do ExcelDocument com texto, totais, sheets metadata, schema summary e tables_data.
- src/ingestion_layer/processors/excel_processor.py
  - Simbolo relevante: trilha de extracao legada .xls, processamento de linha, montagem do schema por aba e estatistica numerica por coluna
  - Comportamento confirmado: trilha legada .xls, preservacao de linhas estruturadas, inferencia de tipo e estatistica numerica por coluna.
