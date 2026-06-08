# Manual conceitual, executivo, comercial e estratégico: hash governado do YAML agentic

## 1. O que é esta feature

O hash governado é o selo de integridade do trecho agentic governado do YAML.

Em linguagem simples: pense no YAML como um contrato escrito. O hash governado é o carimbo que diz “este pedaço crítico do contrato foi recomposto pelo trilho oficial do sistema e está sincronizado com o assembly”.

Esse selo não serve para esconder conteúdo nem para criptografar nada. O objetivo dele é muito mais pragmático: detectar quando o YAML que está salvo no arquivo deixou de bater com o fragmento canônico que o runtime espera usar.

No projeto, isso vale especialmente para os alvos agentic oficiais:

- workflow;
- deepagent_supervisor.

## 2. Que problema ela resolve

O problema real não é “falta um hash”. O problema real é drift.

Drift significa divergência entre duas coisas que deveriam estar sincronizadas:

- o YAML versionado no repositório;
- o fragmento governado que o assembly recompõe e que o runtime usa como referência canônica.

Sem esse mecanismo, alguém pode alterar o YAML, manter um selo antigo e deixar o sistema em uma situação perigosa: o arquivo parece válido, mas a parte governada já não representa mais o contrato oficial confirmado pelo assembly.

Na prática, isso cria três riscos graves:

- o runtime passa a trabalhar com um artefato aparentemente correto, mas semanticamente fora de sincronia;
- a investigação de erro fica confusa, porque o arquivo no repositório parece “certo” enquanto a validação oficial discorda;
- a operação perde governança, porque não há uma forma objetiva de provar que o YAML foi regenerado pelo caminho oficial.

## 3. Visão executiva

Para liderança, este mecanismo importa porque transforma um risco técnico invisível em um sinal objetivo e auditável.

O benefício executivo não é “temos SHA-256”. O benefício executivo é:

- maior previsibilidade do runtime agentic;
- menor risco de publicação de YAML parcialmente editado;
- trilha mais forte entre validação, confirmação e execução;
- menos tempo desperdiçado em incidentes causados por divergência silenciosa de configuração.

## 4. Visão comercial

Em uma conversa comercial, o valor desta feature é simples de explicar: a plataforma não depende de disciplina manual para manter a configuração agentic consistente. Ela tem um mecanismo formal que detecta quando o trecho governado do YAML saiu do trilho oficial.

Isso ajuda a responder objeções como:

- “Se alguém editar o YAML manualmente, como vocês sabem que isso ainda bate com o runtime?”
- “Como vocês evitam que uma configuração ‘quase certa’ chegue em produção?”
- “Como vocês comprovam que o YAML publicado passou pelo fluxo correto?”

Resposta prática: o sistema recalcula o fragmento governado pelo assembly, gera um fingerprint determinístico e grava esse selo no próprio YAML. Se o conteúdo divergir, a deriva é detectada.

## 5. Visão estratégica

Estratégicamente, o hash governado fortalece quatro pilares da plataforma:

- YAML-first com governança real, e não apenas com convenção informal;
- assembly AST como ponto único de verdade para o trecho agentic;
- runtime mais confiável, porque consegue distinguir YAML sincronizado de YAML apenas “parecido”;
- evolução futura mais segura para DeepAgent e Workflow, sem abrir espaço para caminhos paralelos de publicação.

## 6. Conceitos necessários para entender

### 6.1 YAML governado

Nem todo campo do YAML participa deste selo. O foco é o fragmento governado do alvo agentic.

Em DeepAgent, o detector usa como conjunto padrão:

- selected_supervisor;
- multi_agents;
- tools_library.

Em Workflow, o conjunto padrão é:

- selected_workflow;
- workflows_defaults;
- workflows;
- tools_library.

### 6.2 AST agentic

AST é a forma estruturada e tipada do documento agentic. O assembly não trata o YAML como “texto solto”; ele reconstrói uma representação formal do conteúdo antes de validar, compilar e confirmar.

Em termos 101: a AST é a forma como o sistema entende o YAML de maneira organizada, sem depender de edição manual linha por linha.

### 6.3 Fingerprint

O fingerprint é o hash calculado sobre um payload determinístico contendo:

- target;
- fragment.

O projeto serializa isso de forma estável e aplica SHA-256. Isso permite que o mesmo conteúdo gere sempre o mesmo resultado.

### 6.4 Drift

Drift é a situação em que o hash armazenado no YAML não bate com o hash recalculado a partir do fragmento governado atual.

### 6.5 Selo canônico

O selo fica em:

- metadata.agentic_assembly.governed_hashes.workflow;
- metadata.agentic_assembly.governed_hashes.deepagent_supervisor.

Esse bloco registra:

- hash;
- algorithm;
- version;
- governed_keys.

## 7. Como a feature funciona por dentro

O comportamento real observado no código segue esta sequência conceitual:

1. o sistema identifica qual alvo agentic está presente no YAML;
2. ele extrai o fragmento governado daquele alvo;
3. ele monta um payload determinístico com target + fragment;
4. ele gera o fingerprint SHA-256;
5. ele grava o selo no bloco metadata.agentic_assembly.governed_hashes;
6. no runtime, ele recalcula o hash e compara com o valor armazenado.

Se os valores forem iguais, o YAML está sincronizado para aquele alvo.

Se os valores forem diferentes, o sistema registra deriva.

## 8. Qual interface corrige esse problema

O projeto tem três caminhos oficiais que passam pelo mesmo boundary correto.

### 8.1 Interface web administrativa do assembly

O painel AST administrativo usa o fluxo oficial do backend:

- objective-to-yaml;
- validate;
- confirm.

Em termos práticos, é a interface web que permite gerar preview, validar e confirmar o YAML sem sair do trilho canônico.

### 8.2 Studio de objetivo para YAML

O Studio também chama o mesmo conjunto de rotas oficiais. Isso é importante porque mostra que a interface “mais assistida” não inventa um caminho próprio para o hash. Ela usa o mesmo motor oficial do assembly.

### 8.3 Helper público e CLI

Para operação direta, agora existem dois artefatos específicos:

- o helper público exportado em assembly_service.py;
- o script refresh_agentic_governed_hash.py.

Esses dois artefatos existem justamente para dar autonomia operacional sem abrir um atalho errado.

## 9. Como deve ser a jornada correta de geração do hash

Esta é a parte mais importante do documento.

O hash governado não deve ser tratado como um valor para “preencher”. Ele deve ser tratado como o resultado final de uma jornada oficial.

### Jornada correta

1. partir do YAML atual;
2. identificar o alvo agentic correto;
3. reconstruir a AST oficial a partir do YAML atual;
4. transformar a AST de volta no fragmento governado oficial;
5. passar pelo mesmo trilho do confirm com force=true;
6. obter o YAML final já com metadata.agentic_assembly.governed_hashes sincronizado;
7. persistir esse YAML apenas quando a operação pedir gravação.

### O que não fazer

- não copiar hash antigo de outro arquivo;
- não editar manualmente o valor em metadata;
- não tentar recalcular “por fora” com script ad hoc;
- não tratar o hash como correção cosmética.

### Por que isso importa

Porque o selo só é confiável quando nasce do mesmo caminho que o produto reconhece como canônico.

Se o valor for gerado fora desse trilho, ele até pode “parecer” certo, mas deixa de ter valor de governança.

## 10. Por que o helper público foi exportado

O helper público foi exposto para resolver um problema operacional real: quando o YAML já existe e a pessoa precisa sincronizar o hash governado sem reescrever todo o fluxo manualmente.

O ponto central é este: o helper não inventa hash e não recalcula por fora. Ele pega o YAML atual, reconstrói a AST oficial e delega para o mesmo confirm do serviço, com force=true.

Em termos 101, ele age como um operador disciplinado que refaz o processo pelo balcão oficial, em vez de carimbar o documento na mão.

## 11. O que o script entrega na prática

O CLI existe para dois usos diferentes.

### Uso 1: conferir sem gravar

Você quer saber se o YAML atual gera um novo hash, mas ainda não quer alterar o arquivo.

### Uso 2: sincronizar e gravar

Você já sabe que precisa corrigir o drift e quer persistir o YAML governado no mesmo arquivo.

Essa separação é importante porque evita alterações involuntárias e permite inspeção antes da gravação final.

## 12. FAQ com 20 perguntas e respostas

1. O hash governado existe para segurança criptográfica?
   Não. O objetivo principal aqui é integridade e governança do fragmento governado, não sigilo de conteúdo.

2. O hash protege o YAML inteiro?
   Não. Ele protege o recorte governado do alvo agentic relevante.

3. Por que isso não pode ser feito só com convenção de time?
   Porque convenção humana falha. O hash governado transforma disciplina desejada em verificação objetiva.

4. Se eu mudar um comentário no YAML, o hash muda?
   Não necessariamente. O que importa é o fragmento governado reconstruído pelo assembly, não uma diferença textual irrelevante.

5. Se eu mudar um campo fora do fragmento governado, o hash muda?
   Em regra, não. O foco é o conjunto de chaves governadas do alvo.

6. Por que o alvo importa tanto?
   Porque Workflow e DeepAgent têm fragmentos governados diferentes. O sistema precisa saber qual contrato está sendo selado.

7. O hash pode existir e ainda assim o YAML estar errado?
   Sim. O hash prova sincronização do fragmento governado, mas não substitui validação semântica completa.

8. O que acontece se o hash estiver ausente?
   O runtime pode diagnosticar que falta selo canônico e exigir passagem pelo confirm oficial.

9. O que acontece se o bloco metadata estiver malformado?
   O detector trata isso como metadata inválida, porque o selo deixou de ser confiável.

10. O sistema sempre bloqueia drift?
   Depende do modo de runtime. Há modo de aviso e modo de enforcement.

11. Por que o helper usa force=true?
   Porque o objetivo dele é refazer a sincronização oficial do YAML governado a partir do estado atual, sem ficar preso a bloqueios típicos de fluxo interativo.

12. O helper cria um caminho paralelo ao confirm?
   Não. Ele chama o próprio confirm por dentro, montando o ast_payload a partir da AST oficial do YAML atual.

13. O CLI substitui a interface web?
   Não. Ele complementa a interface web com autonomia operacional para uso direto no repositório.

14. Qual interface web corrige esse problema hoje?
   As interfaces que usam objective-to-yaml, validate e confirm do assembly oficial.

15. Posso usar o script sem informar target?
   Sim, mas somente quando o script conseguir inferir um alvo único a partir do YAML.

16. Quando eu devo informar target manualmente?
   Quando houver ambiguidade ou quando você quiser garantir explicitamente que o recálculo será feito para workflow ou deepagent_supervisor.

17. O hash é gerado a partir do arquivo bruto do disco?
   Não diretamente. O fluxo oficial passa pela reconstrução da AST e pelo confirm do assembly.

18. Isso resolve sozinho qualquer erro de runtime agentic?
   Não. Ele resolve o problema de sincronização do fragmento governado. Outros erros ainda podem existir.

19. O problema real do HIL/DeepAgent pode aparecer como drift?
   Sim. Drift é um dos sinais de que o YAML versionado ficou desalinhado do contrato que o runtime exige.

20. Qual é a regra prática mais segura?
   Nunca trate o hash como campo manual. Trate-o como subproduto do trilho oficial do assembly.

## 13. Impacto técnico

Tecnicamente, o hash governado reduz a chance de rodar um YAML agentic fora de sincronia com o assembly oficial. Isso melhora previsibilidade, investigação e proteção contra edição manual indevida do trecho governado.

## 14. Impacto executivo

Executivamente, isso reduz risco de operação com configuração inconsistente e melhora a confiabilidade do processo de publicação.

## 15. Impacto comercial

Comercialmente, isso reforça a mensagem de plataforma governada. A promessa correta não é “nunca haverá erro”, e sim “o produto detecta e explicita quando a configuração governada saiu do trilho oficial”.

## 16. Impacto estratégico

Estratégicamente, o mecanismo fortalece a proposta YAML-first sem cair em configuração solta. Ele aproxima YAML, AST, validação e runtime em um contrato único.

## 17. Explicação 101

Imagine que o YAML é um contrato assinado entre quem configura o sistema e quem executa o runtime. O hash governado é o selo do cartório. Ele não escreve o contrato, não decide as cláusulas e não substitui revisão jurídica. O que ele faz é confirmar que a parte crítica daquele contrato passou pelo cartório certo.

Se alguém muda a parte crítica depois, o selo deixa de bater. É isso que o sistema está checando.

## 18. Limites e pegadinhas

- hash governado não substitui validação semântica;
- hash governado não significa que toda a lógica do YAML está boa;
- hash governado não deve ser reparado manualmente;
- interface web, helper e CLI só são confiáveis porque usam o mesmo backend oficial.

## 19. Evidências no código

- src/config/agentic_assembly/drift_detector.py
  - Motivo da leitura: confirma o conceito de fragmento governado, fingerprint, selo e deriva.
  - Comportamento confirmado: o hash nasce de target + fragment serializados de modo determinístico com SHA-256.

- src/config/agentic_assembly/assembly_service.py
  - Motivo da leitura: confirma o helper público.
  - Comportamento confirmado: o helper reconstrói a AST do YAML atual e delega ao confirm oficial.

- scripts/refresh_agentic_governed_hash.py
  - Motivo da leitura: confirma a autonomia operacional via CLI.
  - Comportamento confirmado: o script carrega YAML, resolve target, chama o helper e opcionalmente grava o arquivo.

- src/api/routers/config_assembly_router.py
  - Motivo da leitura: confirma a borda HTTP oficial.
  - Comportamento confirmado: a família /config/assembly é o boundary oficial para objective-to-yaml, validate e confirm.

- app/ui/static/js/admin-assembly-ast.js
  - Motivo da leitura: confirma a interface web administrativa.
  - Comportamento confirmado: a UI chama objective-to-yaml, validate e confirm do assembly oficial.

- app/ui/static/js/objective-yaml-studio.js
  - Motivo da leitura: confirma a interface assistida de objetivo para YAML.
  - Comportamento confirmado: o Studio também usa o mesmo trilho de confirm oficial.
