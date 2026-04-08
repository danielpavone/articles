# 08 — AI Agents + Orquestração de Workflows

> Baseado em: Temporal Blog, IntuitionLabs, Diagrid Blog, Floki/Dapr Agents

---

## Por que AI Agents Precisam de Orquestração?

Agentes de IA introduzem um conjunto novo de desafios em sistemas distribuídos:

| Desafio | Por que é difícil |
|---------|-------------------|
| **Indeterminismo** | LLMs não são determinísticos — retries devem ser cuidadosos |
| **Latência variável** | Chamadas a LLMs podem demorar segundos ou minutos |
| **Falhas de API** | Rate limits, timeouts, erros de provider |
| **Loops longos** | Agentes que rodam 24×7 precisam de estado persistido |
| **Human-in-the-loop** | Aprovações humanas podem levar horas/dias |
| **Auditabilidade** | Cada decisão do agente deve ser rastreável |
| **Coordenação** | Múltiplos agentes precisam se coordenar sem conflito |

---

## Dois Tipos de Sistemas Agênticos (Anthropic)

> Anthropic faz uma distinção arquitetural importante:

| Tipo | Descrição | Controle |
|------|-----------|----------|
| **Workflows** | LLMs e ferramentas orquestrados por caminhos de código predefinidos | Alto — mais previsível |
| **Agents** | LLMs dirigem dinamicamente seus próprios processos e uso de ferramentas | Baixo — mais autônomo |

**Implicação prática:** Para casos de uso empresariais onde confiabilidade e manutenibilidade são críticos, **workflows** (orquestração predefinida) geralmente superam agentes totalmente autônomos.

---

## Arquitetura de Agentes com Temporal

### Componentes Principais

```
┌─────────────────────────────────────────────────┐
│                  Temporal Cluster                │
│              (estado + coordenação)              │
└──────────┬──────────────────┬───────────────────┘
           │                  │
    ┌──────┴───────┐   ┌──────┴────────┐
    │ Broker Agent │   │  Judge Agent  │
    │ (interface   │   │  (avaliador   │
    │  usuário)    │   │   LLM)        │
    └──────┬───────┘   └───────────────┘
           │
    ┌──────┴───────────────────────┐
    │       Execution Agent         │
    │    (decisões de trading)      │
    └──────┬───────────────────────┘
           │
    ┌──────┴──────────────────────────────────────┐
    │ Supporting Workflows                         │
    │  - Market Data Subscription                  │
    │  - Order Execution                           │
    │  - Ledger                                    │
    └─────────────────────────────────────────────┘
```

### Por que Temporal?

1. **Schedules** — agentes proativos sem cron jobs customizados
   ```python
   # Executa a cada 30 segundos — crypto markets never sleep
   schedule = TemporalSchedule(
       action=StartWorkflow(NudgeAgentWorkflow),
       spec=ScheduleSpec(interval=timedelta(seconds=30))
   )
   ```

2. **Signals** — comunicação assíncrona entre agentes
   ```python
   # Broker agent envia sinal para execution agent
   await execution_handle.signal("new_market_data", market_data)
   ```

3. **Queries** — consultar estado do agente sem interromper execução
   ```python
   portfolio = await agent_handle.query("get_portfolio")
   ```

4. **Histórico auditável** — cada ação do agente registrada no workflow history

---

## Arquitetura de Agentes com Dapr

### Dapr Agents (Março 2025)

O Dapr Agents é um framework para construção de sistemas agênticos baseado em Dapr Workflows:

```
┌─────────────────────────────────────┐
│          Dapr Agents                │
├─────────────────────────────────────┤
│  Workflow Orchestration (Dapr WF)   │
│  Persistent Memory (State Store)    │
│  Service Exposure (REST endpoints)  │
│  Tool Calling (Activities)          │
└─────────────────────────────────────┘
```

### Pattern: Orchestrator Agent

```python
@wf.workflow(name='orchestrator_agent')
def orchestrator_workflow(ctx: DaprWorkflowContext, task: str):
    # LLM decide próximo speaker/agente
    next_agent = yield ctx.call_activity(
        select_next_agent,
        input={'task': task, 'history': ctx.get_state('history')}
    )
    
    # Envia mensagem e processa resposta
    response = yield ctx.call_activity(
        send_to_agent,
        input={'agent': next_agent, 'task': task}
    )
    
    # Decide se continua ou para
    if should_continue(response):
        return (yield ctx.call_child_workflow(
            orchestrator_workflow,
            input=response.next_task
        ))
    return response.result
```

### Pattern: Tool-Calling Agent

```python
@wf.workflow(name='tool_calling_agent')
def tool_agent_workflow(ctx: DaprWorkflowContext, request: dict):
    # LLM decide se precisa de tool
    plan = yield ctx.call_activity(llm_plan_tools, input=request)
    
    if plan.needs_tools:
        # Executa tools em paralelo
        tool_tasks = [
            ctx.call_activity(execute_tool, input={'tool': t, 'args': a})
            for t, a in plan.tools
        ]
        tool_results = yield wf.when_all(tool_tasks)
        
        # LLM sintetiza resultados
        return (yield ctx.call_activity(
            llm_synthesize,
            input={'request': request, 'results': tool_results}
        ))
    else:
        return (yield ctx.call_activity(llm_respond_direct, input=request))
```

---

## Padrões para AI Workflows (Anthropic Cookbook)

### 1. Augmented LLM (Base)

```
[Input] → [LLM + Tools + Memory + Context] → [Output]
```

Adiciona ferramentas, memória persistente e retrieval ao LLM base.

### 2. Prompt Chaining

```
[Input] → [LLM 1] → [Gate/Check] → [LLM 2] → [Output]
```

Decompõe tarefa complexa. O Gate pode verificar qualidade antes de continuar.

### 3. Routing

```
[Input] → [LLM Router] → [Especialista A ou B ou C]
```

Classifica input e direciona para o agente/pipeline mais adequado.

### 4. Parallelization

```
         ┌──► [LLM A] ──┐
[Input] ─┼──► [LLM B] ──┼──► [Aggregator]
         └──► [LLM C] ──┘
```

Múltiplas perspectivas ou verificações simultâneas.

### 5. Orchestrator-Workers

```
[Input] → [Orchestrator LLM] → plano dinâmico → Workers (parallel)
                                              ← resultados
                               ↓
                         [Synthesis LLM]
```

Mais flexível — o orchestrator define o plano dinamicamente.

### 6. Evaluator-Optimizer Loop

```
[Input] → [Generator LLM] → [output]
                                ↓
                         [Evaluator LLM] → aceito? → Done
                                ↓ não
                         [feedback] → [Generator LLM] (loop)
```

Útil quando critérios de qualidade são mensuráveis.

---

## Boas Práticas para AI Workflows

### 1. Separe Lógica de Orquestração de Chamadas ao LLM

```python
# Bom: LLM call encapsulada em Activity
@activity.defn
async def call_llm(prompt: str) -> str:
    return await openai_client.chat(prompt)  # retry automático se falhar

@workflow.defn
class MyAgentWorkflow:
    @workflow.run
    async def run(self, input: str):
        # Workflow é determinístico
        step1 = await workflow.execute_activity(call_llm, input)
        step2 = await workflow.execute_activity(call_llm, step1)
        return step2
```

### 2. Idempotência para LLM Calls

LLMs podem ser chamados novamente em retries. Use IDs de idempotência:

```python
@activity.defn
async def call_llm_idempotent(request_id: str, prompt: str) -> str:
    # Verifica cache antes de chamar
    cached = await cache.get(request_id)
    if cached:
        return cached
    result = await llm.generate(prompt)
    await cache.set(request_id, result, ttl=3600)
    return result
```

### 3. Human-in-the-Loop para Decisões Críticas

```python
@workflow.defn
class CriticalDecisionWorkflow:
    @workflow.run
    async def run(self, context: dict):
        proposal = await workflow.execute_activity(llm_propose, context)
        
        # Envia para revisão humana
        await workflow.execute_activity(notify_human_reviewer, proposal)
        
        # Aguarda aprovação (pode demorar horas)
        approval = await workflow.wait_condition(
            lambda: self._approval is not None,
            timeout=timedelta(days=1)
        )
        
        if approval.approved:
            return await workflow.execute_activity(execute_proposal, proposal)
        else:
            return await workflow.execute_activity(log_rejection, proposal)
    
    @workflow.signal
    async def submit_approval(self, data: ApprovalData):
        self._approval = data
```

### 4. Observabilidade de Agentes

Todo motor de orquestração maduro oferece:
- **Histórico de execução** por workflow ID
- **Status em tempo real** (running, paused, failed, completed)
- **Inputs/outputs** de cada activity
- **Retry attempts** com timestamps
- **Duração** de cada step

---

## Estado da Arte em 2025/2026

| Tendência | Descrição |
|-----------|-----------|
| **Durable Agents** | Agentes que sobrevivem a restarts e falhas — Temporal, Dapr |
| **MCP + Orquestração** | Temporal + Model Context Protocol para tool interfaces padronizadas |
| **Multi-agent workflows** | Múltiplos agentes especializados em um único workflow durável |
| **LLM-as-Judge** | Agente avaliador que monitora e ajusta outros agentes em tempo real |
| **Ambient Agents** | Agentes proativos 24×7, acionados por schedules e eventos, sem prompt manual |

> Gartner prevê que 40% das aplicações enterprise terão agentes AI especializados até o fim de 2026 (vs. <5% em 2025). Porém, 40% dos projetos agênticos serão cancelados até 2027 por custos e valor desalinhado.

---

## Referências

- [Temporal: Orchestrating Ambient Agents](https://temporal.io/blog/orchestrating-ambient-agents-with-temporal)
- [IntuitionLabs: Agentic AI + Temporal](https://intuitionlabs.ai/articles/agentic-ai-temporal-orchestration)
- [Diagrid: 8 Patterns with Dapr Agents](https://www.diagrid.io/blog/building-effective-dapr-agents)
- [Floki: Building AI Agentic Workflow Engine with Dapr](https://blog.openthreatresearch.com/floki-building-an-ai-agentic-workflow-engine-dapr/)
- [Dapr.io: Workflow Overview](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-overview/)
