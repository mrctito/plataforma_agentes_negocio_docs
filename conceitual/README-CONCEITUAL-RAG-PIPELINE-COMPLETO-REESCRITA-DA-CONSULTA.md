# Manual detalhado da etapa: Reescrita da consulta do pipeline RAG

## 1. O que esta etapa faz

Esta etapa tenta melhorar a recuperabilidade da pergunta antes do roteamento. Ela nao existe para mudar a intencao do usuario, e sim para corrigir, parafrasear e expandir a consulta dentro de limites controlados.

Em linguagem simples: ela tenta deixar a pergunta mais buscavel sem transforma-la em outra pergunta.

## 2. Onde ela entra no fluxo

No IntelligentRAGOrchestrator.intelligent_retrieve, a reescrita acontece logo depois da validacao da pergunta e antes da analise e do roteamento. O resultado dessa etapa define a effective_question que sera analisada pelo restante do pipeline.

## 3. Como o codigo implementa a etapa

O componente principal e QueryRewriter.

O fluxo lido no codigo segue esta ordem.

1. Normaliza a pergunta original.
2. Se a pergunta estiver vazia, devolve passthrough.
3. Se query_rewrite.enabled estiver desligado, devolve passthrough.
4. Se o LLM estiver indisponivel, devolve passthrough.
5. Se nenhuma politica de reescrita estiver habilitada, devolve passthrough.
6. Monta um prompt fixo orientado a manter significado e nao inventar.
7. Chama o LLM com retry controlado.
8. Faz parse da resposta.
9. Sanitiza texto e variacoes.
10. Calcula similaridade entre original e reescrita.
11. Se a similaridade ficar abaixo do limiar, rejeita a reescrita.
12. Se passar no criterio, devolve QueryRewriteResult com applied, reason, similarity e variations.

## 4. Configuracoes confirmadas

Os campos de configuracao lidos em qa_system.query_rewrite sao estes.

- enabled
- enable_paraphrase
- enable_correction
- enable_expansion
- max_variations
- min_similarity
- max_output_chars
- retry_attempts
- retry_wait_min
- retry_wait_max

O detalhe mais importante aqui e min_similarity. O codigo so aceita a nova pergunta quando a distancia semantica em relacao ao texto original continua dentro do limite considerado seguro.

## 5. Decisoes tecnicas importantes

### 5.1. Pass-through e parte do contrato, nao fracasso acidental

QueryRewriter foi desenhado para devolver a pergunta original em varios cenarios validos: disabled, llm_unavailable, policy_disabled, llm_error, parse_error e low_similarity. Isso significa que a reescrita e uma etapa governada, nao uma obrigacao cega.

### 5.2. Similaridade protege contra drift de intencao

O uso de min_similarity e uma barreira tecnica contra o risco mais serio da etapa: melhorar a busca as custas de mudar o sentido da pergunta. Em pratica, o sistema prefere nao reescrever do que reescrever mal.

### 5.3. Retry aqui trata erro transitivo do recurso externo

O codigo usa tenacity para tentar novamente a chamada ao LLM. Isso mostra que a etapa assume dependencia externa sujeita a falha transitiva, mas sem transformar esse retry em permissao para distorcer o resultado.

## 6. O que entra e o que sai

Entradas:

- question
- context
- telemetry
- llm
- configuracao query_rewrite

Saida:

- QueryRewriteResult com original_query, rewritten_query, variations, applied, reason e similarity

## 7. O que pode dar errado

Comportamentos confirmados no codigo:

- llm_error: a reescrita falha e a pergunta original segue adiante.
- parse_error: a resposta do modelo nao e utilizavel e a pergunta original segue.
- low_similarity: a reescrita e rejeitada por seguranca.

Em linguagem simples: o projeto nao trata query rewrite como licença para improvisar.

## 8. Como diagnosticar

Sinais confirmados:

- telemetria query_rewrite com status started, completed, skipped ou error;
- payload de telemetria com applied, similarity, variations e reason;
- log do orchestrator quando a consulta foi efetivamente reescrita para roteamento.

Se uma busca pareceu ruim porque o sistema entendeu a pergunta de forma diferente, esta e a primeira etapa a inspecionar.

## 9. Exemplo pratico guiado

Cenario: o usuario pergunta com ortografia ruim e pouco vocabulário tecnico.

1. QueryRewriter recebe a pergunta original.
2. O prompt orienta correcao, parafrase e expansao conforme a politica ativa.
3. O modelo responde com uma versao mais limpa e possiveis variacoes.
4. O sistema compara a nova versao com a original.
5. Se a similaridade continuar acima do limiar, a nova consulta passa a ser a effective_question.
6. Se nao, a pergunta original segue para o router.

O efeito pratico e melhorar recall sem sacrificar controle.

## 10. Evidencias no codigo

- src/qa_layer/rag_engine/query_rewriter.py
  - Simbolo relevante: QueryRewriteConfig.from_yaml
  - Comportamento confirmado: leitura centralizada das configuracoes de query rewrite.
- src/qa_layer/rag_engine/query_rewriter.py
  - Simbolo relevante: QueryRewriter.rewrite
  - Comportamento confirmado: pass-through governado, chamada ao LLM, parse, saneamento e rejeicao por baixa similaridade.
- src/qa_layer/rag_engine/intelligent_orchestrator.py
  - Simbolo relevante: IntelligentRAGOrchestrator.intelligent_retrieve
  - Comportamento confirmado: uso da effective_question antes da analise e roteamento.
