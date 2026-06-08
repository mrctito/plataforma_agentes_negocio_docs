# Manual detalhado da etapa: Fachada de pergunta do pipeline RAG

## 1. O que esta etapa faz

Esta etapa e a porta de entrada operacional da pergunta no runtime online do RAG. Ela nao decide estrategia de retrieval, nao escolhe router e nao gera a resposta final. A responsabilidade dela e outra: receber a requisicao de consulta, validar o que chegou, preparar anexos multimodais de forma segura, resolver timeout, montar ou reaproveitar o runtime de QA e entregar a pergunta ao pipeline interno com telemetria consistente.

Em linguagem simples: e a recepcao tecnicamente disciplinada do RAG. Se essa camada falhar, o restante do pipeline herda ruido, timeout mal definido, imagem malformada ou diagnostico pobre.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa esta centralizada em QuestionService.execute. Ela vem antes de qualquer decisao de query rewrite, roteamento, retrieval ou geracao. O papel dela e transformar uma entrada de API ou CLI em uma chamada segura ao ContentQASystem.

## 3. O que entra e o que sai

Entradas confirmadas no codigo:

- question
- output_format
- access_control
- metadata_filters
- image_base64

Saida confirmada no codigo:

- payload estruturado com answer, metrics, metadata, logs, telemetry, analysis, confidence, success, duration, access_control, include_sources, sources e source_documents

Esse formato importa porque mostra que a fachada ja prepara nao apenas a resposta final, mas tambem os blocos de diagnostico e observabilidade que permitem explicar o comportamento do pipeline.

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. Decodifica image_base64 com validacao defensiva em uma rotina dedicada de decode base64.
2. Remove prefixo data URI quando a imagem chega como data URL.
3. Usa base64.b64decode com validate=True para evitar aceitar lixo silenciosamente.
4. Se a imagem estiver invalida, registra warning e ignora o anexo em vez de contaminar o restante do pipeline.
5. Registra log de inicio com tamanho da pergunta, formato de saida e presenca de imagem.
6. Inicializa o sistema de QA pela etapa seguinte, que pode reaproveitar cache global.
7. Resolve timeout via InvokeTimeoutGuard.resolve_timeout.
8. Executa qa_system.ask_question sob ainvoke com timeout guardado.
9. Analisa a qualidade da resposta com AnswerQualityAnalyzer.
10. Extrai retrieval_metrics com PipelineDiagnosticsBuilder para logging de fechamento.
11. Enriquece fontes e documentos com SourceEnricher.
12. Devolve um payload final coeso para quem chamou o servico.

Essa ordem mostra um ponto arquitetural importante: o timeout e tratado antes da resposta sair, e a coleta de diagnostico acontece depois que o pipeline interno termina, nao no meio do contrato publico.

## 5. Decisoes tecnicas importantes

### 5.1. Imagem invalida nao derruba a pergunta inteira

O codigo nao trata base64 invalido como motivo para abortar a consulta. Ele registra o problema e segue sem imagem. Isso reduz fragilidade na borda publica quando o usuario envia payload multimodal malformado.

O ganho pratico e claro: um anexo ruim nao precisa desperdiçar uma pergunta textual valida.

O custo tambem e claro: a resposta pode sair sem o sinal visual que o chamador imaginava ter enviado. Por isso o log de imagem ignorada e parte essencial do diagnostico.

### 5.2. Timeout e resolvido na borda do caso de uso

QuestionService nao deixa o timeout implodir espalhado em componentes internos. Ele resolve primeiro o limite aplicavel e depois executa qa_system.ask_question sob InvokeTimeoutGuard. Isso protege o contrato publico da consulta e simplifica observabilidade de latencia.

### 5.3. A fachada nao mistura retrieval com apresentacao final

Mesmo devolvendo fontes, analise e diagnosticos, ela nao implementa retrieval nem generation. Ela orquestra e normaliza. Isso preserva separacao de responsabilidade.

## 6. O que pode dar errado

Falhas confirmadas ou implicadas diretamente no codigo lido:

- image_base64 invalida: a imagem e descartada e o log registra o motivo.
- falha durante qa_system.ask_question: a excecao e registrada, a telemetria marca erro e a excecao sobe.
- timeout: o guard de invocacao e quem delimita o tempo maximo da consulta.

Em termos praticos, esta etapa nao mascara erro interno do pipeline. Ela registra e propaga.

## 7. Como diagnosticar

Os sinais mais uteis confirmados no codigo sao estes.

- Log de inicio da consulta com question_length, output_format, image_present e image_size_bytes.
- Log de imagem ignorada quando o base64 e invalido.
- Log de conclusao com success, duration, answer_length, sources e retrieval_metrics.
- QuestionTelemetryRecorder para registrar duracao, sucesso, erro e token_usage.
- AnswerQualityAnalyzer para classificar o estado do resultado.

Em linguagem simples: se a pergunta falhou logo no comeco, e aqui que a historia operacional comeca.

## 8. Exemplo pratico guiado

Cenario: um endpoint envia uma pergunta textual com um print de tela em base64.

1. A fachada recebe question e image_base64.
2. Se a imagem vier como data URI, o prefixo e removido.
3. Se o binario for valido, a imagem segue como image_bytes.
4. O runtime de QA e resolvido.
5. O timeout da consulta e calculado.
6. O pipeline interno responde.
7. A fachada monta answer, sources, analysis, telemetry e duration.

Se o base64 vier quebrado, a pergunta ainda pode ser respondida, mas o diagnostico vai mostrar que o anexo foi ignorado.

## 9. Evidencias no codigo

- src/services/question_service.py
  - Simbolo relevante: QuestionService.execute
  - Comportamento confirmado: log de entrada, decode defensivo da imagem, timeout guardado, chamada ao qa_system, analise de qualidade, enriquecimento de fontes e payload final.
- src/services/question_service.py
  - Simbolo relevante: QuestionService._decode_image_base64
  - Comportamento confirmado: remocao de data URI, base64 com validate=True e descarte seguro de imagem invalida.
