# Manual detalhado da etapa: Validacao fisica do arquivo no pipeline JSON

## 1. O que esta etapa faz

Esta etapa verifica se o arquivo JSON pode ser tratado como entrada confiavel do pipeline. Ela existe para impedir ingestao melhor-esforco sobre um arquivo grande demais, sem bytes brutos ou com problema de integridade de encoding.

Em linguagem simples: e a inspeção fisica minima antes de o slice acreditar no arquivo.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa esta concentrada no JsonMetadataBuilder.build_from_storage. Ela vem logo depois da resolucao do contrato YAML e antes da construcao do documento enriquecido.

## 3. O que entra e o que sai

Entradas confirmadas:

- StorageDocument com file_size, file_path e raw_bytes
- configuracao de tamanho e lista ordenada de encodings permitidos

Saidas confirmadas:

- validacao de tamanho concluida ou falha explicita
- bytes brutos confirmados
- raw_text decodificado com encoding efetivo
- hash SHA-256 dos bytes brutos para auditoria

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. _validate_size compara file_size com size.max_file_size_mb;
2. _extract_storage_raw_bytes exige raw_bytes no StorageDocument;
3. _decode_and_validate_json_bytes tenta decodificar os bytes usando a lista ordenada de encodings permitidos;
4. para cada encoding, primeiro tenta decode e depois json.loads do texto resultante;
5. se decode falhar, registra warning de decode;
6. se decode passar, mas json.loads falhar, registra warning de parse;
7. no primeiro encoding que passa nos dois testes, devolve decoded_text, effective_encoding, parsed_json e raw_bytes_hash;
8. build_from_storage registra log com effective_encoding, raw_bytes_sha256, tamanho do texto e first_char.

O detalhe importante e que a validacao fisica nao termina no decode. O slice so aceita um encoding como valido quando decode e parse passam juntos.

## 5. Decisoes tecnicas importantes

### 5.1. Bytes brutos sao obrigatorios

O builder nao aceita seguir sem raw_bytes. Se o StorageDocument nao trouxer esse dado, o codigo registra erro e falha. O ganho pratico e manter rastreabilidade de integridade e evitar uma validacao baseada apenas em strings ja montadas por outra camada.

### 5.2. Integridade de encoding e validada junto com parse

Nao basta conseguir decodificar bytes em texto. O texto precisa continuar sendo JSON valido. Isso separa dois tipos de erro que costumam se misturar em pipelines mais frouxos.

### 5.3. Auditoria segura do payload

O codigo registra hash dos bytes, tamanho do texto e primeiro caractere util, mas nao despeja o arquivo inteiro em log. Isso melhora diagnostico sem comprometer observabilidade com volume ou sensibilidade desnecessaria.

## 6. O que pode dar errado

Falhas confirmadas:

- arquivo maior do que o limite configurado;
- StorageDocument sem raw_bytes;
- falha de decode em todos os encodings permitidos;
- decode bem-sucedido, mas parse JSON invalido em todos os encodings tentados.

Na pratica, esta etapa e a fronteira que separa problema de arquivo de problema semantico do conteudo.

## 7. Como diagnosticar

Os sinais mais uteis confirmados no codigo sao:

- warnings por encoding e stage de falha;
- erro final diferenciando falha de decode de falha de parse apos decode;
- raw_bytes_sha256 nos logs;
- effective_encoding escolhido;
- first_char e starts_with_json no resumo seguro do payload.

Em linguagem simples: se o JSON parece legivel no editor, mas o slice rejeita, e aqui que a diferenca entre aparencia e integridade real aparece.

## 8. Exemplo pratico guiado

Cenario: um arquivo chega com bytes em utf-8-sig e tamanho dentro do limite.

1. O builder valida o tamanho.
2. Extrai raw_bytes.
3. Tenta decode com utf-8 e pode falhar ou parsear errado.
4. Tenta utf-8-sig.
5. Decode e parse passam juntos.
6. O builder registra o encoding efetivo e o hash dos bytes.
7. O pipeline segue para montar o documento.

O valor desta etapa e impedir que o slice JSON aceite um arquivo apenas porque alguma representacao textual parecia razoavel.

## 9. Evidencias no codigo

- src/ingestion_layer/processors/json_metadata_builder.py
  - Simbolo relevante: build_from_storage, _validate_size, _extract_storage_raw_bytes e _decode_and_validate_json_bytes
  - Comportamento confirmado: validacao de tamanho, exigencia de raw_bytes, tentativa ordenada de encodings e parse obrigatorio junto com decode.
