# Manual conceitual, executivo, comercial e estrategico: BYOK e isolamento de custos por tenant

## 1. O que e esta capacidade

Esta capacidade permite que cada tenant traga e governe as proprias credenciais de integracao e, quando desejado, pague o consumo diretamente no provedor externo que ele mesmo controla. Em termos simples, a plataforma nao precisa operar todos os tenants com a mesma chave compartilhada de LLM, API externa ou conexao SQL.

O codigo lido mostra tres blocos trabalhando juntos.

1. A credencial autenticada resolve qual tenant esta operando.
2. O diretorio multi-tenant injeta client_code, tenant_id, security_keys e tenant_secrets somente daquele tenant no YAML de runtime.
3. Os catalogos de integracao e os eventos de billing ficam marcados por tenant, credencial, correlation_id, provider e modelo.

Por isso, BYOK aqui nao e so "guardar uma API key". E um desenho de plataforma para separar identidade, segredo, configuracao runtime e consumo operacional por tenant.

## 2. Que problema ela resolve

Sem BYOK e sem isolamento de custos, uma plataforma multi-tenant tende a cair em um desenho fraco.

1. Todos os clientes compartilham o mesmo provedor e a mesma conta pagadora.
2. Segredos de um cliente acabam virando configuracao central do produto.
3. Fica dificil provar quem consumiu o que.
4. O custo do tenant A pode contaminar o tenant B.
5. O produto fica menos vendavel para enterprise, porque governanca e cobranca viram um problema manual.

O codigo lido mostra que a plataforma tenta resolver isso separando o problema em camadas.

1. A camada de autenticacao identifica o tenant correto.
2. A camada de segredos carrega apenas o namespace daquele tenant.
3. A camada de integracoes registra conexoes e perfis com tenant_id explicito.
4. A camada de billing emite eventos estruturados com identificadores suficientes para rastrear consumo.

## 3. Visao executiva

Para lideranca, esta feature importa porque reduz dois riscos de negocio ao mesmo tempo.

1. Risco financeiro: o consumo de IA, API e banco pode ser segregado por cliente em vez de ser absorvido cegamente pela plataforma.
2. Risco de governanca: cada tenant pode operar com seus segredos, politicas e catalogos sem expor os ativos de outro tenant.

Na pratica, isso ajuda a vender a plataforma para clientes mais exigentes, porque a conversa deixa de ser "confie que vamos separar internamente" e passa a ser "o runtime e os segredos ja nascem separados por tenant".

## 4. Visao comercial

Comercialmente, BYOK e isolamento de custos respondem dores muito concretas.

1. Cliente que quer usar a propria conta OpenAI, Azure OpenAI ou outra API.
2. Cliente que precisa manter custeio interno ou chargeback por unidade de negocio.
3. Cliente que nao aceita compartilhar segredo ou faturamento com outros tenants.
4. Cliente que precisa de autonomia para rotacionar credenciais sem abrir ticket de desenvolvimento.

O valor vendavel suportado pelo codigo e este: a plataforma oferece um boundary de autoatendimento e um runtime multi-tenant que injeta contexto e segredos corretos do tenant antes da execucao.

## 5. Visao estrategica

Estrategicamente, essa capacidade fortalece a plataforma em quatro frentes.

1. Reduz acoplamento entre produto e credenciais do cliente.
2. Permite crescer para tenants com politicas diferentes de seguranca e compliance.
3. Melhora a prontidao para enterprise e white-label.
4. Cria base para governanca financeira mais sofisticada no futuro.

O ponto importante e que o isolamento nao depende apenas de interface administrativa. Ele esta no contrato do runtime, no enriquecimento do YAML e nos repositorios persistidos com tenant_id.

## 6. Conceitos necessarios para entender

### BYOK

BYOK significa Bring Your Own Key. Em pratica, o tenant fornece a propria credencial de provedor externo. No codigo lido, isso aparece principalmente em dois formatos.

1. secret_ref apontando para um segredo do proprio tenant.
2. inline_encrypted ou credentials_json_encrypted, quando a credencial fica armazenada de forma criptografada no catalogo governado.

### tenant_id

E o identificador logico do tenant. Ele e obrigatorio para que o runtime saiba qual namespace de segredo, qual catalogo de integracoes e qual contexto de autorizacao deve ser usado.

### client_code

E o identificador funcional do cliente. Ele aparece na credencial autenticada, no portal do cliente e no client_context do YAML. O codigo mostra que client_code e tenant_id caminham juntos, mas tenant_id e o pivor de isolamento no persistido.

### tenant_secrets

E a tabela de segredos genericos do tenant. Ela guarda pares como secret_key e secret_value e e usada para resolver placeholders dentro do payload de security_keys.

### security_keys

E o payload de chaves de runtime carregado para o YAML. Ele funciona como mapa operacional de credenciais por tenant e pode referenciar tenant_secrets com o token $security_credential.

### secret_ref

Nao e o segredo em si. E a referencia logica para um segredo governado. Isso permite mudar o valor real do segredo sem reescrever toda a integracao.

### inline_encrypted

E o modo em que a credencial ou connection string fica guardada criptografada diretamente no registro governado da integracao. O codigo lido mostra que isso existe como alternativa, nao como substituto total de secret_ref.

### isolamento de custos

No contexto desta plataforma, isolamento de custos significa duas coisas diferentes.

1. Separacao do custo no provedor externo, quando cada tenant usa sua propria chave, conta ou conexao.
2. Separacao do rastreio interno, quando o runtime emite eventos de consumo com client_id, provider, model e correlation_id.

Essas duas coisas se reforcam, mas nao sao identicas.

## 7. Como a capacidade funciona por dentro

O fluxo confirmado no codigo lido e o seguinte.

1. Uma access_key autenticada entra no boundary HTTP.
2. A camada de auth recupera user_data e valida permissao.
3. Se tenant_id nao for resolvido, a requisicao falha cedo.
4. Com tenant_id e client_code em maos, o ClientDirectory enriquece o YAML com client_context.client.
5. O SecurityKeysRepository carrega tenant_security_keys e tenant_secrets apenas daquele tenant.
6. Placeholders $security_credential sao resolvidos somente contra o secret store do tenant corrente.
7. O YAML final segue para o runtime, agora com client_context, security_keys e segredos corretos.
8. Quando a execucao usa LLM ou outra integracao, o consumo pode acontecer com credenciais do proprio tenant.
9. Os coletores de billing e telemetria registram eventos estruturados com identificadores do consumo.

O ponto mais importante e este: a separacao nao acontece so na tela administrativa. Ela acontece durante a resolucao do runtime.

## 8. Divisao em etapas ou submodulos

### 8.1. Boundary de autenticacao e escopo do tenant

Esta etapa descobre quem esta chamando a plataforma e qual tenant deve ser usado. O codigo de autenticacao falha se tenant_id estiver ausente no diretorio. Isso e importante porque impede o sistema de continuar com contexto parcial e cair em comportamento ambiguo.

Valor pratico: o runtime nao pode carregar segredo ou integracao sem tenant definido.

### 8.2. Portal do cliente e autoatendimento de credenciais

O portal do cliente permite listar, criar, atualizar e remover credenciais e tenant_secrets. O helper `_resolve_target_client` bloqueia um cliente comum de operar outro client_code. Apenas credenciais superadmin conseguem atuar fora do proprio escopo.

Valor pratico: o tenant ganha autonomia sem perder segregacao.

### 8.3. Enriquecimento do YAML com contexto multi-tenant

Depois que client_code e tenant_id sao resolvidos, o ClientDirectory injeta client_context.client no YAML e entao carrega security_keys e tenant_secrets do tenant. Isso transforma um YAML generico em um runtime multi-tenant concreto.

Valor pratico: a mesma definicao funcional pode operar tenants diferentes sem misturar segredo ou identidade.

### 8.4. Catalogos governados de integracao

Os registros governados de auth profile e conexao SQL exigem tenant_id e aceitam duas estrategias de credencial: referencia segura ou valor criptografado inline. As leituras e updates sao filtrados por tenant_id.

Valor pratico: cada tenant pode publicar seus proprios endpoints, perfis e conexoes sem colisao logica com os demais.

### 8.5. Telemetria e billing

Os coletores de billing registram provider, modelo, tokens, custos estimados, session_id e client_id. Isso nao substitui um faturamento financeiro completo, mas cria uma trilha auditavel de consumo.

Valor pratico: operacao e analise conseguem separar o que foi consumido por credencial e sessao.

## 9. Pipeline principal de BYOK e separacao

### Passo 1. Resolver o tenant da requisicao

O sistema recebe uma credencial, extrai user_data e exige tenant_id valido. Se tenant_id nao existir, falha cedo. Isso evita que o runtime tente usar segredos ou integracoes sem dono definido.

### Passo 2. Determinar o client_code autorizado

No portal do cliente, _resolve_target_client permite operar o proprio client_code e bloqueia acesso a outro cliente quando a credencial nao e superadmin.

### Passo 3. Injetar client_context no YAML

O enriquecimento injeta client_context.client com client_code, tenant_id, yaml_path e metadados do tenant. Isso padroniza o caminho canonicamente esperado pelo restante do runtime.

### Passo 4. Carregar security_keys e tenant_secrets do tenant certo

O repositorio consulta tenant_security_keys e tenant_secrets usando tenant_id. Se security_keys nao existir, a operacao falha. Se um placeholder apontar para segredo inexistente, a operacao falha com erro explicito.

### Passo 5. Resolver placeholders de segredo

Quando o payload contem $security_credential, o loader consulta somente o secret store carregado para aquele tenant. Nao existe no codigo lido uma busca cruzada entre tenants.

### Passo 6. Executar integracoes com estrategia de credencial adequada

Os registros governados de auth profile e sql connection aceitam secret_ref ou inline_encrypted. Isso permite que o tenant escolha referencia a segredo proprio ou credencial criptografada diretamente no catalogo.

### Passo 7. Registrar consumo

Quando ha consumo de tokens, os coletores emitem eventos estruturados com client_id, provider, model, operation e correlation_id. Essa trilha permite auditoria e estimativa de custo.

## 10. Decisoes tecnicas e trade-offs

### Separar tenant_secrets de security_keys

Ganho: o payload operacional pode referenciar segredos sem copiar o valor real para todos os lugares.

Custo: a resolucao fica mais sofisticada e exige pipeline de enrich antes da execucao.

### Permitir secret_ref e inline_encrypted

Ganho: a plataforma atende tanto cenarios de referencia governada quanto necessidade de armazenar credencial criptografada no proprio registro.

Custo: o desenho fica hibrido. Nem todo endpoint administrativo seguro ja consegue testar secret_ref sem um resolvedor explicito de segredos do tenant.

### Falhar cedo quando tenant_id nao existe

Ganho: evita misturar runtime, segredo e custo sem dono definido.

Custo: o sistema fica menos permissivo para cadastros inconsistentes, mas esse custo e desejavel.

### Tabela de precos global

Ganho: simplifica estimativa de custo interno.

Custo: o codigo lido nao implementa uma tabela de precos especifica por tenant. Isso significa que o chargeback interno por regra de preco customizada por cliente nao esta confirmado no slice analisado.

## 11. Configuracoes que mudam o comportamento

### secret_ref versus inline_encrypted

Controla se a integracao usa referencia a segredo governado ou valor criptografado inline.

### credentials_json_encrypted e connection_string_encrypted

Controlam o modo inline criptografado para auth profiles e conexoes SQL.

### client_context.client.tenant_id

E o caminho canonicamente injetado para o tenant no YAML enriquecido.

### billing.enabled

Liga ou desliga o coletor de billing por token no runtime que o consulta.

### PRICING_CONFIG_PATH

Controla o arquivo opcional de tabela de precos. O codigo lido afirma explicitamente que essa tabela de precos e global e nao varia por cliente.

## 12. Contratos, entradas e saidas

O portal do cliente expoe dois contratos centrais para esta capacidade.

1. /client-portal/credentials para gerenciar access keys vinculadas ao cliente.
2. /client-portal/tenant-secrets para gerenciar segredos genericos do tenant.

Os catalogos governados de integracoes tambem recebem tenant_id explicito quando criam auth profiles e SQL connections.

## 13. O que acontece em caso de sucesso

No caminho feliz, o tenant autentica com uma credencial valida, o runtime resolve tenant_id, injeta client_context, carrega security_keys e tenant_secrets corretos, resolve placeholders e executa a integracao com credenciais daquele tenant. Se houver billing ativo, o consumo sai em evento estruturado com identificadores suficientes para analise.

## 14. O que acontece em caso de erro

Os erros mais importantes confirmados no codigo foram estes.

1. tenant_id ausente na credencial autenticada: a auth falha e aborta a operacao.
2. Cliente tentando operar outro client_code no portal sem ser superadmin: retorna 403.
3. security_keys ausentes para o tenant: a resolucao falha cedo.
4. Placeholder apontando para segredo inexistente em tenant_secrets: a operacao falha explicitamente.
5. Perfil de integracao autenticado sem secret_ref nem credencial inline: o model validator rejeita.
6. Conexao SQL em secret_ref sem resolvedor seguro no endpoint de teste administrativo: o endpoint recusa o teste nesta fase.

## 15. Observabilidade e diagnostico

Para diagnosticar problemas nessa capacidade, a ordem pratica e esta.

1. Confirmar se a credencial autenticada carrega tenant_id e client_code.
2. Confirmar se o portal esta operando o client_code autorizado.
3. Confirmar se tenant_security_keys existe para o tenant.
4. Confirmar se tenant_secrets contem as chaves referenciadas por $security_credential.
5. Confirmar se a integracao esta registrada com tenant_id correto.
6. Confirmar os eventos token_billing_event e seus campos provider, model, client_id e correlation_id.

## 16. Impacto tecnico

Esta capacidade reduz acoplamento entre runtime e credenciais compartilhadas, reforca o isolamento multi-tenant no YAML, melhora a rastreabilidade do consumo e cria uma base limpa para chargeback ou billing mais sofisticado no futuro.

## 17. Impacto executivo

Ela reduz risco de cobranca cruzada, melhora governanca de credenciais, simplifica onboarding enterprise e aumenta previsibilidade operacional quando tenants exigem autonomia sobre provedores e faturamento.

## 18. Impacto comercial

Ela permite vender a plataforma para clientes que exigem segredo proprio, custo separado, rotacao autonoma de chave e menor dependencia de operacao manual do fornecedor da plataforma.

## 19. Impacto estrategico

Ela prepara o produto para multi-tenant serio. Sem essa base, crescimento comercial, compliance e governanca financeira ficam presos a processos manuais e a contas compartilhadas.

## 20. Exemplos praticos guiados

### Exemplo 1. Tenant com propria chave de LLM

Cenario: o tenant cadastra um tenant_secret com a chave do provedor e o payload de security_keys referencia esse segredo com $security_credential.

Processamento: a access_key do tenant autentica, o YAML e enriquecido, o loader busca tenant_secrets daquele tenant e substitui o placeholder.

Saida pratica: a chamada externa usa a credencial do proprio tenant.

### Exemplo 2. Tenant com conexao SQL propria

Cenario: o tenant registra uma sql_connection com connection_mode igual a secret_ref ou inline_encrypted.

Processamento: o catalogo governado grava tenant_id com o registro e as leituras posteriores carregam a conexao daquele tenant.

Saida pratica: a tool dinamica opera a base daquele tenant, nao uma base compartilhada sem dono.

### Exemplo 3. Analise interna de consumo

Cenario: operacao precisa explicar quanto uma sessao consumiu em tokens.

Processamento: os coletores registram token_billing_event com client_id, provider, model e correlation_id.

Saida pratica: a analise consegue somar o consumo e atribui-lo a uma credencial ou sessao especifica.

## 21. Explicacao 101

Pense na plataforma como um predio com varios escritorios. Cada tenant e um escritorio. BYOK significa que cada escritorio pode usar a propria chave, a propria conta externa e os proprios acessos. O sistema de portaria primeiro confere de qual escritorio a pessoa veio. So depois ele abre o armario de segredos daquele escritorio. Ele nao pega uma chave qualquer do corredor.

Ja o isolamento de custos e a mesma ideia aplicada ao dinheiro. O ideal e que cada escritorio pague a propria conta e que o predio ainda consiga registrar quem usou o que. O codigo lido mostra que essa plataforma ja faz bem a parte da separacao de credenciais e do rastreio tecnico. O que nao apareceu como contrato completo foi um livro-caixa financeiro nativo por tenant dentro da plataforma.

## 22. Limites e pegadinhas

1. BYOK existe no runtime, mas nem todo endpoint administrativo seguro ja testa secret_ref diretamente.
2. A tabela de precos interna e global. O codigo nao confirmou precificacao por tenant.
3. Os coletores registram client_id e correlation_id, mas eu nao encontrei um ledger financeiro persistido por tenant no slice lido.
4. Isolamento de custos e mais forte quando o tenant usa credencial propria no provedor. Se todos os tenants usarem a mesma conta externa, a separacao vira apenas logica e analitica.

## 23. Troubleshooting

### Sintoma: o runtime nao consegue carregar segredo do tenant

Causa provavel: tenant_secrets sem a chave referenciada ou tenant_id errado no contexto autenticado.

### Sintoma: o cliente consegue ver ou alterar credencial de outro tenant

Pelo codigo lido, isso so deveria acontecer com escopo superadmin. Se ocorrer fora disso, investigue primeiro a montagem de user_data e o client_code resolvido pela credencial.

### Sintoma: billing nao parece separado por tenant

Causa provavel: os tenants estao compartilhando a mesma conta externa, ou a analise esta olhando apenas preco global sem separar client_id, provider e correlation_id.

## 24. Checklist de entendimento

- Entendi que BYOK aqui significa separar credencial por tenant, nao apenas armazenar segredo.
- Entendi que tenant_id e obrigatorio para o runtime multi-tenant.
- Entendi que security_keys e tenant_secrets trabalham juntos.
- Entendi que auth profiles e SQL connections sao tenant-scoped.
- Entendi que a separacao de custo mais forte vem do uso de credenciais proprias no provedor.
- Entendi que a telemetria interna existe, mas um ledger financeiro por tenant nao foi confirmado no codigo lido.

## 25. Evidencias no codigo

- src/api/security/user_auth.py
  - Motivo da leitura: confirmar que tenant_id ausente aborta a autenticacao e que o YAML e enriquecido com contexto do cliente.
  - Comportamento confirmado: falha cedo sem tenant_id e chama enrich_yaml_with_client_context.
- src/api/routers/client_portal_router.py
  - Motivo da leitura: confirmar autoatendimento de credenciais e segredos com escopo do cliente autenticado.
  - Comportamento confirmado: _resolve_target_client bloqueia operacao cross-tenant para nao superadmin e expoe /credentials e /tenant-secrets.
- src/security/security_keys_repository.py
  - Motivo da leitura: confirmar lookup de tenant_secrets por tenant_id e enriquecimento do YAML.
  - Comportamento confirmado: carrega tenant_security_keys, injeta client_context.client.tenant_id e resolve apenas segredos do tenant atual.
- src/security/security_keys_loader.py
  - Motivo da leitura: confirmar sintaxe $security_credential e falha explicita quando o segredo nao existe.
  - Comportamento confirmado: placeholder e resolvido contra tenant_secrets e gera TenantSecretNotFoundError quando necessario.
- src/integrations/models.py
  - Motivo da leitura: confirmar estrategias de credencial suportadas.
  - Comportamento confirmado: auth profiles exigem secret_ref ou credentials_json_encrypted e SQL connections exigem secret_ref ou connection_string_encrypted conforme connection_mode.
- src/integrations/schema.py
  - Motivo da leitura: confirmar tenant_id e constraints de segregacao no persistido.
  - Comportamento confirmado: registros governados possuem tenant_id e constraints unicas por tenant.
- src/integrations/repository.py
  - Motivo da leitura: confirmar que a leitura e atualizacao realmente filtram por tenant_id, nao apenas por schema.
  - Comportamento confirmado: load e update de auth profiles e SQL connections usam tenant_id no filtro.
- src/ingestion_layer/token_billing_collector.py
  - Motivo da leitura: confirmar rastreio de consumo por evento estruturado.
  - Comportamento confirmado: registra token_billing_event com client_id, provider, model, tokens e correlation_id.
- src/core/pricing_config_loader.py
  - Motivo da leitura: confirmar se a tabela de precos varia por cliente.
  - Comportamento confirmado: o proprio modulo afirma que a tabela de precos e global e nao varia por cliente.
