# Sistema Genérico de Jobs

O Job Core é a infraestrutura única para trabalhos assíncronos da plataforma. Ingestão, ETL, backup, conciliação, relatório e futuras rotinas usam o mesmo ciclo de vida; cada domínio fornece apenas seu handler e seu payload.

Em linguagem simples: o Job Core é quem manda em tudo que é "vida e morte" de um trabalho assíncrono
(quando começa, quando pode rodar em paralelo, quando morreu e precisa ser encerrado, quando
termina). Cada domínio — ingestão, ETL, agendamento — é só um **usuário/hóspede** desse motor: ele
sabe processar seu próprio conteúdo (um PDF, uma linha de ETL), mas nunca decide sozinho concorrência,
retentativa, detecção de travamento ou cancelamento. Se um domínio precisar de uma capacidade nova de
execução, a resposta correta é pedir que o Job Core ganhe essa capacidade de forma genérica, nunca
implementar um atalho local. Regra completa (normativa): `src/CLAUDE.md` Parte 4.

## Uma fonte de verdade

`job_core.job_runs` é simultaneamente a fila durável e o estado atual do job. A API persiste o envelope completo com status `queued`; um worker compatível faz claim atômico e muda o estado até um resultado terminal.

`job_core.job_run_events` é a auditoria append-only. Ela conta a sequência de decisões sem competir com `job_runs` pelo estado atual.

Essa divisão responde duas perguntas diferentes:

- `job_runs`: como o job está agora;
- `job_run_events`: o que aconteceu durante a execução.

## Contrato genérico

O envelope fixa apenas identidade, rota, handler, correlação, hierarquia e payload opaco. Campos próprios de um domínio ficam no JSON do envelope; criar um novo tipo de job não exige coluna ou tabela nova no Job Core.

`route_kind + dispatch_mode` seleciona o handler. O worker anuncia as rotas que sabe executar e só disputa jobs dessas rotas. `parent_job_id + root_job_id` preserva fan-out e árvores de execução.

## Concorrência, cancelamento e recuperação

O claim usa ordenação FIFO e `FOR UPDATE SKIP LOCKED`, permitindo vários workers sem claim duplicado. A `concurrency_key` opcional impede duas execuções ativas da mesma operação e alvo.

Cancelamento é uma transição persistida no próprio ledger. Jobs ainda enfileirados podem terminar imediatamente como cancelados; jobs em execução observam o pedido cooperativamente. Heartbeat e reconciliação permitem identificar trabalho órfão sem depender de memória de processo.

## Observabilidade

O `correlation_id` atravessa API, worker, eventos e logs canônicos. Publicação, claim, início, decisões relevantes, heartbeat, cancelamento, reconciliação e fechamento devem produzir evidência estruturada. A interface deriva detalhes e progresso do log canônico; não existe callback de progresso em Redis.

## Gestão

A listagem e a exclusão são escopadas por tenant e solicitante. Somente jobs terminais podem ser removidos. “Excluir todos” significa excluir todos os jobs terminais visíveis naquele escopo; jobs ativos nunca entram na operação.

Para o contrato físico e a migração do ledger antigo, consulte [README-SCHEMA-BANCO.md](../tecnico/README-SCHEMA-BANCO.md#domínio-job-core). Para o fluxo executável, consulte [README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md](../tecnico/README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md).
