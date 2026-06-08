# Manual detalhado da etapa: Governanca do modo moderno do pipeline RAG

## 1. O que esta etapa faz

Esta etapa protege o contrato arquitetural do RAG moderno. Ela impede que o sistema diga que esta operando em pipeline inteligente enquanto, na pratica, esta sem o orquestrador obrigatorio ou com configuracao no lugar errado.

Em linguagem simples: e a etapa que garante honestidade operacional. Ou o modo moderno existe de verdade, ou a consulta falha cedo.

## 2. Problema que ela resolve

Um sistema RAG pode parecer moderno no papel e ainda operar com fallback silencioso no runtime. Isso piora diagnostico, mascara regressao e torna imprevisivel a qualidade da recuperacao. O codigo lido combate exatamente esse risco.

## 3. Controles confirmados no codigo

### 3.1. intelligent_pipeline precisa estar na raiz correta

ContentQASystem._validate_intelligent_pipeline_layout rejeita o caso em que intelligent_pipeline foi colocado dentro de memory. O erro e explicito e orienta a configuracao correta.

O impacto pratico e evitar YAML semanticamente confuso, em que um controle de runtime moderno aparece enterrado em uma secao que nao deveria governar isso.

### 3.2. O runtime state e forçado para modern

QARuntimeState.ensure_modern normaliza o estado efetivo para mode=modern e registra justificativa textual. Se a factory inteligente foi pulada apenas por motivo interno, a justificativa muda, mas o runtime continua declarado explicitamente.

### 3.3. O pipeline moderno indisponivel falha fechado

QAQuestionProcessor.ask_question contem a regra central.

Se:

- is_modern_mode for verdadeiro,
- is_skip_intelligent_factory for falso,
- intelligent_orchestrator estiver ausente,

entao a consulta registra telemetria de erro e levanta ContentQAError com a mensagem de pipeline moderno indisponivel.

Esse ponto e o coracao da governanca do modo moderno. Ele elimina o comportamento perigoso de continuar com um caminho antigo apenas para parecer resiliente.

## 4. Decisoes tecnicas e trade-offs

### 4.1. Fail-fast aqui e decisao de confiabilidade

Falhar cedo pode parecer menos amigavel do que tentar um fallback. Mas, no contexto deste projeto, seguir sem o orchestrator moderno geraria um efeito pior: respostas aparentemente validas, porem construidas por um caminho diferente do prometido.

Na pratica, o codigo escolhe previsibilidade e rastreabilidade em vez de conforto artificial.

### 4.2. skip_intelligent_factory existe para evitar recursao interna, nao para afrouxar o contrato publico

O codigo diferencia claramente o caso de montagem interna, em que o skip pode ser necessario, do caso de execucao normal do pipeline. Isso importa porque impede que uma excecao tecnica de bootstrap vire permissao generica para burlar o modo moderno.

## 5. O que entra e o que sai

Entradas relevantes:

- yaml_config
- runtime_state
- intelligent_orchestrator
- flags de modern mode e skip_intelligent_factory

Saidas relevantes:

- consulta autorizada a seguir no modo moderno
- ou ContentQAError explicito impedindo a execucao incoerente

## 6. O que pode dar errado

Falhas confirmadas:

- intelligent_pipeline em local invalido do YAML
- pipeline moderno indisponivel quando deveria ser obrigatorio
- pergunta vazia, que e barrada antes de entrar no restante do fluxo

O que o codigo evita deliberadamente:

- fallback silencioso para um caminho alternativo quando o modo moderno foi prometido

## 7. Como diagnosticar

Os sinais mais importantes estao nestes logs e payloads.

- log RAG inteligente | usado=sim ou nao | motivo=...
- marker PIPELINE_ROUTING_DECISION com modern_mode e skip_intelligent_factory
- telemetria qa_pipeline_failed quando o pipeline moderno deveria existir e nao existe
- justificativa do QARuntimeState no resumo consolidado de pipeline

Em linguagem simples: essa etapa diz se o sistema esta realmente no modo moderno ou apenas tentando parecer que esta.

## 8. Exemplo pratico guiado

Cenario: um YAML habilita o runtime moderno, mas o orchestrator inteligente nao foi montado.

1. QAQuestionProcessor recebe a pergunta.
2. Ele calcula modern_mode=true e has_intelligent=false.
3. Registra a razao intelligent_orchestrator_ausente.
4. Grava telemetria qa_pipeline_failed.
5. Levanta ContentQAError.

O efeito pratico e impedir que a pergunta seja respondida por um caminho diferente do contrato esperado.

## 9. Evidencias no codigo

- src/qa_layer/content_qa_system.py
  - Simbolo relevante: ContentQASystem._validate_intelligent_pipeline_layout
  - Comportamento confirmado: rejeita intelligent_pipeline em memory.
- src/qa_layer/qa_runtime_state.py
  - Simbolo relevante: QARuntimeState.ensure_modern
  - Comportamento confirmado: normaliza o runtime para modo moderno e registra justificativa.
- src/qa_layer/qa_question_processor.py
  - Simbolo relevante: QAQuestionProcessor.ask_question
  - Comportamento confirmado: falha fechada quando o modo moderno exige orchestrator e ele esta ausente.
