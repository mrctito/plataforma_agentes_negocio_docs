# Manual detalhado da etapa: Analise e roteamento do pipeline RAG

## 1. O que esta etapa faz

Esta etapa tenta entender o tipo de pergunta e decidir a estrategia de retrieval mais adequada. Ela existe porque uma plataforma RAG corporativa nao pode tratar da mesma forma uma pergunta conceitual, uma busca por codigo exato, uma consulta com filtro estruturado e uma pergunta tabular.

Em linguagem simples: aqui o sistema decide como procurar, nao apenas o que responder.

## 2. Componentes principais

Dois componentes sustentam a etapa.

- QueryAnalyzer, que extrai features semanticas da pergunta.
- AdaptiveQueryRouter, que transforma essas features em estrategia de retrieval com confianca e fallback.

No orchestrator, a decisao final e consolidada por _analyze_and_route_query, que delega ao RoutingDecisionMaker apoiado por esses componentes.

## 3. O que o analisador extrai

QueryAnalyzer produz QueryFeatures com estes blocos principais.

- query_type
- data_type
- domain
- original_query e cleaned_query
- complexity e confidence
- entities e keywords
- requires_filters, requires_temporal e requires_real_time
- suggested_processors
- intent, context_hints e technical_terms

Isso mostra que a analise nao se limita a uma classificacao unica. Ela tenta montar um retrato util da pergunta para o router.

## 4. Como o codigo detecta sinais

O analisador compila regex para detectar, entre outros:

- codigos tecnicos como DNIT, NBR, ABNT e ISO;
- perguntas procedurais;
- perguntas factuais;
- perguntas conceituais;
- sinais de dados estruturados;
- sinais de documentos tecnicos;
- temporalidade;
- comparacao.

Tambem carrega uma base de conhecimento com entidades conhecidas, areas tecnicas, termos de projeto e status operacionais. Na pratica, isso permite que o pipeline perceba mais do que a similaridade vetorial da frase.

## 5. Como o router decide

AdaptiveQueryRouter faz tres movimentos principais.

1. Analisa caracteristicas como has_exact_codes, has_technical_terms, has_structured_filters e has_conceptual_terms.
2. Calcula complexidade.
3. Determina estrategia, confianca e fallback.

As estrategias confirmadas no codigo sao estas.

- semantic
- bm25
- hybrid
- selfquery
- hybrid_with_selfquery

O router tambem depende de configuracao obrigatoria obtida por get_modern_hybrid_decision_config. Se essa configuracao faltar, ele falha explicitamente com ContentQAError.

## 6. Decisoes tecnicas importantes

### 6.1. Sinais tecnicos e codigos exatos puxam a decisao para hybrid

O proprio comentario do orchestrator reforca essa intencao: quando a consulta traz caracteristicas tecnicas ou codigos exatos, a decisao deve favorecer busca hibrida, mesmo que uma leitura superficial sugerisse caminho apenas semantico.

O ganho pratico e proteger casos em que embeddings sozinhos tendem a errar match literal.

### 6.2. Structured filters podem empurrar para self-query

Quando a pergunta sugere filtros estruturados, o router nao trata isso como mero detalhe textual. Ele considera a necessidade de retrieval orientado a filtros.

### 6.3. A etapa produz confianca e fallback

Isso importa porque o pipeline nao devolve apenas uma decisao seca. Ele devolve quao confiante esta e qual caminho alternativo pode ser usado se a estrategia principal falhar.

## 7. O que entra e o que sai

Entradas:

- effective_question
- contexto opcional
- configuracao moderna de decisao

Saidas:

- QueryFeatures
- QueryAnalysis
- RoutingDecision consolidada com processor_type, retriever_strategy, confidence, should_expand_query, requires_fusion e fallback_processor

## 8. O que pode dar errado

Falhas confirmadas no codigo:

- consulta vazia fornecida ao router gera erro explicito;
- ausencia da configuracao obrigatoria de decisao hibrida gera ContentQAError;
- qualquer erro de analise e encapsulado como Query analysis failed.

O que o codigo evita:

- usar decisao opaca sem fatores explicativos;
- esconder falta de configuracao estrutural.

## 9. Como diagnosticar

Sinais uteis confirmados:

- log de decisao com strategy, confidence, complexity e fallback;
- marker PIPELINE_ROUTING_DECISION;
- last_telemetry do router;
- bloco routing_decision no payload final;
- bloco analise_query em PipelineDiagnosticsBuilder.

Em linguagem simples: esta etapa explica por que o sistema escolheu determinado caminho de busca.

## 10. Exemplo pratico guiado

Cenario: a pergunta menciona uma norma e um codigo tecnico.

1. QueryAnalyzer detecta technical_codes e termos tecnicos.
2. AdaptiveQueryRouter marca has_exact_codes e has_technical_terms.
3. A confianca para hybrid sobe.
4. RoutingDecision registra processador hibrido, possivel expansao e eventual fusao.
5. A recuperacao especializada recebe esse comando e executa o caminho adequado.

O valor pratico e evitar que uma busca literal critica dependa apenas de vizinhanca vetorial.

## 11. Evidencias no codigo

- src/qa_layer/rag_engine/query_analyzer.py
  - Simbolo relevante: QueryAnalyzer.analyze e QueryFeatures
  - Comportamento confirmado: classificacao semantica, dominio, complexidade, entidades, palavras-chave e sinais estruturados.
- src/qa_layer/rag_engine/adaptive_router.py
  - Simbolo relevante: AdaptiveQueryRouter.analyze_and_route
  - Comportamento confirmado: estrategias de retrieval, confianca, complexidade, fatores de decisao e fallback.
- src/qa_layer/rag_engine/intelligent_orchestrator.py
  - Simbolo relevante: IntelligentRAGOrchestrator._analyze_and_route_query
  - Comportamento confirmado: integracao da analise com a decisao final de roteamento.
