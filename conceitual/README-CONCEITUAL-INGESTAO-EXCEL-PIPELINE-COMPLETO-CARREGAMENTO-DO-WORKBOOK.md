# Manual detalhado da etapa: Carregamento do workbook do pipeline Excel

## 1. O que esta etapa faz

Esta etapa abre o arquivo Excel com a engine adequada e escolhe conscientemente entre o caminho moderno e o caminho legado. Ela existe porque o valor do slice Excel depende de ler workbook como workbook, e nao apenas como arquivo zipado, blob binario ou texto tabulado.

Em linguagem simples: e a etapa que transforma bytes validos em workbook operacional.

## 2. Onde ela entra no fluxo

No codigo lido, a decisao mora em _load_workbook, com fallback explicito para _load_xls_workbook quando o arquivo e .xls ou foi classificado como EXCEL_XLS.

## 3. O que entra e o que sai

Entradas confirmadas:

- raw_bytes do StorageDocument
- extensao efetiva do arquivo
- flags read_only e data_only

Saidas confirmadas:

- workbook aberto por openpyxl para caminho principal
- workbook aberto por xlrd em modo best_effort_xls para legado .xls
- falha explicita quando nenhum caminho consegue abrir o arquivo

## 4. Como o codigo implementa a etapa

O fluxo real observado e este.

1. confirma que raw_bytes existem;
2. resolve a extensao do arquivo;
3. tenta openpyxl.load_workbook com BytesIO, read_only e data_only vindos da configuracao;
4. se openpyxl falhar e o arquivo for .xls, registra log e tenta o caminho legado via xlrd;
5. se xlrd abrir, registra processing_mode best_effort_xls e limitações operacionais;
6. se falhar tudo, levanta ExcelWorkbookLoadError.

O ponto importante e que o fallback nao e implicito para qualquer erro em qualquer formato. Ele existe apenas para o formato legado .xls, que e uma escolha de compatibilidade consciente do runtime.

## 5. Decisoes tecnicas importantes

### 5.1. openpyxl e o caminho principal do slice

Isso alinha o pipeline ao formato openxml moderno e preserva acesso mais rico a estrutura do workbook.

### 5.2. O caminho .xls e explicitamente best-effort

O codigo nao esconde as limitacoes do formato legado. Ao contrario, registra um processing_mode proprio e lista limitacoes estruturais. Isso melhora governanca e observabilidade.

### 5.3. data_only privilegia valor sobre formula

Quando ativo, o runtime prioriza o valor calculado armazenado na planilha, e nao a formula como expressao executavel. O ganho pratico e estabilizar a ingestao e manter foco em consulta do dado.

## 6. O que pode dar errado

Falhas e limites confirmados:

- raw_bytes ausentes impedem qualquer carga;
- openpyxl pode falhar em workbook invalido ou zip quebrado;
- xlrd pode nao estar disponivel no ambiente para .xls;
- .xls protegido por senha e rejeitado;
- o caminho legado tem limitacoes fortes de recurso e fidelidade.

## 7. Como diagnosticar

Os sinais mais uteis desta etapa sao:

- log de falha ao carregar com openpyxl;
- log de tentativa do caminho legado .xls;
- log de pipeline em modo best_effort_xls com lista de limitacoes;
- ExcelWorkbookLoadError final quando nenhuma carga foi possivel.

Em linguagem simples: se o arquivo passou na portaria mas nao virou workbook, a explicacao real mora aqui.

## 8. Exemplo pratico guiado

Cenario: um arquivo legado estoque-2018.xls entra no pipeline.

1. a validacao do contrato passa;
2. _load_workbook tenta openpyxl e falha;
3. como a extensao e .xls, o runtime nao aborta de imediato;
4. aciona _load_xls_workbook via xlrd;
5. registra modo best_effort_xls e suas limitacoes;
6. segue para extracao e analise estrutural com contrato reduzido.

O valor desta etapa e manter compatibilidade com legado sem fingir equivalencia total com .xlsx.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/excel_processor.py
  - Simbolo relevante: _load_workbook e _load_xls_workbook
  - Comportamento confirmado: caminho principal com openpyxl, fallback explicito para .xls via xlrd e exposicao de limitacoes do modo legado.
