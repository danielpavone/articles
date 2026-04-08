# 06 — Padrões de Workflow

> Baseado em: Dapr Docs, Diagrid Blog, Temporal Blog

---

## Os 6 Padrões Fundamentais

### 1. Task Chaining (Encadeamento de Tarefas)

**Descrição:** Tarefas executadas em sequência. O output de uma é o input da próxima.

```
[Step 1] → [Step 2] → [Step 3] → resultado
```

**Quando usar:** ETL pipelines, onboarding sequencial, aprovações em cadeia.

**Pseudocódigo:**
```python
def pipeline(ctx, input):
    a = yield ctx.call(step_a, input)
    b = yield ctx.call(step_b, a)
    c = yield ctx.call(step_c, b)
    return c
```

---

### 2. Fan-out / Fan-in (Paralelização + Agregação)

**Descrição:** Dispara múltiplas tasks em paralelo, depois agrega os resultados.

```
         ┌──► [Task A] ──┐
[Input] ─┼──► [Task B] ──┼──► [Aggregate] → resultado
         └──► [Task C] ──┘
```

**Quando usar:** Chamadas independentes a múltiplos serviços, processar lotes de itens em paralelo.

**Pseudocódigo:**
```python
def parallel_workflow(ctx, items):
    tasks = [ctx.call(process_item, item) for item in items]
    results = yield when_all(tasks)
    return sum(results)
```

---

### 3. Monitor (Long-running Poll)

**Descrição:** Workflow que periodicamente verifica o estado de algo externo, com durable sleep entre verificações.

```
[Start] → [Check] → aguarda 1h → [Check] → aguarda 1h → ... → [Done]
```

**Quando usar:** Polling de APIs externas, monitoramento de jobs, aguardar aprovações com deadline.

```python
def monitor(ctx, resource_id):
    deadline = ctx.now() + timedelta(days=3)
    while ctx.now() < deadline:
        status = yield ctx.call(check_status, resource_id)
        if status == "complete":
            return
        yield ctx.sleep(timedelta(hours=1))
    yield ctx.call(escalate, resource_id)
```

---

### 4. External Event / Human-in-the-Loop

**Descrição:** Workflow pausa e aguarda input externo (humano ou sistema) antes de continuar.

```
[Inicia] → [Pede aprovação] → ⏸️ aguarda sinal → [Aprovado] → [Executa]
                                                  [Rejeitado] → [Cancela]
```

**Quando usar:** Aprovações humanas, confirmação de usuário final, integração com sistemas lentos.

---

### 5. Compensation (Saga / Rollback)

**Descrição:** Para cada step com efeito colateral, define uma ação de compensação. Em caso de falha, executa compensações em ordem reversa.

```
[Step A] ✓ → [Step B] ✓ → [Step C] ✗
                  ↓ (compensate B)
             [Step B undo] → [Step A undo]
```

**Quando usar:** Transações distribuídas, qualquer fluxo multi-serviço com necessidade de consistência.

---

### 6. Durable Timers / Scheduling

**Descrição:** Workflows que executam em schedules recorrentes ou após esperas longas.

```python
# Cobra assinatura mensalmente por 1 ano
def subscription(ctx, user_id):
    for month in range(12):
        yield ctx.sleep(timedelta(days=30))
        yield ctx.call(charge_subscription, user_id)
```

**Quando usar:** Cobrança recorrente, lembretes periódicos, SLA monitoring.

---

## Padrões Avançados para AI Agents

Baseado no trabalho do Diagrid com Dapr Agents e Temporal com agentes:

### Prompt Chaining

Decompõe uma tarefa complexa em múltiplos calls ao LLM, onde cada call processa o output do anterior.

```
[Tarefa] → [LLM 1: extrair] → [LLM 2: analisar] → [LLM 3: formatar] → [Output]
```

**Quando usar:** Tarefas que requerem etapas de raciocínio distintas e validação entre etapas.

### Orchestrator-Workers

Um LLM orquestrador planeja e delega sub-tarefas para worker LLMs especializados.

```
[Input] → [LLM Orquestrador] → plano dinâmico
                │
     ┌──────────┼──────────┐
     ▼          ▼          ▼
[Worker A]  [Worker B]  [Worker C]
(pesquisa) (cálculo)   (redação)
     └──────────┼──────────┘
                ▼
          [Síntese final]
```

### Routing (Roteamento Condicional)

LLM decide para qual especialista direcionar a tarefa.

```
[Input] → [LLM Router] → detecta intenção
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
[Agente   [Agente    [Agente
 Vendas]   Suporte]   Técnico]
```

### Parallelization (Múltiplas Perspectivas)

Múltiplos LLMs processam o mesmo input em paralelo, com votação ou agregação dos resultados.

```
         ┌──► [LLM perspectiva A] ──┐
[Input] ─┼──► [LLM perspectiva B] ──┼──► [Aggregator LLM]
         └──► [LLM perspectiva C] ──┘
```

---

# 07 — Comparativo de Ferramentas

> Baseado em: procycons.com (Kestra vs Temporal vs Prefect), Diagrid Blog

---

## Landscape 2025

```
┌────────────────────────────────────────────────────────┐
│                  Workflow Orchestration                 │
├────────────────┬────────────────┬───────────────────────┤
│ Durable Exec.  │ Data Pipelines │ BPM / Visual          │
├────────────────┼────────────────┼───────────────────────┤
│ Temporal       │ Apache Airflow │ Camunda               │
│ Dapr Workflows │ Prefect        │ Activiti              │
│ Azure Durable  │ Dagster        │ jBPM                  │
│ AWS Step Fn.   │ Kestra         │                       │
│ Conductor      │ Luigi          │                       │
└────────────────┴────────────────┴───────────────────────┘
```

---

## Comparativo Principal

### Temporal vs Dapr Workflows vs Prefect vs Kestra

| Dimensão | Temporal | Dapr Workflows | Prefect | Kestra |
|----------|----------|----------------|---------|--------|
| **Foco** | Mission-critical | Cloud-native / polyglot | ML / Data Science | ETL / Data Pipelines |
| **Configuração** | Código imperativo | Código imperativo | Python-nativo | YAML declarativo |
| **Deployment** | Temporal Server | Dapr Sidecar | Prefect Server | Kestra Server |
| **Vendor lock-in** | Baixo (open-source) | Mínimo (CNCF) | Baixo | Baixo |
| **Linguagens** | Go, Java, Python, TS, .NET, Ruby | Go, Python, .NET, Java, JS | Python | YAML + scripts |
| **Retry/Compensação** | Nativo e rico | Nativo | Básico | Básico |
| **Observabilidade** | Web UI nativo | OpenTelemetry | Prefect UI | Kestra UI |
| **AI Agents** | Nativo (Schedules, Signals) | Dapr Agents (2025) | Limitado | Limitado |
| **Escala** | Horizontal ilimitada | Horizontal via Actors | Cloud gerenciado | Cloud gerenciado |
| **Melhor para** | Microserviços críticos | Apps cloud-native K8s | Ciência de dados | ETL e automação |

---

## Quando Usar Qual?

### Use Temporal quando:
- Você precisa de **confiabilidade máxima** (99.999% uptime documentado)
- Workflows podem durar **dias, semanas ou meses**
- Há **transações financeiras** ou processos críticos de negócio
- O time usa **múltiplas linguagens** e quer uma API uniforme
- **AI agents** precisam de durabilidade e auditabilidade

### Use Dapr Workflows quando:
- Você já está no **ecossistema Dapr** / Kubernetes
- Precisa de **vendor neutrality** (CNCF governance)
- Quer **trazer seu próprio banco** de dados para state storage
- Precisa combinar **orquestração + coreografia** na mesma plataforma
- Trabalha com **polyglot** e quer sidecar pattern

### Use Prefect quando:
- O time é principalmente **cientistas de dados** em Python
- O foco é **pipelines de ML** e data workflows
- Você quer **managed cloud** com mínima operação
- Workflows são majoritariamente **batch processing**

### Use Kestra quando:
- O time prefere **YAML/declarativo** em vez de código
- O foco é **ETL e data engineering**
- Você precisa de **80+ conectores** out-of-the-box
- Integração com **data warehouses** é prioritária

### Use AWS Step Functions quando:
- Você está **100% na AWS**
- Prefere **serverless managed** sem operar infraestrutura
- Workflows são relativamente simples e de curta duração
- O custo por execução é aceitável na escala pretendida

---

## Camunda (BPMN)

Para casos onde **processos de negócio complexos** precisam ser modelados visualmente e envolvem participantes não-técnicos:

- Notação **BPMN 2.0** — padrão visual de processos
- Integração com **Java ecosystem**
- Suporte a **processos de longa duração** com estado
- **Decision tables (DMN)** para regras de negócio
- **Forms** para human tasks

**Quando usar Camunda:** compliance, processos regulatórios, workflows que incluem aprovações humanas com stakeholders de negócio que precisam entender/modificar o fluxo.

---

## Referências

- [Procycons: Kestra vs Temporal vs Prefect 2025](https://procycons.com/en/blogs/workflow-orchestration-platforms-comparison-2025/)
- [Diagrid: 8 Patterns with Dapr Agents](https://www.diagrid.io/blog/building-effective-dapr-agents)
- [Temporal: How it Works](https://temporal.io/how-it-works)
- [Dapr: Workflow Patterns](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-patterns/)
