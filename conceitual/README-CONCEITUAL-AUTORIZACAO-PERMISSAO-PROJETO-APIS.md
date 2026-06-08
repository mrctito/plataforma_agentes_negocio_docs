# Manual conceitual, executivo, comercial e estratégico: autorização e permissão do projeto e APIs

## 1. O que é esta capacidade

O sistema de autorização e permissão deste projeto é a camada que transforma identidade em poder operacional controlado. Ele não existe só para responder se uma chave ou sessão é válida. Ele existe para decidir quem pode executar cada capacidade da plataforma, em qual contexto multi-tenant, com qual escopo humano ou técnico e com qual trilha de governança.

Na prática, esta capacidade liga quatro coisas que normalmente ficam soltas em muitos projetos:

1. a credencial técnica usada por integrações e APIs;
2. a sessão web usada pela interface administrativa e pelo portal humano;
3. a governança organizacional por papel e grants explícitos em tenant;
4. a filtragem final do que pode ou não aparecer ao usuário quando documentos recuperados carregam ACL.

Isso faz desta feature um sistema, não um helper isolado. Ela participa da borda HTTP, da sessão federada, da governança administrativa, dos canais e do consumo do acervo já indexado.

## 2. Que problema ela resolve

Sem essa camada, a plataforma teria três riscos graves.

O primeiro é o risco de identidade fragmentada. A API técnica aceitaria uma chave, a UI aceitaria um cookie e cada uma decidiria permissão de um jeito diferente. Isso tornaria auditoria, suporte e incidentes muito mais difíceis de explicar.

O segundo é o risco de autorização superficial. Em vez de um catálogo claro de capacidades, o sistema dependeria de ifs espalhados em endpoints, com comportamentos difíceis de prever quando um tenant muda de papel, quando um admin delega acesso ou quando um operador perde autorização no meio da sessão.

O terceiro é o risco de vazar dado mesmo quando a borda HTTP está protegida. Em RAG e busca corporativa, não basta autorizar a pergunta; também é preciso filtrar os documentos recuperados conforme ACL do próprio conteúdo.

O desenho atual resolve esses três problemas com um modelo único de permissões catalogadas, grants humanos efetivos, enforcement na borda HTTP, refresh de snapshot de sessão e filtro de ACL em tempo de consulta.

## 3. Visão executiva

Para liderança, esta feature importa porque reduz risco operacional e melhora governança sem paralisar a plataforma. Ela permite que o produto exponha capacidades sensíveis, como scheduler, integrações, canais, background execution e geração de configuração, sem depender de conhecimento tribal sobre quem pode fazer o quê.

O efeito prático é previsibilidade. Quando um incidente ocorre, o time consegue responder com mais clareza se o problema foi falta de credencial, falta de grant humano, sessão desatualizada, membership inativo ou ACL do conteúdo. Isso reduz tempo de diagnóstico, evita suporte improvisado e melhora confiança em ambientes corporativos.

## 4. Visão comercial

Comercialmente, esta capacidade sustenta uma promessa importante: a plataforma consegue acelerar integrações, agentes, workflows e canais sem abrir mão de controle.

A mensagem correta para cliente não é “todo usuário pode tudo desde que tenha login”. A mensagem correta é “cada capacidade é publicada com permissão explícita, o tenant governa memberships humanos com grants auditáveis e o conteúdo recuperado ainda respeita ACL na consulta”.

Isso ajuda a responder objeções comuns de cliente enterprise:

1. como separar operação humana de integração técnica;
2. como delegar acesso administrativo sem entregar controle total;
3. como revogar acesso sem depender de restart ou intervenção manual em cache;
4. como impedir que a IA mostre documentos restritos para grupos errados.

## 5. Visão estratégica

Estrategicamente, esta feature fortalece a plataforma em quatro frentes.

Primeiro, ela consolida um catálogo central de capacidades. Isso é importante porque a plataforma cresce por slices, como RAG, AG-UI, config assembly, portal do cliente, canais e integrações. Sem um catálogo comum, cada slice tenderia a inventar sua própria semântica de acesso.

Segundo, ela sustenta o desenho multi-tenant. Os memberships humanos são resolvidos por tenant, com papel organizacional e grants explícitos persistidos. Isso prepara o produto para governança mais fina sem reescrever toda a borda HTTP.

Terceiro, ela mantém a autorização como capacidade transversal, e não como detalhe de um único módulo. O mesmo desenho aparece na API técnica, na sessão web, na governança administrativa e na recuperação de documentos.

Quarto, ela reduz acoplamento entre identidade e experiência de uso. A interface administrativa pode evoluir, novos canais podem surgir e novos endpoints podem ser publicados sem reinventar o modelo de permissão a cada vez.

## 6. Conceitos necessários para entender

### 6.1. Principal

Principal é o sujeito que recebe autorização. No código lido, o modelo canônico reconhece três tipos: credencial de máquina, membership humano e principal persistido de canal. Em termos simples, isso responde à pergunta “quem está pedindo para fazer algo”.

### 6.2. Permissão catalogada

Permissão catalogada é a capacidade nomeada de forma estável, como admin.scheduler.read, agent.execute ou client.portal.secrets.write. Ela é o contrato público de autorização entre routers, UI, documentação e testes. Isso evita nomes improvisados por endpoint.

### 6.3. Audience

Audience define se uma permissão pode ser usada por principal humano, técnico, ambos ou se é pública. Isso importa porque nem toda capacidade deve ser delegável a todo tipo de principal. Algumas permissões aceitam sessão web e chave técnica; outras são apenas técnicas.

### 6.4. Papel organizacional

Papel organizacional é o perfil básico do membership humano em um tenant. No catálogo atual, owner, admin, member e billing_manager não significam apenas rótulos de tela. Eles são a base dos grants herdados.

### 6.5. Grant explícito

Grant explícito é uma exceção persistida sobre o papel base. Ele pode permitir ou negar uma permissão específica. Em linguagem simples, é o mecanismo que responde “além do papel padrão, este membership pode ou não pode fazer isso”.

### 6.6. Snapshot de autorização

Snapshot é a fotografia de papel e permissões efetivas guardada na sessão web. O projeto não trata esse snapshot como verdade eterna. Antes de enforcement sensível, ele pode ser recalculado no diretório central se a governança tiver mudado.

### 6.7. ACL de conteúdo

ACL é a regra de acesso do próprio documento recuperado. Mesmo que a requisição tenha sido autorizada, o conteúdo pode ser bloqueado se o documento exigir grupos específicos. Isso evita que a autorização da borda seja confundida com autorização do dado.

## 7. Como esta capacidade funciona por dentro

O sistema funciona em camadas complementares.

Na camada normativa, o catálogo central define as chaves de permissão, os tipos de principal, a audiência suportada e os grants base por papel. É a camada que padroniza o idioma de autorização da plataforma.

Na camada de borda HTTP, o runtime resolve a permissão exigida por cada endpoint e decide se o acesso deve vir por X-API-Key, por sessão web autenticada ou por outro modo declarado pela rota. Em muitos endpoints, a borda também injeta user_data para auditoria, rate limit e escolha de contexto.

Na camada humana, a sessão federada ou local carrega papel e permissões efetivas do membership ativo. Se a governança mudar depois do login, a próxima request protegida recalcula o snapshot antes do enforcement.

Na camada administrativa, owners e admins conseguem governar memberships humanos, mas com uma regra forte: admin não pode promover ou alterar owner. Essa camada grava grants allow e deny persistidos por tenant e invalida o snapshot da sessão para o refresh seguinte.

Na camada de dados recuperados, o motor de ACL decide se o documento pode aparecer ao usuário com base em is_restricted, allows_anonymous, visibility e permitted_groups.

## 8. Divisão em submódulos lógicos

### 8.1. Catálogo normativo de capacidades

É a parte que transforma permissão em contrato compartilhado. Ela existe para impedir nomes arbitrários e para dizer, de forma explícita, se uma capacidade admite principal humano, técnico ou ambos.

Valor entregue: previsibilidade entre API, documentação, UI e governança.

### 8.2. Enforcement na borda HTTP

É a parte que olha a rota atual e aplica a exigência correta. Ela existe para impedir que cada endpoint reimplemente autenticação e autorização manualmente.

Valor entregue: consistência de acesso entre routers e menor risco de esquecer proteção em um endpoint novo.

### 8.3. Governança humana por tenant

É a parte que calcula permissões efetivas do membership e permite atualização administrativa de papel e grants explícitos. Ela existe porque, em produto multi-tenant, login sozinho não resolve governança organizacional.

Valor entregue: delegação controlada sem reinventar perfis por tela.

### 8.4. Principal persistido de canal

É a parte que controla remetentes externos de WhatsApp e Instagram quando o canal exige persistência de principal. Ela existe para que o runtime multicanal consiga operar com observe_only, allowlist ou blocklist.

Valor entregue: controle operacional de remetente sem quebrar o runtime do canal.

### 8.5. ACL de conteúdo recuperado

É a parte que filtra documentos depois da recuperação. Ela existe porque autorizar a pergunta não é suficiente quando o dado carregado tem grupos permitidos próprios.

Valor entregue: defesa adicional contra exposição indevida de conteúdo corporativo.

## 9. Decisões e trade-offs

### 9.1. Um catálogo central em vez de nomes locais por endpoint

Ganho: o sistema fala uma língua única de autorização.

Custo: adicionar capacidade nova exige cadastro explícito no catálogo e atualização consciente.

Impacto prático: aumenta disciplina, mas evita drift de nomenclatura e facilita governança.

### 9.2. Papel base mais grants explícitos

Ganho: o papel resolve o grosso do acesso e os grants ajustam exceções.

Custo: é preciso governar allow e deny com cuidado para não criar combinações confusas.

Impacto prático: o projeto evita perfis artesanais demais, mas ainda permite delegação fina.

### 9.3. Snapshot de sessão com refresh antes de enforcement

Ganho: a sessão continua rápida, mas não fica cega a mudanças de governança.

Custo: existe complexidade adicional de invalidação e reidratação do snapshot.

Impacto prático: revogação e ajuste de grants não dependem de esperar expiração completa do cookie.

### 9.4. ACL também no dado, não só na rota

Ganho: reduz risco de vazamento em fluxos RAG e busca.

Custo: exige que a ingestão e o storage preservem metadados de autorização.

Impacto prático: resposta boa tecnicamente ainda pode ser suprimida se o documento não for permitido ao usuário atual.

## 10. Impacto técnico

Tecnicamente, a feature reforça três padrões importantes do projeto: catálogo central, governança multi-tenant e separação entre autorização da requisição e autorização do conteúdo.

Ela também reduz acoplamento porque o endpoint não precisa conhecer todos os detalhes do membership. A camada de diretório resolve o contexto e a borda só consome o resultado.

## 11. Impacto executivo

Executivamente, a feature reduz risco de permissão incorreta, melhora rastreabilidade de acessos administrativos e diminui dependência de intervenção manual para revogar ou delegar acessos.

## 12. Impacto comercial

Comercialmente, a capacidade ajuda a posicionar a plataforma como enterprise-ready. O produto não depende de uma única chave mestre nem de perfis genéricos sem auditabilidade. Ele já traz governança, revogação e separação entre integração técnica e operação humana.

## 13. Impacto estratégico

Estrategicamente, esta base prepara o produto para crescer com menos dívida de segurança. Novos módulos podem herdar o catálogo e a governança existentes, em vez de criar outra matriz de acesso paralela.

## 14. Explicação 101

Pense no sistema como um prédio com vários andares.

A autenticação responde quem entrou no prédio. A autorização responde em quais andares essa pessoa ou integração pode circular. A governança responde quem pode entregar crachás para outras pessoas. E a ACL do documento responde se, mesmo dentro do andar certo, aquele armário específico pode ser aberto.

O projeto foi desenhado para não confundir essas quatro coisas.

## 15. Limites e pegadinhas

1. Ter sessão válida não garante poder administrativo. Os endpoints administrativos de governança humana ainda verificam papel owner ou admin no tenant alvo.
2. Ter permissão de borda não garante visibilidade de documento. A ACL do conteúdo pode bloquear a evidência depois da recuperação.
3. O catálogo normativo declara precedência canônica com superadmin, explicit deny, explicit allow, role base e default deny, mas no slice humano lido a composição executável confirmada foi role base mais allow menos deny; não encontrei no código lido um fluxo separado exercendo superadmin humano na borda web.
4. O projeto possui dois mecanismos complementares na borda HTTP: metadata por endpoint_permission e dependências explícitas com require_permission. Tratar isso como duplicação simples seria leitura rasa; o código mostra convivência intencional entre registro central por rota e endpoints que precisam de user_data ou sessão web.
5. A documentação anterior citava README-AUTORIZACAO.md, mas esse arquivo não foi encontrado no workspace lido. Isso é lacuna documental, não comportamento do runtime.

## 16. Checklist de entendimento

- Entendi a diferença entre principal técnico, membership humano e principal de canal.
- Entendi que permissões são catalogadas e têm audience explícita.
- Entendi que papel organizacional não é o único fator; grants allow e deny completam a governança.
- Entendi que sessão web pode ser recalculada antes do enforcement.
- Entendi que owner e admin não têm o mesmo poder na governança de memberships.
- Entendi que a autorização da rota não substitui a ACL do conteúdo.
- Entendi o valor executivo, comercial e estratégico dessa capacidade.

## 17. Evidências no código

- src/api/security/permissions.py
  - Motivo da leitura: confirmar catálogo, audience, papéis e grants base.
  - Comportamento confirmado: PermissionKeys centraliza a taxonomia e MembershipRole define herança base.
- src/api/security/user_auth.py
  - Motivo da leitura: confirmar enforcement por X-API-Key e sessão web.
  - Comportamento confirmado: require_permission aceita sessão federada quando não há chave e recalcula snapshot humano quando necessário.
- src/security/user_yaml_membership_governance_repository_mixin.py
  - Motivo da leitura: confirmar grants humanos efetivos e regras de governança owner/admin.
  - Comportamento confirmado: permissões efetivas são role base mais allow menos deny; owner/admin governam, mas admin não altera owner.
- src/security/access_control.py
  - Motivo da leitura: confirmar ACL na consulta.
  - Comportamento confirmado: documentos restritos dependem de groups, autenticação e metadados de ACL.
- src/channel_layer/processor.py
  - Motivo da leitura: confirmar principal persistido de canal.
  - Comportamento confirmado: canais podem operar com observe_only, allowlist ou blocklist sobre remetente persistido.
