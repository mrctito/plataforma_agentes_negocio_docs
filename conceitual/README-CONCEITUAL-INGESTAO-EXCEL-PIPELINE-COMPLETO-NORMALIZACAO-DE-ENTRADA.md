# Manual detalhado da etapa: Normalizacao de entrada do pipeline Excel

## 1. O que esta etapa faz

Esta etapa decide quando um arquivo deve entrar no slice Excel e com qual tipo de conteudo ele sera tratado dentro da esteira. Ela existe para impedir duas perdas de governanca muito comuns em ingestao tabular: tratar planilha como texto generico e deixar a especializacao do processor depender de inferencia fraca mais adiante.

Em linguagem simples: e a etapa que diz "isso e Excel de verdade dentro do runtime" antes de qualquer tentativa de abrir o workbook.

## 2. Onde ela entra no fluxo

No codigo lido, a entrada canonica do slice Excel nasce em duas decisoes combinadas.

1. FileSystemDataSource infere a extensao .xlsx como EXCEL_XLSX e .xls como EXCEL_XLS.
2. A factory de processors resolve ExcelContentProcessor para os dois tipos.

O resultado pratico e que a planilha ja chega ao processor com um content_type especializado, em vez de cair primeiro num caminho textual e depois tentar ser reinterpretada.

## 3. O que entra e o que sai

Entradas confirmadas:

- caminho do arquivo
- extensao do arquivo
- bytes materializados pelo data source

Saidas confirmadas:

- StorageDocument com content_type especializado para Excel
- roteamento para ExcelContentProcessor
- separacao entre caminho moderno .xlsx e legado .xls

## 4. Como o codigo implementa a etapa

O fluxo real observado e este.

1. o data source local recebe o caminho do arquivo;
2. a inferencia de tipo olha a extensao antes de qualquer parse do workbook;
3. .xlsx vira EXCEL_XLSX;
4. .xls vira EXCEL_XLS;
5. a factory de processors registra o mesmo ExcelContentProcessor para os dois tipos;
6. quando o document processor executor aciona o processor, o slice Excel ja sabe se esta num caminho openxml ou num caminho legado.

O detalhe importante e que o desenho separa identificacao do tipo e especializacao do processamento. Isso melhora previsibilidade e evita ramificacoes ad hoc mais tarde.

## 5. Decisoes tecnicas importantes

### 5.1. O tipo de conteudo do Excel e especializado logo no boundary de entrada

Isso evita que o arquivo seja tratado como texto, markdown ou binario generico antes de chegar ao processor correto. O ganho pratico e impedir perda precoce de semantica tabular.

### 5.2. .xlsx e .xls entram como tipos diferentes

O runtime nao finge paridade completa entre formatos. Ele registra dois content types distintos e deixa o processor decidir as diferencas operacionais depois. Isso melhora clareza arquitetural.

### 5.3. A especializacao nasce no processor, nao num client Excel dedicado

Diferente de alguns slices que passam por clients finos dedicados, o caminho Excel se ancora na combinacao data source mais processor factory. O efeito pratico e reduzir camadas extras no boundary local.

## 6. O que pode dar errado

Os riscos e limites confirmados ou implicados sao estes.

- extensao incorreta faz o arquivo cair em outro slice;
- identificacao por extensao nao garante que o binario interno seja um workbook valido;
- caminhos que entrem por outra superficie precisam manter a mesma resolucao de content_type para nao criar drift.

## 7. Como diagnosticar

Os sinais mais uteis desta etapa sao:

- content_type do documento antes do processor;
- logs e traces que mostrem se o tipo foi EXCEL_XLSX ou EXCEL_XLS;
- comportamento posterior de openpyxl ou xlrd, que denuncia se a normalizacao inicial escolheu o slice certo.

Em linguagem simples: se um Excel foi parar num processor errado, a causa raiz quase sempre esta aqui.

## 8. Exemplo pratico guiado

Cenario: um arquivo vendas-mensais.xlsx entra pelo data source local.

1. o FileSystemDataSource olha a extensao .xlsx;
2. classifica o documento como EXCEL_XLSX;
3. a factory resolve ExcelContentProcessor;
4. o slice segue para validacao do contrato do arquivo e carga do workbook.

O valor desta etapa e impedir que uma planilha perca sua identidade antes de o runtime especializado comecar.

## 9. Evidencias no codigo

- src/ingestion_layer/datasources/filesystem_data_source.py
  - Simbolo relevante: inferencia de tipo por extensao
  - Comportamento confirmado: .xlsx vira EXCEL_XLSX e .xls vira EXCEL_XLS.
- src/ingestion_layer/core/factories.py
  - Simbolo relevante: registro lazy de processors
  - Comportamento confirmado: ExcelContentProcessor atende os dois content types de Excel.
