# Manual técnico, executivo, comercial e estratégico: tools de WhatsApp Business

## 1. O que esta família realmente cobre

Este manual cobre a família de runtime confirmada no código para WhatsApp Cloud API. No recorte lido, ela é composta por duas tools concretas.

- whatsapp_send_text_message
- whatsapp_send_template_message

Isso significa que este documento trata da camada de envio de mensagem pelo catálogo agentic. Ele não descreve todo o ciclo de provisionamento comercial do canal, onboarding Meta ou roteamento externo de webhook. Esses assuntos vivem em outros slices do projeto.

## 2. Que problema ela resolve

Sem essa family, qualquer agente que quisesse usar WhatsApp precisaria lidar diretamente com autenticação, formatação de payload, versão da API, timeout e mensagens de erro. Isso criaria dois problemas.

- alto custo de repetição por fluxo ou cliente;
- risco de inconsistência entre jornadas que usam o mesmo canal.

A family atual encapsula esse trabalho e expõe uma interface mais simples para o agente.

## 3. Visão conceitual

O conceito aqui é simples e importante: a plataforma trata WhatsApp como capability de comunicação, não como detalhe HTTP bruto. O agente não deveria precisar conhecer endpoint da Meta nem montar payloads manualmente toda vez.

Em linguagem prática:

- quando precisa mandar texto livre, usa a tool de texto;
- quando precisa mandar comunicação governada e padronizada, usa a tool de template.

Isso reduz improvisação e melhora consistência operacional.

## 4. Visão técnica

A factory lida confirmada no código é create_whatsapp_cloud_tools. Ela impõe estes requisitos.

- user_session.correlation_id é obrigatório para rastreamento;
- security_keys.WHATSAPP_CLOUD_API_TOKEN é obrigatório;
- tool_config.whatsapp_cloud.phone_number_id é obrigatório;
- api_version assume v20.0 quando não é informada;
- timeout assume 10 segundos quando não é informado.

### 4.1. Tool de texto

A tool whatsapp_send_text_message exige:

- to_number
- body
- preview_url opcional

Se os campos obrigatórios estiverem ausentes, a tool falha com validação explícita. Em caso de sucesso, devolve JSON textual com a resposta do canal.

### 4.2. Tool de template

A tool whatsapp_send_template_message exige:

- to_number
- template_name
- language_code, com padrão pt_BR
- components_json opcional

Se components_json vier malformado, a tool falha explicitamente em vez de enviar payload inconsistente.

## 5. Visão executiva

Para liderança, esta family reduz atrito para jornadas de comunicação transacional e relacionamento. O valor não está em apenas enviar mensagem, mas em fazer isso de forma padronizada, com rastreabilidade por correlation_id e sem reinventar integração em cada fluxo.

## 6. Visão comercial

Comercialmente, WhatsApp tem alto valor porque é um dos canais mais naturais para confirmação, aviso, retomada de atendimento e comunicação operacional. A existência de duas tools separadas já ajuda a vender dois cenários diferentes.

- texto livre para interação direta e rápida;
- template para comunicação padronizada e repetível.

Isso torna a demonstração mais concreta e menos genérica.

## 7. Visão estratégica

Estrategicamente, essa family consolida WhatsApp como capability central do catálogo. Ela pode ser combinada com varejo, atendimento, CRM, observabilidade e workflows, sem exigir uma integração exclusiva por caso.

Esse é o valor de plataforma: o mesmo canal fica reutilizável em múltiplos domínios.

## 8. Como a family funciona por dentro

O fluxo técnico confirmado é este.

1. A factory valida correlation_id e segredos.
2. Ela carrega a configuração do bloco whatsapp_cloud.
3. Instancia um cliente compartilhado da Cloud API.
4. Expõe duas tools: uma para texto e outra para template.
5. Cada tool valida seus parâmetros mínimos.
6. Em caso de sucesso, serializa a resposta em JSON textual.
7. Em caso de falha operacional, registra log com contexto do destinatário e devolve mensagem de erro compreensível.

## 9. O que acontece em caso de sucesso

No caminho feliz, o agente recebe uma resposta serializada que já pode ser usada como evidência de envio ou como insumo do próximo passo do fluxo. Isso evita que a camada agentic tenha de interpretar manualmente um payload HTTP cru.

## 10. O que acontece em caso de erro

Os erros confirmados no código são estes.

- ausência de correlation_id;
- token ausente em security_keys;
- phone_number_id ausente no bloco whatsapp_cloud;
- to_number ou body ausentes na tool de texto;
- to_number, template_name ou language_code ausentes na tool de template;
- components_json inválido na tool de template.

Na prática, esses erros ajudam a separar problema de configuração de problema de provider.

## 11. Limites e pegadinhas

- Esta family cobre envio pelo runtime, não o processo inteiro de habilitação comercial do canal.
- O manual não deve ser lido como documentação de provisionamento Meta.
- A response da tool vem em JSON textual; quem consome precisa tratar isso como retorno de integração, não como garantia semântica de conclusão de negócio.
- Template não é o mesmo caso de uso de texto livre; escolher errado aumenta atrito operacional.

## 12. Quando usar esta family

Use esta family quando o problema principal for comunicação via WhatsApp e o canal já estiver configurado para o tenant ou ambiente. Se o problema for provisionar conta, ativar onboarding ou operar diretório de clientes do canal, o slice correto é outro.

## 13. Evidências no código

- src/agentic_layer/tools/vendor_tools/whatsapp_tools/whatsapp_toolkit.py
  - Motivo da leitura: confirmar a factory, o contrato de entrada e as duas tools concretas.
  - Comportamento confirmado: create_whatsapp_cloud_tools exige correlation_id, token, phone_number_id e expõe envio de texto e template.
- src/agentic_layer/tools/tools_library_builder.py
  - Motivo da leitura: confirmar como a family aparece no discovery.
  - Comportamento confirmado: o builder registra a family, mas a factory em si expõe as duas tools concretas no runtime.
