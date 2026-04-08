# 📚 Base de Conhecimento: Orquestração de Workflows

> Curado em Abril de 2026 | Fontes: Temporal.io, Dapr, Microsoft, AWS, microservices.io e outros

---

## Estrutura da Base

| # | Arquivo | Tópico |
|---|---------|--------|
| 01 | [Fundamentos](./01-fundamentos.md) | O que é orquestração, durable execution, conceitos-chave |
| 02 | [Orquestração vs Coreografia](./02-orquestracao-vs-coreografia.md) | Comparação, trade-offs e quando usar cada um |
| 03 | [Temporal.io](./03-temporal-io.md) | Arquitetura, primitivos, casos de uso |
| 04 | [Dapr Workflows](./04-dapr-workflows.md) | Arquitetura sidecar, padrões, CNCF |
| 05 | [Padrão Saga](./05-saga-pattern.md) | Transações distribuídas, compensação, exemplos |
| 06 | [Padrões de Workflow](./06-workflow-patterns.md) | Task chaining, fan-out/fan-in, monitor, timers |
| 07 | [Comparativo de Ferramentas](./07-comparativo-ferramentas.md) | Temporal vs Dapr vs Prefect vs Kestra vs Conductor |
| 08 | [AI Agents + Orquestração](./08-ai-agents-orquestracao.md) | Agentic workflows, LLMs e workflow engines |

---

## Conceitos-Chave (Quick Reference)

```
Durable Execution  →  Workflow que sobrevive a falhas, reinicia do ponto exato
Workflow           →  Definição da lógica de negócio (o "quê" e "quando")
Activity           →  Unidade de trabalho executável (chamadas externas, I/O)
Worker             →  Processo que executa Workflows e Activities
Saga               →  Sequência de transações locais com compensações
Orchestration      →  Controlador central dirige o fluxo
Choreography       →  Serviços reagem a eventos de forma autônoma
```

---

## Referências Principais

- [Temporal.io](https://temporal.io)
- [Dapr Docs](https://docs.dapr.io)
- [microservices.io — Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Azure Architecture — Saga](https://learn.microsoft.com/azure/architecture/patterns/saga)
- [AWS Prescriptive Guidance — Saga Choreography](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-choreography.html)
- [Diagrid Blog](https://www.diagrid.io/blog)
- [IntuitionLabs — Agentic AI + Temporal](https://intuitionlabs.ai/articles/agentic-ai-temporal-orchestration)
