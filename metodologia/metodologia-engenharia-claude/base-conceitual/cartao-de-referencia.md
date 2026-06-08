# Cartão de Referência Rápida

> **Para que serve.** Uso diário, não aula. Uma página para responder rápido: *"qual comando uso?"*,
> *"o que ele me garante?"*, *"o que preciso ter pronto antes?"*. Imprima e deixe ao lado do monitor.
> A fonte de verdade continua sendo os agentes/contratos reais — se divergir do código, **o código vence**.

---

## 1. Qual comando chamar — e o que ele garante

| Comando | Use quando | Pré-requisito | O que ele garante (o conteúdo por trás) |
|---|---|---|---|
| `/investigar` | Preciso entender o estado real (sem mudar nada) | Um anchor (arquivo, fluxo, erro) | Achados com evidência (path/linha); zero alucinação; 3 fases |
| `/analisar-log` | Quero entender uma execução por `correlation_id` | Ter o `correlation_id` | Linha do tempo + erros/fallbacks; separa o que o log prova |
| `/planejar` | Vou transformar investigação em ação | **Investigação pronta** | Plano com T1/T2, critérios de aceite, rollback; postura crítica |
| `/implementar` | Vou executar um plano | **Plano aprovado** | Código sob gates (reuso/banco/YAML), anti-legado, definição-de-pronto |
| `/validar-entrega` | Terminei e quero o veredito | Implementação feita | Revisor independente; anti-falso-verde; status explícito |
| `/corrigir-com-log` | Tenho um bug real para consertar | `correlation_id` do erro | Causa raiz (proibido mascarar); validado na suíte |
| `/criar-testes` | Preciso de cobertura nova | Código a proteger | Família correta; reduz risco real (não testa cosmético) |
| `/executar-testes` | A suíte caiu / preciso estabilizar | — | Suíte verde **de verdade**, auditada por telemetria |
| `/validar-log` | Auditar se os logs seguem o contrato | Escopo definido | Cobertura 100% do escopo; caça `logger.error` em except |
| `/validar-observabilidade` | Auditar se há log suficiente | Escopo definido | Fluxo reconstruível por log (debug offline) |
| `/documentar` | Documentar feature a fundo | Código pronto e provado | Doc do código real, não da intenção; nível 101 |

> **Regra de desempate mais comum:** entender log → `/analisar-log`. Consertar bug → `/corrigir-com-log`.
> Entender código → `/investigar`. *(Fluxograma completo em [diagramas.md](diagramas.md), Diagrama 3.)*

---

## 2. O eixo central (a sequência que resolve quase tudo)

```
/investigar  →  /planejar  →  /implementar  →  /validar-entrega
 (evidência)    (rota)         (código vivo)     (veredito)
                                                      │
                                  REPROVADO ──────────┘ volta pro /planejar
```

---

## 3. Os 5 valores que toda tarefa deve respeitar

1. **Evidência > suposição** — afirmou? prove com path/linha.
2. **Rastreabilidade** — todo processo tem `correlation_id`; log é raio-X.
3. **Anti-falso-verde** — pronto = ativo no runtime, não "teste passou".
4. **Reuso > criar** — procure antes de criar; achou 2 soluções? unifique.
5. **Escopo mínimo, qualidade máxima** — "100%" qualifica a execução, não infla o escopo.

---

## 4. Sinais de alerta (se você vir isso, pare)

| Sinal | Por que é problema | O que fazer |
|---|---|---|
| Vou criar uma classe/helper novo | Provável duplicação | Cumprir `reuso-instructions` antes |
| Vou manter um fallback "por segurança" | Legado mascarando causa raiz | Só com pedido explícito registrado |
| Teste passou, vou fechar | Pode ser falso-verde | Provar no caminho oficial (definição-de-pronto) |
| Vou listar a pasta `/logs` | Pode travar a sessão | Buscar por `correlation_id` específico |
| Editei `src/` e não toquei em `tests/` | Cobertura/observabilidade faltando | Criar teste proporcional + checar logs |
| Vou logar o payload inteiro | Dado sensível / ruído | Logar só shape/contagem/decisão |

---

## 5. Checklist — criar um novo agente / skill / regra (governança de extensão)

> Mencionado na [FAQ](faq-objecoes.md) como o ponto que depende de disciplina humana. Use este checklist
> para estender o sistema **sem causar drift**.

**Antes de criar um novo AGENTE:**
- [ ] A função já não é coberta por um agente existente? (evitar sobreposição)
- [ ] É uma responsabilidade **única** e clara? (SRP — não um "faz-tudo")
- [ ] O conteúdo define: papel, gatilho, fases/contrato, proibições, saída materializada?
- [ ] Referencia os contratos de `rules/` em vez de **duplicá-los**?
- [ ] Define como o resultado volta (relatório, status explícito)?

**Antes de criar uma nova SKILL:**
- [ ] É um **atalho fino** que delega ao agente? (padrão preferido) Se for "gorda", o procedimento
      realmente **não pode** ser pulado/evoluído no agente?
- [ ] O `description` está no formato `Use when:` com **desambiguação** das skills parecidas?
- [ ] Não duplica regra que já vive no agente?

**Antes de criar uma nova REGRA (`rules/`):**
- [ ] É contrato profundo (carregado sob demanda) ou registro operacional (memória viva)?
- [ ] Não conflita nem duplica regra existente? (rodar `validar-instructions` depois)
- [ ] Tem rastreabilidade — alguém/algum agente realmente a consome?
- [ ] Se for lição: passa no teste de promoção? *("outro agente, amanhã, em outro slice, evitaria erro?")*

**Depois de qualquer extensão:**
- [ ] Rodar `/validar-instructions` para detectar contradição/redundância.
- [ ] Atualizar o índice da metodologia / `README-TOOLS-LIB.md` se aplicável.

---

**Relacionado (base conceitual):** [Diagramas](diagramas.md) · [FAQ de Objeções](faq-objecoes.md) ·
[Ciclo de Vida do Software](ciclo-de-vida-software.md) ·
[↩ Voltar ao índice da metodologia](../README.md)
</content>
