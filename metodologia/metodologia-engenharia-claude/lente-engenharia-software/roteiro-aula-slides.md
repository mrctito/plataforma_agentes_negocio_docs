# Roteiro de Aula — A Fábrica de Software (tela + fala + interação)

> 🏭 [↩ Doc-mãe da lente](README.md) · [↩ Índice da metodologia](../README.md)
>
> **Como usar este roteiro.** Cada slide tem três blocos: **TELA** (o que projetar — use o
> [deck Marp](apresentacao-marp.md)), **FALA** (o que dizer, em linguagem 101) e, quando útil,
> **INTERAÇÃO** (pergunta para a plateia). Duração-alvo: ~45 min + perguntas. Os slides marcados ⏩ podem
> ser acelerados se o tempo apertar. Esta é a **lente fábrica**; há uma lente paralela (futebol) com o
> mesmo conteúdo — escolha pelo público.

---

## ATO 0 — Abertura (≈5 min)

### Slide 1 — Capa
- **TELA:** "A Fábrica de Software — como transformamos a IA em uma linha de produção com controle de qualidade."
- **FALA:** "Não vou explicar 'o que é um agente' — isso está na doc da Anthropic. Vou mostrar a
  **engenharia por trás dos nossos arquivos de configuração**, usando uma metáfora que mapeia quase 1:1 na
  engenharia de software de verdade: uma **fábrica enxuta**."

### Slide 2 — A pergunta que abre tudo
- **TELA:** "Qual a diferença entre 'usar IA pra programar' e 'fazer engenharia com IA'?"
- **INTERAÇÃO:** jogue a pergunta. Deixe 2-3 pessoas responderem. Não corrija — só ouça e anote.
- **FALA:** "Guardem as respostas; no fim a gente compara. Minha resposta em uma frase: é a diferença
  entre **produção artesanal** e uma **linha de produção**."

### Slide 3 — A tese central
- **TELA:** "Nós montamos uma linha de produção de software dentro de arquivos de texto."
- **FALA:** "O valor não está na máquina (a IA). Está na **engenharia do processo**. A nossa estrutura
  obriga a IA a produzir como uma fábrica madura com QA — não como um artesão apressado e genial."

---

## ATO 1 — O Layout da Linha (≈8 min)

### Slide 4 — O chão de fábrica
- **TELA:** a tabela artefato → parte da fábrica (Norma, dispositivos, estações, ordens, SOPs).
- **FALA:** "Cinco grupos de arquivos, cinco partes da fábrica. No topo, a **Norma** — lida sempre. As
  linhas de proteção são **dispositivos automáticos**. No fluxo, as **estações**. E as **instruções de
  trabalho** na prateleira, consultadas na hora certa."

### Slide 5 — O parque em números
- **TELA:** "22 · 19 · 12 · 7 · **0**" (estações · ordens · especificações · dispositivos · commands).
- **FALA:** "O zero é de propósito. A gente **escolheu** não usar 'commands'. As skills já fazem esse
  papel, melhor. Menos mecanismos paralelos = menos confusão. Volto nisso no Ato 3."

### Slide 6 — A célula central
- **TELA:** `/investigar → /planejar → /implementar → /validar-entrega` (inspeção → PCP → fabricação → QA).
- **FALA:** "Esta célula resolve quase todo trabalho não-trivial. O segredo é a **segregação de funções**:
  quem inspeciona não planeja, quem planeja não fabrica, quem fabrica não libera a própria peça. Cada
  passagem de bastão é uma checagem independente."
- **INTERAÇÃO:** "Por que separar quem fabrica de quem aprova?" → puxe a ideia de controle interno.

### Slide 7 — Os 5 valores
- **TELA:** evidência · rastreabilidade · anti-falso-verde · reuso · escopo mínimo.
- **FALA:** "Todos os arquivos existem para forçar estes cinco valores. É a política de qualidade da
  fábrica. Eles vão reaparecer em cada estação."

---

## ATO 2 — As Estações e as Especificações (≈18 min)

### Slide 8 — A Norma: CLAUDE.md
- **TELA:** "Lida em 100% das sessões. Curta e referencial."
- **FALA:** "A Norma converte cultura em **especificação aplicada**. Não é 'seria bom testar' — é 'a peça
  não passa sem ensaio'. É enxuta de propósito: a instrução densa entra só quando o trabalho toca o tema."

### Slide 9 — O detalhe genial do "100%"
- **TELA:** "'100%' qualifica a execução, não autoriza o escopo."
- **FALA:** "Erro clássico do operário zeloso: ouvir '100%' e refazer a planta inteira. A Norma corta
  isso. Faça o **lote certo** com perfeição — não invente encomenda nova. Controle de custo puro."

### Slide 10 — As especificações de processo (rules/)
- **TELA:** SOPs profundos · contratos de passagem de bastão · diário de bordo (Kaizen).
- **FALA:** "São as instruções de trabalho na estação. Ponto DRY elegante: os contratos de passagem de
  bastão vivem em arquivo próprio para **não duplicar** entre estações. A fábrica obedece às regras que
  impõe ao produto."

### Slide 11 — A especificação mais valiosa: o raio-X
- **TELA:** `log-instructions.md` — correlation_id, campos canônicos, logger.exception, anti-varredura.
- **FALA:** "É a **caixa-preta** da produção. Quando algo dá errado no cliente, a diferença entre
  'resolvemos em minutos' e 'dias no escuro' mora aqui. O log permite reconstruir o que aconteceu **depois**
  — debug offline."
- **INTERAÇÃO:** "Quem já perdeu uma tarde porque o log não dizia nada?"

### Slide 12 — Anti-falso-verde
- **TELA:** "Parece pronto, mas a linha oficial não usa a peça nova."
- **FALA:** "O golpe mais comum em automação: tudo verde, produto real não mudou. A definição-de-pronto
  exige provar a peça **instalada no produto**, não só fabricada na bancada."
- **INTERAÇÃO:** "Quem já viu um ensaio passar e o defeito continuar no cliente?"

### Slide 12-bis — O laudo que prova: a suíte
- **TELA:** "A suíte emite laudo auditável: `telemetry.json` · `state.json` · `logs/` · `reports/`; `run_id` novo/`--resume`/`--checkup`; foco × fechamento."
- **FALA:** "Anti-falso-verde é a regra; a **suíte** é a máquina que emite a prova. Não é um 'verde' que
  some — é laudo que fica. Ordem de leitura: resumo primeiro, detalhe depois. `pytest` direto é a lupa de
  bancada; a suíte é o **certificado de ensaio** que outro turno reabre amanhã. Sem artefato auditável,
  não há validação confiável."
- **APROFUNDAMENTO:** [../base-conceitual/suite-como-prova.md](../base-conceitual/suite-como-prova.md)

### Slide 13 — As estações: a célula central
- **TELA:** inspeção → PCP → fabricação → QA, com "contratos opostos de propósito".
- **FALA:** "Quatro estações, quatro propósitos opostos. Essa oposição é intencional — é segregação de
  funções, como nos controles internos de uma empresa: quem aprova não é quem executa."

### Slide 14 — O laboratório de metrologia ⏩
- **TELA:** analisar-log · validar-log · validar-observabilidade · corrigir-erros-com-log · testar-cli-log-analyzer.
- **FALA (rápido):** "Cinco estações que garantem diagnóstico. Detalhe fino: o analisar-log valida
  **paralelismo real** por sobreposição temporal — 'a UI disse 5 workers' não prova que 5 rodaram juntos."

### Slide 15 — Manutenção do chão (infra) ⏩
- **TELA:** postgres · redis · job-core · scheduler → snapshot, dry-run, lockout.
- **FALA (rápido):** "A manutenção é a mais paranoica: fotografa antes de mexer, mexe o mínimo, registra
  tudo. Disciplina de mexer numa máquina ligada com checklist."

### Slide 16 — Lote piloto e engenharia da qualidade ⏩
- **TELA:** testes de fluxo real (três provas) · documentar · inventário · validar-instructions.
- **FALA (rápido):** "O lote piloto prova na vida real — acaba com o 'passou na minha bancada'. E a
  engenharia da qualidade mantém o conhecimento fiel ao código."

---

## ATO 3 — A Ordem de Produção e os Dispositivos (≈10 min)

### Slide 17 — Skill: a ordem que aciona a estação
- **TELA:** "A skill aciona a estação e sai da frente."
- **FALA:** "A skill é o **botão**; o agente é o **cérebro**. Mudou 'quando acionar'? Toca a skill. Mudou
  'como executar'? Toca o agente. Cada motivo de mudança tem um lugar só. SRP aplicado à config."

### Slide 18 — A engenharia do "Use when:"
- **TELA:** auto-disparo · desambiguação · exclusão.
- **FALA:** "O `description` é contrato semântico. Numa fábrica de 22 postos, saber **qual acionar** é
  metade da qualidade. Ele até diz 'para isso, prefira a outra ordem'."

### Slide 19 — Por que NÃO usamos commands
- **TELA:** tabela command × skill × agent.
- **FALA:** "As skills já fazem o papel do command, melhor: auto-disparo e estação isolada. Command seria
  um **terceiro mecanismo paralelo** — exatamente a duplicação que a Norma proíbe."

### Slide 20 — O conceito-chave: poka-yoke vs. sensor
- **TELA:** tabela poka-yoke/Andon (bloqueia) × sensor/alarme (avisa).
- **FALA:** "Poka-yoke é o dispositivo à prova de erro: o conector só entra de um jeito. Andon é o cordão
  que para a linha. Sensor é a luz amarela: avisa, mas você decide. A engenharia é escolher o instrumento
  certo para cada risco."
- **INTERAÇÃO:** "Qual a diferença entre uma regra que a gente 'pede' e uma que o dispositivo 'garante'?"

### Slide 21 — Os guards (poka-yoke / Andon)
- **TELA:** bash-guard (varredura de logs, rm -rf, drop) · write-guard. "Falha aberto."
- **FALA:** "Dois acidentes clássicos: travar a sessão com listagem gigante e destruir dados. O poka-yoke
  torna os dois **impossíveis** — não depende de a IA 'lembrar'. E falha aberto: na dúvida do próprio
  funcionamento, não trava a produção indevidamente."

### Slide 22 — Os sensores (alarmes)
- **TELA:** logging-nudge · py-lint · py-discipline.
- **FALA:** "Transformamos a especificação densa de logs num **radar automático** que corrige no momento
  mais barato: a hora da edição. Por que alarme e não trava? Porque pede julgamento — um print() pode ser
  legítimo num script de sandbox."

### Slide 23 — O Kaizen que se fecha sozinho
- **TELA:** session-start (lê lições) ↔ stop-loop (cobra registro).
- **FALA:** "Começo do turno: **lembra** das lições. Fim do turno: **cobra** o registro de novas. A
  melhoria contínua acontece automaticamente, sem depender de alguém lembrar de documentar."

---

## ATO 4 — Um Lote na Linha (≈8 min)

### Slide 24 — Vamos ver a fábrica produzindo
- **TELA:** *"A ingestão de PDFs da DNIT termina com erro intermitente. Conserta."*
- **FALA:** "Pedido real, do tipo que chega todo dia. Vou marcar qual estação opera, qual especificação
  consulta e qual dispositivo está de olho."

### Slide 25 — A produção, estação por estação
- **TELA:** briefing → metrologia → inspeção → PCP → fabricação → lote piloto → QA → registro.
- **FALA:** "A metrologia lê o log e acha a etapa do defeito. A inspeção mede o código real com evidência.
  O PCP emite o roteiro e mapeia os ensaios. A fabricação conserta, e os sensores alertam se ela esquecer
  um log. O lote piloto prova na ingestão real, com três provas. O QA só libera se a peça estiver ativa no
  runtime."

### Slide 26 — O lote pode voltar
- **TELA:** REPROVADO → volta ao PCP; APROVADO → stop-loop cobra o Kaizen.
- **FALA:** "A linha **não é de mão única**. Devolver **com inventário** — exatamente o que faltou — é o
  que impede o falso-verde de chegar ao cliente."

### Slide 27 — O placar: defeitos neutralizados
- **TELA:** tabela defeito → quem neutraliza.
- **FALA:** "Cada problema clássico de engenharia tem um dono. Não é sorte — é cobertura intencional."

---

## ATO 5 — Os Dois Motores e a Aposta (≈10 min)

### Slide 28 — Os dois motores invisíveis
- **TELA:** Jidoka+Andon (corrige este lote) × Kaizen (melhora os próximos).
- **FALA:** "Jidoka é 'automação com toque humano': a máquina **para sozinha** ao detectar anomalia, em vez
  de produzir refugo. Kaizen é a melhoria entre turnos. Um garante o lote de **hoje**; o outro que o de
  **amanhã** começa melhor. E há uma ponte: a lição de um conserto vira prevenção de uma família de
  defeitos."

### Slide 29 — Por que funciona num código gigante ⏩
- **TELA:** âncora · ler só o necessário · provar por fatia · sem varredura cega.
- **FALA (rápido):** "São as travas de escala. É o que separa 'funciona no exemplo' de 'funciona no sistema
  real'."

### Slide 30 — A virada de papel: de operário a gerente de produção
- **TELA:** tabela artesanal × orquestração (digitar→ordenar; revisar diff→ler evidência).
- **FALA:** "O humano sobe do nível da linha de código para o nível da **evidência e da decisão**. Não
  confere se a variável foi nomeada certo — disso cuidam os SOPs e os dispositivos. Confere se a inspeção
  tem evidência, se o roteiro é seguro, se a validação provou."

### Slide 30-bis — A ordem de serviço bem especificada
- **TELA:** "Emita uma OS clara: **Objetivo · Escopo · Restrições · Evidência · Critério de aceite**. Sem evidência (4), não se abre OS — pede-se inspeção."
- **FALA:** "Se o humano é o gerente de produção, a OS é a ferramenta dele. Ordem vaga dá lote vago; ordem
  com essas cinco partes vira trabalho auditável. Regra de ouro: sem evidência, não se abre ordem de
  fabricação — manda-se inspecionar a matéria-prima primeiro. Abrir OS sem evidência é convidar o refugo."
- **INTERAÇÃO:** "Qual dessas cinco partes mais falta nas ordens reais? *(quase sempre o critério de aceite)*"
- **APROFUNDAMENTO:** [../base-conceitual/operar-a-metodologia.md](../base-conceitual/operar-a-metodologia.md)

### Slide 31 — A afirmação ousada: a inspeção final fica redundante
- **TELA:** design review no PCP · estilo nos sensores · reuso na fabricação · veredito no QA.
- **FALA:** "O code review não acabou — foi **desmontado em dezenas de verificações contínuas**,
  distribuídas pelo processo e fechadas por um inspetor independente. O que era gargalo no fim virou malha
  de garantias do começo ao fim. Não é menos revisão — é mais, contínua e sem fadiga."

### Slide 32 — A leitura honesta (e onde mora a garantia)
- **TELA:** o humano sobe de altitude · garantia = qualidade dos contratos · decisões novas pedem humano.
- **FALA:** "Sejamos honestos: 'sem digitar código' não é 'sem engenharia'. O esforço migra da digitação
  para a **curadoria dos contratos** e a **leitura crítica da evidência**. É mais alavancado, não
  inexistente. A garantia mora no **conteúdo dos contratos**, não no agente."

---

## ATO 6 — Fechamento (≈5 min)

### Slide 33 — A tese, agora completa
- **TELA:** "Uma linha de produção onde a IA opera como fábrica madura com QA — e melhora sozinha a cada turno."
- **FALA:** "O valor não está em nenhum arquivo isolado. Está no **processo** — em como eles operam juntos.
  Produção artesanal entrega uma peça boa de vez em quando. Uma fábrica enxuta entrega qualidade
  previsível, repetível e auditável, lote após lote."

### Slide 34 — O que levar pra casa
- **TELA:** segregação de funções · rastreabilidade · anti-falso-verde · poka-yoke vs. alarme · Jidoka+Kaizen.
- **FALA:** "Se levarem cinco ideias, levem essas. Valem para qualquer sistema de software, com ou sem IA."

### Slide 35 — Encerramento + perguntas
- **TELA:** "Obrigado. Perguntas?"
- **INTERAÇÃO:** reabra a pergunta do início e compare com as respostas iniciais.
- **FALA:** "Tudo está documentado: capítulos, base conceitual, FAQ de objeções e diagramas. Obrigado."
- **RESERVA:** use a [FAQ de Objeções](../base-conceitual/faq-objecoes.md) para as perguntas difíceis.

---

## Apêndice — Cola do apresentador (one-liners de bolso)

- **Falso-verde:** "parece pronto, mas a linha real não usa a peça nova."
- **Raio-X:** "o log permite reconstruir a produção depois que ela aconteceu."
- **Poka-yoke vs. sensor:** "um torna o erro impossível; o outro só acende a luz amarela."
- **Jidoka/Andon:** "a máquina para sozinha ao detectar defeito, em vez de produzir refugo."
- **Kaizen:** "cada defeito vira um ajuste de procedimento para o próximo turno."
- **Segregação de funções:** "quem fabrica não libera a própria peça."
- **"100%":** "qualifica a execução do lote, não autoriza inventar encomenda nova."

---

**Material de apoio:** [deck Marp](apresentacao-marp.md) ·
[diagramas](../base-conceitual/diagramas.md) · [FAQ de objeções](../base-conceitual/faq-objecoes.md) ·
[cartão de referência](../base-conceitual/cartao-de-referencia.md) · [↩ doc-mãe](README.md)
