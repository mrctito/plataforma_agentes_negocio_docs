# Manual técnico por etapa: replay e auditoria do AG-UI

## 1. O que esta etapa cobre

Esta etapa cobre o event store append only, o provider canônico de persistência e as rotas de replay usadas por suporte, auditoria e troubleshooting.

## 2. Portas e implementações confirmadas

O slice define uma porta AgUiEventStore e duas implementações concretas:

- InMemoryAgUiEventStore para development e test
- PostgresAgUiEventStore para persistência durável

O router não decide isso por conta própria. Ele consome get_canonical_ag_ui_event_store, que resolve o provider canônico com base no ambiente e nas variáveis operacionais.

## 3. Regras do provider canônico

As regras confirmadas no código são estas:

- development, test e testing podem usar memory
- fora desses ambientes, AG_UI_EVENT_STORE_PROVIDER precisa ser configurado explicitamente
- provider memory fora de development e test é recusado
- provider suportado hoje é memory ou postgres

Isso impede que produção caia silenciosamente para replay efêmero.

## 4. Garantias do store

As garantias úteis desta etapa são:

- append only
- sequence monotônica por run
- idempotência por run_id e sequence quando o payload é idêntico
- erro explícito quando a mesma sequence chega com payload divergente
- sanitização recursiva de campos sensíveis antes de persistir ou reproduzir

Os campos mais sensíveis, como api_key, authorization, password, token, dsn, connection string, database URL, encrypted_data, correlation_id interno e chaves de SQL livre (`sql`, `raw_sql`, `sql_query`, `statement`) são redigidos para um marcador seguro.

## 5. Replay por run e por thread

O slice oferece duas leituras:

- replay de um run específico
- replay da thread inteira

Isso permite investigar tanto uma execução isolada quanto a narrativa completa de uma conversa com múltiplos runs.

## 6. Erros e limites relevantes

Os problemas mais importantes nesta etapa são:

- provider não configurado fora de development e test
- provider inválido ou não suportado
- falha transitória de banco na persistência ou leitura
- conflito de sequence com payload divergente
- uso de replay efêmero em ambiente que precisava de histórico durável
- payload com SQL livre, DSN ou correlation_id interno que precisa aparecer redigido no replay

No estado atual, a retenção do provider PostgreSQL não tem TTL automático no adapter. Em termos práticos, os eventos ficam duráveis até a operação aplicar política de limpeza na tabela canônica. O ponto importante para o boundary é que produção não usa memória por fallback escondido.

## 7. Diagnóstico recomendado

Para investigar replay e auditoria:

1. confirme ENVIRONMENT e AG_UI_EVENT_STORE_PROVIDER
2. valide se o ambiente pode ou não usar memory
3. em postgres, verifique DSN, schema e table configurados
4. se houver falha de escrita, inspecione se a sequence foi repetida com payload divergente
5. no suporte funcional, compare replay por run e replay por thread para entender a narrativa inteira

## 8. Evidências no código

- src/api/services/ag_ui_event_store.py
  - Motivo: porta, providers e regras canônicas.
  - Comportamento confirmado: replay usa provider resolvido por ambiente, sem fallback implícito fora de development e test.
- src/api/services/ag_ui_event_store.py
  - Motivo: segurança do histórico.
  - Comportamento confirmado: payload sensível é sanitizado antes de persistência e replay.
- src/api/routers/ag_ui_router.py
  - Motivo: superfícies públicas de auditoria.
  - Comportamento confirmado: o boundary expõe replay por run e por thread usando o provider canônico.
