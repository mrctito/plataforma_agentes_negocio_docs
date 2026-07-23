# Manual tecnico e operacional: Gestao SaaS x Tenant e telas de administracao

## 1. Escopo tecnico deste manual

Este manual cobre o boundary comercial SaaS (`src/saas_project`, `src/saas_commercial`,
`src/api/routers/saas_router.py`, `src/api/services/saas_http_service.py`), as telas de gestao
que operam esse boundary (`ui-admin-saas-projects.html` e sua familia de sete abas,
`ui-saas-my-projects.html`, `ui-saas-project.html`) e a governanca de tenant/usuario que sustenta a
identidade usada pelo SaaS (`ui-admin-gov-tenants.html`, `ui-admin-gov-users.html`,
`ui-admin-gov-memberships.html`, `ui-admin-gov-human-grants.html`,
`ui-admin-gov-tenant-secrets.html`, `ui-admin-gov-channel-end-users.html`) e os dois hubs de
navegacao (`saas_index.html`, `governanca_index.html`).

O modelo fisico de dados (`saas_projects`, `saas_project_releases`, `saas_plans`,
`saas_subscriptions`, `saas_entitlements`, `saas_billing_events`, `saas_user_preferences`, tabelas
`tenant_*`) esta documentado em `docs/tecnico/README-SCHEMA-BANCO.md`, secao "Estado aplicado do
modulo SaaS". Este manual nao repete o dicionario de colunas — ele documenta o **boundary, o fluxo, as
permissoes e o papel de cada tela** que operam esse modelo.

## 2. Entrypoints reais envolvidos

### 2.1. Boundary HTTP SaaS

Um unico router, `src/api/routers/saas_router.py`, incluido em `service_api.py` (linha 1652 no
codigo lido), concentra toda a superficie SaaS. Duas funcoes internas resolvem identidade:

- `_session(request)` — exige sessao federada valida (`user_account_id` presente); usada pelas
  rotas de assinante.
- `_admin_scope(request)` — chama `_session` e, alem disso, resolve e autoriza o tenant escolhido
  pela UI (`X-SaaS-Tenant-Id`) contra o membership humano do operador via
  `ClientDirectory.resolve_membership_authorization_context`; usada pelas rotas administrativas.

### 2.2. Boundary de governanca de tenant/usuario

- `src/api/routers/admin/tenants_router.py` (`/admin/tenants*`) → `AdminTenantsService`.
- `src/api/routers/admin/users_router.py` (`/admin/users`, `/admin/channels`,
  `/admin/reload-users`, `/admin/user-stats/{...}`) → `AdminUsersService`.
- `src/api/routers/auth_router.py` — memberships humanas (`/admin/memberships` e
  `/admin/memberships/invitations`) e o catalogo de permissao (`/api/auth/admin/permission-catalog`).
- `src/security/client_directory.py` — diretorio central de tenants, memberships, chaves e segredos,
  consumido tanto pela governanca quanto pelo `_admin_scope` do SaaS.

### 2.3. Telas servidas

A pagina administrativa do produto SaaS e servida autenticada por `src/api/routers/ui_router.py`
(rotas `GET /admin/saas/projects` e `GET /admin/saas/projects/{project_id}`, ambas devolvendo
`ui-admin-saas-projects.html`). As demais telas HTML deste manual sao estaticas em
`app/ui/static/`, carregadas via `/ui/static/<arquivo>.html`.

## 3. Fluxo tecnico ponta a ponta

```text
Tela admin SaaS (saas-products-admin.js)
        |  PrometeuAdminApiClient.adminFetchJson (cookie de sessao federada)
        v
saas_router.py  --_admin_scope()--> ClientDirectory.resolve_membership_authorization_context
        |
        v
SaasHttpService (orquestra)
        |-- SaasHttpRepository (read-models de projeto/release/plano/assinante; PostgresQueryExecutor)
        |-- build_project_release_service() -> src/saas_project/service.py (lifecycle de release, CAS)
        |-- SaasCommercialService -> src/saas_commercial/service.py (plano/assinatura/entitlement/billing)
        v
PostgreSQL (tabelas saas_*, tenants, user_accounts, tenant_yaml)
```

O runtime de execucao (Agent/AG-UI, RAG, canal) e um consumidor separado: ele resolve
`projectKey` -> release ativa -> YAML/manifesto por fora deste router, mas usa o mesmo modelo de
dados (`saas_project_active_releases`) para decidir qual snapshot executar.

## 4. Passo a passo tecnico do ciclo do produto

### 4.1. Criar e publicar

`POST /api/admin/saas/projects` cria o projeto (`SAAS_PROJECTS_MANAGE`). `GET
/api/admin/saas/projects/{id}/tenant-yamls`-equivalente (`GET .../tenant-yamls`) lista YAMLs
elegiveis do tenant. `POST .../releases` cria a release em `draft`, resolvendo hash/manifesto a
partir do `tenant_yaml` escolhido. `POST .../releases/{release_id}/publish` congela a release
(`published`), validando `yaml_hash`/`manifest_hash` informados contra o que foi calculado.

### 4.2. Ativar

`POST .../releases/{release_id}/activate` troca o ponteiro
(`saas_project_active_releases`) por compare-and-set, usando `expected_release_id` para detectar
troca concorrente; grava `activated_by_user_account_id` a partir da sessao do operador
(`SAAS_PROJECTS_MANAGE`).

### 4.3. Ofertar (plano)

`POST .../plans` cria plano (`SAAS_PROJECTS_MANAGE`), validando que `operations` e subconjunto das
operacoes da release ativa (`service.create_plan`, `src/api/services/saas_http_service.py:274`).
`PATCH .../plans/{plan_id}` (`update_plan`, linha 312) edita nome/operacoes/preco/moeda com a mesma
validacao. `POST .../plans/{plan_id}/status` (`set_plan_status`, linha 354) transiciona
`active/inactive/archived`. As tres acoes chamam `_log_state_change` com `event_name` explicito
(`saas_http.plan.created|.updated|.status_changed`) e `saas_plan_id` no contexto de log.

### 4.4. Assinar (assinante)

`GET /api/public/saas/projects/{tenant_id}/{project_key}` (publico, sem autenticacao) devolve
capacidades e planos. `POST /api/saas/projects/{tenant_id}/{project_key}/checkout`
(`SAAS_SUBSCRIBE`, exige sessao + `Idempotency-Key`) cria a assinatura e concede entitlements.
`POST .../subscriptions/{id}/{confirm|cancel}` transiciona a assinatura.

### 4.5. Medir e auditar

`GET .../projects/{id}/audit` (paginado, filtros `source/operation/release_id/target_correlation_id`)
e `GET .../projects/{id}/usage` (agregado, filtros `operation/release_id`) — ambos
`SAAS_PROJECTS_READ` — sao as duas rotas que a rodada mais recente (P3) passou a expor nas abas
"Auditoria" e "Uso" da tela administrativa; ate entao existiam apenas no backend.

## 5. Modos de autorizacao confirmados no codigo

O decorador `endpoint_permission(permission, mode, ui_session_required)`
(`src/api/security/permission_metadata.py`) tem `mode` default `AccessMode.HEADER` (exige
`X-API-Key`). O comportamento real, confirmado por leitura, difere por familia de tela:

| Superficie | Modo | Identidade exigida |
|---|---|---|
| `saas_router.py` (admin e assinante) | `AccessMode.CUSTOM`, `ui_session_required=True` | sessao federada; admin tambem exige membership no `X-SaaS-Tenant-Id` |
| `admin/tenants_router.py`, `admin/users_router.py` | default `AccessMode.HEADER` (nao sobrescrito) | `X-API-Key` com a permissao `ADMIN_TENANTS_*`/`ADMIN_USERS_*`/`ADMIN_CHANNELS_*` |
| `auth_router.py` `/admin/memberships` | sem `@endpoint_permission` (fallback `PUBLIC` no middleware) | sessao federada validada manualmente dentro do handler (`_load_federated_session_record`) |

Na pratica: **a tela de produtos SaaS e as telas de governanca de membership/grants operam por
sessao de navegador** (cookie); **as telas de tenants/usuarios/segredos/canal operam por
`X-API-Key`**, resolvida no navegador pelo `PrometeuAdminLayoutBridge` (`resolveAccessKey`). Isso
nao e um detalhe cosmetico: uma integracao tecnica que queira automatizar a tela de tenants precisa
de uma API key com a permissao certa; a mesma automacao para a tela SaaS precisaria simular uma
sessao de navegador, o que essas rotas nao aceitam por API key.

## 6. Catalogo de permissoes do modulo

Fonte: `src/api/security/permissions.py` (`class PermissionKeys`, linhas 115-119; descritores em
`PERMISSION_CATALOG`, linhas 564-587). Todas as permissoes SaaS tem `audience=HUMAN` — nao sao
concedidas a credencial de maquina.

| Permissao | Uso confirmado | Rotas |
|---|---|---|
| `SAAS_PROJECTS_READ` | leitura administrativa de projetos/releases/audit/usage | `GET` de projetos, releases, tenant-yamls, audit, usage, `GET /api/admin/saas/tenants` |
| `SAAS_PROJECTS_MANAGE` | lifecycle de projeto/release/plano | `POST/PATCH` de projetos, `POST` releases/publish/activate, `POST/PATCH` planos, `POST` status de plano |
| `SAAS_COMMERCIAL_READ` | leitura de assinantes e decisao de entitlement | `GET` planos (listagem publica administrativa), `GET` subscribers, `GET` entitlement |
| `SAAS_COMMERCIAL_MANAGE` | cancelamento governado de assinatura | `POST .../subscribers/{id}/cancel` |
| `SAAS_SUBSCRIBE` | checkout e gestao da propria assinatura | `POST checkout`, `POST transition`, `GET /api/saas/me/projects`, `POST cancel_own_subscription` |

A governanca de tenant/usuario usa um catalogo separado e mais antigo (`ADMIN_TENANTS_READ`,
`ADMIN_TENANTS_WRITE`, `ADMIN_TENANTS_STATUS`, `ADMIN_USERS_READ`, `ADMIN_USERS_WRITE`,
`ADMIN_USERS_RELOAD`, `ADMIN_CHANNELS_VIEW`, `ADMIN_CHANNELS_MANAGE`), todo em `AccessMode.HEADER`
(`src/api/routers/admin/tenants_router.py`, `src/api/routers/admin/users_router.py`).

## 7. Telas de gestao: o que cada uma faz e o que exige

### 7.1. `ui-admin-saas-projects.html` (`saas-products-admin.js`)

Tela administrativa principal do modulo, servida autenticada em `/admin/saas/projects`. Exige sessao
federada com membership autorizado na organizacao selecionada (`X-SaaS-Tenant-Id`, com preferencia
lembrada em `localStorage` desde a correcao de UX da rodada anterior). Sete abas no detalhe do
projeto (`data-saas-tab`):

- **Visao geral** — identidade e status do projeto; edicao de nome/status (`SAAS_PROJECTS_MANAGE`).
- **Versoes** — lista releases, publica nova versao e ativa (formulario inline substituiu o
  `prompt()` nativo desde a rodada mais recente).
- **Capacidades** — operacoes publicadas pela release ativa (somente leitura).
- **Planos** — lista/cria/edita/inativa/reativa/arquiva plano, com preco e moeda reais (antes
  fixo "R$ 0,00"); usa `formatMoney` e os endpoints de T6 (`PATCH .../plans/{id}`,
  `POST .../plans/{id}/status`).
- **Assinantes** — lista paginada (offset/limit real, filtros `status`/`query` no servidor; antes
  usava `limit=25&offset=0` fixo com filtro client-side), ve entitlement e cancela assinatura.
- **Uso** — nova nesta rodada: agregados por operacao/release (`GET .../usage`), com filtros.
- **Auditoria** — nova nesta rodada: eventos paginados e filtraveis (`GET .../audit`, reusa o
  paginador de Assinantes).

Todas as confirmacoes destrutivas (mudar status de plano, ativar release, cancelar assinante,
arquivar projeto) usam `window.prometeuConfirmar` (dialogo canonico compartilhado da familia admin,
`app/ui/static/js/shared/alpine-components.js`) — o modulo falha ao carregar (`throw` no topo do
arquivo) se essa dependencia nao estiver presente, para nunca cair de volta a `confirm()` nativo.

### 7.2. `ui-saas-my-projects.html` (`saas-project-public.js`)

Tela do assinante: lista as proprias assinaturas (`GET /api/saas/me/projects`) e permite cancelar
(`POST /api/saas/me/subscriptions/{id}/cancel`). Exige apenas sessao federada da propria conta — nao
aceita nem exibe seletor de tenant.

### 7.3. `ui-saas-project.html`

Pagina publica de um produto especifico (catalogo + checkout), servida por `ui_router.py`. Consome
`GET /api/public/saas/projects/{tenant}/{key}` (sem autenticacao) para exibir capacidades e planos, e
`POST .../checkout` / `POST .../subscriptions/{id}/{action}` (exige sessao + `Idempotency-Key`) para
assinar. Nunca renderiza YAML, entrypoint ou chave tecnica.

### 7.4. `saas_index.html` — hub da familia SaaS

Pagina de navegacao (`data-area-home-key="saas"`) que lista, a partir do catalogo compartilhado
`admin-area-navigation-catalog.js` (`saasActions`), os atalhos para: Home, Produtos SaaS
(`/admin/saas/projects`), Tenants, Memberships, Grants humanos, Usuarios finais de canal,
Credenciais e Segredos de tenant. Nao renderiza dado proprio — e so navegacao.

### 7.5. `ui-admin-gov-tenants.html` (`admin-gov-tenants.js`)

CRUD de tenants (organizacoes): criar, editar, consultar status. Exige `X-API-Key` com
`ADMIN_TENANTS_READ`/`WRITE`/`STATUS`, resolvida pelo `PrometeuAdminLayoutBridge`.

### 7.6. `ui-admin-gov-users.html` (`admin-users.js`)

Credenciais tecnicas (`/admin/users`) e canais (`/admin/channels`) do tenant; recarga de cadastro
(`/admin/reload-users`) e estatisticas (`/admin/user-stats/{...}`). Exige `X-API-Key` com
`ADMIN_USERS_*`/`ADMIN_CHANNELS_*`.

### 7.7. `ui-admin-gov-memberships.html` (`admin-memberships.js`)

Lista e revoga memberships humanos (`/api/auth/admin/memberships`,
`/api/auth/admin/memberships/{id}/revoke`) e convites (`/api/auth/admin/memberships/invitations`).
Exige apenas sessao federada — a rota nao tem `@endpoint_permission`, mas o handler exige
`_load_federated_session_record` internamente.

### 7.8. `ui-admin-gov-human-grants.html` (`admin-human-grants.js`)

Grants efetivos por membership: cruza `/api/auth/admin/memberships` com
`/api/auth/admin/permission-catalog` para mostrar/editar o que cada pessoa pode fazer. Mesma
exigencia de sessao federada que a tela de memberships.

### 7.9. `ui-admin-gov-tenant-secrets.html` (`admin-tenant-secrets.js`)

Segredos genericos por tenant (`tenant_secrets`), usados por BYOK e por resolucao de placeholder no
YAML (ver `docs/tecnico/README-TECNICO-BYOK-ISOLAMENTO-CUSTOS-TENANT.md`). Exige `X-API-Key`.

### 7.10. `ui-admin-gov-channel-end-users.html` (`admin-channel-end-users.js`)

Allow/blocklist de remetentes por canal (`tenant_channel_end_users`), a partir de `/admin/channels`.
Exige `X-API-Key`.

### 7.11. `governanca_index.html` — hub da familia Governanca

Mesmo padrao de `saas_index.html`, listando os atalhos das seis telas acima a partir de
`governancaActions` no catalogo compartilhado.

## 8. Segregacao por environment

`SaasHttpRepository.environment` (`src/saas_project/http_repository.py:25-29`) e uma property que
deriva o ambiente do segregador canonico (`self._environment_name()`, herdado de
`ClientDirectoryBase`) — **nao** um literal fixo. Todas as queries de leitura do boundary (projetos,
releases, plano, assinantes) usam `_environment_literal()` para injetar esse valor com seguranca no
SQL (`sql.Literal`). Isso e o oposto do estado anterior a esta rodada, em que o read-model fixava
`environment='prod'` de forma literal.

O dominio comercial (`src/saas_commercial/service.py`) tem uma trava adicional e independente:
`SaasCommercialService._require_prod(environment)` recusa operar quando o ambiente nao e `prod` —
checkout, transicao de assinatura e concessao de entitlement so funcionam hoje em producao. Ou seja:
**leitura de projeto/release/plano ja segue o `ENVIRONMENT` canonico do processo; escrita comercial
(assinatura/billing/entitlement) segue restrita a `prod`** ate que exista necessidade real de simular
comercio em outro ambiente.

## 9. Contratos e campos relevantes

Todos os schemas do boundary herdam de `SaasApiModel` (`src/api/schemas/saas_models.py:17-20`), com
`model_config = ConfigDict(extra="forbid")` — qualquer campo nao declarado no request e rejeitado.
Payloads centrais confirmados:

- `SaasProjectCreateRequest` — `project_key` (regex `^[a-z0-9][a-z0-9_-]{1,62}[a-z0-9]$`),
  `display_name`.
- `SaasPlanUpdateRequest` — `display_name`, `operations`, `price_minor` (`ge=0`), `currency`
  (regex `^[A-Z]{3}$`).
- `SaasPlanStatusRequest` — `status` (`active/inactive/archived`).
- `SaasAuditEventPayload` — `event_name`, `source` (`release|billing|interaction|job`), `status`,
  `occurred_at`, `correlation_id`, `operation`, `release_id`, hashes de release, `usage_event_key`,
  `usage_counted` — nunca conteudo de configuracao.
- `SaasUsageMetricPayload` — `operation`, `release_id`, hashes, `total`, `succeeded`, `failed`,
  `average_duration_ms` — agregado autoral, sem custo financeiro estimado.
- `PublicProjectPayload` — `project_key`, `display_name`, `capabilities`, `plans` — o unico payload
  publico, sem YAML nem entrypoint.
- `MyProjectPayload` — inclui `access_mode`
  (`agent_chat|rag_chat|external_channel|unavailable`), que e o campo que a UI do assinante usa para
  decidir que experiencia oferecer, sem nunca expor a release por tras.

Toda acao de mutacao de assinatura (`checkout`, `transition_subscription`, `cancel_own_subscription`)
exige o header `Idempotency-Key` (`min_length=1, max_length=256`) — reenviar a mesma chave nao
duplica o efeito.

### 9.1. O YAML por tras de uma release (contrato existente, nao um formato novo)

O modulo SaaS nao inventa um dialeto de YAML proprio: a release aponta para um `tenant_yaml` ja
escrito segundo o contrato descrito em `docs/tecnico/README-CONFIGURACAO-YAML.md`. O unico requisito
adicional e que o YAML declare, quando houver agente, um `selected_entrypoint` valido — e o manifesto
da release (`manifest_json`) e derivado automaticamente disso, nunca escrito a mao pelo operador:

```yaml
# trecho ilustrativo de um tenant_yaml elegivel para virar release (campos ja documentados
# em README-CONFIGURACAO-YAML.md; nao ha campo especifico de "SaaS" dentro do YAML)
selected_entrypoint:
  kind: deepagent
  ref: ag_ui_pdv_operacoes_supervisor
agent-instructions-md: |
  Instrucoes do supervisor, versionadas junto com o YAML.
```

Ao publicar, o compilador le esse YAML, calcula `yaml_hash` (SHA-256 do conteudo), monta
`manifest_json` (`schema_version`, `yaml_hash`, `instructions_hash`, `entrypoint`, `operations`,
`manifest_hash`) e a release passa a ser essa combinacao imutavel. O plano so pode oferecer
operacoes que constam em `manifest_json.operations`.

## 10. Exemplo tecnico guiado

Cenario: publicar o produto "PDV Vendas" (agentic) e assinar como cliente de teste.

1. Operador autentica na UI (sessao federada) e seleciona a organizacao `pdv-vendas` no seletor de
   tenant da tela SaaS.
2. `POST /api/admin/saas/projects` cria o projeto `pdv-vendas` com `display_name="PDV Vendas"`.
3. `GET .../tenant-yamls` lista os YAMLs elegiveis; o operador escolhe o YAML com
   `selected_entrypoint` do supervisor de operacoes.
4. `POST .../releases` cria a release em `draft`; `POST .../releases/{id}/publish` congela hash e
   manifesto (`operations` inclui `agent` e `rag`).
5. `POST .../releases/{id}/activate` troca o ponteiro do projeto para essa release.
6. `POST .../plans` cria o plano oferecendo apenas `agent` e `rag` (nunca `ingest`/`etl`, mesmo que
   a release publique essas operacoes).
7. Um cliente abre `ui-saas-project.html?tenant=pdv-vendas&project=...`, ve o plano, faz
   `POST .../checkout` com `Idempotency-Key` — a assinatura nasce `active` (provider `simulated`) e
   o entitlement `agent`+`rag` e concedido.
8. O cliente usa o chat; o boundary de execucao AG-UI resolve a release ativa e verifica o
   entitlement antes de abrir o stream.
9. A administracao confere na aba Uso quantas execucoes daquela release tiveram sucesso/falha, e na
   aba Auditoria o evento de checkout com o `correlation_id` daquela sessao.

## 11. Exemplo completo em JavaScript (catalogo publico -> checkout -> minhas assinaturas)

O exemplo abaixo reflete exatamente os tres endpoints do fluxo de assinante
(`saas_router.py:711-823`): o catalogo publico nao exige autenticacao; checkout e a listagem "minhas
assinaturas" exigem a sessao federada do navegador (cookie `credentials: 'include'`) e, no caso do
checkout, um `Idempotency-Key` para tornar o reenvio seguro.

```javascript
// Roda no navegador, com o usuario ja logado (sessao federada existente via cookie).
const TENANT_ID = 'pdv-vendas';
const PROJECT_KEY = 'operacoes';

async function gerarIdempotencyKey() {
  // qualquer string unica por tentativa de checkout serve; UUID e o padrao usado pela UI oficial.
  return crypto.randomUUID();
}

async function buscarCatalogoPublico(tenantId, projectKey) {
  const resposta = await fetch(`/api/public/saas/projects/${tenantId}/${projectKey}`, {
    method: 'GET',
    headers: { Accept: 'application/json' },
  });
  const corpo = await resposta.json();
  if (!resposta.ok) {
    throw new Error(`Catalogo publico falhou (${resposta.status}): ${corpo?.detail?.message || 'erro desconhecido'}`);
  }
  console.log('Correlacao do catalogo:', resposta.headers.get('X-Correlation-Id'));
  return corpo.project; // { project_key, display_name, capabilities, plans }
}

async function assinarPlano(tenantId, projectKey, planId) {
  const idempotencyKey = await gerarIdempotencyKey();
  const resposta = await fetch(`/api/saas/projects/${tenantId}/${projectKey}/checkout`, {
    method: 'POST',
    credentials: 'include', // envia o cookie de sessao federada
    headers: {
      'Content-Type': 'application/json',
      'Idempotency-Key': idempotencyKey,
    },
    body: JSON.stringify({ plan_id: planId }),
  });
  const corpo = await resposta.json();
  if (!resposta.ok) {
    // 401 = sem sessao; 404 = projeto/plano nao encontrado; 409 = conflito de estado;
    // 422 = contrato invalido (ex.: plan_id de outro projeto).
    throw new Error(`Checkout falhou (${resposta.status}): ${corpo?.detail?.message || 'erro desconhecido'} | correlacao=${corpo?.detail?.correlation_id}`);
  }
  return corpo.subscription; // { subscription_id, project_id, plan_id, status, version, simulation }
}

async function listarMinhasAssinaturas() {
  const resposta = await fetch('/api/saas/me/projects', {
    method: 'GET',
    credentials: 'include',
    headers: { Accept: 'application/json' },
  });
  const corpo = await resposta.json();
  if (!resposta.ok) {
    throw new Error(`Listagem de assinaturas falhou (${resposta.status}): ${corpo?.detail?.message}`);
  }
  return corpo.items; // [{ project_id, subscription_id, subscription_status, access_mode, ... }]
}

// Uso ponta a ponta
(async () => {
  const projeto = await buscarCatalogoPublico(TENANT_ID, PROJECT_KEY);
  const planoEscolhido = projeto.plans.find((plano) => plano.status === 'active');
  if (!planoEscolhido) {
    console.warn('Nenhum plano ativo disponivel para assinatura.');
    return;
  }

  const assinatura = await assinarPlano(TENANT_ID, PROJECT_KEY, planoEscolhido.plan_id);
  console.log('Assinatura criada:', assinatura.subscription_id, assinatura.status);

  const minhasAssinaturas = await listarMinhasAssinaturas();
  console.log('Projetos assinados:', minhasAssinaturas.map((item) => item.project_key));
})();
```

Pontos de atencao confirmados no codigo: o `plan_id` enviado no checkout precisa pertencer ao
projeto/tenant do path (validado no service); `access_mode` retornado em "minhas assinaturas" decide
se a UI deve abrir chat de agente, chat RAG, ou apenas orientar o cliente para um canal externo —
nunca ha URL de webhook, YAML ou entrypoint no payload do assinante.

## 12. O que acontece em caso de sucesso

Toda resposta 2xx do boundary SaaS traz `correlation_id` no corpo e no header `X-Correlation-Id`
(`_correlation`, `saas_router.py:122-125`). Mutacoes de estado (criar/publicar/ativar projeto,
release, plano; checkout; transicao de assinatura) tambem emitem log canonico via
`SaasHttpService._log_state_change`, com `event_name` explicito (ex.:
`saas_http.plan.status_changed`), `environment`, `tenant_id`, `project_id` e, quando aplicavel,
`saas_plan_id`/`release_id`. A tela administrativa exibe a correlacao apos cada acao — e o primeiro
lugar a olhar para reconstruir o que aconteceu.

## 13. O que acontece em caso de erro

`_translate_error` (`saas_router.py:137-144`) centraliza a traducao de excecao para HTTP:

| Excecao | HTTP | Quando acontece |
|---|---|---|
| `ReleaseConflictError`, `CommercialStateConflictError`, `IntegrityError` | 409 `state_conflict` | ativacao concorrente do ponteiro, transicao de assinatura em estado incompativel, violacao de unique |
| `ProjectReleaseNotFoundError` | 404 `not_found` | projeto/release fora do escopo tenant+ambiente |
| `CommercialContractError`, `ValueError` | 422 `invalid_contract` | plano oferecendo operacao fora da release ativa, payload semanticamente invalido |
| autenticacao ausente | 401 `authentication_required` | sessao federada ausente/expirada |
| tenant fora do membership | 403 `tenant_scope_denied` | `X-SaaS-Tenant-Id` sem membership autorizado com a permissao exigida |
| tenant nao informado | 422 `tenant_scope_required` | rota administrativa sem `X-SaaS-Tenant-Id` |

Um defeito historico relevante, ja corrigido antes desta rodada (fora do escopo P3): `GET
/api/auth/admin/memberships` retornava 500 porque `AdminMembershipEntryPayload` exigia `str` para um
campo que a query do diretorio devolvia como `uuid.UUID` sem `::text`. O padrao correto (cast
explicito na SQL) ja era usado pelo modulo SaaS (`http_repository.py`, `list_subscribers`) e foi
replicado no diretorio de memberships.

## 14. Observabilidade e diagnostico

Logs importantes: os eventos `saas_http.plan.created`, `saas_http.plan.updated`,
`saas_http.plan.status_changed` (e os equivalentes de projeto/release/assinatura ja existentes desde
T8-T13) carregam `correlation_id`, `environment`, `tenant_id`, `project_id` e o identificador do
recurso mutado. Ordem pratica de investigacao:

1. Capturar o `correlation_id` exibido na tela apos a acao que falhou ou que precisa ser auditada.
2. Abrir o log oficial dessa correlacao (`logs/<correlation_id>*.json`, via `python -m
   src.log_analyzer`).
3. Confirmar em qual camada o fluxo parou: autorizacao de tenant (`_admin_scope`), resolucao de
   projeto/release (`SaasHttpRepository`), validacao de contrato de plano (`SaasHttpService`), ou
   efeito no banco (executor central).
4. Para problemas de "cliente nao acessa o produto", cruzar a aba Auditoria do projeto (filtrando por
   `target_correlation_id` da sessao do cliente) com a aba Assinantes (status/entitlement).

## 15. Troubleshooting

### Sintoma: `403 tenant_scope_denied` ao abrir a tela SaaS

Causa provavel: o operador nao tem membership ativo com a permissao exigida
(`SAAS_PROJECTS_READ`/`MANAGE` etc.) na organizacao selecionada. Verificar em
`ui-admin-gov-human-grants.html` os grants efetivos daquele membership.

### Sintoma: plano aceita salvar mas a operacao "some" depois de trocar de release

Causa provavel: a nova release ativa nao publica mais aquela operacao. O contrato de `update_plan`
so valida no momento da edicao — trocar a release ativa depois nao corrige planos ja existentes
automaticamente.

### Sintoma: acao de mutacao (status de plano, cancelar assinante) reenviada nao muda nada

Comportamento esperado quando o `Idempotency-Key` e reaproveitado para a mesma tentativa — nao e
bug; e a garantia de nao duplicar efeito. Gerar uma chave nova para uma tentativa realmente nova.

### Sintoma: tela de tenants/usuarios pede API key mas a tela SaaS nao

Comportamento esperado — sao duas familias de autorizacao diferentes (secao 5). Nao existe hoje um
unico mecanismo de autenticacao para toda a area administrativa.

## 16. Explicacao 101

O boundary SaaS e uma "portaria unica": toda acao de administrar ou assinar um produto passa por
`saas_router.py`, que primeiro confere quem esta pedindo (sessao + membership, ou sessao do proprio
assinante) e so depois repassa para o service certo. A tela administrativa e um painel com sete
gavetas (abas) sobre o mesmo projeto; cada gaveta chama um endpoint especifico e mostra o resultado.
As telas de governanca (tenants, usuarios, memberships) sao o cadastro de "quem e quem" que a
portaria consulta antes de deixar alguem administrar um produto.

## 17. Checklist tecnico

- Confirmei que a rota que vou tocar usa `AccessMode.CUSTOM` (sessao) ou `HEADER` (API key) antes de
  escrever cliente/teste.
- Confirmei que toda escrita nova em `saas_*` passa pelo executor central
  (`PostgresQueryExecutor`), nunca `cursor.execute` direto.
- Confirmei que a operacao de plano respeita o subconjunto de operacoes da release ativa.
- Confirmei que a mutacao nova emite `_log_state_change` com `event_name` explicito.
- Confirmei que nenhuma escrita comercial nova ignora `_require_prod` sem justificativa expressa.
- Confirmei que o `environment` usado em queries novas vem de `self._repository.environment`
  (ou equivalente canonico), nunca de um literal.
- Validei a mudanca com o `correlation_id` real de uma execucao, lendo o log oficial dessa execucao.

## 18. Como colocar para funcionar

Nao ha instalacao separada: o boundary SaaS ja esta incluido em `service_api.py` e as telas ja estao
publicadas em `app/ui/static/`. Para operar:

1. Suba API + Worker + Scheduler (`./run.sh +a +w +s`), com `ENVIRONMENT` definido conforme o
   ambiente desejado — lembrando que o dominio comercial so aceita escrita em `prod` (secao 8).
2. Acesse `/ui/static/saas_index.html` (hub) ou diretamente `/admin/saas/projects` com uma sessao
   federada de um usuario com membership autorizado.
3. Siga o passo a passo da secao 10 para publicar um produto, ou use o exemplo completo em JavaScript
   da secao 11 para simular o lado do assinante contra a API real.
4. Para automatizar as telas de governanca (tenants/usuarios/segredos/canal), use uma `X-API-Key` com
   a permissao `ADMIN_*` correspondente — nunca a mesma chave usada para SaaS/memberships, que exigem
   sessao de navegador.

O tutorial operacional passo a passo (do YAML ao lancamento) esta em
`docs/usuario/TUTORIAL-CRIAR-CONFIGURAR-LANCAR-PRODUTO-SAAS.md`.

## 19. FAQ tecnica

**1. Qual router concentra as rotas SaaS?**
`src/api/routers/saas_router.py`, incluido em `service_api.py` (linha 1652 no codigo lido).

**2. A tela SaaS aceita `X-API-Key`?**
Nao. Todas as rotas de `saas_router.py` usam `AccessMode.CUSTOM` com `ui_session_required=True` —
exigem sessao federada de navegador.

**3. Como o backend sabe qual tenant o operador esta administrando?**
Pelo header `X-SaaS-Tenant-Id`, validado em `_admin_scope` contra o membership humano do operador via
`ClientDirectory.resolve_membership_authorization_context`.

**4. Onde fica o SQL do modulo?**
Em `src/saas_project/repository.py` (writer de lifecycle) e `src/saas_project/http_repository.py`
(read-models do boundary), ambos via `PostgresQueryExecutor` — nenhum `cursor.execute` cru.

**5. Como o `environment` chega nas queries do boundary?**
Via a property `SaasHttpRepository.environment`, que deriva do segregador canonico do processo, e via
`_environment_literal()` para injetar esse valor com seguranca no SQL.

**6. Por que o dominio comercial recusa operar fora de `prod`?**
`SaasCommercialService._require_prod` bloqueia checkout, transicao de assinatura e concessao de
entitlement quando `ENVIRONMENT != prod`, ate existir necessidade real de simular comercio em outro
ambiente.

**7. Onde entram as validacoes de que um plano so oferece operacoes da release ativa?**
Em `SaasHttpService.create_plan` e `update_plan` (`src/api/services/saas_http_service.py`), antes de
delegar ao repository.

**8. Que `event_name` procurar no log para uma mudanca de plano?**
`saas_http.plan.created`, `saas_http.plan.updated` ou `saas_http.plan.status_changed`, todos emitidos
por `_log_state_change`.

**9. O endpoint de auditoria (`.../audit`) e paginado?**
Sim, `limit` 1-200 e `offset`, com `total` no corpo. O endpoint de uso (`.../usage`) e agregado, sem
paginacao.

**10. Como a UI evita reenviar `limit=25&offset=0` fixo na lista de assinantes?**
Desde a rodada mais recente, `saas-products-admin.js` usa um paginador offset/limit reutilizavel
(`createOffsetPaginator`/`renderPaginator`) tambem aplicado a auditoria.

**11. Existe endpoint para editar plano e mudar seu status separadamente?**
Sim: `PATCH .../plans/{plan_id}` edita nome/operacoes/preco/moeda; `POST .../plans/{plan_id}/status`
transiciona `active/inactive/archived`. Sao dois endpoints, nao um so.

**12. O checkout precisa de `Idempotency-Key`?**
Sim, assim como as demais rotas de transicao de assinatura — sem esse header a request e rejeitada
pela validacao do FastAPI (`Header(..., min_length=1, max_length=256)`).

**13. Onde investigar um 500 legado de memberships?**
No diretorio (`ClientDirectory`/`auth_router.py::list_admin_memberships`) — o defeito de UUID sem
`::text` ja foi corrigido; se reaparecer em outro campo do diretorio, o padrao de correcao e o mesmo
usado no modulo SaaS.

**14. As telas de governanca de tenants/usuarios usam o mesmo cliente HTTP que a tela SaaS?**
Nao integralmente. A tela SaaS usa `PrometeuAdminApiClient` (`adminFetchJson`) de forma consistente;
as telas de governanca usam `adminFetch` com padroes mais heterogeneos entre si — convergencia
registrada como melhoria futura, fora do escopo desta rodada.

**15. Qual arquivo documenta o dicionario de colunas das tabelas `saas_*`?**
`docs/tecnico/README-SCHEMA-BANCO.md`, secao "Estado aplicado do modulo SaaS" — este manual nao o
duplica.

**16. Onde ver a explicacao conceitual (nao tecnica) deste modulo?**
Em `docs/conceitual/README-CONCEITUAL-GESTAO-SAAS-TENANT.md`.

## 20. Evidencias no codigo

- `src/api/routers/saas_router.py` — boundary unico; `_session`/`_admin_scope`/`_correlation`/
  `_translate_error`; todas as rotas administrativas e de assinante (linhas 147-868 no codigo lido).
- `src/api/services/saas_http_service.py` — `create_plan`/`update_plan`/`set_plan_status`
  (linhas 274-381), `_log_state_change` (linha 850), `audit_events`/`usage_metrics` (linhas 623/681).
- `src/saas_project/http_repository.py` — `environment` property (linhas 25-29),
  `_environment_literal` (linha 31), `list_admin_tenants`/`list_subscribers` com cast `::text`.
- `src/saas_commercial/service.py` — `_require_prod` (linha 343 e chamadas em 60/114/174/260/325).
- `src/api/security/permissions.py` — `PermissionKeys.SAAS_*` (linhas 115-119),
  `PERMISSION_CATALOG` (linhas 564-587).
- `src/api/security/permission_metadata.py` — `AccessMode`, `endpoint_permission` (default
  `AccessMode.HEADER`).
- `src/api/routers/auth_router.py` — `list_admin_memberships` (linha 2415), sem
  `@endpoint_permission`, sessao validada manualmente.
- `src/api/routers/admin/tenants_router.py`, `src/api/routers/admin/users_router.py` — rotas de
  governanca via `X-API-Key` (`ADMIN_TENANTS_*`, `ADMIN_USERS_*`, `ADMIN_CHANNELS_*`).
- `src/api/schemas/saas_models.py` — `SaasApiModel` (`extra="forbid"`),
  `SaasPlanUpdateRequest`/`SaasPlanStatusRequest`, `SaasAuditEventPayload`,
  `SaasUsageMetricPayload`, `PublicProjectPayload`, `MyProjectPayload`.
- `app/ui/static/js/saas-products-admin.js` — sete abas, `formatMoney`, `createOffsetPaginator`,
  dependencia obrigatoria de `window.prometeuConfirmar`.
- `app/ui/static/js/shared/admin-area-navigation-catalog.js` — `saasActions`/`governancaActions`
  (catalogo dos dois hubs).
- `docs/.interno/.planos/saas-tenant-auditoria/investigacao--2026-07-15--06-42-05--saas-tenant-auditoria.md`
  e `docs/.interno/.planos/saas-tenant-auditoria/plano--2026-07-15--08-23-25--saas-tenant-auditoria-p3.md`
  — origem forense de todas as afirmacoes sobre estado anterior/posterior desta rodada.
