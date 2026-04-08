# 01 — Fundamentos de Orquestração de Workflows

> Baseado em: Temporal.io docs, Hossted OSSpedia, Jack Vanlightly Blog, IntuitionLabs

---

## O que é Orquestração de Workflows?

Orquestração de workflows é o mecanismo pelo qual um sistema coordena a execução de tarefas distribuídas, garantindo que cada etapa seja executada na ordem correta, com tratamento de falhas e recuperação automática.

Em arquiteturas de microsserviços, à medida que a lógica de negócio é particionada em serviços independentes, surge a necessidade de **coordenar interações entre esses serviços** de forma confiável. É aqui que entram os motores de orquestração.

---

## Por que isso é difícil?

Sistemas distribuídos enfrentam desafios que sistemas monolíticos não têm:

- **Falhas parciais**: um serviço pode cair no meio de uma transação
- **Latência de rede**: chamadas remotas podem falhar ou demorar
- **Estado distribuído**: dados estão espalhados em múltiplos bancos
- **Consistência eventual**: não é possível usar ACID transactions entre serviços
- **Ordenação de eventos**: eventos podem chegar fora de ordem

> "Distributed systems break, APIs fail, networks flake, and services crash. That's not your problem anymore."
> — temporal.io

---

## Durable Execution (Execução Durável)

**Durable Execution** é o conceito central dos motores modernos de orquestração. A ideia é simples: **o estado de um workflow é persistido automaticamente a cada passo**, de modo que se o processo falhar (crash, restart, network timeout), ele pode ser **retomado exatamente de onde parou**.

### Como funciona:

1. Cada passo do workflow é registrado em um **event log** persistente
2. Em caso de falha, o engine **replays** o histórico de eventos
3. O código do workflow é re-executado, mas ações já concluídas são puladas
4. O workflow continua a partir do **último ponto de progresso**

### Analogia:

> Durable Execution é para workflows imperativos o que o Kafka é para workflows orientados a eventos. Ambos persistem progresso de forma durável, permitindo recuperação de falhas.
> — Jack Vanlightly, "Coordinated Progress"

### Propriedades garantidas:

| Propriedade | Descrição |
|-------------|-----------|
| At-least-once | Tarefas são executadas ao menos uma vez |
| Idempotência | Activities devem ser idempotentes para re-execuções seguras |
| Replay | Histórico pode ser reproduzido para recuperação ou debug |
| State persistence | Estado é salvo automaticamente sem código extra |

---

## Primitivos Fundamentais

### Workflow
Define a lógica de negócio de alto nível. É o "script" que orquestra a execução das Activities. Pode:
- Executar Activities em sequência ou paralelo
- Aguardar eventos externos (Signals)
- Dormir por dias ou meses sem consumir recursos
- Ser pausado, cancelado ou reiniciado

### Activity
Uma função que executa trabalho real — I/O, chamadas de API, operações em banco. Características:
- Possui timeout configurável
- Retries automáticos com backoff exponencial
- Pode ser longa (heartbeat para indicar progresso)
- Deve ser **idempotente** sempre que possível

### Worker
Processo da aplicação que executa o código de Workflows e Activities. Os Workers:
- Se conectam ao servidor de orquestração via task queues
- São stateless (o estado fica no servidor)
- Podem ser escalados horizontalmente
- Nunca recebem chamadas diretas — **puxam tarefas** da fila

### Task Queue
Canal de comunicação entre o servidor de orquestração e os Workers. Permite:
- Roteamento de workflows para Workers específicos
- Balanceamento de carga
- Versionamento de workflows em produção

---

## Categorias de Motores de Orquestração

### Por abordagem técnica:

| Tipo | Exemplos | Característica |
|------|----------|----------------|
| **Durable Execution** | Temporal, Dapr Workflows, Azure Durable Functions | Código imperativo com estado persistido automaticamente |
| **DAG-based** | Apache Airflow, Prefect, Dagster | Grafos dirigidos acíclicos, foco em pipelines de dados |
| **BPMN** | Camunda, Activiti | Notação visual de processos de negócio |
| **Serverless** | AWS Step Functions, Azure Logic Apps | YAML/JSON declarativo, gerenciado pelo cloud |
| **Event-driven** | Kafka Streams, AWS EventBridge | Coreografia reativa, sem orquestrador central |

---

## Quando Usar um Motor de Orquestração?

**Use quando:**
- O processo envolve múltiplos serviços ou sistemas externos
- Há necessidade de retries, timeouts e compensações
- O fluxo pode durar minutos, horas, dias ou semanas
- Você precisa de visibilidade do estado do processo
- Há lógica condicional ou paralela entre etapas
- Consistência entre serviços é crítica (transações distribuídas)

**Evite quando:**
- O processo é simples e roda em um único serviço
- A latência adicionada pelo orquestrador é inaceitável
- O fluxo é puramente reativo e sem estado central

---

## O Modelo Mental: Monolito vs Microsserviços vs Orquestração

```
Monolito:
  [Código] → chamada local → [Código] → banco local
  ✓ ACID, fácil de debugar
  ✗ Escalabilidade limitada, acoplamento

Microsserviços sem orquestração:
  [Serviço A] → HTTP → [Serviço B] → mensagem → [Serviço C]
  ✓ Escalável, independente
  ✗ Sem visibilidade, falhas silenciosas, estado espalhado

Microsserviços com orquestração:
  [Orquestrador] → task queue → [Worker A]
                 → task queue → [Worker B]
  ✓ Visibilidade total, retries, estado centralizado
  ✓ "Code like a monolith, scale like a microservice"
```

---

## Referências

- [Temporal: How it Works](https://temporal.io/how-it-works)
- [Hossted: Temporal Revolutionizing Workflow Orchestration](https://hossted.com/knowledge-base/osspedia/devops/application-development/temporal-revolutionizing-workflow-orchestration/)
- [Jack Vanlightly: Coordinated Progress Part 1](https://jack-vanlightly.com/blog/2025/6/11/coordinated-progress-part-1)
- [IntuitionLabs: Agentic AI + Temporal](https://intuitionlabs.ai/articles/agentic-ai-temporal-orchestration)
- [Temporal: Mastering Durable Execution](https://temporal.io/blog/durable-execution-in-distributed-systems-increasing-observability)
