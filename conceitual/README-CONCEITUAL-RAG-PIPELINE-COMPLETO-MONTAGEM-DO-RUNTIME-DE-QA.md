# Manual detalhado da etapa: Montagem do runtime de QA do pipeline RAG

## 1. O que esta etapa faz

Esta etapa monta o runtime que executara o RAG online. Ela decide se o sistema pode reaproveitar um ContentQASystem ja construido, se precisa montar um novo runtime, quais componentes modernos entram em jogo e como o estado de execucao sera consolidado para o restante do pipeline.

Em linguagem simples: esta etapa nao responde a pergunta. Ela constrói a maquina que sera usada para responder.

## 2. Problema que ela resolve

Construir todo o runtime do zero a cada pergunta aumenta latencia, repete bootstrap caro e dificulta governanca do modo moderno. Ao mesmo tempo, reaproveitar runtime sem criterio cria risco de usar pipeline velho para YAML novo.

O desenho lido no codigo resolve isso combinando:

- cache global de pipeline por hash canonico do YAML;
- runtime state explicito para o modo moderno;
- montagem centralizada do ContentQASystem;
- invalidacao global quando o ambiente sinaliza que o cache ficou obsoleto.

## 3. Como o fluxo acontece

O caminho real sai de QuestionService._initialize_qa_system e entra em PipelineCacheManager e ContentQASystem.

1. QuestionService pede a instancia singleton de PipelineCacheManager.
2. O cache calcula o hash do YAML e tenta localizar um pipeline reutilizavel.
3. Se encontrar, registra hit, atualiza reuse_count e devolve o runtime.
4. Se nao encontrar, QuestionService cria CredentialManager e instancia ContentQASystem.
5. O novo sistema e armazenado no cache global para reuse futuro.

No interior de ContentQASystem, a montagem continua.

1. O runtime inicial nasce de QARuntimeState.from_yaml.
2. Quando nao esta em skip interno, QARuntimeAssembly tenta montar o pipeline moderno obrigatorio.
3. O runtime state e normalizado por ensure_modern.
4. O YAML de runtime derivado e construido por build_runtime_yaml_view com intelligent_pipeline forçado na visao de runtime.
5. O sistema configura llm, embeddings, vector store, qa chain, memoria e pipeline inteligente.
6. O reporter de diagnostico do pipeline e anexado ao contexto.

## 4. Decisoes tecnicas importantes

### 4.1. Cache por hash do YAML, nao por heuristica fraca

PipelineCacheManager nao reaproveita runtime por nome informal, tenant solto ou coincidencia parcial. Ele trabalha com hash do YAML e com metadata de configuracao quando disponivel. Isso reduz o risco de misturar runtime de configuracoes diferentes.

### 4.2. Cache thread-safe e com TTL

O cache usa LRUCache, lock reentrante, metrica de hits e misses, TTL e sinal global de invalidacao. Isso mostra que a reutilizacao foi pensada como infraestrutura compartilhada, nao como dicionario improvisado em memoria.

### 4.3. O modo moderno vira estado explicito do runtime

QARuntimeState.ensure_modern forca o modo modern e registra a justificativa do runtime efetivo. Isso e importante porque o estado final de execucao nao depende apenas do YAML original; ele depende tambem do que a montagem conseguiu resolver em runtime.

### 4.4. A visao derivada do YAML protege o contrato original

build_runtime_yaml_view trabalha sobre copia do YAML, nao sobre mutacao destrutiva do objeto de entrada. O beneficio pratico e impedir que o pipeline injete estado transitorio de runtime no contrato de configuracao recebido.

## 5. O que entra e o que sai

Entradas principais confirmadas:

- yaml_config
- correlation_id
- metadata de configuracao quando disponivel
- credential_manager

Saidas principais confirmadas:

- ContentQASystem pronto para receber pergunta
- runtime state moderno consolidado
- cache hit ou cache miss registrado operacionalmente

## 6. O que pode falhar

Falhas relevantes confirmadas no codigo:

- max_items ou ttl_seconds invalidos no cache geram ValueError.
- falha ao carregar configuracoes de QA via CredentialManager gera ContentQAError.
- ModernPipelineUnavailableError durante a montagem do pipeline moderno vira ContentQAError explicito.
- configuracao invalida de layout do intelligent_pipeline tambem falha de forma explicita.

Em linguagem simples: esta etapa prefere parar cedo a continuar com runtime incoerente.

## 7. Como diagnosticar

Sinais uteis confirmados no codigo:

- logs de pipeline cache reused, building, stored e evicted;
- reuse_count no cache quando um runtime e reaproveitado;
- resumo consolidado de pipeline com modo, estrategia, justificativa, vectorstore_id e vector_store_type;
- log de ativacao do pipeline inteligente na visao de runtime.

Se o sistema estiver mais lento do que o esperado, ou usando runtime errado, esta etapa e uma das primeiras a investigar.

## 8. Exemplo pratico guiado

Cenario: duas perguntas consecutivas usam o mesmo YAML do mesmo acervo.

1. A primeira pergunta produz cache miss.
2. QuestionService constroi ContentQASystem.
3. ContentQASystem monta llm, vector store, memoria, chain e orchestrator.
4. O sistema e armazenado no cache.
5. A segunda pergunta gera cache hit.
6. O mesmo runtime e reutilizado sem repetir o bootstrap caro.

O ganho pratico e menor latencia sem abrir mao de coerencia configuracional.

## 9. Evidencias no codigo

- src/services/question_service.py
  - Simbolo relevante: QuestionService._initialize_qa_system
  - Comportamento confirmado: busca pipeline no cache, cria ContentQASystem quando necessario e armazena o resultado.
- src/qa_layer/pipeline_cache_manager.py
  - Simbolo relevante: PipelineCacheManager.get_cached_system e store_pipeline
  - Comportamento confirmado: cache global por hash, TTL, metrics, reuse_count e invalidacao.
- src/qa_layer/content_qa_system.py
  - Simbolo relevante: ContentQASystem.__init__
  - Comportamento confirmado: runtime assembly moderno, runtime state, configuracao dos componentes e reporter de diagnostico.
- src/qa_layer/qa_runtime_state.py
  - Simbolo relevante: QARuntimeState.ensure_modern
  - Comportamento confirmado: normalizacao explicita do runtime no modo moderno.
