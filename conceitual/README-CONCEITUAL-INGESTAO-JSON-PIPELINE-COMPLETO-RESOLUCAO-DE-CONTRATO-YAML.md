# Manual detalhado da etapa: Resolucao de contrato YAML do pipeline JSON

## 1. O que esta etapa faz

Esta etapa define qual configuracao JSON o runtime vai aceitar como valida e como essa configuracao e consolidada antes de qualquer leitura do arquivo. Ela existe para impedir duas coisas perigosas: configuracao espalhada em varios caminhos e continuidade silenciosa de contratos legados.

Em linguagem simples: e a etapa que decide qual YAML manda no slice JSON e qual YAML deve ser rejeitado.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa esta centralizada em src/utils/json_config_resolver.py e e consumida logo na inicializacao do JsonMetadataBuilder. Isso significa que o contrato e resolvido antes de validacao fisica, antes de decode e antes do chunking.

## 3. O que entra e o que sai

Entradas confirmadas:

- yaml_config completo da execucao

Saidas confirmadas:

- configuracao JSON final consolidada
- lista ordenada de encodings permitidos
- sub-blocos resolvidos de size, schema_detection, coupon_processing, catalog_processing, quality_filters, metadata e normalization
- rejeicao explicita de caminhos legados

## 4. Como o codigo implementa a etapa

O fluxo real observado segue esta ordem.

1. valida se yaml_config e um dict; se nao for, devolve o default interno do slice JSON;
2. verifica se algum caminho legado foi usado;
3. se detectar caminho legado, levanta ValueError com guidance explicita;
4. coleta apenas candidatos do caminho canonico ingestion.content_profiles.type_specific.json;
5. faz deep merge entre default interno e a configuracao encontrada;
6. disponibiliza helpers especializados para encodings, size, schema, catalogo, cupons, quality, metadata e normalization.

O ponto importante e que o resolvedor nao aceita varios caminhos concorrentes por conveniencia. Ele centraliza a fonte de verdade e falha cedo quando o YAML ainda estiver preso a topologias antigas.

## 5. Decisoes tecnicas importantes

### 5.1. Contrato canônico único

O codigo aceita apenas ingestion.content_profiles.type_specific.json como caminho de verdade. Isso reduz ambiguidade estrutural e facilita auditoria de configuracao.

### 5.2. Fail-fast para chaves legadas

Os caminhos content_profiles.type_specific.json, ingestion.json e json na raiz nao sao apenas ignorados. Eles sao rejeitados com ValueError. O ganho pratico e impedir que o runtime pareca funcionar com contrato errado.

### 5.3. Default interno coerente, nao improvisado

Quando a configuracao existe, ela nao substitui tudo. Ela e mesclada sobre um default interno que ja declara domain_profile, encoding, size, schema_detection, cupons, catalogo, quality, metadata e normalization. Isso cria um baseline operacional consistente.

## 6. O que pode dar errado

Os cenarios de falha confirmados ou implicados sao estes.

- uso de caminho legado gera erro explicito;
- yaml_config nao sendo dict leva o slice a depender apenas do default interno;
- configuracao parcial pode parecer valida, mas deixar comportamento importante sob default;
- coexistencia com json_processing_profile na raiz cria uma superficie paralela que exige cuidado documental.

## 7. Como diagnosticar

Os sinais mais uteis desta etapa sao:

- erro com guidance apontando para ingestion.content_profiles.type_specific.json;
- comportamento padrao de encoding, size ou quality quando a configuracao esperada nao foi encontrada;
- diferenca entre o contrato resolvido pelo builder e o perfil padrao ainda influenciado por json_processing_profile.

Em linguagem simples: se o slice JSON se comporta diferente do esperado logo no inicio, quase sempre vale revisar primeiro o contrato resolvido aqui.

## 8. Exemplo pratico guiado

Cenario: o tenant ainda usa ingestion.json no YAML.

1. O JsonMetadataBuilder inicia.
2. resolve_json_ingestion_config chama validate_json_yaml_contract.
3. O resolvedor encontra ingestion.json.
4. Levanta ValueError com guidance para usar ingestion.content_profiles.type_specific.json.
5. O pipeline para antes de tocar no arquivo.

O valor pratico desta etapa e impedir que uma configuracao errada contamine as etapas seguintes com comportamento ambigoo.

## 9. Evidencias no codigo

- src/utils/json_config_resolver.py
  - Simbolo relevante: validate_json_yaml_contract e resolve_json_ingestion_config
  - Comportamento confirmado: caminho canonico unico, merge com defaults e rejeicao explicita de caminhos legados.
- src/ingestion_layer/processors/json_metadata_builder.py
  - Simbolo relevante: inicializacao do builder
  - Comportamento confirmado: consumo do contrato resolvido e dos helpers especializados antes do restante do slice.
