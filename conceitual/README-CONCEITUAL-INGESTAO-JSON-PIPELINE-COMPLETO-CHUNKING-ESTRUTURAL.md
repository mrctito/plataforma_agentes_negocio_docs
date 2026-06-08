# Manual detalhado da etapa: Chunking estrutural do pipeline JSON

## 1. O que esta etapa faz

Esta etapa converte o documento JSON em chunks coerentes com a sua estrutura. Ela nao corta texto por tamanho fixo de forma cega. O objetivo dela e respeitar o formato do payload: objeto, array ou valor primitivo.

Em linguagem simples: e a etapa que quebra o JSON do jeito menos artificial possivel dentro do slice atual.

## 2. Onde ela entra no fluxo

No codigo lido, esta etapa aparece em dois caminhos.

- o caminho assincrono canonico, em _split_into_chunks;
- a superficie sincronica auxiliar, em create_chunks.

Ambos reutilizam a mesma logica estrutural de criacao de metadata e de chunks.

## 3. O que entra e o que sai

Entradas confirmadas:

- document
- content JSON normalizado
- profile ativo
- metadata estrutural enriquecida

Saidas confirmadas:

- ContentChunk com chunk_type especifico para objeto, array, item de array ou primitivo
- metadata estrutural adicional como json_keys, content_type_specific, json_key, array_index e key_path
- limite de chunks respeitado pelo perfil

## 4. Como o codigo implementa a etapa

O fluxo assincrono canonico segue esta ordem.

1. resolve o profile;
2. faz json.loads do content normalizado;
3. cria metadata estrutural adicional;
4. chama _create_json_chunks;
5. aplica max_chunks_per_document do perfil;
6. aplica domain processing quando permitido.

Dentro de _create_json_chunks, a escolha de estrategia depende do tipo do valor raiz.

- se for dict, chama _chunk_json_object;
- se for list, chama _chunk_json_array;
- se for primitivo, cria um unico chunk json_primitive.

Para objetos, existem dois modos.

- preserve_structure ligado: chunk unico com o objeto inteiro;
- preserve_structure desligado: um chunk por chave de primeiro nivel.

Para arrays, tambem existem dois modos.

- flatten_arrays desligado ou array com um item: chunk unico com o array inteiro;
- flatten_arrays ligado e array multiplo: um chunk por item.

## 5. Decisoes tecnicas importantes

### 5.1. A estrutura do JSON manda mais que o tamanho do texto

O processor olha primeiro para a natureza do payload. Isso e importante porque um array e um objeto exigem cortes diferentes se o objetivo for preservar significado recuperavel.

### 5.2. Preserve_structure e flatten_arrays mudam profundamente a granularidade

Essas duas flags alteram mais do que detalhes de formato. Elas definem se o slice vai privilegiar contexto completo do objeto/array ou granularidade maior para retrieval mais fino.

### 5.3. Existe fallback controlado so em parte da superficie

No caminho sincronico create_chunks, quando o JSON esta invalido, o profile standard pode cair em chunk simples com warning; o profile schema_metadata nao aceita isso. Isso mostra que o slice tolera degradacao apenas quando a governanca do perfil permite.

## 6. O que pode dar errado

Falhas e limites confirmados:

- json.loads pode falhar em _split_into_chunks e abortar o chunking;
- preserve_structure pode gerar chunk unico grande demais para certos cenarios;
- flatten_arrays pode multiplicar demais a cardinalidade de chunks;
- create_chunks pode cair em fallback simples no profile standard quando o JSON estiver invalido.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- logs de JSON CHUNKING e JSON ANALYSIS;
- chunk_type nos chunks gerados;
- metadata content_type_specific com depth, total_keys e numeric_paths;
- logs de estrategia preserve_structure ou key_based_chunks;
- logs de array completo ou flattened_items.

Em linguagem simples: se o retrieval ficou ruim ou super fragmentado, a explicacao quase sempre passa pela granularidade escolhida aqui.

## 8. Exemplo pratico guiado

Cenario: um JSON raiz e um array de 300 itens e flatten_arrays esta ligado.

1. O processor resolve o profile standard.
2. Faz json.loads do content.
3. Cria metadata estrutural.
4. _create_json_chunks detecta um array.
5. _chunk_json_array entra no modo flattened_items.
6. Cada item vira um chunk com array_index e array_length.
7. O limite max_chunks_per_document ainda pode truncar a saida final.

O valor desta etapa e aproximar a segmentacao do formato real do dado, sem fingir que todo JSON deveria ser tratado como um texto corrido.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/json_processor.py
  - Simbolo relevante: etapa assincrona de chunking, criacao de metadata JSON e fabrica estrutural de chunks
  - Comportamento confirmado: caminho assincrono canonico, metadata estrutural e bifurcacao por tipo de payload.
- src/ingestion_layer/processors/json_processor.py
  - Simbolo relevante: chunking de objetos, chunking de arrays e superficie sincronica de criacao de chunks
  - Comportamento confirmado: modos preserve_structure, flatten_arrays e fallback controlado da superficie sincronica.
