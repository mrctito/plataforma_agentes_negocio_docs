# Manual detalhado da etapa: Publicacao segura do NL2YAML

## 1. O que esta etapa faz

Esta etapa grava o YAML final no repositorio governado quando e somente quando a publicacao foi solicitada explicitamente. Ela existe para impedir que o fluxo unico objective_to_yaml publique arquivo automaticamente e para limitar a escrita ao perimetro oficial do produto.

Em linguagem simples: e a porta controlada de saida da esteira, separada do preview.

## 2. Onde ela entra no fluxo

No codigo lido, essa etapa nao acontece dentro do objective_to_yaml. Ela so ocorre no confirm quando apply=true. O fluxo unico para em dry-run; a persistencia real depende de uma chamada posterior de confirmacao.

## 3. O que entra e o que sai

Entradas confirmadas:

- final_yaml ja consolidado
- apply=true
- output_path explicito
- force opcional

Saidas confirmadas:

- arquivo YAML salvo em caminho governado
- saved_path formatado
- logs de normalizacao e aplicacao do output_path

## 4. Como o codigo implementa a etapa

O fluxo real segue esta ordem.

1. confirm executa toda a consolidacao final do documento;
2. se apply=false, encerra sem gravar nada;
3. se apply=true e output_path estiver ausente, levanta erro;
4. normaliza o output_path pedido pelo operador;
5. restringe esse caminho ao perimetro governado de app/yaml e a sufixos .yaml ou .yml;
6. salva o documento final usando o editor canônico;
7. devolve saved_path formatado para a interface.

O detalhe mais importante e que publicacao aqui nao e um side effect inevitavel do NL2YAML. Ela e um segundo ato, com controle proprio e restricao de caminho.

## 5. Decisoes tecnicas importantes

### 5.1. Objective-to-yaml nunca publica sozinho

Isso separa criacao de preview e gravacao em arquivo. O ganho pratico e permitir revisão e governanca antes de tocar o repositorio.

### 5.2. Output path nao e arbitrario

O caminho precisa cair dentro de app/yaml. Isso evita bypass do perimetro governado e gravacao em qualquer lugar do workspace.

### 5.3. Sufixo do arquivo tambem e validado

Aceitar apenas .yaml e .yml reforca que a publicacao continua sendo de contrato YAML, nao dump generico de qualquer formato.

## 6. O que pode dar errado

Os limites e riscos confirmados sao estes.

- apply=true sem output_path gera erro imediato;
- output_path fora de app/yaml deve ser rejeitado pela normalizacao canônica;
- sufixo fora de .yaml ou .yml nao pertence ao contrato de publicacao;
- force altera a postura diante de erros, mas nao elimina a governanca do caminho.

## 7. Como diagnosticar

Os sinais mais uteis sao:

- absence de saved_path no objective_to_yaml, o que e esperado porque ele nao publica;
- logs de AST confirm normalizou output_path e AST confirm aplicado em arquivo;
- erro explicito pedindo output_path quando apply=true;
- saved_path retornado pelo confirm em sucesso.

Em linguagem simples: se alguem disser que NL2YAML gravou arquivo sozinho, o codigo lido contradiz isso.

## 8. Exemplo pratico guiado

Cenario: o operador revisou o preview final e agora quer publicar o YAML.

1. a UI envia confirm com apply=true;
2. informa output_path dentro de app/yaml;
3. o service normaliza o caminho e valida o sufixo;
4. salva o YAML final;
5. devolve saved_path para a interface.

O valor desta etapa e deixar claro que publicar e uma decisao operacional separada da geracao do preview.

## 9. Evidencias no codigo

- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: confirm
  - Comportamento confirmado: so persiste quando apply=true e output_path foi informado.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: _normalize_output_path_for_apply
  - Comportamento confirmado: publicacao e restringida ao perimetro governado e a caminhos YAML validos.
- src/config/agentic_assembly/assembly_service.py
  - Simbolo relevante: _save_final_yaml
  - Comportamento confirmado: gravacao do YAML final usa o caminho canônico de persistencia.
- app/ui/static/js/objective-yaml-studio.js
  - Simbolo relevante: previewReadyForPublish e publishForm
  - Comportamento confirmado: a interface separa preview e publicacao como acoes distintas.
