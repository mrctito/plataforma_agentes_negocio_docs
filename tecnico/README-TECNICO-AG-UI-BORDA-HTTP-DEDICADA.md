# Manual técnico por etapa: borda HTTP dedicada do AG-UI

## 1. O que esta etapa cobre

Esta etapa cobre o boundary FastAPI do slice AG-UI. Ela é responsável por expor discovery público de capabilities, iniciar runs por SSE e reproduzir eventos persistidos por run ou thread.

Em termos operacionais, esta é a porta oficial do produto para AG-UI. Se a borda estiver errada, o resto do runtime nem chega a ser exercitado.

## 2. Superfícies públicas confirmadas

O router dedicado expõe quatro rotas:

- GET /ag-ui/capabilities
- POST /ag-ui/runs
- GET /ag-ui/runs/{run_id}/events
- GET /ag-ui/threads/{thread_id}/events

Todas usam a permissão AGENT_EXECUTE e rate limit da família agent.

POST /ag-ui/runs e o boundary público alvo para terceiros. Ele recebe `AgUiRunRequest`, exige fonte de configuracao explicita por `yaml_config`, `yaml_inline_content` ou `encrypted_data`, deriva o runtime pelo YAML e bloqueia envio de segredos ou selecao paralela por `agent_id`.

Esse bloqueio e estrutural. O backend rejeita chaves internas nao apenas no topo do JSON, mas tambem quando elas aparecem escondidas em `input`, `metadata` ou outros blocos aninhados. Na pratica, `yaml_config`, `yamlPath`, `yamlInlineContent`, `encryptedData`, `tenantId`, `securityKeys` e `toolsLibrary` nao podem viajar de forma disfarçada como se fossem dado funcional comum.

Nao existe mais rota publica por `agent_id`, nem para execucao nem para capabilities. Isso elimina identidades paralelas de URL e mantém discovery e run no mesmo boundary YAML-first.

## 3. Como o router trata discovery

GET /ag-ui/capabilities recebe executionKind opcional e delega a descoberta ao serviço de capabilities. O comportamento confirmado é falha fechada para executionKind desconhecido e resposta pública sem SQL, DSN ou segredos.

O mesmo discovery agora concentra também os metadados AG-UI necessários para frontend tools, exemplos, `UISpecs` e resume. O produto nao mantém um segundo endpoint público so para projetar capabilities por `agent_id`.

Na prática, isso separa claramente duas responsabilidades:

- discovery informa o que pode ser feito e quais metadados AG-UI o frontend precisa entender
- run executa o que foi escolhido

## 4. Como o router trata execução

POST /ag-ui/runs faz oito validacoes centrais antes de abrir o stream:

1. exige autenticação e permissão
2. exige uma fonte de configuracao explicita no body
3. exige body no formato `AgUiRunRequest`
4. deriva o execution kind pelo YAML, falhando fechado em ambiguidade
5. rejeita `executionKind` divergente quando ele vier no body
6. autentica e autoriza com base na configuracao resolvida
7. monta `AgUiRunContext` sem depender de `agent_id`
8. injeta um encoder SSE sobre o orquestrador

O retorno é StreamingResponse com media_type text/event-stream, header X-Correlation-Id e Cache-Control igual a no-cache.

Essa separação evita que a superfície pública nova carregue YAML sensível ou campos internos de runtime.

## 5. Como o router trata replay

As rotas de replay recebem um AgUiEventStore canônico e devolvem AgUiReplayResponse já ordenado. O boundary não reimplementa ordenação, sanitização nem storage local. Ele apenas consome a porta de replay configurada pelo provider canônico.

Isso importa porque replay e execução continuam usando o mesmo contrato público, mas responsabilidades diferentes.

## 6. Decisões técnicas importantes

### 6.1. AG-UI usa rota própria

Executar AG-UI em /ag-ui/runs evita acoplamento com identidade paralela por `agent_id` e deixa explicito que o contrato publico e YAML-first, nao uma resposta JSON arbitraria nem uma URL por runtime.

### 6.2. Resume reutiliza o mesmo endpoint

O boundary não cria rota separada de continue. Isso simplifica a superfície pública e mantém todo run AG-UI no mesmo contrato.

Na pratica, isso significa o seguinte: integracoes publicas novas iniciam e retomam no mesmo `POST /ag-ui/runs`, sem alias, sem URL paralela por runtime e sem compatibilizacao por `agent_id`.

### 6.3. Configuração sensível fica fora do payload público

No endpoint publico, a configuracao da Plataforma de Agentes de IA e resolvida no backend a partir de tenant, access key e da fonte de configuracao explicita. O cliente externo confiavel pode enviar `yaml_config`, `yaml_inline_content` ou `encrypted_data` para o boundary canônico. O browser publico nao deve expor `security_keys`, `tools_library`, DSN, SQL livre nem credenciais.

### 6.4. Provisioning e execução são fluxos separados

Provisioning e a etapa administrativa de cadastrar ou confirmar qual YAML pertence a uma access key, sessao ou tenant. Execucao e a etapa publica em que o cliente chama `/ag-ui/runs` com `AgUiRunRequest` oficial.

Essa separação evita que um sistema terceiro carregue configuração sensível a cada chamada. Se o tenant ainda não tiver `yaml_path` cadastrado para runtimes YAML-backed, o endpoint público falha com erro claro antes de iniciar o stream. Capability packs registrados, como `retail_demo`, continuam podendo rodar sem YAML enviado pelo cliente porque já são governados pelo backend.

### 6.5. Rotas publicas por agent_id foram removidas

O boundary publico nao expõe mais execucao nem discovery por `agent_id`. Isso elimina o risco de manter uma segunda superficie viva por conveniencia ou por compatibilidade indefinida.

### 6.6. Frontend tools não são platform tools

`frontend_tools` são ferramentas de interface declaradas no contrato AG-UI. Elas servem para a UI expor ações visuais permitidas ao protocolo, como abrir um painel ou pedir uma confirmação controlada.

`platform_tools` são as tools internas da plataforma, resolvidas pelo YAML e pelo catálogo governado do backend. Elas podem envolver LangChain, ToolsFactory, RAG, banco vetorial ou integrações de infraestrutura. O browser nunca escolhe essas tools diretamente.

Na prática, o boundary público só aceita `RunAgentInput.tools` quando a capability já publicou a mesma tool em `frontendTools`. A decisão fica registrada em log com agent_id, execution_kind, capability, nomes aceitos ou rejeitados e motivo.

## 7. Erros típicos da borda

Os erros mais úteis desta etapa são:

- 404 quando alguem ainda chama uma URL publica inexistente fora do boundary canônico
- 400 quando o endpoint publico nao recebe `yaml_config`, `yaml_inline_content` nem `encrypted_data`
- 400 quando o endpoint publico recebe `executionKind` divergente, segredo ou payload invalido
- 400 quando `input` ou `metadata` tentam carregar campo interno, segredo ou seletor de runtime fora do contrato
- 400 quando runtime YAML-backed não tem `yaml_path` cadastrado no tenant ou access key
- 404 no discovery para executionKind inexistente
- falha de autenticação ou permissão antes de chegar ao orquestrador
- erro de consumo SSE no cliente quando o stream não respeita o contrato

## 8. Diagnóstico recomendado

Para investigar esta etapa:

1. confirme a permissão AGENT_EXECUTE no boundary
2. para o endpoint público, valide se o tenant/access key possui `yaml_path` cadastrado quando o runtime depender de YAML
3. valide se o YAML informado realmente seleciona um runtime suportado
4. confira se `input` e `metadata` nao tentam transportar segredo, runtime alternativo ou contrato paralelo
5. confirme se o cliente usa apenas `/ag-ui/runs` e `/ag-ui/capabilities`, sem URL paralela por runtime
6. confira o header X-Correlation-Id na abertura do stream
7. verifique se o replay consulta o provider canônico, não memória local improvisada

## 9. Evidências no código

- src/api/routers/ag_ui_router.py
  - Motivo: boundary HTTP do slice.
  - Comportamento confirmado: discovery, execução SSE e replay vivem em rotas próprias sob /ag-ui.
- src/api/routers/ag_ui_router.py
  - Motivo: contrato público consolidado.
  - Comportamento confirmado: a OpenAPI publica discovery, run e replay apenas no boundary canônico, sem rotas por `agent_id`.
- src/api/services/ag_ui_public_run_resolver.py
  - Motivo: resolução segura do boundary público.
  - Comportamento confirmado: bloqueia YAML bruto, executionKind, tenant_id e segredos no payload público.
- src/api/routers/ag_ui_router.py
  - Motivo: validação de fonte de configuração.
  - Comportamento confirmado: a rota canônica falha com 400 quando o payload não traz yaml_config, yaml_inline_content ou encrypted_data.
- src/api/routers/ag_ui_router.py
  - Motivo: contrato do stream.
  - Comportamento confirmado: a execução responde como text event stream e devolve correlation id no header.
