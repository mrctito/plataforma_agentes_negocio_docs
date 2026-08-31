# Runbook operacional das integrações do MVP Aidan

## Objetivo e limite

Este runbook serve para liberar, observar, revogar e reverter somente os seis serviços que a tela
**Integrações e canais** apresenta hoje. Ele não cria conector, credencial, tabela ou política nova.
Nunca copie token, senha ou `secret_ref` para chamado, log, captura ou comando.

A sequência comum é **tenant → conexão → capability**. O tenant inicial é `aidan`; o rollout só
avança depois de a conexão exata e a capability exata passarem. Uma tela carregada ou um cadastro
`configured` não substituem probe nem ação real.

## Gate comum de rollout

1. Confirme tenant, ambiente e `connection_code` na resposta factual de
   `GET /admin/integrations/capabilities`.
2. Confirme somente as capabilities necessárias à jornada que será liberada.
3. Execute **Verificar conexões agora** e guarde o `correlation_id`.
4. Faça uma ação real e permitida; acompanhe callbacks, probes, respostas **401 / 403 / 429**,
   latência, backlog e duplicatas.
5. Avance para a próxima capability apenas se o fato persistido e o efeito externo concordarem.
6. Em falha, pare a ação mutante. Não habilite uma segunda toolkit, não copie credencial para YAML
   e permaneça **sem fallback de produção em `.env`**.

## Meta e Instagram [provider-meta-instagram]

- **Owner de conexão:** `integrations.provider_connection_registry`, conexão
  `instagram_business`; segredo resolvido server-side pela referência persistida.
- **Prova mínima:** autorização Meta concluída, capabilities concedidas, probe da conta e uma
  leitura real de mídia/comentário. Webhook exige callback público e permissões Meta aprovadas.
- **Revogação Aidan:** um owner do clube usa **Desconectar esta conexão**, que chama
  `POST /admin/integrations/connections/{connection_code}/revoke`. O registry passa a `revoked`,
  zera capabilities e a projeção passa a status `unavailable`; a referência fica apenas para
  auditoria e o resolvedor não abre o cofre.
- **Revogação externa:** se a credencial estiver comprometida, remova também o acesso no painel da
  Meta. Essa ação humana invalida o token na origem; não apaga publicações, comentários ou mensagens.
- **Prova negativa:** repita uma leitura pelo endpoint de conteúdo. Ela deve falhar antes da Graph API.

## WhatsApp [provider-whatsapp]

- **Owner atual:** `public.tenant_channels`, não o registry do Instagram. O envio só seleciona canal
  `active`; webhook, credencial e fila são fatos separados na tela.
- **Prova mínima:** challenge do webhook, mensagem real recebida, resposta permitida e fila sem
  duplicata, todos com correlação reconstruível.
- **Desativação Aidan:** um gestor de canais usa **Desativar este canal**, que chama
  `POST /admin/aidan/channels/{channel_id}/deactivate`. O owner `tenant_channels` passa a `inactive`,
  preserva configuração e grava ator + `correlation_id` na mesma transação. Novos envios deixam de
  resolver o canal; se um item já foi retirado da fila na corrida da desativação, o worker o bloqueia
  antes do agente e registra `channel.worker.message_blocked_channel_inactive`, sem fingir requeue.
- **Credencial comprometida:** revogue também o acesso no painel Meta. A desativação no Aidan corta o
  uso pela plataforma, mas não invalida o token na origem nem apaga dados do WhatsApp.
- **Prova negativa:** depois da desativação, uma tentativa nova de envio precisa falhar antes da Meta e
  uma mensagem enfileirada não pode chamar o processador/agente. Nunca use `DELETE` ou DML ad hoc.

## Caixa de e-mail [provider-email]

- **Owner atual:** configuração protegida da caixa mais o canal `email` em `tenant_channels`.
- **Prova mínima:** autenticação SMTP e IMAP, envio autorizado, leitura da mensagem e ausência de
  duplicata. SMTP saudável não prova IMAP saudável.
- **Desativação Aidan:** a mesma ação **Desativar este canal** chama
  `POST /admin/aidan/channels/{channel_id}/deactivate`. O status `inactive` retira a caixa das resoluções
  de envio e leitura sem apagar servidor, remetente ou referência protegida; o health passa a
  `unavailable` e a mutação fica auditada.
- **Credencial comprometida:** revogue também a senha de aplicativo na conta de e-mail. A desativação
  do canal não substitui essa revogação externa.
- **Prova negativa:** após desativar, envio e leitura precisam falhar fechados antes de SMTP/IMAP. Não
  substitua a operação por exclusão física.

## Shopify [provider-shopify]

- **Owners de conexão:** `shopify_storefront` para catálogo/carrinho e `shopify_admin` para
  estoque/pedidos/clientes. Cada operação verifica a capability no registry imediatamente antes de
  construir o client externo; tela e agente usam o mesmo `ShopifyCommerceService`.
- **Prova mínima:** probe separado das duas conexões, leitura real de catálogo e obtenção de
  `checkout_url`, sem concluir pagamento.
- **Revogação Aidan:** use a rota de revogação para a conexão exata. Depois, rode o **teste negativo do agente**:
  busca ou checkout deve falhar antes de qualquer I/O Shopify. Revogue também a instalação ou
  token no painel Shopify se houver comprometimento.
- **Rollback:** desative primeiro a capability mutante `shopify.cart.create`; reverta a release/YAML
  canônico somente se o delta do agente estiver envolvido. Não crie client ou tool paralela.

## Site e conhecimento [provider-site]

- **Owner atual:** fonte publicada, job de ingestão e release/YAML canônico do tenant. O site não usa
  token de provider no navegador.
- **Prova mínima:** URL acessível, ingestão encerrada, documento presente no store físico e pergunta
  respondida com fonte e data.
- **Rollback:** pare a ingestão, desative a fonte problemática e reative a última release publicada
  comprovadamente boa. Não devolva conteúdo antigo como se fosse atual.

## Totem web [provider-web-kiosk]

- **Owner atual:** sessão web e grants temporários; não há gerenciamento do dispositivo nem credencial
  de provider externa.
- **Prova mínima:** ativação única, visita isolada, handoff e encerramento sem contexto residual.
- **Rollback:** pare novas ativações e reative a última release/YAML canônico estável. Grants expiram e
  não devem ser copiados, persistidos em YAML ou reutilizados em outro aparelho.

## Critério de encerramento

O rollout só fecha quando o health factual acompanha a revogação/desativação, a auditoria mantém ator
e `correlation_id`, e a prova negativa mostra que tela, worker e agente não usam a conexão ou canal
interrompido. Limitação externa permanece bloqueio nomeado; nunca vira sucesso documental.
