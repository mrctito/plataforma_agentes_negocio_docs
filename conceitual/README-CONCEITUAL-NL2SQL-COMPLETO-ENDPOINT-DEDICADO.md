# Manual detalhado da etapa: Endpoint dedicado do NL2SQL

## 1. O que esta etapa faz

Esta etapa recebe a pergunta em linguagem natural e transforma esse pedido em um contrato HTTP administrativo estavel. Ela existe para manter NL2SQL como slice proprio, separado de dyn_sql, separado de /agent/execute e separado de fluxos improvisados de prompt livre.

Em linguagem simples: e a porta oficial do produto para pedir uma SQL proposta.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa vive no router com prefixo /config/nl2sql. Ela roda antes do servico dedicado e antes da engine schema_rag_sql. Tudo que entra no fluxo passa primeiro por esse boundary.

## 3. O que entra e o que sai

Entradas confirmadas:

- prompt obrigatorio
- user_email obrigatorio
- dialect obrigatorio entre postgresql, mysql e mssql
- uma fonte explicita de YAML via yaml_config, yaml_config_path, yaml_inline_content ou encrypted_data
- top_k entre 1 e 20
- correlation_id opcional

Saidas confirmadas:

- Nl2SqlResponse tipado
- correlation_id efetivo para rastreamento
- erro 400 quando o contexto YAML ou o contrato de entrada esta invalido
- erro 500 quando ha falha interna inesperada

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. o router declara um prefixo proprio em /config/nl2sql;
2. exige permissao config.generate e rate limit administrativo;
3. resolve correlation_id a partir do payload, do request state ou de uma geracao nova;
4. resolve um rótulo de usuario so para logging e auditoria;
5. se yaml_config vier no payload, aceita esse dicionario como contexto em memoria;
6. caso contrario, usa o resolvedor compartilhado de YAML para abrir yaml_config_path, yaml_inline_content ou encrypted_data;
7. chama o Nl2SqlService com prompt, user_email, yaml_config, dialect, top_k e source_hint;
8. devolve a resposta estruturada do servico;
9. traduz ValueError para HTTP 400 e excecao inesperada para HTTP 500.

O detalhe mais importante e que esse boundary nao tenta executar SQL nem deduzir contexto escondido. Ele apenas normaliza a entrada para o slice dedicado.

## 5. Decisoes tecnicas importantes

### 5.1. NL2SQL tem rota propria

Manter o slice em /config/nl2sql/generate evita confundir geração assistida de SQL com execução agentic genérica ou com tool dyn_sql.

### 5.2. Fonte de YAML continua explicita

O router aceita varias formas de entregar o YAML, mas nao inventa uma configuracao implicita. Isso preserva governanca e rastreabilidade.

### 5.3. Correlation ID nasce cedo

Como o correlation_id ja e resolvido no boundary, todo o restante do fluxo pode logar e diagnosticar a mesma execucao de ponta a ponta.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- yaml_config informado com tipo diferente de dicionario gera erro de contexto;
- nenhuma fonte valida de YAML impede o fluxo de seguir;
- dialect fora do contrato nao passa do request tipado ou da normalizacao posterior;
- excecao inesperada no servico vira HTTP 500 com mensagem controlada.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- log de inicio do fluxo dedicado com user e top_k;
- source_hint resolvido para o contexto YAML;
- erro 400 explicando falha de contexto;
- correlation_id retornado no payload final.

Em linguagem simples: se a pergunta nem entra direito na esteira, o problema costuma estar aqui.

## 8. Exemplo pratico guiado

Cenario: uma tela administrativa quer gerar SQL revisavel para a base financeira de um ERP.

1. a UI envia prompt, user_email, dialect e yaml_config_path;
2. o router resolve o YAML daquele runtime;
3. gera ou herda correlation_id;
4. chama o servico dedicado;
5. devolve uma resposta tipada com SQL proposta, diagnosticos e execution_context.

O valor desta etapa e dar ao produto um ponto unico e previsivel para NL2SQL.

## 9. Evidencias no codigo

- src/api/routers/config_nl2sql_router.py
  - Simbolo relevante: router com prefixo dedicado de configuracao NL2SQL
  - Comportamento confirmado: boundary proprio em /config/nl2sql.
- src/api/routers/config_nl2sql_router.py
  - Simbolo relevante: generate_nl2sql
  - Comportamento confirmado: rota publica de geração assistida de SQL com resposta tipada.
- src/api/routers/config_nl2sql_router.py
  - Simbolo relevante: resolvedor de correlation_id e de payload YAML
  - Comportamento confirmado: entrada normalizada antes de chamar o servico.
- src/api/schemas/nl2sql_models.py
  - Simbolo relevante: Nl2SqlRequest e Nl2SqlResponse
  - Comportamento confirmado: contrato HTTP estavel do slice dedicado.
