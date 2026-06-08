# Manual detalhado da etapa: Analise estrutural por aba do pipeline Excel

## 1. O que esta etapa faz

Esta etapa tenta entender o papel de cada planilha antes de extrair conteudo como se toda folha fosse uma tabela limpa. Ela existe porque workbook real costuma misturar aba tabular, formulario, folha esparsa, aba mista e tabela nativa formal.

Em linguagem simples: e a etapa em que o runtime tenta descobrir o que cada aba realmente e.

## 2. Onde ela entra no fluxo

No codigo lido, essa inteligencia esta concentrada no ExcelSheetAnalyzer e e acionada a cada aba durante a extracao do workbook moderno ou do caminho legado por grade.

## 3. O que entra e o que sai

Entradas confirmadas:

- worksheet do openpyxl ou grade equivalente do xlrd
- analysis_config com limites e thresholds

Saidas confirmadas:

- ExcelSheetInfo com nome, indice, max_row, max_column, tables, data_density e structure_type
- ExcelTableInfo com intervalo, headers, tipos, row_count, column_count, source de deteccao e metadata nativa quando houver

## 4. Como o codigo implementa a etapa

O fluxo real observado combina tres eixos.

### 4.1. Densidade da aba

O analisador mede a proporcao de celulas nao vazias pela area observada. Esse sinal ajuda a distinguir folha quase vazia de folha operacionalmente densa.

### 4.2. Tabelas nativas

Quando a worksheet expõe tables, o analisador extrai o intervalo da tabela, resolve cabecalhos, identifica header_row_count e totals_row_count, coleta metadata de colunas, totais e formulas e produz ExcelTableInfo com detection_source igual a native_table.

### 4.3. Heuristica de grade

Se a tabela nativa nao existir, o analisador varre uma janela limitada de linhas e colunas procurando regioes que parecam tabela. Ele encontra fim de linha, fim de coluna, detecta cabecalho por predominancia de texto e infere tipos de coluna por amostras.

No fim, o analisador classifica a aba como sparse, form, table ou mixed com base em densidade e cobertura das tabelas detectadas.

## 5. Decisoes tecnicas importantes

### 5.1. Tabela nativa tem precedencia sobre heuristica

Isso e importante porque o workbook ja pode trazer a intencao estrutural pronta. Quando o runtime usa essa informacao, ele reduz risco de falso positivo heuristico.

### 5.2. A heuristica e deliberadamente limitada

O analisador usa limites configuraveis de linhas e colunas para inspecao. O ganho pratico e evitar custo excessivo e exploracao cega em planilhas enormes.

### 5.3. Estrutura da aba vira metadata operacional

O resultado da analise nao fica isolado num helper. Ele alimenta metadata do documento e influencia extracao, filtros de densidade e tabelas detectadas.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- uma aba real pode nao expor tabela nativa;
- a heuristica pode encontrar falso positivo em grades decorativas;
- densidade baixa pode classificar uma aba util como pouco valiosa;
- estrutura mixed mostra que nem sempre a folha cabe num modelo unico.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- data_density por aba;
- structure_type na metadata de cada sheet;
- tables_detected e lista de tabelas com to_dict;
- native_metadata quando a tabela veio do workbook;
- limites de analise e warnings de configuracao invalida para max_analysis_rows e max_analysis_cols.

Em linguagem simples: se a extracao ficou estranha ou a aba pareceu mal entendida, o ponto de investigacao principal e esta etapa.

## 8. Exemplo pratico guiado

Cenario: uma aba chamada Vendas possui uma tabela nativa do Excel com cabecalho e linha de totais.

1. o analisador mede a densidade da aba;
2. detecta ws.tables e extrai a tabela nativa;
3. registra headers, totals_row_count, colunas e formulas estruturadas;
4. monta ExcelTableInfo com detection_source native_table;
5. classifica a aba como table ou mixed, conforme a cobertura no workbook.

O valor desta etapa e preservar o maximo de intencao tabular real antes de transformar tudo em texto e chunks.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/excel_sheet_analyzer.py
  - Simbolo relevante: analyze_sheet_structure, extract_native_tables e build_sheet_info
  - Comportamento confirmado: densidade, tabela nativa e montagem do retrato estrutural da aba.
- src/ingestion_layer/processors/excel_sheet_analyzer.py
  - Simbolo relevante: analise heuristica de grade, deteccao de cabecalho, inferencia de tipos e classificacao de estrutura
  - Comportamento confirmado: fallback heuristico quando a tabela nativa nao existe e classificacao final sparse, form, table ou mixed.
