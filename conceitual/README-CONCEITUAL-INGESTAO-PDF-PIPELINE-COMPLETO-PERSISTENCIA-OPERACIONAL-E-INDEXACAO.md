# Manual detalhado da etapa: Persistencia operacional e indexacao do pipeline PDF

## 1. O que esta etapa faz

Esta etapa transforma o resultado do processor PDF em acervo operacional de verdade. Ela fecha o pipeline: injeta metadata canonica nos chunks, indexa no vector store, persiste o documento processado, registra telemetria de sucesso e atualiza estatisticas consolidadas do slice PDF.

Em linguagem simples: e o momento em que o PDF deixa de ser apenas processado e passa a existir como ativo consultavel da plataforma.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa esta centralizada em DocumentIndexingExecutor.finalize. Ela acontece depois que o processor ja devolveu o documento preparado e os chunks finais. Portanto, ela nao decide parsing nem chunking; ela fecha o resultado desses passos.

## 3. O que entra e o que sai

Entradas confirmadas:

- ChunkedFilePipelineDocument com prepared_document, storage_document e chunks
- request com vector_store, telemetry_run_id, source_system, vectorstore_id e stats_collector

Saidas confirmadas:

- chunks indexados no vector store
- documento persistido com hashes e chaves canonicas
- telemetria de sucesso registrada
- FilePipelineExecutionResult com chunks e processed_info

## 4. Como o codigo implementa a etapa

O fluxo real observado em finalize segue esta ordem.

1. resolve a identidade canonica da origem do documento;
2. calcula document_hash e content_hash;
3. monta processed_info com dados basicos do documento;
4. se o conteudo for PDF, constroi pdf_runtime_summary e preenche strategy_used e library_used quando estiverem vazios;
5. extrai metadata para telemetria e avalia se o documento deve ser ignorado por skip de telemetria;
6. deriva canonical_source_key e document_version_id;
7. injeta metadata canonica em cada chunk, incluindo vectorstore_id, correlation_id, document_hash e document_version_id;
8. chama index_chunks no vector store;
9. se a indexacao falhar, registra erro e interrompe o fechamento;
10. persiste o documento processado;
11. registra telemetria de sucesso;
12. atualiza ETA e estatisticas consolidadas do PDF quando aplicavel;
13. devolve FilePipelineExecutionResult com chunks e processed_info.

O ponto estrutural importante e que a indexacao vem antes da persistencia final do documento processado. Isso reforca a visao de que o acervo indexado e parte do contrato de sucesso desta etapa.

## 5. Decisoes tecnicas importantes

### 5.1. Hashes e chaves canonicas sao parte do fechamento, nao detalhe de storage

O executor calcula document_hash, content_hash, canonical_source_key e document_version_id antes de enriquecer os chunks. Isso importa porque a plataforma quer rastreabilidade, deduplicacao operacional e identidade consistente do documento.

### 5.2. PDF ganha resumo operacional proprio no fechamento

Quando o content_type e PDF, o executor chama o builder do resumo operacional do runtime. Se strategy_used e library_used ainda estiverem vazios na metadata do documento, ele completa esses campos com o resumo montado.

Na pratica, isso garante que a trilha operacional do PDF continue inteligivel ate no fim da esteira comum.

### 5.3. Falha de telemetria nao apaga o sucesso do documento

O codigo trata um caso importante com maturidade operacional: se o registro da telemetria de sucesso falhar, o documento continua sendo considerado processado. A excecao fica registrada, mas o acervo nao e descartado retroativamente por causa disso.

## 6. O que pode dar errado

Falhas confirmadas ou diretamente implicadas pelo codigo:

- documento pode ser ignorado pela avaliacao de telemetria quando skip estiver ativo;
- canonical_source_key pode nao ser derivavel, o que vira IngestionPersistenceError;
- vector store pode recusar os chunks;
- persist_document pode falhar;
- telemetria de sucesso pode falhar, embora o documento siga processado.

Em termos prativos, esta etapa separa falha de indexacao, falha de persistencia e falha de telemetria, o que melhora muito o troubleshooting.

## 7. Como diagnosticar

Os sinais mais uteis confirmados no codigo sao:

- logs de recusa do vector store para os chunks;
- presence de pdf_runtime_summary em processed_info;
- strategy_used e library_used preenchidos no documento PDF;
- logs de "Arquivo processado e indexado" com chunks e, quando houver, ETA do PDF;
- logs de falha ao persistir documento processado;
- logs de falha na telemetria de sucesso sem perda do status de processamento.

Em linguagem simples: se o PDF foi processado, mas nao apareceu corretamente no acervo, esta etapa explica se o problema foi indexacao, persistencia ou apenas observabilidade.

## 8. Exemplo pratico guiado

Cenario: o processor PDF terminou com chunks validos e metadata operacional consistente.

1. O executor calcula hashes e chaves canonicas.
2. Enriquece todos os chunks com metadata comum.
3. O vector store indexa os chunks.
4. O documento processado e persistido.
5. A telemetria de sucesso e registrada.
6. O slice PDF atualiza estatisticas consolidadas e ETA, quando aplicavel.
7. O resultado final volta como FilePipelineExecutionResult.

O valor desta etapa e fechar a historia do documento de ponta a ponta, sem deixar o PDF pela metade entre processamento e acervo.

## 9. Evidencias no codigo

- src/ingestion_layer/file_pipeline_services.py
  - Simbolo relevante: DocumentIndexingExecutor.finalize
  - Comportamento confirmado: enriquecimento canonico dos chunks, indexacao no vector store, persistencia do documento e telemetria final.
- src/ingestion_layer/file_pipeline_services.py
  - Simbolo relevante: o trecho de fechamento especifico para PDF dentro do executor
  - Comportamento confirmado: construcao de pdf_runtime_summary, preenchimento de strategy_used e library_used e atualizacao de estatisticas consolidadas do slice PDF.
