# Chat Q&A/RAG — como capturar as fontes e exibir na sua UI

> Para quem está construindo um chat próprio sobre a plataforma. Explica o contrato das fontes
> devolvidas pelo Q&A/RAG, por que a citação **não** vem no meio do texto e como montar o bloco
> de fontes no final da resposta.

## 1. A regra que muda tudo: o modelo não cita, você exibe

A resposta do modelo vem **limpa**, sem referência no corpo do texto. Isso é imposto pelas regras
de prompt da plataforma (`src/shared/prompts/default_prompts.py`, regra 14), que proíbem o modelo
de inserir código de norma, nome de documento, número de página ou marcador entre colchetes ao
longo dos parágrafos — e também de escrever a própria seção "Fontes".

O motivo é prático: referência no meio do texto polui a leitura e, pior, o modelo numerava as
fontes pela posição em que apareciam no prompt ("Fonte 1", "Fonte 2"), o que não identifica nada
para quem lê. Quem tem a identidade real de cada documento é a plataforma, não o modelo.

Consequência para a sua UI: **relacionar as fontes no fim da resposta é responsabilidade do
cliente**, a partir do array `sources`. Se a sua UI não fizer isso, o usuário vê a resposta sem
nenhuma referência.

## 2. O contrato: `sources`

A resposta do `ask` traz `sources: list[dict]` (`src/api/schemas/rag_models.py`), **uma entrada por
documento** que sustentou a resposta. Os campos que interessam para exibição:

| Campo | O que é | Sempre presente? |
|---|---|---|
| `reference` | **Referência pronta para exibir**: nome do documento + código da norma/manual + páginas, já formatada | Sim |
| `title` / `document_title` | Nome do documento, desambiguado quando dois documentos diferentes têm o mesmo nome | Sim |
| `codigo_norma` | Código da norma (ex.: `DNIT-TER 198_2021`) | Só quando o documento tem |
| `codigo_manual` | Código do manual (ex.: `IPR 715`) | Só quando o documento tem |
| `pages` | **Todas** as páginas daquele documento que sustentaram a resposta, sem repetição e em ordem crescente (ex.: `["4", "5", "6"]`) | Quando o documento é paginado |
| `page` | A **primeira** página da lista acima. Existe para consumidores que leem um valor escalar | Quando o documento é paginado |
| `page_url` | Caminho/URL de origem do documento | Quando disponível |
| `relevance_score` | Score do **melhor** trecho recuperado daquele documento | Sim |
| `content_preview` | Trecho do conteúdo (200–300 caracteres) do melhor trecho | Sim |

### Uma linha por documento, não por trecho

O RAG recupera **trechos**, e vários trechos costumam vir do mesmo PDF. O backend agrupa esses
trechos por documento antes de devolver `sources`: cinco trechos de dois PDFs viram duas entradas,
cada uma com as páginas consolidadas.

```text
DNIT 161_2022 ... .pdf | Norma DNIT-EM 161_2022 | p. 4, 5, 6
DNIT 380_2022 ... .pdf | Norma DNIT 380_2022 | p. 6
```

O agrupamento é por identidade do documento — `document_id` e, quando ele falta, o caminho de
origem. Dois documentos **diferentes** com o mesmo nome de arquivo continuam sendo duas entradas,
desambiguadas pelo id (`Relatorio.pdf (doc-2)`) ou por sufixo numérico (`Relatorio.pdf #2`).

Consequência prática para sua UI: **não deduplique nem agrupe fontes do seu lado**. Se você conta
linhas de fonte para exibir "N documentos consultados", esse número agora é de documentos, não de
trechos.

### Use `reference` — não recomponha a string

`reference` é montada no backend pelo **mesmo** código que rotula o contexto enviado ao modelo
(`GenerationEngine._build_document_reference`). Ela já resolve as regras chatas:

- ordem: **nome do documento → código da norma → código do manual → páginas**;
- omite o que não existe, sem deixar separador solto;
- junta as páginas do mesmo documento numa lista só, ordenada e sem repetição;
- preserva a desambiguação quando dois documentos distintos têm o mesmo nome.

Recompor isso no seu front duplica uma regra de domínio numa segunda linguagem — quando a regra
mudar no backend, sua UI fica divergente e ninguém percebe.

Formato resultante — com código, sem código e com várias páginas:

```text
715_manual_de_hidrologia_basica.pdf | Manual IPR 715 | p. 15
GUIA_DE_ANALISE_2018_VERSAO_BETA.pdf | p. 89
DNIT 161_2022 – EM – Geocompostos drenantes.pdf | Norma DNIT-EM 161_2022 | p. 4, 5, 6
```

### Documento sem código é normal, não é erro

Parte do acervo não tem código porque o PDF não tem o arquivo `.txt` de metadados (sidecar) ao
lado. Nesse caso a fonte exibe só o que existe — nome e página. **Não invente, não deduza e não
complete código ausente**: a citação precisa ser conferível contra o documento real.

## 3. Exemplo mínimo de exibição

O helper oficial do projeto está em
`app/ui/static/js/shared/ui-webchat-runtime-utils.js` (`buildQaSourcesSummary` /
`composeQaModeMessage`). O essencial cabe em poucas linhas:

```js
function montarBlocoDeFontes(sources, limite = 5) {
  if (!Array.isArray(sources) || sources.length === 0) return '';
  const linhas = sources.slice(0, limite).map(function (src, i) {
    // `reference` vem pronta do backend; o título é só o plano B.
    const rotulo = (src && typeof src.reference === 'string' && src.reference.trim())
      || src.title
      || src.document_title
      || 'Documento';
    return (i + 1) + '. ' + rotulo;
  });
  return '\n\nFontes:\n' + linhas.join('\n');
}

const mensagemFinal = resposta.answer + montarBlocoDeFontes(resposta.sources);
```

Quantas fontes exibir é escolha sua, mas o backend já limita o array pela chave YAML
`qa_system.formatting.max_source_references` (ver §5).

Para uma UI mais rica, use os campos individuais em vez da string: `codigo_norma`/`codigo_manual`
viram um selo, `page` vira âncora de navegação, `content_preview` vira tooltip do trecho.

## 4. Armadilha comum: JS servido do cache

As páginas do projeto versionam o script com query string
(`ui-webchat-runtime-utils.js?v=20260728_1`). Ao alterar o JS **sem** bumpar esse `?v=`, o
navegador continua servindo a versão antiga e a mudança parece não ter efeito — inclusive a
exibição das fontes. Se o comportamento não mudar depois de uma alteração no front, confirme o
que está sendo realmente servido antes de procurar o problema no backend:

```bash
curl -s http://localhost:5555/ui/static/js/shared/ui-webchat-runtime-utils.js | grep reference
```

## 5. Quantas fontes vêm na resposta

`qa_system.formatting.max_source_references` (YAML) define o limite. Ele governa **duas** coisas
de propósito: quantos trechos entram no contexto do modelo e quantos trechos alimentam a lista de
fontes. Manter os dois iguais evita a resposta citar N documentos enquanto a UI exibe N+k —
inclusive fontes que o modelo nunca leu.

Atenção: o limite corta **trechos**, e a lista final é agrupada por documento (§2). Com o limite em
5, cinco trechos do mesmo PDF viram **uma** entrada em `sources`. Ou seja, a lista devolvida pode
ser bem menor que o limite — e isso é o comportamento correto, não perda de fonte.

Para conferir esse efeito no log da execução, o passo `generation:sources_selected` registra
`chunks_considered` (trechos que entraram), `sources_selected` (fontes devolvidas),
`documents_grouped` (documentos que tinham mais de um trecho) e `chunks_merged` (quantos trechos
foram absorvidos pelo agrupamento).

## 6. Como conferir o metadado de um documento

Para saber se um PDF tem mesmo `codigo_norma`/`codigo_manual` no acervo, consulte a fonte de
verdade em vez de deduzir pela resposta:

```bash
python .claude/scripts/qdrant/inspect_document_metadata.py \
    --tenant <tenant> --vectorstore <vectorstore_id> --document <parte-do-nome> --qdrant
```

Detalhes e demais ferramentas de acesso: `.claude/rules/ferramentas-acesso-dados.md`.

## Referências no código

- Contrato da resposta: `src/api/schemas/rag_models.py`
- Composição da referência e das fontes: `src/qa_layer/rag_engine/generation_engine.py`
  (`_build_document_reference`, `_format_intelligent_sources`, `_reduce_source_candidates`,
  `_build_source_identity_key`)
- Regras de prompt (proibição de citar no corpo): `src/shared/prompts/default_prompts.py`
- Helper de exibição: `app/ui/static/js/shared/ui-webchat-runtime-utils.js`
