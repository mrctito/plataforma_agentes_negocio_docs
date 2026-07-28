# HIL, APIs e WhatsApp: visão conceitual

## 1. O que é HIL

Human-in-the-Loop é a capacidade de interromper uma automação em um ponto
sensível, apresentar o contexto a uma pessoa e retomar exatamente a mesma
thread com uma decisão registrada.

Não é apenas “mandar uma pergunta para alguém”. HIL precisa garantir:

- qual ação está pendente;
- quais decisões são permitidas;
- quem pode decidir;
- até quando a decisão vale;
- qual thread deve continuar;
- como impedir aprovação acidental ou duplicada;
- como auditar o primeiro job e a continuação.

## 2. Por que existe

Há ações que não devem ser executadas apenas porque um modelo as sugeriu:
pagamentos, mensagens externas, alterações de registro, exclusões e decisões
com impacto financeiro ou regulatório.

HIL permite automatizar a preparação sem automatizar cegamente a autorização.
O sistema chega até o ponto de decisão, um humano revisa, e só então a mesma
execução prossegue.

## 3. Dois contextos

### Foreground

A pessoa ainda está na interação original. A API devolve uma pausa e o cliente
chama `/agent/continue` ou `/workflow/continue` com a mesma thread e a mesma
correlação.

### Background

O trabalho acontece fora do request e pode esperar horas. A plataforma cria um
pedido durável, notifica aprovadores e, depois da decisão, publica um segundo
job para continuar.

O segundo desenho é necessário porque não é seguro manter um processo web
aberto esperando uma pessoa.

## 4. A jornada durável

```text
primeiro job
  -> ação sensível
  -> pedido HIL pendente
  -> WhatsApp/e-mail/UI segura
  -> decisão validada
  -> segundo job
  -> mesma thread
  -> resultado ou nova rodada HIL
```

Cada job tem seu próprio `correlation_id`. O vínculo entre eles é preservado,
mas a identidade não é reciclada.

## 5. As quatro decisões

| Decisão | Significado |
| --- | --- |
| `approve` | autoriza a ação como proposta |
| `edit` | altera os argumentos e autoriza a versão revisada |
| `reject` | recusa a ação |
| `respond` | devolve uma resposta humana ao agente |

A interface mostra apenas as decisões permitidas no pedido. `edit` exige a
ação revisada; não é apenas um rótulo de botão.

## 6. O papel dos canais

WhatsApp e e-mail são meios de entrega, não donos da regra HIL. O mesmo serviço
central valida token, prazo, aprovador e decisão.

No WhatsApp:

- aprovar e rejeitar podem ser botões diretos;
- editar abre uma página segura, porque argumentos estruturados não cabem em
  uma decisão curta;
- a resposta volta pelo webhook normal;
- uma bridge reconhece o payload HIL antes que ele seja tratado como conversa.

Assim, um clique de aprovação não vira também uma nova mensagem para o agente.

## 7. Segurança da revisão

O token é uma credencial temporária. O banco guarda apenas seu hash e um hint.
O token não deve aparecer em log nem em query string.

Uma página de revisão pode receber o segredo no fragmento `#` da URL. O
fragmento fica no navegador; a página o envia em `POST` para carregar o
contexto e registrar a decisão.

O serviço também verifica identidade e canal esperados. Ter o token não ignora
automaticamente a política de aprovadores.

## 8. O que significam `200` e `202`

`POST /agent/hil/decisions` possui dois sucessos:

- `200 resolved`: a continuação ocorreu dentro da chamada e o resultado vem em
  `continuation`;
- `202 continuation_submission_confirmed`: um segundo job foi publicado e a
  resposta traz `continuation_job_id`.

`202` não significa que o agente terminou. Significa que a plataforma aceitou
o próximo trabalho durável. A UI deve acompanhar o Job Core antes de mostrar
sucesso final.

## 9. WorkflowAgent também é suportado

Aprovação durável não é exclusiva de DeepAgent. Quando um WorkflowAgent em
background pausa, a decisão publica um segundo job que retoma a mesma thread.
Esse segundo job também pode gerar uma nova interrupção e uma nova aprovação.

Workflow aceita uma ação/decisão por retomada assíncrona. Isso mantém o ponto
de continuação inequívoco.

## 10. Onde cada fato vive

| Dado | Dono |
| --- | --- |
| Agenda | Scheduler |
| Lifecycle, lease, tentativas e eventos | Job Core |
| Pedido HIL, decisão e notificação | `agent_background.agent_hil_approval_requests` |
| Estado pausado | checkpointer da thread |
| Comunicação pendente ao webchat/canal | `agent_background.background_execution_outbox` |

Essa divisão evita que uma tabela de domínio tente competir com o Job Core
sobre o estado da execução.

## 11. Exemplo de negócio

Uma conciliação encontra um pagamento de alto valor:

1. o WorkflowAgent prepara os dados;
2. antes de enviar o pagamento, pausa;
3. o aprovador recebe WhatsApp com “Aprovar”, “Rejeitar” e link de revisão;
4. ele abre o link, reduz o valor e envia `edit`;
5. a plataforma valida sua identidade;
6. um segundo job retoma a mesma thread com os argumentos editados;
7. a trilha liga ação original, decisão e execução final.

O benefício é claro: a automação faz o trabalho repetitivo, mas a autorização
sensível continua humana e auditável.

## 12. Pegadinhas

- Aprovação por GET é insegura.
- Reenviar a decisão sem consultar o estado pode criar conflito.
- `202` exige polling/stream do novo job.
- `edit` não deve ser reduzido a um botão sem payload revisado.
- Uma nova pausa cria uma nova aprovação; não reabre o pedido antigo.
- A correlação do primeiro job não deve ser usada como identidade do segundo.
- A tabela legada em `public` não é a fonte do runtime atual.

## 13. Como implementar uma integração

1. Renderize `action_requests`, `review_configs` e `allowed_decisions`.
2. Guarde `thread_id`, `approval_request_id` e as correlações recebidas.
3. Envie token e decisão somente por `POST`.
4. Trate `200` e `202` como branches diferentes.
5. No branch `202`, acompanhe `continuation_job_id`.
6. Nunca registre token, YAML ou argumentos sensíveis em logs de cliente.

Veja exemplos completos no [manual técnico de HIL](../tecnico/README-TECNICO-HIL-APIS-WHATSAPP.md)
e no [tutorial de WorkflowAgent, Skills e HIL](../usuario/TUTORIAL-101-WORKFLOW-SKILLS-HIL.md).

## 14. Evidências executáveis

- `src/api/schemas/agent_hil_models.py`
- `src/api/services/hil_approval_decision_service.py`
- `src/api/services/hil_approval_notification_service.py`
- `src/api/services/hil_approval_channel_bridge.py`
- `src/api/services/background_hil_continuation_submission_service.py`
- `src/api/services/background_hil_continuation_execution_service.py`
- `src/api/services/workflow_hil_continuation_service.py`
- `src/api/repositories/agent_hil_approval_requests_repository.py`
