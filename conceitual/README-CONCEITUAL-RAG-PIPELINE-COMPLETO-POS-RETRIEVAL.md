# Manual detalhado da etapa: Pos-retrieval do pipeline RAG

## 1. O que esta etapa faz

Esta etapa pega os documentos recuperados e decide o que ainda precisa ser feito antes da geracao final. Ela nao procura novos significados do zero; ela melhora, limpa, funde, complementa e filtra o conjunto retornado pela etapa de retrieval.

Em linguagem simples: e a triagem fina depois da busca bruta.

## 2. Subetapas confirmadas no codigo

O pos-retrieval lido no codigo cobre principalmente estes blocos.

- fusao de rankings quando requires_fusion for verdadeiro;
- deduplicacao de documentos;
- cache semantico para retrievers elegiveis;
- aplicacao de ACL apos a recuperacao;
- normalizacao final dos documentos.

## 3. Resultado lexical já resolvido pelo provider

Quando a estratégia é híbrida, o Qdrant já entrega o ranking combinado por RRF sobre dense+sparse e o Azure Search já combina texto+vetor no índice. O pós-retrieval recebe esse resultado pronto; não executa FTS PostgreSQL, retriever BM25 ou fusão lexical paralela.

## 4. Fusao formal de resultados

Quando a decisao de roteamento marca requires_fusion, o orchestrator chama a etapa interna de aplicacao de fusao. O motor de fusao e construido por HybridFusion com configuracao lida do YAML moderno.

O codigo mostra que a fusao nao e cosmetica. Ela usa algoritmo configurado, incluindo Weighted RRF quando aplicavel, e pesos por retriever. Isso e importante porque combina sinais diferentes sem depender de uma ordem arbitraria do primeiro retriever que respondeu.

## 5. Deduplicacao como protecao de qualidade

O codigo deduplica por chunk_id quando disponivel e, na falta dele, usa conteudo como identidade. Em linguagem simples: se dois retrievers trouxerem o mesmo trecho por caminhos diferentes, o pipeline tenta nao inflar artificialmente a evidencia.

## 6. Cache semantico

O orchestrator prepara semantic cache quando a configuracao permite e quando ha embeddings disponiveis. Os retrievers elegiveis para cache sao semanticos e afins, como vector_search, semantic_search, hybrid_search, self_query e multi_query.

O trace registra hit, miss e store. Isso importa porque perguntas parecidas podem reaproveitar retrieval anterior, mas o diagnostico continua sabendo que houve cache e nao execucao completa do retriever.

## 7. ACL pos-retrieval

Depois da execucao de retrieval, o orchestrator chama AccessControlEvaluator.filter_documents. So entao os documentos passam por normalize_documents e seguem para a geracao final.

Esse detalhe e central para seguranca: o sistema pode encontrar um documento relevante e, ainda assim, nao poder usa-lo na resposta. O projeto reconhece explicitamente essa diferenca.

## 8. Decisoes tecnicas importantes

### 8.1. ACL depois da busca e nao antes da explicacao da relevancia

O retrieval trabalha sobre o acervo, mas o uso final da evidencia respeita o acesso do usuario. Isso deixa o pipeline tecnicamente honesto: relevancia e permissao nao sao a mesma coisa.

### 8.2. Ausência de sparse falha fechada

O caminho híbrido não troca silenciosamente para dense-only. Uma coleção incompatível precisa ser corrigida e reingerida pelo lifecycle oficial.

### 8.3. Cache semantico e rastreavel

O codigo nao so usa cache; ele registra quando usou. Sem isso, o cache aceleraria o sistema, mas confundiria troubleshooting.

## 9. O que pode dar errado

Falhas ou efeitos confirmados:

- cache semantico pode falhar ao salvar e registrar erro estruturado.
- ACL pode retirar todos os documentos, deixando o conjunto final vazio.

Em termos praticos, uma resposta ruim depois do retrieval pode ser problema de enriquecimento, fusao, cache ou ACL, nao apenas do retriever inicial.

## 10. Como diagnosticar

Sinais uteis confirmados:

- retrieval_trace por tentativa;
- steps de retrieval provider-native, rag:fusion e rag:access_control;
- controle_acesso no payload final;
- resultado_retrieval em PipelineDiagnosticsBuilder;
- logs de cache semantico hit, miss e store.

## 11. Exemplo pratico guiado

Cenario: a busca híbrida provider-native retorna chunks de mais de uma expansão.

1. O provider entrega os rankings dense+sparse já combinados.
2. Resultados de expansões adicionais são mesclados quando aplicável.
3. Duplicados são removidos.
4. A ACL retira o que o usuário não pode ver.
5. O conjunto final segue para o LLM.

Sem essa etapa, o sistema entregaria ao modelo um contexto mais fraco e menos governado.

## 12. Evidencias no codigo

- src/qa_layer/rag_engine/retrieval_engine.py
  - Simbolo relevante: execução provider-native, fusão e deduplicação.
  - Comportamento confirmado: busca híbrida nativa, mesclagem e deduplicação.
- src/qa_layer/rag_engine/intelligent_orchestrator.py
  - Simbolo relevante: _execute_routing_decision e trecho de AccessControlEvaluator.filter_documents em intelligent_retrieve
  - Comportamento confirmado: fusao condicional, ACL pos-retrieval e normalizacao final.
