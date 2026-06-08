# Manual detalhado da etapa: Validacao semantica do NL2YAML

## 1. O que esta etapa faz

Esta etapa verifica se a AST montada faz sentido para o target escolhido e para o YAML base governado. Ela existe porque sintaxe valida nao basta: um documento pode estar bem formado e ainda assim ser semanticamente errado, incoerente com o target ou em drift com o fragmento governado existente.

Em linguagem simples: e a etapa que decide se o rascunho realmente pode ser tratado como configuracao confiavel.

## 2. Onde ela entra no fluxo

No codigo lido, validate roda depois do draft ter produzido ast_draft sem perguntas pendentes e com relatorio inicial valido o bastante para seguir. No objective_to_yaml, se validate reprovar, o fluxo bloqueia com blocking_stage igual a validate.

## 3. O que entra e o que sai

Entradas confirmadas:

- target resolvido
- base_yaml
- ast_payload do draft
- strict verdadeiro

Saidas confirmadas:

- validation_report consolidado
- diagnostics de parse, drift e politica de base mista
- compiled_fragment
- success condicionado ao resultado semantico quando strict e verdadeiro

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. validate recebe target, base_yaml e ast_payload;
2. faz parse do payload AST para AgenticDocumentAST;
3. resolve o catalogo efetivo de tools para o target;
4. chama _validate_document para rodar a semantica correta daquele alvo;
5. detecta governed YAML drift no YAML base;
6. monta diagnostics de mixed base policy quando o YAML mistura escopos relevantes;
7. mergeia todos os diagnosticos num unico validation_report;
8. expõe o compiled_fragment que representa o recorte governado compilado.

O detalhe mais importante e que validate nao checa apenas a AST nova. Ele tambem observa o contexto do YAML base e a governanca do fragmento que ja existia antes.

## 5. Decisoes tecnicas importantes

### 5.1. Validacao depende do target real

Workflow e deepagent usam logicas semanticas diferentes. Isso impede que um documento passe so porque parece YAML valido genericamente.

### 5.2. Drift governado e parte da validacao

O sistema nao ignora divergencias entre o que o bloco governado deveria ser e o que o YAML base passou a conter. Isso fortalece governanca e detecta edicao manual perigosa.

### 5.3. Base mista gera diagnostico, nao silencio

Quando o YAML base mistura escopos de modo relevante, o service gera diagnosticos de politica aplicada. Isso ajuda a explicar por que certos merges ou validacoes se comportaram de determinado modo.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- ast_payload pode ser parseado com diagnosticos de erro;
- tool_catalog efetivo pode revelar tools invalidas ou ausentes para o target;
- drift governado pode tornar o documento semantica ou operacionalmente inseguro;
- politica de base mista pode introduzir warnings ou bloqueios relevantes;
- validate pode reprovar mesmo quando o draft parece visualmente convincente.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- validation_report.error_count e warning_count;
- diagnostics vindos de parse, drift e mixed base policy;
- compiled_fragment retornado por validate;
- blocking_stage igual a validate no objective_to_yaml;
- diff entre o que o preview mostrava e o que a validacao aceita semanticamente.

Em linguagem simples: se o rascunho parece bom, mas o sistema ainda bloqueia, a razao costuma estar nesta etapa.

## 8. Exemplo pratico guiado

Cenario: o draft gerou uma AST para supervisor, mas o YAML base foi alterado manualmente e o bloco governado entrou em drift.

1. validate parseia a AST do draft;
2. resolve o catalogo de tools efetivo;
3. valida a semantica do supervisor;
4. detecta drift no fragmento governado do YAML base;
5. consolida esse diagnostico no validation_report;
6. objective_to_yaml bloqueia em validate e devolve o relatorio ao operador.

O valor desta etapa e impedir que o pipeline trate divergencia manual como se fosse detalhe irrelevante.

## 9. Evidencias no codigo

- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: validate
  - Comportamento confirmado: parse da AST, validacao por target, drift governado e compiled_fragment consolidado.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: _detect_governed_yaml_drift
  - Comportamento confirmado: validacao inclui deteccao de divergencia no fragmento governado do YAML base.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: objective_to_yaml
  - Comportamento confirmado: reprovar validate interrompe o fluxo unico com blocking_stage igual a validate.
