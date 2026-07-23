# Manual conceitual, executivo, comercial e estrategico: Gestao SaaS x Tenant e telas de administracao

## 1. O que e esta capacidade

O modulo SaaS transforma um YAML de tenant, ja usado para configurar agentes, RAG, ingestao e ETL,
em um **produto publicavel e assinavel**. Ele nao substitui o modelo de tenant/YAML existente; ele
adiciona uma camada comercial em cima dele.

O codigo lido mostra a cadeia completa. Um `tenant` (organizacao) possui `saas_projects` (produtos).
Cada projeto publica `saas_project_releases`, que sao fotografias imutaveis de um `tenant_yaml`. Um
ponteiro (`saas_project_active_releases`) diz qual release esta em uso agora. Sobre o projeto existem
`saas_plans` (ofertas comerciais). Uma conta (`user_accounts`) assina um plano e vira
`saas_subscriptions`. A assinatura concede `saas_entitlements`, que sao direitos concretos por
operacao (`agent`, `rag`, `ingest` ou `etl`) — nunca um agente especifico. `saas_billing_events` e o
livro-razao (hoje simulado) desses eventos comerciais.

Em paralelo a esse nucleo comercial existe a **governanca de tenant e usuario**: quem pertence a qual
organizacao (`tenant_users`/membership), quais credenciais tecnicas existem, quais segredos o tenant
guarda e quais usuarios finais um canal (WhatsApp/Instagram) reconhece. Essa governanca nao concede
assinatura por si so — ela e o cadastro organizacional que a camada comercial usa para saber quem pode
administrar o que.

Por fim, essa capacidade inclui as **telas de administracao** que operam esse modelo: a tela de
produtos SaaS (com sete abas — Visao geral, Versoes, Capacidades, Planos, Assinantes, Uso e
Auditoria), a tela do assinante, a pagina publica de checkout e a familia de telas de governanca
(tenants, credenciais, memberships, grants, segredos e usuarios finais de canal).

## 2. Que problema ela resolve

Sem essa camada, uma plataforma de agentes configurada por YAML tem um problema comercial concreto:
o YAML e um artefato tecnico vivo, editavel a qualquer momento, e nao existe forma segura de vender
"uma versao dele" sem correr o risco de um cliente pagante ser afetado por uma mudanca feita para
outro cliente, ou de o YAML/API key vazarem para quem apenas assina o produto.

O codigo lido mostra quatro problemas resolvidos ao mesmo tempo.

1. **Estabilidade de contrato para quem paga.** Uma release publicada e imutavel; o assinante nunca
   sofre uma mudanca de comportamento sem uma nova versao publicada explicitamente.
2. **Venda por operacao, nao por agente.** O entitlement e por `agent/rag/ingest/etl`; o cliente
   escolhe projeto e operacao, a release fixa o YAML e o YAML fixa o entrypoint. Isso evita expor um
   seletor de agente ao publico.
3. **Isolamento de identidade.** Quem administra o produto usa sessao federada com membership
   validado por organizacao (`X-SaaS-Tenant-Id`); quem assina usa apenas a propria conta. Nenhum dos
   dois manipula YAML ou API key diretamente.
4. **Rastreabilidade administrativa.** Cada acao do boundary devolve um `correlation_id` visivel na
   tela, e o backend registra eventos de estado (criacao, publicacao, ativacao, mudanca de plano)
   como trilha auditavel.

## 3. Visao executiva

Para lideranca, esta capacidade importa por tres razoes praticas.

1. **Ela separa "configurar um agente" de "vender um produto".** O time tecnico continua editando
   YAML livremente; o que vira oferta comercial e uma fotografia publicada e imutavel dele. Isso
   reduz o risco de uma mudanca tecnica derrubar silenciosamente um cliente pagante.
2. **Ela cria trilha de auditoria e uso por produto.** A administracao consegue ver, por projeto, quem
   assina, o que consumiu (`saas_billing_events`, agregados de uso) e o historico de eventos de
   release/plano — sem precisar ler log bruto.
3. **Ela reaproveita a governanca de tenant/usuario ja existente.** Nao ha um segundo sistema de
   identidade para o comercial; quem administra o produto e quem administra a organizacao usam a
   mesma sessao federada e o mesmo cadastro de membership.

## 4. Visao comercial

Comercialmente, este modulo responde a uma pergunta concreta: "como transformamos uma configuracao
de agente/RAG em algo que um cliente pode assinar, sem expor a configuracao nem o motor por tras
dela?"

O codigo confirma um caminho de venda completo e simulado ponta a ponta: publicar um produto,
oferecer um plano, o cliente assinar, o sistema conceder apenas os direitos do plano e a
administracao acompanhar consumo e auditoria. Hoje o provedor de cobranca e `simulated` (sem gateway
real, sem cartao, valor zero por padrao) — o valor comercial vendavel e o **desenho do funil**, nao
ainda a cobranca real. A partir desta rodada, planos tambem podem ter preco/moeda reais editados pela
tela administrativa (a mutacao existe no banco), o que aproxima o modulo de um checkout comercial de
verdade quando um provedor de pagamento for plugado.

## 5. Visao estrategica

Estrategicamente, este modulo fortalece a plataforma em quatro frentes.

1. **Desacopla runtime de comercializacao.** O motor agentic (DeepAgent/Workflow/RAG/ETL) nao sabe
   que existe assinatura; o boundary SaaS decide entitlement antes de delegar ao runtime.
2. **Prepara multi-produto por tenant.** Um mesmo tenant pode publicar varios projetos SaaS
   independentes, cada um com seu proprio ciclo de release e plano.
3. **Da lastro para expansao de canais.** Um projeto pode ser consumido por chat direto (Agent/RAG)
   ou por canal externo (WhatsApp/Instagram) sem duplicar o modelo comercial.
4. **Cria um unico boundary de autorizacao para evoluir.** Toda nova capacidade comercial (edicao de
   plano, auditoria, uso) entra pelo mesmo `saas_router`, com o mesmo padrao de permissao e
   correlacao — reduz o custo de adicionar a proxima funcionalidade.

## 6. Conceitos necessarios para entender

### tenant

E a organizacao dona do produto. Um tenant pode ter varios `saas_projects`. O `owner_user_account_id`
e apenas o dono administrativo do cadastro; ele nao concede, por si so, direito de uso do produto.

### saas_project (projeto SaaS)

E o produto publicavel. Tem uma `project_key` publica, um status (`active/inactive/archived`) e,
quando publicado, uma release ativa. Nao existe coluna de agente ou workflow no projeto — a
capacidade real vem do YAML por tras da release.

### saas_project_release (release)

E a fotografia imutavel de exatamente um `tenant_yaml`, com hash SHA-256 do conteudo e um manifesto
derivado (`manifest_json`) contendo `operations`, `entrypoint` e hashes. Uma release publicada ou
retirada nunca mais e alterada — trocar a oferta exige publicar uma release nova.

### ponteiro de release ativa

E a linha que diz, por ambiente e projeto, qual release esta em uso agora. Trocar o ponteiro (ativar
outra release, ou fazer rollback para uma release anterior ja publicada) e uma troca atomica que nao
reescreve assinatura nem entitlement.

### saas_plan (plano)

E a oferta comercial de um projeto: nome, status e o subconjunto de operacoes vendidas
(`agent/rag/ingest/etl`), que precisa ser um subconjunto das operacoes publicadas pela release ativa.
Preco, moeda e configuracao de provedor ficam em `provider_config_json`, um campo neutro que nao
amarra o modelo a um gateway especifico.

### saas_subscription (assinatura)

E o vinculo entre uma conta pessoal (`user_accounts`) e um plano. Tem status
(`pending/trialing/active/past_due/paused/cancelled/expired`) e uma versao para controle de
concorrencia otimista — a mesma conta nao pode ter duas assinaturas vivas do mesmo plano ao mesmo
tempo.

### saas_entitlement (entitlement)

E o direito concreto concedido pela assinatura: uma operacao (`agent/rag/ingest/etl`) do projeto, com
validade e limites. Nao existe `agent_id` aqui — o cliente nunca escolhe um agente, apenas uma
operacao; a release e o YAML decidem qual entrypoint responde.

### environment como segregador

Toda tabela `saas_*` (exceto o cadastro basico de tenant) tem uma coluna `environment`, e o boundary
deriva esse valor do `ENVIRONMENT` canonico do processo — nunca de um literal fixo. O dominio
comercial (assinatura, billing, entitlement) hoje so opera quando `ENVIRONMENT=prod`; nao existe ainda
comercio simulado em dev/homolog.

### correlation_id na tela

Cada resposta do boundary SaaS devolve um `correlation_id` (corpo e header); as telas administrativas
exibem esse valor apos cada acao, permitindo abrir o log oficial daquela execucao especifica.

## 7. Como a capacidade funciona por dentro

O fluxo confirmado no codigo tem duas metades: administrar o produto e consumi-lo.

Do lado administrativo: uma sessao web federada autentica o operador; a tela pede o membership
autorizado da organizacao (`X-SaaS-Tenant-Id`); o boundary valida esse escopo antes de qualquer
leitura ou escrita. Publicar um produto segue uma sequencia encadeada: criar projeto, escolher um
`tenant_yaml` elegivel, criar release em rascunho, publicar (congela hash e manifesto), ativar
(troca o ponteiro), criar ou editar plano com as operacoes realmente oferecidas.

Do lado do assinante: a pagina publica do projeto mostra apenas capacidades e planos, nunca YAML ou
API key. Ao assinar, o backend resolve o projeto pela chave publica, cria a assinatura e concede os
entitlements do plano. Quando essa conta usa o chat ou o RAG do produto, o boundary de execucao
(Agent/AG-UI ou RAG) resolve a release ativa e verifica a operacao antes de delegar ao runtime — o
runtime em si nunca sabe que existe assinatura ou plano.

## 8. Divisao em etapas ou submodulos

### 8.1. Catalogo e ciclo de vida do produto

Responsavel por projeto, release e ponteiro ativo. Recebe pedidos administrativos de
criar/publicar/ativar; entrega um snapshot resolvivel (YAML + hash + manifesto + operacoes). O que
pode dar errado: tentar publicar/ativar uma release que nao pertence ao mesmo projeto/ambiente (a FK
composta bloqueia), ou tentar sobrescrever uma release ja publicada (o guard de imutabilidade
rejeita). Diagnostico: conferir status da release e o ponteiro ativo do projeto.

### 8.2. Comercial (plano, assinatura, billing, entitlement)

Responsavel pela oferta e pelo relacionamento comercial. Recebe plano + conta; entrega assinatura e
entitlements. O que pode dar errado: plano oferecendo uma operacao que a release ativa nao publica
(rejeitado por contrato); duas assinaturas vivas do mesmo plano para a mesma conta (bloqueado por
indice). Diagnostico: conferir operacoes da release ativa versus operacoes do plano, e o status atual
da assinatura.

### 8.3. Boundary HTTP unico

Um unico router concentra toda a superficie SaaS (administrativa e de assinante), aplicando
autorizacao, resolucao de escopo de tenant e `correlation_id` de forma uniforme. O que pode dar
errado: escopo de tenant nao autorizado (403), recurso nao encontrado no escopo (404), conflito de
estado concorrente (409). Diagnostico: o corpo do erro sempre traz `code`, `message` e
`correlation_id`.

### 8.4. Tela de administracao de produtos SaaS

Cobre o ciclo completo do produto em sete abas, do cadastro a auditoria. E o unico ponto de entrada
administrativo do modulo comercial. O que pode dar errado: operador sem organizacao com projetos
selecionada por padrao (fricção de UX, nao erro); acao disparada com projeto trocado no meio de um
carregamento assincrono (corrigido nesta rodada com trava de corrida). Diagnostico: conferir a
correlacao exibida apos cada acao e, se necessario, o log dessa correlacao.

### 8.5. Tela e pagina do assinante

Cobrem a experiencia de quem paga: ver as proprias assinaturas, cancelar, e a pagina publica de
checkout. Sao propositalmente minimalistas — nunca expoem configuracao interna.

### 8.6. Governanca de tenant e usuario

Cobre o cadastro organizacional que sustenta o modulo comercial: quem e cada tenant, quem participa
dele (membership), quais credenciais tecnicas e segredos existem, e quais usuarios finais um canal
reconhece. Essas telas nao vendem nada — elas garantem que o boundary SaaS tenha, do lado de fora, um
tenant e um membership validos para autorizar.

## 9. Pipeline principal (publicar, ativar, vender, consumir, auditar)

1. **Publicar**: escolher um `tenant_yaml` valido, compilar hash e manifesto, criar release em
   rascunho, publicar (congela).
2. **Ativar**: trocar o ponteiro do projeto para a release publicada (compare-and-set); toda nova
   resolucao passa a usar essa release.
3. **Ofertar**: criar ou editar um plano com o subconjunto de operacoes que sera vendido; publicar
   preco/moeda reais quando aplicavel.
4. **Assinar**: o cliente escolhe um plano na pagina publica; o checkout cria a assinatura e concede
   os entitlements daquele plano.
5. **Consumir**: a cada uso (chat, RAG, canal), o boundary de execucao verifica assinatura ativa e
   entitlement antes de delegar ao runtime.
6. **Auditar e medir**: a administracao consulta o historico de eventos (auditoria) e os agregados de
   uso (sucesso/falha/duracao) por operacao e por release, agora direto na tela.

Rollback e uma variacao do passo 2: apontar o ponteiro para uma release anterior ainda publicada, sem
tocar assinatura ou entitlement.

## 10. Decisoes tecnicas e trade-offs

### Release imutavel em vez de YAML mutavel direto

Ganho: o assinante nunca sofre mudanca de contrato sem uma nova publicacao explicita; rollback e
trivial (trocar ponteiro).

Custo: toda mudanca de configuracao de um produto ja vendido exige o ciclo completo
draft→publish→activate, mesmo para ajustes pequenos.

### Entitlement por operacao, nunca por agente

Ganho: o modelo comercial fica simples e estavel mesmo quando a topologia interna de subagentes muda;
o assinante nunca escolhe motor.

Custo: nao existe hoje venda granular por subagente ou por skill especifica dentro de uma operacao.

### Provider comercial simulado e restrito a `ENVIRONMENT=prod`

Ganho: evita ambiguidade de cobranca real antes de existir gateway de pagamento; evita testar fluxo
comercial contra dados de producao a partir de outro ambiente.

Custo: nao ha hoje um caminho de teste de checkout completo em dev/homolog — a validacao end-to-end
comercial so acontece em prod.

### Abas na mesma tela em vez de tela nova para Uso e Auditoria

Ganho: reuso maximo do mecanismo de abas ja existente, menor risco de regressao na tela mais
madura do modulo, e nenhuma mudanca no catalogo de navegacao.

Custo: a tela de produto cresce em numero de abas (agora sete); a navegacao exige que o operador saiba
que esses dados existem por projeto, nao em um painel global.

### Mutacao de plano via `UPDATE` em campo `jsonb`, sem alterar o schema

Ganho: editar preco/moeda/operacoes e trocar status de plano nao exigiu nenhuma migracao de banco —
a estrutura ja suportava a mudanca, faltava apenas o endpoint.

Custo: preco e moeda vivem dentro de um campo de configuracao neutro (`provider_config_json`), nao em
colunas dedicadas — qualquer relatorio financeiro futuro precisa ler dentro desse JSON.

## 11. Configuracoes que mudam o comportamento

### `ENVIRONMENT`

Segrega toda leitura/escrita `saas_*`. Controla, entre outras coisas, se o dominio comercial aceita
operar (`_require_prod` hoje trava esse dominio a `prod`).

### Status do projeto, da release e do plano

`active/inactive/archived` (projeto e plano) e `draft/published/retired` (release) controlam o que
fica visivel/oferecivel. Um plano `inactive` deixa de aparecer na oferta publica sem apagar historico;
uma release `retired` nunca mais pode ser reativada como se fosse rascunho.

### Operacoes do plano versus operacoes da release ativa

O plano so pode oferecer operacoes que a release ativa publica. Se a release ativa mudar e deixar de
publicar uma operacao que um plano ja oferecia, o sistema nao corrige o plano automaticamente — isso
exige acao administrativa.

## 12. Contratos, entradas e saidas

O boundary administrativo exige sessao federada valida e, para as rotas de projeto/plano/assinante,
um `X-SaaS-Tenant-Id` que corresponda a um membership autorizado do operador. O boundary do assinante
exige apenas a propria sessao — nunca um tenant escolhido manualmente. Acoes de mutacao de assinatura
(cancelar, confirmar) exigem tambem um cabecalho `Idempotency-Key`, para que reenviar a mesma acao por
falha de rede nao duplique o efeito. O payload publico do checkout nunca recebe YAML, API key ou
seletor de agente — apenas identificador de plano.

## 13. O que acontece em caso de sucesso

No caminho feliz, o operador publica e ativa uma release, cria um plano com as operacoes corretas, o
cliente assina pelo checkout publico, recebe entitlements imediatos e passa a usar o chat/RAG do
produto. A administracao ve, na mesma tela, o preco e status reais do plano, a lista paginada de
assinantes, os agregados de uso por operacao/release e a trilha de auditoria — tudo com a correlacao
da acao que gerou cada carregamento.

## 14. O que acontece em caso de erro

Os erros mais relevantes confirmados no boundary sao: tentar administrar um tenant fora do
membership autorizado (acesso negado); operar um recurso que nao pertence ao escopo tenant/ambiente
corrente (nao encontrado); publicar/ativar/mudar de estado uma release ou assinatura em conflito com
o estado atual (conflito, ex.: ponteiro ja trocado por outra acao concorrente); enviar um plano com
operacao fora do que a release ativa publica (contrato invalido). Um defeito historico relevante —
`GET /admin/memberships` retornando erro 500 por falha ao serializar um identificador de membership —
ja foi corrigido antes desta rodada; ele afetava as telas de governanca, nao o boundary SaaS.

## 15. Observabilidade e diagnostico

A ordem pratica de diagnostico, do ponto de vista de quem opera a tela, e: (1) copiar a correlacao
exibida apos a acao que falhou; (2) abrir o log oficial dessa correlacao; (3) verificar em que camada
o fluxo parou — autorizacao de tenant, resolucao de projeto/release, validacao de operacao do plano,
ou execucao no runtime downstream (Agent/RAG/canal); (4) para produtos publicados, confirmar se a
release ativa e o plano realmente concordam nas operacoes oferecidas.

## 16. Impacto tecnico

O modulo reduz acoplamento entre o motor agentic e a comercializacao (o runtime nunca sabe que existe
assinatura), concentra toda a autorizacao e correlacao comercial em um unico boundary, e fecha lacunas
de observabilidade que existiam ate esta rodada (uso e auditoria agora sao consumiveis pela propria
tela, nao apenas pelo backend).

## 17. Impacto executivo

Ele reduz o risco de um cliente pagante ser afetado por uma mudanca tecnica nao intencional, cria
trilha auditavel de quem administra o que por organizacao, e aumenta a previsibilidade operacional ao
padronizar como qualquer novo produto e publicado, vendido e medido.

## 18. Impacto comercial

Ele viabiliza vender "um produto" (um caso de uso especifico, com preco e limite proprios) em vez de
vender "acesso a plataforma". Isso favorece clientes que querem pacotes fechados e previsiveis (ex.:
"chat de duvidas tecnicas sobre normas DNIT" como produto isolado) e cria a base para precificacao real
quando um gateway de pagamento for conectado.

## 19. Impacto estrategico

Ele posiciona a plataforma para crescer como um catalogo de produtos por tenant, nao como uma unica
configuracao monolitica. A separacao entre catalogo (release imutavel), comercial (plano/assinatura) e
governanca (tenant/membership) permite evoluir cada frente de forma independente sem reescrever as
outras.

## 20. Exemplos praticos guiados

### Exemplo 1. Publicar um produto de conhecimento (perfil DNIT)

Cenario: um tenant de engenharia quer vender acesso a um chat de duvidas tecnicas sobre normas
ingeridas via RAG, sem expor agente algum.

Processamento: o operador escolhe o `tenant_yaml` que so publica `rag`/`ingest`, publica e ativa a
release, cria um plano oferecendo apenas `rag`.

Saida pratica: o assinante ve, na pagina publica, um produto de "chat RAG"; ao assinar, ganha
entitlement de `rag` e usa Q&A pelo `projectKey`, sem jamais ver YAML ou API key.

### Exemplo 2. Editar preco de um plano ja publicado

Cenario: um plano criado com preco zero (padrao de demonstracao) precisa refletir um preco real.

Processamento: a administracao abre a aba Planos, edita o plano existente informando o novo
preco/moeda; o backend valida que as operacoes continuam dentro do que a release ativa publica e
grava a mudanca com log de estado.

Saida pratica: a tela passa a mostrar o preco real formatado, sem exigir criar um plano novo nem
publicar uma nova release.

### Exemplo 3. Investigar por que um cliente nao acessa o produto

Cenario: um assinante relata que nao consegue usar o chat do produto.

Processamento: a administracao abre a aba Assinantes, localiza a assinatura, confere status e
entitlements; se necessario, abre a aba Auditoria filtrando por aquela correlacao ou operacao.

Saida pratica: a causa aparece no proprio historico — assinatura cancelada, entitlement expirado, ou
operacao que o plano nunca ofereceu.

## 21. Explicacao 101

Pense em um restaurante com cardapio fechado. O YAML e a cozinha inteira — tudo que ela sabe fazer. O
projeto SaaS e o restaurante publicado com um endereco (`project_key`). A release e o cardapio impresso
de hoje: uma vez entregue ao cliente, ninguem troca um prato no meio do jantar dele — se a cozinha
mudar a receita, o restaurante imprime um cardapio novo para os proximos clientes. O plano e o combo
que o cliente compra (algumas entradas do cardapio, nao a cozinha inteira). A assinatura e o ticket do
cliente; o entitlement e o carimbo que diz exatamente o que aquele ticket libera. As telas de
governanca sao o cadastro da empresa dona do restaurante: quem trabalha ali, quais credenciais e
segredos existem — sem isso nao ha como saber quem tem autoridade para mexer no cardapio.

## 22. Limites e pegadinhas

1. O provedor de cobranca ainda e simulado; nao ha gateway de pagamento real nem cartao cobrado.
2. O dominio comercial (assinatura/billing/entitlement) so opera quando `ENVIRONMENT=prod` — nao ha
   ambiente de teste comercial isolado hoje.
3. Editar plano nao republica a release; se a mudanca desejada e de YAML/comportamento, o caminho
   correto continua sendo publicar uma nova release, nao editar plano.
4. A auto-selecao de organizacao na tela administrativa lembra a ultima organizacao usada, mas um
   operador com membership em varias organizacoes ainda precisa confirmar qual esta selecionada antes
   de agir.
5. Nao existe hoje venda por subagente, skill ou entrypoint especifico — apenas por operacao macro.

## 23. Troubleshooting

### Sintoma: produto mostra uma capacidade que o cliente nao deveria ter

Causa provavel: o plano oferece uma operacao mais ampla do que deveria, ou a release ativa foi trocada
sem revisar os planos vigentes. Compare operacoes da release ativa com operacoes do plano.

### Sintoma: assinatura nao libera acesso mesmo com status ativo

Causa provavel: entitlement expirado, revogado, ou a operacao pedida pelo cliente nao consta no plano
assinado. Confira a aba Assinantes e o detalhe de entitlement por operacao.

### Sintoma: tela de governanca (memberships/grants) aparece vazia ou com erro

Causa provavel historica: falha de serializacao de identificador no diretorio de memberships — ja
corrigida antes desta rodada. Se reaparecer, tratar como regressao e investigar pelo log da
correlacao exibida.

## 24. Checklist de entendimento

- Entendi que release e imutavel e que trocar oferta exige publicar uma release nova.
- Entendi que entitlement e por operacao (`agent/rag/ingest/etl`), nunca por agente.
- Entendi que administrar produto exige sessao federada com membership autorizado na organizacao.
- Entendi que assinar um produto usa apenas a conta do assinante, sem selecionar tenant manualmente.
- Entendi que o dominio comercial hoje so opera em `ENVIRONMENT=prod` e com provedor simulado.
- Entendi que as telas de governanca sustentam o cadastro que o boundary SaaS usa para autorizar.
- Entendi que cada acao devolve uma correlacao que permite abrir o log oficial daquela execucao.

## 25. FAQ conceitual

**1. O que e um "projeto SaaS" na pratica?**
E o produto publicavel dentro de um tenant: um nome, um endereco publico (`project_key`) e, quando
publicado, uma release ativa que define o que o assinante pode usar.

**2. Qual a diferenca entre projeto, release e plano?**
Projeto e o produto em si; release e uma fotografia imutavel do YAML por tras dele; plano e a oferta
comercial (quais operacoes daquela release sao vendidas e por quanto).

**3. O assinante escolhe qual agente vai usar?**
Nao. O assinante escolhe projeto e, indiretamente pelo plano, operacao (`agent/rag/ingest/etl`). A
release e o YAML decidem qual entrypoint responde.

**4. Por que a release nao pode ser editada depois de publicada?**
Para garantir que quem ja assinou nao seja afetado por uma mudanca de configuracao feita para outro
fim. Qualquer mudanca de comportamento exige publicar uma release nova.

**5. Um tenant pode ter mais de um produto SaaS?**
Sim. Tenant tem relacao um-para-muitos com projeto; cada projeto tem seu proprio ciclo de
release/plano/assinatura.

**6. O que acontece se eu ativar a release errada?**
E possivel reverter apontando o ponteiro de volta para uma release anterior ainda publicada
(rollback), sem regravar assinatura ou entitlement.

**7. Por que existe um plano "gratuito simulado" com preco zero?**
E o padrao usado enquanto nao ha gateway de pagamento real; hoje a mutacao de preco ja existe, entao
um plano pode ter preco real definido pela administracao mesmo sem cobranca de fato acontecer.

**8. Quem pode administrar um produto SaaS?**
Um usuario com sessao federada e membership autorizado (com a permissao correta) na organizacao
selecionada — nunca uma credencial tecnica generica.

**9. O assinante precisa de API key para usar o produto que assinou?**
Nao no caminho autenticado por sessao; o backend valida sessao, assinatura, release e entitlement e
dispensa API key nesse caso. Integracoes tecnicas sem essa identidade comercial continuam exigindo
API key.

**10. O que e um entitlement, em uma frase?**
E o carimbo concreto que diz "esta assinatura pode usar esta operacao deste projeto agora".

**11. Por que existem telas de governanca separadas da tela SaaS?**
Porque membership, credencial e segredo sao conceitos organizacionais mais amplos que o comercial —
eles sustentam varias areas da plataforma, nao so o SaaS.

**12. O que muda quando o `ENVIRONMENT` e diferente de `prod`?**
Leitura de projeto/release/plano continua funcionando no ambiente corrente, mas o dominio comercial
(assinatura/billing/entitlement) hoje recusa operar fora de `prod`.

**13. Por que a tela mostra "Uso" e "Auditoria" separadas?**
Uso e um resumo agregado (quantidade, sucesso/falha, duracao media por operacao/release); Auditoria e
o historico de eventos individuais, paginado e filtravel.

**14. Cancelar uma assinatura apaga o historico dela?**
Nao. O cancelamento e uma transicao de estado; a assinatura e os eventos relacionados permanecem como
trilha auditavel.

**15. Editar um plano existente pode oferecer uma operacao que a release ativa nao publica?**
Nao. O contrato valida que as operacoes do plano sao sempre um subconjunto das operacoes publicadas
pela release ativa do projeto.

**16. Onde entra a documentacao tecnica detalhada deste modulo?**
Em `docs/tecnico/README-TECNICO-GESTAO-SAAS-TENANT.md`, com arquitetura, permissoes, exemplo de API e
o papel tecnico de cada tela.

## 26. Evidencias no codigo

- `src/saas_project/` (`repository.py`, `service.py`, `resolver.py`, `http_repository.py`) — ciclo de
  vida de projeto/release/ponteiro e read-models do boundary.
- `src/saas_commercial/` (`service.py`, `repository.py`) — plano, assinatura, billing, entitlement, com
  `_require_prod` restringindo o dominio comercial a `ENVIRONMENT=prod`.
- `src/api/routers/saas_router.py` — boundary HTTP unico; identidade administrativa por sessao
  federada + `X-SaaS-Tenant-Id`; identidade de assinante por `user_account_id` da sessao.
- `src/api/services/saas_http_service.py` — orquestra os services autorais para o boundary; registra
  eventos de estado.
- `app/ui/static/ui-admin-saas-projects.html` + `app/ui/static/js/saas-products-admin.js` — tela
  administrativa com sete abas (`data-saas-tab`).
- `docs/.interno/.planos/saas-tenant-auditoria/investigacao--2026-07-15--06-42-05--saas-tenant-auditoria.md`
  e `docs/.interno/.planos/saas-tenant-auditoria/plano--2026-07-15--08-23-25--saas-tenant-auditoria-p3.md`
  — investigacao e execucao que fecharam as lacunas de uso/auditoria/paginacao/gestao de planos.
