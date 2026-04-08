# 05 — Padrão Saga

> Baseado em: microservices.io, Temporal Blog, Azure Architecture Center, AWS Prescriptive Guidance, Baeldung, Dev.to

---

## O Problema que o Saga Resolve

Em arquiteturas de microsserviços, cada serviço tem seu próprio banco de dados. Quando uma operação de negócio **precisa modificar dados em múltiplos serviços**, você não pode usar uma transação ACID tradicional.

**Exemplo:** Processar um pedido envolve:
- Reservar estoque (Inventory Service — banco A)
- Cobrar cartão (Payment Service — banco B)
- Registrar pedido (Order Service — banco C)
- Agendar envio (Shipping Service — banco D)

```
Problema com 2PC (Two-Phase Commit):
- Locks distribuídos = alta latência
- Indisponibilidade de um serviço bloqueia tudo
- Não escala para microsserviços independentes
```

---

## O que é o Padrão Saga?

Uma **Saga** é uma sequência de transações locais onde:

1. Cada transação atualiza **seu próprio banco** e publica um evento ou mensagem
2. Se uma transação falha, a Saga executa **transações de compensação** para desfazer as mudanças anteriores

> "Uma Saga garante que ou todas as operações completam com sucesso, ou as transações de compensação correspondentes são executadas para desfazer o trabalho previamente concluído."
> — Baeldung on Computer Science

### Propriedades-Chave

| Propriedade | Descrição |
|-------------|-----------|
| **Sem locks distribuídos** | Cada serviço gerencia sua transação local |
| **Compensação** | Cada operação tem um rollback equivalente |
| **Consistência eventual** | Sistema chega a um estado consistente, não instantaneamente |
| **Idempotência** | Compensações devem ser idempotentes e retryable |

---

## Dois Estilos de Implementação

### 1. Saga Orquestrada (Orchestration-based)

Um **orquestrador central** (ex: Temporal, Dapr Workflow, AWS Step Functions) gerencia toda a sequência:

```
┌───────────────────┐
│   Orquestrador    │
│   (Saga Manager)  │
└───┬───────────────┘
    │
    ├──► Reservar Inventário ──► sucesso
    │        (Compensation: Liberar Inventário)
    │
    ├──► Cobrar Pagamento ──► sucesso
    │        (Compensation: Reembolsar)
    │
    └──► Agendar Envio ──► FALHA!
             │
             ▼
    [Aciona compensações em ordem reversa]
    ──► Reembolsar Pagamento
    ──► Liberar Inventário
```

**Implementação com Temporal (Go):**

```go
func OrderSagaWorkflow(ctx workflow.Context, order Order) error {
    var compensations []func(workflow.Context) error

    // Step 1: Reservar inventário
    err := workflow.ExecuteActivity(ctx, ReserveInventory, order).Get(ctx, nil)
    if err != nil {
        return err
    }
    compensations = append(compensations, func(ctx workflow.Context) error {
        return workflow.ExecuteActivity(ctx, ReleaseInventory, order).Get(ctx, nil)
    })

    // Step 2: Cobrar pagamento
    err = workflow.ExecuteActivity(ctx, ChargePayment, order).Get(ctx, nil)
    if err != nil {
        // Executa compensações
        for i := len(compensations) - 1; i >= 0; i-- {
            compensations[i](ctx)
        }
        return err
    }
    compensations = append(compensations, func(ctx workflow.Context) error {
        return workflow.ExecuteActivity(ctx, RefundPayment, order).Get(ctx, nil)
    })

    // Step 3: Agendar envio
    err = workflow.ExecuteActivity(ctx, ScheduleShipping, order).Get(ctx, nil)
    if err != nil {
        for i := len(compensations) - 1; i >= 0; i-- {
            compensations[i](ctx)
        }
        return err
    }

    return nil
}
```

### 2. Saga Coreografada (Choreography-based)

Não há orquestrador. Cada serviço reage a eventos e publica novos eventos:

```
Order Service
  │── publica "OrderCreated" ──►
                               Event Bus
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
            Inventory         Payment        (aguarda)
           Service             Service
                │                │
    reserva estoque        cobra cartão
                │                │
    publica "StockReserved"  publica "PaymentProcessed"
                                 │
                                 ▼
                          Shipping Service
                               agenda envio

[Se Payment falha]
  │── publica "PaymentFailed" ──►
                               Event Bus
                                  │
                                  ▼
                          Inventory Service
                        libera estoque reservado
```

**Implementação com AWS EventBridge (Python/Lambda):**

```python
# Inventory Service - reage a OrderCreated
def handle_order_created(event):
    order_id = event['detail']['orderId']
    if reserve_stock(order_id):
        publish_event('inventory-bus', 'StockReserved', {'orderId': order_id})
    else:
        publish_event('inventory-bus', 'StockReservationFailed', {'orderId': order_id})

# Payment Service - reage a StockReserved
def handle_stock_reserved(event):
    order_id = event['detail']['orderId']
    if charge_payment(order_id):
        publish_event('payment-bus', 'PaymentProcessed', {'orderId': order_id})
    else:
        publish_event('payment-bus', 'PaymentFailed', {'orderId': order_id})
        # Inventory precisa ouvir PaymentFailed e liberar o estoque

# Compensation: Inventory ouve PaymentFailed
def handle_payment_failed(event):
    order_id = event['detail']['orderId']
    release_stock(order_id)
    publish_event('inventory-bus', 'StockReleased', {'orderId': order_id})
```

---

## Comparativo: Orquestrada vs Coreografada

| Critério | Orquestrada | Coreografada |
|----------|-------------|--------------|
| **Visibilidade** | Alta — fluxo explícito no orquestrador | Baixa — fluxo emerge das reações |
| **Debugging** | Fácil | Difícil (rastrear logs distribuídos) |
| **Acoplamento** | Orquestrador acoplado aos serviços | Serviços só acoplados aos eventos |
| **Lógica condicional** | Natural (código imperativo) | Complexa (eventos diferentes por caminho) |
| **Compensações** | Gerenciadas pelo orquestrador | Responsabilidade de cada serviço |
| **Single PoF** | Sim (orquestrador) | Não |
| **Escalabilidade** | Orquestrador pode ser gargalo | Naturalmente elástico |
| **Cyclic deps** | Improvável | Risco real |

---

## Transações de Compensação

### Requisitos

Toda compensação deve ser:

1. **Idempotente** — pode ser executada múltiplas vezes sem efeitos colaterais
2. **Retryable** — pode ser tentada novamente em caso de falha
3. **Reversível semanticamente** — não é um rollback técnico, mas uma operação de negócio inversa

### Exemplos

| Operação Original | Compensação |
|-------------------|-------------|
| Reservar estoque | Liberar reserva |
| Cobrar cartão | Emitir reembolso |
| Criar pedido | Cancelar pedido |
| Registrar transferência | Estornar transferência |
| Enviar email de confirmação | Enviar email de cancelamento |

> **Nota importante:** Emails e notificações geralmente **não são compensáveis** (você não pode "desfazer" um email já enviado). Nesses casos, uma compensação semântica seria enviar um novo email informando o cancelamento.

---

## Desafios Comuns e Soluções

### Dual-Write Problem

**Problema:** Como garantir que a atualização do banco E a publicação do evento aconteçam atomicamente?

**Solução: Transactional Outbox Pattern**
```sql
-- Tudo em uma transação:
BEGIN;
  UPDATE orders SET status = 'reserved' WHERE id = ?;
  INSERT INTO outbox (event_type, payload) VALUES ('OrderReserved', ?);
COMMIT;

-- Processo separado (relay):
SELECT * FROM outbox WHERE published = false;
-- Para cada evento: publica no broker, atualiza published = true
```

### Leituras "Sujas" (Falta de Isolamento)

Como Sagas não têm o "I" do ACID, transações concorrentes podem ver estados intermediários.

**Countermeasures:**
- **Semantic lock**: marcar recurso como "pending" durante processamento
- **Pessimistic lock**: bloquear o recurso na camada de negócio
- **Reread value**: re-ler o valor antes de commitar para detectar conflitos

### Ausência de Ações de Compensação

**Erro comum:** implementar a Saga sem definir a compensação para cada passo.

**Prática recomendada:** Para cada activity, defina:
```
Activity: X
Input: y
Compensation: X_undo(y)
Idempotency key: {workflow_id}:{step_name}
```

---

## Saga Log (Execution Coordinator)

O **Saga Execution Coordinator (SEC)** é o componente que registra o progresso da Saga:

```
Saga Log:
  [T1: OrderCreated        → completed]
  [T2: InventoryReserved   → completed]
  [T3: PaymentCharged      → FAILED]
  [C2: InventoryReleased   → pending]  ← compensação
  [C1: OrderCancelled      → pending]  ← compensação
```

Em motores como o Temporal, o Saga Log é o próprio event history do workflow — acessível via Web UI e APIs.

---

## Frameworks para Saga

| Framework | Linguagem | Abordagem |
|-----------|-----------|-----------|
| **Temporal** | Go, Java, Python, TS | Durable execution, código imperativo |
| **Dapr Workflows** | Polyglot | Sidecar, CNCF |
| **MassTransit** | .NET | State machine, RabbitMQ/Azure SB |
| **Camunda** | Java | BPMN visual + código |
| **AWS Step Functions** | JSON (ASL) | Serverless, gerenciado |
| **Eventuate** | Java | Event sourcing nativo |
| **Axon Framework** | Java | CQRS + Event Sourcing |

---

## Referências

- [microservices.io: Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Temporal: Mastering Saga Patterns](https://temporal.io/blog/mastering-saga-patterns-for-distributed-transactions-in-microservices)
- [Azure: Saga Design Pattern](https://learn.microsoft.com/azure/architecture/patterns/saga)
- [AWS: Saga Choreography](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-choreography.html)
- [Baeldung: Saga Pattern in Microservices](https://www.baeldung.com/cs/saga-pattern-microservices)
- [Dev.to: Transactions in Microservices Part 3 — Saga + Temporal](https://dev.to/federico_bevione/transactions-in-microservices-part-3-saga-pattern-with-orchestration-and-temporalio-3e17)
- [Alok: Saga Orchestration vs Choreography](https://aloknecessary.github.io/blogs/saga-orchestration-vs-choreography/)
