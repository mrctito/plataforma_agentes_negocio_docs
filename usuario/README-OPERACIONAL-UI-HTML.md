# Padrao operacional de UI HTML

Este documento explica o padrao atual para paginas HTML estaticas da plataforma. Ele serve para evitar que novas telas voltem ao modelo antigo de CSS inline, scripts soltos e aparencia diferente entre areas do produto.

Em linguagem simples: uma pagina pode ter conteudo proprio, mas nao deve parecer um sistema diferente. O usuario precisa reconhecer a plataforma, entender onde esta e conseguir usar formularios, botoes, tabelas e mensagens sem surpresa visual.

## Quando usar este documento

Use este guia antes de criar ou alterar paginas em `app/ui/static` ou templates HTML em `templates`.

Use tambem quando uma tela parecer antiga, destoante ou sem classificacao clara no inventario visual.

Este documento nao muda contratos de backend. Ele nao autoriza alterar endpoints, payloads, autenticacao, `correlation_id`, storage, cache ou regra de negocio.

## Familias visuais

As paginas operacionais administrativas usam `data-admin-shell-family` para declarar a familia visual.

As familias atuais protegidas por teste sao:

- `plataforma`: telas administrativas e operacionais da plataforma.
- `governanca`: telas de tenants, usuarios, permissoes e provisionamento governado.
- `dnit`: telas do dominio DNIT, com identidade propria mas alinhadas aos tokens business.

Quando uma pagina nao usa `data-admin-shell-family`, ela precisa estar classificada no contrato de inventario. Isso evita uma situacao ruim: ninguem sabe se a tela ficou fora do padrao por decisao consciente ou por esquecimento.

As excecoes conhecidas ficam em categorias explicitas:

- `auth`: entrada/autenticacao.
- `hub`: hubs e centros de conta.
- `legal`: termos e politica de privacidade.
- `active`: pagina ativa que nao usa shell administrativo, como revisao HIL segura.
- `template`: arquivos usados como base ou exemplo tecnico.
- `demo`: demonstracoes controladas.
- `legacyCompatibility`: legado mantido por compatibilidade, sem promocao automatica para produto ativo.

## Tokens e CSS compartilhados

O padrao visual deve reaproveitar os CSS compartilhados existentes.

Arquivos principais:

- `app/ui/static/css/plataforma-agentes-ia-design-system.css`: tokens globais, incluindo tokens business de superficie, borda, raio, sombra e foco.
- `app/ui/static/css/admin-theme.css`: componentes administrativos compartilhados, como cards, formularios, tabelas, status e paineis de log.
- CSS de familia ou pagina: usado quando a tela tem identidade propria, como DNIT ou HIL.

Regra pratica: primeiro procure uma classe ou token compartilhado. Crie CSS especifico de pagina apenas quando a necessidade for realmente propria daquela tela.

## O que evitar

Nao crie `<style>` inline para componentes padrao.

Nao crie `<script>` inline para logica de pagina quando a logica pode ficar em arquivo `.js` versionado.

Nao use gradientes fortes, sombras pesadas ou raios exagerados em telas operacionais. Essas telas sao de trabalho: precisam ser legiveis, sobrias e previsiveis.

Nao use `letter-spacing` negativo. Isso costuma piorar leitura e pode quebrar em telas pequenas.

Nao crie fallback visual ou funcional escondido. Se um asset obrigatorio nao carregar, o problema deve aparecer em teste ou auditoria.

## Templates, demos e legados

Nem todo HTML antigo deve virar tela ativa.

Antes de remover ou migrar um arquivo, confirme uso por busca de links, imports e testes.

Classificacao atual de referencia:

- `hil-approval-review.html`: ativo. E usado por fluxo HIL e possui testes dedicados.
- `ui-chat-demo.html`: demo.
- `_layout-mestre.html`, `_layout-mestre-v1.html` e `_template-bootstrap-alpine.html`: templates internos.
- `templates/ag-ui-official-third-party/frontend/index.html`: template AG-UI para terceiros.
- `index-v1.html` e `index-v2.html`: legado mantido por compatibilidade, sem remocao nesta rodada.

Remocao de legado exige tarefa propria e prova de nao uso. Sem prova, manter classificado e protegido por teste.

## Cache-busting

Quando alterar CSS ou JavaScript carregado por HTML estatico, atualize a query de versao do asset quando ela existir.

Exemplo pratico:

```html
<link rel="stylesheet" href="/ui/static/css/dnit-notebooklm.css?v=20260512_1">
```

Isso evita que o navegador continue usando o arquivo antigo em cache e faca a auditoria visual parecer inconsistente.

## Auditoria visual obrigatoria

Teste estatico nao substitui browser real.

Para paginas HTML alteradas, abra a pagina em desktop e mobile. Verifique:

- status HTTP dos HTMLs e CSS alterados;
- ausencia de overflow horizontal;
- cabecalho e navegacao legiveis;
- formularios com labels visiveis;
- botoes com foco/hover perceptiveis;
- tabelas, logs e cards sem sobreposicao;
- console sem erro novo relacionado aos assets alterados.

Para auditoria apenas visual de paginas estaticas, subir somente a API com `./run.sh +a` pode ser suficiente. Evite iniciar worker quando a pagina nao depende dele, porque worker pode consumir fila real e gerar efeito operacional fora do escopo visual.

Quando a validacao funcional realmente depender de worker, use o fluxo completo previsto para a funcionalidade e registre logs com cuidado.

## Testes automatizados relacionados

Os contratos principais ficam em:

- `tests/frontend/html_visual_inventory_contract.test.js`.
- `tests/frontend/admin_family_visual_contract.test.js`.
- Testes especificos de pagina, como contratos DNIT e HIL.

Durante desenvolvimento, use a suite oficial focada com `.venv`:

```bash
source .venv/bin/activate && npx jest tests/frontend/html_visual_inventory_contract.test.js tests/frontend/admin_family_visual_contract.test.js
```

Quando houver CSS ou JS frontend relevante, rode tambem:

```bash
source .venv/bin/activate && python suite_de_testes_padrao.py --frontend --run-id frontend-contratos-html
```

Esse e o gate oficial do slice visual no boundary da suite. Em termos simples:
para HTML, CSS e JavaScript de UI, a prova principal deve nascer de `--frontend`,
e nao de heuristica por pasta ou tentativa de reaproveitar `--focus-tests` para
arquivos que nao sao alvos pytest.

No fechamento normal de plano amplo, use `--complete` como gate amplo padrao. Se precisar de um fechamento mais barato sem integration e sem slow, use `--complete-lite`. Evite `--complete-full` porque ele e muito lento e pesado. Se a mudanca tocar a UI, complemente com `--frontend` apenas quando precisar de evidencia dedicada do slice visual. Se tambem tocar backend compartilhado, complemente com `--backend` apenas quando houver necessidade tecnica clara de um gate intermediario adicional.

## Checklist antes de concluir uma mudanca

- A pagina usa familia ou excecao classificada.
- O CSS comum foi reaproveitado antes de criar regra nova.
- Nao houve mudanca em endpoint, payload, autenticacao, storage, cache ou `correlation_id`.
- CSS/JS alterado tem cache-busting quando aplicavel.
- Testes focados passaram.
- Auditoria visual desktop/mobile foi feita.
- Erros reais vistos em terminal/log foram registrados no backlog quando forem erros de produto.
