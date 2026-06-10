# Exemplos de Integração com API RAG

Produto: Plataforma de Agentes de IA

Esta pasta contém exemplos práticos de como integrar sua aplicação com a API RAG em diferentes linguagens de programação.

Explicando de forma simples: o cliente manda uma pergunta e um YAML de configuração. Esse YAML vai criptografado para o servidor, o servidor executa o fluxo RAG e devolve a resposta com fontes.

## O que este manual resolve

Este documento existe para evitar dois erros comuns. O primeiro é tentar
integrar com a API RAG adivinhando payload, criptografia e ordem das
chamadas. O segundo é confundir exemplo de cliente com contrato oficial
do backend.

Aqui, os exemplos servem como ponte prática entre o contrato HTTP e a
implementação do cliente. O objetivo não é só mostrar código, mas deixar
claro quando usar cada endpoint, que pré-condições precisam existir e
como distinguir erro de ambiente, erro de autenticação e erro de
payload.

## Quando usar este manual

Use este manual quando a dúvida for operacional e prática, por exemplo:

- como montar um cliente novo em Python, Ruby ou JavaScript;
- como seguir a ordem correta entre `/crypto/session-key` e
  `/rag/execute`;
- como validar se a criptografia e o payload estão coerentes;
- como reproduzir a mesma integração em outra linguagem.

Não use este manual como substituto da documentação conceitual de RAG,
autenticação ou YAML. Nesses casos, ele é apoio de implementação, não
fonte única de entendimento.

## 📚 Arquivos Disponíveis

| Arquivo                            | Linguagem          | Descrição                                                        |
| ---------------------------------- | ------------------ | ---------------------------------------------------------------- |
| `rag_api_client.py`                | Python             | Cliente completo com criptografia FERNET+RSA-OAEP                |
| `rag_api_client.rb`                | Ruby               | Cliente completo com bibliotecas padrão (`net/http` + `OpenSSL`) |
| `rag_api_client.js`                | JavaScript/Node.js | Cliente completo com `axios` e `crypto` nativo do Node.js        |
| `react_webchat_client_example.jsx` | React/browser      | Exemplo mínimo do caminho real do WebChat usando `PayloadCrypto` |

## 🚀 Pré-requisitos

### 1️⃣ API em Execução

Certifique-se que a API RAG está rodando:

```bash
# Terminal 1: Iniciar API
cd /caminho/para/projeto
python app/main.py

# Terminal 2: Verificar health
curl http://localhost:8000/health
```

Observação importante sobre porta padrão dos exemplos:

- Python e JavaScript usam `http://localhost:8000`
- Ruby usa `http://localhost:5555` por padrão (pode ser sobrescrito com `--base-url`)

### 2️⃣ Arquivo YAML de Configuração

Você precisa de um arquivo YAML válido com:

- `access_key` cadastrada no banco de dados
- Configurações do vector store
- Configurações do LLM

Exemplo: `app/yaml/rag-config-auto.yaml`

## 🐍 Python

### Instalação de Dependências em Python

```bash
pip install httpx cryptography pyyaml
```

### Execução em Python

```bash
python examples/rag_api_client.py
```

### Uso Programático em Python

```python
from rag_api_client import RAGAPIClient

# Criar cliente
client = RAGAPIClient(api_base_url="http://localhost:8000")

# Fazer consulta
result = client.ask(
    config_path="app/yaml/rag-config-auto.yaml",
    question="Como gerar SPED no Menew?",
    user_email="developer@empresa.com",
    correlation_id="20250204_120000-123e4567-e89b-12d3-a456-426614174000"
)

# Usar resultado
print(result["answer"])
for source in result["sources"]:
    print(f"  - {source}")
```

## 💎 Ruby

### Instalação de Dependências em Ruby

```bash
# O exemplo Ruby usa apenas bibliotecas padrão do Ruby
# (net/http, openssl, json, yaml, etc.)
```

### Execução em Ruby

```bash
ruby examples/rag_api_client.rb
```

### Uso Programático em Ruby

```ruby
require_relative 'rag_api_client'
require 'yaml'

yaml_path = 'app/yaml/rag-config-auto.yaml'
yaml_data = YAML.safe_load(File.read(yaml_path, encoding: 'UTF-8')) || {}
api_key = yaml_data.dig('authentication', 'access_key')

# Criar cliente
client = RAGAPIClient.new(
  api_base_url: 'http://localhost:5555',
  api_key: api_key
)

# Fazer consulta
result = client.ask(
  config_path: 'app/yaml/rag-config-auto.yaml',
  question: 'Como gerar SPED no Menew?',
  user_email: 'developer@empresa.com',
  correlation_id: '20250204_120000-123e4567-e89b-12d3-a456-426614174000'
)

# Usar resultado
puts result['answer']
result['sources']&.each { |source| puts "  - #{source}" }
```

## 🟨 JavaScript/Node.js

### Instalação de Dependências em JavaScript

```bash
npm install axios
```

### Execução em JavaScript

```bash
node examples/rag_api_client.js
```

### Uso Programático em JavaScript

```javascript
const { RAGAPIClient } = require('./rag_api_client');

// Criar cliente
const client = new RAGAPIClient('http://localhost:8000');

// Fazer consulta
(async () => {
  const result = await client.ask({
    configPath: 'app/yaml/rag-config-auto.yaml',
    question: 'Como gerar SPED no Menew?',
    userEmail: 'developer@empresa.com',
    correlationId: '20250204_120000-123e4567-e89b-12d3-a456-426614174000'
  });

  // Usar resultado
  console.log(result.answer);
  result.sources?.forEach(source => console.log(`  - ${source}`));
})();
```

## 🔒 Entendendo a Criptografia (ELI5)

Pense assim: o YAML é um pacote sensível. Você coloca esse pacote em um cofre (Fernet), e depois coloca a chave do cofre em um envelope lacrado para o servidor (RSA-OAEP). Só o servidor consegue abrir esse envelope.

O contrato atual do endpoint `/rag/execute` (OpenAPI) espera `encrypted_data` no formato abaixo.
Use esse formato como referência oficial para qualquer cliente novo:

### Fluxo Completo

```text
1. Cliente solicita sessão criptográfica
   POST /crypto/session-key
   → Retorna: session_id, public_key_pem, ttl_seconds

2. Cliente cifra o YAML localmente
  - Primeiro usa Fernet (simétrico) para proteger o YAML
  - Depois protege a chave Fernet com RSA-OAEP (SHA-256)
  - Chave RSA pública é efêmera (recebida da sessão)

3. Cliente envia dados criptografados
   POST /rag/execute
   Headers: Content-Type: application/json, X-API-Key: <chave>
   Body: {
      "operation": "ask",
      "payload": {
        "question": "Quais são os requisitos mínimos de hardware para o Taste One?",
        "user_email": "lucas.baltazar@linx.com.br",
        "format": "json", // ou "text"
        "execution_mode": "auto", // ou "direct_sync", "direct_async"
        "encrypted_data": {
          "session_id": "...",
          "wrapped_key": "...",
          "encrypted_yaml": "...",
          "original_filename": "config-webchat.yaml",
          "encryption_scheme": "FERNET+RSA-OAEP",
          "yaml_operational_contract": "..."
        }
      }
    }
   // Nota: o helper browser oficial (PayloadCrypto.buildEncryptedData) NÃO gera mais
   // o campo "encrypted_keys" — as credenciais são injetadas pelo backend. Clientes
   // novos devem usar o envelope acima, sem "encrypted_keys".

4. Servidor descriptografa e processa
   - Usa chave privada efêmera (válida por 2 min)
   - Resolve credenciais via banco interno
   - Nunca expõe credenciais ao cliente
  - `subprocess` não é modo público válido no HTTP; se enviado explicitamente, a API falha fechado e orienta uso de `direct_async`

5. Servidor retorna resposta
  JSON com answer, sources e métricas

Resumo ELI5: você só precisa seguir 2 endpoints (`/crypto/session-key` e `/rag/execute`) e enviar `X-API-Key` correto.
```

### Por que Criptografia?

 **Segurança**: YAML pode conter configurações sensíveis
 **Isolamento**: Cliente não precisa acessar banco de credenciais
 **Auditoria**: Todas as requisições são rastreáveis
 **Compliance**: Credenciais nunca trafegam descriptografadas

## 📊 Estrutura da Resposta

Todos os clientes retornam o mesmo formato:

```json
{
  "answer": "Para gerar o SPED no sistema Menew...",
  "correlation_id": "20250204_120000-123e4567-e89b-12d3-a456-426614174000",
  "timestamp": "2026-03-02T12:34:56.123456+00:00",
  "execution_mode": "direct_sync",
  "task_id": null,
  "status_url": null,
  "polling_url": null,
  "stream_url": null,
  "sources": [
    {
      "source": "manual_menew.pdf",
      "score": 0.92
    }
  ],
  "source_documents": [
    {
      "content": "Trecho relevante do documento...",
      "metadata": {
        "source": "manual_menew.pdf",
        "page": 42
      }
    }
  ],
  "analysis": {},
  "metrics": {
    "processing_time_ms": 2340,
    "retrieval_time_ms": 450,
    "llm_time_ms": 1890
  },
  "token_usage": {},
  "logs": [],
  "metadata": {},
  "duration": 2.34,
  "vectorstore_id": "rag_store_food",
  "success": true
}
```

## ⚠️ Tratamento de Erros

Todos os exemplos tratam estes cenários:

| HTTP Status | Situação                | Ação                         |
| ----------- | ----------------------- | ---------------------------- |
| **400**     | Parâmetros inválidos    | Verificar payload e YAML     |
| **401**     | Autenticação falhou     | Verificar access_key no YAML |
| **404**     | Endpoint não encontrado | Verificar URL da API         |
| **500**     | Erro interno            | Verificar logs do servidor   |
| **503**     | Serviço indisponível    | Retry com backoff            |
| **Timeout** | Request demorou         | Aumentar timeout             |

### Exemplo de Tratamento (Python)

```python
try:
    result = client.ask(...)
except RuntimeError as e:
    if "API não está disponível" in str(e):
        print("🔴 Servidor offline. Verifique se main.py está rodando")
    elif "400" in str(e):
        print("🔴 Erro de validação. Verifique o YAML e access_key")
    else:
        print(f"🔴 Erro inesperado: {e}")
```

## 🧪 Testando a Integração

### 1. Health Check

```bash
curl http://localhost:8000/health
# Esperado: {"status":"healthy", "timestamp":"...", "version":"..."}
```

### 2. Sessão Criptográfica

```bash
curl -X POST http://localhost:8000/crypto/session-key
# Esperado: JSON com session_id e public_key_pem
```

### 3. Executar Exemplo

```bash
# Python
python examples/rag_api_client.py

# Ruby
ruby examples/rag_api_client.rb

# JavaScript
node examples/rag_api_client.js
```

### 4. Verificar Logs

## 🧭 Exemplos de uso da API Assembly AST

Esta seção mostra exemplos completos de uso da API de Assembly AST para dois cenários práticos:

1. gerar draft de AST a partir de linguagem natural;
2. descobrir quais tools do catálogo são mais indicadas para uma situação antes de montar o YAML.

### Quando usar `recommend-tools`

Use `POST /config/assembly/recommend-tools` quando a dúvida ainda for estratégica.
Exemplo: "quais tools eu devo usar para ler JSON e gerar TXT?".

Use `POST /config/assembly/objective-to-yaml` quando você quer o resultado final do produto em uma única chamada: YAML validado ou perguntas bloqueantes.

Use `POST /config/assembly/draft` quando você já quer começar a montar a estrutura AST/YAML.

Resumo simples:

- `recommend-tools` ajuda a escolher ferramentas.
- `objective-to-yaml` entrega YAML final ou perguntas obrigatórias.
- `draft` ajuda a montar configuração quando você precisa controlar o fluxo passo a passo.

### Exemplo 1: `curl` para recomendar tools

```bash
curl -X POST http://localhost:8000/config/assembly/recommend-tools \
  -H "Content-Type: application/json" \
  -H "X-API-Key: SUA_API_KEY" \
  -d '{
    "user_email": "ops@empresa.com",
    "base_yaml": {
      "llm": {
        "provider": "openai"
      }
    },
    "situation": "Como faço para ler um arquivo json e transformar em txt?",
    "target": "workflow",
    "constraints": {
      "llm_schema_max_attempts": 3
    }
  }'
```

Resposta esperada no caminho feliz:

```json
{
  "success": true,
  "analysis_summary": "A situação pede uma tool de leitura JSON e, se houver no catálogo, uma tool de escrita textual.",
  "recommended_tools": [
    {
      "tool_id": "json_reader",
      "reason": "A descrição informa leitura de arquivos JSON e retorno estruturado."
    }
  ],
  "solution_steps": [
    {
      "title": "Ler o JSON",
      "description": "Use a tool para abrir o arquivo e obter o conteúdo estruturado.",
      "tool_ids": ["json_reader"]
    }
  ],
  "limitations": [],
  "diagnostics": [],
  "catalog_size": 123,
  "correlation_id": "20260306_120000-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
}
```

### Exemplo 2: `curl` para objetivo -> YAML final

```bash
curl -X POST http://localhost:8000/config/assembly/objective-to-yaml \
  -H "Content-Type: application/json" \
  -H "X-API-Key: SUA_API_KEY" \
  -d '{
    "user_email": "ops@empresa.com",
    "prompt": "Monte um workflow de atendimento com triagem, consulta ao CRM e resposta final.",
    "target": "auto",
    "generation_mode": "llm_schema",
    "base_yaml": {
      "llm": {
        "provider": "openai"
      }
    }
  }'
```

Resposta esperada no caminho feliz:

```json
{
  "success": true,
  "requested_target": "auto",
  "resolved_target": "workflow",
  "blocking_stage": null,
  "final_yaml": {
    "selected_workflow": "wf_atendimento"
  },
  "final_yaml_text": "selected_workflow: wf_atendimento\n",
  "chosen_tools": [
    {
      "tool_id": "crm_lookup",
      "source": "confirm.final_yaml",
      "paths": ["workflows[0].nodes[0].tools[0]"]
    }
  ],
  "questions": [],
  "diagnostics": [],
  "correlation_id": "20260306_120010-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
}
```

Se ainda faltar informação obrigatória, o mesmo endpoint responde com `success=false`, `blocking_stage` e `questions`, em vez de inventar um YAML enganoso.

### Exemplo 3: `curl` para draft AST

```bash
curl -X POST http://localhost:8000/config/assembly/draft \
  -H "Content-Type: application/json" \
  -H "X-API-Key: SUA_API_KEY" \
  -d '{
    "user_email": "ops@empresa.com",
    "prompt": "Crie um workflow de atendimento com triagem e resposta final.",
    "target": "workflow",
    "generation_mode": "auto",
    "constraints": {
      "auto_heuristic_fallback_enabled": true
    },
    "return_content": true,
    "base_yaml": {
      "selected_workflow": "wf_base",
      "workflows": []
    }
  }'
```

### Exemplo 4: JavaScript para `recommend-tools`

```javascript
async function recommendTools() {
  const response = await fetch('http://localhost:8000/config/assembly/recommend-tools', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': process.env.API_KEY
    },
    body: JSON.stringify({
      user_email: 'ops@empresa.com',
      base_yaml: {
        llm: {
          provider: 'openai'
        }
      },
      situation: 'Quero criar uma planilha excel com resumo diário de vendas.',
      target: 'workflow'
    })
  });

  const data = await response.json();
  console.log('Resumo:', data.analysis_summary);
  console.log('Tools:', data.recommended_tools);
  console.log('Passos:', data.solution_steps);
}
```

### Exemplo 5: Python para `recommend-tools`

```python
import requests

payload = {
    "user_email": "ops@empresa.com",
    "base_yaml": {
        "llm": {
            "provider": "openai"
        }
    },
    "situation": "Quero criar uma planilha excel com consolidado de vendas.",
    "target": "workflow"
}

response = requests.post(
    "http://localhost:8000/config/assembly/recommend-tools",
    headers={
        "Content-Type": "application/json",
        "X-API-Key": "SUA_API_KEY"
    },
    json=payload,
    timeout=60,
)

response.raise_for_status()
data = response.json()

print(data["analysis_summary"])
for tool in data["recommended_tools"]:
    print(tool["tool_id"], "-", tool["reason"])
```

### Exemplo 6: erro esperado quando a recomendação não pode ser sustentada

Se a resposta estruturada da LLM citar uma tool que não existe no catálogo real, o backend invalida a recomendação.

Formato esperado:

```json
{
  "success": false,
  "analysis_summary": "A resposta do modelo não pôde ser aceita como recomendação confiável.",
  "recommended_tools": [],
  "solution_steps": [],
  "limitations": [],
  "diagnostics": [
    {
      "code": "AST_TOOL_RECOMMENDATION_TOOL_INVALIDO",
      "message": "LLM retornou tool_id fora do catálogo permitido: tool_inventada",
      "path": "recommended_tools.tool_id",
      "severity": "error",
      "target": "workflow"
    }
  ],
  "catalog_size": 123,
  "correlation_id": "20260306_120001-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
}
```

Significado prático:

- a API respondeu corretamente;
- mas a recomendação foi bloqueada por segurança e consistência de catálogo;
- o cliente deve revisar a situação enviada ou o catálogo disponível, em vez de usar a sugestão inválida.

```bash
# Logs da API (servidor)
tail -f /logs/api_main.log

# Buscar por correlation_id
grep "20250204_120000-123e4567-e89b-12d3-a456-426614174000" /logs/api_main.log
```

## 🎯 Casos de Uso

### Integração em Aplicação Web (Python/Django)

```python
# views.py
from django.http import JsonResponse
from .rag_client import RAGAPIClient

def perguntar_rag(request):
    client = RAGAPIClient()

    result = client.ask(
        config_path="config/rag-production.yaml",
        question=request.POST.get('question'),
        user_email=request.user.email,
        correlation_id="20250204_120000-123e4567-e89b-12d3-a456-426614174000"
    )

    return JsonResponse({
        'answer': result['answer'],
        'sources': result['sources']
    })
```

### Integração em API Ruby on Rails

```ruby
# app/services/rag_service.rb
class RagService
  def initialize
    @client = RAGAPIClient.new(
      api_base_url: ENV['RAG_API_URL']
    )
  end

  def ask(question, user_email)
    @client.ask(
      config_path: Rails.root.join('config', 'rag.yaml'),
      question: question,
      user_email: user_email,
      correlation_id: "20250204_120000-123e4567-e89b-12d3-a456-426614174000"
    )
  rescue StandardError => e
    Rails.logger.error("RAG Error: #{e.message}")
    { error: e.message }
  end
end
```

### Integração em API Node.js/Express

```javascript
// routes/rag.js
const express = require('express');
const { RAGAPIClient } = require('../clients/rag_api_client');

const router = express.Router();
const ragClient = new RAGAPIClient(process.env.RAG_API_URL);

router.post('/ask', async (req, res) => {
  try {
    const result = await ragClient.ask({
      configPath: './config/rag-production.yaml',
      question: req.body.question,
      userEmail: req.user.email,
      correlationId: `express-${Date.now()}`
    });

    res.json({
      answer: result.answer,
      sources: result.sources
    });

  } catch (error) {
    console.error('RAG Error:', error);
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## 🔧 Troubleshooting

### Erro: "API não está disponível"

```bash
# Verificar se API está rodando
ps aux | grep "python.*main.py"

# Iniciar API se necessário
python app/main.py
```

### Erro: "Access_key informada não está cadastrada"

```bash
# Verificar access_key no YAML
grep "access_key" app/yaml/seu-config.yaml

# Consultar banco de dados
python -c "
from src.security.client_directory import ClientDirectory
directory = ClientDirectory.instance()
record = directory.get_access_key_record('SUA_ACCESS_KEY')
print(record)
"
```

### Erro de Criptografia

```bash
# Python: Reinstalar cryptography
pip uninstall cryptography
pip install cryptography

# Ruby: Reinstalar openssl
gem uninstall openssl
gem install openssl

# Node: Reinstalar axios
npm uninstall axios
npm install axios
```

## Limites e pegadinhas

- Exemplo funcional não substitui contrato oficial da API. Quando houver
  dúvida de rota, schema ou campo de resposta, a fonte canônica continua
  sendo o backend e o inventário HTTP.
- Health check verde não prova que a integração completa está pronta. A
  sessão criptográfica, a access key e o YAML ainda podem falhar depois.
- Conseguir cifrar o YAML não significa que o conteúdo está válido. A
  criptografia protege transporte, não corrige configuração errada.
- Um cliente de exemplo pode precisar ajuste de porta ou base URL para o
  ambiente local real.

## Leitura relacionada

- 🔐 **[README-CONCEITUAL-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md](../conceitual/README-CONCEITUAL-AUTENTICACAO-GOOGLE-MFA-PROJETO-APIS.md)** - Sistema de autenticação
- ⚙️ **[API-ENDPOINTS-SWAGGER.md](../tecnico/API-ENDPOINTS-SWAGGER.md)** - Documentação da API
- 🔧 **[README-CONFIGURACAO-YAML.md](../tecnico/README-CONFIGURACAO-YAML.md)** - Estrutura do YAML
- 🔗 **[API-ENDPOINTS-SWAGGER.md](../tecnico/API-ENDPOINTS-SWAGGER.md)** - Inventário atual das rotas públicas

## Evidência no código

- `src/api/service_api.py` (`GET /health`)
- `src/api/routers/crypto_router.py` (`POST /crypto/session-key`)
- `src/api/routers/rag_router.py` (`POST /rag/execute`)
- `src/api/schemas/rag_models.py` (`RagQuestionRequest`, `RagQuestionResponse`)
- `examples/rag_api_client.py`
- `examples/rag_api_client.js`
- `examples/rag_api_client.rb`

## Lacunas no código

- Não encontrado no código: nenhum endpoint adicional específico para clientes multilíngues além dos endpoints HTTP já documentados.
- Onde deveria estar: `src/api/routers/` (caso no futuro existam rotas dedicadas por SDK/linguagem).

## 💡 Contribuindo

Para adicionar exemplos em outras linguagens:

1. Implemente a mesma interface dos exemplos existentes
2. Siga o padrão de criptografia RSA-OAEP + SHA-256
3. Use a estrutura de payload idêntica ao webchat
4. Adicione documentação clara e exemplos de uso
5. Teste com a API em execução

## Checklist de entendimento

- Entendi a ordem entre sessão criptográfica e execução RAG.
- Entendi a diferença entre exemplo de cliente e contrato oficial da API.
- Entendi como separar erro de servidor offline, access key, payload e criptografia.
- Entendi quais manuais consultar quando a dúvida deixa de ser só de integração prática.

## Encerramento

Documento mantido pela equipe da Plataforma de Agentes de IA.
