# 04 — Dapr Workflows

> Baseado em: docs.dapr.io, Diagrid Blog, dash0.com, blog.openthreatresearch.com

---

## O que é o Dapr?

**Dapr** (Distributed Application Runtime) é um runtime de código aberto que fornece APIs integradas para comunicação, estado, workflows e AI agêntica. Graduado pela CNCF em novembro de 2024, o Dapr é projetado para **desacoplar a lógica da aplicação da infraestrutura** — tornando o código portável entre clouds, on-prem e ambientes híbridos.

> "Workflows give you the option to code as if you were a monolith, but then autoscale like a microservice."
> — Yaron Schneider, co-criador do Dapr

---

## Arquitetura: O Modelo Sidecar

O diferencial fundamental do Dapr é o **padrão sidecar**: em vez de um servidor centralizado separado, o motor de workflow roda **dentro do sidecar Dapr** — um processo auxiliar colocado junto à sua aplicação.

```
┌─────────────────────────────────────────────────┐
│               Pod / Container                    │
│                                                  │
│  ┌────────────────┐    gRPC stream               │
│  │  Sua Aplicação │◄──────────────► ┌──────────┐ │
│  │  (Workflows +  │                 │  Dapr    │ │
│  │   Activities)  │                 │ Sidecar  │ │
│  └────────────────┘                 └────┬─────┘ │
│                                          │       │
└──────────────────────────────────────────┼───────┘
                                           │
                    ┌──────────────────────┼──────────┐
                    │   State Store        │           │
                    │   (Redis/Postgres/   │           │
                    │    Cosmos/etc.)      │           │
                    └─────────────────────┘           │
                                                      │
                         (Kubernetes: Actor Placement │
                          Service coordena instâncias)│
```

### Fluxo de Comunicação

1. Aplicação inicia com o Dapr Workflow SDK
2. SDK abre um **gRPC stream** com o sidecar
3. Sidecar envia **work items** (start workflow, run activity, resume step)
4. Aplicação executa o código e envia resultado de volta pelo mesmo stream
5. **Nenhuma porta inbound** precisa ser aberta na aplicação

---

## Fundações Técnicas

### Dapr Actors → Workflows

O motor de workflows do Dapr é construído sobre o **runtime de Actors**:

- Actors fornecem durabilidade e escalabilidade
- Cada instância de workflow é um Actor
- Actors são distribuídos pelo cluster automaticamente
- O **Durable Task Framework (DTFx)** para Go (`durabletask-go`) fornece a engine de execução

> "Essencialmente pegamos o melhor do durable execution engine da Microsoft, trouxemos para o Dapr, e adaptamos para funcionar com Actors. O resultado foi um engine escalável e de alta performance."
> — Yaron Schneider

### Event Sourcing

Dapr Workflows usa **event sourcing** para durabilidade:
- Cada passo do workflow gera eventos
- Eventos são persistidos no state store
- Replayability permite recuperação de qualquer falha

---

## Padrões de Workflow Suportados

### 1. Task Chaining

Execução sequencial onde o output de cada passo é input do próximo.

```python
@wf.workflow(name='order_processing')
def order_processing_workflow(ctx: DaprWorkflowContext, order_id: str):
    # Sequência de atividades
    inventory = yield ctx.call_activity(check_inventory, input=order_id)
    payment   = yield ctx.call_activity(process_payment, input=order_id)
    shipping  = yield ctx.call_activity(schedule_shipping, input=payment)
    return shipping
```

**Uso:** Pipelines com dependências estritas entre etapas.

### 2. Fan-out / Fan-in

Execução paralela de múltiplas tasks, com agregação dos resultados.

```python
@wf.workflow(name='parallel_processing')
def parallel_workflow(ctx: DaprWorkflowContext, items: list):
    # Dispara todas em paralelo
    tasks = [ctx.call_activity(process_item, input=item) for item in items]
    
    # Aguarda todas completarem
    results = yield wf.when_all(tasks)
    
    # Agrega
    return aggregate(results)
```

**Uso:** Processamento de lotes, chamadas independentes a múltiplos serviços.

### 3. Monitor (Polling Loop)

Workflow que verifica periodicamente o estado de um recurso externo.

```python
@wf.workflow(name='order_monitor')
def monitor_workflow(ctx: DaprWorkflowContext, order_id: str):
    deadline = ctx.current_utc_datetime + timedelta(days=3)
    
    while ctx.current_utc_datetime < deadline:
        status = yield ctx.call_activity(check_order_status, input=order_id)
        
        if status == "delivered":
            yield ctx.call_activity(notify_customer, input=order_id)
            return
        
        # Aguarda antes de verificar novamente (durable sleep)
        yield ctx.create_timer(timedelta(hours=1))
    
    yield ctx.call_activity(escalate_order, input=order_id)
```

**Uso:** Monitoramento de jobs externos, polling de APIs de terceiros.

### 4. External Event Interaction (Human-in-the-Loop)

Workflow aguarda input humano ou evento externo.

```python
@wf.workflow(name='approval_workflow')
def approval_workflow(ctx: DaprWorkflowContext, request_id: str):
    yield ctx.call_activity(send_approval_request, input=request_id)
    
    # Aguarda aprovação (pode levar dias)
    approval = yield ctx.wait_for_external_event("ApprovalReceived")
    
    if approval.approved:
        yield ctx.call_activity(execute_request, input=request_id)
    else:
        yield ctx.call_activity(reject_request, input=request_id)
```

**Uso:** Aprovações humanas, aguardar confirmação de sistemas externos.

### 5. Compensation (Saga)

Implementação do padrão Saga com ações de compensação.

```python
@wf.workflow(name='saga_workflow')
def saga_workflow(ctx: DaprWorkflowContext, order_id: str):
    compensations = []
    
    try:
        yield ctx.call_activity(reserve_inventory, input=order_id)
        compensations.append(lambda: ctx.call_activity(release_inventory, input=order_id))
        
        yield ctx.call_activity(charge_payment, input=order_id)
        compensations.append(lambda: ctx.call_activity(refund_payment, input=order_id))
        
        yield ctx.call_activity(schedule_shipping, input=order_id)
        
    except Exception as e:
        # Executa compensações em ordem reversa
        for compensate in reversed(compensations):
            yield compensate()
        raise
```

---

## APIs de Gerenciamento

O Dapr Workflow expõe APIs HTTP e gRPC para gerenciar workflows:

| Operação | Endpoint |
|----------|----------|
| Iniciar workflow | `POST /v1.0-beta1/workflows/{component}/{name}/start` |
| Consultar estado | `GET /v1.0-beta1/workflows/{component}/{instanceId}` |
| Pausar | `POST /v1.0-beta1/workflows/{component}/{instanceId}/pause` |
| Retomar | `POST /v1.0-beta1/workflows/{component}/{instanceId}/resume` |
| Enviar evento | `POST /v1.0-beta1/workflows/{component}/{instanceId}/raiseEvent/{event}` |
| Terminar | `POST /v1.0-beta1/workflows/{component}/{instanceId}/terminate` |
| Purgar | `DELETE /v1.0-beta1/workflows/{component}/{instanceId}` |

---

## Multi-Application Workflows

A partir do Dapr v1.15, workflows podem orquestrar activities em **aplicações diferentes**:

```
App A (Orquestrador)  →  chama atividades em App B
                      →  chama atividades em App C
```

Mantendo as garantias de segurança, confiabilidade e durabilidade do Dapr entre os limites de aplicação.

---

## Integrações com Building Blocks do Dapr

Uma das maiores vantagens do Dapr é que o Workflow se integra nativamente com todos os outros building blocks:

| Building Block | Uso no Workflow |
|----------------|-----------------|
| **State Management** | Armazenar/recuperar estado de negócio durante o workflow |
| **Pub/Sub** | Publicar eventos como resultado de activities |
| **Service Invocation** | Chamar outros serviços Dapr de dentro de activities |
| **Bindings** | Integrar com sistemas externos (queues, bancos, APIs) |
| **Secrets** | Acessar credenciais de forma segura |
| **Actors** | Base do runtime de workflows |

---

## Comparação: Dapr vs Temporal

| Dimensão | Dapr Workflows | Temporal |
|----------|---------------|----------|
| **Arquitetura** | Sidecar pattern | Servidor separado |
| **Governança** | CNCF (vendor-neutral) | Temporal Technologies |
| **Infraestrutura** | Bring your own DB | Requer Temporal Server |
| **Vendor lock-in** | Mínimo | Baixo (open-source, mas há Temporal Cloud) |
| **Integração** | Nativa com Dapr ecosystem | Via SDK, independente |
| **Maturidade** | v1.15 (2025), estável | Maduro, > 5 anos |
| **Performance** | Centenas de milhões txn/dia (Derivco) | Comprovado em Netflix, Maersk, etc. |
| **Orq + Coreo** | Suporte a ambos | Foco em orquestração |
| **AI Agents** | Dapr Agents (2025) | Suporte nativo via Schedules/Signals |

---

## Observabilidade com OpenTelemetry

O Dapr Workflows se integra com OpenTelemetry para rastreamento distribuído:

> **Desafio**: Workflows são assíncronos — activity executions acontecem em threads diferentes, em momentos diferentes, potencialmente em processos diferentes. O trace context precisa fluir através de todas as camadas.

**Solução**: Propagar `traceparent` através das mensagens de workflow:
1. Frontend injeta trace context na requisição inicial
2. Workflow engine propaga context para cada activity
3. Activities propagam para chamadas HTTP outbound
4. Resultado: **um único trace** por execução de workflow

---

## Dapr Agents

Em março de 2025, o Dapr lançou o **Dapr Agents** — framework para construção de sistemas agênticos sobre Dapr Workflows:

- **Persistent Memory**: estado do agente no state store do Dapr
- **Workflow Orchestration**: interações gerenciadas pelo Dapr Workflow
- **Service Exposure**: endpoints REST para gerenciamento
- **Polyglot**: agentes em qualquer linguagem suportada

---

## Referências

- [Dapr: Workflow Overview](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-overview/)
- [Dapr: Workflow Architecture](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-architecture/)
- [Dapr: Workflow Patterns](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-patterns/)
- [Diagrid: Code like Monolith, Scale like Microservice](https://www.diagrid.io/blog/code-like-a-monolith-scale-like-a-microservice-durable-workflows-for-cloud-native-apps)
- [Dash0: Dapr Workflows + OpenTelemetry](https://www.dash0.com/blog/deep-diving-into-dapr-workflows-and-opentelemetry-tracing-the-invisible-parts-of-asynchronous)
- [Floki: Building AI Agentic Workflow Engine with Dapr](https://blog.openthreatresearch.com/floki-building-an-ai-agentic-workflow-engine-dapr/)
- [Diagrid: 8 Patterns with Dapr Agents](https://www.diagrid.io/blog/building-effective-dapr-agents)
