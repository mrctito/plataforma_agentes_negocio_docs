# Manual detalhado da etapa: Indexacao e persistencia comuns do pipeline JSON

## 1. O que esta etapa faz

Esta etapa faz o slice JSON convergir para a esteira comum da ingestao. Depois que o processor terminou de produzir chunks, o runtime padrao injeta metadata canonica, indexa no vector store, persiste o documento processado e registra telemetria.

Em linguagem simples: e o momento em que o JSON sai do slice especializado e entra no acervo comum da plataforma.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa esta centralizada em DocumentIndexingExecutor.finalize. Ela vem depois da materializacao do documento e do chunking estrutural.

## 3. O que entra e o que sai

Entradas confirmadas:

- ChunkedFilePipelineDocument com storage_document, prepared_document e chunks
- request com vector_store, vectorstore_id, telemetry_run_id, tenant_code e source_system

Saidas confirmadas:

- chunks indexados
- documento persistido
- telemetria de sucesso registrada
- FilePipelineExecutionResult com chunks e processed_info

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. resolve canonical_source_uri;
2. calcula document_hash e content_hash;
3. monta processed_info com file_path, title, hashes, content_type e source_system;
4. extrai metadata para telemetria e avalia skip, quando houver;
5. deriva canonical_source_key e document_version_id;
6. enriquece cada chunk com vectorstore_id, source_file, correlation_id, content_type, source, document_hash, content_hash, document_id, canonical_source_key e document_version_id;
7. chama index_chunks no vector store;
8. persiste o documento processado;
9. registra telemetria de sucesso;
10. devolve FilePipelineExecutionResult.

O ponto importante e que, para JSON, nao existe um fechamento paralelo especial. O slice especializado precisa respeitar o mesmo contrato operacional do restante da ingestao.

## 5. Decisoes tecnicas importantes

### 5.1. O slice JSON nao inventa storage proprio

Depois do processamento, o JSON usa exatamente a esteira comum. Isso reduz acoplamento e evita dois modelos de persistencia para o mesmo produto.

### 5.2. Metadata canonica entra antes da indexacao

Os chunks recebem document_hash, content_hash, canonical_source_key e document_version_id antes de index_chunks. O ganho pratico e indexar ja com identidade forte e rastreabilidade completa.

### 5.3. Falha de telemetria nao invalida o documento processado

Se a telemetria de sucesso falhar, o codigo registra a excecao, mas o documento continua considerado processado. Isso mostra maturidade operacional na separacao entre dado persistido e observabilidade.

## 6. O que pode dar errado

Falhas confirmadas ou implicadas:

- avaliacao de telemetria pode mandar skip e abortar o fechamento;
- canonical_source_key pode nao ser derivavel;
- vector store pode recusar os chunks;
- persistencia do documento pode falhar;
- telemetria de sucesso pode falhar sem invalidar o processamento ja concluido.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- logs de documento ignorado pela telemetria;
- logs de recusa do vector store;
- logs de falha ao persistir documento processado;
- logs de falha na telemetria de sucesso com o documento ainda considerado processado;
- metadata canonica presente nos chunks persistidos.

Em linguagem simples: se o JSON foi processado, mas nao apareceu corretamente no acervo, esta etapa mostra se o problema foi indexacao, persistencia ou so telemetria.

## 8. Exemplo pratico guiado

Cenario: o processor JSON terminou com 120 chunks de um array de produtos.

1. O executor calcula hashes e chaves canonicas.
2. Enriquece os 120 chunks com metadata comum.
3. O vector store indexa esses chunks.
4. O documento processado e persistido.
5. A telemetria de sucesso e registrada.
6. O resultado volta como FilePipelineExecutionResult.

O valor desta etapa e transformar o trabalho especializado do slice JSON em acervo operacional real e rastreavel.

## 9. Evidencias no codigo

- src/ingestion_layer/file_pipeline_services.py
  - Simbolo relevante: DocumentIndexingExecutor.finalize
  - Comportamento confirmado: enriquecimento canonico dos chunks, indexacao no vector store, persistencia do documento e telemetria final.
- src/ingestion_layer/clients/json_client.py
  - Simbolo relevante: JsonFileClient
  - Comportamento confirmado: cliente fino que delega o processamento JSON ao builder e ao processor antes da convergencia com a esteira comum.
