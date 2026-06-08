# Manual detalhado da etapa: Bootstrap do runtime PDF

## 1. O que esta etapa faz

Esta etapa prepara o mundo operacional em que cada PDF sera processado. Ela nao extrai texto, nao cria chunks e nao indexa nada. A responsabilidade dela e outra: resolver a configuracao valida, inicializar servicos e construir os dois pipelines internos que o restante do slice PDF vai usar.

Em linguagem simples: e a montagem da oficina antes de o documento entrar na esteira.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa esta centralizada em PdfRuntimeCoordinator.initialize_runtime. Ela roda antes da extracao documental principal e antes do fluxo rico. O processor PDF passa a ter, depois desse passo, os servicos de OCR, tabelas, metadata, multimodalidade, chunking e pipelines textuais prontos para uso.

## 3. O que entra e o que sai

Entrada confirmada:

- configuracao YAML completa do runtime PDF
- host do processor PDF com metodos de inicializacao e factories de servico

Saida confirmada:

- ContentType.PDF registrado como tipo suportado
- configuracoes resolvidas por secao
- runtime de chunking inicializado
- runtime de OCR por pagina e OCR document-level inicializado
- runtime de tabelas, qualidade, metadata, imagens e anexos inicializado
- servicos de suporte ao parsing construidos
- pipeline de extracao criado sob demanda
- pipeline de limpeza textual criado sob demanda
- runtime multimodal inicializado
- overrides de dominio aplicados sobre a configuracao PDF quando permitidos

## 4. Como o codigo implementa a etapa

O fluxo real observado em PdfRuntimeCoordinator.initialize_runtime segue esta ordem.

1. fixa o processor como suportando apenas ContentType.PDF;
2. configura o logging auxiliar de pdfminer para reduzir ruido e padronizar observabilidade;
3. resolve um snapshot canonicamente organizado das configuracoes PDF;
4. inicializa parametros de chunking;
5. inicializa OCR por pagina e OCR document-level;
6. inicializa runtime de tabelas;
7. inicializa filtros de qualidade e classificacao;
8. inicializa metadata, imagens e anexos;
9. monta os servicos de suporte ao parsing;
10. inicializa pipelines de extracao e de texto, alem do runtime multimodal;
11. finaliza o runtime de dominio aplicado ao PDF.

O ponto importante e que o coordinator nao implementa toda a logica sozinho. Ele orquestra chamadas do host e centraliza a ordem correta. Isso reduz acoplamento e permite trocar componentes do runtime sem reescrever o fluxo principal.

## 5. Decisoes tecnicas importantes

### 5.1. Resolver tudo antes de tocar no documento

O desenho escolhe pagar o custo de inicializacao antes do primeiro PDF. O ganho pratico e previsibilidade: quando um documento entra na esteira, o runtime ja sabe quais engines, pipelines e flags estao disponiveis.

O custo e obvio: configuracao ruim falha cedo. Mas essa e exatamente a intencao do slice. Melhor falhar no bootstrap do que permitir um runtime ambigoo e so descobrir o problema depois de metade do documento processado.

### 5.2. Separar extracao e limpeza textual em pipelines explicitos

O coordinator nao monta um unico pipeline monolitico. Ele cria um pipeline de extracao e outro de texto. Isso importa porque diagnostico de PDF raramente e uma pergunta unica. O operador precisa saber se o problema aconteceu na leitura dos bytes, no parsing ou na limpeza do texto.

### 5.3. Aplicar domain processing ja no bootstrap quando houver pdf_overrides

O codigo mostra um detalhe importante: dominio nao serve apenas para enriquecer chunks no fim. O processor inicializa DomainProcessingResolver e, se um dominio ativo trouxer pdf_overrides com apply_globally=true, esses overrides entram no runtime antes da extracao.

Na pratica, isso significa que o dominio pode alterar comportamento do parser PDF de forma governada, sem hardcode no core.

## 6. O que pode dar errado

Os riscos confirmados ou diretamente implicados por esse desenho sao estes.

- contrato YAML legado ou proibido faz o runtime falhar cedo;
- dependencia opcional de engine pode estar indisponivel mais adiante, dependendo do mode configurado;
- inicializacao do runtime de OCR, tabelas ou multimodalidade pode falhar se a configuracao estiver inconsistente;
- pdf_overrides de dominio mal formulados podem distorcer o runtime antes da extracao.

Em termos operacionais, se o bootstrap falha, o resto do pipeline nem deveria comecar.

## 7. Como diagnosticar

Os sinais mais uteis para esta etapa sao:

- logs de inicializacao do processor PDF;
- evidencias de resolucao de contrato canonicamente em ingestion.content_profiles.type_specific.pdf;
- logs de domain processing PDF habilitado ou desabilitado;
- logs de overrides PDF aplicados por dominio;
- ausencia de pipelines internos ou falha imediata antes da extracao.

Em linguagem simples: se o PDF nao chega nem a tentar extrair texto, a investigacao comeca aqui.

## 8. Exemplo pratico guiado

Cenario: o tenant habilita duas engines de parsing, OCR document-level e um dominio com pdf_overrides globais.

1. O processor PDF inicia.
2. O coordinator resolve o snapshot do contrato PDF.
3. Os parametros de chunking e OCR sao carregados.
4. Os servicos de parsing e os pipelines internos sao montados.
5. DomainProcessingResolver identifica o dominio ativo.
6. Os pdf_overrides desse dominio sao mesclados na configuracao efetiva.
7. O runtime final fica pronto para receber o primeiro documento.

O valor pratico desta etapa e impedir que o documento vire o lugar onde a configuracao e improvisada.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/pdf_runtime_coordinator.py
  - Simbolo relevante: PdfRuntimeCoordinator.initialize_runtime
  - Comportamento confirmado: ordem de bootstrap, criacao dos pipelines de extracao e texto e finalizacao do runtime PDF.
- src/ingestion_layer/processors/pdf_processor.py
  - Simbolo relevante: os metodos de setup de dominio e de aplicacao de overrides do PDF
  - Comportamento confirmado: resolucao de domain processing no slice PDF e merge de pdf_overrides por dominio ativo.
