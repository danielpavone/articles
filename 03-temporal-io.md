# 03 — Temporal.io

> Baseado em: temporal.io, Suraj Subramanian (Medium), Rajesh Vinayagam (Medium), GitHub temporalio, IntuitionLabs

---

## O que é o Temporal?

Temporal é uma **plataforma de durable execution** open-source para construir aplicações escaláveis sem sacrificar produtividade ou confiabilidade. Originalmente um fork do **Cadence** (Uber), é desenvolvido pela Temporal Technologies — fundada pelos criadores do Cadence, Maxim Fateev e Samar Abbas.

**O modelo central**: você escreve código de negócio normal (Go, Java, Python, TypeScript, .NET...) e o Temporal garante que esse código **sempre rode até o fim**, mesmo que o processo crash, a rede falhe ou o servidor reinicie.

---

## Arquitetura

### Visão Geral

```
┌─────────────────────────────────────────────────┐
│                  Temporal Cluster                │
│  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  Frontend    │  │    History Service       │  │
│  │  Service     │  │  (estado dos workflows)  │  │
│  │  (API GW)    │  └──────────────────────────┘  │
│  └──────┬───────┘  ┌──────────────────────────┐  │
│         │          │    Matching Service      │  │
│         │          │    (task queues)         │  │
│         │          └──────────────────────────┘  │
│  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  Persistence │  │    Elasticsearch         │  │
│  │  (SQL/Cass.) │  │    (search opcional)     │  │
│  └──────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────┘
          ▲                        ▲
          │                        │
   ┌──────┴──────────────────────┐
   │       Workers (sua app)      │
   │   ┌──────────┐ ┌──────────┐  │
   │   │ Workflow │ │Activity  │  │
   │   │  Code    │ │  Code    │  │
   │   └──────────┘ └──────────┘  │
   └────────────────────────────┘
```

### Componentes Internos do Cluster

| Componente | Função |
|------------|--------|
| **Frontend Service** | API Gateway — ponto de entrada para CLI, SDK, Web UI |
| **History Service** | Persiste e gerencia o histórico de execução de cada workflow |
| **Matching Service** | Gerencia task queues e atribui tarefas aos Workers |
| **Persistence Layer** | MySQL, PostgreSQL ou Cassandra para armazenar estado |
| **Elasticsearch** | Opcional — busca avançada e filtragem de workflows |

### Direção das Conexões

> Todas as conexões são **unidirecionais da aplicação para o Temporal**. Você nunca precisa abrir portas de firewall para o Temporal acessar sua aplicação.

---

## Primitivos do Temporal

### 1. Workflow

O Workflow define a lógica de negócio de alto nível. Escrito em linguagem de programação normal, o código parece imperativo mas é **durável por natureza**.

```python
@workflow.defn
class OrderWorkflow:
    @workflow.run
    async def run(self, order_id: str) -> str:
        # Cada activity tem retry automático
        await workflow.execute_activity(
            reserve_inventory,
            order_id,
            start_to_close_timeout=timedelta(seconds=30),
        )
        await workflow.execute_activity(
            charge_payment,
            order_id,
            start_to_close_timeout=timedelta(seconds=60),
        )
        await workflow.execute_activity(
            schedule_shipping,
            order_id,
            start_to_close_timeout=timedelta(seconds=30),
        )
        return "completed"
```

**Garantias do Workflow:**
- Estado persistido automaticamente
- Pode "dormir" por dias/semanas: `await workflow.sleep(timedelta(days=30))`
- Pode aguardar eventos externos
- Pode ser consultado sem mudar estado (Query)
- Pode receber eventos em tempo real (Signal)

### 2. Activity

A unidade de trabalho real. É onde você faz I/O, chama APIs externas, acessa bancos.

```python
@activity.defn
async def charge_payment(order_id: str) -> None:
    result = await payment_api.charge(order_id)
    if not result.success:
        raise ApplicationError("Payment failed", non_retryable=True)
```

**Retry Policy:**
```python
RetryPolicy(
    initial_interval=timedelta(seconds=1),
    backoff_coefficient=2.0,
    maximum_attempts=10,
    maximum_interval=timedelta(minutes=5),
    non_retryable_error_types=["PaymentDeclinedError"]
)
```

### 3. Signal

Permite enviar dados ou eventos para um workflow em execução — comunicação bidirecional assíncrona.

```python
# Enviando sinal
await client.get_workflow_handle(workflow_id).signal("approve", ApprovalData(...))

# Recebendo no workflow
@workflow.signal
async def approve(self, data: ApprovalData) -> None:
    self._approval = data
```

### 4. Query

Consulta o estado atual de um workflow sem alterá-lo (leitura síncrona).

```python
@workflow.query
def get_status(self) -> str:
    return self._current_status
```

### 5. Schedule

Substitui cron jobs com garantias de durable execution:

```python
# Executa todo dia às 9h com retry automático
await client.create_schedule(
    "daily-report",
    Schedule(
        action=ScheduleActionStartWorkflow(ReportWorkflow.run, ...),
        spec=ScheduleSpec(cron_expressions=["0 9 * * *"]),
    ),
)
```

### 6. Child Workflow

Permite que um workflow inicie sub-workflows independentes, com seus próprios históricos e políticas de retry.

---

## Modelo de Execução (Replay)

O mecanismo central do Temporal é o **deterministic replay**:

1. Cada ação do workflow (executar activity, esperar signal, dormir) gera um **event** no histórico
2. O histórico é persistido no servidor
3. Se o Worker cair, o Temporal re-executa o código do workflow do início
4. Mas ações **já concluídas** são restauradas do histórico (não re-executadas)
5. O workflow continua do ponto de parada

```
Evento 1: WorkflowExecutionStarted
Evento 2: ActivityTaskScheduled (reserve_inventory)
Evento 3: ActivityTaskStarted
Evento 4: ActivityTaskCompleted ✓
Evento 5: ActivityTaskScheduled (charge_payment)  ← crash aqui
  [Worker reinicia]
  [Replay: eventos 1-4 são restaurados sem re-executar]
  [Continua a partir do evento 5]
```

**Implicação importante:** código de workflow deve ser **determinístico** — não use `random()`, `datetime.now()` diretamente, ou chamadas I/O dentro do workflow (apenas dentro de activities).

---

## Task Queues

Task queues são o mecanismo de despacho entre o servidor Temporal e os Workers:

```
Temporal Server          Workers
     │                      │
     │  ← poll task queue ──┘ (Workers puxam trabalho)
     │
     ├── task queue "payment-service" → Payment Workers
     ├── task queue "inventory-service" → Inventory Workers
     └── task queue "shipping-service" → Shipping Workers
```

**Usos avançados:**
- Roteamento de workflows para workers específicos (por região, por versão)
- Blue/green deployment de workflows
- Rate limiting por queue

---

## Casos de Uso Ideais

| Domínio | Exemplo |
|---------|---------|
| **Finanças** | Transferências bancárias, processamento de pagamentos, SAGA de pedidos |
| **E-commerce** | Order fulfillment, retornos, reembolsos |
| **SaaS / Onboarding** | Fluxo de ativação de conta com múltiplos passos e aprovações humanas |
| **CI/CD** | Pipelines de deploy com rollback automático |
| **AI Agents** | Orchestração de agentes LLM com memória persistente e retries |
| **Healthcare** | Fluxos de aprovação clínica, coordenação de exames |
| **Logistics** | Rastreamento de envios, coordenação de transportadoras |

---

## Temporal vs Alternativas

| Dimensão | Temporal | AWS Step Functions | Azure Durable Functions |
|----------|----------|--------------------|------------------------|
| **Código** | Linguagem nativa (Go, Java, Python...) | JSON/YAML (ASL) | C#, JS |
| **Cloud** | Multi-cloud / self-hosted | AWS only | Azure only |
| **Durabilidade** | Event sourcing nativo | Gerenciado pela AWS | Gerenciado pela Azure |
| **Observabilidade** | Web UI + history | CloudWatch | Azure Monitor |
| **Vendor lock-in** | Nenhum | Alto | Alto |
| **Escala** | Horizontal ilimitada | Gerenciada | Gerenciada |

---

## SDKs Disponíveis

- **Go** — SDK mais maduro
- **Java** — completo, Workflow e Activity annotations
- **Python** — async/await nativo
- **TypeScript/JavaScript** — suporte completo
- **.NET (C#)** — integração com ecosystem Microsoft
- **Ruby** — suporte experimental

---

## Casos Reais de Uso em Produção

- **NVIDIA** — gerencia frota de GPUs em múltiplos clouds
- **ANZ Bank** — origination de home loans (projeto que levaria 1 ano entregue em semanas)
- **Maersk** — operações de logistics (feature delivery: de 60-80 dias para 5-10 dias)
- **Netflix** — orquestração de workflows internos
- **DigitalOcean** — sincronização de transações distribuídas de storage

---

## Temporal para AI Agents

Com a crescente adoção de LLMs e agentes autônomos, o Temporal ganhou relevância como **runtime de agentes**:

```python
@workflow.defn
class TradingAgentWorkflow:
    @workflow.run
    async def run(self) -> None:
        while True:
            # Acorda periodicamente (via Schedule)
            market_data = await workflow.execute_activity(
                fetch_market_data,
                start_to_close_timeout=timedelta(seconds=10)
            )
            decision = await workflow.execute_activity(
                llm_decide_trade,  # chamada ao LLM
                market_data,
                start_to_close_timeout=timedelta(seconds=30)
            )
            if decision.should_trade:
                await workflow.execute_activity(
                    execute_order,
                    decision,
                    start_to_close_timeout=timedelta(seconds=10)
                )
            await workflow.sleep(timedelta(seconds=30))
```

**Benefícios para agentes:**
- **Schedules** — agentes proativos sem cron jobs customizados
- **Signals & Queries** — human-in-the-loop sem pausar o loop do agente
- **UI** — cada ação do agente é auditável no histórico
- **Durabilidade** — agentes 24×7 sem perda de estado

---

## Referências

- [Temporal: How it Works](https://temporal.io/how-it-works)
- [Suraj Subramanian: Temporal in Microservices](https://medium.com/@surajsub_68985/temporal-revolutionizing-workflow-orchestration-in-microservices-architectures-f8265afa4dc0)
- [Rajesh Vinayagam: Taming Chaos with Temporal](https://contact-rajeshvinayagam.medium.com/temporal-io-taming-the-chaos-of-distributed-systems-with-elegant-workflow-orchestration-d8fdc38e42e6)
- [GitHub: temporalio/temporal](https://github.com/temporalio/temporal)
- [Temporal Blog: Orchestrating Ambient Agents](https://temporal.io/blog/orchestrating-ambient-agents-with-temporal)
- [Temporal Blog: Durable Execution](https://temporal.io/blog/durable-execution-in-distributed-systems-increasing-observability)
- [IntuitionLabs: Agentic AI Workflows](https://intuitionlabs.ai/articles/agentic-ai-temporal-orchestration)
