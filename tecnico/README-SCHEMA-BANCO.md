# Produto: Plataforma de Agentes de IA

## Manual do Esquema de Banco de Dados

Guia de referência do schema PostgreSQL utilizado pela Plataforma de Agentes de IA Generic RAG.
Este documento descreve o schema público com base no DDL atual informado para o ambiente.
Ao final, ele tambem documenta o schema implementado da solucao de integracoes, para que a modelagem persistente do modulo nao fique solta fora do manual oficial.

## Visão Geral

- Banco suportado: PostgreSQL.
- Tabelas que usam `gen_random_uuid()` dependem de suporte a `pgcrypto`.
- O schema atual está organizado, na prática, em oito grupos:
- estado e checkpoints,
- job core e ledger operacional de jobs,
- autenticação e login,
- ingestão de conteúdo,
- interações, eventos e aprovações humanas,
- execução agentic em background,
- tenants e segurança,
- memória de usuário.

### Estado do modelo de ingestão vetorial `vector_*` (comprovado em runtime — 2026-06-09)

Existe um **modelo novo de ingestão vetorial** (família `vector_*`: `vector_dataset_master`,
`vector_ingestion_runs`, `vector_ingestion_run_documents`, `vector_active_documents` e tabelas
filhas de chunks/páginas). Após a migração `migracao-modelo-vetorial-ingestao`, o estado
**comprovado na fonte de verdade real** (PostgreSQL) é:

- **Escrita ligada e provada:** o caminho assíncrono de ingestão de PDF (com fan-out por
  documento) GRAVA no modelo novo — abre o run de negócio, registra os documentos vistos no
  lote (com `publication_action`) e publica o acervo vivo. Provado para o dataset de TESTE
  `dnit_teste` (tenant `engenharia_dnit`): `vector_ingestion_runs`=1,
  `vector_ingestion_run_documents`=11, `vector_active_documents`=5.
- **Leitura da tela e guard do RAG no modelo novo:** a leitura operacional da tela de ingestão
  (histórico de runs, documentos do lote, acervo vivo, resumo) e o guard de documento vivo do
  RAG (`ActiveDocumentVersionGuard`) consomem o modelo novo via `VectorActiveArchiveRepository`.
  A leitura legada de `ingestion_*` no caminho da tela foi cortada (sem fallback).
- **Corte PARCIAL por dado, não por código:** o modelo novo está POPULADO apenas para o dataset
  de teste. PRODUÇÃO (`dnit_producao`) ainda **não** foi reconstruída (rebuild), então seu
  acervo novo está vazio (`vector_active_documents`=0, ponteiros do master nulos). Por isso o
  guard do RAG é ESTRITO só onde o acervo novo está populado e TOLERANTE onde está vazio,
  evitando RAG vazio em silêncio na produção.
- **Tabelas antigas `ingestion_*`:** continuam existindo e são usadas pelo control plane do Job
  Core, streaming/status ao vivo, worker e manutenção (reconciliação). Elas **não** são mais a
  fonte de verdade do histórico/acervo de NEGÓCIO no caminho da tela.

## Que problema este manual resolve

Schema de banco não é detalhe de infraestrutura. Neste projeto, ele é
parte do contrato operacional de ingestão, autenticação, memória,
interação, governança humana e integrações. Sem um manual único, cada
time acabaria interpretando tabela, chave e relação crítica de um jeito,
o que abre espaço para diagnóstico errado, consulta insegura e evolução
de produto desconectada do dado persistido.

## Visão executiva

Executivamente, este manual reduz risco de leitura equivocada do estado
do produto. Ele ajuda liderança técnica e operação a distinguir dataset
vivo de histórico, identidade lógica de documento de hash de conteúdo e
cadastro governado de execução em runtime.

## Visão comercial

Comercialmente, um schema bem explicado sustenta promessas difíceis de
fazer sem evidência, como rastreabilidade de ingestão, retomada de HIL,
memória de usuário, multi-tenant com isolamento e catálogo governado de
integrações. O cliente pode não ver a tabela, mas percebe o efeito
quando a plataforma consegue auditar e explicar o próprio estado.

## Visão estratégica

Estrategicamente, o schema mostra onde a plataforma consolida ativos
duráveis. Ele é a base que permite conectar ingestão, RAG, autenticação,
governança humana e integrações sem depender de memória informal ou de
contratos espalhados.

## Convenções Gerais

- Campos `created_at` e `updated_at` usam `timestamptz` com default `now()` quando a tabela precisa de auditoria temporal.
- Campos `metadata`, `metadata_json`, `permissions_json`, `rate_limits_json`, `keys_json`, `payload_json`, `raw_claims` e `evidence_summary` usam `jsonb` para dados flexíveis.
- O tipo `public.ingestion_document_type` é uma dependência obrigatória para as tabelas de ingestão que o utilizam.
- Há colunas geradas automaticamente pelo banco:
- `ingestion_document_manifest.status` espelha `ingestion_status`.
- `ingestion_document_chunks.fts_content` gera o índice textual a partir de `text_content`.
- `interaction_runs.total_tokens` soma `input_tokens` e `output_tokens`.

## Regra de Integridade do Dataset de Ingestão

- Para o produto, BM25, PostgreSQL e banco vetorial formam um único dataset operacional do acervo.
- O eixo lógico desse dataset é `tenant_code + vectorstore_id`.
- O parâmetro `vector_store.if_exists` deve reger o conjunto vivo inteiro do acervo, e não apenas o provider vetorial.
- Regra obrigatória de identidade: `vectorstore_id` é sempre o identificador lógico do acervo. Ele não é o nome físico garantido do provider.
- Regra obrigatória de alvo físico: `physical_vector_target` é sempre o nome físico efetivamente usado pelo banco vetorial ativo para o par `tenant_code + vectorstore_id`.
- No Qdrant, o alvo físico é a `collection_name`. No Azure Search, o alvo físico é o `index_name`.
- Regra de naming de novos alvos físicos: o target novo precisa carregar, de forma sanitizada e compatível com o provider, os quatro componentes operacionais mínimos `provider + tenant + vectorstore + geração`.
- Exemplos conceituais de target físico novo: no Qdrant, um alvo pode assumir formato como `qdrant_tenant_demo_store_principal_g3`; no Azure Search, um alvo pode assumir formato como `azure_tenant_demo_store_principal_g3`.
- Regra prática de runtime: o caminho canônico não pode deduzir `index_name` do Azure a partir de `vector_store.id` por conveniência. O `index_name` físico precisa vir do lifecycle por meio de `physical_vector_target`.
- Regra de compatibilidade: target físico legado já persistido continua válido enquanto estiver ativo no lifecycle. O runtime não pode recalcular o nome físico por conveniência nem substituir silenciosamente o valor salvo no banco.
- Em termos práticos, `overwrite`, `update` e `skip` precisam produzir o mesmo efeito semântico sobre:
- manifesto e tabelas derivadas do documento no PostgreSQL,
- índice BM25 persistido,
- banco vetorial ativo, seja Qdrant ou Azure Search.
- Regra operacional de `overwrite`: a limpeza destrutiva do dataset vivo remove `ingestion_document_manifest`, as tabelas filhas dependentes do manifesto e o BM25 correspondente da geração substituída.
- Histórico operacional, auditoria, runs e evidências de execução não fazem parte da limpeza destrutiva do dataset vivo e devem seguir política própria de retenção.
- Consequência prática de `overwrite`: `ingestion_runs`, `ingestion_run_documents`, `ingestion_datasets` e `ingestion_dataset_generations` permanecem preservados como trilha operacional. Quando o manifesto antigo é removido, `ingestion_run_documents.manifest_id` pode ficar nulo para manter a auditoria do run sem reapontar para um documento que já não faz parte do acervo vivo.
- Consequência obrigatória: o sistema não pode considerar sucesso quando apenas uma dessas materializações foi atualizada e as demais ficaram antigas.

## Relações Principais

- `ingestion_datasets` é a entidade central do dataset vivo por `tenant_code` e `vectorstore_id`.
- `ingestion_dataset_generations` materializa cada geração preparada, ativa, abortada, falha ou substituída do dataset.
- `ingestion_document_manifest` é a entidade central do domínio de documentos.
- `ingestion_document_chunks`, `ingestion_document_pages` e `ingestion_document_images` dependem do manifesto do documento.
- `ingestion_runs` representa uma execução de ingestão, `ingestion_run_documents` guarda o detalhe de cada documento dentro dessa execução e `ingestion_run_slot_leases` materializa o controle de concorrência do fan-out por documento sem concentrar contenção na linha pai.
- `interaction_runs` representa a execução principal de uma interação e `interaction_run_events` guarda os eventos associados.
- O schema `job_core` concentra o ledger canônico do runtime assíncrono de jobs, por meio de `job_core.job_runs` e `job_core.job_run_events`.
- O schema `ag_ui` concentra o replay durável do protocolo AG-UI, separando a trilha visual de runs e threads do ledger de background e do histórico genérico de interação.
- O schema `scheduler` concentra a agenda canônica e o ledger canônico de execução, por meio de `scheduler.scheduled_jobs` e `scheduler.job_executions`.
- `agent_hil_approval_requests` representa a pausa Human-in-the-Loop assíncrona de agentes em background, guardando o pedido de aprovação, o canal esperado, o token seguro, a decisão e a trilha mínima de auditoria para retomada.
- O schema `agent_background` concentra a capacidade de Execução Agentic em Background, separando alvo autorizado, solicitação criada por prompt, projeção compatível de run, eventos, HIL durável ligado ao run e outbox operacional.
- `user_accounts` é a entidade central da conta pessoal do usuário.
- `user_auth_identities`, `user_password_credentials`, `user_account_yaml` e `user_account_payment_cards` dependem de `user_accounts`.
- `tenants` é a entidade central do domínio de organizações.
- `system_domains` é o catálogo global de domínios funcionais disponíveis para projetos organizacionais.
- `tenant_access_keys`, `tenant_security_keys`, `tenant_secrets`, `tenant_channels`, `tenant_channel_end_users`, `tenant_users`, `tenant_user_yaml`, `tenant_user_projects`, `tenant_payment_cards` e `tenant_audit_log` dependem de `tenants`.
- `tenant_users` também depende de `user_accounts` para representar membership organizacional.
- `tenant_user_projects` também depende de `system_domains` para classificar o projeto dentro de um domínio funcional explícito.
- `tenant_user_project_details` depende de `tenant_user_projects`.

## Domínio Estado e Checkpoints

### bm25_indexes

- Finalidade prática: persistir o índice BM25 pela materialização física da geração ativa ou preparada.
- Papel no dataset vivo: esta tabela faz parte do mesmo conjunto operacional do acervo controlado por `tenant_code + vectorstore_id`, mas a chave operacional do BM25 é física, não lógica. Na prática, ela representa o índice textual da geração materializada apontada pelo lifecycle do dataset.
- Chave primária: `bm25_target_id`.
- Colunas:
- `bm25_target_id`: identificador físico do índice BM25 materializado para uma geração específica.
- `tenant_code`: tenant dono do índice materializado.
- `vectorstore_id`: identificador lógico do acervo, mantido para diagnóstico e filtros operacionais.
- `generation_id`: geração do dataset à qual este índice BM25 pertence.
- `schema_version`: versão do formato persistido.
- `entries_count`: quantidade de entradas no índice.
- `documents_count`: quantidade de documentos representados.
- `checksum`: hash de integridade do índice persistido.
- `owner_email`: e-mail do responsável, quando existir.
- `payload`: conteúdo serializado do índice em `bytea`.
- `created_at`: criação do registro.
- `updated_at`: última atualização do registro.
- `detection_keywords`: palavras auxiliares de detecção em `jsonb`.
- `query_expansion_vocabulary`: vocabulário auxiliar para expansão em `jsonb`.
- `vocabulary_stats`: estatísticas do vocabulário em `jsonb`.
- `last_vocabulary_at`: última atualização do vocabulário derivado.
- Índices e restrições: PK em `bm25_target_id`; índice operacional por `tenant_code, vectorstore_id, generation_id`; relação com `ingestion_dataset_generations(generation_id)` para manter rastreabilidade da geração dona do índice.

### checkpoint_migrations

- Finalidade prática: controlar a versão aplicada das migrações do subsistema de checkpoint.
- Chave primária: `v`.
- Colunas:
- `v`: número da versão aplicada.

### checkpoint_blobs

- Finalidade prática: guardar blobs serializados por thread, namespace, canal e versão.
- Chave primária composta: `thread_id`, `checkpoint_ns`, `channel`, `version`.
- Colunas:
- `thread_id`: identificador da execução.
- `checkpoint_ns`: namespace do checkpoint, com default vazio.
- `channel`: canal do dado persistido.
- `version`: versão do blob.
- `type`: tipo lógico do blob.
- `blob`: conteúdo binário serializado.
- Índices e restrições:
- PK composta em `thread_id, checkpoint_ns, channel, version`.
- Índice `checkpoint_blobs_thread_id_idx` em `thread_id`.

### checkpoint_writes

- Finalidade prática: guardar fragmentos escritos durante o ciclo de checkpoints.
- Chave primária composta: `thread_id`, `checkpoint_ns`, `checkpoint_id`, `task_id`, `idx`.
- Colunas:
- `thread_id`: identificador da execução.
- `checkpoint_ns`: namespace do checkpoint, com default vazio.
- `checkpoint_id`: identificador do snapshot.
- `task_id`: identificador da tarefa.
- `idx`: posição do fragmento.
- `channel`: canal da escrita.
- `type`: tipo lógico do conteúdo.
- `blob`: conteúdo binário persistido.
- `task_path`: caminho da tarefa, com default vazio.
- Índices e restrições:
- PK composta em `thread_id, checkpoint_ns, checkpoint_id, task_id, idx`.
- Índice `checkpoint_writes_thread_id_idx` em `thread_id`.

### checkpoints

- Finalidade prática: guardar o snapshot principal do estado por execução.
- Chave primária composta: `thread_id`, `checkpoint_ns`, `checkpoint_id`.
- Colunas:
- `thread_id`: identificador da execução.
- `checkpoint_ns`: namespace do checkpoint, com default vazio.
- `checkpoint_id`: identificador do snapshot.
- `parent_checkpoint_id`: checkpoint pai, quando houver.
- `type`: tipo lógico do checkpoint.
- `checkpoint`: estado serializado em `jsonb`.
- `metadata`: metadados auxiliares em `jsonb`, com default objeto vazio.
- Índices e restrições:
- PK composta em `thread_id, checkpoint_ns, checkpoint_id`.
- Índice `checkpoints_thread_id_idx` em `thread_id`.

## Domínio Job Core

### job_core.job_runs

- Finalidade prática: registrar o ciclo de vida canônico de cada job assíncrono aceito pelo Job Core V1.
- Papel operacional: esta tabela é o ledger principal do runtime assíncrono. Em termos simples, ela responde qual job entrou, qual handler ficou responsável, de qual correlação esse job faz parte, se ele é raiz ou filho, em que estado está e como terminou.
- Relação com o transporte: o publish oficial monta `QueuedJobEnvelope`, o worker consome esse envelope e o Job Core persiste a evolução do job aqui. Isso separa claramente transporte de mensageria de estado operacional do trabalho.
- Evolução de schema já existente: o DDL oficial não depende apenas de `CREATE TABLE IF NOT EXISTS`. Ele também executa `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` e adiciona checks faltantes quando necessário, para impedir falso verde em ambiente onde o schema `job_core` já existia sem o envelope completo.
- Chave primária: `job_id`.
- Colunas:
- `job_id`: identificador único do job.
- `route_kind`: categoria operacional do envelope, usada junto com `dispatch_mode` para resolver o handler correto no runtime.
- `dispatch_mode`: modo operacional do envelope, usado para distinguir fluxos como `prepared_yaml`, `resolve_on_worker`, `document_fanout_child`, `background_execution_run` e equivalentes.
- `job_type`: tipo lógico do job recebido pelo envelope.
- `handler_key`: chave do handler resolvido pelo runtime para executar o job.
- `correlation_id`: identificador lógico único da execução de ponta a ponta.
- `parent_job_id`: job pai, quando este registro representa um job filho.
- `root_job_id`: identificador do job raiz da árvore.
- `envelope_payload`: payload bruto e opaco do envelope recebido pelo worker, em `jsonb`.
- Observação operacional sobre `envelope_payload`: é aqui que continuam aparecendo campos especializados do transporte e do domínio, como `worker_execution_correlation_id`, `run_id`, `encrypted_data`, `parent_run_id`, `execution_id` e equivalentes. Eles não viram colunas top-level do ledger genérico.
- `envelope_metadata`: metadados originais do envelope, em `jsonb`.
- `status`: estado atual do job. O DDL confirma os valores `queued`, `claimed`, `running`, `cancelling`, `cancel_requested`, `cancelled`, `succeeded`, `partial_success`, `failed`, `stale`, `orphaned` e `reconciled_failed`.
- `final_reason`: motivo final registrado para encerramento, quando existir.
- `output_payload`: saída estruturada do job em `jsonb`.
- `metadata`: metadados estruturados do runtime em `jsonb`.
- `owner_worker_id`: identificador durável do worker que assumiu o claim mais recente.
- `claimed_at`: instante em que o worker reservou o job no ledger.
- `last_heartbeat_at`: instante do último heartbeat persistido pelo runtime para provar liveness.
- `created_at`: instante de criação do registro.
- `started_at`: instante em que o handler iniciou a execução.
- `finished_at`: instante em que a execução terminou.
- Índices e restrições:
- PK em `job_id`.
- Índice `ix_job_runs_correlation_id` em `correlation_id` para investigação por correlação.
- Índice `ix_job_runs_root_job_created_at` em `root_job_id, created_at desc` para reconstruir árvores de execução.
- Check `job_runs_status_check` limitando `status` aos estados canônicos do runtime.
- Checks `job_runs_envelope_payload_json_check`, `job_runs_envelope_metadata_json_check`, `job_runs_output_payload_json_check` e `job_runs_metadata_json_check` exigindo objeto JSON.
- Check `job_runs_finished_after_started_check` impedindo término anterior ao início.

### job_core.job_run_events

- Finalidade prática: guardar a trilha de eventos estruturados emitidos durante o ciclo de vida de cada job.
- Papel operacional: esta tabela não substitui `job_core.job_runs`. Ela detalha a história do job. Em termos simples, `job_runs` responde “como o job está”; `job_run_events` responde “o que aconteceu com ele ao longo do caminho”.
- Relação com `job_core.job_runs`: cada evento pertence a um job e é removido em cascata se o job for removido.
- Chave primária: `event_id`.
- Colunas:
- `event_id`: identificador UUID do evento.
- `job_id`: job dono do evento.
- `event_name`: nome canônico do evento. O DDL confirma eventos como `job_core.envelope.received`, `job_core.envelope.validated`, `job_core.envelope.rejected`, `job_core.execution.claimed`, `job_core.execution.executed`, `job_core.execution.stale`, `job_core.execution.orphaned`, `job_core.execution.reconciled_failed`, `job_core.execution.cancelled`, `job_core.execution.failed`, `job_core.execution.partial_success` e `job_core.execution.succeeded`.
- `status`: estado associado ao evento. O conjunto válido repete os estados canônicos do runtime.
- `event_payload`: detalhes estruturados do evento em `jsonb`.
- `created_at`: instante de criação do evento.
- Índices e restrições:
- PK em `event_id`.
- FK `job_run_events_job_fk` para `job_core.job_runs(job_id)` com `ON DELETE CASCADE`.
- Índice `ix_job_run_events_job_created_at` em `job_id, created_at desc` para leitura cronológica por job.
- Check `job_run_events_event_name_check` limitando o conjunto de eventos canônicos.
- Check `job_run_events_status_check` limitando `status` aos estados válidos do runtime.
- Check `job_run_events_payload_json_check` exigindo objeto JSON.

### Como o processamento paralelo agnóstico usa o schema `job_core`

- Escopo correto: o mecanismo genérico e agnóstico de jobs paralelos usa diretamente apenas `job_core.job_runs` e `job_core.job_run_events` como ledger durável do worker. Tabelas como `ingestion_runs`, `ingestion_run_documents`, `ingestion_run_slot_leases`, `agent_background.*` e `scheduler.*` pertencem a especializações ou a slices produtores, e não substituem o ledger canônico do runtime genérico.
- O que entra em `job_core.job_runs` no primeiro write: `PostgresJobRunStore.create_run(...)` persiste a identidade mínima do envelope aceito pelo worker, gravando `job_id`, `route_kind`, `dispatch_mode`, `job_type`, `handler_key`, `correlation_id`, `parent_job_id`, `root_job_id`, `envelope_payload`, `envelope_metadata`, `status`, `final_reason`, `output_payload`, `metadata`, `owner_worker_id`, `claimed_at`, `last_heartbeat_at`, `created_at`, `started_at` e `finished_at`.
- Em termos práticos de ingestão, isso significa que o job pai `prepared_yaml`, o job pai `resolve_on_worker` e o filho `document_fanout_child` entram no mesmo ledger e se diferenciam por `route_kind + dispatch_mode` e pelo conteúdo opaco de `envelope_payload`.
- O que muda ao longo da execução: `PostgresJobRunStore.transition_status(...)` atualiza `status` como fonte de verdade do lifecycle. Quando o job entra em execução, o método preenche `started_at`. Quando termina, também grava `finished_at`, `final_reason`, `output_payload` e o `metadata` operacional mais recente.
- O que entra em `job_core.job_run_events`: `PostgresJobRunStore.append_event(...)` cria a trilha estruturada com `job_id`, `event_name`, `status`, `event_payload` e `created_at`. Essa tabela é usada para reconstruir a narrativa do job sem depender só de log bruto.
- A sequência canônica de eventos nasce do `JobCoreExecutor`, do `OperationalRunReconciliationService` e do catálogo `JobCoreEventName`. Em linguagem simples, o banco guarda a trilha `received -> validated -> claimed -> executed -> terminal` e, quando o worker some sem heartbeat, também registra `stale/orphaned -> reconciled_failed` para fechar o lifecycle sem requeue implícito.
- Colunas que sustentam o paralelismo genérico:
- `route_kind` + `dispatch_mode`: formam a chave operacional que o runtime usa para localizar o handler correto. Em linguagem simples, são os dois campos que dizem “para qual trilho esse envelope vai”.
- `parent_job_id` + `root_job_id`: materializam a árvore pai-filhos. Isso permite rastrear fan-out, agregação e investigações por lote sem criar tabela extra só para a hierarquia.
- `correlation_id`: costura todos os jobs da mesma execução lógica. Quando vários jobs filhos correm em paralelo, é esse campo que permite juntar banco, logs e eventos sob a mesma investigação.
- `status`, `claimed_at`, `last_heartbeat_at`, `started_at`, `finished_at` e `final_reason`: formam o estado observável do job. Em termos simples, respondem se o job só foi enfileirado, se algum worker já assumiu o claim, se ainda existe prova recente de liveness, como terminou e quando isso aconteceu.
- `envelope_payload`, `envelope_metadata`, `output_payload`, `metadata` e `event_payload`: carregam o conteúdo opaco de entrada, saída e telemetria por especialização. O core genérico não tenta abrir coluna própria para cada domínio; ele preserva esses dados em `jsonb` e mantém fixa só a casca operacional comum.
- Decisão arquitetural importante: identificadores especializados, como `execution_id`, `schedule_id`, `request_id`, `run_id`, `manifest_id` ou `run_document_id`, não viram colunas top-level do ledger genérico. Eles entram no `envelope_payload` quando a especialização precisa deles. Isso mantém o Job Core agnóstico ao domínio e evita que o schema canônico cresça a cada novo tipo de job.
- Leitura operacional mínima do runtime genérico:
- `get_run(job_id=...)` lê o estado terminal ou intermediário de um job específico.
- `list_stale_runs(...)` identifica jobs ativos cujo heartbeat expirou e que precisam de reconciliação explícita.
- `list_events(job_id=...)` reconstrói a sequência `received -> validated -> executed -> terminal` do job.
- o índice `ix_job_runs_correlation_id` sustenta investigação transversal por correlação.
- o índice `ix_job_runs_root_job_created_at` sustenta reconstrução rápida da árvore de jobs mais recente sob a mesma raiz.
- Limites explícitos do V1: o DDL oficial e seus testes confirmam que o ledger genérico não modela `retry_wait`, `dead_letter` nem `serialize`. Esses conceitos não fazem parte do contrato canônico do Job Core atual e não devem ser inferidos como colunas ausentes “a completar”.

## Domínio Autenticação e Login

### federated_login_audit

- Finalidade prática: auditar sessões de login federado e o estado de TOTP.
- Chave primária: `session_id`.
- Colunas:
- `session_id`: identificador da sessão.
- `provider_id`: provedor federado.
- `subject`: identificador do usuário no provedor.
- `email`: e-mail autenticado.
- `email_verified`: indica se o e-mail foi confirmado pelo provedor.
- `token_audience`: audience do token.
- `token_issuer`: emissor do token.
- `token_issued_at`: data de emissão do token.
- `token_expires_at`: data de expiração do token.
- `issued_at`: instante lógico do login processado.
- `created_at`: criação do registro.
- `raw_claims`: claims completas em `jsonb`.
- `totp_secret_encrypted`: segredo TOTP cifrado.
- `totp_enabled`: indica se TOTP está habilitado.
- `totp_confirmed_at`: momento de confirmação do TOTP.
- `totp_last_verified_at`: última verificação bem-sucedida.
- `totp_recovery_codes_encrypted`: códigos de recuperação cifrados.
- `totp_failed_attempts`: contador de falhas consecutivas.
- `totp_locked_until`: data até a qual o TOTP permanece bloqueado.
- Índices e restrições:
- PK em `session_id`.
- Índice `federated_login_audit_email_idx` em `email`.
- Índice parcial `federated_login_audit_totp_email_idx` em `email` quando `totp_enabled` é verdadeiro.
- Índice parcial `federated_login_audit_totp_locked_idx` em `totp_locked_until` quando o campo não é nulo.

## Domínio Ingestão de Conteúdo

### ingestion_datasets

- Finalidade prática: representar o dataset lógico vivo do acervo por `tenant_code` e `vectorstore_id`.
- Papel no ciclo de vida: esta tabela é a fonte de verdade do dataset ativo. Ela aponta qual geração está visível para leitura e separa o acervo vivo do histórico operacional de runs.
- Uso na ingestão de PDF: antes de gravar páginas, chunks e imagens, o runtime resolve qual dataset lógico está valendo para o par `tenant_code + vectorstore_id`. Em termos simples, esta tabela responde: “qual acervo oficial está ativo agora para este cliente e esta coleção?”.
- O que ela grava no fluxo de PDF: política `if_exists`, status do dataset, geração ativa, alvo físico do banco vetorial e alvo físico do BM25. Ela não guarda páginas, texto ou imagens de PDFs; ela guarda o ponteiro do acervo vivo.
- Para que é usada: orientar `overwrite`, `update` e `skip`, resolver o alvo físico usado pelo vector store e pelo BM25 e permitir que o sistema leia o acervo ativo sem depender do último run executado.
- Limite importante: `ingestion_runs` pode ter histórico de execuções antigas, canceladas ou falhas. O acervo vivo deve ser lido por `ingestion_datasets.active_generation_id`, não pelo run mais recente.
- Chave primária: `dataset_id`.
- Colunas:
- `dataset_id`: identificador UUID do dataset lógico.
- `tenant_code`: código lógico do tenant.
- `vectorstore_id`: identificador lógico do acervo.
- `if_exists_policy`: política canônica do dataset vivo, com os valores `update`, `skip` e `overwrite`.
- `status`: estado atual do dataset, como `registered`, `preparing`, `active` ou `failed`.
- `active_generation_id`: geração hoje exposta para leitura no acervo vivo.
- `physical_vector_target`: alvo físico do banco vetorial preparado ou ativo, quando existir. Este campo é a fonte de verdade do provider em runtime e pode apontar tanto para target novo tenantizado quanto para target legado preservado por compatibilidade.
- Explicação 101: este campo responde à pergunta “qual recurso físico real o provider deve abrir agora?”. No Azure Search, é daqui que nasce o `index_name` usado em runtime. No Qdrant, é daqui que nasce a `collection_name`.
- `physical_bm25_target`: alvo físico do BM25 preparado ou ativo, quando existir.
- `metadata`: metadados do lifecycle em `jsonb`.
- `created_at`: criação do dataset lógico.
- `updated_at`: última atualização do dataset lógico.
- Índices e restrições:
- PK em `dataset_id`.
- Unique `ingestion_datasets_tenant_vector_unique` em `tenant_code, vectorstore_id`.
- Índice `idx_ingestion_datasets_tenant_vector` em `tenant_code, vectorstore_id`.
- Índice parcial `idx_ingestion_datasets_active_generation` em `active_generation_id` quando não nulo.
- FK `ingestion_datasets_active_generation_fk` para `ingestion_dataset_generations.generation_id` com `ON DELETE SET NULL`.
- Check `ingestion_datasets_if_exists_policy_check` limitando `if_exists_policy` a `update`, `skip` ou `overwrite`.
- Check `ingestion_datasets_status_check` limitando `status` ao conjunto de estados do dataset lógico.

### ingestion_dataset_generations

- Finalidade prática: registrar cada geração preparada, ativa, abortada, falha ou substituída do dataset lógico.
- Papel no ciclo de vida: permite staging, ativação observável, aborto explícito e preservação de histórico sem apagar imediatamente o acervo anterior.
- Uso na ingestão de PDF: cada rodada que precisa preparar uma nova materialização do acervo cria ou usa uma geração. No `overwrite`, por exemplo, a geração nova pode ser preparada com alvos físicos próprios antes de virar a geração ativa.
- O que ela grava no fluxo de PDF: número da geração, status da geração, alvos físicos de vector store e BM25, `created_by_run_id`, `correlation_id` e metadata de lifecycle. Esses dados explicam qual geração nasceu de qual execução e qual materialização física pertence a ela.
- Para que é usada: dar rastreabilidade entre dataset vivo, BM25 persistido e banco vetorial. Também permite abortar ou rebaixar uma geração sem confundir isso com o histórico operacional do run.
- Relação com BM25: `bm25_indexes.generation_id` aponta para esta tabela. Por isso, quando uma geração de ingestão é destruída de forma controlada, o índice BM25 associado também precisa ser tratado como parte do mesmo conjunto operacional.
- Chave primária: `generation_id`.
- Colunas:
- `generation_id`: identificador UUID da geração.
- `dataset_id`: dataset lógico ao qual a geração pertence.
- `tenant_code`: código lógico do tenant.
- `vectorstore_id`: identificador lógico do acervo.
- `generation_number`: número monotônico da geração dentro do dataset.
- `if_exists_policy`: política usada para construir essa geração.
- `status`: estado da geração, como `preparing`, `active`, `failed`, `aborted` ou `superseded`.
- `physical_vector_target`: alvo físico do banco vetorial usado por essa geração. Em datasets novos, ele deve seguir a convenção física tenantizada do provider; em datasets antigos, ele pode continuar refletindo o target legado já persistido até que uma nova geração válida o substitua.
- Explicação 101: a geração guarda o nome físico que foi realmente preparado para aquele momento do acervo. Isso evita recalcular nome por adivinhação e protege a compatibilidade com targets antigos já existentes.
- `physical_bm25_target`: alvo físico do BM25 usado por essa geração.
- `created_by_run_id`: run que originou a geração, quando existir.
- `correlation_id`: correlação canônica do fluxo que criou a geração.
- `metadata`: metadados da geração em `jsonb`.
- `created_at`: criação da geração.
- `committed_at`: momento em que a geração foi promovida a ativa.
- `aborted_at`: momento em que a geração foi abortada.
- Índices e restrições:
- PK em `generation_id`.
- FK `ingestion_dataset_generations_dataset_fk` para `ingestion_datasets.dataset_id` com `ON DELETE CASCADE`.
- FK `ingestion_dataset_generations_run_fk` para `ingestion_runs.run_id` com `ON DELETE SET NULL`.
- Unique `ingestion_dataset_generations_dataset_number_unique` em `dataset_id, generation_number`.
- Índice `idx_ingestion_dataset_generations_dataset` em `dataset_id, generation_number DESC`.
- Índice `idx_ingestion_dataset_generations_tenant_vector_status` em `tenant_code, vectorstore_id, status, generation_number DESC`.
- Unique parcial `ux_ingestion_dataset_generations_active` em `dataset_id` quando `status = 'active'`, impedindo duas gerações ativas para o mesmo dataset.
- Check `ingestion_dataset_generations_if_exists_policy_check` limitando `if_exists_policy` a `update`, `skip` ou `overwrite`.
- Check `ingestion_dataset_generations_status_check` limitando `status` aos estados válidos da geração.
- Check `ingestion_dataset_generations_generation_number_check` exigindo `generation_number > 0`.

### ingestion_document_manifest

- Finalidade prática: registrar cada documento conhecido pela ingestão.
- Papel no dataset vivo: esta é a face PostgreSQL do acervo ativo e deve permanecer semanticamente alinhada com BM25 e banco vetorial para o mesmo `tenant_code + vectorstore_id`.
- Uso na ingestão de PDF: é a ficha principal de cada PDF ou documento equivalente. O runtime consulta esta tabela antes de processar para decidir se deve pular, atualizar ou gravar uma nova versão, e atualiza a linha quando o documento é persistido com sucesso.
- O que ela grava no fluxo de PDF: identidade lógica do documento (`document_identity_key`), fingerprint binário do arquivo (`pdf_binary_sha256`), hash do conteúdo/versionamento (`document_hash`), caminho e nome mais recentes observados, tamanho, tipo, MIME, total de páginas, estado de ingestão, última execução, ACL e metadata operacional. Em linguagem simples: ela diz “qual PDF lógico é este e qual é a versão oficial dele agora?”.
- Para que é usada: localizar documentos do acervo vivo, aplicar idempotência, avaliar `if_exists=skip`, fazer filtros de autorização e apontar para a versão ativa em `active_document_version_id`.
- Relação com páginas, chunks e imagens: as tabelas `ingestion_document_pages`, `ingestion_document_chunks` e `ingestion_document_images` dependem do `manifest_id`. O manifesto é o pai lógico; os filhos carregam o conteúdo detalhado.
- Relação com `overwrite`: quando a política é `overwrite`, esta tabela representa exatamente o acervo PostgreSQL que deve ser removido da geração substituída. Em termos simples: destruir o manifesto antigo e seus filhos é a forma de retirar o documento do dataset vivo sem apagar a trilha histórica do run.
- Chave primária: `manifest_id`.
- Colunas:
- `manifest_id`: identificador UUID do documento.
- `tenant_code`: código lógico do tenant.
- `vectorstore_id`: coleção de destino.
- `source_system`: origem do documento.
- `document_identity_key`: identidade lógica canônica do documento. Para PDF, o formato canônico é `pdf:sha256:<pdf_binary_sha256>`.
- `pdf_binary_sha256`: SHA-256 dos bytes brutos do PDF, calculado antes de OCR, parser e chunking. Quando ausente para PDF, o runtime deve falhar fechado.
- `canonical_source_key`: identidade canônica da fonte lógica observada por último. Ela continua útil como provenance operacional, mas não é mais a chave única do manifesto para PDF.
- `document_hash`: hash do conteúdo/versionamento efetivo persistido para o documento.
- `document_path`: caminho original do documento.
- `document_name`: nome amigável do arquivo.
- `file_size_bytes`: tamanho do arquivo em bytes.
- `file_last_modified`: última alteração do arquivo na origem.
- `external_document_id`: identificador externo estável do documento na origem, como o `page_id` do Confluence.
- `ingestion_status`: estado operacional da ingestão, com default `pending`.
- `active_document_version_id`: versão de conteúdo atualmente oficial para a fonte lógica deste manifesto.
- `last_run_id`: última execução relacionada, sem FK explícita.
- `last_ingested_at`: último instante de ingestão.
- `locked`: resumo operacional indicando restrição de leitura e/ou edição na origem.
- `has_read_restriction`: indica se há restrição de leitura na origem.
- `has_update_restriction`: indica se há restrição de edição na origem.
- `is_restricted`: flag canônica de documento privado para ACL do sistema.
- `allows_anonymous`: indica se a origem permite acesso anônimo.
- `permitted_groups`: grupos autorizados normalizados em `text[]` para filtros SQL e ACL.
- `authorization_checked_at`: momento da última leitura de autorização na origem.
- `authorization_source`: fonte usada para resolver a autorização.
- `authorization_snapshot`: snapshot bruto ou resumido da ACL em `jsonb`.
- `metadata`: metadados do documento em `jsonb`.
- `created_at`: criação do registro.
- `updated_at`: última atualização do registro.
- `document_type`: tipo do documento usando `public.ingestion_document_type`.
- `mime_type`: tipo MIME do documento.
- `total_pages`: total de páginas, quando aplicável.
- `status`: coluna gerada automaticamente a partir de `ingestion_status`.
- Índices e restrições:
- PK em `manifest_id`.
- Índice `idx_ingestion_manifest_last_run` em `last_run_id`.
- Índice `idx_manifest_hash` em `document_hash`.
- Índice `idx_manifest_path` em `document_path`.
- Índice `idx_manifest_source_key` em `canonical_source_key`.
- Índice `idx_manifest_document_identity` em `document_identity_key`.
- Índice `idx_manifest_type_status` em `document_type, status`.
- Índice `idx_ing_manifest_source_external_id` em `tenant_code, vectorstore_id, source_system, external_document_id`.
- Índice `idx_ing_manifest_acl_flags` em `tenant_code, vectorstore_id, source_system, is_restricted, allows_anonymous`.
- Índice `idx_ing_manifest_acl_checked_at` em `authorization_checked_at DESC NULLS LAST`.
- Índice GIN `idx_ing_manifest_permitted_groups` em `permitted_groups`.
- Índice `idx_ingestion_document_manifest_active_version` em `active_document_version_id`.
- Índice único `ux_ingestion_manifest_tenant_vector_identity` em `tenant_code, vectorstore_id, document_identity_key`.
- FK `active_document_version_id` para `ingestion_document_versions.document_version_id` com `ON DELETE SET NULL`.

Regra de identidade do manifesto:

- `document_identity_key` é a identidade lógica do documento. Para PDF, ela responde a pergunta “qual é o chassi binário deste arquivo?”.
- `canonical_source_key` é a identidade da ocorrência de fonte. Ela responde a pergunta “em qual origem esse PDF foi encontrado?”.
- O lookup e o upsert do manifesto de PDF devem sempre usar `document_identity_key`.
- `document_hash` continua essencial para detectar reprocessamento idempotente e mudança real de conteúdo, mas não pode mais decidir sozinho a identidade lógica do PDF.
- `active_document_version_id` aponta qual edição de conteúdo está oficialmente publicada para aquele documento lógico naquele momento.
- Consequência prática: duas fontes diferentes com o mesmo PDF colidem no mesmo manifesto lógico; a provenance de cada fonte passa a viver em `ingestion_document_source_occurrences`.

### ingestion_document_versions

- Finalidade prática: guardar a trilha de versões de conteúdo de cada documento lógico sem confundir fonte com edição.
- Papel no dataset vivo: esta tabela materializa a edição oficialmente ativa do documento no Postgres e permite substituir conteúdo sem trocar a identidade lógica do PDF.
- Uso na ingestão de PDF: sempre que o conteúdo/versionamento efetivo do mesmo PDF muda, o runtime deriva um `document_version_id` a partir de `document_identity_key + version_fingerprint`, ativa essa versão e rebaixa a versão anterior para `superseded` quando necessário.
- O que ela grava no fluxo de PDF: versão de conteúdo, manifesto dono, tenant, vectorstore, identidade lógica do documento, identidade da fonte observada na ativação, hash do conteúdo, status da versão e metadata da ativação. Ela não representa um novo documento lógico; representa uma nova edição do mesmo documento.
- Para que é usada: garantir que páginas, chunks e imagens apontem para a mesma edição de conteúdo. Isso evita misturar chunks de uma versão antiga com páginas de uma versão nova.
- Regra prática: somente uma versão pode ficar `active` para a mesma combinação `tenant_code + vectorstore_id + document_identity_key`.
- Chave primária: `document_version_id`.
- Colunas:
- `document_version_id`: identificador determinístico da versão, derivado de `document_identity_key + version_fingerprint`.
- `manifest_id`: manifesto dono da fonte lógica.
- `tenant_code`: código lógico do tenant.
- `vectorstore_id`: coleção lógica do acervo.
- `document_identity_key`: identidade lógica estável do documento.
- `canonical_source_key`: identidade da fonte observada quando a versão foi ativada.
- `document_hash`: hash do conteúdo representado por esta versão.
- `status`: estado da versão, com valores `active` ou `superseded`.
- `metadata`: snapshot do metadata do manifesto no momento da ativação, em `jsonb`.
- `activated_at`: momento em que esta versão se tornou oficial.
- `superseded_at`: momento em que esta versão deixou de ser a oficial, quando aplicável.
- `created_at`: criação do registro.
- `updated_at`: última atualização do registro.
- Índices e restrições:
- PK em `document_version_id`.
- FK `manifest_id` para `ingestion_document_manifest.manifest_id` com `ON DELETE CASCADE`.
- Check `ingestion_document_versions_status_check` restringindo `status` a `active` e `superseded`.
- Índice `idx_ingestion_document_versions_manifest` em `manifest_id, status, activated_at DESC`.
- Índice `idx_ingestion_document_versions_identity` em `tenant_code, vectorstore_id, document_identity_key, activated_at DESC`.
- Unique parcial `ux_ingestion_document_versions_active_identity` em `tenant_code, vectorstore_id, document_identity_key` quando `status = 'active'`.

### ingestion_document_source_occurrences

- Finalidade prática: guardar a provenance de cada fonte que aponta para um manifesto lógico já conhecido.
- Papel no dataset vivo: esta tabela responde “onde esse PDF apareceu?” sem recriar manifesto duplicado para o mesmo arquivo.
- Uso na ingestão de PDF: sempre que um PDF novo é persistido ou um PDF já conhecido reaparece em outra origem, o runtime faz upsert da ocorrência de fonte nesta tabela.
- O que ela grava no fluxo de PDF: `manifest_id`, tenant, vectorstore, `source_system`, `canonical_source_key`, URI da fonte, identificador externo, ACL observada e metadata da ocorrência.
- Regra prática: pode haver várias ocorrências para o mesmo `manifest_id`, mas somente uma por `tenant_code + vectorstore_id + source_system + canonical_source_key`.
- Chave primária: composta por `tenant_code, vectorstore_id, source_system, canonical_source_key`.
- Colunas principais:
- `manifest_id`: manifesto lógico dono do PDF.
- `source_system`: origem observada.
- `canonical_source_key`: identidade estável da fonte.
- `source_uri`: localização concreta observada na fonte.
- `external_document_id`: id estável exposto pela origem, quando existir.
- `is_restricted`, `allows_anonymous`, `permitted_groups`, `authorization_checked_at`, `authorization_source`, `authorization_snapshot`: ACL da ocorrência.
- `metadata`: metadata operacional da ocorrência.
- `created_at` e `updated_at`: trilha temporal da ocorrência.
- Índices e restrições:
- PK composta em `tenant_code, vectorstore_id, source_system, canonical_source_key`.
- FK `manifest_id` para `ingestion_document_manifest.manifest_id` com `ON DELETE CASCADE`.
- Índice `idx_ingestion_document_source_occurrences_manifest` em `manifest_id`.

Como ler na prática:

- `ingestion_document_manifest` diz qual é o documento lógico do acervo.
- `ingestion_document_versions` diz qual edição de conteúdo esse documento lógico já teve.
- `ingestion_document_source_occurrences` diz em quais fontes esse mesmo PDF apareceu.
- `active_document_version_id` no manifesto aponta para a única edição que o runtime pode tratar como oficial.
- Se uma nova ingestão do mesmo documento falhar antes do commit final, a versão anterior continua sendo a ativa.
- Para legado antigo sem `active_document_version_id`, a reconciliação correta deve ser idempotente e derivada de `document_identity_key + version_fingerprint`. Em termos práticos: não gerar UUID aleatório, não adivinhar versão ativa e não reinventar fingerprint de PDF sem bytes comprovados.

### ingestion_runs

- Finalidade prática: registrar cada execução de ingestão.
- Papel no ciclo de vida: esta tabela é histórico operacional e auditoria de execução. Ela não deve ser confundida com o dataset vivo do acervo e não entra automaticamente em limpeza destrutiva de `overwrite`.
- Uso na ingestão de PDF: representa a execução pai da ingestão. O runtime cria ou atualiza esta linha quando o job é enfileirado, quando o worker inicia e quando a execução termina.
- Regra operacional crítica: para o mesmo par `tenant_code + vectorstore_id`, só pode existir um run pai ativo por vez. Em termos simples, filhos do mesmo pai continuam paralelos, mas um segundo lote pai novo não pode nascer enquanto o primeiro ainda estiver ativo.
- Semântica dessa exclusividade: a trava considera ativos os estados `pending`, `queued`, `running`, `processing` e `cancelling`. Os estados `completed`, `failed` e `cancelled` continuam sendo apenas histórico e não bloqueiam uma nova ingestão pai.
- Regra importante de admissão: essa exclusividade ignora `vector_store.if_exists`. A política `skip`, `update` ou `overwrite` decide como o dataset vivo será tratado, mas não autoriza dois pais simultâneos brigando pelo mesmo acervo lógico.
- O que ela grava no fluxo de PDF: tenant, vectorstore, origem, YAML usado, `task_id`, horários de fila/início/fim, heartbeat do worker, progresso, status, contadores agregados, mensagem operacional, erro resumido, metadata e `correlation_id`.
- Para que é usada: tela administrativa de runs, cancelamento, reconciliação, observabilidade, análise de falha e ligação entre API, worker e logs. Ela conta “o que aconteceu com a execução”, não “qual documento está vivo no acervo”.
- Limite importante observado no runtime atual: tokens e custos existem no schema, mas o fechamento atual do run ainda não consolida esses valores de uma fonte canônica. Até essa correção existir, não use essas colunas como prova de custo real.
- Chave primária: `run_id`.
- Colunas:
- `run_id`: identificador UUID da execução.
- `tenant_code`: código lógico do tenant.
- `vectorstore_id`: coleção alvo.
- `source_system`: origem processada.
- `yaml_path`: YAML usado na execução.
- `started_at`: início da execução, com default `now()`.
- `finished_at`: fim da execução.
- `total_documents`: total de documentos considerados.
- `processed_documents`: total de documentos processados.
- `skipped_documents`: total de documentos ignorados.
- `failed_documents`: total de documentos com falha.
- `total_chunks`: total de chunks gerados.
- `total_tokens`: total geral de tokens.
- `embedding_tokens`: tokens consumidos por embedding.
- `llm_tokens`: tokens consumidos por LLM.
- `embedding_cost`: custo de embedding.
- `llm_cost`: custo de LLM.
- `duration_seconds`: duração total da execução.
- `status`: status da execução.
- `correlation_id`: correlação com logs.
- `error_summary`: resumo do erro, quando houver.
- `metadata`: metadados da execução em `jsonb`.
- `created_at`: criação do registro.
- `task_id`: identificador do job assíncrono associado ao run, quando a execução nasce pela fila.
- `queued_at`: instante em que o job foi aceito e enfileirado.
- `updated_at`: última atualização operacional do run.
- `last_heartbeat_at`: último sinal do worker responsável pela execução.
- `heartbeat_interval_seconds`: intervalo esperado entre heartbeats.
- `stale_after_seconds`: janela após a qual o run pode ser considerado sem heartbeat recente.
- `worker_instance_id`: instância lógica do worker que assumiu o run.
- `worker_hostname`: hostname do worker.
- `worker_pid`: PID do worker.
- `attempt_number`: tentativa operacional do job, quando controlada pelo executor assíncrono.
- `progress_current`: progresso atual usado pela leitura operacional.
- `progress_total`: total usado para calcular progresso.
- `progress_pct`: percentual de progresso persistido.
- `status_message`: mensagem operacional curta para UI e diagnóstico.
- `cancel_requested`: indica pedido de cancelamento.
- `cancel_requested_at`: momento do pedido de cancelamento.
- Índices e restrições:
- PK em `run_id`.
- Índice `idx_ingestion_runs_tenant_vector_started` em `tenant_code, vectorstore_id, started_at`.
- Índice único parcial `ux_ingestion_runs_active_vectorstore` em `tenant_code, vectorstore_id` quando `vectorstore_id` não é nulo e `status` está em `pending`, `queued`, `running`, `processing` ou `cancelling`. Esse índice é a trava durável que impede dois runs pais ativos ao mesmo tempo para o mesmo acervo lógico.
- Leitura operacional segura: os scripts `scripts/sql/20260511_audit_ingestion_schema_columns.sql`, `scripts/sql/20260511_audit_ingestion_column_population.sql` e `scripts/sql/20260511_audit_ingestion_integrity_checks.sql` são guardiões read-only deste contrato. Eles existem para auditar o schema e os invariantes sem escrever no banco.

### ingestion_run_documents

- Finalidade prática: registrar o estado canônico de cada documento dentro de uma execução de ingestão.
- Papel no ciclo de vida: esta tabela pertence ao histórico operacional do run. Ela ajuda a auditar como o acervo foi construído, mas não substitui o estado vivo do dataset representado por manifesto, BM25 e banco vetorial.
- Uso na ingestão de PDF: cada PDF dentro de um run ganha uma linha lógica nesta tabela. O runtime registra documentos pulados por `skip`, documentos já sincronizados, documentos processados com sucesso e documentos com erro ou cancelamento.
- Relação com o pai único por vectorstore: esta tabela não participa da trava de admissão do pai. Ela existe para controlar e auditar os filhos do mesmo run pai, inclusive quando eles executam em paralelo sobre o mesmo `tenant_code + vectorstore_id`.
- O que ela grava no fluxo de PDF: caminho, nome, tamanho, hash, status dentro do run, quantidade de chunks, tokens por documento quando disponíveis, tempo de processamento, erro, metadata, tipo, MIME, total de páginas e vínculo com o manifesto.
- Para que é usada: detalhar o run pai, alimentar telas que mostram documentos da execução, orientar cancelamento/retry por documento e permitir auditoria granular sem consultar diretamente páginas e chunks.
- Limite importante observado no runtime atual: esta tabela deve contar a mesma história física do manifesto, principalmente em `total_pages`, mas a investigação recente encontrou divergências históricas. O contrato correto é que a fonte estruturada do documento alimente manifesto e detalhe do run de forma coerente.
- Relação com `overwrite`: esta tabela não entra na limpeza destrutiva do dataset vivo. Se o manifesto antigo sair por `overwrite`, o `manifest_id` histórico deste registro pode ser limpo para `NULL`, preservando a trilha do run sem manter um vínculo inválido para o acervo ativo removido.
- Chave primária: `run_document_id`.
- Colunas:
- `run_document_id`: identificador UUID estável do documento dentro do run. O valor é determinístico a partir do `run_id` e do `document_path` normalizado, para que reprocessamentos do mesmo documento atualizem a mesma linha.
- `run_id`: execução pai.
- `manifest_id`: manifesto correspondente, quando houver.
- `tenant_code`: código lógico do tenant.
- `vectorstore_id`: coleção relacionada.
- `document_hash`: hash do documento.
- `document_path`: caminho do documento.
- `document_name`: nome do documento.
- `file_size_bytes`: tamanho do arquivo.
- `chunk_count`: quantidade de chunks gerados.
- `embedding_tokens`: tokens de embedding.
- `llm_tokens`: tokens de LLM.
- `processing_seconds`: tempo de processamento do documento.
- `status`: estado do documento na execução.
- `error_message`: mensagem de erro.
- `metadata`: metadados adicionais em `jsonb`.
- `created_at`: criação do registro.
- `updated_at`: última atualização do registro. Na prática, esse campo mostra quando o estado operacional do documento foi atualizado pela última vez dentro do run.
- `document_type`: tipo do documento.
- `mime_type`: tipo MIME.
- `total_pages`: total de páginas.
- Semântica operacional:
- cada combinação de `run_id` com `document_path` normalizado representa uma única entidade lógica.
- mudanças de estado como `queued`, `running`, `retrying`, `success`, `skipped`, `error` e `cancelled` devem atualizar a mesma linha, em vez de criar histórico duplicado para o mesmo documento.
- status terminais do mesmo run, como `success`, `skipped`, `error` e `cancelled`, não podem voltar para estados ativos por redelivery atrasado ou requeue acidental. Em termos simples, documento terminado não reabre dentro do mesmo `run_id`.
- a mesma atualização de estado deve também refrescar `updated_at`, para que leitura operacional, ordenação e diagnóstico usem o instante real da última transição observada.
- o campo `metadata` deve carregar o contexto operacional mais recente do documento, incluindo quando disponível `last_transition_at`, `attempt_count`, `worker_identity`, `parent_correlation_id`, `document_correlation_id` e `next_retry_at`.
- Índices e restrições:
- PK em `run_document_id`.
- FK `manifest_id` para `ingestion_document_manifest.manifest_id` sem cascade.
- FK `run_id` para `ingestion_runs.run_id` com `ON DELETE CASCADE`.
- Índice `idx_ingestion_run_documents_manifest` em `manifest_id`.
- Índice `idx_ingestion_run_documents_run` em `run_id`.
- Índice `idx_ingestion_run_documents_tenant_vector` em `tenant_code, vectorstore_id`.
- Índice `idx_run_documents_type_status` em `document_type, status`.

### ingestion_run_slot_leases

- Finalidade prática: materializar o semáforo canônico do fan-out por documento, limitando quantos documentos do mesmo run podem executar em paralelo sem usar lock pessimista na linha de `ingestion_runs`.
- Uso na ingestão de PDF: quando a ingestão executa documentos em paralelo, esta tabela representa os slots disponíveis para os filhos. O documento só entra em execução ativa depois de conquistar ou renovar um slot.
- O que ela grava no fluxo de PDF: slot do run, documento dono do slot, URI da fonte, worker, correlação do documento, horário de aquisição, heartbeat, expiração e metadata do lease.
- Para que é usada: controlar concorrência por run sem bloquear a linha pai com `SELECT ... FOR UPDATE`. Em termos simples: ela evita que documentos demais rodem ao mesmo tempo no mesmo lote.
- Limite importante: slot não é documento e não é resultado final. É uma permissão temporária de execução. O resultado continua sendo registrado em `ingestion_run_documents` e o agregado em `ingestion_runs`.
- Chave primária: `tenant_code, run_id, slot_index`.
- Colunas:
- `tenant_code`: código lógico do tenant.
- `run_id`: execução pai dona do conjunto de slots.
- `vectorstore_id`: coleção associada ao lote, útil para purge e diagnóstico operacional.
- `slot_index`: número ordinal do slot dentro do run. Na prática, se o paralelismo efetivo for 5, existirão 5 slots canônicos numerados de 1 a 5.
- `lease_run_document_id`: documento que está ocupando o slot naquele instante, quando houver.
- `lease_source_uri`: origem do documento atualmente dono do slot.
- `lease_worker_identity`: identidade do worker que conquistou ou renovou o slot.
- `lease_document_correlation_id`: correlação operacional do documento que ocupa o slot.
- `lease_acquired_at`: momento em que o slot foi conquistado.
- `lease_heartbeat_at`: última renovação operacional observada do slot.
- `lease_expires_at`: prazo máximo do lease atual.
- `metadata`: metadados do lease em `jsonb`, com default de objeto vazio.
- `created_at`: criação da linha do slot, com default `now()`.
- `updated_at`: última atualização do slot, com default `now()`.
- Semântica operacional:
- o run pai materializa um conjunto fixo de slots canônicos conforme o `effective_parallelism` resolvido para o fan-out.
- o documento continua sendo a entidade canônica de execução; o slot é apenas uma permissão temporária de concorrência.
- um documento só pode entrar em `running` ou `retrying` depois de conquistar um slot livre ou reutilizar o seu próprio slot vigente.
- renovação e liberação do slot acontecem junto da transição canônica do documento, evitando coordenação por `SELECT ... FOR UPDATE` na linha pai.
- o fechamento agregado do run pai deixa de depender de escrita a cada filho; o lote pode ser sincronizado no encerramento real, quando não restam documentos ativos.
- Índices e restrições:
- PK em `tenant_code, run_id, slot_index`.
- FK `run_id` para `ingestion_runs.run_id` com `ON DELETE CASCADE`.
- FK `lease_run_document_id` para `ingestion_run_documents.run_document_id` com `ON DELETE SET NULL`, para limpar automaticamente o vínculo quando o documento associado desaparecer.
- Check `slot_index > 0`, garantindo que não exista slot zero ou negativo.
- Índice `idx_ingestion_run_slot_leases_run_active` em `tenant_code, run_id, lease_expires_at, slot_index`.
- Índice `idx_ingestion_run_slot_leases_tenant_vector` em `tenant_code, vectorstore_id`.
- Índice `idx_ingestion_run_slot_leases_document` em `tenant_code, run_id, lease_run_document_id`.
- Índice único parcial `ux_ingestion_run_slot_leases_document` para impedir que o mesmo documento ocupe mais de um slot ao mesmo tempo quando `lease_run_document_id` estiver preenchido.

### ingestion_document_chunks

- Finalidade prática: guardar os chunks textuais produzidos para cada documento.
- Uso na ingestão de PDF: depois que o PDF é extraído e quebrado em partes menores, cada parte textual vira uma linha nesta tabela. O runtime grava o texto, posição, páginas cobertas, metadados e identificador do vetor correspondente no provider vetorial.
- O que ela grava no fluxo de PDF: conteúdo textual pesquisável, índice do chunk, intervalo de páginas e caracteres, modelo de embedding, id do vetor e metadata técnica. A coluna `fts_content` é gerada pelo PostgreSQL a partir de `text_content`.
- Para que é usada: busca textual PostgreSQL/FTS, exportação de textos, recuperação RAG e conferência entre texto persistido, BM25 e banco vetorial.
- Limite importante: esta tabela não escolhe qual versão é oficial sozinha. A leitura correta deve respeitar `document_version_id` e o `active_document_version_id` do manifesto.
- Chave primária: `chunk_id`.
- Colunas:
- `chunk_id`: identificador UUID do chunk.
- `manifest_id`: documento pai.
- `document_version_id`: versão de conteúdo oficialmente associada aos chunks atuais do documento.
- `chunk_index`: posição do chunk no documento.
- `text_content`: texto do chunk.
- `page_start`: página inicial coberta pelo chunk.
- `page_end`: página final coberta pelo chunk.
- `char_start`: posição inicial de caractere.
- `char_end`: posição final de caractere.
- `embedding_vector_id`: identificador do vetor.
- `embedding_model`: modelo de embedding usado.
- `reindex_required`: indica necessidade de reindexação.
- `metadata`: metadados do chunk em `jsonb`.
- `created_at`: criação do registro.
- `fts_content`: coluna gerada com índice textual em português.
- Índices e restrições:
- PK em `chunk_id`.
- Unique `ingestion_document_chunks_manifest_id_chunk_index_key` em `manifest_id, chunk_index`.
- FK `manifest_id` para `ingestion_document_manifest.manifest_id` com `ON DELETE CASCADE`.
- FK `document_version_id` para `ingestion_document_versions.document_version_id` com `ON DELETE CASCADE`.
- Índice GIN `idx_chunks_fts_content` em `fts_content`.
- Índice `idx_chunks_manifest` em `manifest_id`.
- Índice `idx_ingestion_document_chunks_document_version` em `document_version_id`.
- Índice `idx_chunks_vector` em `embedding_vector_id`.

### ingestion_document_pages

- Finalidade prática: guardar a visão por página do documento.
- Uso na ingestão de PDF: cada página extraída do PDF pode ser persistida com texto bruto, texto limpo, HTML, imagem, thumbnail, confiança de OCR, indicação de tabelas e metadata. Isso preserva a leitura por página, mesmo quando os chunks agrupam conteúdo de mais de uma página.
- O que ela grava no fluxo de PDF: número da página, textos extraídos, recursos visuais associados, sinais de OCR/tabela e versão de conteúdo do documento.
- Para que é usada: auditoria visual/textual de PDF, depuração de OCR, reconstrução de evidência por página e associação de imagens à página correta.
- Limite importante: a página também deve apontar para `document_version_id`. Isso evita que uma página antiga seja lida como se pertencesse à versão ativa nova.
- Chave primária: `page_id`.
- Colunas:
- `page_id`: identificador UUID da página.
- `manifest_id`: documento pai.
- `document_version_id`: versão de conteúdo oficialmente associada à página.
- `page_number`: número da página.
- `text_raw`: texto bruto.
- `text_clean`: texto limpo.
- `html_content`: versão HTML da página.
- `image_uri`: URI da imagem da página.
- `thumbnail_uri`: URI do thumbnail da página.
- `ocr_confidence`: confiança do OCR.
- `has_tables`: indica se há tabela detectada.
- `metadata`: metadados da página em `jsonb`.
- `created_at`: criação do registro.
- Índices e restrições:
- PK em `page_id`.
- Unique `ingestion_document_pages_manifest_id_page_number_key` em `manifest_id, page_number`.
- FK `manifest_id` para `ingestion_document_manifest.manifest_id` com `ON DELETE CASCADE`.
- FK `document_version_id` para `ingestion_document_versions.document_version_id` com `ON DELETE CASCADE`.
- Índice `idx_pages_manifest` em `manifest_id`.
- Índice `idx_ingestion_document_pages_document_version` em `document_version_id`.

### ingestion_document_images

- Finalidade prática: guardar imagens extraídas dos documentos e seus metadados multimodais.
- Uso na ingestão de PDF: imagens encontradas nas páginas do PDF, ou imagens geradas como representação visual de página, são persistidas aqui com localização, OCR, descrição de visão e eventual vetor multimodal.
- O que ela grava no fluxo de PDF: origem e storage da imagem, MIME, dimensões, página, índice da imagem, texto OCR, confiança, descrição visual, modelo usado, vetor e metadata.
- Para que é usada: recuperação multimodal, auditoria de OCR/visão, ligação entre página e imagem e diagnóstico de assets persistidos no storage.
- Limite importante: `page_id` pode ser nulo quando a imagem não puder ser ligada a uma linha de página, mas `manifest_id` e `document_version_id` preservam o vínculo com o documento e a edição de conteúdo.
- Chave primária: `image_id`.
- Colunas:
- `image_id`: identificador UUID da imagem.
- `manifest_id`: documento pai.
- `document_version_id`: versão de conteúdo oficialmente associada à imagem atual.
- `page_id`: página relacionada, quando existir.
- `page_number`: número da página de origem.
- `image_index`: posição da imagem dentro da página.
- `source_uri`: URI original da imagem.
- `storage_uri`: URI persistida no storage.
- `mime_type`: tipo MIME da imagem.
- `width_px`: largura em pixels.
- `height_px`: altura em pixels.
- `ocr_text`: texto OCR extraído.
- `ocr_confidence`: confiança do OCR.
- `vision_description`: descrição gerada para a imagem.
- `vision_model`: modelo de visão usado.
- `embedding_vector_id`: identificador do vetor da imagem.
- `embedding_model`: modelo de embedding usado.
- `metadata`: metadados adicionais em `jsonb`.
- `created_at`: criação do registro.
- `updated_at`: última atualização do registro.
- Índices e restrições:
- PK em `image_id`.
- Unique `ux_img_per_page` em `manifest_id, page_number, image_index`.
- FK `manifest_id` para `ingestion_document_manifest.manifest_id` com `ON DELETE CASCADE`.
- FK `document_version_id` para `ingestion_document_versions.document_version_id` com `ON DELETE CASCADE`.
- FK `page_id` para `ingestion_document_pages.page_id` com `ON DELETE SET NULL`.
- Índice `idx_img_manifest_page` em `manifest_id, page_number`.
- Índice `idx_ingestion_document_images_document_version` em `document_version_id`.
- Índice `idx_img_vector` em `embedding_vector_id`.

### ingestion_domain_processors

- Finalidade prática: guardar configuração de processadores de domínio por tenant e por vectorstore.
- Uso na ingestão de PDF: o pipeline pode carregar configurações de domínio persistidas para decidir quais enriquecimentos, overrides ou ajustes automáticos aplicar ao conteúdo ingerido. Essa tabela não guarda o PDF; ela guarda como o conteúdo daquele domínio deve ser tratado.
- O que ela grava no fluxo de PDF: domínio, prioridade, habilitação, override de processador, configuração manual e configuração automática por escopo de tenant/vectorstore.
- Para que é usada: alimentar o `DomainConfigRepository` e os agregadores de auto-configuração, permitindo que regras de domínio evoluam sem depender apenas de YAML estático.
- Limite importante: configurações desta tabela afetam processamento e enriquecimento, mas não substituem o manifesto, as versões, páginas, chunks ou imagens do acervo.
- Chave primária: `id`.
- Colunas:
- `id`: identificador bigserial do registro.
- `tenant_id`: tenant do escopo, podendo ser nulo.
- `vectorstore_id`: coleção do escopo, podendo ser nula.
- `domain_key`: chave do domínio.
- `display_name`: nome amigável do domínio.
- `enabled`: indica se o processador está ativo.
- `priority`: prioridade de aplicação.
- `processor_override`: sobrescrita explícita do processador.
- `created_at`: criação do registro.
- `updated_at`: última atualização do registro.
- `deleted_at`: remoção lógica.
- `manual_config`: configuração manual em `jsonb`.
- `auto_config`: configuração automática em `jsonb`.
- Índices e restrições:
- PK em `id`.
- Unique `ingestion_domain_processors_domain_tenant_vector_uq` em `domain_key, tenant_id, vectorstore_id`.
- Check `ingestion_domain_processors_priority_check` garantindo `priority >= 0`.
- FK `tenant_id` para `tenants.tenant_id` com `ON DELETE SET NULL` e `ON UPDATE CASCADE`.

## Modelo de Ingestão Vetorial `vector_*` (modelo ATIVO — DDL real e de-para)

> **Aviso de status (leia antes de usar esta seção).** Esta seção descreve o **modelo ativo**
> de ingestão vetorial (família `vector_*`). O resumo do estado comprovado em runtime está em
> "Estado do modelo de ingestão vetorial `vector_*`", no topo deste manual. Aqui ficam o **DDL
> real** das tabelas e o **de-para** a partir do modelo antigo `ingestion_*`.
>
> - **O corte foi EXECUTADO** (migração `migracao-modelo-vetorial-ingestao`): a escrita da
>   ingestão de PDF (caminho assíncrono com fan-out) grava no modelo novo, e a leitura da tela e
>   o guard do RAG consomem o modelo novo via `VectorActiveArchiveRepository`. A leitura legada
>   de `ingestion_*` no caminho da tela foi cortada (sem fallback).
> - **Populado por dado, não por código:** o acervo novo está populado para o dataset de TESTE
>   (`dnit_teste`). PRODUÇÃO (`dnit_producao`) ainda **não** foi reconstruída (rebuild pendente),
>   então seu acervo novo está vazio até a re-ingestão. O guard do RAG é estrito onde o acervo
>   está populado e tolerante onde está vazio.
> - **Tabelas antigas `ingestion_*`:** continuam existindo e servindo ao control plane do Job
>   Core, streaming/status ao vivo, worker e manutenção; deixaram de ser a fonte de verdade do
>   histórico/acervo de NEGÓCIO no caminho da tela.
> - O DDL abaixo é o **DDL real, extraído do banco** via `information_schema` e `pg_constraint`
>   (reconstruído a partir dos metadados físicos). Não há índice extra além dos que dão suporte às
>   constraints (7 PRIMARY KEY + 5 UNIQUE).
> - Fonte original do de-para (documento interno do corte):
>   `docs/.interno/README-TECNICO-DE-PARA-MODELO-DADOS-INGESTAO-VETORIAL.md`. Se este resumo
>   divergir do código executável, vale primeiro o código.

### Por que existe um modelo novo

O modelo atual de ingestão mistura quatro ideias diferentes no mesmo eixo: acervo vivo
publicado, histórico de runs, documento lógico deduplicado e detalhe de conteúdo persistido.
Na prática isso confunde a operação humana — um run pode mostrar 100 documentos processados
enquanto o acervo vivo tem só 5, e isso só faz sentido para o runtime por causa do colapso
por identidade lógica escondida.

O modelo novo parte de uma regra simples e legível:

1. existe um cadastro **master** do acervo ligado ao banco vetorial;
2. existem **runs** de ingestão desse acervo (histórico, não acervo vivo);
3. existem **documentos observados** em cada run;
4. existe uma **projeção explícita** dos documentos atualmente publicados no acervo;
5. o detalhe físico (páginas, chunks, imagens) do documento ativo fica ligado ao documento
   publicado, e não a um manifesto híbrido difícil de explicar.

### Tabelas do modelo novo

Quatro tabelas centrais e três tabelas de detalhe de conteúdo ativo:

- `vector_dataset_master`: ficha master do acervo. Responde qual é o acervo, de qual tenant,
  qual o alvo vetorial, qual o alvo BM25 e qual foi o último run que alterou o acervo publicado.
- `vector_ingestion_runs`: histórico de execuções de ingestão do acervo (o "boletim" de cada lote).
- `vector_ingestion_run_documents`: lista de documentos observados dentro de cada run (tudo o
  que o lote viu, processou, pulou, falhou ou substituiu).
- `vector_active_documents`: projeção explícita do que está vivo agora no acervo,
  independentemente de qual run publicou cada documento.
- `vector_active_document_pages`, `vector_active_document_chunks`,
  `vector_active_document_images`: conteúdo físico persistido do documento ativo, agora ligado
  ao documento publicado. Em `vector_active_document_chunks`, o `vector_point_id` carrega, quando
  existir, o identificador físico do ponto no banco vetorial.

### DDL real (extraído do banco)

> DDL reconstruído a partir dos metadados físicos do PostgreSQL (`information_schema.columns`,
> `pg_constraint`, `pg_indexes`). Reflete o estado atual das tabelas no banco. Ordem das tabelas
> ajustada para respeitar dependências de FK ao recriar do zero (master e runs antes das tabelas
> que as referenciam).

```sql
CREATE TABLE vector_dataset_master (
    dataset_id UUID NOT NULL,
    tenant_code TEXT NOT NULL,
    vectorstore_id TEXT NOT NULL,
    vector_provider TEXT NOT NULL,
    vector_target TEXT NOT NULL,
    bm25_provider TEXT,
    bm25_target TEXT,
    status TEXT NOT NULL,
    last_published_run_id UUID,
    last_completed_run_id UUID,
    last_sync_at TIMESTAMPTZ,
    if_exists_policy TEXT NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT vector_dataset_master_pkey PRIMARY KEY (dataset_id),
    CONSTRAINT ux_vector_dataset_master_tenant_vector UNIQUE (tenant_code, vectorstore_id)
);

CREATE TABLE vector_ingestion_runs (
    run_id UUID NOT NULL,
    dataset_id UUID NOT NULL,
    tenant_code TEXT NOT NULL,
    vectorstore_id TEXT NOT NULL,
    source_system TEXT,
    trigger_mode TEXT,
    if_exists_policy TEXT NOT NULL,
    status TEXT NOT NULL,
    correlation_id TEXT,
    task_id TEXT,
    queued_at TIMESTAMPTZ,
    started_at TIMESTAMPTZ,
    finished_at TIMESTAMPTZ,
    total_documents INTEGER NOT NULL DEFAULT 0,
    processed_documents INTEGER NOT NULL DEFAULT 0,
    skipped_documents INTEGER NOT NULL DEFAULT 0,
    failed_documents INTEGER NOT NULL DEFAULT 0,
    total_chunks INTEGER NOT NULL DEFAULT 0,
    error_summary TEXT,
    status_message TEXT,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT vector_ingestion_runs_pkey PRIMARY KEY (run_id),
    CONSTRAINT vector_ingestion_runs_dataset_id_fkey FOREIGN KEY (dataset_id) REFERENCES vector_dataset_master(dataset_id) ON DELETE CASCADE
);

CREATE TABLE vector_ingestion_run_documents (
    run_document_id UUID NOT NULL,
    run_id UUID NOT NULL,
    dataset_id UUID NOT NULL,
    tenant_code TEXT NOT NULL,
    vectorstore_id TEXT NOT NULL,
    source_system TEXT,
    source_uri TEXT,
    external_document_id TEXT,
    document_path TEXT,
    document_name TEXT,
    document_type TEXT,
    mime_type TEXT,
    file_size_bytes BIGINT,
    file_last_modified TIMESTAMPTZ,
    content_fingerprint TEXT,
    document_hash TEXT,
    total_pages INTEGER,
    chunk_count INTEGER,
    processing_seconds DOUBLE PRECISION,
    status TEXT NOT NULL,
    publication_action TEXT NOT NULL,
    error_message TEXT,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT vector_ingestion_run_documents_pkey PRIMARY KEY (run_document_id),
    CONSTRAINT vector_ingestion_run_documents_run_id_fkey FOREIGN KEY (run_id) REFERENCES vector_ingestion_runs(run_id) ON DELETE CASCADE,
    CONSTRAINT vector_ingestion_run_documents_dataset_id_fkey FOREIGN KEY (dataset_id) REFERENCES vector_dataset_master(dataset_id) ON DELETE CASCADE
);

CREATE TABLE vector_active_documents (
    active_document_id UUID NOT NULL,
    dataset_id UUID NOT NULL,
    tenant_code TEXT NOT NULL,
    vectorstore_id TEXT NOT NULL,
    source_system TEXT,
    source_uri TEXT,
    external_document_id TEXT,
    canonical_document_key TEXT NOT NULL,
    current_run_id UUID NOT NULL,
    source_run_document_id UUID NOT NULL,
    document_path TEXT,
    document_name TEXT,
    document_type TEXT,
    mime_type TEXT,
    file_size_bytes BIGINT,
    file_last_modified TIMESTAMPTZ,
    content_fingerprint TEXT,
    document_hash TEXT,
    total_pages INTEGER,
    vector_document_key TEXT,
    bm25_document_key TEXT,
    status TEXT NOT NULL,
    published_at TIMESTAMPTZ NOT NULL,
    removed_at TIMESTAMPTZ,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT vector_active_documents_pkey PRIMARY KEY (active_document_id),
    CONSTRAINT ux_vector_active_documents_dataset_key UNIQUE (dataset_id, canonical_document_key),
    CONSTRAINT vector_active_documents_current_run_id_fkey FOREIGN KEY (current_run_id) REFERENCES vector_ingestion_runs(run_id) ON DELETE RESTRICT,
    CONSTRAINT vector_active_documents_dataset_id_fkey FOREIGN KEY (dataset_id) REFERENCES vector_dataset_master(dataset_id) ON DELETE CASCADE,
    CONSTRAINT vector_active_documents_source_run_document_id_fkey FOREIGN KEY (source_run_document_id) REFERENCES vector_ingestion_run_documents(run_document_id) ON DELETE RESTRICT
);

CREATE TABLE vector_active_document_pages (
    active_page_id UUID NOT NULL,
    active_document_id UUID NOT NULL,
    dataset_id UUID NOT NULL,
    page_number INTEGER NOT NULL,
    page_label TEXT,
    page_text TEXT,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT vector_active_document_pages_pkey PRIMARY KEY (active_page_id),
    CONSTRAINT ux_vector_active_document_pages UNIQUE (active_document_id, page_number),
    CONSTRAINT vector_active_document_pages_active_document_id_fkey FOREIGN KEY (active_document_id) REFERENCES vector_active_documents(active_document_id) ON DELETE CASCADE,
    CONSTRAINT vector_active_document_pages_dataset_id_fkey FOREIGN KEY (dataset_id) REFERENCES vector_dataset_master(dataset_id) ON DELETE CASCADE
);

CREATE TABLE vector_active_document_chunks (
    active_chunk_id UUID NOT NULL,
    active_document_id UUID NOT NULL,
    dataset_id UUID NOT NULL,
    page_number INTEGER,
    chunk_index INTEGER NOT NULL,
    chunk_text TEXT NOT NULL,
    vector_point_id TEXT,
    embedding_model TEXT,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT vector_active_document_chunks_pkey PRIMARY KEY (active_chunk_id),
    CONSTRAINT ux_vector_active_document_chunks UNIQUE (active_document_id, chunk_index),
    CONSTRAINT vector_active_document_chunks_active_document_id_fkey FOREIGN KEY (active_document_id) REFERENCES vector_active_documents(active_document_id) ON DELETE CASCADE,
    CONSTRAINT vector_active_document_chunks_dataset_id_fkey FOREIGN KEY (dataset_id) REFERENCES vector_dataset_master(dataset_id) ON DELETE CASCADE
);

CREATE TABLE vector_active_document_images (
    active_image_id UUID NOT NULL,
    active_document_id UUID NOT NULL,
    dataset_id UUID NOT NULL,
    page_number INTEGER,
    image_index INTEGER NOT NULL,
    storage_uri TEXT,
    mime_type TEXT,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT vector_active_document_images_pkey PRIMARY KEY (active_image_id),
    CONSTRAINT ux_vector_active_document_images UNIQUE (active_document_id, image_index),
    CONSTRAINT vector_active_document_images_active_document_id_fkey FOREIGN KEY (active_document_id) REFERENCES vector_active_documents(active_document_id) ON DELETE CASCADE,
    CONSTRAINT vector_active_document_images_dataset_id_fkey FOREIGN KEY (dataset_id) REFERENCES vector_dataset_master(dataset_id) ON DELETE CASCADE
);
```

### De-para de tabelas

Tabelas atuais que viram tabelas novas:

| Tabela atual (ativa hoje)     | Tabela nova (planejada)          | Regra de destino                                                                                  |
| ----------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------- |
| `ingestion_datasets`          | `vector_dataset_master`          | Vira a ficha master do acervo.                                                                    |
| `ingestion_runs`              | `vector_ingestion_runs`          | Continua como histórico de run, mas passa a ser ligado explicitamente ao master.                  |
| `ingestion_run_documents`     | `vector_ingestion_run_documents` | Continua como detalhe do lote, mas perde a ambiguidade com manifesto.                             |
| `ingestion_document_manifest` | `vector_active_documents`        | Deixa de ser manifesto híbrido e vira lista explícita de documentos publicados.                   |
| `ingestion_document_pages`    | `vector_active_document_pages`   | Passa a ligar ao documento ativo publicado.                                                       |
| `ingestion_document_chunks`   | `vector_active_document_chunks`  | Passa a ligar ao documento ativo publicado.                                                       |
| `ingestion_document_images`   | `vector_active_document_images`  | Passa a ligar ao documento ativo publicado.                                                       |

Tabelas atuais cujo conteúdo deixa de ser contrato principal:

| Tabela atual                            | Destino no novo modelo                                              | Motivo                                                                                                       |
| --------------------------------------- | ------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| `ingestion_dataset_generations`         | morre como entidade principal                                      | o modelo novo usa `last_published_run_id` e publicação explícita, sem camada de geração separada.            |
| `ingestion_document_versions`           | morre como entidade principal                                      | versionamento implícito passa a viver no próprio documento publicado e na relação com o item do run.        |
| `ingestion_document_source_occurrences` | morre como entidade principal                                      | provenance de origem passa a ficar no documento do run e no documento ativo.                                 |
| `ingestion_run_slot_leases`             | pode continuar só como infraestrutura de concorrência              | não é modelo de domínio do acervo, fica fora do contrato de leitura do produto.                             |
| `bm25_indexes`                          | permanece como tabela técnica de materialização física            | continua útil para operação física do índice, mas não define o acervo publicado.                            |

### De-para de campos (por tabela)

`ingestion_datasets` -> `vector_dataset_master`:

| Campo atual              | Campo novo                    | Regra                                                                                              |
| ------------------------ | ----------------------------- | -------------------------------------------------------------------------------------------------- |
| `dataset_id`             | `dataset_id`                  | mantém identidade do acervo.                                                                       |
| `tenant_code`            | `tenant_code`                 | mantém semântica.                                                                                  |
| `vectorstore_id`         | `vectorstore_id`              | mantém semântica.                                                                                  |
| `physical_vector_target` | `vector_target`               | vira nome explícito do alvo vetorial.                                                              |
| `physical_bm25_target`   | `bm25_target`                 | vira nome explícito do alvo BM25.                                                                  |
| `status`                 | `status`                      | mantém semântica de estado do acervo.                                                              |
| `if_exists_policy`       | `if_exists_policy`            | mantém semântica operacional.                                                                      |
| `active_generation_id`   | `last_published_run_id`       | deixa de apontar para geração e passa a apontar para o último run que alterou o acervo publicado.  |
| `updated_at`             | `last_sync_at` e `updated_at` | `last_sync_at` marca a última sincronização com mudança real; `updated_at` continua trilha da linha. |

`ingestion_runs` -> `vector_ingestion_runs`: mantém praticamente todos os campos com a mesma
semântica (`run_id`, `tenant_code`, `vectorstore_id`, `source_system`, `task_id`, `queued_at`,
`started_at`, `finished_at`, contadores, `status`, `correlation_id`, `error_summary`,
`status_message`, `metadata`). Mudança principal: o `dataset_id`, antes indireto, passa a ser
**FK explícita obrigatória** para `vector_dataset_master`.

`ingestion_run_documents` -> `vector_ingestion_run_documents`: mantém os campos descritivos do
item do lote (`run_document_id`, `run_id`, `tenant_code`, `vectorstore_id`, `document_path`,
`document_name`, `document_type`, `mime_type`, `file_size_bytes`, `document_hash`, `total_pages`,
`chunk_count`, `processing_seconds`, `status`, `error_message`, `metadata`). Mudanças:

| Campo atual                          | Campo novo               | Regra                                                                                                       |
| ------------------------------------ | ------------------------ | ----------------------------------------------------------------------------------------------------------- |
| `manifest_id`                        | morre como coluna direta | o vínculo com o documento ativo é resolvido por `source_run_document_id` em `vector_active_documents`.       |
| `external_document_id` (em metadata) | `external_document_id`   | vira coluna explícita.                                                                                       |
| `source_uri` (em metadata)           | `source_uri`             | vira coluna explícita.                                                                                       |
| `source_system` (herdado do run)     | `source_system`          | vira coluna explícita por documento.                                                                         |
| sem equivalente                      | `content_fingerprint`    | nasce para distinguir conteúdo físico de identidade de origem.                                               |
| sem equivalente                      | `publication_action`     | nasce para registrar `published`, `updated`, `unchanged`, `removed`, `skipped` ou `failed`.                  |

`ingestion_document_manifest` -> `vector_active_documents`:

| Campo atual                  | Campo novo               | Regra                                                                              |
| ---------------------------- | ------------------------ | ---------------------------------------------------------------------------------- |
| `manifest_id`                | `active_document_id`     | deixa de ser manifesto híbrido e vira id do documento publicado.                   |
| `document_identity_key`      | `canonical_document_key` | vira chave canônica explícita do documento publicado.                              |
| `last_run_id`                | `current_run_id`         | passa a ser FK obrigatória para o run que publicou a versão atual.                 |
| `pdf_binary_sha256`          | `content_fingerprint`    | generaliza o fingerprint para qualquer tipo de conteúdo.                           |
| `ingestion_status`           | `status`                 | reflete estado do documento publicado, sem misturar run com manifesto.            |
| `last_ingested_at`           | `published_at`           | passa a marcar a publicação atual no acervo.                                       |
| `active_document_version_id` | morre                    | não existe mais camada separada de versão ativa.                                   |
| sem equivalente              | `source_run_document_id` | nasce para apontar o item de lote que originou a publicação atual.                 |
| sem equivalente              | `vector_document_key`    | nasce para identificar o documento no banco vetorial quando existir esse conceito. |
| sem equivalente              | `bm25_document_key`      | nasce para identificar o documento no índice BM25 quando existir esse conceito.    |
| sem equivalente              | `removed_at`             | nasce para deleção lógica ou retirada do acervo.                                   |

(Os campos comuns de descrição — `tenant_code`, `vectorstore_id`, `source_system`,
`document_path`, `document_name`, `document_type`, `mime_type`, `file_size_bytes`,
`file_last_modified`, `external_document_id`, `document_hash`, `metadata` — mantêm a semântica.)

Tabelas de conteúdo detalhado: `ingestion_document_pages/chunks/images` ->
`vector_active_document_pages/chunks/images`, trocando o vínculo `manifest_id` por
`active_document_id`; `vector_active_document_chunks` passa a aceitar `vector_point_id` explícito.

### Campos que morrem e campos que nascem

Morrem no novo contrato principal: `ingestion_datasets.active_generation_id`,
`ingestion_document_manifest.active_document_version_id`, toda a tabela
`ingestion_document_versions` (`document_version_id`, `status`, `activated_at`,
`superseded_at`), `ingestion_document_source_occurrences.canonical_source_key` como eixo
principal, e qualquer lógica que exija `manifest_id + document_version_id` para descobrir o que
está ativo. Em resumo: a camada de geração, manifesto híbrido e versão ativa separada deixa de
ser o centro do modelo.

Nascem: em `vector_dataset_master` — `vector_provider`, `vector_target`, `bm25_provider`,
`bm25_target`, `last_published_run_id`, `last_completed_run_id`, `last_sync_at`. Em
`vector_ingestion_run_documents` — `source_uri`, `external_document_id`, `content_fingerprint`,
`publication_action`. Em `vector_active_documents` — `canonical_document_key`, `current_run_id`,
`source_run_document_id`, `content_fingerprint`, `vector_document_key`, `bm25_document_key`,
`removed_at`.

### Regras de gravação (quando o corte ocorrer)

- `vector_dataset_master`: uma linha única por `tenant_code + vectorstore_id`, criada no
  bootstrap do acervo. Atualizar `vector_target`/`bm25_target` quando o acervo físico mudar;
  `last_published_run_id` só quando um run alterar de fato o acervo publicado;
  `last_completed_run_id` quando o run terminar com sucesso (mesmo sem publicar nada novo);
  `last_sync_at` só quando o estado publicado mudar de verdade.
- `vector_ingestion_runs`: uma linha por execução pai, com `dataset_id` obrigatório; status
  atualizado ao longo do run; contadores fechados no término. Nunca é leitura primária do
  acervo vivo.
- `vector_ingestion_run_documents`: uma linha por item observado. `status` conta o que aconteceu
  no processamento; `publication_action` conta o efeito sobre o acervo ativo
  (`published`/`updated`/`unchanged`/`removed`/`skipped`/`failed`). O vínculo com o documento
  ativo não é gravado aqui — é resolvido por `source_run_document_id` em `vector_active_documents`.
  `content_fingerprint` representa o conteúdo físico processado, não a identidade de origem.
- `vector_active_documents`: uma linha ativa por `dataset_id + canonical_document_key`. Conteúdo
  novo do mesmo documento atualiza a mesma linha e troca `current_run_id`,
  `source_run_document_id`, `content_fingerprint`, `document_hash` e `published_at`. Remoção
  lógica marca `status` e `removed_at` antes de qualquer passo destrutivo físico. É a tabela que
  a UI deve usar para contar documentos vivos.
- Tabelas de detalhe: sempre representam a versão atualmente publicada. Ao publicar/atualizar um
  documento ativo, o detalhe físico é recriado de forma consistente e o detalhe antigo do mesmo
  `active_document_id` deixa de valer. Chunks ativos carregam `vector_point_id` quando existir.

### Regras de leitura (quando o corte ocorrer)

- **UI administrativa:** separar explicitamente três perguntas — quantos runs existem
  (`vector_ingestion_runs`), quantos documentos o último run viu
  (`vector_ingestion_run_documents`) e quantos documentos estão vivos agora
  (`vector_active_documents`). Resumo do acervo vem de `vector_dataset_master`. É proibido
  deduzir "documentos vivos" a partir de `vector_ingestion_runs` ou "documentos do lote" a
  partir de `vector_active_documents`.
- **Backend de ingestão:** escreve o histórico do lote em `vector_ingestion_runs` e
  `vector_ingestion_run_documents`; decide publicação olhando o documento ativo por
  `dataset_id + canonical_document_key`; atualiza `vector_active_documents` só no publish do
  item; atualiza `vector_dataset_master.last_published_run_id` só quando o lote alterar o acervo
  publicado; em ingestão incremental, documentos ativos de runs antigos continuam no acervo até
  serem substituídos ou removidos.
- **Backend de RAG:** não usa `vector_ingestion_run_documents` como fonte do acervo consultável.
  Usa `vector_dataset_master` para resolver acervo e target físico, `vector_active_documents`
  para o conjunto lógico ativo quando precisar de suporte no PostgreSQL, e
  `vector_active_document_chunks` para trilhas que dependam do conteúdo persistido do documento ativo.

### Ordem única de execução do corte (quando for decidido fazer)

1. criar todas as tabelas novas;
2. implementar a escrita nova no runtime sem reaproveitar as tabelas antigas como fonte primária;
3. popular o novo modelo a partir de uma rodada controlada de rebuild do acervo;
4. apontar a UI apenas para as tabelas novas;
5. apontar as leituras administrativas e operacionais apenas para as tabelas novas;
6. bloquear escrita funcional nas tabelas antigas;
7. remover do código qualquer dependência de leitura funcional das tabelas antigas;
8. remover as tabelas antigas do contrato de produto e deixá-las aptas para drop definitivo.

O corte precisa ser claro e completo: o sistema novo nasce inteiro, a leitura muda inteira e o
contrato antigo deixa de valer inteiro. O que não pode existir é convivência longa entre dois
modelos principais dizendo coisas diferentes sobre o mesmo acervo — exatamente a situação
transitória de hoje, que esta seção documenta para resolver no futuro.

## Domínio Interações e Eventos

### interaction_runs

- Finalidade prática: registrar a interação principal entre usuário e sistema.
- Chave primária: `interaction_id`.
- Colunas:
- `interaction_id`: identificador UUID da interação.
- `tenant_id`: tenant da execução.
- `client_code`: código do cliente.
- `source`: origem da chamada.
- `user_email`: e-mail do usuário, quando houver.
- `channel`: nome do canal.
- `channel_id`: identificador do canal.
- `customer_identifier`: identificador do cliente final.
- `workflow_id`: identificador do workflow.
- `workflow_name`: nome do workflow.
- `agent_id`: identificador do agente.
- `agent_name`: nome do agente.
- `vectorstore_id`: coleção consultada.
- `question_text`: pergunta recebida.
- `answer_text`: resposta enviada.
- `input_tokens`: tokens de entrada.
- `output_tokens`: tokens de saída.
- `total_tokens`: coluna gerada automaticamente pela soma de entrada e saída.
- `latency_ms`: latência total em milissegundos.
- `cost_usd`: custo estimado em dólar.
- `sentiment_score`: score de sentimento.
- `confidence_score`: score de confiança.
- `metadata`: metadados adicionais em `jsonb`.
- `error_flag`: indica erro na interação.
- `error_message`: mensagem de erro.
- `correlation_id`: correlação com logs.
- `request_timestamp`: instante da requisição.
- `response_timestamp`: instante da resposta.
- `created_at`: criação do registro.
- `evidence_summary`: resumo das evidências em `jsonb`.
- `observacoes`: observações livres.
- `no_answer`: indica ausência de resposta útil.
- Índices e restrições:
- PK em `interaction_id`.
- Índice `ix_interaction_runs_correlation` em `correlation_id`.
- Índice `ix_interaction_runs_created_at` em `created_at` descendente.
- Índice parcial `ix_interaction_runs_vectorstore` em `vectorstore_id` quando o campo não é nulo.
- Trigger `tr_interaction_runs_tokens` executa `fn_interaction_runs_total_tokens()` antes de inserção ou atualização.

### interaction_run_events

- Finalidade prática: guardar eventos granulares associados a uma interação principal.
- Chave primária: `event_id`.
- Colunas:
- `event_id`: identificador UUID do evento.
- `interaction_id`: interação pai.
- `event_type`: tipo do evento.
- `event_timestamp`: horário do evento.
- `payload`: dados do evento em `jsonb`.
- `created_at`: criação do registro.
- Índices e restrições:
- PK em `event_id`.
- FK `interaction_id` para `interaction_runs.interaction_id` com `ON DELETE CASCADE`.
- Índice `ix_interaction_run_events_event_type` em `event_type`.
- Índice `ix_interaction_run_events_interaction` em `interaction_id`.

### ag_ui.run_events

- Finalidade prática: guardar o replay durável dos eventos do protocolo AG-UI por run e por thread.
- O que resolve na prática: evita perder a trilha visual quando a API reinicia, quando há mais de uma réplica ou quando o operador precisa reconstruir uma execução depois do fato.
- Chave primária: `event_id`.
- Colunas:
- `event_id`: identificador UUID do evento persistido.
- `tenant_id`: tenant opcional da execução, quando o boundary autenticado disponibiliza esse contexto.
- `correlation_id`: correlação ponta a ponta da execução AG-UI.
- `thread_id`: thread formal do protocolo AG-UI.
- `run_id`: run visual ao qual a sequência pertence.
- `sequence`: ordem monotônica do evento dentro do run.
- `event_type`: tipo do evento AG-UI persistido.
- `payload`: payload sanitizado em `jsonb`, sem token, senha, DSN ou segredo.
- `created_at`: instante persistido do evento.
- Índices e restrições:
- PK em `event_id`.
- Unique `ux_ag_ui_run_events_run_sequence` em `run_id, sequence`, protegendo idempotência do replay.
- Check `ag_ui_run_events_sequence_check` exigindo `sequence >= 1`.
- Check `ag_ui_run_events_payload_json_check` garantindo `payload` como objeto JSON.
- Índice `idx_ag_ui_run_events_thread_time` em `thread_id, created_at ASC, run_id, sequence` para replay de thread.
- Índice `idx_ag_ui_run_events_correlation_time` em `correlation_id, created_at DESC` para investigação operacional.
- Índice parcial `idx_ag_ui_run_events_tenant_time` em `tenant_id, created_at DESC` quando o tenant existir.
- Índice `idx_ag_ui_run_events_type_time` em `event_type, created_at DESC` para suporte e diagnóstico por tipo de evento.
- Onde é usado e como:
- `src/api/services/ag_ui_event_store.py` resolve o provider canônico do AG-UI e grava/lê essa tabela quando `AG_UI_EVENT_STORE_PROVIDER=postgres`.
- `src/api/routers/ag_ui_router.py` usa esse replay em `GET /ag-ui/runs/{run_id}/events` e `GET /ag-ui/threads/{thread_id}/events`.
- Em linguagem simples: esta tabela é a memória durável da experiência AG-UI. Ela não substitui o runtime do agente; ela preserva a história visual já sanitizada para replay, suporte e auditoria.

### agent_hil_approval_requests

Esta tabela registra o pedido formal de aprovação humana quando um agente em background para em um ponto sensível e precisa esperar decisão externa, como WhatsApp ou e-mail. Ela não substitui o checkpointer do runtime agentic. O checkpointer continua sendo o lugar onde o LangGraph guarda o estado interno da execução. A função desta tabela é outra: guardar o lado auditável e operacional da aprovação humana.

Em termos conceituais, pense nela como a fila oficial de aprovações pendentes. Cada linha representa uma pausa que pode ser aprovada, editada ou rejeitada. A tabela também guarda como a notificação foi enviada, quem deveria responder, se o token ainda vale, quando a decisão chegou e qual foi o resultado final.

Em linguagem simples: quando o agente para e pede ajuda humana, esta tabela vira o “protocolo” do pedido. Ela responde perguntas práticas como estas: qual execução parou, para quem o pedido foi enviado, por qual canal, se a aprovação ainda está em aberto, se já expirou, quem decidiu e quando a execução pode continuar com segurança.

- Finalidade prática: materializar a pausa Human-in-the-Loop assíncrona como registro durável, auditável e seguro.
- O que ela resolve na prática: evita depender apenas de estado efêmero em memória ou Redis para uma aprovação humana que pode precisar de rastreabilidade posterior.
- Chave primária: `approval_request_id`.
- Colunas:
- `approval_request_id`: identificador UUID do pedido de aprovação.
- `correlation_id`: correlação ponta a ponta da execução original, usada para juntar aprovação, logs e retomada.
- `thread_id`: thread formal do runtime agentic que deverá ser retomada depois da decisão.
- `task_id`: identificador da task assíncrona, quando a execução tiver sido iniciada em background com controle de progresso.
- `user_email`: e-mail do usuário associado à execução original.
- `user_code`: código interno do usuário autenticado que originou a execução.
- `tenant_id`: tenant ao qual o pedido pertence, quando esse contexto existir de forma explícita.
- `client_code`: código lógico do cliente, útil para filtros operacionais e multi-tenant.
- `supervisor_id`: supervisor agentic responsável pela execução pausada.
- `agent_mode`: modo do runtime que gerou a pausa, como `agent`, `deepagent` ou `workflow`.
- `protocol_version`: versão do contrato público HIL associado ao pedido.
- `action_requests`: lista em `jsonb` das ações pendentes que precisam de revisão humana.
- `review_configs`: regras em `jsonb` que descrevem como cada ação pode ser revisada.
- `allowed_decisions`: decisões aceitas pelo contrato, armazenadas em `jsonb`, como `approve`, `edit` e `reject`.
- `status`: estado macro do pedido, como `pending`, `resolved`, `expired`, `failed` ou `cancelled`.
- `notification_status`: estado do envio da notificação, como `not_started`, `sent`, `partial` ou `failed`.
- `approval_token_hash`: hash do token de aprovação. O token real não deve ser persistido em texto puro.
- `approval_token_hint`: dica curta do token para suporte operacional, sem revelar o segredo completo.
- `expected_approver_email`: e-mail esperado do aprovador, quando a política exigir essa validação.
- `expected_channel`: canal esperado para a resposta, como `whatsapp` ou `email`.
- `expected_channel_user_id`: identificador esperado do usuário no canal, como telefone/wa_id ou identificador equivalente.
- `notification_channel`: canal efetivamente usado no envio da notificação.
- `notification_provider`: provider concreto usado no envio, como Meta WhatsApp Cloud API ou SMTP.
- `provider_message_id`: identificador devolvido pelo provider para rastrear a mensagem enviada.
- `decision_type`: decisão final aplicada ao pedido, como `approve`, `edit` ou `reject`.
- `decision_payload`: payload em `jsonb` com os detalhes da decisão, por exemplo edição manual dos argumentos da ação.
- `decided_by_email`: e-mail do aprovador que efetivamente resolveu o pedido, quando houver.
- `decided_by_user_code`: código interno do aprovador, quando esse mapeamento existir.
- `decided_channel`: canal pelo qual a decisão chegou de fato.
- `decided_channel_user_id`: identificador do respondente no canal que produziu a decisão.
- `decided_at`: instante em que a decisão foi aceita como válida pelo sistema.
- `expires_at`: prazo máximo para a aprovação continuar válida.
- `created_at`: criação do pedido de aprovação.
- `updated_at`: última atualização do pedido.
- `metadata`: metadados auxiliares em `jsonb` para telemetria, troubleshooting e governança.
- Índices e restrições:
- PK em `approval_request_id`.
- Unique em `approval_token_hash` para impedir dois pedidos com o mesmo token lógico.
- Unique parcial `ux_agent_hil_approval_requests_active_pause` em `correlation_id, thread_id` quando `status = 'pending'`, impedindo duas pausas abertas para a mesma execução pausada.
- Índice `idx_agent_hil_approval_requests_correlation_thread` em `correlation_id, thread_id` para lookup rápido na retomada.
- Índice parcial `idx_agent_hil_approval_requests_task` em `task_id` quando o campo não é nulo.
- Índice parcial `idx_agent_hil_approval_requests_pending_expiration` em `expires_at` quando `status = 'pending'`, útil para expiração e manutenção.
- Índice parcial `idx_agent_hil_approval_requests_expected_approver` em `expected_approver_email, status, expires_at` quando houver aprovador explícito.
- Índice parcial `idx_agent_hil_approval_requests_channel_user` em `expected_channel, expected_channel_user_id, status` para validar decisões vindas do canal correto.
- Índice `idx_agent_hil_approval_requests_tenant_status` em `tenant_id, client_code, status, created_at DESC` para operação multi-tenant.
- Check `agent_hil_approval_requests_agent_mode_check` limitando `agent_mode` a `agent`, `deepagent` ou `workflow`.
- Check `agent_hil_approval_requests_status_check` limitando `status` a `pending`, `resolved`, `expired`, `failed` ou `cancelled`.
- Check `agent_hil_approval_requests_notification_status_check` limitando `notification_status` aos estados operacionais previstos.
- Check `agent_hil_approval_requests_expected_channel_check` limitando o canal esperado aos conectores aceitos.
- Check `agent_hil_approval_requests_decision_type_check` limitando `decision_type` a `approve`, `edit` ou `reject`.
- Checks JSON garantindo `action_requests` e `review_configs` como array e `decision_payload` e `metadata` como objeto.
- Check `agent_hil_approval_requests_resolved_requires_decision_check` exigindo `decision_type` e `decided_at` quando `status = 'resolved'`.
- Check `agent_hil_approval_requests_expires_after_created_check` exigindo `expires_at > created_at`.

#### Estrutura operacional da tabela HIL

Em termos práticos, o DDL formaliza cinco garantias estruturais:

1. a tabela usa UUID com `pgcrypto` para `approval_request_id`;
2. existe unicidade forte para `approval_token_hash` e para a pausa pendente por `correlation_id + thread_id`;
3. `agent_mode`, `status`, `notification_status`, `expected_channel` e `decision_type` ficam limitados por checks explícitos;
4. os blocos JSON operacionais, como `action_requests`, `review_configs`, `allowed_decisions`, `decision_payload` e `metadata`, são validados pelo banco para manter o tipo esperado;
5. a tabela recebe índices separados para retomada por thread, expiração, aprovador esperado, canal esperado e operação multi-tenant.

Isso importa porque a integridade de HIL não depende só da aplicação. O banco também impede estados impossíveis, como duas pausas pendentes para a mesma execução ou uma aprovação resolvida sem decisão e sem timestamp final.

## Domínio Execução Agentic em Background

### Visão geral do schema agent_background

O schema `agent_background` materializa a capacidade de executar pedidos agentic em segundo plano sem prender a API nem o chat.

Em termos práticos, ele separa sete peças que não devem ficar misturadas:

- o alvo autorizado que pode ser executado;
- a solicitação criada a partir do prompt do usuário;
- a agenda que decide quando isso deve acontecer;
- o run concreto que de fato foi executado;
- os eventos persistidos do ledger operacional;
- a aprovação humana durável ligada ao run, quando houver HIL;
- a fila de outbox para publicação e integração assíncrona.

Essa separação é importante porque pedido, agenda e execução são coisas diferentes. Se tudo fosse para uma tabela só, a operação perderia clareza, o histórico ficaria frágil e seria muito mais difícil investigar atraso, duplicidade, HIL pendente ou falha do worker.

### Onde este schema é usado no código

- `src/agentic_layer/background_execution/services.py` usa o schema para criar solicitações, listar agendas, cancelar, reagendar e consultar runs.
- `src/agentic_layer/background_execution/runtime.py` usa o ledger para carregar o contexto do run, persistir thread, refletir resultado e integrar HIL assíncrono.
- `src/agentic_layer/tools/system_tools/background_execution.py` expõe as tools internas que operam esse schema sem aceitar `tenant_id` livre pelo prompt.
- `src/api/services/admin/background_execution_service.py` e `src/api/routers/admin/background_execution_router.py` usam o schema para leitura administrativa de requests, schedules, runs, eventos e pendências HIL.
- `src/api/services/hil_approval_decision_service.py` e `src/api/services/background_execution_hil_run_finalizer.py` sincronizam a decisão humana com o run do ledger background.

Em linguagem simples: o schema `agent_background` é o banco da história operacional completa da execução background, enquanto o código acima é o conjunto de portas e serviços que lê e escreve essa história.

### background_execution_targets

- Finalidade prática: cadastrar quais alvos agentic podem ser executados em background para cada tenant.
- O que resolve na prática: impede que o runtime execute um agente, deepagent ou workflow apenas porque o prompt pediu. O alvo precisa existir e estar habilitado.
- Chave primária: `target_id`.
- Colunas:
- `target_id`: identificador UUID do alvo cadastrado.
- `tenant_id`: tenant dono do alvo.
- `client_code`: código lógico do cliente, quando houver.
- `target_type`: tipo do alvo, como `agent`, `deepagent` ou `workflow`.
- `target_ref`: referência lógica do alvo dentro do runtime.
- `display_name`: nome amigável do alvo.
- `description`: explicação curta do alvo para operação.
- `enabled`: indica se o alvo pode ser usado.
- `default_timezone`: timezone operacional padrão.
- `default_concurrency_policy`: política padrão de concorrência.
- `default_retry_policy`: política padrão de retry em `jsonb`.
- `metadata`: metadados auxiliares em `jsonb`.
- `created_at`: criação do cadastro.
- `updated_at`: última atualização do cadastro.
- Índices e restrições:
- PK em `target_id`.
- Unique em `tenant_id, target_type, target_ref`, impedindo duplicidade lógica do mesmo alvo no mesmo tenant.
- Check em `target_type` limitando o contrato a `agent`, `deepagent` e `workflow`.
- Check em `default_concurrency_policy` limitando as políticas operacionais previstas.
- Checks JSON garantindo `default_retry_policy` e `metadata` como objeto.
- Índice `idx_background_targets_tenant_enabled` em `tenant_id, enabled, target_type` para lookup rápido do catálogo por tenant.
- Onde é usado e como:
- `BackgroundExecutionService.schedule_request(...)` consulta essa tabela antes de criar a solicitação, usando `get_enabled_target(...)` no repositório.
- A tool `schedule_background_execution_request` depende desse cadastro para validar que o alvo solicitado existe e está autorizado.

### background_execution_requests

- Finalidade prática: registrar a solicitação criada a partir do prompt do usuário.
- O que resolve na prática: guarda a intenção original sem executar o comando imediatamente no chat.
- Chave primária: `request_id`.
- Colunas:
- `request_id`: identificador UUID da solicitação.
- `tenant_id`: tenant da solicitação.
- `client_code`: código lógico do cliente.
- `user_email`: usuário que criou o pedido.
- `user_code`: código interno do usuário, quando existir.
- `source_channel`: canal de origem do pedido.
- `source_conversation_id`: conversa de origem, quando houver.
- `target_id`: alvo autorizado escolhido para executar o pedido.
- `requested_command`: comando original pedido pelo usuário.
- `normalized_command`: comando normalizado para operação e auditoria.
- `input_payload`: payload complementar de entrada em `jsonb`.
- `yaml_snapshot_hash`: hash do snapshot YAML usado no run.
- `yaml_snapshot`: snapshot seguro da configuração agentic.
- `context_snapshot`: contexto operacional persistido em `jsonb`.
- `status`: estado macro da solicitação, como `draft`, `scheduled`, `paused`, `cancelled`, `completed` ou `failed`.
- `created_correlation_id`: correlação original da criação do pedido.
- `metadata`: metadados auxiliares em `jsonb`.
- `created_at`: criação da solicitação.
- `updated_at`: última atualização.
- `cancelled_at`: data de cancelamento, quando aplicável.
- Índices e restrições:
- PK em `request_id`.
- FK `target_id` para `background_execution_targets.target_id` com `ON DELETE RESTRICT`.
- Check em `status` limitando os estados válidos do pedido.
- Checks JSON garantindo `input_payload`, `yaml_snapshot`, `context_snapshot` e `metadata` no tipo esperado.
- Índices por tenant, usuário, `created_correlation_id` e `target_id` para consulta operacional.
- Onde é usado e como:
- `BackgroundExecutionService.schedule_request(...)` grava a solicitação junto com a agenda.
- As APIs administrativas listam essas linhas para mostrar o que foi pedido, por quem, para qual alvo e com qual status.

### background_execution_schedules

- Finalidade prática: preservar a trilha histórica do modelo antigo do slice background.
- O que resolve na prática: mantém rastreabilidade de migração e compatibilidade controlada para leitura histórica, sem voltar a ser a agenda canônica do runtime.
- Chave primária: `schedule_id`.
- Colunas:
- `schedule_id`: identificador UUID da agenda.
- `request_id`: solicitação dona da agenda.
- `tenant_id`: tenant da agenda.
- `schedule_type`: tipo da agenda, como `once`, `interval` ou `cron`.
- `run_at`: instante planejado para execução única.
- `cron_expression`: expressão cron quando a agenda é recorrente por calendário.
- `interval_seconds`: intervalo recorrente em segundos.
- `timezone`: timezone de cálculo da agenda.
- `next_run_at`: próxima execução planejada.
- `last_run_at`: último instante executado.
- `last_run_id`: último run associado.
- `misfire_policy`: política para janela perdida.
- `concurrency_policy`: política para concorrência entre runs da mesma agenda.
- `max_parallel_runs`: limite de paralelismo permitido.
- `enabled`: indica se a agenda está habilitada.
- `status`: estado da agenda, como `active`, `paused`, `cancelled`, `completed` ou `failed`.
- `metadata`: metadados auxiliares em `jsonb`.
- `created_at`: criação da agenda.
- `updated_at`: última atualização.
- `cancelled_at`: cancelamento, quando houver.
- Índices e restrições:
- PK em `schedule_id`.
- FK `request_id` para `background_execution_requests.request_id` com `ON DELETE CASCADE`.
- FK `last_run_id` para `background_execution_runs.run_id` com `ON DELETE SET NULL`.
- Checks garantindo coerência entre `schedule_type`, `run_at`, `cron_expression` e `interval_seconds`.
- Checks limitando `misfire_policy`, `concurrency_policy`, `status` e `max_parallel_runs`.
- Índice `idx_background_schedules_due` para localizar rapidamente agendas ativas vencidas.
- Índices por tenant, status e `request_id` para leitura operacional.
- Onde é usado e como:
- Esse registro não governa mais o disparo oficial do runtime.
- O scheduling novo do background nasce em `scheduler.scheduled_jobs`, enquanto a solicitação de domínio continua em `background_execution_requests`.

### background_execution_runs

- Finalidade prática: registrar a projeção compatível de cada execução concreta visível para o slice background.
- O que resolve na prática: mantém histórico operacional, HIL e APIs administrativas do domínio background sem transformar esse ledger na fonte canônica do scheduler.
- Chave primária: `run_id`.
- Colunas:
- `run_id`: identificador UUID do run.
- `request_id`: solicitação que originou o run.
- `schedule_id`: agenda que disparou o run, quando houver.
- `tenant_id`: tenant do run.
- `client_code`: código lógico do cliente.
- `user_email`: usuário dono da solicitação original.
- `user_code`: código interno do usuário.
- `target_id`: alvo executado.
- `planned_run_at`: janela planejada da execução.
- `started_at`: início real do run.
- `finished_at`: término real do run.
- `status`: estado do run, como `queued`, `dispatching`, `running`, `waiting_hil`, `completed`, `failed`, `cancelled`, `expired` ou `skipped`.
- `correlation_id`: correlação ponta a ponta do run.
- `thread_id`: thread formal do runtime agentic.
- `worker_id`: worker que processou o run, quando houver.
- `attempt_number`: tentativa corrente.
- `max_attempts`: máximo de tentativas permitidas.
- `final_response`: resposta final textual do runtime.
- `result_payload`: resultado estruturado em `jsonb`.
- `telemetry`: telemetria operacional em `jsonb`.
- `error_type`: tipo de erro, quando houver falha.
- `error_message`: mensagem de erro estruturada.
- `error_payload`: detalhes do erro em `jsonb`.
- `metadata`: metadados auxiliares em `jsonb`.
- `created_at`: criação do run.
- `updated_at`: última atualização.
- Índices e restrições:
- PK em `run_id`.
- FKs para `background_execution_requests`, `background_execution_schedules` e `background_execution_targets`.
- Unique parcial `ux_background_runs_schedule_window` em `schedule_id, planned_run_at`, protegendo idempotência da janela planejada.
- Check em `status` limitando os estados do ledger.
- Check em `attempt_number` e `max_attempts` exigindo valores positivos.
- Checks JSON garantindo `result_payload`, `telemetry`, `error_payload` e `metadata` como objeto.
- Índices por tenant, usuário, `correlation_id` e runs ativos.
- Onde é usado e como:
- `AgenticBackgroundExecutionRuntime.execute_run(...)` lê o contexto do run e persiste thread, resultado, telemetria e erro.
- As APIs administrativas usam essa tabela para listar runs recentes, ativos e detalhar um run específico.
- As tools `get_background_execution_result`, `get_last_background_execution_result`, `list_recent_background_executions` e `list_running_background_agents` consultam esse ledger.
- A verdade canônica do disparo continua em `scheduler.job_executions`; esta tabela espelha o que o slice background precisa para operar HIL e auditoria própria.

### background_execution_events

- Finalidade prática: guardar os eventos granulares persistidos do ledger background.
- O que resolve na prática: permite reconstruir a história do run mesmo quando o operador não está olhando o log bruto em tempo real.
- Chave primária: `event_id`.
- Colunas:
- `event_id`: identificador UUID do evento.
- `run_id`: run ao qual o evento pertence.
- `request_id`: solicitação relacionada.
- `tenant_id`: tenant do evento.
- `event_type`: tipo do evento.
- `event_timestamp`: horário do evento.
- `severity`: severidade operacional.
- `message`: mensagem principal do evento.
- `payload`: dados estruturados do evento em `jsonb`.
- `correlation_id`: correlação ponta a ponta.
- `created_at`: criação do registro.
- Índices e restrições:
- PK em `event_id`.
- FKs para `background_execution_runs` e `background_execution_requests` com `ON DELETE CASCADE`.
- Check em `severity` limitando os níveis aceitos.
- Check JSON garantindo `payload` como objeto.
- Índices por `run_id`, por `tenant_id + event_type` e por `correlation_id`.
- Onde é usado e como:
- O repositório PostgreSQL do slice grava e lista esses eventos para leitura operacional.
- A API administrativa de eventos usa essa tabela para investigação por `run_id` ou `correlation_id`.

### agent_background.agent_hil_approval_requests

- Finalidade prática: guardar a pausa HIL durável especificamente ligada ao ledger de background execution.
- O que resolve na prática: liga a aprovação humana diretamente ao `run_id`, em vez de deixar a decisão solta apenas na correlação e na thread.
- Chave primária: `approval_request_id`.
- Colunas:
- `approval_request_id`: identificador UUID do pedido HIL.
- `run_id`: run background ao qual a pausa pertence.
- `correlation_id`: correlação ponta a ponta da execução.
- `thread_id`: thread formal que será retomada.
- `task_id`: tarefa assíncrona relacionada, quando houver.
- `user_email`: usuário da execução original.
- `user_code`: código interno do usuário. No DDL atual este campo é obrigatório.
- `tenant_id`: tenant da aprovação.
- `client_code`: código lógico do cliente.
- `supervisor_id`: supervisor responsável pelo runtime pausado.
- `agent_mode`: modo do runtime, como `agent`, `deepagent` ou `workflow`.
- `protocol_version`: versão do contrato HIL usado.
- `action_requests`: ações pendentes em `jsonb`.
- `review_configs`: regras de revisão em `jsonb`.
- `allowed_decisions`: decisões aceitas em `jsonb`.
- `status`: estado do pedido, como `pending`, `resolved`, `expired`, `failed` ou `cancelled`.
- `notification_status`: estado do envio da notificação.
- `approval_token_hash`: hash do token de aprovação.
- `approval_token_hint`: dica operacional curta do token.
- `expected_approver_email`: aprovador esperado, quando houver política explícita.
- `expected_channel`: canal esperado para resposta.
- `expected_channel_user_id`: usuário esperado no canal.
- `notification_channel`: canal efetivamente usado no envio.
- `notification_provider`: provider concreto do envio.
- `provider_message_id`: identificador retornado pelo provider.
- `decision_type`: decisão final aceita.
- `decision_payload`: detalhes da decisão em `jsonb`.
- `decided_by_email`: e-mail do aprovador que respondeu.
- `decided_by_user_code`: código interno do aprovador.
- `decided_channel`: canal que trouxe a decisão.
- `decided_channel_user_id`: usuário do canal que respondeu.
- `decided_at`: momento em que a decisão foi aceita.
- `expires_at`: prazo limite da aprovação.
- `created_at`: criação do pedido.
- `updated_at`: última atualização.
- `metadata`: metadados auxiliares em `jsonb`.
- Índices e restrições:
- PK em `approval_request_id`.
- FK `run_id` para `background_execution_runs.run_id` com `ON DELETE SET NULL`.
- Unique em `approval_token_hash`.
- Checks em `agent_mode`, `status`, `notification_status` e `decision_type`.
- Checks JSON garantindo arrays em `action_requests`, `review_configs`, `allowed_decisions` e objeto em `decision_payload` e `metadata`.
- Índice `idx_agent_hil_approval_pending_pause` em `correlation_id, thread_id, status, created_at DESC` para lookup rápido da pausa por execução e estado.
- Índice `idx_agent_hil_approval_tenant_user_status` em `tenant_id, user_email, status, expires_at ASC` para operação multi-tenant e manutenção.
- Índice parcial `idx_agent_hil_approval_run` em `run_id` quando o campo não é nulo.
- Onde é usado e como:
- `hil_approval_decision_service` e `background_execution_hil_run_finalizer` reconciliam a decisão humana com o run background.
- A API administrativa de HIL lista essas linhas para mostrar pendência, expiração, canal, aprovador esperado e decisão final.

### background_execution_outbox

- Finalidade prática: guardar publicações assíncronas pendentes do domínio de background execution.
- O que resolve na prática: evita perder a intenção de publicar um evento ou despacho quando a gravação no banco já aconteceu, mas a entrega assíncrona ainda não foi concluída.
- Chave primária: `outbox_id`.
- Colunas:
- `outbox_id`: identificador UUID da mensagem de outbox.
- `tenant_id`: tenant dono do item.
- `event_type`: tipo de evento a publicar.
- `aggregate_type`: agregado de domínio relacionado.
- `aggregate_id`: identificador do agregado.
- `payload`: conteúdo estruturado da publicação em `jsonb`.
- `status`: estado operacional da publicação, como `pending`, `published`, `failed` ou `dead_letter`.
- `attempt_count`: contador de tentativas.
- `next_attempt_at`: próxima tentativa planejada.
- `last_error`: último erro resumido.
- `created_at`: criação do item.
- `updated_at`: última atualização.
- `published_at`: momento de publicação bem-sucedida, quando houver.
- Índices e restrições:
- PK em `outbox_id`.
- Check em `status` limitando os estados do outbox.
- Check em `attempt_count` impedindo valor negativo.
- Check JSON garantindo `payload` como objeto.
- Índice `idx_background_outbox_pending` para drenagem operacional de itens `pending` e `failed` por ordem de próxima tentativa.
- Índice `idx_background_outbox_aggregate` por `aggregate_type, aggregate_id`.
- Onde é usado e como:
- nos pontos de código lidos nesta sessão, não apareceu consumidor Python direto dessa tabela. Hoje o DDL deixa esse contrato durável pronto para publicação e reconciliação assíncrona do domínio background, sem precisar introduzir fila ad hoc fora do schema.

## Domínio Tenants e Segurança

### tenants

- Finalidade prática: cadastro central de tenants da plataforma.
- Chave primária: `tenant_id`.
- Colunas:
- `tenant_id`: identificador interno do tenant.
- `client_code`: código lógico do cliente.
- `display_name`: nome exibido.
- `domain`: domínio associado.
- `tier`: nível comercial ou técnico.
- `is_anonymous_flow`: indica fluxo anônimo.
- `is_active`: indica se o tenant está ativo.
- `metadata_json`: metadados do tenant em `jsonb`.
- `default_user_email`: usuário padrão associado.
- `created_at`: criação do registro.
- `updated_at`: última atualização do registro.
- `cnpj`: CNPJ do cliente.
- `website`: site do cliente.
- `email_comercial`: e-mail comercial.
- `telefone_contato`: telefone de contato.
- `meta_app_id`: app id da integração Meta.
- `meta_access_token`: token de acesso Meta.
- `meta_whatsapp_business_account_id`: identificador WABA.
- `meta_graph_api_version`: versão da Graph API.

### Visao geral do schema integrations

O schema `integrations` concentra o cadastro governado das integracoes externas e das tools parametrizadas da plataforma. Em termos conceituais, ele separa o que e catalogo global da plataforma do que e cadastro multi-tenant de conectores e ativos funcionais. Assim, a plataforma consegue administrar tools builtin, grupos funcionais, credenciais, fontes OpenAPI, conexoes SQL, endpoints HTTP, queries e procedures de forma rastreavel, sem depender de configuracao solta espalhada em arquivos. No runtime atual mais evidenciado em codigo, esse schema ja participa diretamente da resolucao governada de `dyn_api<...>` e `dyn_sql<...>`, enquanto o catalogo builtin global e carregado do banco pelo cache central.

Em linguagem simples: pense nesse schema como uma prateleira oficial de integracoes aprovadas. Em vez de cada time repetir URL, segredo, query ou nome de tool em varios lugares, tudo fica guardado em tabelas com dono, status, descricao e vinculos claros. Uma parte dessa prateleira serve para a propria plataforma saber quais tools builtin existem. Outra parte serve para cada tenant registrar as suas conexoes, endpoints, queries e procedures homologadas. Quando um agente ou servico precisa usar uma capacidade governada, ele consulta essa prateleira primeiro, valida se o item esta ativo e so depois monta a execucao.

### Tabela global do catalogo builtin

- `integrations.builtin_tool_registry` e global, sem `tenant_id`, porque representa capacidades nativas da plataforma inteira.
- O objetivo operacional dessa tabela e substituir o catalogo builtin legado em arquivo por uma fonte unica de verdade persistida em banco.
- O builder oficial relê as tools builtin, sincroniza essa tabela e preserva o estado administrativo de ativacao por `status`.
- O runtime e o cache central leem o catalogo a partir dessa tabela, sem fallback implicito para artefatos legados em JSON.

### integrations.builtin_tool_registry

- Finalidade pratica: armazenar o catalogo global de tools builtin da plataforma como fonte unica de verdade para builder, cache, runtime e administracao.
- Chave primaria: `id`.
- Escopo: global, sem `tenant_id`.
- Colunas:
- `id`: identificador canonico da tool builtin. E a chave usada pelo runtime, pelo YAML e pela AST agentic.
- `impl`: caminho de implementacao direta da tool, quando a binding e direta e nao vem de factory.
- `factory_impl`: caminho da factory que gera tools parametrizadas, quando a binding e baseada em factory.
- `tool_name`: nome exposto da tool derivada pela factory, quando existir.
- `factory_returns`: tipo de retorno declarado pela factory. O DDL limita os valores aceitos a `list`, `single`, `tool` ou `callable`.
- `description`: descricao canonica principal da tool para uso pelo runtime.
- `tool_description`: descricao administrativa complementar, separada de `description`, para governanca e UI.
- `config`: configuracao estruturada em `jsonb`, sempre no formato de objeto.
- `category`: categoria funcional da tool para filtro e agrupamento.
- `tags`: lista de tags em `jsonb`, sempre no formato de array.
- `status`: estado operacional da tool. O contrato aceito e `active`, `disabled` ou `deprecated`.
- `discovered_from`: origem da descoberta da tool pelo builder.
- `factory_function`: nome da funcao de factory responsavel pela geracao, quando existir.
- `tool_type`: tipo de binding da tool. O DDL aceita `direct`, `factory_generated` ou `mcp`. Tools `mcp` são descobertas/persistidas pela sincronização do catálogo MCP e NÃO têm binding local de execução (são resolvidas em runtime pelo `MCPToolsResolver`).
- `decorator`: decorator encontrado pelo builder durante a descoberta da tool.
- `function_name`: nome da funcao real detectada no codigo-fonte.
- `path_verified`: indica se o path resolvido pelo builder foi validado com sucesso.
- `created_by`: responsavel pela criacao inicial da row.
- `updated_by`: responsavel pela ultima alteracao administrativa da row.
- `created_at`: criacao do registro.
- `updated_at`: ultima atualizacao do registro.
- `metadata_json`: metadados auxiliares em `jsonb` para telemetria e governanca.
- Indices e restricoes:
- PK em `id`.
- Check exigindo `id`, `description`, `tool_description` e `category` nao vazios.
- Check limitando `status` a `active`, `disabled` ou `deprecated`.
- Check limitando `factory_returns` aos valores esperados pela factory.
- Check limitando `tool_type` a `direct`, `factory_generated` ou `mcp`.
- Check garantindo `config` como objeto `jsonb`.
- Check garantindo `tags` como array `jsonb`.
- Check de binding impedindo combinacao invalida entre `impl` e `factory_impl`, isentando `tool_type='mcp'` (que nao possui binding local). Migracao idempotente: `scripts/sql/20260608_alter_builtin_tool_registry_allow_mcp_tool_type.sql` e auto-aplicacao pelo `IntegrationsSchemaBootstrapper.ensure_schema`.
- Indice `ix_builtin_tool_registry_status_category` em `status, category`.
- Indice `ix_builtin_tool_registry_updated_at` em `updated_at DESC`.
- Indice funcional `ix_builtin_tool_registry_lower_id` em `lower(id)`.
- Indice funcional `ix_builtin_tool_registry_lower_tool_name` em `lower(COALESCE(tool_name, ''))`.
- Indice funcional `ix_builtin_tool_registry_lower_category` em `lower(category)`.
- Indice GIN `ix_builtin_tool_registry_tags_gin` em `tags`.
- Semantica operacional:
- `status` e a unica fonte de verdade para habilitar e desabilitar a tool builtin.
- Rows em `disabled` nao podem ser religadas automaticamente pelo sync do builder.
- A exclusao fisica deve ficar restrita a rows obsoletas, isto e, itens que ja nao pertencem mais ao conjunto builtin descoberto pelo builder.
- O runtime deve consumir somente as rows ativas, sem fallback para o artefato legado em arquivo.

### integrations.integration_group_registry

- Finalidade pratica: agrupar operacoes HTTP, queries e procedures por dominio funcional dentro de cada tenant.
- Para que serve na pratica: dar contexto de negocio e organizacao para ativos tecnicos que, sem grupo, ficariam soltos e mais dificeis de governar, filtrar e publicar.
- Chave primaria: `group_id`.
- Colunas:
- `group_id`: identificador UUID do grupo, com geracao padrao por `gen_random_uuid()`.
- `tenant_id`: tenant do grupo.
- `group_code`: codigo estavel do grupo.
- `name`: nome amigavel do agrupamento.
- `description`: explica o que pertence ao grupo.
- `is_active`: indica se o grupo continua em uso.
- `created_by`: responsavel pela criacao.
- `updated_by`: responsavel pela ultima atualizacao.
- `created_at`: criacao do registro.
- `updated_at`: ultima atualizacao do registro.
- `metadata_json`: metadados adicionais do grupo.
- Indices e restricoes:
- PK em `group_id`.
- Unique `uq_integration_group_tenant_code` em `tenant_id, group_code`.
- Check `ck_integration_group_description_not_blank` garantindo `description` nao vazia.
- Indices `idx_integration_group_tenant_active` e `ix_integration_group_registry_tenant_active` em `tenant_id, is_active`.

### integrations.api_swagger_source_registry

- Finalidade pratica: guardar a origem Swagger/OpenAPI usada para importar ou rastrear operacoes HTTP governadas.
- Para que serve na pratica: manter o documento bruto, o hash e a trilha de importacao para que a plataforma saiba de onde uma operacao veio, se ela mudou e quando foi sincronizada pela ultima vez.
- Chave primaria: `swagger_source_id`.
- Colunas:
- `swagger_source_id`: identificador UUID da origem, com geracao padrao por `gen_random_uuid()`.
- `tenant_id`: tenant dono da origem.
- `swagger_code`: codigo estavel da origem Swagger.
- `name`: nome amigavel da origem.
- `description`: descricao da origem importada.
- `source_type`: tipo da origem, como upload, URL ou manual.
- `source_uri`: URL usada na importacao, quando houver.
- `source_file_name`: nome do arquivo, quando a origem veio de upload.
- `document_hash`: hash do documento importado.
- `openapi_version`: versao OpenAPI identificada.
- `raw_document_json`: documento bruto importado em `jsonb`.
- `import_status`: estado da importacao. O default informado e `imported`.
- `imported_by`: quem importou.
- `imported_at`: quando a importacao aconteceu.
- `last_synced_at`: ultima sincronizacao posterior, quando existir.
- `is_active`: indica se a origem continua valida.
- `metadata_json`: metadados adicionais.
- `created_at`: criacao do registro.
- `updated_at`: ultima atualizacao do registro.
- Indices e restricoes:
- PK em `swagger_source_id`.
- Unique `uq_api_swagger_source_tenant_code` em `tenant_id, swagger_code`.
- Check `ck_api_swagger_source_type` limitado a `upload`, `url` ou `manual`.
- Check `ck_api_swagger_description_not_blank` garantindo `description` nao vazia.
- Indices `idx_api_swagger_source_tenant_active` e `ix_api_swagger_source_registry_tenant_active` em `tenant_id, is_active`.

### integrations.api_auth_profile_registry

- Finalidade pratica: centralizar perfis de autenticacao reutilizaveis para operacoes HTTP de um tenant.
- Para que serve na pratica: evitar repetir segredo, cabecalho, URL de token e regra de extracao em cada endpoint cadastrado. A operacao aponta para um perfil pronto e o runtime reaproveita esse contrato.
- Chave primaria: `auth_profile_id`.
- Colunas:
- `auth_profile_id`: identificador UUID do perfil, com geracao padrao por `gen_random_uuid()`.
- `tenant_id`: tenant dono do perfil.
- `auth_profile_code`: codigo estavel do perfil de autenticacao.
- `name`: nome amigavel do perfil.
- `description`: descricao funcional e operacional do perfil.
- `auth_type`: tipo de autenticacao. O DDL aceita `none`, `bearer_static`, `api_key_header`, `api_key_query`, `basic`, `oauth_client_credentials` e `custom_token_endpoint`.
- `auth_method`: metodo HTTP usado na autenticacao. O DDL aceita `GET`, `POST`, `PUT` e `PATCH`, com default `POST`.
- `auth_url`: URL de autenticacao.
- `auth_headers_json`: cabecalhos usados na autenticacao.
- `auth_body_json`: corpo usado na autenticacao.
- `token_key`: chave da resposta onde o token sera extraido. O default informado e `access_token`.
- `token_prefix`: prefixo do token enviado na chamada autenticada. O default informado e `Bearer`.
- `secret_ref`: referencia para segredo centralizado.
- `credentials_json_encrypted`: credencial criptografada, se o desenho optar por persistir no banco.
- `cache_ttl_seconds`: tempo esperado de cache do token.
- `is_active`: indica se o perfil esta ativo.
- `created_by`: responsavel pela criacao.
- `updated_by`: responsavel pela ultima atualizacao.
- `created_at`: criacao do registro.
- `updated_at`: ultima atualizacao.
- `metadata_json`: metadados adicionais.
- Indices e restricoes:
- PK em `auth_profile_id`.
- Unique `uq_api_auth_profile_tenant_code` em `tenant_id, auth_profile_code`.
- Check `ck_api_auth_profile_type` limitando os tipos aceitos em `auth_type`.
- Check `ck_api_auth_profile_method` limitando `auth_method`.
- Check `ck_api_auth_profile_description_not_blank` garantindo `description` nao vazia.
- Check `ck_api_auth_profile_secret_source` exigindo `secret_ref` ou `credentials_json_encrypted` quando `auth_type` nao for `none`.
- Indices `idx_api_auth_profile_tenant_active` e `ix_api_auth_profile_registry_tenant_active` em `tenant_id, is_active`.

### integrations.sql_connection_registry

- Finalidade pratica: centralizar conexoes SQL aprovadas para uso por queries e procedures governadas.
- Para que serve na pratica: separar a definicao da conexao da definicao da query. Assim, varias queries e procedures podem apontar para a mesma conexao aprovada sem duplicar segredo nem string de conexao.
- Chave primaria: `sql_connection_id`.
- Colunas:
- `sql_connection_id`: identificador UUID da conexao, com geracao padrao por `gen_random_uuid()`.
- `tenant_id`: tenant dono da conexao.
- `connection_code`: codigo estavel da conexao.
- `name`: nome amigavel da conexao.
- `description`: descricao funcional e operacional da conexao.
- `db_engine`: tipo de banco. O DDL aceita `postgresql`, `mysql`, `mssql`, `oracle`, `sqlite` e `other`.
- `connection_mode`: define se a conexao vem de `secret_ref` ou de string criptografada inline. O DDL aceita `secret_ref` e `encrypted_inline`, com default `secret_ref`.
- `secret_ref`: referencia para segredo centralizado.
- `connection_string_encrypted`: string de conexao criptografada, se essa estrategia for usada.
- `default_database_name`: banco padrao da conexao.
- `default_schema_name`: schema padrao da conexao.
- `validation_query`: query simples de validacao.
- `read_only`: indica se a conexao deve ser tratada como somente leitura.
- `is_active`: indica se a conexao esta ativa.
- `created_by`: responsavel pela criacao.
- `updated_by`: responsavel pela ultima atualizacao.
- `created_at`: criacao do registro.
- `updated_at`: ultima atualizacao.
- `metadata_json`: metadados adicionais.
- Indices e restricoes:
- PK em `sql_connection_id`.
- Unique `uq_sql_connection_tenant_code` em `tenant_id, connection_code`.
- Check `ck_sql_connection_engine` limitando os engines aceitos em `db_engine`.
- Check `ck_sql_connection_mode` limitando `connection_mode`.
- Check `ck_sql_connection_secret_source` exigindo `secret_ref` ou `connection_string_encrypted`, de acordo com o modo escolhido.
- Check `ck_sql_connection_description_not_blank` garantindo `description` nao vazia.
- Indices `idx_sql_connection_tenant_active` e `ix_sql_connection_registry_tenant_active` em `tenant_id, is_active`.

### integrations.api_operation_registry

- Finalidade pratica: cadastrar cada operacao HTTP reutilizavel, com opcao de publicacao para agentes.
- Para que serve na pratica: transformar uma chamada externa em um ativo governado. Em vez de a URL, o metodo e os parametros ficarem dispersos em YAML ou codigo solto, eles ficam registrados de forma padronizada e auditavel.
- Chave primaria: `api_operation_id`.
- Colunas:
- `api_operation_id`: identificador UUID da operacao, com geracao padrao por `gen_random_uuid()`.
- `tenant_id`: tenant dono da operacao.
- `operation_code`: codigo estavel da operacao.
- `name`: nome amigavel da operacao.
- `description`: descricao semantica da operacao. Esse campo e central para a descricao funcional da tool derivada e o DDL exige pelo menos 10 caracteres uteis.
- `operation_type`: tipo funcional da operacao, com default `generic`.
- `group_id`: referencia ao grupo funcional.
- `swagger_source_id`: referencia a origem Swagger, quando a operacao veio de importacao.
- `swagger_operation_id`: identificador da operacao dentro do Swagger.
- `protocol_type`: protocolo da integracao. O DDL aceita `rest_json`, `rest_form`, `soap`, `graphql` e `webhook`, com default `rest_json`.
- `http_method`: metodo HTTP da chamada.
- `base_url`: URL base da API.
- `path_template`: caminho da operacao, possivelmente com parametros.
- `auth_profile_id`: perfil de autenticacao reutilizado pela operacao.
- `path_params_schema_json`: estrutura dos parametros de rota.
- `query_params_schema_json`: estrutura dos parametros de query string.
- `header_template_json`: cabecalhos adicionais da operacao.
- `body_template_json`: corpo padrao ou template da operacao.
- `response_mapping_json`: regras de interpretacao da resposta.
- `tags_json`: tags funcionais da operacao.
- `timeout_seconds`: timeout da execucao.
- `publish_to_agents`: indica se a operacao esta pronta para virar tool.
- `is_active`: indica se a operacao esta ativa.
- `created_by`: responsavel pela criacao.
- `updated_by`: responsavel pela ultima atualizacao.
- `created_at`: criacao do registro.
- `updated_at`: ultima atualizacao.
- `metadata_json`: metadados adicionais.
- Indices e restricoes:
- PK em `api_operation_id`.
- Unique `uq_api_operation_tenant_code` em `tenant_id, operation_code`.
- FK `group_id` para `integrations.integration_group_registry.group_id`.
- FK `swagger_source_id` para `integrations.api_swagger_source_registry.swagger_source_id`.
- FK `auth_profile_id` para `integrations.api_auth_profile_registry.auth_profile_id`.
- Check `ck_api_operation_method` limitando `http_method` a `GET`, `POST`, `PUT`, `PATCH` ou `DELETE`.
- Check `ck_api_operation_protocol` limitando `protocol_type`.
- Check `ck_api_operation_description_not_blank` exigindo `description` minimamente significativa.
- Check `ck_api_operation_tags_is_array` garantindo que `tags_json` seja array.
- Check `ck_api_operation_timeout_positive` garantindo `timeout_seconds > 0`.
- Indices `idx_api_operation_tenant_active` e `ix_api_operation_registry_tenant_active` em `tenant_id, is_active`.
- Indices `idx_api_operation_publishable` e `ix_api_operation_registry_publish_active` em `tenant_id, publish_to_agents, is_active`.
- Indices `idx_api_operation_group` e `ix_api_operation_registry_group` em `group_id`.
- Indices `idx_api_operation_swagger_source` e `ix_api_operation_registry_swagger_source` em `swagger_source_id`.
- Observacao importante do runtime atual: a resolucao dinamica mais evidenciada em codigo para `dyn_api<...>` busca `operation_code` por `tenant_id`, exige `publish_to_agents=true`, `is_active=true` e hoje aceita apenas `protocol_type='rest_json'` na trilha governada lida no resolver.

### integrations.sql_query_registry

- Finalidade pratica: cadastrar queries SQL reutilizaveis com governanca propria por tenant.
- Para que serve na pratica: padronizar query, parametros, timeout e limite de linhas, permitindo que a mesma consulta seja reutilizada com seguranca por telas administrativas, servicos internos e tools dinamicas.
- Chave primaria: `sql_query_id`.
- Colunas:
- `sql_query_id`: identificador UUID da query, com geracao padrao por `gen_random_uuid()`.
- `tenant_id`: tenant dono da query.
- `query_code`: codigo estavel da query.
- `name`: nome amigavel da query.
- `description`: descricao semantica da query. Esse campo serve de base para descricao funcional e o DDL exige pelo menos 10 caracteres uteis.
- `group_id`: referencia ao grupo funcional.
- `connection_id`: conexao SQL usada pela query.
- `query_kind`: tipo funcional da query. O DDL aceita `select`, `report`, `lookup`, `analytics` e `other`, com default `select`.
- `sql_text`: SQL efetivamente executado.
- `parameter_schema_json`: estrutura dos parametros aceitos.
- `tags_json`: tags funcionais da query.
- `result_format`: formato esperado de retorno, com default `json`.
- `max_rows`: limite de linhas retornadas, com default `200`.
- `timeout_seconds`: timeout da execucao.
- `publish_to_agents`: indica se a query pode virar tool.
- `is_active`: indica se a query esta ativa.
- `created_by`: responsavel pela criacao.
- `updated_by`: responsavel pela ultima atualizacao.
- `created_at`: criacao do registro.
- `updated_at`: ultima atualizacao.
- `metadata_json`: metadados adicionais.
- Indices e restricoes:
- PK em `sql_query_id`.
- Unique `uq_sql_query_tenant_code` em `tenant_id, query_code`.
- FK `group_id` para `integrations.integration_group_registry.group_id`.
- FK `connection_id` para `integrations.sql_connection_registry.sql_connection_id`.
- Check `ck_sql_query_kind` limitando `query_kind`.
- Check `ck_sql_query_description_not_blank` exigindo `description` minimamente significativa.
- Check `ck_sql_query_tags_is_array` garantindo que `tags_json` seja array.
- Check `ck_sql_query_max_rows_positive` garantindo `max_rows > 0`.
- Check `ck_sql_query_timeout_positive` garantindo `timeout_seconds > 0`.
- Indices `idx_sql_query_tenant_active` e `ix_sql_query_registry_tenant_active` em `tenant_id, is_active`.
- Indices `idx_sql_query_publishable` e `ix_sql_query_registry_publish_active` em `tenant_id, publish_to_agents, is_active`.
- Indices `idx_sql_query_connection` e `ix_sql_query_registry_connection` em `connection_id`.
- Indices `idx_sql_query_group` e `ix_sql_query_registry_group` em `group_id`.
- Observacao importante do runtime atual: a trilha governada de `dyn_sql<...>` resolve `query_code` por `tenant_id`, exige `publish_to_agents=true`, `is_active=true` e carrega a conexao relacionada em `integrations.sql_connection_registry` antes de montar a tool executavel.

### integrations.sql_procedure_registry

- Finalidade pratica: cadastrar stored procedures reutilizaveis com governanca propria por tenant.
- Para que serve na pratica: registrar chamadas aprovadas de procedure com sua conexao, parametros e timeout, evitando execucao ad hoc e deixando claro quais procedimentos foram homologados para uso operacional.
- Chave primaria: `sql_procedure_id`.
- Colunas:
- `sql_procedure_id`: identificador UUID da procedure, com geracao padrao por `gen_random_uuid()`.
- `tenant_id`: tenant dono da procedure.
- `procedure_code`: codigo estavel da procedure.
- `name`: nome amigavel da procedure.
- `description`: descricao semantica da procedure. Esse campo tambem e base funcional de catalogacao e o DDL exige pelo menos 10 caracteres uteis.
- `group_id`: referencia ao grupo funcional.
- `connection_id`: conexao SQL usada pela procedure.
- `procedure_schema_name`: schema do banco onde a procedure vive, quando necessario.
- `procedure_name`: nome da procedure no banco.
- `call_template`: template opcional da chamada.
- `parameter_schema_json`: estrutura dos parametros aceitos.
- `tags_json`: tags funcionais da procedure.
- `fetch_mode`: modo de leitura do retorno, como `all`, `one` ou `none`.
- `include_columns`: indica se o retorno deve trazer nomes de colunas.
- `result_format`: formato esperado do retorno, com default `json`.
- `timeout_seconds`: timeout da execucao.
- `publish_to_agents`: indica se a procedure pode virar tool.
- `is_active`: indica se a procedure esta ativa.
- `created_by`: responsavel pela criacao.
- `updated_by`: responsavel pela ultima atualizacao.
- `created_at`: criacao do registro.
- `updated_at`: ultima atualizacao.
- `metadata_json`: metadados adicionais.
- Indices e restricoes:
- PK em `sql_procedure_id`.
- Unique `uq_sql_procedure_tenant_code` em `tenant_id, procedure_code`.
- FK `group_id` para `integrations.integration_group_registry.group_id`.
- FK `connection_id` para `integrations.sql_connection_registry.sql_connection_id`.
- Check `ck_sql_procedure_fetch_mode` limitando `fetch_mode`.
- Check `ck_sql_procedure_description_not_blank` exigindo `description` minimamente significativa.
- Check `ck_sql_procedure_tags_is_array` garantindo que `tags_json` seja array.
- Check `ck_sql_procedure_timeout_positive` garantindo `timeout_seconds > 0`.
- Indices `idx_sql_procedure_tenant_active` e `ix_sql_procedure_registry_tenant_active` em `tenant_id, is_active`.
- Indices `idx_sql_procedure_publishable` e `ix_sql_procedure_registry_publish_active` em `tenant_id, publish_to_agents, is_active`.
- Indices `idx_sql_procedure_connection` e `ix_sql_procedure_registry_connection` em `connection_id`.
- Indices `idx_sql_procedure_group` e `ix_sql_procedure_registry_group` em `group_id`.
- Leitura pratica importante: essa tabela ja faz parte do cadastro funcional e administrativo do modulo de integracoes. No ecossistema de tools, `proc_sql<...>` ja existe como familia parametrizada da plataforma, e `procedure_code` e o identificador natural desse cadastro governado.

### Relacoes principais da solucao

- `integrations.builtin_tool_registry` e global e nao depende de nenhuma tabela multi-tenant do schema `integrations`.
- `integrations.api_operation_registry.group_id` depende de `integrations.integration_group_registry.group_id`.
- `integrations.api_operation_registry.swagger_source_id` depende de `integrations.api_swagger_source_registry.swagger_source_id`.
- `integrations.api_operation_registry.auth_profile_id` depende de `integrations.api_auth_profile_registry.auth_profile_id`.
- `integrations.sql_query_registry.group_id` depende de `integrations.integration_group_registry.group_id`.
- `integrations.sql_query_registry.connection_id` depende de `integrations.sql_connection_registry.sql_connection_id`.
- `integrations.sql_procedure_registry.group_id` depende de `integrations.integration_group_registry.group_id`.
- `integrations.sql_procedure_registry.connection_id` depende de `integrations.sql_connection_registry.sql_connection_id`.

### Leitura pratica da solucao

- Quando o time da plataforma precisa reconstruir o catalogo builtin, ele consulta `integrations.builtin_tool_registry` como fonte persistida de verdade.
- Quando a operacao administrativa precisa ativar, desativar, classificar ou auditar tools builtin, ela trabalha sobre `integrations.builtin_tool_registry` e usa `status` como contrato de ativacao.
- Quando o time tecnico cadastra um acesso SQL compartilhado, ele primeiro registra a conexao em `integrations.sql_connection_registry`.
- Quando o time tecnico cadastra autenticacao compartilhada de API, ele registra o perfil em `integrations.api_auth_profile_registry`.
- Quando a equipe funcional organiza o catalogo por assunto, ela usa `integrations.integration_group_registry` para agrupar os ativos daquele tenant.
- Quando uma importacao OpenAPI precisa de rastreabilidade, o documento de origem fica em `integrations.api_swagger_source_registry`.
- Quando a equipe funcional cadastra um endpoint governado, ela usa `integrations.api_operation_registry` e pode apontar tanto para um grupo quanto para um perfil de autenticacao reaproveitavel.
- Quando a equipe funcional cadastra uma query governada, ela usa `integrations.sql_query_registry` e aponta para uma conexao SQL aprovada.
- Quando a equipe funcional cadastra uma stored procedure homologada, ela usa `integrations.sql_procedure_registry` e aponta para uma conexao SQL aprovada.
- Quando a solucao publica isso para agentes, a leitura mais evidenciada em codigo hoje e: `dyn_api<...>` resolve `operation_code` em `integrations.api_operation_registry` e `dyn_sql<...>` resolve `query_code` em `integrations.sql_query_registry`, sempre no contexto de `user_session.tenant_id`.
- Em linguagem simples: o schema `integrations` funciona como a prateleira oficial dos conectores aprovados. Primeiro a plataforma descobre a capacidade, depois confere se ela esta ativa, e so entao monta a execucao. Isso reduz improviso, evita configuracao solta e deixa o caminho de auditoria mais claro.
- Check `user_accounts_status_check` limitando `account_status` aos estados suportados.

### user_auth_identities

- Finalidade prática: guardar identidades externas de autenticação, como Google.
- Chave primária: `user_auth_identity_id`.
- Colunas:
- `user_auth_identity_id`: identificador UUID da identidade.
- `user_account_id`: conta pessoal dona da identidade.
- `provider_type`: provedor, como `google`.
- `provider_subject`: identificador estável do provedor, como o `sub` do Google.
- `provider_email`: e-mail devolvido pelo provedor.
- `provider_email_verified`: indica se o provedor confirmou o e-mail.
- `last_login_at`: último login usando essa identidade.
- `metadata_json`: metadados adicionais em `jsonb`.
- `created_at`: criação do vínculo.
- `updated_at`: última atualização do vínculo.
- Índices e restrições:
- PK em `user_auth_identity_id`.
- FK `user_account_id` para `user_accounts.user_account_id` com `ON DELETE CASCADE`.
- Unique em `provider_type, provider_subject`.
- Check `user_auth_identities_provider_check` restringindo `provider_type` ao valor `google` no DDL atual.

### user_password_credentials

- Finalidade prática: guardar a credencial local de usuário e senha da conta pessoal.
- Chave primária: `user_account_id`.
- Colunas:
- `user_account_id`: conta pessoal dona da credencial.
- `password_hash`: hash da senha.
- `password_algorithm`: algoritmo de hash, preferencialmente `argon2id`.
- `password_set_at`: momento em que a senha foi definida.
- `password_changed_at`: última troca de senha.
- `must_change_password`: força troca de senha no próximo login.
- `failed_login_attempts`: contador de falhas consecutivas.
- `locked_until`: bloqueio temporário por excesso de falhas.
- `last_login_at`: último login com senha.
- `metadata_json`: metadados adicionais em `jsonb`.
- `created_at`: criação do registro.
- `updated_at`: última atualização do registro.
- Índices e restrições:
- PK em `user_account_id`.
- FK `user_account_id` para `user_accounts.user_account_id` com `ON DELETE CASCADE`.
- Check para limitar `password_algorithm` ao algoritmo suportado.
- Check para `failed_login_attempts >= 0`.

### user_account_yaml

- Finalidade prática: guardar o YAML único da conta pessoal do usuário.
- Chave primária: `user_account_yaml_id`.
- Colunas:
- `user_account_yaml_id`: identificador UUID do vínculo.
- `user_account_id`: conta pessoal dona do YAML.
- `yaml_path`: caminho do YAML pessoal.
- `status`: estado do vínculo, com default `active`.
- `execution_mode`: modo de execução associado.
- `descricao`: descrição funcional do vínculo.
- `metadata_json`: metadados adicionais em `jsonb`.
- `created_at`: criação do vínculo.
- `updated_at`: última atualização do vínculo.
- Índices e restrições:
- PK em `user_account_yaml_id`.
- FK `user_account_id` para `user_accounts.user_account_id` com `ON DELETE CASCADE`.
- Unique `user_account_yaml_user_account_id_key` em `user_account_id`.
- Unique `user_account_yaml_path_key` em `user_account_id, yaml_path`.

#### Como usar no contexto pessoal

- esta tabela resolve o contexto pessoal, quando o usuário opera fora de uma organização.
- a conta pessoal deve ter no máximo um YAML ativo.
- se não houver YAML pessoal configurado, o sistema deve falhar de forma clara, sem fallback automático para tenant.

### Leitura prática de tenant_users no modelo final

- Finalidade prática: manter `tenant_users` como membership organizacional, não como tabela de autenticação.
- Leitura prática do modelo:
- uma mesma conta pessoal pode participar de zero, uma ou várias organizações.
- o contexto organizacional só existe quando houver linha ativa em `tenant_users`.
- as permissões organizacionais passam a depender do `role` e não da simples existência do vínculo.
- o login continua acontecendo na conta pessoal; `tenant_users` apenas controla escopo organizacional.
- o YAML organizacional não pertence à conta pessoal diretamente; ele pertence ao vínculo do usuário com aquela organização.

#### O que muda na prática em tenant_users

- `tenant_users` deixa de armazenar identidade primária como e-mail e foto.
- a identidade primária passa a ser `user_account_id`.
- isso evita duplicar credencial por tenant e permite que o mesmo usuário entre no contexto pessoal ou em diferentes organizações sem criar contas separadas.

### Leitura prática de tenants no modelo final

- Finalidade prática: continuar usando `tenants` como entidade de organização.
- Leitura prática do modelo:
- `tenants` passa a ser o equivalente funcional da organização no padrão de mercado.
- o tenant concentra billing organizacional, membership e recursos compartilhados.
- o usuário não é obrigado a pertencer a um tenant para usar o sistema; nesse caso ele opera apenas com a conta pessoal.

#### O que muda na prática em tenants

- `owner_user_account_id` permite identificar o dono principal da organização sem depender apenas de `default_user_email`.
- `billing_email` separa contato financeiro do restante dos dados cadastrais do tenant.
- o tenant continua sendo a fonte de verdade para cobrança e governança quando o usuário escolhe operar no contexto organizacional.

### user_account_payment_cards

- Finalidade prática: guardar os cartões de crédito da conta pessoal.
- Chave primária: `user_account_payment_card_id`.
- Colunas:
- `user_account_payment_card_id`: identificador UUID do cartão.
- `user_account_id`: conta pessoal dona do cartão.
- `gateway_provider`: provedor de pagamento ou tokenização.
- `gateway_customer_id`: identificador do cliente no gateway.
- `gateway_payment_method_id`: identificador do método no gateway.
- `card_token`: token do cartão, quando houver.
- `card_fingerprint`: impressão digital segura para deduplicação.
- `card_brand`: bandeira do cartão.
- `card_holder_name`: nome impresso no cartão.
- `card_last4`: últimos 4 dígitos.
- `card_bin`: BIN ou IIN permitido pelo fluxo do gateway.
- `exp_month`: mês de expiração.
- `exp_year`: ano de expiração.
- `billing_address_line1`: primeira linha do endereço de cobrança.
- `billing_address_line2`: complemento do endereço de cobrança.
- `billing_address_city`: cidade do endereço de cobrança.
- `billing_address_state`: estado do endereço de cobrança.
- `billing_address_postal_code`: CEP ou código postal.
- `billing_address_country_code`: país no padrão ISO de duas letras.
- `status`: estado do cartão, como `active`, `inactive` ou `expired`.
- `is_default`: indica o cartão padrão da conta pessoal.
- `is_verified`: indica se o cartão foi verificado pelo processador.
- `last_used_at`: último uso conhecido.
- `metadata_json`: metadados adicionais em `jsonb`.
- `created_at`: criação do registro.
- `updated_at`: última atualização do registro.
- Índices e restrições:
- PK em `user_account_payment_card_id`.
- FK `user_account_id` para `user_accounts.user_account_id` com `ON DELETE CASCADE`.
- Unique em `gateway_provider, gateway_payment_method_id`.
- Unique parcial por conta para garantir um único cartão default ativo.
- Unique parcial por `user_account_id, gateway_provider, card_fingerprint` quando houver fingerprint.
- Check `user_account_payment_cards_country_check` validando `billing_address_country_code` em duas letras maiúsculas quando informado.
- Check `user_account_payment_cards_exp_month_check` validando mês entre 1 e 12.
- Check `user_account_payment_cards_exp_year_check` validando ano maior ou igual a 2000.
- Check `user_account_payment_cards_last4_check` validando exatamente 4 dígitos.

#### Como usar no contexto organizacional

- esta tabela deve ser usada quando o usuário estiver operando no contexto pessoal.
- o cartão default pessoal pertence à conta e não depende de membership em tenant.
- se a conta pessoal nunca entrar em uma organização, esta tabela sozinha resolve o billing do usuário.
- PAN completo e CVV não devem ser persistidos aqui; apenas token, fingerprint e dados mascarados.

### tenant_payment_cards

- Finalidade prática: guardar os cartões de cobrança da organização.
- Chave primária: `tenant_payment_card_id`.
- Colunas:
- `tenant_payment_card_id`: identificador UUID do cartão organizacional.
- `tenant_id`: organização dona do cartão.
- `gateway_provider`: provedor de pagamento ou tokenização.
- `gateway_customer_id`: identificador da organização no gateway.
- `gateway_payment_method_id`: identificador do método no gateway.
- `card_token`: token do cartão, quando houver.
- `card_fingerprint`: impressão digital segura para deduplicação.
- `card_brand`: bandeira do cartão.
- `card_holder_name`: nome impresso no cartão.
- `card_last4`: últimos 4 dígitos.
- `card_bin`: BIN ou IIN permitido pelo fluxo do gateway.
- `exp_month`: mês de expiração.
- `exp_year`: ano de expiração.
- `billing_address_line1`: primeira linha do endereço de cobrança.
- `billing_address_line2`: complemento do endereço de cobrança.
- `billing_address_city`: cidade do endereço de cobrança.
- `billing_address_state`: estado do endereço de cobrança.
- `billing_address_postal_code`: CEP ou código postal.
- `billing_address_country_code`: país no padrão ISO de duas letras.
- `status`: estado do cartão, como `active`, `inactive` ou `expired`.
- `is_default`: indica o cartão padrão da organização.
- `is_verified`: indica se o cartão foi verificado pelo processador.
- `last_used_at`: último uso conhecido.
- `metadata_json`: metadados adicionais em `jsonb`.
- `created_at`: criação do registro.
- `updated_at`: última atualização do registro.
- Índices e restrições:
- PK em `tenant_payment_card_id`.
- FK `tenant_id` para `tenants.tenant_id` com `ON DELETE CASCADE`.
- Unique em `gateway_provider, gateway_payment_method_id`.
- Unique parcial por `tenant_id` para garantir um único cartão default ativo da organização.
- Unique parcial por `tenant_id, gateway_provider, card_fingerprint` quando houver fingerprint.
- Check `tenant_payment_cards_country_check` validando `billing_address_country_code` em duas letras maiúsculas quando informado.
- Check `tenant_payment_cards_exp_month_check` validando mês entre 1 e 12.
- Check `tenant_payment_cards_exp_year_check` validando ano maior ou igual a 2000.
- Check `tenant_payment_cards_last4_check` validando exatamente 4 dígitos.

#### Como usar na prática

- esta tabela deve ser usada quando o usuário estiver operando no contexto da organização.
- o pagamento organizacional não deve depender do cartão pessoal do membro.
- permissões para cadastrar, trocar ou remover cartão organizacional devem ser restritas, na prática, a papéis como `owner`, `admin` ou `billing_manager`.

### Fluxo operacional esperado

- Cadastro pessoal: cria `user_accounts` e, dependendo do método de login, grava em `user_auth_identities` ou `user_password_credentials`.
- Configuração pessoal: grava o YAML único da conta em `user_account_yaml`.
- Criação de organização: cria `tenants`, vincula o criador em `tenant_users` com `role='owner'` e registra `owner_user_account_id`.
- Configuração organizacional: grava o YAML único do vínculo em `tenant_user_yaml`.
- Acesso pessoal: usa apenas `user_accounts` e `user_account_payment_cards`.
- Sessão autenticada padrão: deve nascer com o `yaml_path` pessoal de `user_account_yaml`; a lista de organizações do usuário é contexto separado e explícito.
- Acesso organizacional: exige membership em `tenant_users`; permissões vêm do `role` e a cobrança sai de `tenant_payment_cards`.
- Usuário sem organização: continua funcional, sem obrigatoriedade de pertencer a um tenant.

### Regra obrigatória de resolução de YAML

- Contexto pessoal: resolver exclusivamente por `user_account_id` em `user_account_yaml`.
- Contexto organizacional: resolver exclusivamente por `tenant_user_id` ou por `tenant_id + user_account_id` em `tenant_user_yaml`.
- É proibido escolher YAML por `user_email` solto ou por ordenação implícita de `updated_at` entre vários vínculos.
- É proibido fallback automático entre YAML pessoal e YAML organizacional sem requisito explícito.

## Domínio Memória do Usuário

### user_memory_interactions

- Finalidade prática: guardar interações relevantes para memória de longo prazo do usuário.
- Chave primária: `id`.
- Colunas:
- `id`: identificador bigserial do registro.
- `user_email`: e-mail do usuário.
- `correlation_id`: correlação da execução.
- `session_id`: identificador da sessão.
- `question`: pergunta registrada.
- `answer`: resposta registrada.
- `context_type`: tipo do contexto, com default `unknown`.
- `metadata`: metadados da memória em `jsonb`.
- `recorded_at`: instante lógico em que a memória foi gravada.
- `created_at`: criação do registro.
- Índices e restrições:
- PK em `id`.
- Índice `idx_memory_session` em `user_email, session_id`.
- Índice `idx_memory_user_ts` em `user_email, recorded_at` descendente.

### user_memory_session_summaries

- Finalidade prática: guardar um resumo consolidado por sessão de usuário.
- Chave primária: `id`.
- Colunas:
- `id`: identificador bigserial do resumo.
- `user_email`: e-mail do usuário.
- `session_id`: identificador da sessão.
- `summary`: resumo textual da sessão.
- `metadata`: metadados do resumo em `jsonb`.
- `created_at`: criação do registro.
- `updated_at`: última atualização do resumo.
- Índices e restrições:
- PK em `id`.
- Unique `user_memory_session_summaries_user_email_session_id_key` em `user_email, session_id`.
- Índice `idx_memory_session_user` em `user_email`.

## Leitura Operacional do Schema

- Para analisar integridade do acervo ativo, primeiro descubra em `ingestion_datasets` qual `active_generation_id` e qual `physical_bm25_target` estão valendo para o par `tenant_code + vectorstore_id`. Depois leia `bm25_indexes` por `bm25_target_id`, confirme em `ingestion_document_manifest` qual `active_document_version_id` está oficial para cada fonte e só então compare `ingestion_document_pages`, `ingestion_document_chunks`, `ingestion_document_images` e `ingestion_document_versions` no mesmo acervo lógico.
- Para investigar divergência de dataset, trate `ingestion_runs` e `ingestion_run_documents` como trilha histórica, não como fonte primária do estado vivo do acervo.
- Para localizar um documento ingerido, comece por `ingestion_document_manifest`.
- Para localizar um documento externo do Confluence, consulte `source_system` junto com `external_document_id`.
- Para auditoria e filtros SQL de autorização, priorize `is_restricted`, `allows_anonymous`, `permitted_groups` e `authorization_checked_at` em `ingestion_document_manifest`.
- Para investigar a execução de uma ingestão, use `ingestion_runs` e depois `ingestion_run_documents`.
- Para abrir a trilha completa de uma interação, use `interaction_runs` e depois `interaction_run_events`.
- Para investigar Execução Agentic em Background, comece por `agent_background.background_execution_requests`, depois siga para `scheduler.scheduled_jobs` e `scheduler.job_executions`, e só então leia `agent_background.background_execution_runs` e `agent_background.background_execution_events` como projeção compatível do slice.
- Para investigar uma aprovação humana assíncrona ligada a run background, priorize `agent_background.agent_hil_approval_requests`; ali estão o pedido, o run, o canal, o prazo, o status, o token em hash e a decisão final aceita pelo sistema.
- Para validar conta pessoal e autenticação, comece por `user_accounts`, `user_auth_identities` e `user_password_credentials`.
- Para validar YAML pessoal, use `user_account_yaml`.
- Para validar configuração organizacional, comece por `tenants`, `tenant_access_keys`, `tenant_channels`, `tenant_security_keys` e `tenant_secrets`.
- Para entender o membership, o YAML do usuário e a classificação funcional dos projetos dentro do tenant, use `tenant_users`, `tenant_user_yaml`, `system_domains`, `tenant_user_projects` e `tenant_user_project_details`.
- Para cobrança, separe sempre pagamento pessoal em `user_account_payment_cards` e pagamento organizacional em `tenant_payment_cards`.
- Para recuperar memória conversacional consolidada, use `user_memory_interactions` e `user_memory_session_summaries`.

## Leituras relacionadas

- [README-CONCEITUAL-ARQUITETURA-STACK-PROJETO.md](../conceitual/README-CONCEITUAL-ARQUITETURA-STACK-PROJETO.md): mostra como esse estado persistido sustenta API, worker e scheduler.
- [README-INGESTAO-INDICE.md](./README-INGESTAO-INDICE.md): aprofunda o lado operacional do acervo que este schema materializa.
- [README-CONCEITUAL-RAG-PIPELINE-COMPLETO.md](../conceitual/README-CONCEITUAL-RAG-PIPELINE-COMPLETO.md): explica como o acervo persistido vira retrieval e resposta.
- [README-CONCEITUAL-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md](../conceitual/README-CONCEITUAL-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md): detalha a superfície que consome tabelas de identidade.
- [README-CONCEITUAL-DYNAMIC-API-TOOLS.md](../conceitual/README-CONCEITUAL-DYNAMIC-API-TOOLS.md): explica a governança funcional das tabelas do schema `integrations`.

## Troubleshooting

### O documento existe no histórico do run, mas não aparece no acervo vivo

Causa provável: a investigação começou em `ingestion_runs` ou
`ingestion_run_documents`, que são trilha histórica, e não no conjunto
vivo do dataset.

Como confirmar: volte para `ingestion_datasets`,
`ingestion_dataset_generations`, `ingestion_document_manifest` e a versão
ativa do documento antes de concluir que o acervo foi publicado.

### Há colisão aparente entre documentos com o mesmo conteúdo

Causa provável: leitura confundindo `canonical_source_key` com
`document_hash`.

Como confirmar: trate `canonical_source_key` como identidade da fonte e
`document_hash` como identidade da edição de conteúdo.

### O troubleshooting de HIL ou autenticação parece incompleto no banco

Causa provável: a consulta foi feita só em tabelas de sessão ou só em
logs, sem juntar a trilha persistida principal.

Como confirmar: para HIL em execução background, comece em
`agent_background.background_execution_runs`, depois
`agent_background.agent_hil_approval_requests` e
`agent_background.background_execution_events`; para login e sessão,
comece em `user_accounts`, `user_auth_identities` e nas tabelas
correlatas do fluxo web.

## Checklist de entendimento

- Entendi a diferença entre dataset vivo e histórico operacional.
- Entendi a diferença entre identidade da fonte e hash de conteúdo.
- Entendi por que o schema conecta ingestão, autenticação, HIL e integrações.
- Entendi como o schema `agent_background` separa pedido, agenda, run, eventos e HIL.
- Entendi por onde começar uma investigação sem confundir tabela histórica com fonte de verdade.

## Schema Implementado de Integrações Governadas

## Schema do Banco Demo de Varejo

### Objetivo desta seção

- Esta seção registra o DDL de referência do banco demo de varejo para consulta rápida em tarefas de integração, NL2SQL, SQL dinâmico, UCP e telas administrativas.
- A fonte de conexão deste banco demo deve ser consultada no `.env` pelas variáveis `DATABASE_VAREJO_DSN` e `DATABASE_VAREJO_SCHEMA`.
- O DDL abaixo reflete o schema lógico recebido para o ambiente demo e deve ser tratado como material de referência operacional.

### Como ler na prática

- O schema funcional usado no DDL é `pdv`.
- O domínio comercial principal aparece em `pdv.vendas`, que concentra venda, pagamento, entrega e metadados UCP ligados ao checkout.
- O domínio de catálogo aparece em `pdv.produtos`, `pdv.categorias`, `pdv.subcategorias`, `pdv.marcas` e `pdv.cores`.
- O domínio de cliente aparece em `pdv.clientes`, com apoio de `pdv.cidade`, `pdv.genero` e `pdv.tipo_endereco`.
- O domínio operacional de loja e entrega aparece em `pdv.lojas`, `pdv.tipos_entrega` e `pdv.meios_pagamento`.
- O DDL recebido formaliza apenas a FK de `pdv.vendas.checkout_id` para `pdv.checkout_sessions(checkout_id)`. As demais relações de negócio aparecem pelo padrão dos campos `id_*`, mesmo quando a constraint não está declarada no script.

### Estrutura lógica de referência

O schema demo de varejo está organizado em cinco blocos funcionais:

1. catálogo, com `categorias`, `subcategorias`, `marcas`, `cores` e `produtos`;
2. cliente, com `clientes`, `cidade`, `genero` e `tipo_endereco`;
3. operação comercial, com `lojas`, `tipos_entrega` e `meios_pagamento`;
4. checkout UCP, com `checkout_sessions` e seus campos JSON de capacidade, itens, totais, pagamento, links, fulfillment e sinais de risco;
5. vendas, com a tabela `vendas` concentrando compra, pagamento, entrega, metadados UCP e a FK explícita para `checkout_sessions`.

Os índices descritos no DDL reforçam principalmente lookup por status e data em `checkout_sessions` e consultas operacionais por checkout, data de venda, cliente, loja, produto e status UCP em `vendas`.

Na prática, esta seção deve ser lida como mapa de domínio do banco demo, não como script de provisionamento. O objetivo operacional é orientar integrações, NL2SQL, SQL dinâmico, dashboards e troubleshooting sem manter blocos DDL extensos dentro do manual geral.

## Observações Finais

- Este manual reflete o DDL atual informado para o schema público, incluindo o modelo final de conta pessoal, autenticação e cobrança organizacional.
- Este manual também registra o schema `integrations` já implementado e o desenho contratual já aprovado para a tabela global `integrations.builtin_tool_registry`, para documentar no mesmo lugar o armazenamento dos cadastros técnicos, funcionais e do catálogo builtin.
- Este manual também passa a registrar o schema demo de varejo consultado por `DATABASE_VAREJO_DSN` e `DATABASE_VAREJO_SCHEMA`, para facilitar futuras consultas operacionais de SQL, UCP, dashboards e NL2SQL.
- Estruturas antigas que não aparecem mais no DDL foram removidas deste documento para evitar ambiguidade.
- Quando o DDL mudar, este manual deve ser atualizado no mesmo ciclo para manter a documentação confiável.
