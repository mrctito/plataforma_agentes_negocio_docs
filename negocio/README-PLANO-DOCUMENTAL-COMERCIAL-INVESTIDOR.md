# Plano documental prioritário para comercial e investidores

## 1. Objetivo deste documento

Este documento existe para resolver um problema prático: a plataforma já
tem documentação técnica e conceitual muito forte, mas ainda faltam
alguns artefatos certos para transformar essa base em material realmente
pronto para venda repetível, diligência executiva e apresentação para
investidor.

Em linguagem simples: hoje o repositório já explica muito bem como o
produto funciona. O que este plano organiza é o que ainda precisa ser
documentado para explicar melhor como o produto vende, como reduz risco
e como deve ser apresentado para decisão de compra e de investimento.

Este manual entrega três coisas de forma consolidada:

1. a lista priorizada dos documentos estratégicos mais importantes;
2. o motivo prático de cada documento;
3. o esqueleto editorial recomendado para cada um.

## 2. Leitura executiva

Se a liderança precisar escolher por onde começar, a ordem correta é
esta:

1. formalizar compliance e segurança para conversa enterprise;
2. consolidar um investor brief para leitura executiva rápida;
3. publicar estudos de caso para prova comercial concreta;
4. definir precificação e empacotamento;
5. formalizar roadmap executivo e competitivo.

Essa ordem não é cosmética. Ela reduz risco real em três frentes:

1. reduz objeção de segurança em contas enterprise;
2. reduz fricção em apresentação para investidor ou comprador
   estratégico;
3. reduz improviso do time comercial em reunião de fechamento.

## 3. Priorização recomendada

### 3.1. Prioridade 1: compliance e segurança

**Documento recomendado:** README-COMPLIANCE-E-SEGURANCA-ENTERPRISE.md

**Por que vem primeiro:**

Sem esse documento, a conversa com CTO, segurança, jurídico e compras
enterprise fica fraca mesmo quando o produto é tecnicamente sólido.

**Problema que resolve:**

1. falta de resposta objetiva para perguntas de auditoria e risco;
2. dependência de leitura espalhada em autenticação, autorização,
   logging, BYOK e API;
3. dificuldade do comercial em separar o que já está comprovado do que
   ainda precisa de formalização organizacional.

**Impacto prático:**

1. acelera pré-venda enterprise;
2. melhora diligência técnica;
3. evita promessa indevida de certificação ou conformidade não
   comprovada.

### 3.2. Prioridade 2: investor brief

**Documento recomendado:** README-INVESTOR-BRIEF.md

**Por que vem logo depois:**

Hoje o repositório já sustenta bem a tese técnica do produto, mas essa
leitura está dispersa demais para investidor, conselheiro, parceiro
estratégico ou executivo não técnico.

**Problema que resolve:**

1. ausência de uma narrativa única de problema, tese, diferenciação,
   moat, riscos e próximos passos;
2. excesso de dependência do leitor montar sozinho a história a partir de
   dezenas de manuais;
3. dificuldade para apresentar o ativo como plataforma e não como coleção
   de features.

**Impacto prático:**

1. melhora reunião executiva;
2. reduz tempo de leitura para diligência inicial;
3. aumenta clareza sobre o que já está comprovado e o que ainda é plano.

### 3.3. Prioridade 3: estudos de caso

**Documento recomendado:** [README-ESTUDOS-DE-CASO.md](README-ESTUDOS-DE-CASO.md)

**Por que é o terceiro:**

Depois de segurança e narrativa executiva, a próxima lacuna mais séria é
prova comercial concreta. Sem estudo de caso, o discurso fica forte, mas
 ainda abstrato.

**Problema que resolve:**

1. dificuldade de provar aderência por vertical;
2. dificuldade do prospect se enxergar usando a plataforma;
3. falta de ponte entre documentação técnica e resultado de negócio.

**Impacto prático:**

1. melhora demonstração comercial;
2. fortalece proposta de valor por segmento;
3. ajuda investidor a perceber aplicabilidade real.

### 3.4. Prioridade 4: precificação e pacotes

**Documento recomendado:** README-PRECIFICACAO-E-PACOTES.md

**Por que vem aqui:**

O produto já tem tese, profundidade e base técnica, mas ainda não está
documentado como oferta comercial estruturada.

**Problema que resolve:**

1. venda depende demais de negociação artesanal;
2. falta clareza de como empacotar por cliente, tenant, vertical ou
   capacidade;
3. time comercial pode vender escopo sem guarda-corpo.

**Impacto prático:**

1. melhora repetibilidade comercial;
2. protege margem;
3. organiza expansão de portfólio.

### 3.5. Prioridade 5: roadmap executivo e competitivo

**Documentos recomendados:**

1. README-ROADMAP-EXECUTIVO.md
2. README-COMPETITIVE-LANDSCAPE.md

**Por que fecha o lote inicial:**

Esses dois documentos ajudam a responder duas perguntas que sempre
aparecem depois que a tese já convenceu.

1. para onde o produto vai;
2. por que ele ganha de abordagens alternativas.

**Problema que resolvem:**

1. ausência de direção consolidada em formato executivo;
2. falta de material comparativo para objeção comercial;
3. dificuldade de explicar vantagem estrutural sem depender de discurso
   oral do fundador ou do time técnico.

## 4. Matriz de uso por público

| Documento | Comercial | Pré-venda técnica | Diretoria | Investidor | Compras enterprise |
| --- | --- | --- | --- | --- | --- |
| Compliance e segurança | alto | alto | alto | médio | alto |
| Investor brief | médio | baixo | alto | alto | baixo |
| Estudos de caso | alto | médio | médio | alto | baixo |
| Precificação e pacotes | alto | baixo | alto | médio | médio |
| Roadmap executivo | médio | médio | alto | alto | baixo |
| Competitive landscape | alto | médio | médio | alto | baixo |

## 5. Esqueleto ideal de cada documento

## 5.1. Esqueleto: compliance e segurança enterprise

### Finalidade do documento de compliance

Explicar, de forma honesta e auditável, o que a plataforma já comprova
em segurança e governança, o que ainda não está formalizado como
certificação e como o time comercial deve apresentar isso sem exagero.

### Estrutura recomendada do documento de compliance

1. Objetivo do documento.
2. Resumo executivo para comprador enterprise.
3. O que já está comprovado no produto.
4. Controles técnicos evidenciados no repositório.
5. Controles operacionais que dependem de processo da empresa.
6. O que não deve ser prometido como fato hoje.
7. Checklist de diligência para cliente enterprise.
8. Perguntas frequentes de segurança.
9. Riscos residuais e próximos passos.
10. Fontes documentais do repositório.

### Perguntas que o documento de compliance precisa responder

1. Como a plataforma separa tenants, segredos e permissões?
2. Como autenticação, autorização e rastreabilidade funcionam?
3. Há trilha de auditoria e correlation_id ponta a ponta?
4. Quais políticas dependem do produto e quais dependem da operação da
   empresa?
5. O que já existe de segurança comprovável e o que ainda não pode ser
   apresentado como certificação formal?

## 5.2. Esqueleto: investor brief

### Finalidade do investor brief

Dar ao investidor uma leitura executiva curta, coerente e honesta da
plataforma como ativo de negócio, sem exigir leitura profunda dos manuais
técnicos.

### Estrutura recomendada do investor brief

1. O que é a empresa e o que é a plataforma.
2. Problema estrutural do mercado.
3. Tese do produto.
4. Diferenciais comprovados no repositório.
5. Moat técnico e operacional.
6. Onde o produto já parece maduro.
7. O que ainda precisa de formalização comercial ou empresarial.
8. Principais riscos de execução.
9. Próximos documentos que completam a data room.
10. Perguntas que o investidor provavelmente fará.

### Perguntas que o investor brief precisa responder

1. Por que este produto existe?
2. O que o diferencia de chatbot, framework e automação genérica?
3. O que já está comprovado no código e na documentação?
4. Quais riscos ainda existem?
5. O que precisa ser formalizado para sustentar escala comercial e de
   governança?

## 5.3. Esqueleto: estudos de caso

### Finalidade do documento de estudos de caso

Traduzir a plataforma para histórias comerciais concretas por vertical,
mostrando problema, arquitetura aplicada, ganho operacional e limites.

### Estrutura recomendada do documento de estudos de caso

1. Contexto do cliente ou cenário.
2. Problema anterior.
3. Como a plataforma foi usada.
4. Quais módulos entraram.
5. Resultado operacional esperado ou comprovado.
6. Riscos e limites.
7. Como vender esse caso em reunião.

## 5.4. Esqueleto: precificação e pacotes

### Finalidade do documento de precificação e pacotes

Dar guarda-corpo comercial para venda, proposta e expansão sem depender
de improviso caso a caso.

### Estrutura recomendada do documento de precificação e pacotes

1. Objetivo da política comercial.
2. Unidades possíveis de cobrança.
3. Pacotes por maturidade ou capacidade.
4. Serviços de implantação e expansão.
5. O que entra e o que não entra em cada faixa.
6. Riscos de escopo aberto.

## 5.5. Esqueleto: roadmap executivo

### Finalidade do documento de roadmap executivo

Explicar direção do produto em linguagem de negócio, e não apenas como
lista de desejos técnicos.

### Estrutura recomendada do documento de roadmap executivo

1. Objetivo estratégico do roadmap.
2. Capacidades já estabelecidas.
3. Próximas frentes de consolidação.
4. Próximas frentes de expansão.
5. Dependências organizacionais.
6. O que não deve entrar no roadmap por enquanto.

## 5.6. Esqueleto: competitive landscape

### Finalidade

Organizar objeção competitiva e comparação de categorias sem cair em
marketing raso.

### Estrutura recomendada

1. Categorias de concorrência.
2. Onde a plataforma ganha.
3. Onde a plataforma ainda precisa evoluir.
4. Como responder objeções em venda.
5. Critérios honestos de comparação.

## 6. Sequência sugerida de produção

Se o time quiser produzir isso sem se perder, a melhor sequência é esta.

1. Publicar compliance e segurança.
2. Publicar investor brief.
3. Publicar estudos de caso.
4. Publicar precificação e pacotes.
5. Publicar roadmap executivo.
6. Publicar competitive landscape.

## 7. Critérios de qualidade para os documentos novos

Todos os documentos dessa trilha devem obedecer a estas regras.

1. Não prometer certificação, integração, benchmark ou cliente que não
   esteja comprovado.
2. Separar sempre fato comprovado de plano, hipótese ou próxima etapa.
3. Usar linguagem executiva, mas sem esconder risco residual.
4. Ajudar o comercial a vender sem induzir promessa indevida.
5. Ajudar investidor e cliente enterprise a entender risco, maturidade e
   direção.

## 8. Resultado esperado

Quando esse pacote estiver completo, a documentação deixará de ser forte
apenas para engenharia e passará a ser forte também para:

1. reunião de venda enterprise;
2. pré-venda consultiva;
3. diligência de investidor;
4. parceria estratégica;
5. governança interna de posicionamento.
