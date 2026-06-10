# Compliance e segurança enterprise da plataforma

## 1. Objetivo deste documento

Este documento existe para responder uma pergunta recorrente em contas
enterprise: o que a plataforma já comprova em segurança, governança e
rastreabilidade, e o que ainda depende de formalização organizacional ou
certificação externa.

Em linguagem simples: este manual não existe para prometer selo que o
repositório não prova. Ele existe para organizar, de forma honesta, os
controles que já aparecem na arquitetura e na documentação do produto e
separá-los do que ainda precisa virar política corporativa, processo ou
auditoria formal.

## 2. Resumo executivo

Com base na documentação atual do repositório, a plataforma já demonstra
uma postura técnica forte em seis frentes.

1. segregação multi-tenant com contexto, segredos e integrações por
   tenant;
2. autenticação e autorização explícitas, sem fronteiras soltas;
3. trilha de observabilidade com correlation_id e logging estruturado;
4. governança de configuração por YAML validado e AST no escopo agentic;
5. controle de execução sensível com Human in the Loop;
6. falha fechada em vez de fallback implícito para cenários críticos.

Ao mesmo tempo, o repositório não comprova, por si só, que a empresa já
possui certificações formais como SOC 2, ISO 27001, GDPR ou LGPD
formalmente auditada. Também não comprova, sozinho, um programa completo
de incident response, BCP, disaster recovery, política jurídica de
retenção ou data room regulatória final.

O uso correto deste documento é este:

1. sustentar conversa técnica e comercial com segurança;
2. evitar promessa indevida de compliance formal;
3. organizar o que já está evidenciado e o que ainda precisa de camada
   organizacional.

## 3. O que já está comprovado no produto

## 3.1. Isolamento por tenant

O recorte documental atual comprova que a plataforma foi desenhada para
resolver contexto, segredos, integrações e consumo por tenant, com
tenant_id explícito no runtime e catálogos governados segmentados.

Isso aparece de forma especialmente clara em
[docs/README-CONCEITUAL-BYOK-ISOLAMENTO-CUSTOS-TENANT.md](../conceitual/README-CONCEITUAL-BYOK-ISOLAMENTO-CUSTOS-TENANT.md).

Na prática, isso significa:

1. o runtime não deveria operar sem tenant definido;
2. segredos e integrações não deveriam ser misturados entre clientes;
3. a plataforma já foi pensada para contas enterprise e white-label com
   segregação forte.

## 3.2. Autenticação e sessão governadas

Os documentos de autenticação mostram uma camada dedicada para login,
sessão própria, MFA com Google e política de sessão centralizada.

Referências principais:

1. [docs/README-CONCEITUAL-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md](../conceitual/README-CONCEITUAL-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md)
2. [docs/README-TECNICO-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md](../tecnico/README-TECNICO-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md)

Valor prático para segurança:

1. o produto não depende de sessão improvisada;
2. MFA aparece como capacidade nativa do ecossistema;
3. a borda HTTP já nasce com noção de identidade e contexto.

## 3.3. Autorização e controle de permissão

Os manuais de autorização mostram que a plataforma trata permissão como
contrato transversal, não como detalhe local de endpoint.

Referências principais:

1. [docs/README-CONCEITUAL-AUTORIZACAO-PERMISSAO-PROJETO-APIS.md](../conceitual/README-CONCEITUAL-AUTORIZACAO-PERMISSAO-PROJETO-APIS.md)
2. [docs/README-TECNICO-AUTORIZACAO-PERMISSAO-PROJETO-APIS.md](../tecnico/README-TECNICO-AUTORIZACAO-PERMISSAO-PROJETO-APIS.md)

Valor prático:

1. reduz risco de fronteira exposta sem guarda;
2. melhora auditoria de acesso;
3. sustenta conversa com contas que exigem autorização nominal por
   capacidade.

## 3.4. Rastreabilidade ponta a ponta com correlation_id

O sistema de logs documentado mostra uma arquitetura em que a execução
ganha identidade lógica, trilha estruturada e leitura administrativa por
provider canônico.

Referências principais:

1. [docs/README-CONCEITUAL-ARQUITETURA-LOGGING-CORRELATION-ID.md](../conceitual/README-CONCEITUAL-ARQUITETURA-LOGGING-CORRELATION-ID.md)
2. [docs/README-TECNICO-ARQUITETURA-LOGGING-CORRELATION-ID.md](../tecnico/README-TECNICO-ARQUITETURA-LOGGING-CORRELATION-ID.md)

Valor prático:

1. permite reconstruir a história de uma execução;
2. reduz caixa-preta operacional;
3. melhora suporte, auditoria e investigação de incidente.

## 3.5. Governança de configuração e publicação

O repositório comprova que o comportamento agentic não é tratado como
texto livre. A montagem passa por YAML, AST, validação semântica e
confirmação governada.

Referências principais:

1. [docs/README-CONFIGURACAO-YAML.md](../tecnico/README-CONFIGURACAO-YAML.md)
2. [docs/README-AST-AGENTIC-DESIGNER.md](../tecnico/README-AST-AGENTIC-DESIGNER.md)
3. [docs/README-CONCEITUAL-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md](../conceitual/README-CONCEITUAL-CONFIGURACAO-YAML-AGENTES-WORKFLOW-ETL.md)

Valor prático:

1. reduz improviso de configuração crítica;
2. dificulta drift silencioso entre UI e runtime;
3. melhora previsibilidade em ambientes sensíveis.

## 3.6. Human in the Loop para decisões sensíveis

A documentação AG-UI, HIL e workflows sustenta que a plataforma não foi
desenhada para automação cega em todos os fluxos. Ela já prevê pausa,
aprovação humana, retomada e trilha da decisão.

Referências principais:

1. [docs/README-CONCEITUAL-HIL-APIS-WHATSAPP.md](../conceitual/README-CONCEITUAL-HIL-APIS-WHATSAPP.md)
2. [docs/README-CONCEITUAL-AG-UI.md](../conceitual/README-CONCEITUAL-AG-UI.md)
3. [docs/README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md](../conceitual/README-CONCEITUAL-AGENDAMENTO-AGENTIC-BACKGROUND-HIL.md)

Valor prático:

1. reduz risco em ação sensível;
2. torna a IA mais aceitável para processo corporativo;
3. fortalece governança operacional diante de auditoria.

## 3.7. Falha fechada em vez de fallback implícito

Vários documentos do repositório reforçam a mesma postura: quando uma
dependência obrigatória, configuração crítica ou fronteira de segurança
não fecha, o comportamento desejado é falhar cedo e de forma observável.

Isso aparece, por exemplo, na documentação de arquitetura, logging,
autenticação, canais e BYOK.

Valor prático:

1. evita degradação silenciosa;
2. reduz risco operacional mascarado;
3. melhora confiança em troubleshooting e em governança.

## 4. Matriz resumida de controles

| Tema | Estado documental atual | Valor prático |
| --- | --- | --- |
| Isolamento por tenant | comprovado | separa contexto, segredos e integrações |
| MFA e autenticação | comprovado | fortalece identidade e sessão |
| Autorização | comprovado | limita acesso por capacidade |
| Correlation_id e logs | comprovado | viabiliza auditoria e suporte |
| YAML governado e AST | comprovado | reduz mudança solta e comportamento opaco |
| HIL | comprovado | adiciona controle humano em processo sensível |
| Retry e fail-fast | amplamente evidenciado | reduz falha silenciosa |
| Certificação formal SOC 2 | não comprovado | depende de programa corporativo |
| Certificação formal ISO 27001 | não comprovado | depende de auditoria organizacional |
| Programa formal LGPD/GDPR | não comprovado como certificação | depende de governança jurídica e processual |
| BCP e DR formais | não comprovados | dependem de documentação corporativa própria |
| Política formal de retenção | não comprovada como norma corporativa | depende de governança operacional e jurídica |

## 5. O que este documento não deve prometer

Para usar este material corretamente, é importante deixar claro o que o
repositório não prova por si só.

1. não prova certificação SOC 2;
2. não prova certificação ISO 27001;
3. não prova laudo formal de LGPD ou GDPR;
4. não prova política corporativa completa de incident response;
5. não prova plano corporativo fechado de disaster recovery e business
   continuity;
6. não prova data processing agreement formal para todos os contextos de
   cliente.

Isso não enfraquece o produto. O que enfraquece é prometer o que o
material não comprova. A postura correta é esta:

1. usar o que já está comprovado como base técnica forte;
2. tratar certificação formal como trilha organizacional separada;
3. documentar abertamente o que ainda depende de processo da empresa.

## 6. Como o comercial deve usar este documento

O time comercial deve usar este material como apoio de conversa, não
como substituto do questionário de segurança do cliente.

A lógica correta é:

1. mostrar que a plataforma já nasce com arquitetura séria;
2. apontar os controles técnicos já evidenciados;
3. separar o que é característica do produto do que é exigência de
   certificação corporativa;
4. encaminhar temas formais para segurança, jurídico e liderança quando
   necessário.

## 7. Perguntas frequentes em diligência enterprise

### 7.1. A plataforma separa dados de clientes diferentes?

Com base na documentação atual, sim, a arquitetura foi desenhada para
resolver contexto, segredos e integrações por tenant. O que este manual
afirma com segurança é a existência de segregação lógica e de runtime,
não um laudo externo de isolamento formal.

### 7.2. Há trilha de auditoria?

Sim, a documentação sustenta correlation_id, logging estruturado,
identidade lógica de execução e leitura administrativa de logs por
provider canônico.

### 7.3. A plataforma possui MFA?

Sim, o repositório documenta autenticação com Google e trilha de MFA.

### 7.4. A plataforma impede automação irrestrita?

Ela foi desenhada para combinar contratos de autorização, governança de
configuração, HIL e superfícies controladas de execução. Isso reduz o
espaço para automação cega.

### 7.5. A empresa já tem certificação SOC 2 ou ISO 27001?

Este repositório não comprova isso. Se essa exigência existir, a resposta
correta depende de evidência corporativa fora do escopo deste código.

## 8. Checklist mínimo para diligência de segurança

Quando um cliente enterprise pedir diligência, o pacote mínimo deveria
combinar este documento com os seguintes manuais.

1. [docs/README-CONCEITUAL-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md](../conceitual/README-CONCEITUAL-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md)
2. [docs/README-CONCEITUAL-AUTORIZACAO-PERMISSAO-PROJETO-APIS.md](../conceitual/README-CONCEITUAL-AUTORIZACAO-PERMISSAO-PROJETO-APIS.md)
3. [docs/README-CONCEITUAL-ARQUITETURA-LOGGING-CORRELATION-ID.md](../conceitual/README-CONCEITUAL-ARQUITETURA-LOGGING-CORRELATION-ID.md)
4. [docs/README-CONCEITUAL-BYOK-ISOLAMENTO-CUSTOS-TENANT.md](../conceitual/README-CONCEITUAL-BYOK-ISOLAMENTO-CUSTOS-TENANT.md)
5. [docs/API-ENDPOINTS-SWAGGER.md](../tecnico/API-ENDPOINTS-SWAGGER.md)
6. docs/README-TESTS.MD

## 9. Próximos passos recomendados

Para evoluir de postura técnica forte para pacote enterprise completo,
os próximos documentos mais importantes seriam:

1. política formal de incident response;
2. política formal de retenção e descarte de dados;
3. documento de BCP e disaster recovery;
4. questionário padrão de segurança para pré-venda;
5. matriz de responsabilidades produto versus operação.

## 10. Explicação 101

Pense assim: este produto já mostra muitos sinais de que foi construído
com cabeça de plataforma séria. Ele separa clientes, exige identidade,
registra execuções, controla permissões e evita vários atalhos
perigosos. Isso é diferente de dizer que a empresa já tem todos os selos
e processos formais do mundo corporativo. Uma coisa é a maturidade do
produto. Outra coisa é a maturidade completa do programa de compliance
da empresa. Este documento serve justamente para não misturar as duas.
