# Manual técnico por etapa: publicação segura do NL2YAML

## 1. O que esta etapa cobre

Esta etapa cobre o confirm com apply verdadeiro, que é o único caminho que realmente grava um arquivo YAML no repositório.

## 2. Separação crítica do fluxo

O ponto mais importante confirmado no código é este:

- objective to yaml não grava arquivo
- quem grava arquivo é confirm com apply igual a true

Essa separação evita que um preview governado vire persistência automática por acidente.

## 3. Regras de publicação confirmadas

Quando apply é verdadeiro, o serviço exige:

- output_path obrigatório
- output_path com sufixo yaml ou yml
- normalização do caminho antes da gravação
- permanência do arquivo dentro de app/yaml

Se qualquer uma dessas condições falhar, o serviço levanta ValueError explícito em vez de escrever fora do perímetro governado.

## 4. O que o confirm faz antes de gravar

Mesmo em apply verdadeiro, o serviço não pula a governança. Ele ainda:

- resolve target real do payload
- aplica answers pendentes quando existem
- parseia a AST novamente
- valida semanticamente o documento
- detecta drift e política de base mista
- monta fragmento governado
- faz merge canônico no YAML base
- gera diff preview

Só depois disso ele persiste o arquivo.

## 5. O papel do force

Quando há erros consolidados, confirm bloqueia a publicação se force estiver falso. Isso impede que a camada de persistência seja usada como atalho para gravar YAML tecnicamente inválido.

## 6. Diagnóstico recomendado

Quando a publicação falhar:

1. confira se apply estava realmente true
2. valide output_path e seu sufixo
3. confirme que o caminho continua dentro de app/yaml
4. verifique se validation_report trouxe erros bloqueantes
5. só use force quando o risco estiver entendido e justificado

## 7. Evidências no código

- src/config/agentic_assembly/assembly_service.py
  - Motivo: confirm e aplicação em arquivo.
  - Comportamento confirmado: output_path é obrigatório quando apply é true e a escrita é limitada ao perímetro governado.
- src/config/agentic_assembly/assembly_service.py
  - Motivo: proteção contra persistência indevida.
  - Comportamento confirmado: objective to yaml consolida preview, mas não publica arquivo automaticamente.
