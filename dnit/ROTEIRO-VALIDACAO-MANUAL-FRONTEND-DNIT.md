# Roteiro Padrão de Validação Manual do Frontend DNIT

## Objetivo

Este roteiro define a validação manual padrão da tela de detalhe DNIT quando a mudança tocar frontend, layout, usabilidade, integrações visuais com o backend ou comportamento do assistente.

Ele não substitui a suíte automatizada do repositório quando ela for necessária. Aqui o objetivo é ter uma bateria manual mínima, repetível e objetiva para proteger a experiência da tela.

## Tela-alvo

- Detalhe do projeto DNIT:
  - `http://localhost:5555/ui/static/ui-dnit-project-detail.html?id=<PROJECT_ID>`

## Pré-condições

- Servidor local no ar na porta do `.env`
- Login funcional com:
  - usuário: `joao@teste`
  - senha: `123456`
- Projeto DNIT existente com pelo menos:
  - 1 arquivo já persistido
  - contexto do assistente habilitado no backend

## Bateria padrão obrigatória

### 1. Carregamento inicial da tela

Validar:

- a tela abre sem erro visível
- o header global renderiza corretamente
- o título do projeto não estoura layout
- os três painéis principais aparecem alinhados:
  - arquivos
  - visualização
  - assistente

### 2. Responsividade visual mínima

Validar pelo menos em:

- desktop largo
- desktop médio
- largura reduzida próxima do print de trabalho do usuário

Conferir:

- header não consome espaço excessivo
- cards superiores ficam alinhados e centralizados
- painel do assistente continua utilizável
- não há texto cortado, sobreposição ou scroll estranho no topo

### 3. Lista de arquivos do projeto

Validar:

- arquivos aparecem no painel esquerdo
- seleção visual do arquivo ativo funciona
- nome, tipo e tamanho aparecem corretamente
- trocar o arquivo muda a visualização central

### 4. Visualização de conteúdo

Validar:

- o conteúdo do arquivo selecionado abre na área central
- trocar entre arquivos atualiza o conteúdo
- a rolagem da visualização funciona
- botão/ação de edição continua íntegro quando aplicável

### 5. Upload de arquivo

Validar:

- envio de um arquivo novo pela UI
- arquivo aparece na lista sem recarregar manualmente
- arquivo recém-enviado pode ser selecionado
- após refresh da página, o arquivo continua presente

### 6. Indexação do arquivo

Validar:

- indexação individual do arquivo pela tela
- status visual sai do estado inicial e converge para indexado
- a ação não quebra a tela
- após recarregar a página, o estado indexado continua refletido

### 7. Painel do assistente

Validar:

- painel abre com largura confortável
- área de contexto fica discreta e não rouba espaço excessivo
- campo de edição aceita texto normalmente
- textarea cresce automaticamente até o limite esperado
- botão `Enviar` e botão `Cancelar` mantêm proporção discreta e consistente

### 8. Histórico persistido

Validar:

- criar um trecho/pergunta/resposta vinculado ao arquivo
- a resposta aparece no assistente
- o correlation id retornado aparece na bolha
- após refresh da página, o histórico continua disponível
- salvar histórico não apaga o estado de indexação do arquivo

### 9. Navegação

Validar:

- entrada na tela de detalhe ocorre na mesma aba
- ação de voltar no header leva de volta ao mestre corretamente
- não abre aba nova inesperadamente

### 10. Regressão visual crítica

Antes de dar a tela como aprovada, conferir:

- sem estouro de título
- sem desalinhamento lateral entre margem esquerda e direita
- sem área morta grande acima do conteúdo
- sem redução excessiva da área útil do assistente

## Critério de aprovação

A validação manual do frontend DNIT só pode ser considerada aprovada quando:

- a tela carrega
- o layout permanece estável
- upload funciona
- indexação funciona
- histórico persiste
- refresh não perde o estado principal
- a área do assistente continua confortável para uso intenso

## Evidência mínima recomendada

Registrar pelo menos:

- URL testada
- data/hora
- projeto usado
- arquivo usado no teste
- resultado de cada item obrigatório
- screenshot final da tela, quando a mudança afetar layout
