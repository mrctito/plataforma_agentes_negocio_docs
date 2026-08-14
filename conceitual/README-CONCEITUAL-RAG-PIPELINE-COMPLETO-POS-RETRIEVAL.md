# Manual detalhado da etapa: Pos-retrieval do pipeline RAG

## 1. O que esta etapa faz

Esta etapa pega os documentos recuperados e decide o que ainda precisa ser feito antes da geracao final. Ela nao procura novos significados do zero; ela melhora, limpa, funde, complementa e filtra o conjunto retornado pela etapa de retrieval.

Em linguagem simples: e a triagem fina depois da busca bruta.

## 2. Subetapas confirmadas no codigo

O pos-retrieval lido no codigo cobre principalmente estes blocos.

- fusao hibrida nativa (DBSF) e rerank ColBERT nativo, quando habilitado, dentro da mesma consulta ao vector store;
- gate de evidencia (responder ou recusar por falta de base suficiente);
- fusao de rankings quando requires_fusion for verdadeiro;
- deduplicacao de documentos;
- cache semantico para retrievers elegiveis;
- aplicacao de ACL apos a recuperacao;
- normalizacao final dos documentos.

## 3. Resultado lexical já resolvido pelo provider

Quando a estratégia é híbrida, o Qdrant já entrega o ranking combinado por **DBSF** (Distribution-Based Score Fusion) sobre dense+sparse — desde 2026-08-14, no lugar do RRF (Reciprocal Rank Fusion) usado antes — e o Azure Search já combina texto+vetor no índice. O pós-retrieval recebe esse resultado pronto; não executa FTS PostgreSQL, retriever BM25 ou fusão lexical paralela.

Por que a troca importa em linguagem simples: o RRF media só a *posição* de cada resultado (1º, 2º, 3º...), então um resultado excelente e um apenas razoável, lado a lado no ranking, ficavam quase empatados. O DBSF olha a *distância* entre as notas de cada busca antes de somar — o topo da lista fica mais confiável, e os números resultantes podem passar de 1,0 (não é mais uma porcentagem).

## 4. Rerank nativo do Qdrant (ColBERT) e o gate de evidência (desde 2026-08-14)

Quando o tenant liga `qa_system.reranker.enabled`, a busca híbrida ganha um segundo estágio **dentro
da mesma consulta ao Qdrant**: em vez de devolver direto o resultado do DBSF, o servidor reordena os
~100 melhores candidatos comparando a pergunta com o texto de cada trecho de forma muito mais
detalhada (modelo late interaction ColBERT — um vetor por palavra, não um só por trecho) e devolve os
8 melhores. Nada disso roda num modelo dentro dos nossos containers: é o próprio Qdrant que tokeniza
e reordena. Quando o acervo é antigo (ingerido antes deste recurso) ou o tenant não pediu, a busca
segue normalmente com a ordem do DBSF — o rerank nunca é obrigatório para a busca funcionar.

Depois do rerank (ou do DBSF puro, quando não há rerank), existe um segundo filtro conceitualmente
diferente: o **gate de evidência**. Ele decide se o conjunto recuperado é bom o suficiente para deixar
o modelo responder, ou se é melhor devolver "não encontrei no acervo material suficiente" em vez de
arriscar uma resposta plausível e errada. A regra, em linguagem simples:

- se o rerank **não** reordenou o conjunto, o gate compara o melhor "quão parecido" (score dos
  vetores densos) com um piso configurado — abaixo do piso, bloqueia;
- se o rerank **reordenou** o conjunto, o gate **não bloqueia** — o piso antigo foi calibrado sobre
  outra situação (quando o conjunto devolvido é sempre o topo mais parecido do acervo) e deixou de
  fazer sentido quando o topo vem de uma reordenação por relevância fina. Não existe hoje um segundo
  piso calibrado para o caminho com rerank.

**Pendência real, sem solução definitiva ainda:** o gate registra os dois números (o do vetor denso
e o do rerank) em log a cada pergunta, para que uma calibração futura tenha dados reais — mas hoje,
com rerank ligado, não existe nenhuma trava de score. É decisão pendente do usuário se e como
calibrar essa trava; enquanto isso, o próprio modelo de linguagem continua recusando honestamente
perguntas sem base no acervo, mesmo sem ajuda do gate.

## 5. Fusao formal de resultados

Quando a decisao de roteamento marca requires_fusion, o orchestrator chama a etapa interna de aplicacao de fusao. O motor de fusao e construido por HybridFusion com configuracao lida do YAML moderno.

O codigo mostra que a fusao nao e cosmetica. Ela usa algoritmo configurado, incluindo Weighted RRF quando aplicavel, e pesos por retriever. Isso e importante porque combina sinais diferentes sem depender de uma ordem arbitraria do primeiro retriever que respondeu. Atenção: este é outro mecanismo, que funde resultados de **múltiplos retrievers** (por exemplo self-query e multi-query); não é o mesmo RRF que a busca híbrida nativa do Qdrant já substituiu por DBSF (§3).

## 6. Deduplicacao como protecao de qualidade

O codigo deduplica por chunk_id quando disponivel e, na falta dele, usa conteudo como identidade. Em linguagem simples: se dois retrievers trouxerem o mesmo trecho por caminhos diferentes, o pipeline tenta nao inflar artificialmente a evidencia.

## 7. Cache semantico

O orchestrator prepara semantic cache quando a configuracao permite e quando ha embeddings disponiveis. Os retrievers elegiveis para cache sao semanticos e afins, como vector_search, semantic_search, hybrid_search, self_query e multi_query.

O trace registra hit, miss e store. Isso importa porque perguntas parecidas podem reaproveitar retrieval anterior, mas o diagnostico continua sabendo que houve cache e nao execucao completa do retriever.

## 8. ACL pos-retrieval

Depois da execucao de retrieval, o orchestrator chama AccessControlEvaluator.filter_documents. So entao os documentos passam por normalize_documents e seguem para a geracao final.

Esse detalhe e central para seguranca: o sistema pode encontrar um documento relevante e, ainda assim, nao poder usa-lo na resposta. O projeto reconhece explicitamente essa diferenca.

## 9. Decisoes tecnicas importantes

### 9.1. ACL depois da busca e nao antes da explicacao da relevancia

O retrieval trabalha sobre o acervo, mas o uso final da evidencia respeita o acesso do usuario. Isso deixa o pipeline tecnicamente honesto: relevancia e permissao nao sao a mesma coisa.

### 9.2. Ausência de sparse falha fechada

O caminho híbrido não troca silenciosamente para dense-only. Uma coleção incompatível precisa ser corrigida e reingerida pelo lifecycle oficial.

### 9.3. Cache semantico e rastreavel

O codigo nao so usa cache; ele registra quando usou. Sem isso, o cache aceleraria o sistema, mas confundiria troubleshooting.

## 10. O que pode dar errado

Falhas ou efeitos confirmados:

- cache semantico pode falhar ao salvar e registrar erro estruturado.
- ACL pode retirar todos os documentos, deixando o conjunto final vazio.

Em termos praticos, uma resposta ruim depois do retrieval pode ser problema de enriquecimento, fusao, cache, rerank, gate de evidência ou ACL, nao apenas do retriever inicial.

## 11. Como diagnosticar

Sinais uteis confirmados:

- retrieval_trace por tentativa;
- steps de retrieval provider-native, rag:fusion e rag:access_control;
- evento `ingestion.vector_store.qdrant.hybrid_search.rerank` (sempre em info, reordenou ou nao, com o motivo);
- log `generation:evidence_gate` (bloqueou ou nao, com os dois scores);
- controle_acesso no payload final;
- resultado_retrieval em PipelineDiagnosticsBuilder;
- logs de cache semantico hit, miss e store.

## 12. Exemplo pratico guiado

Cenario: a busca híbrida provider-native retorna chunks de mais de uma expansão, com rerank ligado.

1. O provider entrega os rankings dense+sparse já combinados por DBSF.
2. Se o rerank estiver ligado e o acervo tiver o vetor ColBERT, o Qdrant reordena o topo por relevância fina, na mesma consulta.
3. Resultados de expansões adicionais são mesclados quando aplicável.
4. Duplicados são removidos.
5. O gate de evidência decide se o conjunto sustenta uma resposta.
6. A ACL retira o que o usuário não pode ver.
7. O conjunto final segue para o LLM.

Sem essa etapa, o sistema entregaria ao modelo um contexto mais fraco e menos governado.

## 13. Evidencias no codigo

- src/qa_layer/rag_engine/retrieval_engine.py
  - Simbolo relevante: execução provider-native, fusão e deduplicação.
  - Comportamento confirmado: busca híbrida nativa, mesclagem e deduplicação.
- src/ingestion_layer/vector_stores/qdrant_client.py
  - Simbolo relevante: search_hybrid, _resolve_native_rerank_target.
  - Comportamento confirmado: fusão DBSF sempre; rerank ColBERT server-side quando habilitado e o acervo tiver o vetor `_late`.
- src/qa_layer/rag_engine/intelligent_orchestrator.py
  - Simbolo relevante: _execute_routing_decision, _avaliar_gate_de_evidencia e trecho de AccessControlEvaluator.filter_documents em intelligent_retrieve
  - Comportamento confirmado: fusao condicional, gate de evidência (com o ramo específico para conjunto reordenado por rerank), ACL pos-retrieval e normalizacao final.
