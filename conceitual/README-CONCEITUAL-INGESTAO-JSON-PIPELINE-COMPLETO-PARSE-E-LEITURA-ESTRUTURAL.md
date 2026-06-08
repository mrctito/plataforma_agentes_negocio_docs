# Manual detalhado da etapa: Parse e leitura estrutural do pipeline JSON

## 1. O que esta etapa faz

Esta etapa transforma o texto JSON validado em estrutura navegavel e extrai conhecimento estrutural suficiente para que o restante do pipeline saiba com o que esta lidando. Ela existe porque JSON valido nao e necessariamente JSON entendido.

Em linguagem simples: e a etapa que troca texto cru por entendimento estrutural do payload.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa acontece principalmente dentro de JsonMetadataBuilder.build_document, logo depois de o raw_text e o parsed_json ficarem disponiveis.

## 3. O que entra e o que sai

Entradas confirmadas:

- raw_text ja validado
- parsed_json vindo da validacao fisica

Saidas confirmadas:

- deteccao de colecoes de cupons e produtos
- schema summary
- numeric stats
- samples de catalogo e cupom
- structured_content opcional em metadata
- json_normalized_shadow em metadata
- TextDocument final preservando o JSON bruto como content

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. decide se o json_data ja veio pronto ou se precisa parsear raw_text;
2. detecta colecoes de dicionarios em listas de topo ou em listas dentro de objetos;
3. aplica heuristicas internas de classificacao de cupom e produto sobre os samples;
4. normaliza os registros encontrados;
5. calcula estatisticas de cupom e catalogo;
6. percorre a estrutura inteira em _analyze_json_structure para montar schema_summary e numeric_stats;
7. coleta amostras e metadados opcionais;
8. valida quality_flags;
9. monta metadata completa;
10. preserva structured_content apenas em metadata;
11. gera json_normalized_shadow para reforco textual;
12. monta o TextDocument final mantendo o JSON bruto no corpo principal.

O detalhe importante e que o builder produz entendimento estrutural sem adulterar o content principal do documento. A inteligencia fica sobretudo na metadata.

## 5. Decisoes tecnicas importantes

### 5.1. Heuristicas de negocio atuam sobre colecoes detectadas

_detect_record_sets nao tenta classificar qualquer no arbitrario. Ele procura listas de dicionarios e so entao aplica heuristicas de cupom e produto. Isso reduz classificacao oportunista demais em objetos isolados.

### 5.2. Schema summary e numeric stats nascem da caminhada completa da arvore

_analyze_json_structure percorre recursivamente objetos e arrays, registra tipos, samples e estatisticas numericas por path. Isso cria visibilidade fina do documento sem exigir flattening total.

### 5.3. Conteudo principal continua sendo o JSON bruto

Mesmo com structured_content e shadow normalizada disponiveis, o metodo que compoe o conteudo final preserva o raw_text como base do documento. O ganho pratico e manter fidelidade a fonte e deixar enriquecimento semanticamente auxiliar na metadata.

## 6. O que pode dar errado

Os limites e riscos confirmados ou implicados sao estes.

- um JSON valido pode nao conter nenhuma colecao reconhecivel de catalogo ou cupom;
- heuristicas podem ter falso positivo ou falso negativo;
- schema_summary e numeric_stats podem ficar grandes em estruturas muito profundas;
- falta de registros de negocio nao invalida o arquivo, mas reduz o enriquecimento semantico disponivel.

## 7. Como diagnosticar

Os sinais mais uteis desta etapa sao:

- cupom_estatisticas e catalogo_estatisticas na metadata;
- amostras_cupons e amostras_produtos;
- json_schema_summary e json_numeric_stats;
- quality_flags e qualidade_aprovada;
- json_structured_content e json_normalized_shadow.

Em linguagem simples: se o arquivo entrou, mas o acervo ficou pobre, esta etapa mostra se faltou estrutura detectavel ou se o problema apareceu so depois, no chunking.

## 8. Exemplo pratico guiado

Cenario: um export de ERP chega com objeto raiz contendo listas de produtos e cupons.

1. O builder identifica duas colecoes de dicionarios.
2. As heuristicas reconhecem uma lista como produto e outra como cupom.
3. O pipeline calcula estatisticas de catalogo e cupons.
4. Gera schema summary e numeric stats para os caminhos mais relevantes.
5. A metadata final fica rica, enquanto o content do documento continua sendo o JSON original.

O valor desta etapa e construir entendimento estrutural sem transformar o slice JSON em ETL tabular forcado.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/json_metadata_builder.py
  - Simbolo relevante: build_document
  - Comportamento confirmado: composicao do documento enriquecido a partir de parsed_json, metadata estrutural e JSON bruto preservado.
- src/ingestion_layer/processors/json_metadata_builder.py
  - Simbolo relevante: deteccao de record sets, heuristicas de classificacao de cupom e produto e analise estrutural do JSON
  - Comportamento confirmado: classificacao de colecoes de negocio, schema summary e numeric stats por path.
