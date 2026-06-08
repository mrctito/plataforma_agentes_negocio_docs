# Manual técnico por etapa: domínio varejo demo do AG-UI

## 1. O que esta etapa cobre

Esta etapa cobre o adapter retail_demo, que é a superfície de domínio mais rica do slice AG-UI atual. Ela combina capabilities fechadas, catálogo governado de query, validação estrita de parâmetros e caminho especial para dashboard dinâmico.

O capability pack publico do acelerador e `retail_demo`. A execucao oficial acontece por `POST /ag-ui/runs`, com `AgUiRunRequest` no corpo da requisicao e selecao governada do dominio pelo YAML e pelos metadados do payload.

## 2. Configuração obrigatória

O runtime resolve DATABASE_VAREJO_DSN e DATABASE_VAREJO_SCHEMA do ambiente. Se qualquer um faltar, o adapter falha com erro explícito de configuração incompleta.

Isso importa porque o browser não escolhe conexão nem injeta DSN. A conexão é sempre resolvida no backend, por ambiente governado.

As demos administrativas enviam `user_email`, capability, parâmetros de negocio e uma fonte YAML explicita quando o layout mestre exige isso. O browser nunca envia DSN, query ou secret. O backend usa o YAML governado e `metadata.capabilityPackId=retail_demo` para escolher o dominio governado.

## 3. Capabilities públicas confirmadas

O catálogo público expõe:

- sales_summary
- checkout_funnel
- catalog_opportunities
- customer_segments
- dashboard_dynamic

Cada capability vem acompanhada de descrição, parâmetros esperados e, quando aplicável, ui specs úteis para renderização.

## 4. Segurança do catálogo de queries

O adapter protege o domínio de três formas complementares:

- mantém catálogo fechado por capability
- valida que cada query aprovada tenha exatamente uma instrução SELECT
- rejeita chaves e campos que tentem introduzir SQL livre no payload

As chaves livres bloqueadas incluem sql, raw_sql, sql_query e statement. Essa filtragem é recursiva para evitar bypass por objetos aninhados.

## 5. Dashboard dinâmico

Quando a capability é dashboard_dynamic, o adapter não segue o fluxo simples de uma query. Ele desvia para o serviço de materialização de dashboard e passa a emitir custom events específicos do processo, além de snapshots de estado com status materializing, ready ou validation_failed.

Esse comportamento separa dois casos técnicos diferentes:

- consulta governada simples em dyn_sql
- construção validada de uma DashboardSpec renderizável

## 6. Erros típicos do domínio

Os erros mais relevantes desta etapa são:

- capability não permitida
- parâmetros fora do catálogo aprovado
- configuração de banco incompleta
- widget ou data source inválido no dashboard dinâmico
- tentativa de enviar SQL livre ou segredo no payload

## 7. Diagnóstico recomendado

Para investigar problemas nesta etapa:

1. confirme se a capability existe no catálogo público
2. valide se os parâmetros enviados batem com os nomes aprovados
3. cheque se DATABASE_VAREJO_DSN e DATABASE_VAREJO_SCHEMA existem no ambiente
4. para dashboard, confira o último status do retailDashboard no snapshot de estado
5. inspecione custom events de validação e materialização

## 8. Adaptação para outro ERP ou PDV

Para criar um acelerador parecido para outro sistema, o caminho seguro é criar um capability pack proprio, registrar esse dominio no backend e manter a fronteira do browser pequena.

O frontend deve enviar somente:

- `threadId` e `runId`
- `user_email`
- `input` com capability e parametros de negocio
- `metadata` operacional de tela e `capabilityPackId`
- uma fonte explicita de configuracao (`yaml_inline_content`, `yaml_config` ou `encrypted_data`, conforme o boundary usado)

O backend deve concentrar:

- resolução de DSN e secrets
- catálogo de queries ou procedures aprovadas
- validação read-only quando usar SQL
- validação de parâmetros
- emissão dos eventos AG-UI oficiais

Em termos práticos: a tela pergunta por uma capacidade de negócio; o servidor decide a query ou procedure segura. Isso evita que um cliente web consiga trocar conexão, mandar SQL livre ou escolher um fluxo interno não autorizado.

## 9. Evidências no código

- src/api/services/ag_ui_retail_demo_adapter.py
  - Motivo: adapter governado de varejo.
  - Comportamento confirmado: retail_demo resolve capabilities fechadas e não aceita SQL livre do browser.
- src/api/services/ag_ui_retail_demo_adapter.py
  - Motivo: segurança do catálogo.
  - Comportamento confirmado: queries aprovadas são validadas como uma única instrução SELECT.
- src/api/services/ag_ui_retail_demo_adapter.py
  - Motivo: caminho especial de dashboard.
  - Comportamento confirmado: dashboard_dynamic usa fluxo de materialização dedicado e emite eventos próprios de domínio.
- app/ui/static/js/shared/ag-ui-retail-demo-page.js
  - Motivo: controller público das telas do acelerador.
  - Comportamento confirmado: monta `AgUiRunRequest` para `/ag-ui/runs`, com `capabilityPackId`, fonte YAML explicita e sem DSN, SQL ou `correlation_id` criado no browser.
