# Manual técnico: Gestão e ciclo de vida das conversas do chat

## 1. O que é este documento (nível 101)

Toda conversa feita pelo webchat da plataforma — seja Q&A/RAG, Agent, DeepAgent ou Workflow — é **guardada no banco** para que o usuário possa reabrir, continuar e renomear depois. Este documento explica **como essas conversas são armazenadas, listadas, apagadas e mantidas ao longo do tempo**.

Não confunda com dois assuntos vizinhos:

- **Como a pergunta vai e volta do backend** (payload, endpoints por modo, criptografia): isso está em [Processo de conversa da plataforma](README-TECNICO-PROCESSO-CONVERSA-PLATAFORMA.md).
- **Como o componente de chat embutível guarda sessões no navegador** (histórico local, múltiplas conversas headless no cliente): isso está no [Guia do Componente WebChat Embutível](../usuario/GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md).

Aqui o foco é o **lado servidor**: a persistência canônica das conversas, o ciclo de vida (ativa → apagada → purgada) e as rotinas automáticas que cuidam disso.

## 2. Fonte de verdade lida no código

- `src/chat/repository.py` — `ChatConversationsRepository` e `ChatMessagesRepository` (a camada de persistência).
- `src/chat/models.py` — `ChatConversationRecord`, `ChatMessageRecord`.
- `src/api/routers/chat_conversations_router.py` — endpoints do usuário (prefixo `/chat`).
- `src/agentic_layer/supervisor/checkpoint_cleaner.py` — `LangGraphCheckpointCleaner` (purga + apagar checkpoint).
- `src/api/services/chat_conversation_retention_maintenance_job.py` — rotina diária de retenção.
- `src/api/services/chat_conversation_compaction_maintenance_job.py` — rotina diária de compactação.
- Dicionário de dados das tabelas: [README-SCHEMA-BANCO.md](README-SCHEMA-BANCO.md).

## 3. Modelo de dados

Duas tabelas no schema `chat`:

- **`chat.conversations`** — uma linha por conversa. Campos principais: `conversation_id` (identidade estável), `tenant_code` e `user_email` (dono), `config_ref` (o YAML que originou a conversa), `scope_ref` (recorte opcional), `mode` (`qa`, `agent`, `deepagent`, `workflow`), `thread_id` (liga a conversa ao checkpoint do LangGraph), `title`, `status` (`active` ou `deleted`), `message_count`, `metadata`, `created_at`, `updated_at`, `last_message_at`.
- **`chat.messages`** — uma linha por mensagem, com `conversation_id` referenciando `chat.conversations` por **chave estrangeira `ON DELETE CASCADE`**. Apagar a conversa remove as mensagens automaticamente, sem SQL extra.

O `thread_id` é o elo com a memória de execução: o estado do agente/DeepAgent vive num **checkpoint do LangGraph** endereçado por esse `thread_id`. Conversa e checkpoint são coisas separadas que precisam ser tratadas juntas no fim da vida (ver seção 6).

## 4. Endpoints do usuário (escopados por dono)

Todos exigem usuário autenticado e são **escopados por `tenant_code` + `user_email`**: um usuário só enxerga e mexe nas próprias conversas.

| Método | Rota | O que faz |
| --- | --- | --- |
| `GET` | `/chat/conversations` | Lista as conversas do usuário (exclui as apagadas), ordenadas por atividade recente. |
| `GET` | `/chat/conversations/{conversation_id}` | Abre uma conversa e suas mensagens. |
| `PATCH` | `/chat/conversations/{conversation_id}` | Renomeia a conversa. |
| `DELETE` | `/chat/conversations/{conversation_id}` | **Soft-delete**: marca `status = deleted`. A conversa some da lista, mas o registro permanece no banco. |

Importante: o `DELETE` do usuário **não apaga fisicamente** nem remove o checkpoint. Ele apenas oculta. A remoção física é responsabilidade da rotina de retenção (seção 7), que também limpa o checkpoint.

## 5. A camada de persistência (`ChatConversationsRepository`)

O repositório é o **boundary único** de acesso às tabelas de conversa. Ele estende `ClientDirectoryBase`, o que traz de graça: execução centralizada de SQL com retry, log canônico por operação e propagação de `correlation_id`. Nenhum serviço de domínio executa SQL de conversa por fora dele.

Operações confirmadas:

- `upsert_conversation(...)` — cria ou atualiza a conversa de forma idempotente pela chave natural; usa `COALESCE` para não sobrescrever campos existentes com `NULL`.
- `list` de conversas do usuário — escopado por dono, com "baldes" por `scope_ref`, excluindo `deleted`, paginado por `offset`/`limit`.
- `rename_conversation(...)` — renomeia respeitando o escopo do dono.
- soft-delete — marca `status = deleted` respeitando o escopo do dono.
- `touch_last_message(...)` — atualiza `last_message_at` e recalcula `message_count` a partir das mensagens reais.
- `list_conversations_last_message_before(last_message_before=...)` — **administrativo, cross-tenant**: lista conversas paradas antes de um instante. Base da retenção.
- `list_deepagent_conversations_for_compaction(active_within_days=...)` — **administrativo, cross-tenant**: lista conversas `agent`/`deepagent` com `config_ref` e `thread_id`, opcionalmente restritas a atividade recente. Base da compactação.
- `hard_delete_conversation(conversation_id=...)` — **administrativo**: apaga fisicamente a linha (e as mensagens caem por cascade). Não é exposto ao usuário.

As duas naturezas de acesso não se misturam: **operações do usuário são sempre escopadas por dono**; **operações de manutenção são cross-tenant/cross-user** e existem só para as rotinas administrativas.

## 6. Ciclo de vida de uma conversa

```
        cria/usa                DELETE do usuário            retenção (>60 dias parada)
 [inexistente] ─────► [active] ──────────────────► [deleted] ──────────────────────► [removida do banco]
                         │                                                                    ▲
                         └───────────────── retenção também alcança conversas active antigas ─┘
```

- **active** — conversa viva, aparece na lista do usuário, recebe novas mensagens.
- **deleted (soft-delete)** — oculta para o usuário, mas ainda ocupa espaço no banco e ainda tem checkpoint. É um estado de transição, não o fim.
- **removida** — a rotina de retenção apaga fisicamente a linha (cascade nas mensagens) **e** apaga o checkpoint do LangGraph associado. Só aqui a conversa deixa de existir de fato.

**Acoplamento obrigatório conversa ↔ checkpoint:** apagar a conversa sem apagar o checkpoint deixaria um checkpoint órfão consumindo espaço; apagar o checkpoint sem a conversa deixaria uma conversa quebrada. Por isso a remoção física trata os dois juntos, e a remoção do checkpoint passa **exclusivamente pela API oficial do LangGraph** `BaseCheckpointSaver.adelete_thread(thread_id)` (via `LangGraphCheckpointCleaner.delete_thread`), nunca por manipulação direta das tabelas de checkpoint.

## 7. Retenção automática (rotina diária)

A rotina `chat-conversation-retention-cleanup` roda todo dia às **02:00 (America/Sao_Paulo)** (cron `0 5 * * *` em UTC) dentro do scheduler de manutenção (ver [Scheduler §7.3](README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md)).

O que ela faz, por conversa vencida:

1. Lista as conversas com `last_message_at` mais velho que **60 dias** (`older_than_days`, configurável). Isso alcança tanto conversas `active` esquecidas quanto conversas já em `deleted`.
2. Apaga o checkpoint do LangGraph via `adelete_thread` (API oficial).
3. Apaga fisicamente a linha de `chat.conversations` (`hard_delete_conversation`); as mensagens caem por cascade.
4. Falha em uma conversa **não interrompe** a rodada: o erro é logado e a rotina segue para a próxima. A operação é idempotente e se auto-recupera na rodada seguinte.

Configuração (`src/config/settings.py`): `CHAT_CONVERSATION_RETENTION_MAINTENANCE_ENABLED` (default ligado), `..._CRON_EXPRESSION` (`0 5 * * *`), `..._OLDER_THAN_DAYS` (`60`).

## 8. Compactação de contexto (rotina diária)

A rotina `chat-conversation-compaction` roda às **03:00 (America/Sao_Paulo)** (cron `0 6 * * *` em UTC), logo após a retenção. Ela compacta o contexto das conversas DeepAgent **com atividade recente** (janela default de 2 dias), reduzindo custo e latência sem perder o histórico essencial.

O mecanismo (tool oficial `compact_conversation`, evento de resumo e índice de corte) está detalhado em **[DeepAgent Supervisor §7.8](README-TECNICO-DEEPAGENT-SUPERVISOR-COMPLETO.md)**. Configuração: `CHAT_CONVERSATION_COMPACTION_MAINTENANCE_ENABLED` (default ligado), `..._CRON_EXPRESSION` (`0 6 * * *`), `..._ACTIVE_WITHIN_DAYS` (`2`).

## 9. Sessões no cliente (navegador)

Além do que o servidor guarda, o **componente de chat embutível** mantém sessões de conversa no navegador (histórico local, várias conversas salvas, hidratação de mensagens externas). Isso é uma camada separada e complementar, documentada em [Guia do Componente WebChat Embutível — §22 e §22.1](../usuario/GUIA-COMPONENTE-WEBCHAT-EMBUTIVEL.md). O servidor é a fonte durável; o cliente é a conveniência de navegação.

## 10. Erros a evitar

- **Não** executar SQL de conversa por fora do `ChatConversationsRepository` (ele é o boundary com retry, log e escopo).
- **Não** apagar checkpoint por SQL direto: use sempre `adelete_thread` via `LangGraphCheckpointCleaner`.
- **Não** tratar soft-delete como remoção definitiva: a conversa e o checkpoint continuam existindo até a retenção.
- **Não** rodar operações administrativas (retenção/compactação) sem escopo cross-tenant consciente: elas atravessam tenants por natureza e só devem viver nas rotinas de manutenção.

## 11. Evidências no código

- `src/chat/repository.py` — todas as operações de persistência de conversa e mensagem, sobre `ClientDirectoryBase` (execução central de SQL, retry, log, `correlation_id`).
- `src/api/routers/chat_conversations_router.py` — `GET/GET/PATCH/DELETE /chat/conversations[...]`, escopados por dono; o `DELETE` é soft-delete.
- `src/agentic_layer/supervisor/checkpoint_cleaner.py` — `purge_chat_conversations_older_than` e `delete_thread` → `adelete_thread` oficial.
- `src/api/services/chat_conversation_retention_maintenance_job.py` — orquestra a retenção e devolve resumo auditável.
- `src/api/services/chat_conversation_compaction_maintenance_job.py` — orquestra a compactação diária, isolando falha por conversa.
