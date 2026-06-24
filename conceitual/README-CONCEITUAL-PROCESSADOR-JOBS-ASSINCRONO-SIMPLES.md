# Processador de Jobs Assíncrono Simples (spec-101)

## Atenção antes de ler

Este documento descreve um mecanismo distinto do **Job Core genérico** (RabbitMQ + Dramatiq + worker + fan-out por documento). Os dois mecanismos coexistem no produto. Se a sua dúvida for sobre o Job Core genérico, leia primeiro:

- [README-CONCEITUAL-SISTEMA-JOBS-WORKER-PARALELISMO.md](README-CONCEITUAL-SISTEMA-JOBS-WORKER-PARALELISMO.md)

**Nota de posicionamento estratégico:** o Job Core genérico (Dramatiq + Worker dedicado) está marcado como legado em depreciação futura. O processador simples (spec-101) é a direção de evolução para ingestão assíncrona de PDF na plataforma.

---

## 1. O que é este processador

O Processador de Jobs Assíncrono Simples, também chamado de spec-101, é um segundo mecanismo de processamento assíncrono do produto, criado especificamente para ingestão de PDFs de forma direta, sem passar pelo Worker dedicado do Job Core genérico.

A diferença central está em onde o trabalho nasce e onde ele é executado. No Job Core genérico, o endpoint publica um envelope no RabbitMQ e um Worker separado consome esse envelope, reserva slot e processa. No spec-101, a API grava o job numa tabela PostgreSQL, envia um sinal curto de alarme via RabbitMQ, e o **processo worker oficial** acorda um dispatcher próprio da spec-101. Esse dispatcher, já no worker, cria um processo operacional temporário para executar o job.

Em linguagem simples: o Job Core usa uma fila longa e um trabalhador permanente. O spec-101 usa uma fila curta de wakeup, uma tabela simples como fila durável e um processo temporário criado sob demanda no runtime do worker.

---

## 2. Analogia didática

Imagine uma padaria com dois balcões.

O **balcão do Job Core** recebe pedidos complexos — bolos decorados, encomendas de casamento — e os coloca numa esteira que passa por vários setores da fábrica: embalagem, decoração, conferência e entrega. O gerente da fábrica (Worker) coordena cada etapa da esteira. É robusto, mas tem vários passos.

O **balcão do spec-101** recebe pedidos de pão simples. O atendente anota o pedido numa lista (tabela do banco), toca a sineta (mensagem RabbitMQ de alarme), e um padeiro temporário (processo Python `multiprocessing`) é chamado para pegar a lista, pegar os PDFs e assar em paralelo. Quando termina, o padeiro vai embora e o status da lista é atualizado.

A vantagem do balcão spec-101 é que ele é mais direto para trabalhos de ingestão de PDF onde o fluxo não precisa do Worker dedicado.

---

## 3. Problema que ele resolve

O Job Core genérico usa Dramatiq e um Worker separado como processo de consumo permanente. Esse Worker tem overhead de inicialização e gerenciamento de threads. Para o cenário de ingestão de PDFs simples, precisava de um caminho mais leve que:

- aceitasse o pedido imediatamente e devolvesse um `job_id` para acompanhamento;
- gravasse o estado de forma durável (PostgreSQL) sem depender da memória volátil;
- processasse os PDFs em paralelo com um pool de runners controlado;
- isolasse cada PDF de forma que um erro num PDF não interrompesse os outros;
- atualizasse o status final garantido (`success`, `error` ou `cancelled`) ao terminar;
- deixasse o dashboard de ingestão enxergar o job mesmo sem passar pelo ledger completo do Job Core genérico.

---

## 4. Fluxo em linguagem não-técnica

O fluxo do spec-101 hoje pode ser entendido em nove etapas.

**Etapa 1 — Recepção do pedido.** O usuário ou sistema chama o endpoint de ingestão. A API cria uma linha na tabela `job_core.async_jobs` com status `pending`, grava os parâmetros do job no campo JSON, preserva `tenant_code` e `vectorstore_id` para leitura operacional e devolve o `job_id` imediatamente. O usuário não espera o processamento terminar.

**Etapa 2 — Alarme para o dispatcher.** Junto com a criação da linha, a API publica uma mensagem curta na fila `ingestion_dispatcher_wakeup`. Essa mensagem não carrega o trabalho: ela apenas acorda o dispatcher com `correlation_id` e `job_id`.

**Etapa 3 — Worker acorda o dispatcher da spec-101.** O processo worker oficial mantém um loop dedicado para essa fila. Ao receber o wakeup, ele consulta o banco, conta quantos jobs estão ativos, reconcilia jobs presos sem atividade há tempo demais e decide se há slot para iniciar o próximo job.

**Etapa 4 — Criação do processo do job.** Para cada job que couber nos slots disponíveis, o dispatcher cria um processo separado (`multiprocessing.Process` com `daemon=False`). O processo não pode ser daemon porque ele precisa criar subprocessos runners.

**Etapa 5 — Execução dentro do processo do job.** O processo marca o job como `processing`, persiste o `runner_pgid` do líder para eventual cancelamento e resolve a lista de PDFs que o job deve processar.

**Etapa 6 — Processamento paralelo dos PDFs.** O processo cria N runners e distribui os PDFs via fila em memória. Cada runner processa um PDF por vez usando o pipeline real de ingestão. Um erro num PDF não derruba os outros.

**Etapa 7 — Fechamento normal.** O processo do job consolida os resultados, conta sucessos e erros e grava o status final (`success` se todos os PDFs passaram, `error` se algum falhou) com a mensagem de resumo.

**Etapa 8 — Cancelamento simples, quando pedido.** Se o usuário solicitar cancelamento, a API não mata o job diretamente. Ela publica um wakeup na fila `ingestion_killer_wakeup`. O Killer, rodando no worker, marca o job como `cancelling`, envia `SIGTERM` ao processo líder para ele drenar o lote e, se necessário, escala para `SIGKILL` do grupo inteiro. Quando termina, fecha o job como `cancelled`.

**Etapa 9 — Leitura operacional.** O dashboard de ingestão não tenta reconstruir o lote inteiro como se fosse o Job Core genérico. Para a spec-101 ele mostra um detalhe mínimo: status, tempos, `correlation_id`, `vectorstore_id`, metadados principais e link para os logs centrais.

---

## 5. Diferença clara entre spec-101 e Job Core genérico

| Aspecto | Job Core genérico | spec-101 |
| --- | --- | --- |
| Transporte principal | RabbitMQ com envelopes versionados | Tabela PostgreSQL como fila + RabbitMQ como alarme |
| Consumidor | Worker dedicado (processo permanente com Dramatiq) | Worker oficial com dispatcher próprio da spec-101 + processo temporário por job |
| Paralelismo | Fan-out documental via jobs filhos no Worker | Pool de runners em memória dentro do processo do job |
| Ledger de lifecycle | `PostgresJobRunStore` (tabelas dedicadas do Job Core) | `job_core.async_jobs` (tabela única simples) |
| Rastreamento de PDF individual | Job filho com `worker_execution_correlation_id` próprio | Resultado em memória, log com `correlation_id` do job |
| Uso de Dramatiq | Sim, como runtime do Worker | Não |
| Status do job | Ciclo completo do Job Core (`PENDING`, `RUNNING`, etc.) | Ciclo simples: `pending → processing → success/error`, com trilha de cancelamento `cancelling → cancelled` |
| Status atual | Legado em depreciação futura | Direção de evolução da plataforma |

---

## 6. Quando usar cada um

**Use o Job Core genérico quando:**

- o job não é de ingestão de PDF (ETL, background execution, etc.);
- você precisa do ciclo de vida completo do Job Core com eventos auditados;
- o handler de domínio já está registrado no registry do Job Core;
- o rastreamento individual de cada documento filho (via `worker_execution_correlation_id`) é necessário no dashboard.

**Use o spec-101 quando:**

- o pedido é de ingestão de PDF e o fluxo simples é suficiente;
- o objetivo é devolver `job_id` imediatamente e processar em background;
- a configuração YAML de ingestão já está disponível e o plano de fan-out resolve os PDFs corretamente;
- a simplicidade operacional da tabela única é aceitável.

---

## 7. Estado atual e limitações conhecidas

### O processamento de PDFs funciona de ponta a ponta

O mecanismo está operacional. O processo do job é criado com `daemon=False`, permitindo que ele crie os subprocessos runners. Os runners processam os PDFs usando o pipeline real de ingestão e reportam progresso autêntico via callback. Existe teste de regressão que falha se `daemon=True` for reintroduzido no processo do job.

**Ressalva honesta:** a validação foi realizada pela suíte de testes focada (e2e offline). Uma ingestão real end-to-end com instâncias de Postgres e Qdrant em produção não foi documentada como executada após essa correção. O código está pronto; a validação em infraestrutura real é o próximo grau de confiança.

### Pools de banco de dados fork-safe

Cada runner cria seu próprio pool de conexão com o PostgreSQL após o fork, sem reutilizar o pool do processo pai. Isso evita corrupção de conexões em cenário multiprocesso e está integrado desde o commit `4401577ff`.

### Reconciliação básica de jobs presos

Se um processo de job terminar de forma inesperada, o mecanismo atual já possui uma reconciliação básica no próprio store. Quando o dispatcher acorda, ele pode marcar como `error` um job que ficou em `processing` sem atividade por tempo demais. Isso libera slots travados sem depender de intervenção manual imediata.

O que essa reconciliação **não** faz:

- não recupera o processamento do ponto em que parou;
- não reconstrói resultado parcial útil;
- não substitui um reconciliador mais sofisticado.

Em linguagem simples: ela evita que o pátio fique bloqueado por um caminhão quebrado, mas não conserta a carga.

### Cancelamento simples via Killer

O spec-101 hoje já tem cancelamento simples, mas ele não funciona como cancelamento fino por documento. O caminho atual é:

- a API acorda o Killer pela fila própria;
- o Killer marca o job como `cancelling`;
- o processo líder recebe `SIGTERM` para parar de pegar novos PDFs;
- se o processo não terminar na carência, o Killer escala para `SIGKILL`;
- o job fecha como `cancelled`.

Limites honestos do modelo atual:

- o cancelamento é por lote, não por PDF individual;
- o callback de progresso do pipeline não faz cancelamento cooperativo fino no meio de um mesmo PDF;
- o Killer atual pressupõe o cenário operacional de um worker por host, porque atua no `process-group` local gravado em `runner_pgid`.

### Dashboard mínimo da spec-101

O dashboard de ingestão já enxerga jobs da spec-101, mas de forma propositalmente mínima. A filosofia é:

- o banco mostra o estado básico do lote;
- o log conta a história detalhada;
- não existe preview real de documentos-filhos como no Job Core genérico;
- o usuário acompanha detalhe e abre os logs centrais pelo `correlation_id`.

---

## 8. Impacto executivo e estratégico

O spec-101 representa uma estratégia de simplificação deliberada. Em vez de especializar o Job Core genérico para o caso de PDF simples, ele cria um caminho mais direto com uma tabela única e um dispatcher leve.

O valor estratégico é tornar o processamento de PDF rastreável via banco sem depender da fila RabbitMQ como ledger durável. Qualquer consulta SQL em `job_core.async_jobs` mostra o estado de todos os jobs de ingestão, com status simples, mensagem final e timestamps de início e fim — sem precisar passar pelo Job Core.

O posicionamento como direção futura da plataforma significa que melhorias em observabilidade, cancelamento, dashboard e reconciliação continuarão sendo feitas neste mecanismo, não no Job Core genérico.

Para referência cruzada com o mecanismo genérico, leia:

- [README-CONCEITUAL-SISTEMA-JOBS-WORKER-PARALELISMO.md](README-CONCEITUAL-SISTEMA-JOBS-WORKER-PARALELISMO.md)
- [README-TECNICO-PROCESSADOR-JOBS-ASSINCRONO-SIMPLES.md](../tecnico/README-TECNICO-PROCESSADOR-JOBS-ASSINCRONO-SIMPLES.md)
