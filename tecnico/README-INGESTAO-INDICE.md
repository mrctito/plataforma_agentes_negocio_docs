# Índice dos pipelines de ingestão

Este documento é o ponto de entrada para os manuais de ingestão da plataforma. Ele organiza os READMEs de PDF, Excel, HTML, JSON, Confluence e Web Scraping que já existem em `docs/`.

Use este índice quando a dúvida for: qual manual devo ler para entender como um tipo de conteúdo vira acervo consultável, auditável e pronto para RAG?

Antes de mergulhar no detalhe de um formato específico, agora também existe um dossiê separado sobre a infraestrutura assíncrona que carrega os jobs até o worker e sustenta o paralelismo por documento. Isso é importante porque, no produto real, um PDF não nasce direto no processor especializado; ele passa primeiro pelo mecanismo genérico de jobs, worker, fila e fan-out.

## Visão geral

O pipeline de ingestão é avançado porque ele não trata conteúdo como simples upload seguido de embedding. Em linguagem simples, embedding é a transformação do conteúdo em uma representação numérica usada para busca semântica. Se a plataforma apenas fizesse isso, qualquer erro na leitura do arquivo entraria no acervo e contaminaria as respostas depois.

Se a sua dúvida for especificamente sobre o modelo de dados do acervo, sobre a troca do modelo atual para um modelo novo mais simples e sobre qual tabela virou qual, use também este documento complementar:

- README-TECNICO-DE-PARA-MODELO-DADOS-INGESTAO-VETORIAL.md

Se a sua dúvida já não for mais sobre o desenho do modelo, mas sim sobre a ordem completa de implementação, corte, validação e remoção do legado, use também este plano técnico complementar:

- README-TECNICO-PLANO-IMPLEMENTACAO-MODELO-DADOS-INGESTAO-VETORIAL.md

O desenho atual é mais rico. A ingestão separa aquisição, identificação do tipo de conteúdo, materialização do documento, extração especializada, limpeza, enriquecimento, chunking, indexação e persistência. Chunking significa quebrar um documento em pedaços menores e úteis para busca. Persistência significa gravar o resultado de forma rastreável para consulta, auditoria e reprocessamento.

A arquitetura também é multimodal quando o tipo de conteúdo justifica isso. Multimodal quer dizer que o pipeline pode usar mais de uma forma de evidência, como texto, tabela, imagem, anexo, HTML e metadados operacionais. Isso aparece de forma mais forte em PDF, Web Scraping e Confluence, onde imagens, páginas, anexos e estrutura visual podem carregar informação relevante.

Outro ponto importante é que a ingestão não é um bloco único. O código confirma processors especializados para PDF, Excel, JSON, HTML, Web e Confluence, além de uma fábrica comum que registra esses processors e uma etapa de indexação que envia chunks ao vector store. Vector store é o banco ou índice usado para guardar os chunks de forma pesquisável por semântica.

Na prática, isso torna o pipeline avançado por seis motivos.

- Ele entende que formatos diferentes exigem tratamento diferente.
- Ele preserva estrutura quando ela importa, como páginas de PDF, abas de Excel, chaves de JSON e árvore HTML.
- Ele evita misturar aquisição remota com interpretação do documento.
- Ele permite enriquecimento multimodal onde isso agrega valor real.
- Ele conecta cada slice especializado à mesma esteira comum de indexação e persistência.
- Ele deixa o diagnóstico mais claro, porque cada manual técnico mostra o caminho, os contratos e os pontos de falha de cada tipo de ingestão.

## Como escolher o manual certo

Se você quer entender o valor de negócio, a visão executiva, os conceitos e exemplos de uso, comece pelo manual conceitual.

Se você precisa implementar, revisar YAML, diagnosticar erro, confirmar caminho de runtime ou entender classes e contratos, leia o manual técnico correspondente.

## Índice por tipo de ingestão

### Fundamentos do worker, jobs e paralelismo

- [README-CONCEITUAL-SISTEMA-JOBS-WORKER-PARALELISMO.md](../conceitual/README-CONCEITUAL-SISTEMA-JOBS-WORKER-PARALELISMO.md)
- [README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md](README-TECNICO-SISTEMA-JOBS-WORKER-PARALELISMO.md)

Leia estes dois manuais antes dos documentos de PDF quando a dúvida envolver `POST /rag/ingest`, job pai, fila, worker dedicado, fan-out documental, job filho, contrato de paralelismo ou extensão do mecanismo para novos tipos de job. Eles explicam primeiro a espinha dorsal agnóstica e só depois a especialização da ingestão.

### PDF

- [README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO.md](../conceitual/README-CONCEITUAL-INGESTAO-PDF-PIPELINE-COMPLETO.md)
- [README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-PDF-PIPELINE-COMPLETO.md)

O pipeline de PDF é o mais rico em decisões documentais. Ele trata PDF digital, PDF escaneado, texto nativo, OCR, parsing por engines, tabelas, imagens, multimodalidade, chunking e manifesto operacional. Use estes documentos quando o problema envolver qualidade de extração, OCR, parsing, páginas, tabelas, imagens ou diagnóstico de PDF difícil.

Nesta atualização, os manuais de PDF também passaram a cobrir a fronteira assíncrona completa do produto: `POST /rag/ingest`, job pai, worker pai, fan-out documental, worker filho e o momento em que o slice PDF especializado realmente entra em ação.

### Excel

- [README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO.md](../conceitual/README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO.md)
- [README-TECNICO-INGESTAO-EXCEL-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-EXCEL-PIPELINE-COMPLETO.md)

O pipeline de Excel preserva a natureza tabular do arquivo. Ele diferencia workbook, planilhas, linhas, colunas, tipos, tabelas nativas, tabelas heurísticas e chunks orientados por linha. Use estes documentos quando o conteúdo for `.xlsx` ou `.xls` e a pergunta depender de estrutura de planilha, não apenas de texto bruto.

### JSON

- [README-CONCEITUAL-INGESTAO-JSON-PIPELINE-COMPLETO.md](../conceitual/README-CONCEITUAL-INGESTAO-JSON-PIPELINE-COMPLETO.md)
- [README-TECNICO-INGESTAO-JSON-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-JSON-PIPELINE-COMPLETO.md)

O pipeline de JSON trata dado semi-estruturado como estrutura, não como texto solto. Ele valida bytes, encoding e parse, cria resumo de schema, calcula estatísticas, reconhece alguns sinais de domínio e faz chunking estrutural. Use estes documentos para catálogos, cupons, dumps de API, schema metadata e payloads `.json`.

### HTML

- [README-CONCEITUAL-INGESTAO-HTML-PIPELINE-COMPLETO.md](../conceitual/README-CONCEITUAL-INGESTAO-HTML-PIPELINE-COMPLETO.md)
- [README-TECNICO-INGESTAO-HTML-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-HTML-PIPELINE-COMPLETO.md)

O pipeline de HTML cuida da interpretação do conteúdo HTML. Ele remove ruído básico de marcação, como scripts e estilos, normaliza texto e prepara o conteúdo para chunking. Use estes documentos quando a dúvida for sobre HTML como formato de documento. Para captura de páginas remotas, use a seção de Web Scraping.

### Confluence

- [README-CONCEITUAL-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md](../conceitual/README-CONCEITUAL-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md)
- [README-TECNICO-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-CONFLUENCE-PIPELINE-COMPLETO.md)

O pipeline de Confluence transforma páginas de wiki corporativa em acervo consultável. Ele trabalha com páginas seed, recursão por filhos, visibilidade, autorização, anexos, qualidade, materialização do conteúdo e enriquecimento multimodal. Use estes documentos quando a fonte de conhecimento estiver no Confluence e houver necessidade de governança de escopo e rastreabilidade.

### Web Scraping

- [README-CONCEITUAL-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md](../conceitual/README-CONCEITUAL-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md)
- [README-TECNICO-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md](README-TECNICO-INGESTAO-WEBSCRAP-PIPELINE-COMPLETO.md)

O pipeline de Web Scraping cuida da aquisição remota de páginas por URL. Ele lida com seeds explícitas, estratégia de captura, autenticação, rate limiting, proxy, cache, deduplicação, anexos, documento prefetched e reentrada na esteira comum. Use estes documentos quando a origem for uma URL ou página web, não apenas um arquivo HTML já disponível.

## Relação com ETL e RAG

Ingestão não é a mesma coisa que ETL. ETL é a esteira para extrair, transformar e carregar dados estruturados com contrato próprio. Ingestão documental é a esteira que transforma documentos e conteúdos semi-estruturados em acervo pesquisável.

Ingestão também não é a mesma coisa que RAG. RAG é a etapa que recupera informações do acervo para apoiar uma resposta. A ingestão vem antes: ela decide se o acervo nasceu limpo, estruturado, rastreável e útil. Se a ingestão for ruim, o RAG trabalha sobre conteúdo ruim.

Para continuar a leitura, use também:

- [README-CONCEITUAL-ETL-COMPLETO.md](../conceitual/README-CONCEITUAL-ETL-COMPLETO.md)
- [README-TECNICO-ETL-COMPLETO.md](README-TECNICO-ETL-COMPLETO.md)
- [README-CONCEITUAL-RAG-PIPELINE-COMPLETO.md](../conceitual/README-CONCEITUAL-RAG-PIPELINE-COMPLETO.md)
- [README-TECNICO-RAG-PIPELINE-COMPLETO.md](README-TECNICO-RAG-PIPELINE-COMPLETO.md)

## Leitura recomendada

Para uma primeira visão completa, siga esta ordem:

1. Leia este índice.
2. Leia o manual conceitual do tipo de ingestão que você quer entender.
3. Leia o manual técnico do mesmo tipo.
4. Consulte o índice central em [README-INDICE.MD](../README.md) para navegar para ETL, RAG, YAML, segurança e arquitetura.

## Regra prática

Quando houver dúvida entre dois documentos, escolha pelo tipo de fonte real.

- Arquivo PDF: use PDF.
- Planilha `.xlsx` ou `.xls`: use Excel.
- Arquivo `.json`: use JSON.
- HTML já materializado como documento: use HTML.
- Página corporativa no Confluence: use Confluence.
- URL remota ou site: use Web Scraping.

Essa separação evita misturar responsabilidades. O ganho prático é diagnóstico mais rápido, configuração mais clara e menos risco de tratar formatos diferentes como se fossem o mesmo problema.
