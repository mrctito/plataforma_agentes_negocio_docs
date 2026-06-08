# Manual técnico, executivo, comercial e estratégico: sistema de caching completo

## 1. O que esta documentação técnica cobre

Este manual cobre o comportamento operacional do sistema de caching confirmado no código lido. O foco aqui não é vender a ideia do cache, e sim explicar como ele realmente funciona, quais camadas existem, como as chaves são derivadas, como a invalidação se propaga, quais endpoints administrativos operam o subsistema e como investigar falhas reais.

## 2. Entry points reais do sistema

O caching aparece em vários entry points do runtime:

- carregamento e parse de YAML via integração com o pool global de configuração;
- inicialização de pipelines de pergunta e resposta no serviço de consulta;
- montagem de DeepAgent e demais slices agentic ativos;
- resolução de LLM, vector store, tools dinâmicas e engines SQL no WarmResourcePool;
- emissão e consumo de sessões efêmeras em superfícies que precisam guardar chaves privadas temporárias;
- endpoints administrativos de observabilidade e invalidação.

O principal cuidado técnico é não tratar todos esses entry points como se compartilhassem a mesma implementação. Eles compartilham princípios, não a mesma classe.

## 3. Conceitos técnicos centrais

### 3.1 Prefixo de ambiente

As chaves são namespaced por ambiente. O código usa ENVIRONMENT para gerar prefixos como env::development ou fragmentos equivalentes. Isso isola cache entre development, staging e produção.

### 3.2 Tenant compartilhado por YAML

Quando o runtime precisa reaproveitar recursos pesados por contexto funcional, ele deriva uma identidade de tenant compartilhado a partir do hash canônico do YAML, e não apenas do usuário autenticado. Isso aparece no CacheKeyRegistry.shared_tenant_key.

### 3.3 Hash canônico do YAML

O sistema tenta privilegiar pristine_hash armazenado em `_config_metadata`. Quando esse valor existe, ele vira a identidade canônica da configuração. Se não existir, o hash é recalculado sobre o payload sanitizado, removendo chaves sensíveis e metadados que não devem contaminar o reaproveitamento.

### 3.4 Invalidação global por timestamp

O sinal global de invalidação é lido a partir de uma chave Redis central. Os caches locais checam esse timestamp antes de reutilizar o artefato. Se o timestamp for mais novo do que o último visto, a memória local é limpa.

## 4. Arquitetura em camadas

### 4.1 Camada base de Redis

RedisManager é o singleton por processo que encapsula conexão, pooling, ping e disponibilidade. RedisGenericCache reaproveita esse manager para armazenar payloads genéricos com namespace e helpers JSON. RedisUtils fornece wrappers resilientes para operações pontuais como leitura, escrita com TTL, deleção e scan.

Essa camada é infraestrutura. Ela não sabe o significado de pipeline, supervisor ou YAML. Ela só oferece armazenamento compartilhado e operações padronizadas.

### 4.2 Camada de chaves e identidade

CacheKeyRegistry centraliza a derivação de:

- shared_tenant_key;
- chaves de dyn_sql, dyn_api e stored procedures;
- chaves de workflow e supervisor compilados;
- hashes determinísticos de payloads.

Esse ponto é crítico porque evita que cada componente invente sua própria política de chave.

### 4.3 Camada de pool aquecido

WarmResourcePool é o coração do cache local em memória. Ele mantém:

- _llms;
- _stores;
- _dynamic_tools;
- _db_engines;
- _llm_caches;
- _generic_cache;
- acesso ao cache global de YAML.

Cada categoria tem contrato próprio de chave, de TTL e de invalidação.

### 4.4 Camada de caches especializados

Existem caches especializados fora do WarmResourcePool:

- PipelineCacheManager para ContentQASystem;
- cache global de ToolsFactory em supervisor_cache;
- armazenamento de supervisores e DeepAgents no cache genérico do pool;
- cache de workflow compilado no runtime de workflow;
- SessionCacheFactory para sessão efêmera.

### 4.5 Camada administrativa

AdminCacheService e os routers administrativos expõem limpeza, estatísticas, snapshot de memória e invalidação por categoria. Essa camada é o contrato oficial de operação do subsistema.

## 5. Como a configuração entra no sistema

O fluxo confirmado no código é:

1. O YAML bruto entra no runtime.
2. O conteúdo pode ser parseado usando get_yaml_config do WarmResourcePool.
3. A chave do cache YAML é um SHA256 curto do conteúdo bruto.
4. O YAML parseado pode fornecer pristine_hash em `_config_metadata`.
5. Esse hash canônico passa a influenciar shared_tenant_key e caches especializados, como pipeline, supervisor e workflow.

O efeito prático é importante: o sistema tenta reaproveitar por identidade de configuração, não só por strings soltas de arquivo ou por usuário.

## 6. Cache YAML global

RedisYamlCache é um cache híbrido. Ele tenta usar Redis quando o manager está habilitado e saudável. Quando isso não acontece, ele mantém um fallback TTL em memória.

### O que ele armazena

Ele armazena um envelope com:

- stored_at_iso;
- stored_at_monotonic;
- config.

Esse envelope permite snapshot administrativo com idade e TTL.

### Como ele invalida

- invalidate remove uma chave específica;
- clear apaga o namespace inteiro;
- purge_expired só faz faxina real quando a camada em uso é o fallback em memória;
- size e keys refletem o backend atual.

### Risco operacional

Quando Redis não está presente, o comportamento continua funcional, mas deixa de ser compartilhado entre processos. Isso é especialmente importante em cenários com múltiplos workers.

## 7. WarmResourcePool por tenant

get_pool resolve ou cria um pool aquecido isolado por tenant lógico. Se tenant_key não é informado, ele tenta derivar a identidade a partir do YAML. Se isso não for possível, cai para um tenant global namespaced pelo ambiente.

### 7.1 Cache de LLM

get_llm combina a identidade do tenant com a configuração de LLM, temperatura customizada e modelo customizado. O pool também pode criar um cache específico de respostas de LLM usando LLMCacheConfigurator, ObservedCache e PrefixCache, normalmente com Redis como backend.

Configurações confirmadas:

- llm.cache.enabled;
- llm.cache.redis.ttl_seconds;
- llm.cache.redis.namespace;
- llm.cache.redis.url;
- llm.cache.redis.use_manager.

Se o cache de LLM estiver desligado, o modelo continua existindo, mas sem backend de cache acoplado.

### 7.2 Cache de vector store

get_vectorstore usa chave composta por tenant, tipo do vector store e identidade normalizada do recurso. Isso impede reconstrução repetida da store para o mesmo tenant e a mesma assinatura funcional.

### 7.3 Cache de tools dinâmicas

get_dynamic_tool usa chave textual, normalmente construída de forma determinística por CacheKeyRegistry para dyn_sql, dyn_api e stored procedures. A invalidação pode ser total, por tenant ou por sufixo de chave.

### 7.4 Cache de engine SQL

get_db_engine usa hash da connection string mais fragmento de tenant. A invalidação chama dispose quando a engine suporta esse método, o que é essencial para não deixar pool interno pendurado.

### 7.5 Cache genérico

O cache genérico armazena qualquer valor com TTL opcional por entrada. Ele é usado para artefatos que não merecem uma categoria dedicada, como supervisores já montados ou workflows compilados.

Esse cache também suporta snapshot administrativo detalhado com tipo, TTL restante e horário de expiração.

## 8. SessionCacheFactory e sessão efêmera

SessionCacheFactory decide automaticamente o backend de sessão efêmera.

Ordem confirmada no código:

1. tenta RedisSessionCache se Redis estiver habilitado;
2. se falhar, tenta FileSessionCache usando SESSION_CACHE_DIR ou tmp/session_keys;
3. se isso também falhar, usa InMemorySessionCache.

### Contrato funcional

- store grava chave privada com TTL;
- pop recupera e remove;
- peek apenas consulta.

### Impacto prático

Redis é o caminho mais adequado para ambiente multi-instância. Arquivo e memória preservam funcionamento local, mas reduzem portabilidade distribuída.

## 9. PipelineCacheManager

PipelineCacheManager é o cache especializado de ContentQASystem.

### Como ele calcula a identidade

Ele usa CacheKeyRegistry.canonical_yaml_hash. Isso significa que, se o pristine_hash estiver presente, dois YAMLs funcionalmente equivalentes tendem a convergir para a mesma entrada.

### O que ele mede

Ele mantém métricas de:

- hits;
- misses;
- stores;
- evictions.

### Como ele reage à invalidação global

Antes de usar o cache, ele checa o timestamp global. Se houver invalidação mais nova, limpa a estrutura e registra o evento em log.

### Onde ele entra no fluxo

QuestionService usa PipelineCacheManager na inicialização de ContentQASystem. Em hit, o pipeline é reaproveitado. Em miss, ele é construído do zero e então armazenado.

## 10. Cache agentic de supervisores, DeepAgents e workflows

### 10.1 ToolsFactory compartilhado

supervisor_cache mantém um TTLCache global de ToolsFactory com lock compartilhado. O cache é indexado por combinação de hash de YAML e IDs das tools relevantes.

### 10.2 Supervisor tradicional

AgentSupervisor resolve cache_key e consulta o cache genérico do WarmResourcePool. Se encontrar entrada com hash e versão compatíveis, reutiliza o supervisor montado. Se a versão mudou ou o hash mudou, invalida e reconstrói.

### 10.3 DeepAgent supervisor

DeepAgent repete a mesma estratégia: hash do YAML, chave de cache, validação de versão, limpeza em incompatibilidade e armazenamento do artefato pronto no pool genérico.

### 10.4 Workflow compilado

O runtime de workflow também usa chave canônica derivada para armazenar o workflow compilado no cache genérico do pool. Isso reduz recompilação de grafos determinísticos equivalentes.

## 11. Sinal global de invalidação

O mecanismo central usa uma chave Redis de invalidação global. Resource pool, cache de pipelines e cache de ToolsFactory consultam esse timestamp antes de reutilizar estado local.

O desenho é importante:

- não existe push ativo que despeja memória de todos os processos ao mesmo tempo;
- existe uma checagem preguiçosa, porém explícita, no momento do uso;
- quando o timestamp avança, o processo local descarta seu conteúdo quente.

Esse comportamento é leve e suficiente para a maioria dos fluxos operacionais, mas precisa ser entendido corretamente em troubleshooting.

## 12. Contratos e endpoints administrativos

### 12.1 Invalidação e limpeza

O router administrativo de cache expõe endpoints para:

- limpar tudo;
- resetar cache de credential manager;
- flush de Redis;
- reset de BM25;
- invalidar cache de LLM;
- invalidar tools;
- invalidar YAML;
- invalidar DB;
- invalidar cache genérico;
- invalidar vector store.

### 12.2 Observabilidade

O router administrativo de observabilidade expõe:

- /admin/cache/memory para snapshot do processo;
- /admin/cache/stats para estatísticas detalhadas de cache;
- /admin/status para status consolidado;
- /admin/status/async-jobs para snapshot operacional do contrato assíncrono.

### 12.3 Permissões

Os endpoints de cache e observabilidade exigem permissões administrativas específicas, como ADMIN_CACHE_VIEW e ADMIN_STATUS.

## 13. O que aparece no cache stats administrativo

O cache stats consolidado separa dois universos:

### Memória local por tenant

Ele lista por tenant:

- llms;
- vectorstores;
- dynamic_tools;
- db_engines;
- cache_generico;
- credential_managers.

Também detalha snapshot de:

- chaves de LLM;
- chaves de vector store;
- chaves de tools dinâmicas;
- engines de banco com máscara de credencial;
- itens do cache genérico;
- configs YAML com idade, TTL e vector_store_id.

### Redis compartilhado

O snapshot Redis administrativo confirma contagem e alguns agrupamentos para:

- embeddings, agrupados por namespace;
- mídias de WhatsApp;
- progress_registry;
- cache BM25, incluindo prefixes e detalhes por vectorstore.

Esse ponto é importante porque mostra que o Redis do projeto abriga mais de uma família de cache observável, e que a operação administrativa já diferencia esses grupos.

## 14. Caminho feliz

No caminho feliz, o fluxo típico é:

1. O chamador resolve a identidade do tenant e o hash do YAML.
2. O cache local ou especializado encontra uma entrada válida.
3. O recurso retorna imediatamente, sem reconstrução.
4. Logs registram reaproveitamento.
5. O endpoint administrativo de stats passa a mostrar a presença desse recurso em memória ou no Redis.

Exemplo confirmado no fluxo de perguntas:

- QuestionService consulta PipelineCacheManager.
- Em hit, ContentQASystem é reutilizado.
- Em miss, o pipeline é criado, armazenado e então reutilizável nas próximas consultas equivalentes.

## 15. Cenários de erro confirmados

### Redis indisponível na superfície YAML

Consequência: RedisYamlCache usa fallback em memória.

Diagnóstico: cache continua funcionando no processo local, mas deixa de ser compartilhado.

### Falha ao consultar invalidação global

Consequência: o cache local continua operando com o último estado conhecido.

Diagnóstico: warnings explícitos de falha de consulta do timestamp global.

### Entrada incompatível por hash ou versão

Consequência: supervisor ou DeepAgent invalidam a entrada e constroem novamente.

Diagnóstico: logs de invalidação seguidos de cache miss e armazenamento novo.

### Redis indisponível para sessão efêmera

Consequência: SessionCacheFactory tenta arquivo e depois memória.

Diagnóstico: sessões podem continuar funcionando localmente, mas com semântica diferente de compartilhamento.

## 16. Troubleshooting orientado

### Sintoma: supervisor recompila sempre

Verifique:

- se o hash do YAML está estável;
- se a versão do cache mudou;
- se houve invalidação global recente;
- se a chave do cache genérico está sendo preservada entre chamadas.

### Sintoma: pipeline RAG nunca é reutilizado

Verifique:

- presença ou ausência de pristine_hash em `_config_metadata`;
- logs de PipelineCacheManager;
- se o YAML realmente é equivalente entre chamadas;
- se houve clear administrativo do cache.

### Sintoma: cache stats mostra Redis saudável, mas workers não compartilham artefatos

Verifique:

- se a categoria observada é realmente distribuída ou só local em memória;
- se a superfície em questão usa WarmResourcePool local, SessionCacheFactory ou RedisYamlCache em fallback;
- se o recurso esperado vive no cache genérico local, não em Redis.

### Sintoma: limpeza global não parece surtir efeito completo

Verifique:

- publicação do timestamp global de invalidação;
- logs de aplicação da invalidação no resource pool e no cache de pipeline;
- se o processo ainda não reutilizou aquele cache depois da invalidação.

## 17. Como colocar para funcionar

O caminho confirmado no código/configuração lidos é este:

1. Definir o ambiente com ENVIRONMENT.
2. Habilitar Redis quando o objetivo for compartilhamento distribuído das superfícies que dependem dele.
3. Garantir que SESSION_CACHE_DIR exista ou seja gravável se a sessão cair para backend de arquivo.
4. Consultar /admin/cache/stats para verificar se o cache está sendo ocupado.
5. Usar endpoints administrativos de invalidação para validar comportamento de limpeza e reconstrução.

Caminho de execução mais específico para cada slice depende do fluxo que o consome, mas o contrato operacional de observação já está confirmado nesses endpoints administrativos.

## 18. Exemplos práticos guiados

### Exemplo 1: mesma consulta, mesmo YAML, pipeline reutilizado

Cenário: duas consultas sucessivas usam a mesma configuração funcional.

Processamento: QuestionService calcula o hash canônico, consulta PipelineCacheManager e recebe o ContentQASystem já construído na segunda chamada.

Saída esperada: logs de pipeline_cache_status reused e menor custo de inicialização.

### Exemplo 2: operador invalida YAML e força reparse

Cenário: um operador limpa o cache YAML.

Processamento: a próxima chamada ao parse não encontra a config no RedisYamlCache nem no fallback local e executa novamente a factory de parsing.

Saída esperada: aumento pontual de custo, seguido de armazenamento renovado.

### Exemplo 3: Redis fora do ar para sessão efêmera

Cenário: o backend distribuído está indisponível.

Processamento: SessionCacheFactory tenta Redis, falha e passa a usar arquivo ou memória.

Saída esperada: o fluxo continua, mas com menor capacidade de compartilhamento entre processos.

## 19. Explicação 101

Pense no sistema como um conjunto de depósitos diferentes para coisas diferentes. Um depósito guarda configurações já lidas. Outro guarda ferramentas já montadas. Outro guarda pipelines completos de resposta. Outro guarda sessões muito curtas. Se tudo fosse jogado no mesmo depósito, seria rápido no começo, mas impossível de governar depois.

O código deste projeto prefere depósitos especializados, com chaves previsíveis, tempo de vida controlado e limpeza administrativa quando necessário.

## 20. Limites e pegadinhas

- Cache YAML híbrido não entrega exatamente a mesma semântica em Redis e em fallback de memória.
- O cache genérico é poderoso, mas exige cuidado porque pode armazenar artefatos heterogêneos.
- O snapshot administrativo não significa persistência durável; ele mostra estado transitório do runtime.
- O fato de Redis estar habilitado não significa que todo cache relevante esteja nele.
- O sistema tem invalidação global, mas ela é reativa no momento do próximo uso do cache local.

## 21. Checklist de entendimento

- Entendi como o hash canônico do YAML influencia o cache.
- Entendi a função do WarmResourcePool.
- Entendi a diferença entre RedisYamlCache, PipelineCacheManager e SessionCacheFactory.
- Entendi onde o DeepAgent e outros slices agentic reaproveitam artefatos.
- Entendi o que a API administrativa consegue invalidar e observar.
- Entendi como diagnosticar misses, invalidações e degradação para fallback local.

## 22. Evidências no código

- src/core/cache_key_registry.py
  - Motivo da leitura: confirmar derivação central de chaves e hashes.
  - Símbolos relevantes: canonical_yaml_hash, shared_tenant_key, dynamic_sql_key, workflow_cache_key.
  - Comportamento confirmado: identidade determinística por ambiente e por YAML sanitizado.

- src/core/resource_pool.py
  - Motivo da leitura: confirmar a malha principal de cache local, YAML híbrido e invalidação global.
  - Símbolos relevantes: RedisYamlCache, WarmResourcePool, get_pool, list_active_pools.
  - Comportamento confirmado: caches por tipo, TTL padrão, snapshot, fallback YAML e limpeza reativa por timestamp global.

- src/security/session_cache.py
  - Motivo da leitura: confirmar backend polimórfico de sessão efêmera.
  - Símbolos relevantes: SessionCacheFactory, RedisSessionCache, FileSessionCache, InMemorySessionCache.
  - Comportamento confirmado: ordem de seleção Redis, arquivo, memória.

- src/qa_layer/pipeline_cache_manager.py
  - Motivo da leitura: confirmar cache especializado de ContentQASystem.
  - Símbolos relevantes: PipelineCacheManager, get_cached_system, store_pipeline.
  - Comportamento confirmado: cache singleton com LRU, TTL, métricas e invalidação global.

- src/agentic_layer/supervisor/supervisor_cache.py
  - Motivo da leitura: confirmar cache compartilhado de ToolsFactory.
  - Símbolos relevantes: get_or_create_tools_factory, invalidate_tools_factory_cache.
  - Comportamento confirmado: TTLCache global por processo com locks e invalidação global.

- src/agentic_layer/supervisor/deep_agent_supervisor.py
  - Motivo da leitura: confirmar reaproveitamento de DeepAgent.
  - Símbolos relevantes: pool.get, pool.set, _supervisor_cache_key.
  - Comportamento confirmado: mesma disciplina de hash, versão e reconstrução controlada.

- src/services/question_service.py
  - Motivo da leitura: confirmar consumo real do cache de pipeline.
  - Símbolos relevantes: _initialize_qa_system.
  - Comportamento confirmado: consulta ao PipelineCacheManager antes de criar ContentQASystem.

- src/api/routers/admin/cache_router.py
  - Motivo da leitura: confirmar contratos HTTP administrativos de invalidação.
  - Símbolos relevantes: rotas /cache/clear-all, /cache/invalidate-llm, /cache/invalidate-yaml, /cache/invalidate-vs.
  - Comportamento confirmado: governança operacional por categoria de cache.

- src/api/routers/admin/observability_router.py
  - Motivo da leitura: confirmar contratos HTTP de observabilidade.
  - Símbolos relevantes: /admin/cache/memory, /admin/cache/stats.
  - Comportamento confirmado: snapshots e estatísticas detalhadas do subsistema.

- src/api/services/admin/cache_service.py
  - Motivo da leitura: confirmar composição do payload operacional.
  - Símbolos relevantes: get_cache_stats, get_memory_snapshot.
  - Comportamento confirmado: visão por tenant, por categoria em memória e por grupos Redis como embeddings, mídias, progresso e BM25.

- src/api/routers/admin/admin_runtime_router.py
  - Motivo da leitura: confirmar taxonomia administrativa das chaves Redis.
  - Símbolos relevantes: _collect_redis_cache_stats.
  - Comportamento confirmado: Redis stats agrupando namespaces de embeddings, WhatsApp media, progress registry e BM25.
