# Parallel Task Execution and Multi-Agent Architectures for AI Systems

**Research Date:** 2026-01-30

A comprehensive research document covering architecture patterns, code examples, and performance considerations for messaging bot gateway systems.

---

## 1. Multi-Agent Orchestration Patterns

### Supervisor-Worker Architecture

The supervisor pattern employs a hierarchical architecture where a central orchestrator coordinates all multi-agent interactions. According to Databricks, **72% of enterprise AI projects now involve multi-agent architectures** (up from 23% in 2024).

**Control Flow:**
```
User Input → Supervisor Agent
    ├── Decompose into subtasks
    ├── Delegate to specialized workers (parallel)
    ├── Monitor progress
    ├── Validate outputs
    └── Synthesize final response
```

### Agent-as-Tool Pattern

Per Anthropic's multi-agent research system, the agent-as-tool model keeps the main agent in charge while using specialist agents as tools:

```python
class OrchestratorAgent:
    def process(self, task):
        subtasks = self.decompose(task)

        # Invoke sub-agents in parallel as tools
        results = await asyncio.gather(*[
            self.invoke_specialist(subtask)
            for subtask in subtasks
        ])

        return self.synthesize(results)
```

---

## 2. Async Python Patterns for AI Agents

### Using `asyncio.gather()` for Concurrent Execution

```python
import asyncio
from typing import List

async def process_agent_tasks(prompts: List[str]) -> List[str]:
    """Run multiple agent conversations concurrently."""
    tasks = [agent.run(prompt) for prompt in prompts]
    results = await asyncio.gather(*tasks)
    return results

asyncio.run(process_agent_tasks(["task1", "task2", "task3"]))
```

### Rate Limiting with Semaphores

```python
class RateLimitedAgent:
    def __init__(self):
        self.openai_semaphore = asyncio.Semaphore(45)   # Under 50 limit
        self.db_semaphore = asyncio.Semaphore(150)      # PostgreSQL
        self.cache_semaphore = asyncio.Semaphore(500)   # Redis

    async def call_openai(self, prompt: str) -> str:
        async with self.openai_semaphore:
            return await self.client.chat.completions.create(...)

    async def parallel_inference(self, prompts: List[str]) -> List[str]:
        return await asyncio.gather(*[
            self.call_openai(p) for p in prompts
        ])
```

### Coordination Patterns Comparison

| Pattern | Use Case | Behavior |
|---------|----------|----------|
| `asyncio.gather()` | All results needed | Waits for all, returns in order |
| `asyncio.wait()` | Fine-grained control | first_completed, all_completed options |
| `asyncio.as_completed()` | Process as ready | Returns tasks as they complete |
| `asyncio.TaskGroup` | Safety-critical | Cancels remaining if one fails |

---

## 3. TypeScript/Node.js Parallel Processing

### Concurrency-Limited Execution

From Moltbot's `src/media-understanding/concurrency.ts`:

```typescript
export async function runWithConcurrency<T>(
  tasks: Array<() => Promise<T>>,
  limit: number,
): Promise<T[]> {
  if (tasks.length === 0) return [];
  const resolvedLimit = Math.max(1, Math.min(limit, tasks.length));
  const results: T[] = Array.from({ length: tasks.length });
  let next = 0;

  const workers = Array.from({ length: resolvedLimit }, async () => {
    while (true) {
      const index = next;
      next += 1;
      if (index >= tasks.length) return;
      results[index] = await tasks[index]();
    }
  });

  await Promise.allSettled(workers);
  return results;
}
```

### Lane-Based Command Queue

From Moltbot's `src/process/command-queue.ts`:

```typescript
export const enum CommandLane {
  Main = "main",
  Cron = "cron",
  Subagent = "subagent",
  Nested = "nested",
}

// Configurable concurrency per lane
export function setCommandLaneConcurrency(lane: string, maxConcurrent: number) {
  const state = getLaneState(lane);
  state.maxConcurrent = Math.max(1, Math.floor(maxConcurrent));
  drainLane(lane);
}
```

This pattern enables:
- Serialized execution within lanes (prevents interleaving)
- Parallel execution across lanes (cron vs main vs subagent)
- Configurable concurrency limits per lane

### Worker Threads for CPU-Intensive Tasks

```typescript
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

if (isMainThread) {
  const worker = new Worker('./worker.ts', {
    workerData: { task: 'heavy-computation' }
  });

  worker.on('message', (result) => {
    console.log('Worker result:', result);
  });
} else {
  const result = performHeavyComputation(workerData.task);
  parentPort?.postMessage(result);
}
```

---

## 4. Claude Code CLI Sub-Agent Patterns

### Built-in Subagents

| Subagent | Purpose | Tools |
|----------|---------|-------|
| **Explore** | Codebase search/analysis (read-only) | Read, Grep, Glob |
| **Plan** | Research during planning mode | Read, Grep, Glob |
| **General-purpose** | Complex multi-step tasks | Read, Write, Edit, Bash, Glob, Grep |

### Creating Custom Subagents

Place subagent definitions in `.claude/agents/`:

```markdown
---
name: code-reviewer
description: Reviews code for security and best practices
tools:
  - Read
  - Grep
  - Glob
---

You are a code reviewer. Analyze the provided code for:
- Security vulnerabilities
- Performance issues
- Best practice violations

Return a structured review with severity levels.
```

### Tool Configuration by Role

| Role | Recommended Tools |
|------|-------------------|
| Read-only (reviewers) | Read, Grep, Glob |
| Research (analysts) | Read, Grep, Glob, WebFetch, WebSearch |
| Code writers | Read, Write, Edit, Bash, Glob, Grep |
| Documentation | Read, Write, Edit, Glob, Grep, WebFetch |

### Session Isolation Pattern

From Moltbot's `sessions-spawn-tool.ts`:

```typescript
// Validate requester is not a subagent (prevent nesting)
if (isSubagentSessionKey(requesterSessionKey)) {
  return jsonResult({
    status: "forbidden",
    error: "sessions_spawn is not allowed from sub-agent sessions",
  });
}

// Create isolated session for child
const childSessionKey = `agent:${targetAgentId}:subagent:${crypto.randomUUID()}`;

// Call gateway to spawn the agent
await callGateway({
  method: "agent",
  params: {
    message: task,
    sessionKey: childSessionKey,
    lane: AGENT_LANE_SUBAGENT,  // Isolated lane
  },
});
```

---

## 5. CPU Pinning and Resource Allocation

### Using taskset for CPU Affinity

```bash
# Run process on specific CPUs (0-3)
taskset -c 0-3 python agent.py

# Set affinity for running process
taskset -p -c 4-7 <pid>

# Multiple isolated instances
taskset -c 0-7 python instance1.py &
taskset -c 8-15 python instance2.py &
```

### Performance Benefits

- HPC benchmarks: **45% reduction** in completion time
- Database workloads: **20-30% improvement**
- HFT systems: **22% latency reduction**

### Cgroups for Resource Management

```bash
# Create cgroup for AI agents
sudo cgcreate -g cpu,memory:ai_agents

# Set CPU quota (50% of one CPU)
sudo cgset -r cpu.cfs_quota_us=50000 ai_agents

# Set memory limit (4GB)
sudo cgset -r memory.limit_in_bytes=4G ai_agents

# Run process in cgroup
sudo cgexec -g cpu,memory:ai_agents python agent.py
```

### Taskset vs Cgroups

| Feature | taskset | cgroups |
|---------|---------|---------|
| CPU affinity | Yes | Yes |
| Memory limits | No | Yes |
| I/O throttling | No | Yes |
| Complexity | Low | High |

---

## 6. Non-Blocking Task Patterns

### Cron vs Heartbeat

**Cron Jobs:**
- Isolated, scheduled executions
- Each job runs independently
- No shared context between runs

**Heartbeats:**
- Regular interval checks (default: 30 min)
- Share main session context
- Can batch multiple checks into single run
- Smart suppression (HEARTBEAT_OK for no-action)

### Celery + Redis for Background Tasks

```python
from celery import Celery

app = Celery('ai_tasks', broker='redis://localhost:6379/0')

@app.task
def process_llm_request(prompt: str, model: str = "gpt-4"):
    response = llm_client.chat(prompt, model=model)
    return response

# API endpoint
@app.post("/agent/process")
async def start_agent(request: AgentRequest):
    task = process_llm_request.delay(request.prompt)
    return {"task_id": task.id}
```

### Priority Queues

```python
app.conf.task_routes = {
    'tasks.urgent_*': {'queue': 'high_priority'},
    'tasks.batch_*': {'queue': 'low_priority'},
    'tasks.*': {'queue': 'default'},
}
```

---

## 7. Framework Comparison

| Framework | Strengths | Best For |
|-----------|-----------|----------|
| **LangGraph** | State management, checkpointing | Complex workflows with persistence |
| **OpenAI Agents SDK** | Handoffs, guardrails, tracing | Production agents with tool use |
| **Google ADK** | ParallelAgent, workflow agents | GCP integration, parallel tasks |
| **Temporal** | Distributed workflows, durability | Long-running, mission-critical |
| **Celery + Redis** | Task queues, retries, scheduling | Background job processing |

---

## 8. Recommendations for Moltbot Gateway

### Recommended Concurrency Limits

```typescript
const LANE_CONCURRENCY = {
  main: 1,           // Serialize main interactions
  subagent: 3,       // Allow parallel subagents
  cron: 2,           // Parallel cron jobs
  nested: 1,         // Serialize nested calls
};
```

### Resource Allocation Strategy

**Development/Small Scale:**
- Default Node.js event loop
- No explicit CPU pinning
- Semaphore-based rate limiting for APIs

**Production/Multi-Instance:**
- Use cgroups for memory limits
- Consider taskset for CPU isolation
- Separate Redis for task queues
- Worker pools for CPU-intensive tasks

**High-Throughput:**
- Kubernetes CPU manager for pod isolation
- Horizontal scaling with multiple gateway instances
- Dedicated inference servers with NUMA awareness

### Key Implementation Patterns

```typescript
// Pattern 1: Parallel with concurrency limit
const results = await runWithConcurrency(
  tasks.map(t => () => processTask(t)),
  MAX_CONCURRENT
);

// Pattern 2: Lane-isolated execution
await enqueueCommandInLane(CommandLane.Subagent, async () => {
  return await agent.run(task);
});

// Pattern 3: Parallel information gathering
const [status, config, health] = await Promise.all([
  getServiceStatus(),
  loadConfiguration(),
  checkHealth(),
]);
```

---

## Sources

- [Databricks: Multi-Agent Supervisor Architecture](https://www.databricks.com/blog/multi-agent-supervisor-architecture-orchestrating-enterprise-ai-scale)
- [Anthropic: Multi-Agent Research System](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Google ADK: Parallel Agents](https://google.github.io/adk-docs/agents/workflow-agents/parallel-agents/)
- [OpenAI Agents SDK: Multi-Agent](https://openai.github.io/openai-agents-python/multi_agent/)
- [Claude Code Docs: Custom Subagents](https://code.claude.com/docs/en/sub-agents)
- [VoltAgent: Awesome Claude Code Subagents](https://github.com/VoltAgent/awesome-claude-code-subagents)
- [Linux Man Pages: taskset](https://man7.org/linux/man-pages/man1/taskset.1.html)
- [Moltbot Docs: Cron vs Heartbeat](https://docs.molt.bot/automation/cron-vs-heartbeat)
- [JellyfishTechnologies: LLM-Driven API Orchestration](https://www.jellyfishtechnologies.com/llm-driven-api-orchestration-using-langchain-celery-redis-queue/)
