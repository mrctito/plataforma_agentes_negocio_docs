# Manual detalhado da etapa: Prontidao operacional do NL2YAML

## 1. O que esta etapa faz

Esta etapa verifica se o ambiente esta pronto para gerar o YAML final com seguranca. Ela existe para bloquear cedo quando faltam feature flag, identidade do operador, provider LLM estruturado ou catalogo efetivo de tools.

Em linguagem simples: e a inspecao de pista antes de permitir que a linha de montagem rode.

## 2. Onde ela entra no fluxo

No codigo lido, o preflight roda logo depois da resolucao do target dentro de objective_to_yaml. Se ele reprovar, draft, validate e confirm nem chegam a acontecer.

## 3. O que entra e o que sai

Entradas confirmadas:

- user_email
- resolved_target
- generation_mode
- base_yaml
- constraints
- status da feature flag

Saidas confirmadas:

- ready verdadeiro ou falso
- summary operacional
- checks estruturados por codigo, label e status
- diagnostics detalhados
- llm_provider e supports_structured_output
- catalog_size

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. o service resolve novamente o target efetivo para o preflight;
2. registra checks iniciais de autenticacao e permissao ja garantidas na borda HTTP;
3. valida se FEATURE_AGENTIC_AST_ENABLED esta ativa;
4. valida se user_email do operador foi informado;
5. trata o generation_mode, incluindo a politica de fallback heuristico explicito do modo auto;
6. se o modo for heuristic, marca que provider LLM estruturado nao e obrigatorio;
7. se houver configuracao llm valida, roda preflight especifico do gerador LLM e verifica structured output;
8. se o modo for auto com fallback autorizado, converte bloqueios de LLM em caminho pronto para heuristica;
9. resolve o catalogo efetivo de tools e verifica se ele nao esta vazio;
10. consolida checks, diagnostics e summary para decidir se o ambiente esta ready ou blocked.

O detalhe mais importante e que o preflight nao tenta produzir AST. Ele so confirma se existe base operacional suficiente para deixar o draft seguir.

## 5. Decisoes tecnicas importantes

### 5.1. Feature flag e requisito real, nao formalidade

Se FEATURE_AGENTIC_AST_ENABLED estiver desabilitada, o preflight falha com erro explicito. Isso impede UX ilusoria em ambiente nao liberado.

### 5.2. Modo auto so cai para heuristica com opt-in explicito

O backend nao inventa fallback heuristico por conveniencia. Ele exige constraints.auto_heuristic_fallback_enabled ou a feature flag dedicada para autorizar essa queda.

### 5.3. Catalogo efetivo e parte da prontidao

Mesmo com LLM pronto, o fluxo bloqueia se o catalogo de tools estiver vazio ou indisponivel. Isso reforca que NL2YAML monta contrato governado, e nao texto solto.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- feature flag desabilitada bloqueia tudo;
- user_email ausente quebra rastreabilidade e bloqueia o draft;
- llm raiz ausente ou invalido bloqueia llm_schema;
- provider sem structured output suficiente bloqueia geracao estruturada;
- catalogo efetivo vazio ou indisponivel bloqueia o fluxo;
- modo auto sem opt-in nao pode cair para heuristica quando o LLM falha.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- preflight_ready e preflight_summary na resposta;
- preflight_checks com status ready ou error;
- diagnostics como AST_PREFLIGHT_FEATURE_DESABILITADA, AST_PREFLIGHT_LLM_CONFIG_AUSENTE e AST_PREFLIGHT_CATALOGO_VAZIO;
- llm_provider retornado pelo preflight;
- catalog_size igual a zero ou ausente.

Em linguagem simples: quando o fluxo para antes mesmo do rascunho, e porque a pista operacional nao foi aprovada aqui.

## 8. Exemplo pratico guiado

Cenario: o operador pede generation_mode=llm_schema, mas o YAML base nao possui configuracao llm e o catalogo de tools esta vazio.

1. o preflight confirma autenticacao e permissao;
2. identifica ausencia de llm valido na raiz;
3. identifica catalogo vazio;
4. monta checks com status error;
5. devolve ready=false e summary informando quantos bloqueios precisam ser corrigidos.

O valor desta etapa e economizar tempo: a plataforma bloqueia antes de produzir preview enganoso.

## 9. Evidencias no codigo

- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: preflight
  - Comportamento confirmado: checagem de feature flag, user_email, LLM estruturado, fallback auto e catalogo efetivo.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: _build_preflight_summary
  - Comportamento confirmado: consolidacao de checks em resumo operacional legivel.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: objective_to_yaml
  - Comportamento confirmado: bloqueio em preflight encerra o fluxo unico com blocking_stage igual a preflight.
