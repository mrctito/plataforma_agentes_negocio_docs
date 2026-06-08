# Manual detalhado da etapa: Enriquecimento de dominio e qualidade do pipeline JSON

## 1. O que esta etapa faz

Esta etapa transforma um JSON estruturalmente valido em um documento mais util para negocio e para retrieval posterior. Ela combina duas linhas de enriquecimento diferentes.

- quality e estatistica produzidas no builder;
- domain processing aplicado depois, sobre os chunks, quando o perfil permite.

Em linguagem simples: e a etapa em que o slice JSON deixa de ser apenas tecnico e passa a carregar semantica de negocio.

## 2. Onde ela entra no fluxo

Ela atravessa dois momentos do pipeline.

- primeiro no JsonMetadataBuilder, que calcula stats, optional metadata e quality_flags;
- depois no JsonContentProcessor, que pode chamar DomainProcessingResolver sobre os chunks finais.

## 3. O que entra e o que sai

Entradas confirmadas:

- colecoes detectadas de cupons e produtos
- configuracao de quality_filters, coupon_processing e catalog_processing
- domain_specific_processing
- perfil do documento

Saidas confirmadas:

- quality_flags e qualidade_aprovada
- cupom_estatisticas e catalogo_estatisticas
- metadados opcionais coletados
- chunks enriquecidos por dominio quando aplicavel
- applied_domains para log e diagnostico

## 4. Como o codigo implementa a etapa

O fluxo observado combina duas partes.

Na parte de qualidade e estatistica:

1. o builder detecta colecoes de negocio;
2. normaliza os registros;
3. calcula estatisticas especificas de cupons e catalogo;
4. coleta metadados opcionais configurados;
5. executa _validate_quality e registra quality_flags.

Na parte de dominio:

1. o processor cria DomainProcessingResolver em _setup_domain_processing;
2. o resolver inicializa MetadataSchemaRegistry e DomainProcessorFactory;
3. a factory registra e instancia os processadores habilitados;
4. create_all_enabled_processors ordena tudo por get_processing_priority;
5. depois do chunking, _apply_domain_processing chama apply_processing(document, chunks);
6. o retorno vem como DomainProcessingOutcome com chunks enriquecidos.

## 5. Decisoes tecnicas importantes

### 5.1. Qualidade e dominio nao sao a mesma coisa

Quality flags indicam se o documento parece operacionalmente coerente para o caso de uso. Domain processing tenta acrescentar semantica especializada de negocio. O slice separa essas duas responsabilidades, o que melhora clareza de diagnostico.

### 5.2. Domain processing e capability transversal, nao processor paralelo

O codigo nao cria um processor JSON diferente para food service, cupom ou catalogo. Ele reutiliza DomainProcessingResolver como capability compartilhada, com cadeia de plugins priorizados. O ganho pratico e evolucao sem multiplicar slices paralelos.

### 5.3. Schema metadata desliga dominio por desenho

O perfil schema_metadata desativa allow_domain_processing. Isso e uma decisao importante de governanca. Um artefato tecnico de schema nao deve receber heuristica de negocio por acidente.

## 6. O que pode dar errado

Falhas e limites confirmados:

- quality_flags podem indicar documento ruim mesmo quando o parse passou;
- o dominio pode nao se aplicar a nenhum chunk;
- um plugin pode extrair metadata invalida e cair em fallback reduzido pela base comum do processador;
- o perfil ativo pode desligar totalmente o domain processing.

Na pratica, esta etapa ajuda a distinguir "JSON valido" de "JSON util".

## 7. Como diagnosticar

Os sinais mais uteis sao:

- quality_flags e qualidade_aprovada;
- cupom_estatisticas e catalogo_estatisticas;
- metadados_opcionais;
- logs de domain processing JSON habilitado ou desabilitado;
- applied_domains e chunks enriquecidos no retorno do resolver.

Em linguagem simples: se o documento entrou, mas nao ficou inteligente o suficiente para o negocio, a explicacao quase sempre mora aqui.

## 8. Exemplo pratico guiado

Cenario: um export JSON de catalogo do varejo chega com nome, sku, preco, categoria e fornecedor.

1. O builder reconhece a colecao como produto.
2. Calcula catalogo_estatisticas.
3. Coleta optional metadata como marca e fornecedor.
4. O profile standard permite domain processing.
5. O resolver aplica o plugin de catalogo por prioridade.
6. Os chunks saem com metadata mais rica para retrieval futuro.

O valor desta etapa e transformar estrutura em sinal de negocio reaproveitavel.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/json_metadata_builder.py
  - Simbolo relevante: calculo de estatisticas de cupons, estatisticas de catalogo, coleta de metadados opcionais e validacao de qualidade
  - Comportamento confirmado: enrichment de qualidade e estatisticas antes do chunking.
- src/ingestion_layer/processors/json_processor.py
  - Simbolo relevante: configuracao do domain processing e aplicacao da cadeia de dominio sobre os chunks
  - Comportamento confirmado: inicializacao do resolvedor de dominio e aplicacao da cadeia sobre os chunks quando o perfil permite.
- src/ingestion_layer/processors/domain_plugins/domain_processing_resolver.py
  - Simbolo relevante: apply_processing
  - Comportamento confirmado: retorno com DomainProcessingOutcome e applied_domains.
