# FAQ de Onboarding (Consultor Júnior)

Esta FAQ foi sincronizada com a codebase lida nesta atualização.
Ela não substitui os manuais donos do assunto, especialmente os de ingestão PDF.
Use este arquivo para tirar dúvidas rápidas de operação, wiring e responsabilidade de cada peça.
Quando precisar de profundidade no slice PDF, leia também os READMEs conceitual e técnico correspondentes.

## 1. Onde mexer

### 1.1
Pergunta: Onde entra a requisição pública de ingestão PDF?

Resposta:
A entrada pública fica em `POST /rag/ingest`.
Esse boundary não processa o PDF dentro da resposta HTTP.
Ele compõe o YAML, resolve contexto do usuário e agenda um job pai assíncrono.
O slice PDF só aparece depois, já dentro da execução do worker.

Onde no código:
- `src/api/routers/rag_ingestion_router.py` - `build_router`
- `src/api/routers/rag_runtime_ingestion_compat.py` - `PreparedAsyncIngestionExecutionService.__call__`

### 1.2
Pergunta: Onde o YAML agentic vira execução governada?

Resposta:
O caminho público fica em `/config/assembly`.
Esse fluxo não trata YAML como texto solto do começo ao fim.
Ele passa por AST, validação, diff, schema e confirmação antes de virar artefato governado.
Na prática, esse é o ponto certo para draft, validate e objective-to-yaml.
Na UI administrativa real, esse mesmo fluxo é consumido por `app/ui/static/js/admin-assembly-ast.js`.

Onde no código:
- `src/api/routers/config_assembly_router.py`
- `src/config/agentic_assembly/assembly_service.py` - `AgenticAssemblyService`

### 1.3
Pergunta: Onde sobe o worker oficial do produto?

Resposta:
O worker oficial não nasce no router nem no processor do PDF.
Ele sobe por um entrypoint próprio e depois inicializa o runtime unificado do processo worker.
Esse runtime exige Dramatiq e RabbitMQ para aceitar jobs assíncronos de ingestão e ETL.
Se esse processo não estiver de pé, a API aceita o pedido, mas o trabalho não anda.

Onde no código:
- `app/worker_main.py`
- `app/runners/api_runner.py` - `run_api_server`
- `app/runners/worker_runner.py` - `run_worker_process`
- `app/runners/scheduler_runner.py` - `run_scheduler_process`
- `src/api/services/worker_process_runtime.py` - `build_worker_process_runtime`

### 1.4
Pergunta: Onde eu altero o paralelismo documental da ingestão?

Resposta:
O paralelismo começa no request e pode ser ajustado pelo YAML resolvido.
Depois disso, o serviço de ingestão decide se o lote fica simples ou se abre fan-out real.
O coordenador especializado é quem inventaria documentos e publica envelopes filhos.
Então o lugar certo não é o processor do PDF, e sim o boundary e a orquestração do fan-out.
No detalhe operacional do pai, o contrato `fanout_overview` resume volume, estados agregados e sinais usados pela UI administrativa.

Onde no código:
- `src/api/routers/rag_runtime_ingestion_compat.py` - `resolve_effective_document_parallelism_for_request`
- `src/services/ingestion_service.py`
- `src/services/document_fanout_coordinator.py`

### 1.5
Pergunta: Onde eu crio ou altero tools reutilizáveis?

Resposta:
Tools reutilizáveis vivem na camada de tools do projeto, não espalhadas em routers.
O padrão canônico usa decorators `@tool` e `@tool_factory`.
Depois de criar ou ajustar a tool, o catálogo builtin precisa ser sincronizado pelo builder oficial.
Isso evita catálogo manual e reduz divergência entre código e runtime.

Onde no código:
- `src/agentic_layer/tools/tool_factory_decorator.py`
- `src/agentic_layer/tools/tools_library_builder.py`
- `src/agentic_layer/tools/system_tools/background_execution.py`

## 2. Modelagem e validações

### 2.1
Pergunta: O que é `correlation_id`?

Resposta:
`correlation_id` é o identificador lógico oficial da execução.
Ele deve nascer no boundary oficial do processo, normalmente no middleware HTTP compartilhado, em um endpoint que atue como boundary dedicado ou em um iniciador não interativo do próprio sistema que abra a execução real, como scheduler ou runner.
Depois disso, o resto do sistema só deve preservar e propagar esse mesmo valor.
Sem ele, você até vê pedaços do fluxo, mas não consegue provar que API, worker, fila e leitura administrativa pertencem à mesma história.
Em sistema distribuído, isso separa diagnóstico real de adivinhação.

Onde no código:
- `src/api/service_api.py` - `log_requests`
- `src/core/base_correlation_component.py`
- `src/core/logging_system.py` - `normalize_correlation_id`

### 2.2
Pergunta: O que é `worker_execution_correlation_id`?

Resposta:
Ele é a identidade operacional específica da execução enfileirada no worker.
Não substitui o `correlation_id` lógico do pedido.
Na ingestão, ele é composto explicitamente com `correlation_id` e `run_id`.
Isso ajuda a distinguir o pedido original da execução reservada para o worker.

Onde no código:
- `src/api/services/ingestion_async_enqueue_service.py` - `build_reserved_worker_execution_correlation_id`
- `src/api/services/ingestion_http_prepared_async_service.py`

### 2.3
Pergunta: Quais envelopes assíncronos de ingestão existem hoje?

Resposta:
O código lido confirma três envelopes principais: `prepared_yaml`, `resolve_on_worker` e `document_fanout_child`.
Eles não são intercambiáveis e têm papéis diferentes.
O pai carrega o lote e a decisão operacional.
O filho carrega uma unidade documental específica já inventariada pelo coordenador.

Onde no código:
- `src/api/services/ingestion_async_enqueue_support.py`
- `src/api/services/async_job_worker_payload_executor.py`

### 2.3.1
Pergunta: Quando eu uso `IngestionJobExecutor` em vez de `schedule_prepared_ingestion_worker_job`?

Resposta:
Use `schedule_prepared_ingestion_worker_job` quando a borda HTTP já tem o YAML pronto e quer publicar `prepared_yaml`.
Use `IngestionJobExecutor` quando a borda ainda vai mandar o payload criptografado e deixar a resolução final para o worker com `resolve_on_worker`.
Os dois caminhos continuam dentro do mesmo mecanismo canônico de jobs.
O erro seria criar um terceiro publish local fora desses dois boundaries.

Onde no código:
- `src/api/services/ingestion_http_prepared_async_service.py` - `schedule_prepared_ingestion_worker_job`
- `src/api/services/ingestion_job_executor.py`

### 2.3.2
Pergunta: O que o worker faz quando recebe `resolve_on_worker`?

Resposta:
O worker não executa o PDF direto.
Primeiro ele tenta ler `encrypted_data` do envelope e, se necessário, recarrega o payload criptografado pela porta de fila.
Depois resolve YAML, descobre `user_email`, verifica cancelamento do pai e só então chama `execute_ingestion_async`.
Na prática, `resolve_on_worker` muda o ponto de resolução do YAML, mas não cria um runtime paralelo.

Onde no código:
- `src/api/services/async_job_worker_payload_executor.py` - `IngestionParentJobHandler`
- `src/api/services/ingestion_job_executor.py`

### 2.4
Pergunta: Que contrato HTTP eu recebo quando chamo `/rag/ingest`?

Resposta:
Você recebe um aceite assíncrono, não o resultado final do PDF.
O payload inclui `task_id`, `correlation_id`, `status`, `execution_mode` e URLs de acompanhamento.
Também saem nomes de log da API e do worker para investigação operacional.
Na prática, a resposta já nasce preparada para polling, stream e cancelamento.

Onde no código:
- `src/api/routers/rag_runtime_ingestion_compat.py` - `PreparedAsyncIngestionExecutionService.__call__`
- `src/api/services/ingestion_http_prepared_async_service.py`

### 2.5
Pergunta: Quando o fan-out documental é bloqueado?

Resposta:
Ele pode ser bloqueado antes e depois da publicação do filho.
Antes, o coordenador barra fontes não elegíveis, como filesystem local não compartilhado.
Depois, a gate canônica barra filhos quando o pai já terminou, falhou ou entrou em cancelamento.
Isso evita filho fantasma rodando trabalho caro sem autorização válida do run pai.

Onde no código:
- `src/services/document_fanout_coordinator.py`
- `src/services/document_fanout_execution_gate.py` - `DocumentFanoutExecutionGate`

## 3. Execução, erros e logs

### 3.1
Pergunta: Receber HTTP 202 significa que deu certo?

Resposta:
Não significa sucesso final.
Significa apenas que o pedido foi aceito e entregue ao fluxo assíncrono.
O passo seguinte é acompanhar status, logs e, quando existir, o overview do fan-out.
Confundir aceite com conclusão é um dos erros mais comuns do onboarding.

Onde no código:
- `src/api/routers/rag_ingestion_router.py`
- `src/api/routers/rag_etl_router.py`
- `src/api/routers/rag_runtime_ingestion_compat.py`

### 3.1.1
Pergunta: Quais markers mostram que o worker realmente ficou pronto?

Resposta:
Os markers mais úteis hoje são `MULTICHANNEL_SUPERVISOR_READY`, `WORKER_RUNTIME_READY` e `WORKER_READY`.
Eles não significam a mesma coisa.
O primeiro confirma o plano de controle multicanal, o segundo confirma o runtime unificado e o último fecha o bootstrap do processo worker.
Se um deles não aparecer, a prontidão ficou incompleta e você ainda não deve culpar o pipeline PDF.

Onde no código:
- `app/runners/worker_runner.py` - `run_worker_process`
- `src/api/services/worker_process_runtime.py` - `WorkerProcessRuntime.start`

### 3.2
Pergunta: Como eu acompanho progresso em tempo real?

Resposta:
O contrato de ingestão já devolve URLs de polling e stream.
O router de status expõe `/api/v1/status` para acompanhamento compatível com o fluxo assíncrono.
Na prática, você usa o `task_id` retornado pela API e acompanha até estado terminal.
Se o processo morrer sem fechar status, o router ainda tenta reconciliar ingestão órfã.

Onde no código:
- `src/api/routers/rag_runtime_ingestion_compat.py`
- `src/api/routers/streaming_router.py`

### 3.3
Pergunta: Em que momento o worker faz ACK da mensagem?

Resposta:
O ACK não é o primeiro passo do worker.
No fluxo lido, o reconhecimento só acontece depois do tratamento correto do payload.
Para filho documental, a persistência do estado terminal vem antes do ACK.
Isso existe para reduzir perda de rastreabilidade e falso sucesso operacional.

Onde no código:
- `src/api/services/async_job_worker_payload_executor.py`
- `src/services/document_fanout_child_executor_service.py`

### 3.4
Pergunta: Como eu diferencio erro de API, fila, worker pai, worker filho e pipeline PDF?

Resposta:
Olhe a falha como uma escada, não como um bloco único.
Primeiro valide o boundary HTTP e o enqueue do job pai.
Depois confirme o roteamento do payload no runtime Dramatiq e, por fim, a execução do filho e do pipeline PDF.
Esse corte evita perder tempo depurando OCR quando o job nem chegou à fila certa.

Onde no código:
- `src/api/services/ingestion_http_prepared_async_service.py`
- `src/api/services/async_job_dramatiq.py`
- `src/services/document_fanout_child_executor_service.py`
- `src/ingestion_layer/processors/pdf_processor.py`

### 3.5
Pergunta: Como funciona o cancelamento cooperativo?

Resposta:
O cancelamento entra por endpoint HTTP e vira sinal durável de cancelamento solicitado.
O objetivo não é matar tudo no meio sem rastreio.
O objetivo é bloquear trabalho novo, propagar o sinal e deixar checkpoints interromperem o que ainda estava em andamento.
No filho documental, o callback também observa se o pai já recebeu pedido de cancelamento.

Onde no código:
- `src/api/routers/rag_cancellation_router.py`
- `src/api/services/cooperative_cancellation_service.py`
- `src/services/document_fanout_child_executor_service.py`

### 3.6
Pergunta: O browser pode gerar `correlation_id` por conta própria?

Resposta:
Não.
No código atual existe até uma função legado no frontend que lança erro de propósito quando alguém tenta gerar esse valor no browser.
O papel da UI é capturar o `correlation_id` devolvido pela API, exibir quando fizer sentido e reaproveitar o mesmo valor em chamadas auxiliares relacionadas à mesma execução.
Se a UI inventar outro id, a trilha de observabilidade quebra.

Onde no código:
- `app/ui/static/js/plataforma-agentes-ia-crypto.js` - `generateCorrelationId`
- `app/ui/static/js/ui-webchat-v3.js` - `_extractResponseCorrelationId`
- `app/ui/static/js/admin-ingestao.js` - `buildPinnedCorrelationHeaders`

### 3.7
Pergunta: Quando eu uso `create_logger_with_correlation` e quando eu uso `create_component_logger`?

Resposta:
Use `create_logger_with_correlation` quando o código estiver dentro de um processo real correlacionado, como request HTTP, job de worker, service de domínio chamado pelo fluxo ou qualquer etapa que já tenha `correlation_id` oficial.
Se o próprio scheduler, runner ou outro iniciador não interativo abrir uma execução real de produto, ele também atua como boundary oficial e cria uma única vez esse `correlation_id` antes de criar o logger oficial da execução.
Use `create_component_logger` apenas para eventos técnicos fora de processo real, como bootstrap, preparação de ambiente ou logs puramente operacionais sem correlação de produto.
Fora do boundary oficial, ninguém pode criar logger com identidade própria por conveniência. Se a classe precisar de um logger local, ele deve nascer com o mesmo `correlation_id` oficial já existente no contexto.
Trocar um pelo outro no lugar errado destrói a rastreabilidade.

Onde no código:
- `src/core/logging_system.py` - `create_logger_with_correlation`
- `src/core/logging_system.py` - `create_component_logger`
- `src/core/base_correlation_component.py` - `_setup_isolated_logger`

### 3.8
Pergunta: O que significa `logging.contract.violation`?

Resposta:
Significa que o formatter recebeu um evento estruturado sem `event_name` canônico válido.
O runtime ainda serializa o log como JSON, mas marca explicitamente que o contrato oficial foi violado.
Isso ajuda a distinguir “log em JSON” de “log realmente canônico”.
Na prática, esse marcador mostra que algum call site ainda não está usando o builder certo ou não informou o nome oficial do evento.

Onde no código:
- `src/core/logging_system.py` - `_build_standard_log_payload`
- `src/core/log_canonical_fields.py` - `build_canonical_log_context`

### 3.9
Pergunta: Qual é a primeira sequência de checagem quando o log parece “sumido”?

Resposta:
Primeiro confirme se a resposta devolveu `X-Correlation-Id`.
Depois veja se o fluxo usou o logger correto, se o `correlation_id` era válido para arquivo dedicado e se sidecar e `correlation_manifest.jsonl` foram criados quando isso era esperado.
Só depois avance para provider administrativo, CloudWatch ou leitura remota.
Essa ordem evita culpar infraestrutura quando o problema real está na origem da correlação ou no call site do logger.

Onde no código:
- `src/api/service_api.py` - `log_requests`
- `src/core/logging_system.py` - `create_logger_with_correlation`
- `src/core/log_origin_metadata.py` - `write_correlation_origin_sidecar`
- `src/api/services/log_provider_service.py` - `resolve_active_log_provider_type`

## 4. Ingestão e ETL

### 4.1
Pergunta: Qual a diferença prática entre ingestão e ETL neste produto?

Resposta:
Ingestão transforma documentos e fontes em acervo consultável.
ETL transforma pipelines estruturados e os executa como fluxos próprios.
Os dois compartilham infraestrutura assíncrona, mas não são o mesmo domínio.
Se você diagnosticar ETL como se fosse PDF, vai olhar o lugar errado.

Onde no código:
- `src/api/routers/rag_ingestion_router.py`
- `src/api/routers/rag_etl_router.py`
- `src/api/services/rag_async_execution_service.py` - `execute_etl_async`

### 4.2
Pergunta: O que o job pai de ingestão faz de verdade?

Resposta:
Ele não existe para parsear PDF.
Ele prepara contexto, monitoração, telemetria, run pai e decisão operacional do lote.
Quando o fan-out é elegível, ele inventaria documentos e publica envelopes filhos.
Quando não é, ele segue pela esteira simples do domínio de ingestão.

Onde no código:
- `src/api/services/ingestion_http_prepared_async_service.py`
- `src/services/ingestion_service.py`
- `src/services/document_fanout_coordinator.py`

### 4.3
Pergunta: O que o job filho de ingestão faz de verdade?

Resposta:
Ele executa uma unidade documental concreta.
Primeiro ele confere se o pai ainda autoriza execução.
Depois respeita cancelamento cooperativo, roda a esteira do documento e persiste o estado terminal.
Só depois disso a mensagem pode ser tratada como concluída pelo worker.

Onde no código:
- `src/api/services/async_job_worker_payload_executor.py` - `IngestionDocumentJobHandler`
- `src/services/document_fanout_child_executor_service.py` - `DocumentFanoutChildExecutorService.execute`

### 4.4
Pergunta: Como eu valido que o PDF realmente entrou no acervo?

Resposta:
Não pare na resposta 202 da API.
Valide status terminal, logs de conclusão e a etapa final de indexação do documento.
O sinal mais forte é o fechamento com documento processado, chunks indexados e persistência concluída.
Sem isso, o arquivo pode ter sido aceito, mas ainda não virou acervo útil.

Onde no código:
- `src/api/routers/streaming_router.py`
- `src/ingestion_layer/file_pipeline_services.py` - `DocumentIndexingExecutor.finalize`

### 4.5
Pergunta: ETL também pode fazer fan-out real?

Resposta:
Sim, mas o fan-out de ETL é por pipeline habilitado, não por documento PDF.
O runtime assíncrono de ETL calcula paralelismo efetivo e, quando elegível, delega para um coordenador próprio.
Isso mostra que a infraestrutura assíncrona é compartilhada, mas a unidade de paralelismo muda conforme o domínio.
Ingestão paraleliza documento; ETL paraleliza pipeline.

Onde no código:
- `src/api/services/rag_async_execution_service.py` - `execute_etl_async`
- `src/api/services/etl_pipeline_fanout_coordinator.py` - `EtlPipelineFanoutCoordinator`

## 5. RAG

### 5.1
Pergunta: Em que momento o PDF vira chunks?

Resposta:
O PDF vira chunks depois da extração, limpeza, enriquecimento e decisão de estratégia.
Esse passo ainda faz parte da esteira documental, não do RAG de pergunta e resposta.
Depois que os chunks nascem, eles seguem para indexação e persistência.
É só nesse ponto que o documento deixa de ser arquivo bruto e vira acervo pesquisável.

Onde no código:
- `src/ingestion_layer/processors/pdf_processor.py`
- `src/ingestion_layer/file_pipeline_services.py` - `DocumentIndexingExecutor.finalize`

### 5.2
Pergunta: Onde vive o runtime moderno do RAG?

Resposta:
O runtime moderno aparece no sistema de QA, não no router de ingestão.
Ele monta visão de runtime, valida layout moderno e aciona a assembly do pipeline de pergunta.
Em linguagem simples, é ele que organiza como o sistema vai procurar evidência e responder.
Sem esse runtime, a pergunta ficaria muito mais próxima de um fluxo fixo e cego.

Onde no código:
- `src/qa_layer/content_qa_system.py` - `ContentQASystem`
- `src/orchestrators/qa_runtime_assembly.py`

### 5.3
Pergunta: Qual a diferença entre acervo pronto e pergunta respondida?

Resposta:
Acervo pronto é resultado da ingestão ou do ETL encerrado com sucesso.
Pergunta respondida é resultado do runtime de QA consultando esse acervo.
Essas duas coisas se conectam, mas não são a mesma etapa.
Se o acervo nasceu ruim, o RAG só vai consultar material ruim de forma elegante.

Onde no código:
- `src/ingestion_layer/file_pipeline_services.py`
- `src/qa_layer/content_qa_system.py`

### 5.4
Pergunta: Onde entram prompts e contexto na resposta do RAG?

Resposta:
Prompts e contexto entram na camada de QA, depois que a pergunta já chegou ao runtime certo.
O código lido mostra bundle de prompts, estado de runtime e montagem de visão YAML de execução.
Isso quer dizer que a resposta final não depende só do acervo, mas também do contexto escolhido para perguntar.
Em operação, prompt ruim e retrieval ruim podem parecer o mesmo problema se você não separar as camadas.

Onde no código:
- `src/qa_layer/content_qa_system.py`
- `src/shared/prompts/default_prompts.py`
- `src/qa_layer/qa_runtime_state.py`

### 5.5
Pergunta: O que eu olho quando o RAG não trouxe contexto útil?

Resposta:
Primeiro confirme se o documento realmente entrou no acervo.
Depois verifique `vectorstore_id`, runtime escolhido e diagnósticos da pipeline de QA.
Por fim, observe se o problema está na evidência recuperada ou na forma de responder sobre a evidência.
Essa ordem evita culpar prompt quando o acervo ainda está incompleto.

Onde no código:
- `src/qa_layer/content_qa_system.py`
- `src/qa_layer/pipeline_diagnostics_reporter.py`
- `src/qa_layer/evidence_analyzer.py`

## 5.6 Integração de chat e API (cliente externo ou componente embutível)

> Para o passo a passo do zero, ver [GUIA-PRIMEIROS-PASSOS.md](GUIA-PRIMEIROS-PASSOS.md).
> Para montar um chat completo com o componente oficial, ver
> [TUTORIAL-CHAT-PLATAFORMA.md](TUTORIAL-CHAT-PLATAFORMA.md). Para uma interface
> 100% própria sem o componente, ver [GUIA-INTEGRADOR-CHAT-PLATAFORMA.md](GUIA-INTEGRADOR-CHAT-PLATAFORMA.md).

### 5.6.1
Pergunta: Preciso enviar `X-API-Key` e YAML ao mesmo tempo?

Resposta:
Não. A chave de acesso e o YAML são fontes **alternativas** de credencial; basta **uma** delas.
A chave pode vir no header `X-API-Key` ou já estar embutida no próprio YAML em
`authentication.access_key`. Se a chave já está no YAML, o header pode ir vazio sem quebrar o fluxo.
O único campo sempre obrigatório à parte é o `user_email`.
Exigir os três juntos (YAML + `X-API-Key` + e-mail como se fossem cumulativos) é erro de integração.

Onde no código:
- `app/ui/static/js/shared/layout-mestre-api.js` - `_executarRequisicao` (header `X-API-Key`)
- `src/api/security/user_auth.py` - `authenticate_with_yaml_config` (chave do header **ou** do YAML)

### 5.6.2
Pergunta: Qual endpoint eu chamo para cada modo de chat?

Resposta:
Cada modo tem seu boundary próprio:
- `qa` (RAG / pergunta e resposta) -> `POST /rag/execute`, com envelope `{ operation: "ask", payload: { ... } }`;
- `agent` e `deepagent` -> `POST /agent/execute`, com os campos no topo do corpo (`deepagent` força `mode: "deepagent"`);
- `workflow` -> `POST /workflow/execute`.
O handshake criptográfico é sempre o mesmo: `POST /crypto/session-key` antes de cifrar o YAML.
Não invente uma rota nova por modo; o mapa acima é o contrato real.

Onde no código:
- `app/ui/static/js/shared/layout-mestre-api.js` - `LAYOUT_API_CONFIG.endpoints`
- `src/api/routers/rag_router.py`, `src/api/routers/agent_router.py`, `src/api/routers/workflow_router.py`

### 5.6.3
Pergunta: A resposta não trouxe `correlation_id`. E agora?

Resposta:
Primeiro, leia o `correlation_id` nos dois lugares possíveis: o header `X-Correlation-Id` e o corpo
JSON (`correlation_id`, `correlationId`, `log_id`, `interaction_id`, `detail.correlation_id`). O contrato
exige que o backend devolva o `correlation_id` oficial inclusive em erro.
O cliente **nunca** inventa esse valor — se ele faltar de verdade na resposta, isso é falha de produto e
de observabilidade do backend, não algo para o cliente contornar gerando um id local.
Há um caso conhecido: erro 500 em modo debug pode vir como traceback cru sem o header; nesse caso o id
existe no log do backend (localizável por timestamp via `python -m src.log_analyzer`), só não chegou ao
cliente.

Onde no código:
- `app/ui/static/js/shared/layout-mestre-api.js` - `extrairCorrelationIdResposta`
- `src/api/service_api.py` - `log_requests`

### 5.6.4
Pergunta: Como funciona a execução assíncrona (polling) do chat?

Resposta:
Existem três modos de execução: `auto`, `direct_sync` e `direct_async`. Em `direct_async` (ou quando o
backend responde `HTTP 202`, ou quando vem `status_url`/`polling_url`/`stream_url` no corpo), o cliente
não recebe a resposta final na hora. Ele pega o `task_id` e faz polling em
`GET /api/v1/status/{task_id}` (enviando o header `X-API-Key`), a cada ~1s, até o status virar terminal
(`completed`, `failed`, `cancelled`) ou de pausa (`paused`, `awaiting_human_decision`). O resultado final
sai de `result`/`data` do payload de status. O componente embutível faz esse polling automaticamente; um
cliente próprio precisa implementá-lo.

Onde no código:
- `app/ui/static/js/shared/layout-mestre-api.js` - `_extrairInfoAssincrona`
- `app/ui/static/js/shared/ui-webchat-async-runtime.js` - `waitForTaskCompletion`
- `src/api/routers/streaming_router.py` (`/api/v1/status/{task_id}`)

### 5.6.5
Pergunta: Devo construir meu próprio cliente HTTP ou usar o componente pronto?

Resposta:
Para telas da própria plataforma, a opção padrão é **embutir** o componente oficial
`PrometeuEmbeddableChatRuntime` — ele já cuida de handshake, cifra, envio por modo, polling e
`correlation_id`, sem reabrir contrato. Criar um runtime de chat paralelo numa tela é violação de reuso.
Construir cliente próprio (em Python, Node, etc.) só faz sentido para integração externa fora do browser
da plataforma; nesse caso, siga o mesmo contrato de payload e criptografia dos exemplos canônicos, sem
inventar formato.

Onde no código:
- `app/ui/static/js/shared/embeddable-chat-runtime.js`
- `.claude/rules/componente-chat-embutivel.md`
- `examples/rag_api_client.py`, `examples/rag_api_client.js` (clientes externos de referência)

## 6. Tools avançadas

### 6.1
Pergunta: Como uma tool entra no catálogo builtin?

Resposta:
A tool precisa existir no código seguindo o padrão de decorators do projeto.
Depois disso, o catálogo builtin é sincronizado pelo builder oficial.
O builder também injeta factories parametrizadas base, como `dyn_sql` e `dyn_api`.
Isso evita cadastro manual frágil e reduz desvio entre catálogo e runtime.

Onde no código:
- `src/agentic_layer/tools/tools_library_builder.py`
- `src/agentic_layer/tools/tool_factory_decorator.py`

### 6.2
Pergunta: Onde vive a SQL dinâmica?

Resposta:
SQL dinâmica vive em uma factory especializada de domain tools.
Ela suporta modo parametrizado, como `dyn_sql<query_id>`, e modo completo.
Na prática, isso permite expor queries governadas sem espalhar SQL ad hoc em qualquer lugar do sistema.
Se você quiser mexer nesse comportamento, mexa na factory, não no worker de ingestão.

Onde no código:
- `src/agentic_layer/tools/domain_tools/dynamic_sql_tools/dynamic_sql_factory.py` - `create_dynamic_sql_tool`
- `src/agentic_layer/tools/tools_library_builder.py`

### 6.3
Pergunta: Onde vive a API dinâmica?

Resposta:
API dinâmica também nasce por factory especializada.
Ela suporta sintaxe parametrizada como `dyn_api<endpoint_id>` e resolve endpoints do YAML.
Isso organiza método HTTP, URL, headers e cache no lugar certo.
O ganho prático é não transformar integração REST em lógica dispersa por vários agentes.

Onde no código:
- `src/agentic_layer/tools/domain_tools/dynamic_api_tools/dynamic_api_factory.py` - `create_dynamic_api_tool`
- `src/agentic_layer/tools/tools_library_builder.py`

### 6.4
Pergunta: Onde vive o fluxo de NL2SQL?

Resposta:
O fluxo dedicado entra por `/config/nl2sql`.
Ele valida `yaml_config`, `prompt`, `user_email`, dialeto e aplica guardrails antes de devolver uma SQL revisável.
Isso quer dizer que NL2SQL não é só “perguntar e receber SQL”.
Existe contrato HTTP, diagnósticos estruturados e bloqueio quando o guardrail reprova a saída.

Onde no código:
- `src/api/routers/config_nl2sql_router.py`
- `src/api/services/nl2sql_service.py` - `Nl2SqlService`
- `src/api/schemas/nl2sql_models.py`

### 6.5
Pergunta: Onde vive a geração de YAML agentic por linguagem natural?

Resposta:
O endpoint público fica em `/config/assembly/objective-to-yaml`.
Ele depende do feature flag de AST agentic e da permissão correta para geração de configuração.
No backend, a resposta passa por `AgenticAssemblyService`, que não trata esse YAML como texto livre.
O objetivo é sair de linguagem natural para YAML governado com validação real no meio do caminho.

Onde no código:
- `src/api/routers/config_assembly_router.py` - `objective_to_yaml_agentic_ast`
- `src/config/agentic_assembly/assembly_service.py` - `AgenticAssemblyService`

### 6.6
Pergunta: Quando eu devo parar e pedir ajuda em vez de criar um caminho paralelo?

Resposta:
Pare quando a mudança exigir novo caminho de YAML, novo fallback implícito, novo atalho de fila ou paralelismo para fonte local não compartilhada.
Esses são sinais clássicos de que você está saindo do trilho canônico do projeto.
Também pare se a feature já tiver endpoint, factory ou assembly oficial e a sua ideia for montar outro fluxo ao lado.
No onboarding, pedir ajuda cedo aqui evita criar solução que parece rápida e vira dívida estrutural logo depois.

Onde no código:
- `src/config/agentic_assembly/assembly_service.py`
- `src/services/document_fanout_coordinator.py`
- `src/agentic_layer/tools/domain_tools/dynamic_sql_tools/dynamic_sql_factory.py`
- `src/agentic_layer/tools/domain_tools/dynamic_api_tools/dynamic_api_factory.py`
