# Guia AG-UI SDK para terceiros

Este guia e a porta de entrada para integrar uma interface externa ao AG-UI da plataforma Plataforma de Agentes de IA.

Ele nao substitui o manual tecnico. Ele explica o fluxo em linguagem direta, mostra o caminho seguro para o primeiro run e aponta onde aprofundar.

Importante: o pacote `@prometeu/ag-ui-runtime` continua `private: true` dentro do monorepo. Para terceiros, ele nao e um SDK publico instalado por npm. O caminho suportado para fora do repositório e o template de referencia com `@ag-ui/client` e um backend-for-frontend proprio.

Referencias tecnicas principais:

1. [README-TECNICO-AG-UI.md](../tecnico/README-TECNICO-AG-UI.md)
2. [README-TECNICO-AG-UI-BORDA-HTTP-DEDICADA.md](../tecnico/README-TECNICO-AG-UI-BORDA-HTTP-DEDICADA.md)
3. [README-TECNICO-AG-UI-RUNTIME-COMPARTILHADO-DO-FRONTEND.md](../tecnico/README-TECNICO-AG-UI-RUNTIME-COMPARTILHADO-DO-FRONTEND.md)
4. [README-TECNICO-AG-UI-REPLAY-E-AUDITORIA.md](../tecnico/README-TECNICO-AG-UI-REPLAY-E-AUDITORIA.md)
5. [README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md](../tecnico/README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md)
6. [templates/ag-ui-official-third-party](../templates/ag-ui-official-third-party)

## 1. O que e AG-UI na plataforma

AG-UI e o contrato que permite que uma tela acompanhe uma execucao de IA como um processo observavel.

Em vez de receber apenas uma resposta final em texto, a interface recebe eventos. Esses eventos mostram quando a execucao comecou, qual mensagem chegou, qual estado mudou, se alguma ferramenta de interface foi declarada, se houve pausa para revisao humana e como o run terminou.

Em linguagem simples: AG-UI e a trilha entre a tela e o runtime agentic. A tela nao precisa conhecer YAML interno, banco, DSN, catalogo de tools LangChain ou segredos. O boundary produtivo recebe um `AgUiRunRequest` em `POST /ag-ui/runs`, deriva o runtime pelo YAML e devolve um stream de eventos para desenhar a experiencia. Quando um integrador usa um backend-for-frontend proprio, o browser ainda pode montar um `RunAgentInput` local para conversar com esse backend, mas a chamada produtiva para a plataforma continua sendo `/ag-ui/runs`.

### 1.1. AG-UI nao e a spec visual

AG-UI e o protocolo de eventos. Ele informa quando o run comecou, qual mensagem chegou, qual estado mudou, se houve pausa humana e como a execucao terminou.

Generative UI e outra camada: e o formato visual que viaja dentro de alguns desses eventos para a tela montar um painel, um canvas ou um componente dinamico.

Na Plataforma de Agentes de IA, o perfil visual oficial e `DashboardSpec`.

Isso significa duas coisas praticas:

1. o frontend Plataforma de Agentes de IA nao renderiza JSON visual arbitrario vindo de fora;
2. se um integrador tiver uma spec externa propria, ela precisa ser convertida antes para `DashboardSpec` por um adapter seguro no backend.

Essa regra existe para manter bloqueio de HTML, script, SQL livre, DSN, segredos e `correlation_id` no payload visual.

### 1.2. Contrato oficial para adapter externo

Se um integrador usar A2UI, Open JSON UI, MCP-UI ou qualquer formato visual proprio, essa spec nao entra direto no renderer Plataforma de Agentes de IA.

O contrato oficial e este:

1. O adapter seguro recebe a spec externa no backend, nunca no renderer final.
2. O adapter converte a spec externa para `DashboardSpec` antes da materializacao AG-UI.
3. O payload convertido continua sujeito ao mesmo validador seguro do produto.
4. HTML, JavaScript, SQL livre, DSN, segredo e `correlation_id` continuam proibidos, mesmo que existam na spec de origem.
5. Sem adapter explicito, a spec externa deve falhar fechada.

Em linguagem simples: o Plataforma de Agentes de IA nao aceita desenho arbitrario vindo de fora. Ele so aceita o desenho final no formato governado do proprio produto.

## 2. Contrato publico recomendado

Para integracao nova de terceiros, use este endpoint:

```text
POST /ag-ui/runs
```

O body produtivo deve seguir `AgUiRunRequest`.

A plataforma nao expõe mais rota publica por `agent_id`, nem para execucao nem para capabilities. Se o template third-party ainda usar `RunAgentInput` no browser do integrador, o backend desse integrador deve converter esse envelope para `AgUiRunRequest` antes de chamar a plataforma.

Replay publico:

```text
GET /ag-ui/runs/{run_id}/events
GET /ag-ui/threads/{thread_id}/events
```

Discovery publico:

```text
GET /ag-ui/capabilities
```

Use `GET /ag-ui/capabilities` para descobrir o catalogo de negocio Plataforma de Agentes de IA, ou seja, quais capabilities, exemplos, permissoes e `UISpecs` governadas estao disponiveis para o tenant.

Se o seu cliente precisar de um shape equivalente a `AgentCapabilities`, faca esse mapeamento no backend do integrador a partir do proprio payload de `GET /ag-ui/capabilities`. O produto nao mantem um segundo endpoint publico so para projetar esse contrato por `agent_id`.

## 3. Primeiro agent em 10 minutos

Este fluxo parte do template oficial criado para integradores externos.

### 3.1. Copie o template operacional

Use [templates/ag-ui-official-third-party](../templates/ag-ui-official-third-party).

O template tem duas partes:

1. Backend-for-frontend em FastAPI.
2. Frontend vanilla com Vite e `@ag-ui/client`.

O backend-for-frontend e necessario porque a credencial de acesso a Plataforma de Agentes de IA e a fonte de configuracao precisam ficar no servidor do integrador, nunca no browser.

### 3.2. Configure somente o servidor

No backend do integrador, configure:

```text
PROMETEU_AG_UI_BASE_URL=https://prometeu.exemplo.local
PROMETEU_AG_UI_API_KEY=<chave-servidor>
PROMETEU_AG_UI_USER_EMAIL=operacao@cliente.exemplo
PROMETEU_AG_UI_YAML_INLINE_CONTENT=<yaml-governado-no-servidor>
PUBLIC_TENANT_LABEL=ambiente-demo
```

`PROMETEU_AG_UI_API_KEY` e exemplo de nome de variavel, nao valor real. O valor deve ficar fora do frontend e fora de arquivos versionados.

Se o integrador ja tiver um envelope criptografado emitido pela plataforma, pode usar `PROMETEU_AG_UI_ENCRYPTED_DATA_JSON` no servidor em vez de `PROMETEU_AG_UI_YAML_INLINE_CONTENT`.

### 3.3. Descubra a capability

Chame:

```text
GET /ag-ui/capabilities
```

Se o seu cliente precisa validar aderencia ao shape oficial do SDK antes de abrir o run, chame tambem:

```text
GET /ag-ui/capabilities
```

Olhe principalmente para estes campos:

1. `executionKind`: identifica o runtime ou capability pack exposto.
2. `capability`: identifica a intencao de negocio liberada.
3. `frontendTools`: lista de ferramentas de interface que o browser pode declarar.
4. `supportedEvents`: lista de eventos que a tela deve entender.
5. `supportsHil` e `supportsResume`: indicam pausa humana e retomada.

Na rota canônica por `agent_id`, olhe principalmente para `identity`, `transport`, `tools`, `state`, `humanInTheLoop`, `multiAgent` e `custom.prometeu`. Esse bloco `custom.prometeu` existe para carregar os metadados governados do produto sem deformar o contrato oficial do SDK.

Para capability packs registrados, como `retail_demo` e `erp_backoffice_demo`, o `agent_id` da rota publica e o proprio identificador registrado. Para DeepAgent ou Workflow vinculados a YAML de um tenant, o `agent_id` deve existir na configuracao da Plataforma de Agentes de IA vinculada aquela credencial.

Importante: capability pack continua sendo catalogo de negocio. Nem todo capability pack vira `AgentCapabilities` canônico. Se a rota por `agent_id` recusar esse alvo, isso nao significa defeito; significa apenas que aquele item continua exposto pelo catalogo da Plataforma de Agentes de IA e nao como runtime AG-UI oficial do SDK.

### 3.4. Monte um AgUiRunRequest minimo

Exemplo seguro do servidor confiavel para a plataforma:

```json
{
  "threadId": "thread-demo-001",
  "runId": "run-demo-001",
  "user_email": "operacao@cliente.exemplo",
  "input": {
    "capability": "sales_summary",
    "parameters": {}
  },
  "metadata": {
    "surface": "third-party-demo"
  },
  "yaml_inline_content": "<yaml-governado-no-servidor>"
}
```

Esse JSON representa a chamada do backend confiavel para `POST /ag-ui/runs`, nao o payload exposto no browser.

Se a integracao usar um backend-for-frontend proprio, o browser pode continuar montando um `RunAgentInput` local para falar com esse backend. O ponto importante e que o servidor do integrador converta esse envelope para `AgUiRunRequest` antes de chamar a plataforma.

O browser nao deve enviar diretamente:

1. `security_keys`.
2. `tools_library`.
3. DSN ou connection string.
4. SQL livre.
5. API key.

Os campos `yaml_inline_content` e `encrypted_data` pertencem ao boundary confiavel. Em termos simples: eles podem existir no servidor do integrador ou no backend da plataforma, mas nao devem virar segredo exposto em JavaScript publico.

### 3.5. Consuma o stream com o SDK oficial

No frontend, use `@ag-ui/client`.

Para terceiros, nao consuma `@prometeu/ag-ui-runtime` diretamente. Esse pacote organiza o runtime interno da Plataforma de Agentes de IA e permanece privado. A integracao externa suportada e: template de referencia + `@ag-ui/client` + backend-for-frontend mantendo a chave e a configuracao no servidor.

O template demonstra o fluxo com:

1. `runHttpRequest` para abrir o POST streaming.
2. `transformHttpEventStream` para transformar o stream em eventos oficiais.
3. Um store local simples para desenhar status, mensagens e eventos.

O frontend pode trocar completamente a camada visual. O protocolo nao muda.

### 3.6. Reconstrua com replay

Quando precisar reconstruir a tela depois de refresh, suporte ou auditoria, consulte:

```text
GET /ag-ui/runs/{run_id}/events
```

O replay e sanitizado no event store. Ele existe para reconstruir a experiencia sem reexpor segredos internos.

## 4. Guia de seguranca

### 4.1. Segredos

Segredo fica no servidor.

O browser pode receber uma configuracao publica, como rotas do backend-for-frontend e labels de exibicao. Ele nao pode receber API key, DSN, YAML interno cru, catalogo de tools LangChain, credenciais de banco ou security keys.

### 4.2. Tools LangChain e frontendTools nao sao a mesma coisa

No boundary publico AG-UI, `RunAgentInput.tools` significa somente frontend tools quando o browser fala com o backend-for-frontend do integrador.

Frontend tool e uma capacidade visual da interface, por exemplo abrir um painel aprovado. Ela nao e uma Tool LangChain da plataforma, nao passa pela ToolsFactory e nao autoriza banco, Redis, Qdrant, APIs internas ou SQL.

A regra pratica e simples:

1. Se a capability nao publicou `frontendTools`, envie `tools: []`.
2. Se publicou, envie apenas o nome e o schema exatos da allowlist.
3. Nunca use `tools` para tentar transportar SQL, DSN, segredo, credencial ou selecao de runtime.

### 4.3. Threat model do RunAgentInput publico

O risco principal do payload publico e virar um atalho para driblar a governanca do produto.

Por isso a plataforma registra `ag_ui.public_payload.decision` e opera em falha fechada quando encontra campos sensiveis, runtime divergente ou tentativa de esconder dados internos dentro de `metadata`, `forwardedProps`, `state` ou `messages`.

Em termos simples: se o payload publico tentar agir como YAML, banco, segredo ou roteador de runtime, o backend rejeita a chamada.

## 5. Varejo e vendas

No dominio de varejo e vendas, a interface normalmente quer pedir uma capacidade de negocio, nao um runtime tecnico.

Exemplo pratico: a tela pode pedir `sales_summary`, `catalog_health` ou `turn_closure_review`. O backend usa o YAML para decidir se aquilo sera atendido por DeepAgent, Workflow ou capability pack governado.

## 6. Migracao e adaptacao de ERP

Em ERP, o erro mais comum e tentar usar `agent_id` como se fosse a chave primaria da integracao. Isso cria uma arquitetura paralela e quebra o modelo YAML-first.

O caminho correto e este:

1. O ERP fala com o backend do integrador ou com o boundary confiavel da plataforma.
2. O backend confiavel injeta `yaml_inline_content` ou `encrypted_data`.
3. A plataforma deriva o runtime pelo YAML.
4. A tela consome SSE e replay sem decidir infraestrutura no browser.

Isso reduz acoplamento e evita que cada cliente externo precise reinventar resolucao de tenant, runtime e permissao.
  }
}
```

O browser nao deve enviar:

1. YAML bruto.
2. `executionKind`.
3. `tenantId`.
4. `securityKeys`.
5. `toolsLibrary`.
6. DSN ou connection string.
7. SQL livre.
8. API key.

Esses dados pertencem ao backend da Plataforma de Agentes de IA ou ao backend-for-frontend do integrador.

### 3.5. Consuma o stream com o SDK oficial

No frontend, use `@ag-ui/client`.

Para terceiros, nao consuma `@prometeu/ag-ui-runtime` diretamente. Esse pacote organiza o runtime interno da Plataforma de Agentes de IA e permanece privado. A integracao externa suportada e: template de referencia + `@ag-ui/client` + backend-for-frontend mantendo a chave no servidor.

O template demonstra o fluxo com:

1. `runHttpRequest` para abrir o POST streaming.
2. `transformHttpEventStream` para transformar o stream em eventos oficiais.
3. Um store local simples para desenhar status, mensagens e eventos.

O frontend pode trocar completamente a camada visual. O protocolo nao muda.

### 3.6. Reconstrua com replay

Quando precisar reconstruir a tela depois de refresh, suporte ou auditoria, consulte:

```text
GET /ag-ui/runs/{run_id}/events
```

O replay e sanitizado no event store. Ele existe para reconstruir a experiencia sem reexpor segredos internos.

## 4. Guia de seguranca

### 4.1. Segredos

Segredo fica no servidor.

O browser pode receber uma configuracao publica, como lista de agentes permitidos e rotas do backend-for-frontend. Ele nao pode receber API key, DSN, YAML interno, catalogo de tools LangChain, credenciais de banco ou security keys.

### 4.2. Tools LangChain e frontendTools nao sao a mesma coisa

No boundary publico AG-UI, `RunAgentInput.tools` significa somente frontend tools.

Frontend tool e uma capacidade visual da interface, por exemplo abrir um painel aprovado. Ela nao e uma Tool LangChain da plataforma, nao passa pela ToolsFactory e nao autoriza banco, Redis, Qdrant, APIs internas ou SQL.

A regra pratica e simples:

1. Se a capability nao publicou allowlist em `frontendTools`, envie `tools: []`.
2. Se publicou allowlist, envie apenas nomes e schemas exatamente iguais ao discovery.
3. Se o frontend inventar uma tool, o backend deve rejeitar.
4. Se a tool trouxer SQL, DSN, connection string ou security keys, o backend deve rejeitar.

### 4.3. Threat model do RunAgentInput publico

Threat model significa mapa de ameacas. Em linguagem simples, e a lista do que pode dar errado se um cliente externo enviar campos perigosos no payload.

O endpoint publico aplica falha fechada para estes riscos:

| Campo do `RunAgentInput` | Risco principal                                                      | Regra pratica                                                                                    |
| ------------------------ | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `messages`               | texto muito grande ou tentativa de esconder segredo no prompt        | limite de quantidade e tamanho; DSN e chave privada sao bloqueados quando aparecem como valor    |
| `tools`                  | confundir frontend tool com Tool LangChain, banco ou SQL             | somente tools publicadas em `frontendTools` podem ser enviadas, com schema identico ao discovery |
| `context`                | transportar tenant, DSN, YAML ou segredo como contexto auxiliar      | chaves internas e sensiveis sao recusadas em qualquer profundidade                               |
| `forwardedProps`         | burlar governanca com `executionKind`, YAML, tenant ou security keys | apenas chaves publicas permitidas entram; qualquer chave fora da allowlist falha com HTTP 400    |
| `state`                  | esconder configuracao interna em estado inicial                      | chaves como `yamlPath`, `securityKeys`, `dsn`, `sql` e equivalentes sao recusadas                |
| `resume`                 | retomar uma pausa humana com contrato divergente                     | `resume` top-level e `forwardedProps.resume`, quando ambos existirem, precisam ser iguais        |

Tambem existem limites de tamanho e quantidade para reduzir abuso operacional. Isso evita que uma integracao use AG-UI como canal para payload gigante, segredo, SQL livre ou configuracao bruta da Plataforma de Agentes de IA.

Quando uma rejeicao acontece, o backend registra uma decisao estruturada em log com `ag_ui.public_payload.decision`, motivo da rejeicao, `agent_id` e contagens nao sensiveis. O log nao grava o segredo, o SQL nem o conteudo completo do payload.

O mesmo principio vale para UI generativa: uma spec visual externa nao entra direto no renderer Plataforma de Agentes de IA. Primeiro ela precisa ser convertida para `DashboardSpec`, que e o perfil governado do produto.

### 4.4. SQL e dados de varejo

SQL livre nao entra pelo browser.

Em cenarios de varejo e ERP, a tela envia uma intencao de negocio. O backend resolve isso usando capability governada, catalogo permitido e credencial segura no servidor.

Isso evita dois problemas práticos:

1. Expor estrutura interna do banco para o usuario final.
2. Permitir que uma interface externa execute consulta arbitraria.

### 4.5. Replay

Replay e ferramenta de reconstrucao e auditoria, nao canal de vazamento.

O event store sanitiza payloads para reduzir risco de segredo, SQL livre e identificadores internos aparecerem em evento publico. Mesmo assim, o integrador deve tratar replay como dado operacional sensivel: autentique, autorize e mostre apenas para usuarios com permissao.

### 4.6. HIL e resume

HIL significa human in the loop. Em linguagem simples, e quando a IA pausa e pede decisao humana.

Quando a capability indicar suporte, a pausa aparece como `RUN_FINISHED` com `outcome` de interrupcao. A retomada volta pelo mesmo endpoint de run, usando `resume` no `RunAgentInput`.

## 5. Varejo e vendas

O dominio de varejo e a vitrine mais madura do AG-UI atual.

O caso tipico e este:

1. A tela pede uma capability de negocio, como resumo de vendas ou analise operacional.
2. O backend aplica regras governadas.
3. A execucao publica eventos AG-UI.
4. A interface mostra status, mensagens, ferramentas visuais e estado.
5. O replay permite reconstruir o que aconteceu.

Use o discovery antes de hardcodar qualquer capability. O que existe em um ambiente pode variar conforme configuracao, permissao e catalogo ativo.

Documentos complementares:

1. [README-CONCEITUAL-AG-UI-DOMINIO-VAREJO-DEMO.md](../conceitual/README-CONCEITUAL-AG-UI-DOMINIO-VAREJO-DEMO.md)
2. [README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md](../tecnico/README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md)

## 6. Migracao e adaptacao de ERP

Para adaptar AG-UI a um ERP ou PDV existente, evite comecar pela tela. Comece pelo contrato de negocio.

Sequencia recomendada:

1. Liste as telas em que IA deve ajudar o usuario.
2. Para cada tela, escreva a intencao de negocio em linguagem simples.
3. Converta cada intencao em capability governada no backend.
4. Mantenha DSN, SQL, catalogos e credenciais no servidor.
5. Exponha somente `agent_id`, capability, parametros permitidos e frontendTools aprovadas.
6. Consuma eventos AG-UI e desenhe a experiencia no design system do ERP.
7. Use replay para suporte e auditoria.

Exemplo pratico: uma tela de fechamento de caixa nao deve enviar SQL ao AG-UI. Ela deve enviar algo como "analisar divergencias do turno" com parametros permitidos. O backend decide quais consultas aprovadas ou workflows internos sao usados.

## 7. Matriz de eventos suportados no boundary Plataforma de Agentes de IA

Esta matriz lista os eventos que o schema Plataforma de Agentes de IA envolve diretamente hoje em [src/api/schemas/ag_ui_models.py](../src/api/schemas/ag_ui_models.py).

| Grupo               | Eventos                                                                                     | Uso pratico na UI                                                                                                              |
| ------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Lifecycle           | `RUN_STARTED`, `RUN_FINISHED`, `RUN_ERROR`                                                  | Mostrar inicio, sucesso, pausa humana ou erro terminal.                                                                        |
| Etapas              | `STEP_STARTED`, `STEP_FINISHED`                                                             | Mostrar progresso por etapa quando o adapter emitir etapas.                                                                    |
| Texto               | `TEXT_MESSAGE_START`, `TEXT_MESSAGE_CONTENT`, `TEXT_MESSAGE_END`, `TEXT_MESSAGE_CHUNK`      | Renderizar mensagens incrementais.                                                                                             |
| Tool call           | `TOOL_CALL_START`, `TOOL_CALL_ARGS`, `TOOL_CALL_END`, `TOOL_CALL_RESULT`, `TOOL_CALL_CHUNK` | Mostrar chamada, argumentos e resultado de tool quando o runtime publicar esse detalhe.                                        |
| Estado              | `STATE_SNAPSHOT`, `STATE_DELTA`                                                             | Substituir ou alterar estado visual local. `STATE_DELTA` usa operacoes de patch.                                               |
| Mensagens           | `MESSAGES_SNAPSHOT`                                                                         | Reconstituir lista de mensagens quando houver snapshot completo.                                                               |
| Atividade           | `ACTIVITY_SNAPSHOT`, `ACTIVITY_DELTA`                                                       | Renderizar timeline e progresso operacional.                                                                                   |
| Extensao controlada | `CUSTOM`, `RAW`                                                                             | Transportar eventos especificos quando o adapter precisar, com sanitizacao e validacao do lado da Plataforma de Agentes de IA. |

Eventos adicionais existem no pacote oficial AG-UI instalado, mas nao devem ser prometidos como contrato Plataforma de Agentes de IA sem confirmacao no schema, adapter e testes.

## 8. Checklist antes de publicar uma integracao

Antes de colocar uma integracao AG-UI em producao, confirme:

1. O browser nao recebe API key, DSN, YAML, tenant interno, security keys ou SQL livre.
2. O endpoint usado pela plataforma para execucao e `POST /ag-ui/runs`.
3. O payload produtivo segue `AgUiRunRequest`.
4. `RunAgentInput.tools` esta vazio ou bate exatamente com `frontendTools` do discovery.
5. O backend-for-frontend injeta a fonte de configuracao confiavel no servidor.
6. O frontend consome SSE por POST, nao por GET `EventSource` simples.
7. O replay exige autenticacao e autorizacao.
8. Erros exibem mensagem operacional clara, sem fallback silencioso.
9. O design system renderiza eventos e estado sem interpretar HTML livre.
10. O fluxo foi validado com o template oficial e com o replay.

### 8.1. Checklist de novo componente visual

Se a integracao vai introduzir um componente novo na UI generativa da Plataforma de Agentes de IA, confirme tambem:

1. O componente foi registrado no Component Catalog da Plataforma de Agentes de IA, nao em JSON arbitrario solto.
2. Props, actions e bindings ficaram em allowlist explicita.
3. O cadastro novo continua bloqueando HTML, JavaScript, SQL livre, DSN, segredo e `correlation_id`.
4. Existe teste backend provando que apenas `DashboardSpec` validada chega na materializacao.
5. Existe teste frontend provando que o Component Catalog e o renderer rejeitam payload fora da allowlist.

Esse checklist evita que um componente novo vire uma excecao escondida no contrato visual.

## 9. Arquivos que provam este contrato

Este guia foi escrito com base no fluxo executavel atual, nao apenas em intencao de produto.

Evidencias principais:

1. [src/api/routers/ag_ui_router.py](../src/api/routers/ag_ui_router.py)
2. [src/api/schemas/ag_ui_models.py](../src/api/schemas/ag_ui_models.py)
3. [src/api/services/ag_ui_public_run_resolver.py](../src/api/services/ag_ui_public_run_resolver.py)
4. [src/api/services/ag_ui_frontend_tool_policy.py](../src/api/services/ag_ui_frontend_tool_policy.py)
5. [src/api/services/ag_ui_event_store.py](../src/api/services/ag_ui_event_store.py)
6. [src/api/services/ag_ui_adapter_registry.py](../src/api/services/ag_ui_adapter_registry.py)
7. [src/api/services/ag_ui_langgraph_agent_factory.py](../src/api/services/ag_ui_langgraph_agent_factory.py)
8. [templates/ag-ui-official-third-party](../templates/ag-ui-official-third-party)
9. [tests/unit/test_02-01-52_ag_ui_third_party_template_contract.py](../tests/unit/test_02-01-52_ag_ui_third_party_template_contract.py)
10. [tests/unit/test_02-01-48_ag_ui_router.py](../tests/unit/test_02-01-48_ag_ui_router.py)
11. [tests/unit/test_02-01-38_ag_ui_event_store.py](../tests/unit/test_02-01-38_ag_ui_event_store.py)
12. [tests/js/ag_ui_runtime.test.js](../tests/js/ag_ui_runtime.test.js)

## 10. Proximos passos de leitura

1. Para implementar rapido: templates/ag-ui-official-third-party/README.md
2. Para entender o boundary: [README-TECNICO-AG-UI-BORDA-HTTP-DEDICADA.md](../tecnico/README-TECNICO-AG-UI-BORDA-HTTP-DEDICADA.md)
3. Para entender replay: [README-TECNICO-AG-UI-REPLAY-E-AUDITORIA.md](../tecnico/README-TECNICO-AG-UI-REPLAY-E-AUDITORIA.md)
4. Para runtime frontend: [README-TECNICO-AG-UI-RUNTIME-COMPARTILHADO-DO-FRONTEND.md](../tecnico/README-TECNICO-AG-UI-RUNTIME-COMPARTILHADO-DO-FRONTEND.md)
5. Para varejo/vendas: [README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md](../tecnico/README-TECNICO-AG-UI-DOMINIO-VAREJO-DEMO.md)
