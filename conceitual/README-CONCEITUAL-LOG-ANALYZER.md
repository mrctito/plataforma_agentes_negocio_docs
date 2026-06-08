# Log Analyzer — Visão Conceitual, Executiva, Comercial e Estratégica

## 1. O que é esta feature

O **Log Analyzer** é o módulo oficial de análise estruturada de logs JSONL da plataforma.
Ele substitui completamente o módulo legado `src.analysis` e é a única forma correta de
consumir e extrair inteligência dos logs de execução do sistema.

O módulo oferece dois modos de operação com propósitos diferentes:

- **Análise profunda** (`LogAnalyzerService`): carrega todos os registros de um log em
  memória via Pandas, executa 4 plugins analíticos e retorna um relatório completo com
  métricas de erros, performance, eventos e atividade por componente.

- **Consulta rápida** (`LogQueryService`): lê o log registro a registro via iterador e
  para assim que a resposta puder ser determinada, sem carregar nada em memória. Responde
  a 17 tipos de perguntas operacionais como "houve erro?", "qual o status do fluxo?",
  "o processo está travado?".

Os dois modos compartilham a mesma camada de localização de arquivos e o mesmo contrato
de rastreabilidade por `correlation_id`, mas são pipelines independentes, com engines
diferentes, para necessidades diferentes.

## 2. Que problema ela resolve

Sem o Log Analyzer, diagnosticar uma execução exige:

- Abrir a pasta `/logs` manualmente
- Localizar o arquivo certo por nome ou data
- Parsear JSONL linha a linha sem estrutura
- Montar manualmente a narrativa do que aconteceu
- Não ter como perguntar "houve erro?" de forma programática

Com volumes de logs crescentes (dezenas de milhares de arquivos), isso se torna
inviável operacionalmente e impossível para agentes de IA, ferramentas de suporte
e automações de monitoramento.

O Log Analyzer encapsula toda essa complexidade:

- **Localiza automaticamente** os arquivos certos pelo `correlation_id`, consultando
  o manifest persistente e o leitor canônico de logs
- **Protege contra OOM** com limite de registros configurável
- **Separa análise profunda de consulta operacional** para que a pergunta "houve erro?"
  não precise carregar 50 mil registros em memória
- **Expõe contratos tipados** que podem ser consumidos por API, CLI, MCP e agentes de IA
- **Nunca levanta exceção** para o chamador — falhas são estruturadas dentro do resultado

## 3. Visão executiva

O Log Analyzer é a infraestrutura de observabilidade que torna o diagnóstico de execuções
reproduzível, auditável e automatizável.

Sem ele, cada investigação de falha depende de conhecimento manual sobre a estrutura dos
arquivos, o formato JSONL e os padrões de nomeação. Com ele, qualquer sistema — humano
ou automatizado — pode responder a perguntas operacionais críticas em segundos.

**Impacto direto:**

- Reduz o tempo médio de diagnóstico de falhas em produção
- Elimina dependência de acesso manual à pasta de logs
- Permite que agentes de IA e ferramentas de suporte diagnostiquem execuções sem
  intervenção humana
- Cria base para alertas automáticos baseados em análise de log
- Garante que observabilidade seja um contrato de produto, não uma prática ad hoc

## 4. Visão comercial

Para clientes e equipes de suporte, o Log Analyzer resolve a pergunta mais comum em
toda operação de produto baseada em IA: **"o que aconteceu nessa execução?"**

Antes, responder essa pergunta exigia acesso ao servidor, conhecimento técnico de JSONL
e tempo de investigação manual. Com o Log Analyzer:

- O suporte pode consultar o status de uma execução via API sem acessar o servidor
- O cliente pode receber uma explicação estruturada do que aconteceu na sua solicitação
- Agentes de IA do próprio produto podem diagnosticar suas próprias execuções e se
  autocorrigir com base nos logs
- Ferramentas de monitoramento podem detectar padrões de erro sem análise humana

**Diferencial:** A consulta rápida (fastquery) responde perguntas operacionais com
parada antecipada — não carrega o log inteiro, para na primeira evidência relevante.
Isso torna viável consultar logs grandes em tempo real sem degradar performance.

## 5. Visão estratégica

O Log Analyzer é uma peça de infraestrutura que fortalece a plataforma em três eixos:

**Arquitetura observável por design:** logs JSONL estruturados são inúteis sem uma
camada de consulta. O Log Analyzer transforma logs em ativos consultáveis, criando a
base para dashboards, alertas, auditoria e análise forense.

**Plataforma agentic auto-diagnóstica:** com o LogQueryService exposto via MCP e API,
agentes de IA da própria plataforma podem consultar seus próprios logs de execução,
detectar falhas e reportar status sem intervenção humana. Isso é um pré-requisito para
automações de recuperação e loops de auto-melhoria.

**Governança e auditabilidade:** toda execução rastreada por `correlation_id` pode ser
auditada de forma estruturada. O Log Analyzer garante que a trilha de auditoria seja
acessível programaticamente, não apenas como texto em disco.

## 6. Conceitos necessários para entender

### correlation_id

Identificador oficial de rastreabilidade de um processo de execução. É o elo que conecta
todos os registros de log de uma execução — da entrada HTTP até o último componente chamado.
O Log Analyzer usa o `correlation_id` como chave primária para localizar arquivos e executar
análises. Sem `correlation_id`, não há análise individual possível.

### JSONL

Formato de log do sistema: cada linha do arquivo é um objeto JSON independente. Permite
leitura streaming linha a linha sem carregar o arquivo inteiro na memória. O Log Analyzer
lê exclusivamente JSONL — não parseia texto livre, não interpreta logs não-estruturados.

### Análise profunda vs. consulta rápida

São dois contratos diferentes para duas necessidades diferentes:

- **Análise profunda:** carrega tudo em memória, usa Pandas, executa múltiplos plugins,
  retorna relatório completo. Custo: mais lenta, mais memória. Uso: diagnóstico detalhado,
  relatórios, UI.

- **Consulta rápida (fastquery):** itera registro a registro, para na primeira resposta,
  sem Pandas, sem carregamento em memória. Custo: mínimo. Uso: perguntas operacionais,
  monitoramento, agentes de IA, health checks automatizados.

### Plugin

Componente analítico que recebe os dados já carregados e executa uma análise específica
via dialeto da engine. Cada plugin produz um `dict` com métricas. Os plugins não conhecem
Pandas — só conversam com o dialeto, o que permite trocar a engine sem tocar nos plugins.

### Dialeto

Interface que abstrai as operações de dados (filtrar, contar, agrupar) de forma agnóstica
ao framework concreto (Pandas, Polars, DuckDB). Os plugins fazem perguntas ao dialeto, o
dialeto executa sobre o Pandas (ou outro framework futuro).

### AnalysisResult vs. FastLogQueryResult

São os dois contratos de saída do módulo:

- `AnalysisResult`: resultado da análise profunda. Contém `analyses` (dict indexado por
  `analysis_id`), metadados de execução e erro estruturado.

- `FastLogQueryResult`: resultado da consulta rápida. Contém `answer` (bool/int/str),
  `code` (código estável programático), `summary` (texto legível), `evidence` (registros
  de evidência).

Ambos retornam `success=True/False` e nunca levantam exceção para o chamador.

## 7. Decisões técnicas e trade-offs

**Por que dois serviços separados?**
`LogAnalyzerService` e `LogQueryService` existem separados porque têm pipelines,
engines e contratos diferentes. Unificá-los criaria uma classe com dois modos de
operação mutuamente exclusivos — mais complexa, menos testável, com interfaces ambíguas.

**Por que Pandas na análise profunda?**
Já estava nas dependências do projeto. Oferece estatísticas descritivas, groupby e filtros
prontos. O isolamento via dialeto garante que a troca futura por Polars ou DuckDB não
toque nos plugins.

**Por que consulta rápida sem Pandas?**
Perguntas como "houve erro?" não precisam de DataFrame. Carregar 50 mil registros em
memória para responder com `True` ou `False` é desperdício. O iterador com parada
antecipada responde em milissegundos.

**Por que nunca levantar exceção?**
Chamadores (API, CLI, MCP, agentes) não devem precisar de blocos try/except para consumir
o analisador. Falhas conhecidas (log não encontrado, correlation_id ausente) são retornadas
como resultado estruturado com `success=False`. Isso simplifica todos os boundary externos
e garante que o analisador nunca quebre silenciosamente.

**Por que `logs_dir` é sempre explícito?**
Nunca ler `settings` globais dentro do analisador garante que o módulo seja testável com
qualquer diretório, portável entre ambientes e sem acoplamento a configurações de runtime.

## 8. Limites e pegadinhas

- **Análise profunda carrega tudo em memória:** logs com mais de `max_records` registros
  são truncados. `AnalysisResult.truncated=True` indica isso. A análise é feita sobre a
  amostra.

- **Consulta rápida não garante varredura completa:** com `max_records=5000`, registros
  além desse limite não são vistos. Para logs grandes, aumentar `--max-records`.

- **Localização depende do manifest:** a estratégia primária de localização usa o manifest
  persistente. Se o manifest estiver desatualizado, o fallback via `CanonicalLogReader`
  entra. Se nenhum encontrar o arquivo, `LogFileNotFoundError` é retornado — não é erro
  do analisador, é ausência do arquivo.

- **`error_count` conta ERROR + CRITICAL:** o tipo `error_count` soma registros de nível
  ERROR e CRITICAL. Não é só ERROR puro.

- **Plugins falham independentemente:** se um plugin falhar durante a análise profunda,
  o erro é registrado em `analyses[plugin_id]["error"]` e a análise continua. O resultado
  final ainda tem `success=True` se pelo menos a execução foi concluída.

- **O módulo não lê `.env` nem variáveis de ambiente:** tudo é recebido via parâmetro.
  `logs_dir` deve ser informado explicitamente.

## 9. Impacto técnico

- Elimina o módulo legado `src.analysis` como caminho de diagnóstico
- Cria camada de domínio isolada: o analisador não depende de FastAPI, LangChain, Redis
  nem de nenhuma outra infraestrutura do produto
- Contratos pure-Python (dataclasses, sem Pydantic) garantem testabilidade sem overhead
  de validação de framework
- Pipeline de consulta rápida é O(n) com parada antecipada — custo proporcional à posição
  do primeiro resultado, não ao tamanho total do log

## 10. Impacto executivo

- Diagnóstico de falhas em produção deixa de depender de acesso manual ao servidor
- Time de suporte pode responder perguntas operacionais via API sem conhecimento técnico
  do formato dos logs
- Auditoria de execuções é acessível programaticamente — base para relatórios de SLA

## 11. Impacto comercial

- Agentes de IA do produto podem se auto-diagnosticar e reportar status de execução
- Ferramentas de pré-venda podem demonstrar rastreabilidade e observabilidade como
  capacidades nativas da plataforma
- Suporte técnico pode usar o CLI (`python -m src.log_analyzer query`) para diagnóstico
  rápido sem acesso ao banco ou ao servidor

## 12. Impacto estratégico

- Prepara a plataforma para loops de auto-melhoria: agente falha → consulta o próprio log
  → identifica causa → tenta corrigir
- Base para alertas automáticos por padrão de log (ex.: `error_count > N` dispara notificação)
- Aumenta confiança operacional: toda execução é rastreável, consultável e auditável de
  forma estruturada sem intervenção manual
