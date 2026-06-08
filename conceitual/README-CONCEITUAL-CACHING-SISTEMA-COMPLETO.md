# Manual técnico, executivo, comercial e estratégico: sistema de caching completo

## 1. O que é esta feature

O sistema de caching da plataforma não é um cache único nem um simples Redis compartilhado. Ele é uma malha de reaproveitamento distribuída entre memória local do processo, Redis, cache por tenant, cache por hash de YAML, cache de pipelines RAG, cache de ferramentas dinâmicas, cache de supervisores e cache de sessão efêmera.

Na prática, essa feature existe para evitar reconstrução desnecessária de objetos caros, reduzir latência, limitar reconexões a recursos externos e manter o runtime agentic responsivo mesmo quando o mesmo tenant, o mesmo YAML ou o mesmo conjunto de ferramentas volta a ser usado repetidamente.

O valor principal dela não é guardar dados por guardar. O valor está em reaproveitar artefatos caros e transitórios no ponto certo do fluxo, com isolamento por ambiente, por tenant e por assinatura de configuração.

## 2. Que problema ela resolve

Sem essa malha de cache, o runtime precisaria repetir continuamente operações como:

- parsear o mesmo YAML várias vezes;
- reconstruir modelos LLM e wrappers de cache a cada requisição;
- recriar vector stores e engines SQL já quentes;
- recompilar pipelines RAG e workflows com a mesma configuração;
- reinstanciar ToolsFactory e supervisores que não mudaram;
- perder sessões efêmeras quando múltiplos processos precisam enxergar o mesmo estado curto.

O problema real que ela resolve é custo operacional repetido. Isso inclui custo de CPU, I/O, conexão externa, tempo de inicialização e ruído de observabilidade. Em uma plataforma YAML-first e agentic, montar runtime é caro. Cache aqui é o mecanismo que impede que cada chamada trate o sistema como se estivesse começando do zero.

## 3. Visão executiva

Para liderança, este subsistema reduz desperdício computacional e aumenta previsibilidade operacional. Ele ataca diretamente quatro riscos:

- latência explosiva em fluxos que recompõem runtime muitas vezes;
- consumo desnecessário de recursos externos, como Redis, banco, vector store e provedores de LLM;
- instabilidade operacional por reconstruções concorrentes do mesmo artefato;
- dificuldade de governança quando não existe um caminho claro para invalidação e inspeção.

O ponto executivo importante é que cache, neste projeto, não é um detalhe técnico isolado. Ele é parte do controle de escala e do controle de custo do produto.

## 4. Visão comercial

Do ponto de vista comercial, a plataforma consegue responder melhor a uma objeção comum de clientes corporativos: "o sistema vai ficar lento ou caro quando muitos usuários consultarem a mesma base?"

O código mostra que a resposta honesta é: a plataforma foi desenhada para reaproveitar configuração, pipelines e recursos pesados sempre que a assinatura de execução permite. Isso melhora tempo de resposta em consultas repetidas, reduz reconstrução de contexto e evita refazer trabalho caro quando o cenário não mudou.

O benefício percebido pelo cliente é rapidez mais consistente. O diferencial comercial não é prometer milagre de performance em qualquer cenário, e sim demonstrar que o runtime sabe distinguir quando precisa reconstruir e quando pode reaproveitar com segurança.

Promessa que o código suporta:

- reaproveitamento de artefatos caros por tenant e por hash de configuração;
- invalidação administrativa quando o operador precisa forçar reconstrução;
- observabilidade administrativa para ver o que está ocupado em memória e no Redis.

Promessa que não deve ser feita:

- que toda resposta sempre virá de cache;
- que o Redis sozinho representa todo o sistema de cache;
- que invalidação é instantânea e síncrona em todos os processos sem depender do sinal global e do próximo uso do recurso.

## 5. Visão estratégica

Estratégicamente, o caching fortalece a arquitetura YAML-first porque separa dois problemas diferentes:

- interpretar e montar runtime;
- executar runtime já montado.

Essa separação permite que a plataforma evolua com mais segurança. Novos tipos de tool, novos supervisores, novos slices de retrieval ou novas superfícies administrativas podem entrar no sistema reaproveitando a mesma disciplina de chaveamento, TTL, invalidação global e isolamento por ambiente.

Outro ganho estratégico é a redução de acoplamento temporal. Sem cache, qualquer consulta depende de reconstrução imediata dos mesmos objetos. Com cache, a plataforma consegue trabalhar com tempo de vida controlado de artefatos transitórios, o que prepara melhor o produto para execução paralela, múltiplos workers e cenários multi-tenant.

## 6. Conceitos necessários para entender

### Cache em memória do processo

É o reaproveitamento local dentro de um processo Python. Ele é muito rápido, mas não é automaticamente compartilhado com outros processos. No projeto, esse modelo aparece em pools aquecidos por tenant, caches TTL internos e caches LRU para pipelines.

### Cache distribuído

É o reaproveitamento em uma infraestrutura comum, neste caso Redis. Ele permite compartilhar estado curto entre processos ou pods. No projeto, ele aparece em sessão efêmera, cache YAML quando Redis está disponível, sinais de invalidação global e estatísticas administrativas.

### TTL

TTL significa tempo de vida. O item continua reutilizável só até um limite configurado. No sistema lido, o TTL padrão recorrente é de 900 segundos para vários artefatos do resource pool.

### Chave determinística

É uma chave calculada sempre do mesmo jeito a partir da mesma configuração ou assinatura. O projeto usa hash canônico de YAML, hash de payload sanitizado e prefixo de ambiente para evitar colisão e para permitir reaproveitamento previsível.

### Invalidação global

É o mecanismo que avisa vários caches locais de que devem abandonar seus artefatos. No projeto, isso acontece via timestamp publicado em Redis. O cache local não é apagado por telepatia; ele checa o sinal e se limpa quando percebe que existe uma invalidação mais nova.

### Pool aquecido por tenant

É um conjunto de objetos caros mantidos vivos para um tenant lógico. Em vez de recriar LLM, vector store, engine SQL e tools dinâmicas toda vez, o runtime os mantém em um pool com TTL e invalidação.

## 7. Como a feature funciona por dentro

O fluxo interno do caching começa na construção das chaves. O sistema primeiro deriva uma identidade estável baseada no ambiente e, quando necessário, no YAML sanitizado. Depois ele escolhe a camada de cache correta para aquele tipo de artefato.

Se o artefato for configuração YAML parseada, o caminho passa pelo cache global de YAML, que tenta usar Redis e, na indisponibilidade, mantém um fallback em memória. Se o artefato for runtime quente por tenant, ele passa pelo WarmResourcePool. Se o artefato for pipeline de perguntas e respostas, ele passa pelo PipelineCacheManager. Se o artefato for sessão efêmera, ele passa pelo SessionCacheFactory, que decide o backend disponível.

O sistema também não depende só de expiração. Existe uma governança explícita de invalidação. Endpoints administrativos podem limpar grupos de cache, e o runtime publica um sinal global para que outros processos limpem sua memória local quando forem reutilizar recursos.

## 8. Divisão em etapas ou submódulos

### 8.1 Identidade e chaves de cache

Essa etapa decide como o runtime sabe que dois pedidos são equivalentes para fins de reaproveitamento.

Ela existe para impedir que o cache seja baseado em adivinhação ou em nomes frágeis. O código usa prefixo de ambiente, hash canônico do YAML e assinatura sanitizada de payloads para gerar chaves de tenant, tools dinâmicas, workflows e supervisores.

O valor dessa etapa é governança. Ela reduz o risco de colisão entre ambientes, entre tenants e entre artefatos montados com configurações diferentes.

### 8.2 Cache de infraestrutura Redis

Essa camada existe para o que precisa atravessar processo ou para o que deve ser observado centralmente. O projeto a usa em RedisManager, RedisGenericCache, cache YAML híbrido, sessão efêmera distribuída e sinal global de invalidação.

O ponto importante aqui é que Redis não substitui toda a malha. Ele funciona como infraestrutura compartilhada para partes específicas do sistema.

### 8.3 Cache quente de runtime por tenant

Essa é a espinha dorsal do reaproveitamento em memória. O WarmResourcePool mantém LLMs, vector stores, tools dinâmicas, engines SQL, cache LLM e um cache genérico com TTL opcional.

Essa etapa existe porque esses objetos são caros para recriar e normalmente são reutilizados dentro de uma mesma janela operacional.

### 8.4 Cache especializado de pipeline

O PipelineCacheManager guarda instâncias de ContentQASystem por hash de YAML. Isso resolve um problema específico: reconstruir pipeline RAG inteiro para consultas equivalentes.

Ele é separado porque pipeline não é o mesmo tipo de artefato que LLM, vector store ou tool individual.

### 8.5 Cache de supervisores e workflows compilados

O runtime agentic também reaproveita artefatos de orquestração. Supervisores e DeepAgents podem ser armazenados no pool genérico com hash e versão, e ToolsFactory possui cache global próprio para evitar recriar a mesma fábrica entre supervisores.

Esse submódulo existe para reduzir o custo de assembly agentic, que pode ser tão caro quanto o retrieval em alguns cenários.

### 8.6 Governança e observabilidade administrativa

Sem essa etapa, o sistema teria cache mas não teria operação segura. O código expõe endpoints para limpeza global, invalidação por categoria e snapshots de memória e estatísticas de cache.

Esse bloco existe para o operador conseguir responder perguntas reais: o que está quente, quanto existe por tenant, o que está no Redis e como forçar reconstrução.

## 9. Pipeline ou fluxo principal

O fluxo típico é o seguinte:

1. O runtime recebe YAML ou um contexto já montado.
2. O sistema deriva identidade canônica com prefixo de ambiente e hash sanitizado quando necessário.
3. O componente chamador escolhe a camada correta: sessão, YAML, pool aquecido, pipeline RAG, ToolsFactory ou artefato compilado.
4. O cache verifica se existe item válido dentro do TTL.
5. Se houver hit, devolve o artefato pronto.
6. Se houver miss, entra uma factory controlada que monta o recurso do zero.
7. O recurso novo é armazenado com TTL e metadados mínimos.
8. Antes de novos usos, algumas camadas checam o timestamp global de invalidação.
9. Se existir invalidação mais recente, a memória local é descartada e o próximo acesso recompõe o recurso.

O ponto decisivo é que o sistema não trata todo artefato do mesmo jeito. Cada tipo tem seu próprio contrato de vida útil e de limpeza.

## 10. Decisões técnicas e trade-offs

### Separar cache por tipo de artefato

Ganho: evita uma solução genérica demais, onde sessão, YAML, pipeline e supervisor competiriam com as mesmas regras.

Custo: aumenta a quantidade de componentes a entender.

Impacto: melhora clareza arquitetural e permite invalidação mais precisa.

### Isolar por ambiente e por tenant

Ganho: reduz risco de contaminação cruzada.

Custo: diminui reaproveitamento global absoluto.

Impacto: é uma troca correta para um produto multi-tenant e com múltiplos ambientes.

### Usar Redis como apoio, não como único cache

Ganho: combina compartilhamento distribuído com velocidade local em memória.

Custo: cria comportamento híbrido e exige boa observabilidade.

Impacto: mantém performance local sem abrir mão de coordenação global quando necessário.

### Sinal global por timestamp em vez de limpeza remota síncrona

Ganho: simplifica coordenação entre processos.

Custo: a limpeza é observada sob demanda, no próximo uso.

Impacto: é um modelo mais leve, mas exige explicar que invalidação não é mágica instantânea em todos os workers ao mesmo tempo.

## 11. Configurações que mudam o comportamento

As configurações confirmadas no código lido que mudam comportamento incluem:

- ENVIRONMENT: entra no prefixo das chaves e isola ambientes.
- REDIS_ATIVO e URL do Redis gerenciada pelo RedisManager: definem se o caminho distribuído está disponível.
- SESSION_CACHE_DIR: define o diretório do backend de arquivo da SessionCacheFactory.
- llm.cache.enabled: permite desligar o cache de LLM no YAML.
- llm.cache.redis.ttl_seconds, namespace, url e use_manager: alteram backend, TTL e namespace do cache de LLM.
- pristine_hash em `_config_metadata`: altera a forma como o hash canônico do YAML é calculado.

Valor padrão não confirmado no código lido para qualquer opção não explicitamente mostrada acima não deve ser presumido.

## 12. O que acontece em caso de sucesso

Quando o cache funciona como esperado, o operador percebe menos reconstrução e o sistema registra reaproveitamento. O pipeline RAG pode ser reutilizado pelo hash do YAML, o supervisor pode voltar do pool sem reconstrução completa e o cache stats administrativo passa a mostrar artefatos vivos por tenant e por categoria.

O sucesso também aparece indiretamente em menor churn de conexões, menos custo de montagem e menor tempo de resposta nos fluxos repetidos.

## 13. O que acontece em caso de erro

O código confirma alguns comportamentos importantes:

- se Redis estiver desabilitado para sessão, a SessionCacheFactory tenta backend de arquivo e, se necessário, memória;
- se o cache YAML não conseguir operar com Redis, ele mantém fallback em memória;
- se a leitura do timestamp global de invalidação falhar, o pool registra warning e segue operando com o estado local;
- se um cache especializado detectar incompatibilidade de versão ou hash, ele invalida a entrada e reconstrói.

O ponto operacional é que o sistema prefere continuar funcional em algumas superfícies, mas isso reduz consistência distribuída. Esse é um risco real e precisa ser interpretado como degradação observável, não como equivalência perfeita entre backends.

## 14. Observabilidade e diagnóstico

O subsistema de cache é observável por dois caminhos:

- logs de cache hit, cache miss, invalidação e reconstrução em componentes como supervisor, DeepAgent e pipeline RAG;
- endpoints administrativos que expõem snapshot de memória e estatísticas consolidadas de cache.

O valor prático disso é distinguir rapidamente três cenários:

- problema de reaproveitamento, quando o cache nunca é atingido;
- problema de invalidação excessiva, quando tudo é limpo o tempo inteiro;
- problema de backend distribuído, quando Redis está indisponível e o sistema cai para memória local.

## 15. Impacto técnico

Tecnicamente, a feature encapsula complexidade de tempo de vida de artefatos, reduz acoplamento entre criação e uso de runtime e reforça uma arquitetura mais escalável. Ela também melhora concorrência ao usar locks por chave em caches TTL e ao reaproveitar engines, tools e pipelines de forma mais controlada.

## 16. Impacto executivo

Executivamente, a principal consequência é melhor previsibilidade de custo e de tempo de resposta. O sistema evita que uma mesma configuração consuma recursos como se fosse inédita a cada chamada.

## 17. Impacto comercial

Comercialmente, isso sustenta a narrativa de plataforma pronta para uso corporativo repetitivo, com invalidação controlada e observabilidade administrativa, em vez de uma aplicação que reconstrói tudo a cada operação.

## 18. Impacto estratégico

Estrategicamente, o caching completa a proposta de plataforma reutilizável. YAML-first sem cache vira montagem repetitiva. Agentic sem cache vira assembly caro demais. Operação sem cache observável vira caixa-preta. Este subsistema conecta esses três pontos.

## 19. Explicação 101

Uma forma simples de entender é pensar no sistema como uma cozinha profissional. Alguns ingredientes precisam ficar prontos ou semiprontos para que o serviço não recomece do zero a cada prato. Mas nem tudo pode ficar guardado do mesmo jeito. Há itens que ficam no balcão do cozinheiro, itens que vão para a câmara fria e itens que precisam ser refeitos quando muda o pedido.

Neste projeto, o cache faz exatamente isso. Ele decide o que vale a pena manter pronto por alguns minutos, para quem aquilo vale, quando precisa ser jogado fora e como o operador confere se o estoque transitório ainda faz sentido.

## 20. Limites e pegadinhas

- Redis não representa todo o sistema de cache.
- Invalidação global não significa limpeza síncrona imediata de toda memória local.
- Cache hit não significa que todo o fluxo veio de cache; pode ter sido só uma etapa específica.
- Fallback para arquivo ou memória mantém o sistema de pé, mas muda a semântica de compartilhamento entre processos.
- TTL reduz risco de staleness, mas não substitui invalidação administrativa quando a mudança precisa ser imediata.

## 21. Troubleshooting

### Sintoma: tudo está lento mesmo com tráfego repetido

Causa provável: misses constantes por chave instável, hash mudando ou invalidação excessiva.

Como confirmar: verifique logs de pipeline e DeepAgent para hit ou miss e consulte o snapshot administrativo por tenant.

### Sintoma: workers não compartilham o mesmo reaproveitamento

Causa provável: o backend relevante está em memória local ou a superfície não usa Redis compartilhado.

Como confirmar: consulte cache stats e valide se Redis está disponível e habilitado para aquele tipo de cache.

### Sintoma: operador limpou cache, mas alguns processos ainda parecem usar artefatos antigos

Causa provável: a limpeza depende do sinal global e do próximo uso do cache local.

Como confirmar: procure logs de invalidação global aplicada no resource pool e no PipelineCacheManager.

## 22. Checklist de entendimento

- Entendi que não existe um único cache central.
- Entendi a diferença entre cache local, distribuído e especializado.
- Entendi por que YAML, pipeline, supervisor e sessão não usam exatamente o mesmo contrato.
- Entendi o papel do TTL.
- Entendi o papel do hash canônico do YAML.
- Entendi o papel da invalidação global por timestamp.
- Entendi o papel da API administrativa de cache.
- Entendi os riscos do fallback para memória ou arquivo.
- Entendi o valor executivo, comercial e estratégico do subsistema.

## 23. Evidências no código

- src/core/resource_pool.py
  - Motivo da leitura: confirmar cache YAML híbrido, pools por tenant, recursos aquecidos, invalidação global e estatísticas.
  - Símbolos relevantes: RedisYamlCache, WarmResourcePool, get_pool, get_global_yaml_cache.
  - Comportamento confirmado: cache local por tipo de artefato, fallback do YAML para memória e limpeza por sinal global.

- src/security/session_cache.py
  - Motivo da leitura: confirmar contrato do cache de sessão efêmera.
  - Símbolos relevantes: SessionCacheFactory, RedisSessionCache, FileSessionCache, InMemorySessionCache.
  - Comportamento confirmado: seleção automática de backend Redis, arquivo ou memória.

- src/qa_layer/pipeline_cache_manager.py
  - Motivo da leitura: confirmar cache especializado de ContentQASystem.
  - Símbolos relevantes: PipelineCacheManager, CachedPipelineEntry.
  - Comportamento confirmado: cache por hash canônico do YAML com métricas, TTL e invalidação global.

- src/agentic_layer/supervisor/supervisor_cache.py
  - Motivo da leitura: confirmar cache compartilhado de ToolsFactory.
  - Símbolos relevantes: get_or_create_tools_factory, invalidate_tools_factory_cache.
  - Comportamento confirmado: cache global de ToolsFactory com TTL e sinal global de invalidação.

- src/api/services/admin/cache_service.py
  - Motivo da leitura: confirmar governança administrativa e observabilidade.
  - Símbolos relevantes: get_cache_stats, invalidate_llm_cache, invalidate_yaml_cache, get_memory_snapshot.
  - Comportamento confirmado: inspeção e invalidação por categoria com visão por tenant e por backend.
