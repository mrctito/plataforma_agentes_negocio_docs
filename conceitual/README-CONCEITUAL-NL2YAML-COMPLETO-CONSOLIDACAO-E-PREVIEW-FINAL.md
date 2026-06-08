# Manual detalhado da etapa: Consolidacao e preview final do NL2YAML

## 1. O que esta etapa faz

Esta etapa transforma a AST validada em YAML final revisavel, mas ainda sem persistir arquivo. Ela existe para separar confianca tecnica de publicacao efetiva: primeiro consolidar, mesclar, carimbar e mostrar diff; so depois, se o operador quiser, publicar.

Em linguagem simples: e o dry-run que monta o YAML de verdade sem ainda escrever no repositorio.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa acontece dentro de confirm com apply=false no objective_to_yaml. Se ela falhar, o fluxo bloqueia em confirm. Se der certo, devolve final_yaml, final_yaml_text e diff_preview.

## 3. O que entra e o que sai

Entradas confirmadas:

- base_yaml
- ast_payload validado
- answers opcionais
- apply falso no fluxo objective_to_yaml
- force falso no fluxo objective_to_yaml

Saidas confirmadas:

- final_yaml
- final_yaml_text
- diff_preview
- validation_report consolidado
- diagnostics combinados de parse, validacao, drift e politica de base mista

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. confirm resolve correlation_id e target do payload;
2. parseia a AST novamente para garantir consistencia na fase final;
3. aplica answers quando existirem;
4. revalida o documento contra target, base_yaml e catalogo efetivo;
5. soma drift e mixed base policy aos diagnosticos finais;
6. se houver erro e force estiver desligado, bloqueia a confirmacao;
7. seleciona o fragmento governado final;
8. faz merge com o YAML base usando o compilador do documento;
9. estampa fingerprint do bloco governado no YAML final;
10. gera diff_preview entre before e after;
11. objective_to_yaml usa esse confirm em dry-run para devolver o material final ao operador.

O detalhe mais importante e que objective_to_yaml nao chama uma API separada de renderizacao. Ele reutiliza a mesma confirmacao canônica do assembly, so que em modo sem persistencia.

## 5. Decisoes tecnicas importantes

### 5.1. Confirm em dry-run reaproveita o caminho real de consolidacao

Isso reduz risco de divergencia entre preview e publicacao, porque o mesmo nucleo de confirmacao monta o artefato final.

### 5.2. Fingerprint governado protege o recorte AST

Ao carimbar o bloco governado, o sistema prepara a base para detectar drift em rodadas futuras.

Como esse assunto cresceu e passou a incluir interface oficial, helper público e CLI operacional, o detalhamento completo do hash governado agora está centralizado em:

- [docs/README-CONCEITUAL-HASH-GOVERNADO-AGENTIC.md](README-CONCEITUAL-HASH-GOVERNADO-AGENTIC.md)
- [docs/README-TECNICO-HASH-GOVERNADO-AGENTIC.md](../tecnico/README-TECNICO-HASH-GOVERNADO-AGENTIC.md)

### 5.3. Diff preview faz parte do produto, nao adorno visual

O diff mostra o impacto pratico no YAML antes de qualquer persistencia. Isso ajuda revisão humana e reduz publicacao cega.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- confirm pode reprovar por erros semanticos remanescentes quando force=false;
- final_yaml pode nao ser produzido e bloquear o fluxo em confirm;
- answers mal aplicadas podem reintroduzir invalidade no documento;
- diff_preview vazio nao significa ausencia de validacao, so pode significar merge sem alteracao efetiva.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- blocking_stage igual a confirm;
- diagnostics com codigo AST_OBJECTIVE_TO_YAML_CONFIRM_BLOQUEADO quando o YAML final nao se fecha;
- validation_report do confirm com error_count e warning_count;
- final_yaml_text e diff_preview quando o dry-run fecha corretamente;
- chosen_tools coletadas do final_yaml consolidado.

Em linguagem simples: quando o draft parece bom e a validacao passou, mas o YAML final ainda nao sai, a falha real esta nesta consolidacao.

## 8. Exemplo pratico guiado

Cenario: o draft e validate passaram, mas o operador quer revisar exatamente o que mudaria no YAML base.

1. objective_to_yaml chama confirm com apply=false e force=false;
2. confirm recompila o fragmento governado final;
3. faz merge com o YAML base;
4. estampa o fingerprint do bloco governado;
5. devolve final_yaml, final_yaml_text e diff_preview;
6. a UI usa esse material para liberar ou nao a publicacao.

O valor desta etapa e provar que o preview final nao e mock. Ele vem do mesmo mecanismo que sera usado na aplicacao real.

Em outras palavras: o preview final do NL2YAML e o mesmo trilho que mais tarde pode ser usado para sincronizar o hash governado de um YAML ja existente. A diferenca e o contexto operacional, nao a origem canonica do artefato.

## 9. Evidencias no codigo

- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: confirm
  - Comportamento confirmado: recompila AST, faz merge, estampa hash governado, gera diff e pode aplicar ou nao em arquivo.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: rotina de hash governado e servico de diff preview
  - Comportamento confirmado: consolidacao final inclui fingerprint do fragmento governado e diff preview.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: objective_to_yaml
  - Comportamento confirmado: usa confirm em dry-run com apply=false e force=false para produzir o preview final.
