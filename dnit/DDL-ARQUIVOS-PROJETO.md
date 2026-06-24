# DDL — Arquivos do Projeto e Histórico do Assistente

Este documento descreve a modelagem complementar usada pela tela de detalhe do projeto DNIT para governar:

- o estado semântico de cada arquivo do projeto;
- a referência opaca da vector store individual do arquivo;
- o status de indexação;
- a permissão de participação no assistente;
- o histórico persistido de trecho, pergunta e resposta por arquivo.

## Princípio de desenho

O arquivo físico continua nas tabelas já existentes do workspace:

- `public.tenant_user_projects`
- `public.tenant_user_project_details`

A nova modelagem não substitui esse fluxo. Ela o complementa com duas tabelas genéricas, sem acoplamento nominal ao domínio DNIT:

- `public.tenant_user_project_file_registry`
- `public.project_file_context_entries`

## Tabela `tenant_user_project_file_registry`

Fonte de verdade semântica do arquivo para indexação e uso no assistente.

Campos principais:

- `tenant_user_project_detail_id`:
  chave primária e FK para `tenant_user_project_details`; exclusão em cascata
- `tenant_user_project_id`:
  FK para `tenant_user_projects`
- `tenant_id`:
  tenant dono do arquivo
- `user_email`:
  operador responsável pela última escrita semântica
- `original_file_name` e `full_file_name`:
  nomes persistidos do arquivo
- `storage_bucket` e `storage_object_key`:
  referência canônica do objeto no MinIO
- `vector_store_id`:
  identificador opaco da vector store por arquivo
- `vector_store_ref`:
  referência persistida usada para exclusão e auditoria
- `indexing_status`:
  `not_indexed`, `processing`, `indexed` ou `failed`
- `indexing_error_message`:
  última falha resumida de indexação
- `assistant_enabled`:
  indica se o arquivo pode participar do contexto governado do assistente
- `last_indexed_at`:
  timestamp da última indexação bem-sucedida
- `deleted_at`:
  reservado para evolução futura
- `created_at`, `updated_at`

## Tabela `project_file_context_entries`

Histórico persistido do fluxo já existente na UI:

- trecho selecionado;
- pergunta enviada;
- resposta recebida;
- correlation ID oficial retornado pelo assistente.

Campos principais:

- `project_file_context_entry_id`:
  chave primária
- `tenant_user_project_detail_id`:
  FK para `tenant_user_project_file_registry`
- `tenant_user_project_id`:
  FK para `tenant_user_projects`
- `tenant_id`
- `user_email`
- `excerpt_text`
- `question_text`
- `answer_text`
- `correlation_id`
- `created_at`, `updated_at`

## Cascatas e exclusão

Ao excluir um arquivo do projeto:

1. o binário deve sair do storage;
2. a vector store referenciada em `vector_store_id` deve ser removida;
3. a linha em `tenant_user_project_details` deve ser removida;
4. as linhas em `tenant_user_project_file_registry` e `project_file_context_entries` caem por cascata.

## Script SQL

O DDL operacional desta etapa está em:

- `scripts/sql/20260616_create_project_file_registry_and_context_entries.sql`
