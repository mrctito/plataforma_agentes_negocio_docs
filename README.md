# Plataforma de Agentes de Negócio — Documentação Pública

Este repositório reúne a documentação **conceitual**, **técnica** e a **metodologia de
engenharia com IA** da Plataforma de Agentes de Negócio: uma plataforma de Agentes de IA
configurada por YAML, com suporte nativo a AG-UI/Generative UI, RAG, workflows
determinísticos e supervisão DeepAgent.

> Esta é a documentação aberta. O material comercial, os guias de uso final e o conteúdo
> interno de desenvolvimento não fazem parte deste repositório.

## Como navegar

A documentação está organizada em três trilhas. Comece pela conceitual para entender **o
que** a plataforma faz e **por quê**; vá para a técnica quando precisar de **contratos,
arquitetura e detalhes de implementação**; use a metodologia para entender **como** este
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

### 3. Metodologia de engenharia com IA

Como esta plataforma é desenvolvida com apoio de agentes de IA: princípios, lentes e base
conceitual da metodologia.

Pasta completa: [`metodologia/`](metodologia/).

## Diagramas

As imagens referenciadas pela documentação ficam em [`assets/diagrams/`](assets/diagrams/).
