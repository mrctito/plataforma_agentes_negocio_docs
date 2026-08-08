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
- `governanca`: chave tecnica da area **Gestao** (o rotulo visivel mudou em 2026-08; a chave e as URLs nao). Reune cliente/tenant, configuracao publicada, chaves de API, pessoas, segredos e canais de atendimento.
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

## Padrao das telas da area de Gestao (a partir de 2026-08)

A reorganizacao de 2026-08 fechou tres regras que valem para **toda** tela nova de Gestao. Elas nao
sao estilo: cada uma existe porque o desvio correspondente ja aconteceu e custou caro.

**1. O texto de operador tem dono unico: o catalogo de navegacao.** Cada entrada de
`app/ui/static/js/shared/admin-area-navigation-catalog.js` declara `cardTitle`, `cardSummary`,
`cardWhen` ("Use quando <situacao concreta>"), `cardRole`, `cardGroup` e `cardOrder`. O card do
indice e o cabecalho da tela leem **desse mesmo lugar** — a tela declara
`data-admin-page-description-source="catalog"` e nao escreve titulo nem subtitulo proprios. Antes, a
mesma tela tinha tres descricoes divergentes em tres arquivos.

**2. O indice de Gestao (`governanca_index.html`) agrupa por fluxo, nao por ordem alfabetica.** Os
grupos seguem a ordem do trabalho real: `apoio` ("Comece por aqui", onde vive o Guia da Plataforma),
`configuracao-do-cliente`, `identidade-e-acesso`, `canais-de-atendimento`. Tela nova entra num grupo
existente; grupo novo e decisao, nao efeito colateral.

**3. Telas de Gestao usam o perfil de contexto `gestao` do layout mestre**, e nunca os 3 cards das
telas operacionais:

```html
x-data="prometeuLayoutMestre({ perfilContexto: 'gestao', exigeChaveApi: true, usarAreaPadraoCentralizada: true })"
```

O perfil renderiza e-mail da sessao, o campo de X-API-Key **somente quando o router exige** (com o
motivo do 401 escrito na tela) e a superficie de `correlation_id` da ultima operacao. Card de YAML e
de payload **nao existem** nesse perfil — nao sao escondidos. `exigeChaveApi` segue o contrato real
do router: rota decorada com `Depends(require_permission(...))` ⇒ `true`; rota cookie-only ⇒ `false`,
e a tela mostra o selo "Somente login".

Consequencia pratica das tres regras: **nenhuma tela de Gestao monta cliente HTTP proprio.** Toda
chamada passa por `window.PrometeuAdminApiClient`, que e quem captura o `correlation_id` e o publica
na superficie do bloco. Tela com `fetch` cru nao e investigavel quando falha, e por isso e recusada
por contrato de teste.

### O que a reorganizacao mudou no inventario

- **Tela nova:** `ui-admin-gov-guia-plataforma.html` (Guia da Plataforma) — mapa interativo com os
  fluxos de trabalho passo a passo. E a unica tela da area que nao chama backend nenhum e, por isso,
  **nao monta o layout mestre**: sem chamada nao ha correlacao nem credencial para exibir, e o bloco
  ficaria com campos sem consumidor.
- **Tela removida:** `ui-admin-gov-human-grants.html` foi absorvida por
  `ui-admin-gov-memberships.html` ("Pessoas e Permissoes"). Convidar alguem e dizer o que essa pessoa
  pode fazer eram duas metades da mesma decisao em duas telas, e nada na primeira dizia que a segunda
  existia.
- **Fim das entradas duplicadas:** sete telas apareciam em duas familias do catalogo, uma delas com
  dois nomes diferentes. Cada tela agora e declarada uma unica vez; a area SaaS aponta para a de
  Gestao em vez de repetir as entradas.

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
