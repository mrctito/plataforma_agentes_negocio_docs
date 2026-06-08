# Manual detalhado da etapa: Fechamento na esteira comum do pipeline Excel

## 1. O que esta etapa faz

Esta etapa pega o resultado especializado do slice Excel e o incorpora ao acervo operacional comum da plataforma. Ela existe para evitar persistencia paralela por tipo de arquivo e para garantir que planilha siga o mesmo contrato canonico de hashes, identidade, indexacao e telemetria dos demais documentos.

Em linguagem simples: e a etapa em que a planilha deixa de ser apenas "processada" e passa a existir de fato no acervo do produto.

## 2. Onde ela entra no fluxo

No codigo lido, esta etapa esta centralizada em DocumentIndexingExecutor.finalize, depois que o processor Excel ja produziu seus chunks row-aware.

## 3. O que entra e o que sai

Entradas confirmadas:

- documento chunked pela esteira
- storage_document original
- request com vector_store, vectorstore_id, telemetry_run_id e tenant_code

Saidas confirmadas:

- chunks enriquecidos com metadata canonica
- chunks indexados no vector store
- documento persistido
- telemetria de sucesso registrada

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. resolve canonical_source_uri;
2. calcula document_hash e content_hash;
3. monta processed_info do documento;
4. extrai metadata para telemetria e avalia skip;
5. deriva canonical_source_key e document_version_id;
6. injeta metadata canonica em cada chunk, como vectorstore_id, source_file, correlation_id, content_type, source, hashes e ids;
7. chama index_chunks no vector store;
8. persiste o documento processado;
9. registra telemetria de sucesso.

O ponto importante e que o slice Excel nao cria uma persistencia especial para workbook. Ele converge para a mesma esteira comum usada pelos outros tipos.

## 5. Decisoes tecnicas importantes

### 5.1. Especializacao termina antes da persistencia

O processor Excel concentra a inteligencia tabular, mas a indexacao e persistencia permanecem canonicas. O ganho pratico e evitar ilha arquitetural por formato.

### 5.2. Os chunks recebem identidade forte antes da indexacao

document_hash, content_hash, canonical_source_key e document_version_id entram antes do vector store. Isso melhora rastreabilidade, reprocessamento e auditoria.

### 5.3. Telemetria de sucesso e separada da persistencia do documento

Se a telemetria falhar, o documento pode continuar considerado processado. Essa separacao evita transformar falha de observabilidade em perda de dado ja persistido.

## 6. O que pode dar errado

Falhas confirmadas ou implicadas:

- avaliacao de telemetria pode mandar skip;
- vector store pode recusar os chunks;
- persistencia do documento pode falhar;
- telemetria de sucesso pode falhar sem invalidar o documento persistido.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- log de documento ignorado pela telemetria;
- log de recusa do vector store;
- log de falha ao persistir documento processado;
- log de falha ao registrar telemetria de sucesso.

Em linguagem simples: se a planilha foi processada, mas nao apareceu corretamente no acervo, a investigacao real quase sempre termina aqui.

## 8. Exemplo pratico guiado

Cenario: o workbook foi processado em 1200 chunks row-aware.

1. o executor calcula hashes do documento;
2. injeta metadata canonica nos 1200 chunks;
3. o vector store indexa os chunks;
4. o documento e persistido com sua identidade canonica;
5. a telemetria de sucesso fecha o ciclo operacional.

O valor desta etapa e transformar a inteligencia do slice Excel em ativo operacional duravel, consultavel e auditavel.

## 9. Evidencias no codigo

- src/ingestion_layer/file_pipeline_services.py
  - Simbolo relevante: DocumentIndexingExecutor.finalize
  - Comportamento confirmado: enriquecimento canonico dos chunks, indexacao no vector store, persistencia e telemetria final.
