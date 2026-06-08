# Manual detalhado da etapa: Recuperacao especializada do pipeline RAG

## 1. O que esta etapa faz

Esta etapa executa a estrategia escolhida pelo roteamento. Ela e a parte do pipeline que efetivamente sai para buscar evidencia, mas faz isso de forma especializada por tipo de consulta e por tipo de processador.

Em linguagem simples: se a etapa anterior decidiu como procurar, esta etapa faz a procura real.

## 2. Como o orchestrator organiza a execucao

No IntelligentRAGOrchestrator, a execucao passa por um registry explicito de estrategias. Cada ProcessorType e ligado a um executor canonico.

Os executores confirmados no codigo cobrem:

- JSON_TOOLKIT
- HYBRID_SEARCH
- SELF_QUERY
- MULTI_QUERY
- TRADITIONAL_RAG
- API_RETRIEVER

O uso de StrategyExecutorRegistry evita ifs espalhados no orchestrator e deixa a etapa de retrieval mais modular.

## 3. Caminhos especializados confirmados

### 3.1. JSON e Excel especializado

RetrievalEngine.execute_json_processor tenta primeiro o caminho JSONSpecializedRAGExcel quando ele esta habilitado. Esse caminho coleta documentos tabulares, materializa datasets e pode responder deterministamente certas perguntas antes de cair para o agente generativo JSON.

O ponto tecnico mais importante aqui e que completude da ingestao tabular nao e tratada como detalhe cosmético. Se require_exhaustive_ingestion estiver ativo e a coleta disponivel nao garantir exaustividade, o codigo levanta ExcelIngestionCompletenessError.

Em linguagem simples: planilha incompleta nao vira resposta aparentemente confiavel.

### 3.2. Busca hibrida

RetrievalEngine.execute_hybrid_processor diferencia tres cenarios.

- hybrid desligado, que cai para busca tradicional;
- hybrid nativo, quando o vector store suporta search_hybrid;
- hybrid manual, quando o retriever especializado precisa ser usado.

O codigo tambem enriquece a query hibrida com termos tecnicos detectados, o que ajuda consultas que dependem de vocabulario exato de dominio.

### 3.3. Self-query

RetrievalEngine.execute_self_query_processor tenta primeiro o resolvedor de dominio quando ele esta habilitado e detecta busca estruturada. Se isso nao funcionar, usa o retriever self_query generico. Se nenhum estiver disponivel, o fallback e a busca tradicional.

### 3.4. Multi-query

RetrievalEngine.execute_multi_query_processor usa o retriever multi_query configurado. Se ele nao existir, mas houver base_retriever e LLM, cria um MultiQueryRetriever temporario. Se nem isso for possivel, volta para o caminho tradicional.

### 3.5. Busca tradicional

RetrievalEngine.execute_traditional_processor percorre a ordem de preferencia vector_search, semantic_search e default. Depois disso, ainda pode enriquecer a resposta com FTS.

## 4. Decisoes tecnicas importantes

### 4.1. Fallback existe, mas e explicito e por processador

O codigo nao trata todos os erros do mesmo jeito. Cada processador tenta sua rota principal, registra problema quando falha e usa um fallback conhecido. Isso cria resiliencia sem esconder qual caminho realmente venceu.

### 4.2. Retry externo protege recursos concretos

Na busca hibrida nativa, a chamada ao vector store e envolvida por run_with_external_retry. Isso confirma o padrao do projeto de tratar integrações externas com retry centralizado.

### 4.3. Tabular nao e tratado como texto qualquer

O caminho JSONSpecializedRAGExcel e a prova pratica de que o projeto separa consultas sobre planilhas de retrieval textual comum. Esse e um diferencial tecnico relevante.

## 5. O que entra e o que sai

Entradas:

- RoutingDecision
- effective_question
- top_k
- metadata_filters
- telemetry
- trace_log

Saida:

- lista de documentos ou estruturas equivalentes que alimentarao pos-retrieval e geracao

## 6. O que pode dar errado

Falhas e comportamentos confirmados:

- JSONSpecializedRAGExcel pode falhar por completude insuficiente da ingestao.
- hybrid retriever ausente dispara fallback para tradicional.
- self-query ausente dispara fallback para tradicional.
- multi-query ausente tambem cai para tradicional.
- nenhum retriever tradicional disponivel gera ContentQAError.

Em termos praticos, a etapa prefere um fallback nomeado ou um erro claro, nunca silencio.

## 7. Como diagnosticar

Sinais uteis confirmados:

- logs de decisao de hybrid nativo ou manual;
- trace de retrieval com retriever_label, strategy, duration_ms, top_score e chunks_count;
- logs especificos de ingestao Excel e de fallback do JSON Agent;
- metrics de processor_usage no pipeline.

## 8. Exemplo pratico guiado

Cenario: o usuario pergunta sobre uma planilha Excel materializada.

1. O roteamento identifica caminho JSON ou tabular.
2. execute_json_processor tenta JSONSpecializedRAGExcel.
3. O componente carrega linhas, valida completude e monta datasets.
4. Se a pergunta couber no caminho deterministico, responde sem improviso generativo.
5. Se nao couber e houver LLM, o JSON Agent entra como fallback generativo.

O valor pratico e tratar planilha como dado estruturado, nao como texto solto com embedding.

## 9. Evidencias no codigo

- src/qa_layer/rag_engine/intelligent_orchestrator.py
  - Simbolo relevante: _get_strategy_executor_registry e _execute_routing_decision
  - Comportamento confirmado: registry explicito de executores por ProcessorType.
- src/qa_layer/rag_engine/retrieval_engine.py
  - Simbolo relevante: execute_json_processor, execute_hybrid_processor, execute_self_query_processor, execute_multi_query_processor e execute_traditional_processor
  - Comportamento confirmado: multiplos caminhos especializados com fallback controlado.
- src/qa_layer/json_rag/specialized_rag_excel.py
  - Simbolo relevante: JSONSpecializedRAGExcel.ask_question e o metodo interno de extracao de documentos Excel
  - Comportamento confirmado: coleta tabular, caminho deterministico e erro explicito de completude.
