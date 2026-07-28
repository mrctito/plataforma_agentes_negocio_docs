# Agendamento agentic, background e HIL: visão conceitual

## 1. O problema

Um agente nem sempre deve executar dentro do request que recebeu a demanda.
Há trabalhos que precisam rodar de madrugada, repetir toda semana, sobreviver a
restart, aguardar decisão humana ou entregar o resultado a um canal depois.

Essa capacidade transforma DeepAgent e WorkflowAgent em processos duráveis,
sem transformar o processo da API em um worker improvisado.

## 2. Três responsabilidades

Pense em três cadernos separados:

- Scheduler anota quando executar e qual comando transportar;
- Job Core anota o estado operacional do job;
- domínio agentic anota o alvo, o resultado e as decisões funcionais.

Essa separação evita que duas tabelas tentem dizer, ao mesmo tempo, se o mesmo
job está running, failed ou completed.

## 3. Fluxo principal

```text
usuário/agente
  -> agenda um alvo
  -> Scheduler aguarda a data
  -> Job Core recebe a ocorrência
  -> worker executa DeepAgent ou WorkflowAgent
  -> resultado factual
  -> canal/webchat
```

Os alvos atuais são `deepagent` e `workflow`. O agendamento resolve um alvo
previamente autorizado para o tenant; ele não aceita referência arbitrária.

## 4. Quando há aprovação humana

```text
primeiro job
  -> encontra ação sensível
  -> registra approval_requested
  -> cria aprovação durável
  -> notifica aprovador
  -> pessoa decide
  -> segundo job
  -> retoma a mesma thread
```

O primeiro e o segundo job são execuções diferentes e possuem correlações
diferentes. A thread é a mesma porque o estado lógico precisa continuar.

WorkflowAgent e DeepAgent usam esse desenho. Uma nova pausa pode criar uma
nova rodada HIL.

## 5. Por que dois jobs

Esperar uma pessoa pode levar minutos ou dias. Manter um processo ou request
aberto durante esse período seria frágil, caro e incompatível com o runtime
stateless e multi-processo.

O primeiro job termina depois de registrar a pendência. A decisão posterior
publica um segundo job. Assim, o sistema pode reiniciar, escalar workers e
repetir tentativas sem perder a intenção original.

## 6. `direct_async` versus agendamento

Os dois caminhos usam infraestrutura durável, mas resolvem necessidades
diferentes:

| Necessidade | Caminho |
| --- | --- |
| executar agora, sem prender a resposta HTTP | `direct_async` |
| executar depois ou de forma recorrente | Scheduler |

`direct_async` de WorkflowAgent publica no Job Core. Ele não é uma tarefa
efêmera de `BackgroundTasks`.

## 7. O que o domínio agentic guarda

O domínio guarda quatro famílias de fatos:

1. alvos habilitados por tenant;
2. contexto e resultado/erro funcional da ocorrência;
3. pedidos e decisões HIL;
4. comunicações aguardando entrega ou confirmação.

Ele não guarda agenda própria, eventos próprios ou lifecycle duplicado.

## 8. O que significa cada identidade

| Identidade | Significado |
| --- | --- |
| `request_id` | intenção carregada na agenda |
| `schedule_id` | agenda no Scheduler |
| `job_id` | execução operacional no Job Core |
| `run_id` | fato funcional agentic |
| `correlation_id` | trilha de uma execução |
| `thread_id` | estado retomável do agente/grafo |
| `approval_request_id` | rodada de decisão humana |

Misturar esses ids produz diagnósticos errados. Por exemplo, `run_id` não
substitui o estado do `job_id`.

## 9. Comunicação assíncrona

Terminar o agente e entregar a resposta são fatos diferentes. Um canal pode
estar indisponível mesmo quando o trabalho terminou corretamente.

O outbox registra a comunicação pendente e permite retry, dead letter e ack
sem reexecutar o agente. Isso reduz duplicação de ações e separa falha de
negócio de falha de entrega.

## 10. Exemplos

### Conciliação financeira noturna

O Scheduler dispara um WorkflowAgent. O grafo consulta dados, encontra uma
divergência de alto valor e pede aprovação. Após o aprovador decidir, um
segundo job retoma e publica o consolidado no webchat.

### Relatório semanal de atendimento

Um DeepAgent roda toda segunda-feira, reúne fatos por Tools governadas e gera
o resumo. O outbox entrega o resultado sem manter o primeiro request aberto.

### Reprocessamento com decisão

Um workflow identifica itens inválidos. Em vez de corrigir automaticamente,
pausa para o operador escolher aprovar, editar ou rejeitar. A decisão fica
auditável e a continuação usa a mesma thread.

## 11. Segurança e multi-tenant

Ambiente e tenant participam das identidades persistentes. Um usuário só pode
agendar alvo e consultar fatos do próprio escopo, salvo permissão
administrativa explícita.

Snapshots do YAML são validados e reidratados pelo fluxo oficial. Tokens,
segredos e configuração resolvida não devem ser enviados ao log ou ao canal.

## 12. Como operar

Para responder “o que aconteceu?”, consulte a fonte certa:

1. Scheduler para agenda e próxima execução;
2. Job Core para lifecycle, tentativas e eventos;
3. run factual para resultado ou erro agentic;
4. pedido HIL para pendência/decisão;
5. outbox para entrega ao canal;
6. log oficial pela correlação para reconstruir a história.

Não conclua “não executou” apenas porque uma projeção factual está vazia. A
falha pode ter ocorrido antes da materialização; o Job Core e o log fecham essa
lacuna.

## 13. Limites e pegadinhas

- `approval_requested` é resultado funcional, não status de job.
- `202` na decisão HIL confirma o segundo job, não seu término.
- Cancelar a agenda não apaga o histórico nem necessariamente cancela uma
  ocorrência já publicada.
- O outbox vazio não prova que o agente não executou.
- Não existem tabelas agentic atuais para requests, schedules ou events.
- A tabela HIL legada em `public` não é a fonte do runtime atual.

## 14. Próximas leituras

- [Manual técnico de background e HIL](../tecnico/README-TECNICO-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md)
- [Manual técnico de HIL e canais](../tecnico/README-TECNICO-HIL-APIS-WHATSAPP.md)
- [README do schema físico](../tecnico/README-SCHEMA-BANCO.md)
- [Tutorial de WorkflowAgent, Skills e HIL](../usuario/TUTORIAL-101-WORKFLOW-SKILLS-HIL.md)

## 15. Evidências executáveis

- `src/agentic_layer/background_execution/models.py`
- `src/agentic_layer/background_execution/services.py`
- `src/agentic_layer/background_execution/runtime.py`
- `src/scheduler_layer/job_processes.py`
- `src/api/services/background_hil_continuation_submission_service.py`
- `src/api/services/background_hil_continuation_execution_service.py`
- `src/api/services/background_execution_communication_service.py`
