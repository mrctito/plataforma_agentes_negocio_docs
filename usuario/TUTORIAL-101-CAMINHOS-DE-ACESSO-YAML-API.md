# Tutorial 101 — Os 3 Caminhos de Acesso à API (YAML direto, API Key e Projeto SaaS)

> Público: consultor/integrador iniciando na plataforma. Nível 101: o que é cada caminho,
> quando usar, como funciona por baixo e erros a evitar. Modelo de dados completo:
> [README-SCHEMA-BANCO.md](../tecnico/README-SCHEMA-BANCO.md) (seções indicadas ao longo
> do texto). Exemplos de código por linguagem:
> [README-EXEMPLOS-INTEGRACAO-API.md](README-EXEMPLOS-INTEGRACAO-API.md).

## A ideia central (leia antes dos caminhos)

A plataforma é **YAML-First**: toda execução (pergunta RAG, agente, ingestão, ETL) precisa
de um YAML de configuração que diz qual base consultar, qual modelo usar e como se
comportar. Os "3 caminhos" são apenas **três formas diferentes de o servidor obter esse
YAML** quando um request chega. A pergunta que o boundary faz a cada request é: *"de onde
vem o YAML desta execução?"*

| Caminho | Quem fornece o YAML | Para quem serve |
|---|---|---|
| 1. YAML no request | O próprio cliente envia (cifrado ou inline) | Desenvolvimento, testes, integrações internas que gerenciam o próprio YAML |
| 2. API key → YAML direto | O servidor busca na tabela `tenant_yaml` pela chave | Transição/rollback da migração para o modelo de projeto (não use para integração nova) |
| 3. API key → Projeto SaaS → Release | O servidor resolve projeto → release ativa → YAML imutável | Clientes externos e produção governada (**caminho recomendado**) |

Nos três casos o cliente pode mandar o header `X-API-Key`; o que muda é se o YAML viaja
no request ou vive governado no banco.

---

## Caminho 1 — O cliente envia o YAML no request

**O que é:** o request carrega o YAML — cifrado (`encrypted_data`, via `/crypto/session-key`
+ Fernet + RSA-OAEP), inline (`yaml_inline_content`/`yaml_config_dict`) ou por caminho de
arquivo local (`yaml_config_path`, só em desenvolvimento).

**Quando usar:** você controla o YAML do seu lado — desenvolvimento, testes de configuração,
ferramentas internas. É também o modo que as UIs da plataforma usam (o browser envia o YAML
carregado na tela).

**Como funciona por baixo:** o boundary único de resolução
(`resolve_yaml_configuration`) decifra/aceita o YAML recebido, injeta contexto de
tenant e, quando o YAML não trouxe `security_keys`, tenta materializá-las no servidor antes de
expandir placeholders. O caminho canônico para browser e integrações é enviar somente a
configuração e manter credenciais no backend.

Limite atual importante: o boundary ainda aceita um YAML explícito que já contenha
`security_keys`; nesse caso ele preserva o bloco e não promete rejeição nem reinjeção pelo
diretório. Portanto, “o cliente nunca envia segredos” é uma regra de integração segura, não um
guardrail garantido pelo parser atual. Não coloque chaves reais em JavaScript, log, exemplo ou
payload de navegador.

**Erro a evitar:** enviar YAML explícito **junto** com uma chave que tem binding esperando
que o binding valha — YAML explícito sempre tem precedência; e em operação ligada a projeto
SaaS o YAML alternativo é **rejeitado** (a release publicada é a única fonte).

Passo a passo com código: seção "Entendendo a Criptografia" do
[README-EXEMPLOS-INTEGRACAO-API.md](README-EXEMPLOS-INTEGRACAO-API.md).

---

## Caminho 2 — API key com binding direto para `tenant_yaml`

**O que é:** o YAML fica publicado na tabela `tenant_yaml` e uma chave
(`tenant_access_keys` com `key_kind='tenant_yaml'`) aponta direto para uma linha dessa
tabela. O cliente só manda `X-API-Key`; o servidor carrega o `yaml_content` do banco.

**Modelo de dados:** seção **"2. Catálogo YAML, canais e API keys"** e o dicionário da
tabela **`tenant_yaml`** em [README-SCHEMA-BANCO.md](../tecnico/README-SCHEMA-BANCO.md).
Pontos-chave do desenho:

- `tenant_yaml` guarda os YAMLs **versionados** de cada tenant (`yaml_content` é a fonte de
  runtime; `yaml_path` é só proveniência);
- a FK composta `environment + tenant_id + tenant_yaml_id` impede uma chave apontar para
  YAML de outro tenant ou ambiente;
- linha publicada referenciada por release é **imutável** (trigger
  `trg_released_tenant_yaml_immutable`): mudar configuração = publicar **nova linha** e
  religar a chave, nunca editar a publicada;
- o banco guarda só o **hash SHA-256** da chave (formato `pk_<ambiente>_<aleatório>`).

**Estado atual (importante):** este caminho está em **transição para o modelo de projeto**
(caminho 3). Os endpoints principais de execução (`ask`, `ingest`, `etl`) já declaram a
operação SaaS e por isso exigem que a chave tenha binding de **projeto** — uma chave apenas
com binding direto de `tenant_yaml` recebe `409` com
`"API key sem binding SaaS para a operação solicitada"` (comportamento comprovado em
produção). O binding direto permanece como mecanismo de rollback da migração. **Para
integração nova, use o caminho 3.**

---

## Caminho 3 — API key → Projeto SaaS → Release ativa (recomendado)

**O que é:** a chave aponta para um **projeto** (`tenant_access_keys.saas_project_id` +
`operation`). O servidor resolve: projeto → **release ativa** → `tenant_yaml` congelado da
release. O cliente continua mandando só `X-API-Key`.

**Modelo de dados:** seção **"3. Projeto SaaS, release e manifesto"** em
[README-SCHEMA-BANCO.md](../tecnico/README-SCHEMA-BANCO.md). O desenho em uma frase por
tabela:

- `saas_projects` — o produto publicável de um tenant (tenant 1:N projetos);
- `saas_project_releases` — cada release congela **exatamente um** `tenant_yaml` do mesmo
  tenant/ambiente, com `yaml_hash` e `manifest_json` imutáveis após publicada;
- `saas_project_active_releases` — o ponteiro único de qual release está no ar;
- trocar a configuração = nova release + mover o ponteiro (rollback = voltar o ponteiro).

**Regra da operação:** cada chave carrega **uma** operação (`rag`, `agent`, `ingest` ou
`etl`) e ela precisa bater com a operação do endpoint chamado (o `ask` declara `rag`).
A operação também precisa estar publicada no manifesto da release.

**Exemplo (pergunta RAG):**

```bash
curl -X POST "https://SUA-URL-DA-API/rag/execute" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: pk_prod_SUA_CHAVE" \
  -d '{"operation":"ask","payload":{"question":"...","user_email":"usuario@empresa.com"}}'
```

Nenhum campo de YAML no payload — é a ausência de YAML explícito + a chave com binding de
projeto que aciona a resolução governada.

**Por que este caminho é o recomendado:** o cliente externo nunca recebe YAML; a
configuração é imutável e auditável por release; trocar versão não exige redistribuir
chave; e o hash da release garante que o que está no ar é exatamente o que foi publicado.

---

## Erros comuns (os 3 caminhos)

| Sintoma | Causa | Caminho |
|---|---|---|
| `401` "Cabeçalho X-API-Key é obrigatório" | Header ausente | todos |
| `403` chave inválida | Chave errada, revogada, expirada ou de outro ambiente (`pk_dev_...` em prod) | 2 e 3 |
| `409` "API key sem binding SaaS para a operação solicitada" | Chave sem `saas_project_id`/`operation`, ou operação da chave ≠ operação do endpoint | 3 |
| `422` "não aceita YAML alternativo" | Enviou YAML explícito numa operação ligada a projeto | 1×3 |
| `400` "Falha ao interpretar o YAML governado" | O YAML da release ativa está defasado do contrato vigente (ex.: chave removida do produto) — publicar novo `tenant_yaml` e nova release | 2 e 3 |

## Onde aprofundar

- Referência de cada tela administrativa que publica/ativa/religa (Caminhos 2 e 3), diagrama do
  modelo de chaves de API e passo a passo de um incidente real de configuração defasada:
  [TUTORIAL-101-CICLO-DE-VIDA-YAML-POR-CLIENTE.md](TUTORIAL-101-CICLO-DE-VIDA-YAML-POR-CLIENTE.md)
- Modelo de dados completo (tabelas, FKs, invariantes, lifecycle de release):
  [README-SCHEMA-BANCO.md](../tecnico/README-SCHEMA-BANCO.md)
- Gestão SaaS × Tenant (telas admin, endpoints de projeto/release/plano):
  [README-TECNICO-GESTAO-SAAS-TENANT.md](../tecnico/README-TECNICO-GESTAO-SAAS-TENANT.md)
- Exemplos de integração por linguagem e criptografia do caminho 1:
  [README-EXEMPLOS-INTEGRACAO-API.md](README-EXEMPLOS-INTEGRACAO-API.md)
- Criar e lançar um produto SaaS ponta a ponta:
  [TUTORIAL-CRIAR-CONFIGURAR-LANCAR-PRODUTO-SAAS.md](TUTORIAL-CRIAR-CONFIGURAR-LANCAR-PRODUTO-SAAS.md)
