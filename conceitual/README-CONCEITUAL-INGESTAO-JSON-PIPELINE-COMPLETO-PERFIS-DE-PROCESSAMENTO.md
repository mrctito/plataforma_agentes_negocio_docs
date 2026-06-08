# Manual detalhado da etapa: Perfis de processamento do pipeline JSON

## 1. O que esta etapa faz

Esta etapa escolhe qual modo operacional o processor JSON vai usar para o documento atual. Ela existe porque nem todo JSON deve receber o mesmo tratamento. Um arquivo de negocio pode merecer metadata pesada e plugins de dominio; um artefato de schema metadata precisa ser mais controlado e restritivo.

Em linguagem simples: e a etapa que responde "qual versao do processor JSON este documento precisa agora?".

## 2. Onde ela entra no fluxo

No codigo lido, a decisao mora em JsonContentProcessor._resolve_profile e orienta tanto o caminho assincrono _split_into_chunks quanto a superficie sincronica create_chunks.

## 3. O que entra e o que sai

Entradas confirmadas:

- document.metadata.processing_profile
- document.metadata.document_type
- json_processing_profile na raiz do YAML
- mapa interno de perfis suportados

Saidas confirmadas:

- profile standard ou schema_metadata
- logging da estrategia ativa
- flags operacionais como allow_domain_processing, allow_fallback_chunking e include_json_structure_metadata

## 4. Como o codigo implementa a etapa

O fluxo real observado e este.

1. _build_profiles cria dois perfis internos: standard e schema_metadata;
2. _resolve_profile procura primeiro metadata.processing_profile;
3. se nao houver, tenta inferir schema_metadata por metadata.document_type;
4. se ainda nao houver, usa json_processing_profile na raiz do YAML;
5. se o nome for desconhecido, registra log e cai para standard;
6. _log_profile_selection registra preserve_structure, flatten_arrays, max_chunks_per_document, allow_domain_processing e allow_fallback_chunking.

O profile standard permite dominio, metadata estrutural e fallback controlado. O profile schema_metadata desliga dominio, limita o numero de chunks e endurece o comportamento em caso de JSON invalido.

## 5. Decisoes tecnicas importantes

### 5.1. Perfil e parte da governanca, nao do formato do arquivo

Dois arquivos .json podem cair em perfis diferentes porque o criterio nao e apenas extensao. O processor leva em conta metadata do documento e configuracao de runtime. Isso aproxima o slice de uma governanca por intencao, nao so por tipo de arquivo.

### 5.2. Profile schema_metadata e deliberadamente mais restritivo

Esse perfil reduz ruido e torna o documento tecnico mais previsivel. O ganho pratico e impedir que o slice trate um artefato de schema como se fosse dataset de negocio.

### 5.3. Existe uma superficie paralela de configuracao

O nome do perfil padrao pode vir de json_processing_profile na raiz do YAML. Isso nao invalida o caminho canonico principal, mas cria uma segunda alavanca operacional que precisa ser entendida com cuidado.

## 6. O que pode dar errado

Os riscos e limites confirmados sao estes.

- metadata.processing_profile pode apontar para um perfil inexistente, caso em que o slice cai para standard;
- document_type pode nao refletir corretamente a natureza do arquivo;
- json_processing_profile na raiz pode mudar o comportamento sem estar no bloco principal do JSON;
- schema_metadata desliga dominio e fallback, o que muda bastante a experiencia operacional.

## 7. Como diagnosticar

Os sinais mais uteis desta etapa sao:

- log JSON_PROFILE com profile, preserve_structure, flatten_arrays, allow_domain_processing e allow_fallback_chunking;
- diferenca entre comportamento do caminho assincrono e do create_chunks quando o perfil muda;
- ausencia de dominio ou fallback em documentos classificados como schema_metadata.

Em linguagem simples: se dois JSONs parecidos se comportam de forma muito diferente, quase sempre vale confirmar primeiro o perfil ativo.

## 8. Exemplo pratico guiado

Cenario: um arquivo de schema metadata chega com document_type igual a schema_metadata.

1. _resolve_profile nao encontra processing_profile explicito.
2. Detecta document_type schema_metadata.
3. Seleciona o profile schema_metadata.
4. O processor desliga domain processing.
5. O limite de chunks cai para um.
6. JSON invalido nessa trilha nao recebe fallback permissivo.

O valor desta etapa e alinhar o comportamento do processor ao objetivo real do documento.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/json_processor.py
  - Simbolo relevante: _build_profiles, _resolve_profile e _log_profile_selection
  - Comportamento confirmado: criacao dos perfis standard e schema_metadata, ordem de resolucao do perfil e logging da estrategia ativa.
