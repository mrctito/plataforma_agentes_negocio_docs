# Plataforma de Agentes de Negócio — Documentação Pública

Este repositório reúne a documentação **conceitual**, **técnica**, os **guias e tutoriais**
de uso e integração, e a **metodologia de engenharia com IA** da Plataforma de Agentes de
Negócio: uma plataforma de Agentes de IA configurada por YAML, com suporte nativo a
AG-UI/Generative UI, RAG, workflows determinísticos e supervisão DeepAgent.

> Esta é a documentação aberta. O material comercial e o conteúdo interno de
> desenvolvimento não fazem parte deste repositório.

## 🚀 Comece por aqui: criar um chat usando a API

A **API HTTP é a principal porta de entrada da plataforma** — é por ela que você integra um
chat (Q&A/RAG, Agente, DeepAgent ou Workflow) à sua própria aplicação. Se esta é a sua
primeira vez, siga a trilha de onboarding de chat via API, nesta ordem:

1. **[Primeiros passos](usuario/GUIA-PRIMEIROS-PASSOS.md)** — do zero à primeira resposta real da API, com o menor número de passos.
2. **[Tutorial de chat ponta a ponta](usuario/TUTORIAL-CHAT-PLATAFORMA.md)** — monte um chat completo: os 4 modos de conversa, execução síncrona e assíncrona com polling, tratamento de erro e `correlation_id`.
3. **[Guia do integrador](usuario/GUIA-INTEGRADOR-CHAT-PLATAFORMA.md)** — construa uma interface própria conversando direto com a API, sem usar o componente de chat pronto.

Contrato HTTP de referência: **[endpoints da API (Swagger/OpenAPI)](tecnico/API-ENDPOINTS-SWAGGER.md)**.
Exemplos de integração em várias linguagens: **[exemplos de integração com a API](usuario/README-EXEMPLOS-INTEGRACAO-API.md)**.

## Como navegar

A documentação está organizada em quatro trilhas. Comece pela conceitual para entender **o
que** a plataforma faz e **por quê**; vá para a técnica quando precisar de **contratos,
arquitetura e detalhes de implementação**; use os guias e tutoriais para **colocar a mão na
massa** (integração, chat, tools, canais); e use a metodologia para entender **como** este
software é construído com apoio de IA.

### 1. Conceitual — o que é, por que existe, visão de produto

Explicações em nível introdutório de cada capacidade: RAG, ingestão, agentes e workflows,
DeepAgent, NL2SQL/NL2YAML, AG-UI, tools nativas, observabilidade, segurança e isolamento
por tenant. Ponto de partida recomendado:

- [conceitual/README-CONCEITUAL-ARQUITETURA-STACK-PROJETO.md](conceitual/README-CONCEITUAL-ARQUITETURA-STACK-PROJETO.md) — a visão geral da arquitetura e da stack.

Pasta completa: [`conceitual/`](conceitual/).

### 2. Técnico — arquitetura, contratos, schema e catálogo de tools

Detalhes de implementação, contratos de API, schema de banco, configuração YAML, designer
agentic (AST) e o catálogo de ferramentas (tools). Referências centrais:

- [tecnico/README-TECNICO-ARQUITETURA-STACK-PROJETO.md](tecnico/README-TECNICO-ARQUITETURA-STACK-PROJETO.md) — arquitetura técnica.
- [tecnico/API-ENDPOINTS-SWAGGER.md](tecnico/API-ENDPOINTS-SWAGGER.md) — endpoints da API.
- [tecnico/README-CONFIGURACAO-YAML.md](tecnico/README-CONFIGURACAO-YAML.md) — configuração por YAML.
- [tecnico/README-TOOLS-LIB.md](tecnico/README-TOOLS-LIB.md) — biblioteca de tools.

Pasta completa: [`tecnico/`](tecnico/).

### 3. Guias e tutoriais — uso e integração na prática

Passo a passo para integrar e operar a plataforma, em nível introdutório. É aqui que está o
onboarding de chat via API destacado no topo. Referências centrais:

- [usuario/GUIA-PRIMEIROS-PASSOS.md](usuario/GUIA-PRIMEIROS-PASSOS.md) — quickstart da integração de chat/API.
- [usuario/TUTORIAL-CHAT-PLATAFORMA.md](usuario/TUTORIAL-CHAT-PLATAFORMA.md) — chat funcional completo, ponta a ponta.
- [usuario/GUIA-INTEGRADOR-CHAT-PLATAFORMA.md](usuario/GUIA-INTEGRADOR-CHAT-PLATAFORMA.md) — UI própria sobre a API.
- [usuario/README-EXEMPLOS-INTEGRACAO-API.md](usuario/README-EXEMPLOS-INTEGRACAO-API.md) — exemplos de integração em várias linguagens.
- [usuario/GUIA-USUARIO-TOOLS.md](usuario/GUIA-USUARIO-TOOLS.md) — uso das tools nativas da plataforma.

Pasta completa: [`usuario/`](usuario/).

### 4. Metodologia de engenharia com IA

Como esta plataforma é desenvolvida com apoio de agentes de IA: princípios, lentes e base
conceitual da metodologia.

Pasta completa: [`metodologia/`](metodologia/).

## Diagramas

As imagens referenciadas pela documentação ficam em [`assets/diagrams/`](assets/diagrams/).
