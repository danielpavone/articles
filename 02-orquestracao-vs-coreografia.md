# 02 — Orquestração vs Coreografia

> Baseado em: Orkes.io, Jack Vanlightly, Microsoft, AWS, alokblog

---

## O Problema Central

Ao construir sistemas distribuídos, a pergunta inevitável é: **como os serviços se comunicam e coordenam seu trabalho?**

Existem dois grandes paradigmas:

| | **Orquestração** | **Coreografia** |
|--|---|---|
| **Controle** | Centralizado | Descentralizado |
| **Analogia** | Maestro de orquestra | Dançarinos reagindo à música |
| **Comunicação** | Comandos diretos | Eventos publicados |
| **Visibilidade** | Alta — tudo passa pelo orquestrador | Baixa — fluxo emerge das reações |
| **Acoplamento** | Orquestrador conhece os participantes | Serviços só conhecem os eventos |
| **Debugabilidade** | Fácil — há um ponto central | Difícil — precisa rastrear logs distribuídos |
| **Escala** | Orquestrador pode ser gargalo | Naturalmente elástico |

---

## Orquestração

### Como funciona

Um **controlador central** (o orquestrador) decide a ordem das operações e invoca cada serviço participante. Os participantes não sabem da existência uns dos outros — apenas respondem ao orquestrador.

```
        ┌──────────────────┐
        │   Orquestrador   │
        └────┬──────┬──────┘
             │      │
     invoca  │      │  invoca
             ▼      ▼
      [Serviço A]  [Serviço B]
```

### Fluxo típico (pedido de e-commerce):
1. Orquestrador recebe pedido
2. Orquestrador invoca **Serviço de Inventário** → reserva produto
3. Orquestrador invoca **Serviço de Pagamento** → cobra cartão
4. Orquestrador invoca **Serviço de Envio** → agenda entrega
5. Se qualquer etapa falhar → orquestrador aciona compensações

### Vantagens:
- **Visibilidade total** do estado do processo
- **Controle explícito** sobre sequência, retries e erros
- **Fácil de debugar** — há um log centralizado
- **Compensações** são gerenciadas por um lugar só
- **Lógica condicional** é natural (if/else, loops)

### Desvantagens:
- Orquestrador pode ser **single point of failure**
- **Acoplamento** entre orquestrador e serviços
- Pode se tornar **gargalo** sob alta carga
- Lógica de orquestração pode crescer em complexidade

---

## Coreografia

### Como funciona

Não há controlador central. Cada serviço **publica eventos** após concluir seu trabalho, e outros serviços **reagem** a esses eventos de forma autônoma.

```
[Serviço A] → publica "OrderCreated" → [Event Bus]
                                             │
                    ┌────────────────────────┤
                    ▼                        ▼
            [Serviço B]              [Serviço C]
         (Inventário reage)       (Pagamento reage)
```

### Fluxo típico:
1. Order Service publica **"OrderCreated"**
2. Inventory Service reage, reserva estoque, publica **"StockReserved"**
3. Payment Service reage, cobra cartão, publica **"PaymentProcessed"**
4. Shipping Service reage, agenda entrega

### Vantagens:
- **Desacoplamento** — serviços evoluem independentemente
- **Escalabilidade natural** — sem gargalo central
- **Resiliência** — sem single point of failure
- **Fácil de adicionar/remover** serviços (só se inscrevem nos eventos certos)

### Desvantagens:
- **Visibilidade fragmentada** — fluxo não é explícito em nenhum lugar
- **Debugging difícil** — rastrear um fluxo requer correlacionar logs de N serviços
- **Lógica condicional complexa** — requer publicar eventos diferentes por caminho
- **Compensações** são responsabilidade de cada serviço individualmente
- **Problema do dual-write** — banco + evento devem ser escritos atomicamente

---

## Quando Usar Cada Um?

### Use Orquestração quando:
- O fluxo tem **muitas etapas** com dependências claras
- A **lógica condicional** é complexa (if/else, loops, branching)
- **Visibilidade e auditoria** são críticas (financeiro, saúde, compliance)
- Você precisa de **compensações** bem definidas em caso de falha
- O time precisa entender o fluxo end-to-end rapidamente

### Use Coreografia quando:
- Os serviços são **naturalmente event-driven**
- O acoplamento entre serviços deve ser **mínimo**
- O sistema precisa escalar para **volumes muito altos**
- As etapas podem ocorrer **em paralelo e independentemente**
- Você adiciona/remove serviços com frequência

---

## Trade-offs Detalhados em Produção

### O Problema do Dual-Write

Em ambos os padrões, quando um serviço atualiza seu banco E publica um evento, existe risco de inconsistência se uma das operações falhar.

**Solução:** Transactional Outbox Pattern
```
Transação no banco:
  1. Escreve dado principal
  2. Escreve evento na tabela "outbox" (mesmo banco, mesma transação)

Processo separado (relay):
  - Lê eventos da outbox
  - Publica no broker
  - Remove da outbox após confirmação
```

### Idempotência é Obrigatória

Brokers de mensagens garantem **at-least-once delivery** — sua mensagem pode ser entregue mais de uma vez. Toda operação em qualquer step deve ser idempotente:

```go
// Ruim: não idempotente
func chargeCard(orderId string) error {
    return paymentAPI.Charge(orderId)  // pode cobrar duas vezes!
}

// Bom: verifica se já foi processado
func chargeCard(orderId string) error {
    if already, _ := db.WasCharged(orderId); already {
        return nil  // idempotente
    }
    return paymentAPI.Charge(orderId)
}
```

### Timeout e Retry Policies

| Abordagem | Orquestração | Coreografia |
|-----------|-------------|-------------|
| Retry logic | Centralizada no orquestrador | Distribuída em cada consumer |
| Timeout | Por step, configurável | Depende do broker (DLQ) |
| Visibilidade | Em um lugar | Espalhada por logs |

---

## A Realidade: Sistemas Híbridos

Na prática, sistemas maduros combinam os dois padrões:

> "You can combine orchestration and choreography. Use orchestration for the critical path — the steps that must succeed in order. Use choreography for auxiliary actions — notifications, analytics, cache invalidation — that don't block the main flow."
> — Jack Vanlightly, Coordinated Progress

```
         ┌────────────────────┐
         │   Orquestrador     │  ← controla fluxo crítico
         └──┬────────────┬────┘
            │            │
    [Inventário]    [Pagamento]
            │
            └── publica "OrderShipped" → [Event Bus]
                                               │
                          ┌───────────────────┤
                          ▼                   ▼
                  [Notificações]        [Analytics]
                  (coreografia)        (coreografia)
```

---

## Referências

- [Orkes.io: Orchestration vs Choreography](https://orkes.io/blog/workflow-orchestration-vs-choreography/)
- [Jack Vanlightly: Coordinated Progress Part 1](https://jack-vanlightly.com/blog/2025/6/11/coordinated-progress-part-1)
- [Azure: Saga Design Pattern](https://learn.microsoft.com/azure/architecture/patterns/saga)
- [AWS: Saga Choreography](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-choreography.html)
- [Alok: Saga Orchestration vs Choreography](https://aloknecessary.github.io/blogs/saga-orchestration-vs-choreography/)
- [Temporal: Durable Execution](https://temporal.io/blog/durable-execution-in-distributed-systems-increasing-observability)
