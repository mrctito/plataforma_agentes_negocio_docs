# Manual técnico, executivo, comercial e estratégico: Pipeline de Ingestão de Excel

## 1. O que é esta feature

O pipeline de ingestão de Excel é a capacidade da plataforma de transformar planilhas `.xlsx` e `.xls` em acervo consultável sem perder a semântica tabular mais importante do arquivo. Ele não existe só para converter células em texto. Ele existe para decidir quais planilhas devem entrar, como detectar cabeçalhos, como reconhecer tabelas nativas e heurísticas, como preservar tipos, como montar metadados de schema e como quebrar a planilha em chunks úteis para busca e análise posterior.

No código real, o Excel não é tratado como um documento linear. Ele é tratado como um workbook com planilhas, linhas, colunas, densidade de dados, tabelas e papéis analíticos. Isso muda o valor do pipeline. O objetivo não é apenas indexar texto. O objetivo é manter informação suficiente para que a planilha continue parecendo planilha depois da ingestão.

## 2. Que problema ela resolve

Planilhas parecem simples, mas são uma fonte clássica de perda semântica quando entram em pipelines de ingestão genéricos.

- Uma planilha pode ter várias abas com papéis diferentes.
- Uma aba pode ser uma tabela limpa, um formulário, um misto ou uma folha esparsa.
- O cabeçalho pode estar claro ou pode precisar de inferência.
- O arquivo pode ter estrutura nativa de tabela do workbook ou apenas uma grade implícita.
- A pergunta futura pode depender não do texto inteiro, mas de uma linha específica, de um campo numérico ou de uma coluna com papel de dimensão, tempo ou métrica.

Sem uma esteira especializada, a plataforma sofreria com quatro perdas diretas.

- Perda de contexto tabular, porque cada linha viraria texto desestruturado.
- Perda de governança, porque o sistema não saberia distinguir uma aba útil de uma aba de configuração ou metadata.
- Perda de qualidade analítica, porque tipos numéricos, datas e papéis de coluna seriam achatados cedo demais.
- Perda de valor downstream, porque mecanismos especializados de consulta em Excel perderiam a base estruturada de que precisam.

## 3. Visão executiva

Para liderança, o pipeline de Excel importa porque planilha é um dos formatos mais comuns de conhecimento operacional. Vendas, estoque, financeiro, cadastro, auditoria, indicadores e controle de projeto costumam nascer ou circular em workbook.

Se a ingestão desse formato for rasa, o produto até consegue “ler” a planilha, mas não consegue transformá-la em ativo confiável. Isso impacta diretamente a percepção de valor do cliente, a qualidade de consulta e a credibilidade do acervo.

O código mostra que a plataforma tenta reduzir esse risco com um desenho orientado por estrutura.

- Valida o formato aceito antes de processar.
- Separa caminho moderno `.xlsx` e legado `.xls`.
- Analisa a estrutura de cada aba.
- Preserva linhas, colunas, tipos e papéis analíticos.
- Gera chunks com `row_data`, `column_types` e `column_roles`.

Em termos simples: o sistema não trata Excel como texto grande com quebras de linha. Ele tenta preservar a natureza tabular do documento.

## 4. Visão comercial

Comercialmente, esta feature sustenta uma promessa importante: o cliente pode subir planilhas operacionais e a plataforma não vai esmagar tudo em texto bruto. Ela vai tentar preservar a estrutura útil para perguntas, auditoria e navegação posterior.

Isso resolve dores muito concretas.

- Relatórios mensais em múltiplas abas.
- Bases de indicadores com colunas numéricas e datas.
- Cadastro de itens, clientes ou contratos em tabelas tabulares.
- Planilhas legadas `.xls` ainda presentes em operação.

O diferencial confirmado no código não é só ler `.xlsx`. O diferencial é combinar leitura do workbook, detecção de tabela nativa, heurística de grid, inferência simples de schema, papéis analíticos por coluna e chunking orientado por linha.

O que não deve ser prometido é suporte universal a todo o ecossistema Excel. O próprio código deixa claro que `.xlsm` e `.xlsb` ficam fora do contrato atual e que `.xls` entra apenas em modo best-effort.

## 5. Visão estratégica

Estratégicamente, o pipeline de Excel fortalece a plataforma em cinco frentes.

- Cria uma ponte entre ingestão documental e ingestão tabular sem reinventar toda a esteira.
- Mantém um caminho comum de indexação, persistência e telemetria, mas com processamento especializado por tipo.
- Reforça a ideia de que chunk não é só texto; chunk pode carregar estrutura útil para runtime futuro.
- Alimenta casos especializados de consulta estruturada sobre planilhas.
- Reduz a necessidade de pipelines paralelos para cada tipo de workbook.

Isso é relevante porque Excel costuma ficar no limbo entre ETL e documento. Se o produto trata Excel só como arquivo ou só como tabela plana, ele perde metade do valor. Aqui o desenho tenta equilibrar os dois lados.

## 6. Conceitos necessários para entender

### 6.1. Workbook

Workbook é o arquivo Excel inteiro. Ele pode conter várias planilhas, cada uma com estrutura, densidade e finalidade diferentes.

### 6.2. Planilha ou aba

A planilha é a unidade operacional mais importante do pipeline. O código faz quase toda a extração pensando em sheet por sheet.

### 6.3. Tabela nativa

Tabela nativa é a estrutura formal que o próprio workbook registra em `ws.tables`. Ela é mais confiável do que adivinhar a grade, porque já traz intervalo, nome e metadados da tabela.

### 6.4. Tabela heurística

Tabela heurística é a estrutura inferida quando a planilha não expõe tabela nativa. O analisador tenta descobrir onde a tabela começa, onde termina, se há cabeçalho e qual é o tipo geral da folha.

### 6.5. Densidade de dados

Densidade de dados é a proporção entre células não vazias e área analisada. Ela ajuda o pipeline a distinguir uma planilha útil de uma folha quase vazia, auxiliar ou meramente decorativa.

### 6.6. Cabeçalho automático

Cabeçalho automático é a inferência de que a primeira linha representa nomes de coluna. O pipeline usa uma heurística simples baseada em predominância de texto.

### 6.7. Schema resumido

Schema resumido é o conjunto de metadados por coluna, como tipo inferido, papel analítico, amostras e presença de nulos. Ele existe para dar contexto estrutural ao acervo e aos consumidores posteriores.

### 6.8. Papel analítico

Papel analítico é a classificação prática de uma coluna como métrica, dimensão, identificador ou dimensão temporal. Não é um modelo estatístico avançado; é uma heurística operacional para tornar a planilha mais útil depois da ingestão.

### 6.9. Chunk row-aware

Chunk row-aware é um chunk construído a partir de uma linha da planilha, e não de um bloco arbitrário de sentenças. Esse é o coração do valor do pipeline Excel.

## 7. Como a feature funciona por dentro

O fluxo começa quando a esteira comum identifica a extensão `.xlsx` ou `.xls`, materializa um `StorageDocument` e entrega o documento ao `ExcelContentProcessor`. A partir daí, o processor valida o contrato do arquivo, carrega o workbook, percorre as planilhas, detecta tabelas e densidade, extrai texto e metadados estruturados e devolve um `ExcelDocument` com conteúdo e contexto tabular.

Depois disso, o processamento assíncrono canônico usa o método `process_document` da base para executar:

- pre-processamento comum;
- extração textual do `ExcelDocument`;
- limpeza leve do conteúdo;
- chunking especializado orientado por linha;
- pós-processamento comum.

No fim, a esteira comum adiciona metadata canônica, indexa os chunks no vector store e persiste o documento processado.

O ponto decisivo é este: a parte especializada do Excel não termina na extração de texto. Ela continua até a montagem dos chunks com `row_data`, nomes de coluna, tipos inferidos e papéis analíticos.

## 8. Divisão em etapas ou submódulos

Detalhamento aprofundado por etapa:

1. [Normalizacao de entrada](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-NORMALIZACAO-DE-ENTRADA.md)
2. [Validacao de contrato do arquivo](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-VALIDACAO-DE-CONTRATO-DO-ARQUIVO.md)
3. [Carregamento do workbook](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-CARREGAMENTO-DO-WORKBOOK.md)
4. [Analise estrutural por aba](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-ANALISE-ESTRUTURAL-POR-ABA.md)
5. [Extracao de conteudo e metadados](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-EXTRACAO-DE-CONTEUDO-E-METADADOS.md)
6. [Chunking row-aware](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-CHUNKING-ROW-AWARE.md)
7. [Fechamento na esteira comum](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-FECHAMENTO-NA-ESTEIRA-COMUM.md)

### 8.1. [Normalizacao de entrada](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-NORMALIZACAO-DE-ENTRADA.md)

Esta etapa existe para garantir que o arquivo chega ao processor como `StorageDocument` com tipo coerente. A esteira comum faz a inferência da extensão e escolhe o processor certo.

O que recebe: caminho e bytes do arquivo.

O que faz: resolve `.xlsx` e `.xls` como tipos próprios de Excel.

O que entrega: documento apto a virar `ExcelDocument`.

### 8.2. [Validacao de contrato do arquivo](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-VALIDACAO-DE-CONTRATO-DO-ARQUIVO.md)

Esta etapa existe para rejeitar formatos fora do contrato atual e impedir que o pipeline finja suportar mais do que realmente suporta.

O que recebe: `StorageDocument`.

O que faz: valida extensão, tamanho, existência do arquivo local quando aplicável e presença de bytes.

O que entrega: autorização para carregar o workbook ou falha explícita.

### 8.3. [Carregamento do workbook](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-CARREGAMENTO-DO-WORKBOOK.md)

Esta etapa existe para abrir o Excel com a engine adequada.

O que recebe: bytes do arquivo.

O que faz: tenta `openpyxl` para o caminho moderno e usa `xlrd` apenas para `.xls` legado.

O que entrega: workbook carregado em modo moderno ou best-effort legado.

### 8.4. [Analise estrutural por aba](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-ANALISE-ESTRUTURAL-POR-ABA.md)

Esta etapa existe para descobrir o que cada planilha é de fato.

O que recebe: workbook carregado.

O que faz: detecta tabelas nativas, tabelas heurísticas, densidade e tipo estrutural da folha.

O que entrega: `ExcelSheetInfo` e `ExcelTableInfo`.

### 8.5. [Extracao de conteudo e metadados](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-EXTRACAO-DE-CONTEUDO-E-METADADOS.md)

Esta etapa existe para transformar a planilha em conteúdo indexável sem perder sua semântica principal.

O que recebe: workbook e resultados da análise estrutural.

O que faz: percorre linhas, monta texto por aba, resume schema, calcula estatísticas numéricas e registra tabelas/linhas estruturadas.

O que entrega: `ExcelDocument` com conteúdo textual e metadata rica.

### 8.6. [Chunking row-aware](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-CHUNKING-ROW-AWARE.md)

Esta etapa existe para evitar que a planilha seja quebrada por sentenças como um texto comum.

O que recebe: `ExcelDocument` completo.

O que faz: gera um chunk por linha útil, preservando sheet, row, columns, roles e tipos.

O que entrega: chunks estruturados que continuam parecendo linhas tabulares.

### 8.7. [Fechamento na esteira comum](README-CONCEITUAL-INGESTAO-EXCEL-PIPELINE-COMPLETO-FECHAMENTO-NA-ESTEIRA-COMUM.md)

Esta etapa existe para transformar o resultado do processor em acervo operacional do produto.

O que recebe: chunks do Excel.

O que faz: indexa, persiste e registra telemetria comum.

O que entrega: workbook efetivamente incorporado ao acervo.

## 9. Pipeline ou fluxo principal

![9. Pipeline ou fluxo principal](../assets/diagrams/docs-readme-conceitual-ingestao-excel-pipeline-completo-diagrama-01.svg)

O diagrama mostra que o pipeline Excel não é um parser único. Ele alterna entre leitura de workbook, análise estrutural e montagem de chunks estruturados antes de voltar para a indexação comum.

## 10. Decisões técnicas e trade-offs

### 10.1. `openpyxl` para caminho principal e `xlrd` só para legado `.xls`

Ganho: o pipeline mantém suporte ao formato moderno e ainda atende parte do legado operacional.

Custo: o caminho `.xls` nasce mais limitado e explícito como best-effort.

Impacto: melhora cobertura real do produto sem fingir paridade total entre formatos.

### 10.2. Ler valores em vez de fórmulas como regra principal

Ganho: o acervo recebe o valor calculado armazenado, que é o dado mais útil para consulta comum.

Custo: o pipeline não vira engine de interpretação de fórmulas.

Impacto: simplifica ingestão e reduz risco de comportamento inconsistente entre engines.

### 10.3. Combinar tabela nativa com heurística de grid

Ganho: o pipeline aproveita o melhor dos dois mundos. Quando a tabela formal existe, usa o contrato do workbook. Quando não existe, tenta inferir.

Custo: heurística sempre traz risco de falso positivo ou falso negativo.

Impacto: melhora a cobertura prática de planilhas do mundo real.

### 10.4. Filtrar por densidade mínima de dados

Ganho: evita indexar abas vazias, auxiliares ou pobres demais.

Custo: uma configuração agressiva pode excluir folhas que ainda teriam valor.

Impacto: reduz ruído no acervo quando calibrado corretamente.

### 10.5. Chunking por linha em vez de chunking textual puro

Ganho: preserva a unidade lógica da planilha.

Custo: aumenta o número potencial de chunks e exige disciplina nos metadados.

Impacto: melhora muito a utilidade downstream para cenários analíticos e perguntas estruturadas.

## 11. Comparação com estado da arte

O estado da arte atual para ingestão de Excel se concentra em três famílias principais.

- Leitura orientada a workbook e estruturas nativas, como `openpyxl`.
- Leitura orientada a dataframe com inferência de tipos, como `pandas.read_excel`.
- Leitura orientada a performance e schema explícito, como `polars.read_excel` com engines rápidas e overrides de schema.

Comparando o pipeline do projeto com essas linhas:

### 11.1. Convergências fortes

- O pipeline usa `openpyxl` como engine principal, alinhado com a leitura moderna de `.xlsx` e com a exploração de tabelas nativas do workbook.
- O sistema tenta preservar tipos de datas e números, schema resumido e estatísticas numéricas, o que converge com a preocupação dataframe-centric de `pandas` e `polars` com tipagem útil.
- A extração separada por sheet e a noção de múltiplas abas seguem o comportamento esperado das bibliotecas modernas de leitura de Excel.

### 11.2. Diferenciais práticos do projeto

- O projeto não para no dataframe nem no workbook. Ele transforma a planilha em chunks row-aware com `row_data`, `column_types` e `column_roles` para indexação e consumo posterior.
- O uso simultâneo de tabela nativa e heurística de grid tenta lidar melhor com planilhas corporativas irregulares, onde nem toda folha foi modelada como tabela formal.
- A integração com a esteira comum de ingestão evita criar uma pipeline isolada só para planilhas.

### 11.3. Limites frente ao estado da arte

- O contrato atual rejeita `.xlsm` e `.xlsb`, enquanto leitores dataframe modernos já cobrem mais formatos e engines.
- O caminho `.xls` usa `xlrd` em best-effort e não expõe o mesmo nível de riqueza estrutural do caminho `.xlsx`.
- O pipeline não usa engine orientada a performance como `calamine` ou caminho dataframe de alta velocidade como o ecossistema atual de `polars`.
- Fórmulas, macros, pivots, comentários, validações e vários objetos do workbook ficam fora do escopo confirmado do contrato atual.

## 12. O que acontece em caso de sucesso

No caminho feliz, o arquivo é reconhecido como Excel suportado, o workbook carrega corretamente, as planilhas úteis passam no filtro de densidade, o schema resumido é gerado, os chunks row-aware são criados e a esteira comum indexa e persiste o resultado.

Para o usuário, isso aparece como uma planilha que passa a ser consultável com contexto de aba, coluna e linha. Para operação, isso aparece como documento persistido com metadados de schema, tabelas e estatísticas numéricas.

## 13. O que acontece em caso de erro

Os principais cenários confirmados no código são estes.

- Arquivo sem `file_path`.
- Extensão fora do contrato suportado.
- `.xlsm` ou `.xlsb` recebidos como se fossem suportados.
- Arquivo local inexistente.
- Arquivo acima do limite de tamanho configurado.
- Workbook sem bytes.
- Falha do `openpyxl` no caminho moderno.
- Falha do `xlrd` no caminho legado.
- `.xls` protegido por senha.
- Planilha específica não encontrada em chamadas auxiliares.
- Planilha ignorada por baixa densidade.

O ponto importante é que o pipeline não mascara esses cenários como sucesso parcial silencioso. Alguns viram falha obrigatória e outros viram exclusão explícita de aba com log de motivo.

## 14. Observabilidade e diagnóstico

Os pontos mais úteis de observabilidade do Excel são:

- logs de conversão de `StorageDocument` para `ExcelDocument`;
- logs de falha de carga do workbook;
- log explícito de entrada em modo `best_effort_xls`;
- logs de exclusão de planilha por baixa densidade;
- metadados `excel_engine`, `excel_processing_mode` e `excel_contract_profile`;
- metadados `sheets`, `excel_schema_summary`, `excel_numeric_stats` e `tables_data`;
- metadados de chunk como `sheet_name`, `row_index`, `column_names`, `row_data` e `column_roles`.

Na prática, o troubleshooting eficiente segue esta ordem.

1. Confirmar qual extensão e engine foram usadas.
2. Ver se o arquivo caiu em modo moderno ou legado best-effort.
3. Ver quantas planilhas sobreviveram aos filtros.
4. Ver se houve schema resumido e dados tabulares na metadata.
5. Conferir se os chunks finais carregam `row_data` e contexto de coluna.
6. Só então investigar indexação ou consulta.

## 15. Impacto técnico

O impacto técnico principal é aproximar ingestão documental e ingestão tabular sem destruir o modelo comum da plataforma. O pipeline reforça:

- especialização por tipo de conteúdo sem pipeline paralelo inteiro;
- preservação de semântica tabular em chunks indexáveis;
- enriquecimento estrutural útil para consumidores posteriores;
- contrato explícito sobre o que é moderno, legado ou não suportado;
- maior observabilidade da planilha além do texto bruto.

## 16. Impacto executivo

Para liderança, esta feature reduz o risco de um formato extremamente comum entrar no produto de forma pobre. Também ajuda a sustentar casos corporativos onde a planilha é a principal fonte de verdade operacional.

## 17. Impacto comercial

Para venda e pré-venda, a feature melhora a história em contas que vivem de planilha. O diferencial não é simplesmente aceitar upload de Excel, e sim preservar a estrutura útil da planilha dentro da ingestão.

## 18. Impacto estratégico

Estratégicamente, este pipeline cria uma base concreta para evoluir consultas especializadas sobre dados tabulares sem obrigar o produto a migrar todo Excel para um ETL separado. Ele mantém o workbook dentro da esteira de ingestão, mas com metadados ricos o suficiente para suportar usos mais sofisticados depois.

## 19. Exemplos práticos guiados

### 19.1. Planilha financeira com abas mensais

Cenário: o cliente envia um workbook com doze abas, cada uma representando um mês.

Processamento esperado: o pipeline percorre cada aba, extrai as linhas úteis, ignora folhas auxiliares configuradas, produz schema por coluna e gera chunks por linha.

Impacto prático: cada registro mensal entra no acervo com contexto de aba e colunas.

### 19.2. Workbook legado `.xls`

Cenário: o cliente ainda opera com planilha antiga em formato legado.

Processamento esperado: o pipeline tenta o caminho moderno, identifica a necessidade de fallback e entra em `best_effort_xls` via `xlrd`.

Impacto prático: a plataforma mantém uma rota de compatibilidade, mas explicita suas limitações.

### 19.3. Planilha com abas auxiliares e quase vazias

Cenário: o workbook tem folhas de configuração, observação e abas quase sem dados.

Processamento esperado: nomes excluídos e densidade mínima evitam que o acervo seja poluído por conteúdo de baixo valor.

Impacto prático: a indexação prioriza o que realmente parece dado útil.

## 20. Explicação 101

Pense no pipeline Excel como um analista que recebe uma pasta cheia de planilhas.

Ele primeiro checa se o arquivo é do tipo certo.
Depois decide qual ferramenta consegue abrir aquele workbook.
Em seguida olha aba por aba para entender se aquilo é uma tabela real, um formulário, uma folha vazia ou uma mistura.
Depois anota o nome das colunas, os tipos mais prováveis e os dados mais importantes.
Só no final ele quebra o material em pedaços menores para guardar no acervo.

O valor do pipeline está em não perder a cara de planilha no meio desse caminho.

## 21. Limites e pegadinhas

- `.xlsx` e `.xls` não têm o mesmo nível de riqueza operacional no pipeline.
- O pipeline lê principalmente valores; ele não é uma engine completa de fórmulas.
- Folhas esparsas podem ser descartadas se a densidade mínima estiver alta demais.
- Cabeçalho automático é heurístico, não garantia absoluta.
- Tabela heurística melhora cobertura, mas não substitui um workbook bem modelado.
- Formatos `.xlsm` e `.xlsb` estão fora do contrato atual, mesmo que bibliotecas modernas consigam lidar com eles em outros contextos.

## 22. Checklist de entendimento

- Entendi por que Excel precisa de pipeline próprio.
- Entendi a diferença entre `.xlsx` moderno e `.xls` best-effort.
- Entendi por que o pipeline tenta preservar tabela e schema.
- Entendi o papel da densidade de dados e da detecção de cabeçalhos.
- Entendi por que o chunking row-aware é central.
- Entendi as diferenças entre o pipeline atual e o estado da arte dataframe-centric.
- Entendi os limites que não devem ser prometidos comercialmente.

## 23. Evidências no código

- `src/ingestion_layer/processors/excel_processor.py`
  - Motivo da leitura: entrypoint e implementação principal do pipeline Excel.
  - Símbolo relevante: `ExcelContentProcessor`.
  - Comportamento confirmado: validação, carga do workbook, extração por sheet, schema, contract metadata e chunking row-aware.

- `src/ingestion_layer/processors/excel_sheet_analyzer.py`
  - Motivo da leitura: análise estrutural das planilhas.
  - Símbolo relevante: `ExcelSheetAnalyzer`.
  - Comportamento confirmado: detecção de tabelas nativas, heurística de grid, densidade e tipo estrutural da aba.

- `src/ingestion_layer/processors/base.py`
  - Motivo da leitura: boundary comum do processamento assíncrono.
  - Símbolo relevante: `BaseContentProcessor.process_document`.
  - Comportamento confirmado: executa pre-processamento, extração, limpeza, chunking e pós-processamento.

- `src/ingestion_layer/file_pipeline_services.py`
  - Motivo da leitura: integração com a esteira comum de indexação e persistência.
  - Símbolo relevante: `DocumentProcessorExecutor` e `DocumentIndexingExecutor`.
  - Comportamento confirmado: entrega o `StorageDocument` ao processor, recebe chunks e finaliza indexação e persistência.

- `src/ingestion_layer/core/data_models.py`
  - Motivo da leitura: contrato do documento estruturado de Excel.
  - Símbolo relevante: `ExcelDocument`.
  - Comportamento confirmado: workbook ingerido vira documento com sheet names, totais, raw data e tables data.

- `app/yaml/system/rag-config-modelo.yaml`
  - Motivo da leitura: evidência de configuração canônica usada pelos ambientes do projeto.
  - Símbolo relevante: bloco `excel`.
  - Comportamento confirmado: expõe parâmetros de arquivo, análise, extração e data output que o processor consome.

- `src/qa_layer/json_rag/specialized_rag_excel.py`
  - Motivo da leitura: consumo downstream da estrutura produzida pela ingestão Excel.
  - Símbolo relevante: `_build_structured_entry`.
  - Comportamento confirmado: chunks do Excel carregam contexto suficiente para consulta estruturada posterior.
