# Manual detalhado da etapa: Diagnostico e observabilidade do pipeline RAG

## 1. O que esta etapa faz

Esta etapa nao recupera documentos nem gera resposta, mas torna o pipeline investigavel. Ela coleta e monta os rastros que permitem descobrir se um problema veio da pergunta, da montagem do runtime, do rewrite, do roteamento, do retrieval, da ACL, do cache, da fusao ou da geracao final.

Em linguagem simples: e a diferenca entre um RAG que apenas responde e um RAG que pode ser auditado de ponta a ponta.

## 2. Quais camadas participam

No codigo lido, a observabilidade do pipeline aparece em varias camadas complementares.

- QuestionService registra logs de inicio e fim e consolida retrieval_metrics.
- QAQuestionProcessor cria telemetry recorder para a pergunta.
- IntelligentRAGOrchestrator registra passos do pipeline rag e generation.
- RetrievalEngine registra retrieval_trace por tentativa.
- PipelineDiagnosticsBuilder monta um resumo legivel para consumo externo.

O valor tecnico dessa composicao e nao depender de um unico log gigante para reconstruir a historia.

## 3. Retrieval trace

O retrieval trace e um dos blocos mais importantes do pipeline.

Cada tentativa de busca pode carregar, entre outros:

- attempt_id
- retriever
- processor
- strategy
- vector_store_type
- vectorstore_id
- connector
- query_preview
- top_k
- metadata_filters resumidos
- timestamp
- status
- duration_ms
- chunks_count
- top_score
- avg_score
- error_type e error_message quando houver falha

Em termos praticos, isso permite responder perguntas operacionais como estas.

- Quantos retrievers foram tentados?
- Qual deles retornou documentos?
- O FTS foi chamado?
- O cache foi hit ou miss?
- Houve erro no retriever ou apenas zero resultados?

## 4. Diagnostico estruturado para retorno de servico

PipelineDiagnosticsBuilder nao se limita a copiar logs. Ele organiza os dados em blocos semanticamente uteis.

Blocos confirmados no codigo:

- roteamento
- analise_query
- expansao_query
- bm25
- processadores_dominio
- resultado_retrieval
- controle_acesso

Isso importa porque suporte, QA e produto nao precisam ler apenas log cru. Eles recebem um retrato resumido do que aconteceu.

## 5. Metricas de retrieval para logging de fechamento

QuestionService chama extract_retrieval_metrics e inclui o resultado no log de conclusao da consulta. O builder extrai preview de documentos, scores, status de hybrid retry e top documents quando possivel.

Na pratica, isso deixa o fechamento da operacao muito mais util do que um simples success=true.

## 6. Decisoes tecnicas importantes

### 6.1. Observabilidade faz parte do produto, nao e adorno

O pipeline ja nasce preparado para expor confianca, strategy_used, documentos utilizados, denied_count e metricas de duracao. Isso mostra que o projeto trata diagnostico como requisito funcional.

### 6.2. O trace registra tentativa vazia e tentativa com erro como coisas diferentes

Esse detalhe e essencial. Nao encontrar documentos e diferente de quebrar o retriever. O codigo preserva essa diferenca.

### 6.3. ACL tambem aparece no diagnostico

Sem isso, um operador poderia concluir que o retriever falhou quando, na verdade, os documentos foram encontrados e depois bloqueados pelo controle de acesso.

## 7. O que pode dar errado

Riscos praticos que essa etapa ajuda a detectar:

- pergunta malformada ou vazia;
- pipeline moderno indisponivel;
- rewrite rejeitado por baixa similaridade;
- router escolhendo estrategia inesperada;
- retriever sem resultados;
- falha de FTS;
- cache devolvendo resultado reutilizado;
- ACL removendo todos os documentos;
- LLM lento ou indisponivel.

Sem observabilidade, todos esses sintomas tenderiam a virar a mesma reclamacao genérica: a IA respondeu mal.

## 8. Como investigar um incidente

Sequencia pratica sugerida com base no codigo:

1. Ler o log de inicio e fim em QuestionService.
2. Verificar pipeline_mode e motivo em QAQuestionProcessor.
3. Inspecionar routing_decision e query_analysis no payload final.
4. Ler retrieval_trace para entender quais tentativas aconteceram.
5. Confirmar se houve FTS, fusao, cache ou fallback.
6. Verificar access_control e denied_count.
7. Confirmar llm_generation_time e token_usage.

Em linguagem simples: o diagnostico correto segue a ordem do pipeline, nao um chute sobre o modelo.

## 9. Exemplo pratico guiado

Cenario: o usuario diz que a resposta veio vazia ou muito fraca.

1. O log final mostra success=true, mas denied_count alto.
2. retrieval_trace mostra que documentos foram encontrados.
3. PipelineDiagnosticsBuilder aponta controle_acesso com negados > 0.
4. A conclusao correta deixa de ser o modelo errou e passa a ser a ACL removeu o contexto util.

Esse tipo de explicacao so e possivel porque a observabilidade foi desenhada por etapa.

## 10. Evidencias no codigo

- src/services/question_service.py
  - Simbolo relevante: QuestionService.execute
  - Comportamento confirmado: logs de inicio e fim, telemetria e retrieval_metrics.
- src/services/question/pipeline_diagnostics_builder.py
  - Simbolo relevante: build_diagnostics e extract_retrieval_metrics
  - Comportamento confirmado: montagem de blocos de diagnostico estruturado.
- src/qa_layer/rag_engine/intelligent_orchestrator.py
  - Simbolo relevante: intelligent_retrieve e a rotina de fechamento do pipeline
  - Comportamento confirmado: registro de etapas do pipeline e anexacao de retrieval_trace.
- src/qa_layer/rag_engine/retrieval_engine.py
  - Simbolo relevante: rotinas de registro de tentativa, fechamento de tentativa e registro de erro de retrieval
  - Comportamento confirmado: trilha completa de tentativas de busca.
