# Manual detalhado da etapa: Validacao de contrato do arquivo no pipeline Excel

## 1. O que esta etapa faz

Esta etapa impede que o pipeline Excel finja suportar formatos ou estados de arquivo que nao pertencem ao contrato atual. Ela existe para falhar cedo diante de extensao proibida, arquivo ausente, limite de tamanho excedido ou ausencia de bytes brutos.

Em linguagem simples: e a portaria tecnica que barra planilha invalida antes de abrir workbook.

## 2. Onde ela entra no fluxo

No codigo lido, esta etapa esta em validate_storage_document e e chamada por _validate_and_load_workbook antes da carga do workbook.

## 3. O que entra e o que sai

Entradas confirmadas:

- StorageDocument com file_path, source_type, file_size, raw_bytes e content_type
- configuracao excel.file_handling

Saidas confirmadas:

- autorizacao para carregar o workbook
- falha explicita para formatos fora do contrato
- falha explicita para arquivo ausente, grande demais ou sem bytes

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. exige file_path;
2. resolve a extensao real do arquivo;
3. rejeita formatos conhecidos como nao suportados, hoje .xlsm e .xlsb;
4. se houver allowlist configurada em supported_extensions, exige aderencia a ela;
5. para fontes locais e excel_file, confirma existencia do arquivo em disco;
6. se max_file_size_mb estiver configurado, compara com file_size;
7. exige raw_bytes obrigatoriamente.

O detalhe importante e que esta etapa mistura contrato funcional e contrato operacional. Ela valida tanto o tipo de formato aceito quanto a viabilidade pratica de processamento.

## 5. Decisoes tecnicas importantes

### 5.1. Rejeicao explicita de .xlsm e .xlsb

O codigo nao trata esses formatos como variantes quase suportadas. Ele os rejeita com motivo claro. Isso e importante porque evita prometer compatibilidade inexistente com macros VBA ou workbook binario.

### 5.2. Raw bytes sao obrigatorios

O processor nao aceita continuar apenas com um caminho de arquivo ou um conteudo textual derivado. O ganho pratico e garantir que a carga do workbook trabalhe sobre a representacao binaria real do arquivo.

### 5.3. Limite de tamanho fica no boundary do processor

Isso evita gastar custo de engine em planilhas fora do limite operacional definido pelo tenant ou pelo ambiente.

## 6. O que pode dar errado

Falhas confirmadas:

- documento sem file_path;
- formato .xlsm ou .xlsb;
- extensao fora da allowlist configurada;
- arquivo local inexistente;
- limite de tamanho excedido;
- documento sem raw_bytes.

Na pratica, esta etapa separa erro de contrato de erro de engine.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- mensagem explicita de formato Excel nao suportado;
- erro de arquivo nao encontrado;
- log de arquivo excedendo limite configurado;
- erro de planilha sem bytes.

Em linguagem simples: se a planilha nem chegou a abrir, e muito provavel que o bloqueio real esteja nesta etapa.

## 8. Exemplo pratico guiado

Cenario: um arquivo financeiro.xlsm e enviado para ingestao.

1. o processor resolve a extensao .xlsm;
2. consulta a lista de formatos conhecidos como fora do contrato;
3. rejeita o arquivo antes de tentar openpyxl ou xlrd;
4. devolve falha explicita dizendo que macros VBA nao sao suportadas.

O valor desta etapa e impedir ambiguidade operacional e diagnostico enganoso de erro de engine quando o problema real e de contrato.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/excel_processor.py
  - Simbolo relevante: validate_storage_document
  - Comportamento confirmado: validacao de extensao, allowlist, existencia fisica, limite de tamanho e bytes obrigatorios.
