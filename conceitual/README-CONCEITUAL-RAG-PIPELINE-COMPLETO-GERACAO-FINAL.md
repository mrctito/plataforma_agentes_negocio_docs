# Manual detalhado da etapa: Geracao final do pipeline RAG

## 1. O que esta etapa faz

Esta etapa pega o conjunto final de documentos permitidos, monta contexto textual util para o modelo, renderiza o prompt, chama o LLM com retry e devolve a resposta junto com fontes e metadados estruturados.

Em linguagem simples: e aqui que a evidencia recuperada vira resposta redigida.

## 2. Onde ela acontece

O orchestrator chama _assemble_final_result, que por sua vez delega a GenerationEngine.generate_intelligent_answer. Isso mostra uma separacao clara entre montagem do payload final e a geracao de linguagem propriamente dita.

## 3. Como o codigo implementa a etapa

O fluxo principal lido no codigo segue esta ordem.

1. _assemble_final_result recebe documents, RoutingDecision, token_snapshot e user_context.
2. Se houver documentos ou contexto de usuario relevante, chama _generate_intelligent_answer.
3. GenerationEngine resume a presenca multimodal dos documentos.
4. Monta o context_text a partir de documentos e memoria relevante.
5. Renderiza o system prompt usando placeholders context e question.
6. Garante que o prompt final contenha um bloco de resposta.
7. Chama o LLM com run_with_external_retry.
8. Extrai answer do retorno do modelo.
9. Registra token usage via BillingCollector.
10. Formata fontes e monta o payload final.

## 4. Decisoes tecnicas importantes

### 4.1. Geracao sem LLM disponivel e erro explicito

GenerationEngine nao inventa uma resposta local quando o LLM esta ausente. Ele falha com ContentQAError. Isso protege o contrato da etapa: geracao final e realmente geracao com modelo, nao string montada para fingir sucesso.

### 4.2. O prompt e renderizado de forma defensiva

Se o template nao tiver placeholders esperados, o engine anexa blocos extras de CONTEXTO e PERGUNTA. Se o texto nao indicar resposta, ele acrescenta RESPOSTA ao fim. Isso evita prompt quebrado por template incompleto.

### 4.3. A etapa preserva rastreabilidade de custo

TokenUsageSnapshot e BillingCollector mostram que a geracao final nao e tratada como detalhe opaco. O sistema tenta medir e registrar o custo do trecho mais caro do fluxo.

## 5. O que entra e o que sai

Entradas:

- documents ja filtrados por ACL
- RoutingDecision
- question original
- user_context
- token_snapshot

Saidas:

- answer
- sources
- source_documents
- routing_decision serializada
- pipeline_metrics com pipeline_time e llm_generation_time
- query_analysis resumida
- metadata com strategy_used, confidence e documents_used

## 6. O que pode dar errado

Falhas confirmadas:

- LLM indisponivel gera ContentQAError.
- qualquer erro na chamada ao modelo vira Intelligent answer generation failed.
- se nenhum documento relevante sobrar e nenhum contexto compensar, a resposta padrao passa a ser Nenhum documento relevante encontrado para a consulta.

Essa resposta padrao e importante: ela distingue ausencia de evidencia de sucesso artificial.

## 7. Como diagnosticar

Sinais uteis confirmados:

- steps generation:multimodal_presence, generation:assemble_context, generation:render_prompt e generation:llm_call;
- llm_generation_time em pipeline_metrics;
- logs com tamanho do prompt, fonte do prompt e tamanho da resposta;
- token_usage no payload final quando houver dados reais.

## 8. Exemplo pratico guiado

Cenario: o retrieval encontrou quatro documentos relevantes e o usuario tem memoria recente de perguntas parecidas.

1. GenerationEngine monta contexto combinando documentos e memoria.
2. Renderiza o prompt final.
3. Chama o LLM com retry externo.
4. Recebe a resposta textual.
5. Registra custo aproximado e token usage.
6. O orchestrator devolve answer, sources e metricas no mesmo payload.

O efeito pratico e separar claramente evidencia, contexto do usuario e redacao final.

## 9. Evidencias no codigo

- src/qa_layer/rag_engine/intelligent_orchestrator.py
  - Simbolo relevante: _assemble_final_result
  - Comportamento confirmado: chamada ao engine de geracao, default de ausencia de documentos e montagem do payload final.
- src/qa_layer/rag_engine/generation_engine.py
  - Simbolo relevante: GenerationEngine.generate_intelligent_answer
  - Comportamento confirmado: montagem de contexto, renderizacao de prompt, chamada ao LLM com retry e registro de token usage.
