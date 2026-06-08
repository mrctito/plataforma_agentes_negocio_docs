# Manual detalhado da etapa: Borda HTTP dedicada do AG-UI

## 1. O que esta etapa faz

Esta etapa expõe o boundary HTTP proprio do AG-UI. Ela existe para separar discovery, execucao e replay das rotas legadas de agente, workflow ou telas administrativas.

Em linguagem simples: e a porta oficial pela qual uma interface AG-UI entra no sistema.

## 2. Onde ela entra no fluxo

No codigo lido, essa camada fica no router com prefixo /ag-ui. E ela que recebe o request autenticado, resolve correlation_id, monta o contexto de execucao, aciona o orchestrator e devolve o stream SSE.

## 3. O que entra e o que sai

Entradas confirmadas:

- GET /ag-ui/capabilities com executionKind opcional
- POST /ag-ui/runs com AgUiRunRequest oficial
- GET /ag-ui/runs/{run_id}/events
- GET /ag-ui/threads/{thread_id}/events

Em linguagem simples: o endpoint canônico e `/ag-ui/runs`. Nao existe mais URL publica por `agent_id`, o que evita manter um segundo caminho executavel e um segundo discovery no produto.

Saidas confirmadas:

- menu publico de capabilities AG-UI
- stream text/event-stream com eventos AG-UI
- replay de eventos persistidos por run ou por thread
- header X-Correlation-Id no stream de execucao

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. o router e declarado com prefixo /ag-ui;
2. todas as rotas usam permissao AGENT_EXECUTE e rate limit canônico;
3. list_ag_ui_capabilities consulta o service de discovery e falha fechado para execution_kind desconhecido;
4. run_ag_ui recebe `AgUiRunRequest`, exige fonte de configuracao explicita, deriva o runtime pelo YAML e monta o contexto de execucao;
5. _resolve_correlation_id herda o correlation_id do request state ou gera um novo;
6. o router so monta AgUiRunContext e chama o orchestrator no endpoint publico canônico;
7. cada evento vira SSE pelo encoder apenas no endpoint público oficial;
8. os endpoints de replay consultam o provider canônico do event store e devolvem resposta tipada.

O detalhe mais importante e que o router nao tenta deduzir configuracao nem redirecionar para rotas antigas. O boundary AG-UI existe justamente para ser unico e explicito.

## 5. Decisoes tecnicas importantes

### 5.1. Discovery e execucao estao no mesmo boundary

Capabilities, run e replay convivem no mesmo slice. Isso simplifica integracao para a UI e deixa claro onde comeca e termina o contrato AG-UI.

### 5.2. Falha fechada para configuracao ausente

POST /ag-ui/runs exige uma fonte de configuracao explicita no corpo. O backend resolve a configuracao da Plataforma de Agentes de IA a partir de `yaml_config`, `yaml_inline_content` ou `encrypted_data`, autentica o tenant e deriva o runtime pelo YAML. Isso evita que uma aplicacao terceira precise inventar selecao paralela por `agent_id`.

Em linguagem simples: primeiro existe uma configuracao governada. Depois a aplicacao externa chama o boundary canônico com essa fonte de configuracao. O browser nao deve decidir runtime por URL.

O bloqueio tambem vale para campos aninhados. Se alguem tentar esconder `yaml_config`, `yamlPath`, `encryptedData`, `tenantId`, `securityKeys` ou `toolsLibrary` dentro de `input` ou `metadata`, o backend rejeita o request.

O boundary publico nao oferece mais caminho alternativo por `agent_id`. Isso remove duplicidade de contrato e elimina a chance de o cliente tratar a URL como seletor de runtime.

### 5.3. Correlation ID sai no header do stream

O router devolve X-Correlation-Id ja na resposta SSE. Isso ajuda suporte e frontend a amarrarem a execucao ao replay e aos logs.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- executionKind desconhecido em /capabilities devolve erro explicito;
- request publico com YAML bruto, executionKind, tenant, toolsLibrary ou segredo no body e rejeitado com 400;
- request publico com chave interna aninhada em `state`, `context`, `tools`, `messages` ou `forwardedProps` e rejeitado com 400;
- request publico com frontend tool sem allowlist, schema divergente, SQL, DSN ou connection string e rejeitado com 400;
- runtime baseado em YAML sem `yaml_path` cadastrado para o tenant ou access key e rejeitado com erro claro;
- supervisor classico legado e rejeitado com erro de deprecacao;
- request legado para `/ag-ui/runs` e rejeitado com 410;
- falta de permissao impede acesso ao boundary;
- se o event store canônico estiver mal configurado, o replay nao funciona;
- frontends que tratem AG-UI como GET SSE simples falham, porque aqui a execucao nasce via POST.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- log de capabilities solicitado com execution_kind e usuario;
- log de run recebido com execution_kind, thread_id, run_id e user;
- ausencia do header X-Correlation-Id quando o problema esta antes do stream abrir;
- erro 400 dizendo que o endpoint publico nao aceita YAML, executionKind, tenant_id ou segredos no body;
- erro 400 dizendo que o `RunAgentInput` publico contem campos internos nao permitidos;
- erro 400 dizendo que a frontend tool nao possui allowlist publica ou nao segue o schema permitido;
- erro 400 orientando a cadastrar `yaml_path` quando o agent_id depende de YAML do tenant;
- erro 410 dizendo que a rota legada foi encerrada e que o sucessor e `/ag-ui/runs`;
- erro 404 ao filtrar capabilities por execution_kind inexistente.

Em linguagem simples: se a UI nao consegue nem entrar direito no slice, o problema costuma estar nesta borda.

## 8. Exemplo pratico guiado

Cenario: uma pagina PDV quer abrir um assistente governado.

1. a pagina chama /ag-ui/capabilities para descobrir o menu publico disponivel;
2. escolhe retail_demo e monta payload compativel com a capability;
3. deixa `tools` vazio, exceto se a capability publicar frontendTools com schema permitido;
4. envia POST para /ag-ui/runs com a fonte de configuracao explicita;
5. recebe SSE com X-Correlation-Id;
6. se precisar auditar depois, consulta /ag-ui/runs/{run_id}/events.

Para um DeepAgent ou Workflow configurado por YAML, o passo anterior ao item 1 e administrativo: obter ou montar a fonte de configuracao governada que sera enviada ao boundary canônico. O run publico nao escolhe runtime por URL.

O valor desta etapa e transformar AG-UI em surface oficial de produto, e nao em adaptacao improvisada sobre endpoints antigos.

## 9. Evidencias no codigo

- src/api/routers/ag_ui_router.py
  - Simbolo relevante: router = APIRouter(prefix="/ag-ui")
  - Comportamento confirmado: boundary HTTP dedicado do slice AG-UI.
- src/api/routers/ag_ui_router.py
  - Simbolo relevante: list_ag_ui_capabilities
  - Comportamento confirmado: discovery com falha fechada para execution_kind desconhecido.
- src/api/routers/ag_ui_router.py
  - Simbolo relevante: run_ag_ui
  - Comportamento confirmado: bloqueio explícito da rota legada com `410 Gone` e header de migração.
- src/api/routers/ag_ui_router.py
  - Simbolo relevante: replay_ag_ui_run_events e replay_ag_ui_thread_events
  - Comportamento confirmado: replay de auditoria por run e por thread.
