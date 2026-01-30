# Model Optimization Research Findings
**Date:** 2026-01-30
**Purpose:** Research for optimizing Moltbot's LLM model selection for latency and cost

---

## Table of Contents

1. [Constraints Summary](#constraints-summary)
2. [Cerebras Inference Benchmarks](#1-cerebras-inference-benchmarks)
3. [Local LLM (Mac Studio M2 Ultra 128GB)](#2-local-llm-mac-studio-m2-ultra-128gb)
4. [Gemini (Paid Tier)](#3-gemini-paid-tier)
5. [Model Routing Strategies](#4-model-routing-strategies)
6. [Prompt Optimization Techniques](#5-prompt-optimization-techniques)
7. [Recommended Model Stack for Moltbot (Pro Subscription)](#6-recommended-model-stack-for-moltbot-pro-subscription)
8. [Channel-Specific Model Selection](#7-channel-specific-model-selection)
9. [No-Code Quick Wins (Do These First)](#8-no-code-quick-wins-do-these-first)
10. [Implementation Roadmap (Requires Code Changes)](#9-implementation-roadmap-requires-code-changes)
11. [Moltbot Integration Points](#10-moltbot-integration-points)
12. [Sources](#sources)

---

## Constraints Summary
- **Budget**: <$10/month for paid models
- **Subscriptions**:
  - **Cerebras Pro** ✅ (full 128K context on all models)
  - **Gemini Paid** ✅ (higher rate limits than free tier)
  - ChatGPT Pro (GPT-5.2/Codex via subscription)
- **Cerebras Pro models**: Llama 3.1 8B, Llama 3.3 70B, OpenAI GPT OSS, Qwen 3 32B, Qwen 3 235B, Z.ai GLM 4.7
- **Local hardware**: Mac Studio Ultra M2, 128GB RAM (~70GB available for models)
- **Primary bottleneck**: LLM API latency (85-95% of response time)


---

## 1. Cerebras Inference Benchmarks

### Speed Benchmarks (tokens/second)

| Model | Speed | Context (Free) | Context (Paid) |
|-------|-------|----------------|----------------|
| Llama 3.1 8B | ~2,200 t/s | - | - |
| Llama 3.3 70B | ~2,100-2,500 t/s | - | - |
| Qwen 3 32B | ~2,600 t/s | 65K | 131K |
| Qwen 3 235B | ~1,400 t/s | - | - |
| Z.ai GLM 4.7 | ~1,000 t/s | - | - |
| GPT OSS 120B | ~3,000 t/s | - | - |

### Time to First Token (TTFT)
| Model | TTFT | Full Request |
|-------|------|--------------|
| Llama 3.3 70B | **170 ms** | 0.4s |
| Llama 3.1 405B | 240 ms | - |
| GPT OSS 120B | 280 ms | - |
| Llama 3.1 8B | 440 ms | - |

### Rate Limits
**Free Tier:**
- 30 requests/minute
- 60K input tokens/minute
- 1M daily tokens
- 8K max output tokens

**Pro Tier** ✅ (Your subscription):
- Higher rate limits
- 128K context on all models
- Priority queue access

### Context Length (Pro Subscription)
| Model | Native Context | **Cerebras Pro** ✅ |
|-------|----------------|---------------------|
| Qwen 3 32B | 32K (131K via YaRN) | **131K** |
| Qwen 3 235B (A22B MoE) | 262K (1M via YaRN) | **131K** |
| Llama 3.3 70B | 128K | **128K** |
| Llama 3.1 8B | 128K | **128K** |
| GPT OSS 120B | 128K | **128K** |

**Note:** Native context = official Qwen3 model spec. Cerebras Pro may limit context below native capacity. YaRN = extended context via RoPE interpolation (available in some deployments).

**✅ Pro subscription unlocks full 128K+ context on ALL models** - no context limitations for Moltbot agents.

> **Source:** Native specs from [Qwen3 Blog](https://qwenlm.github.io/blog/qwen3/) and [Qwen3 GitHub](https://github.com/QwenLM/Qwen3). Cerebras limits verified via user's Pro subscription testing (Jan 2026). See also [Cerebras Pricing](https://cerebras.ai/pricing).

### Why Cerebras Is So Fast (MoE Architecture)

From [Cerebras MoE Guide](https://www.cerebras.ai/blog/moe-guide-calculator):

**The Problem with Standard GPUs (H100):**
- 99.5% of inference time spent on **weight loading**, not computation
- Memory-bound: 2.8 TB/s bandwidth limits throughput
- Single H100: ~333 t/s at batch=1, ~6,000 t/s at batch=256
- Multi-GPU: All-to-all communication becomes the bottleneck

**Cerebras WSE Solution:**
- Entire model stored in **on-chip SRAM** (no memory loading)
- Compute-bound immediately at batch size 1
- No all-to-all communication overhead
- Result: 2,000-3,000 t/s sustained throughput

**MoE Notation Clarification:**
- "8x7B" ≠ 7B active parameters
- Actually ~13B active params per token (router + shared components)
- Qwen3-30B MoE: ~6GB active weights loaded per token on GPU

### CePO: Test-Time Compute Reasoning

From [Cerebras CePO Docs](https://inference-docs.cerebras.ai/capabilities/cepo):

**What it is:** Framework adding advanced reasoning to Llama models via test-time compute (like o1/o3).

**How it works (4 stages):**
1. **Planning** - Create step-by-step solution approach
2. **Execution** - Generate multiple response variations
3. **Analysis** - Identify inconsistencies across executions
4. **Best-of-N** - Score responses with structured confidence

**Performance:**
- Model: `llama3.3-70b`
- Speed: **2,200 reasoning tokens/second**

**Usage:**
```bash
pip install cerebras_cloud_sdk optillm
export CEREBRAS_API_KEY='...'
optillm --base-url https://api.cerebras.ai --approach cepo
```

**Use case for Moltbot:** Complex multi-step reasoning, code generation, analysis tasks where accuracy matters more than raw speed.

### Full Capabilities Reference

#### Batch Processing (50% Off)
- **Pricing**: Half cost of real-time API
- **Completion**: Guaranteed within 24 hours
- **Limits**: 50K requests/batch, 200MB max file, 10 concurrent batches
- **Use for**: Evaluations, bulk labeling, content generation at scale

```python
batch = client.batches.create(
    input_file_id=file.id,
    endpoint="/v1/chat/completions",
    completion_window="24h"
)
```

#### Predicted Outputs (Latency Reduction)
For code edits, refactoring, template filling - reuse known tokens:
- **Models**: `gpt-oss-120b`, `llama3.1-8b`, `zai-glm-4.7`
- **Tip**: Set `temperature=0` for higher acceptance
- ⚠️ Rejected tokens still billed at output rate

```python
response = client.chat.completions.create(
    model="gpt-oss-120b",
    messages=[...],
    prediction={"type": "content", "content": existing_code}
)
```

#### Prompt Caching (Automatic, Free)
- **How**: Automatic prefix matching in 100-600 token blocks
- **TTL**: Guaranteed 5 minutes, may persist up to 1 hour
- **Cost**: No extra charge - cached tokens billed at input rate
- **Tip**: Place static content FIRST (system prompt, tools, examples)

Check cache hits via `response.usage.prompt_tokens_details.cached_tokens`

#### Reasoning Models
| Model | Control | Options |
|-------|---------|---------|
| GPT-OSS | `reasoning_effort` | low/medium/high |
| GLM 4.7 | `disable_reasoning` | true/false |
| Qwen3 | `reasoning_format` | parsed/raw/hidden/none |

```python
response = client.chat.completions.create(
    model="zai-glm-4.7",
    reasoning_format="parsed",  # Separate reasoning field
    messages=[...]
)
```

#### Structured Outputs (JSON Schema)
- **Strict mode**: Constrained decoding guarantees valid JSON
- **Limits**: 5K char schema, 10 levels nesting, 500 properties
- ⚠️ No regex patterns, no recursive schemas

```python
response_format={
    "type": "json_schema",
    "json_schema": {"name": "...", "strict": True, "schema": {...}}
}
```

#### Tool Calling
- **Parallel**: `parallel_tool_calls=true` (default)
- **Control**: `tool_choice` = none/auto/required/specific
- Supported on all models

#### Service Tiers (No Price Difference Currently)
| Tier | Priority | Use Case |
|------|----------|----------|
| `priority` | Highest | User-facing, time-critical |
| `default` | Standard | Production workloads |
| `auto` | Dynamic | Flexible priority |
| `flex` | Lowest | Batch-like, higher limits |

#### Metrics (Prometheus)
Available metrics: TTFT, TPOT, e2e latency (p50/p90/p95/p99), token counts, cache rates, error rates.

### Cookbook Examples (Agent Patterns)
- **Voice agents**: LiveKit integration, real-time translation
- **Research assistants**: Academic paper analysis, web search
- **Fact checkers**: Web search + source verification
- **Multi-agent**: LangChain workflows, PydanticAI integration

### Limitations & Gotchas
- **Near 100% utilization**: Requests may be queued during US working hours
- **Context length issues**: Reasoning models can run out of context; use `/no_think` suffix to disable
- **API quirks**: Some OpenAI compatibility issues; streaming may be fragmented

---

## 2. Local LLM (Mac Studio M2 Ultra 128GB)

### Framework Performance Rankings
| Framework | Throughput | Notes |
|-----------|------------|-------|
| MLX | ~230 tok/s | Best sustained throughput |
| MLC-LLM | ~190 tok/s | Best TTFT for moderate prompts |
| llama.cpp | ~150 tok/s | Short-context only |
| Ollama | 20-40 tok/s | User-friendly |
| PyTorch MPS | ~7-9 tok/s | Slowest |

### LLaMA 3 Benchmarks on M2 Ultra

| Model | Quant | Generation | Prompt Proc |
|-------|-------|------------|-------------|
| 8B | Q4_K_M | 76.28 t/s | 1,023 t/s |
| 8B | F16 | 36.25 t/s | 1,202 t/s |
| 70B | Q4_K_M | **12.13 t/s** | 117.76 t/s |
| 70B | F16 | 4.71 t/s | 145.82 t/s |

### Memory Requirements
- **8B Q4_K_M**: ~4.8-5.2 GB
- **8B Q5_K_M**: ~5.7-6.2 GB
- **70B Q4_K_M**: ~35-40 GB (fits in 70GB budget)
- **70B Q5_K_M**: ~45-50 GB (fits in 70GB budget)

### Recommendations
- **Use MLX or vllm-mlx** for highest throughput
- **Use MLC-LLM** for lowest TTFT
- **Q4_K_M or Q5_K_M** quantization for best quality/size tradeoff
- **70B models workable** at 12-20 t/s for interactive use

---

## 3. Gemini (Paid Tier)

> **Note:** This section documents the **paid tier** (Pay-as-you-go / Vertex AI). Free tier has significantly lower limits and EU/UK/Switzerland restrictions.
>
> **Last verified:** 2026-01-30. See [official rate limits](https://ai.google.dev/gemini-api/docs/rate-limits) for current values.

### Paid Tier Rate Limits
| Model | RPM | TPM | TPD |
|-------|-----|-----|-----|
| Gemini 2.5 Pro | 1,000 | 4M | 100M |
| Gemini 2.5 Flash | 2,000 | 4M | 100M |
| Gemini 2.0 Flash | 2,000 | 4M | 100M |
| Gemini 1.5 Pro | 1,000 | 4M | Unlimited |
| Gemini 1.5 Flash | 2,000 | 4M | Unlimited |

### Speed Comparison
| Model | TTFT | Tokens/sec |
|-------|------|------------|
| Gemini 2.0 Flash | 0.25-0.34s | 250 |
| Gemini 2.5 Flash (Reasoning) | ~0.4s | 372 |
| GPT-4o | ~0.5s | 116-131 |
| Claude 3 Sonnet | ~0.6-0.7s | 77-82 |

### Context Window
- **1 million tokens** (2M for some models via Vertex AI)

### Paid vs Free Tier Summary
| Aspect | Free Tier | Paid Tier |
|--------|-----------|-----------|
| RPM | 5-15 | 1,000-2,000 |
| TPM | 250K | 4M |
| EU/UK/CH | Not available | Available |
| Rate limit increases | No | Request via quota page |

---

## 4. Model Routing Strategies

### Open Source Implementations

**RouteLLM** (LMSYS)
- Drop-in OpenAI client replacement
- 5 pre-trained routers: MF (recommended), SW_ranking, BERT, Causal_llm, Random
- Trained on Chatbot Arena dataset + GPT-4 judgments
- **Results**: Up to 70% cost reduction (MT Bench), 30% (MMLU), 40% (GSM8K)

**Semantic Router** (Aurelio Labs)
- Vector-based routing using embeddings
- Ultra-low latency (no LLM call for routing)
- Supports Cohere, OpenAI, HuggingFace encoders
- Routes based on semantic similarity to example utterances

**vLLM Semantic Router**
- ModernBERT classifier for query complexity
- Fast path vs Chain-of-Thought routing
- **Results**: 10% accuracy improvement, 50% latency reduction, 50% fewer tokens

### Routing Patterns
1. **Simple routing**: Select one model per query
2. **Cascading**: Try small model first, escalate if confidence low
3. **AutoMix**: Self-verification before escalation
4. **Cascade routing**: Combines both (theoretically optimal)

### Implementation for Moltbot (Pro Tier)
```text
Default       → Llama 3.3 70B (Cerebras Pro, 170ms TTFT, 128K context)
Ultra-fast    → GPT OSS 120B (Cerebras Pro, 3000 t/s, short responses)
Deep reasoning → Qwen 3 235B (Cerebras Pro, 1400 t/s, 131K context)
Long documents → Gemini 2.5 Flash (1M context) or GPT-5.2
```

**Pro tier advantage**: With 128K context on all Cerebras models, routing can optimize for TTFT/speed rather than context constraints.

### Cascade Routing (ICLR 2025)

The state-of-the-art approach combining routing and cascading:

**Cascade Routing Framework** ([github.com/eth-sri/cascade-routing](https://github.com/eth-sri/cascade-routing))
```python
from selection import CascadeRouter

router = CascadeRouter(
    quality_computer,      # Post-hoc quality estimator
    cost_computer,
    models=['Llama-8B', 'Llama-70B', 'GPT-4o'],
    max_expected_cost=0.003,
    max_depth=3,           # Max models to try
    force_order=True       # Enforce cost-ascending order
)
router.fit(train_queries, train_answers)
```

**Results** (ICLR 2025):
- 80% relative improvement over baselines on RouterBench
- 14% improvement on SWE-Bench
- Up to 3.66x cost savings with routing
- Up to 5x+ savings with cascading + reliable judge (>80% accuracy)

### Router Latency Overhead

Production measurements show router overhead is negligible:
- **Embedding-based routers**: <10ms per query
- **BERT classifiers**: 20-50ms per query
- **LLM-based classifiers**: 100-500ms per query
- **Model switching** (optimized): Sub-millisecond

Compared to typical LLM generation (5-30 seconds), router overhead is practical for synchronous processing.

### Additional Resources
- [RouteLLM Blog](https://lmsys.org/blog/2024-07-01-routellm/)
- [Cascade Routing Paper](https://arxiv.org/abs/2410.10347)
- [RouterArena Evaluation](https://huggingface.co/blog/JerryPotter/who-routes-the-routers)
- [vLLM Semantic Router](https://blog.vllm.ai/2026/01/05/vllm-sr-iris.html)

---

## 5. Prompt Optimization Techniques

### Prompt Compression (LLMLingua)
- Up to **20x compression** with ~1.5% quality loss
- Budget allocation: Instructions 10-20%, Examples 60-80%, Questions 0-10%

### Caching Strategies

**Prefix Caching (Provider)**
| Provider | Latency Reduction | Cost Reduction |
|----------|-------------------|----------------|
| Anthropic | 85% | 90% |
| OpenAI | Automatic | 50% |

**Semantic Caching (Response)**
- 31% of queries have semantic similarity to previous
- Cache hit = 100% latency savings
- Tools: GPTCache, Redis LangCache, PromptCache

### KV Cache Optimization
| Technique | Improvement |
|-----------|-------------|
| PagedAttention (vLLM) | 2-4x throughput |
| NVFP4 KV Cache | 3x lower latency |
| LMCache + vLLM | 3-10x latency reduction |

### Speculative Decoding
- 2-4x speedup at acceptance rates ≥0.6
- EAGLE methods: ~80% acceptance rate
- Frameworks: vLLM, TensorRT-LLM, SGLang

---

## 6. Recommended Model Stack for Moltbot (Pro Subscription)

### Tier 1: Primary - Best All-Around ⭐
**Cerebras Llama 3.3 70B** (Recommended Default)
- **170ms TTFT** (fastest time to first token)
- 2,100-2,500 t/s generation
- **128K context** (Pro tier)
- Use for: All standard tasks, tool use, coding, reasoning

### Tier 2: Ultra-Fast (Simple Tasks)
**Cerebras GPT OSS 120B** or **Llama 3.1 8B**
- GPT OSS: 3,000 t/s (fastest raw speed)
- Llama 8B: 2,200 t/s
- **128K context** (Pro tier)
- Use for: Simple Q&A, quick responses, basic commands

### Tier 3: Maximum Capability
**Cerebras Qwen 3 235B** or **Qwen 3 32B**
- Qwen 235B: 1,400 t/s, **131K context**
- Qwen 32B: 2,600 t/s, **131K context**
- Use for: Complex reasoning, agentic tasks requiring deep thought

**CePO (Test-Time Reasoning)** - via optillm
- Llama 3.3 70B with multi-stage reasoning
- 2,200 reasoning t/s
- Use for: Math, code generation, multi-step analysis where accuracy > speed

### Tier 4: Specialized/Fallback
**Gemini 2.5 Flash** - 1M context for very long documents
**GPT-5.2 (ChatGPT Pro)** - Complex analysis
**Local Llama 3.3 70B Q4_K_M (MLX)** - Offline/privacy-sensitive (~12-20 t/s)

### Implementation Priority (Updated for Pro)
1. **Set Cerebras Llama 3.3 70B as default** - fastest TTFT + 128K context
2. **Implement simple router** for query complexity → model selection
3. **Enable prefix caching** for system prompts (Anthropic/OpenAI)
4. **Add semantic cache** for repeated queries

---

## 7. Channel-Specific Model Selection

Moltbot serves multiple messaging channels with different latency expectations and capabilities. This section maps channels to optimal model choices based on user experience requirements.

### Channel Latency Classification

| Channel Type | User Expectation | TTFT Target | Examples |
|--------------|------------------|-------------|----------|
| **Synchronous** | Immediate response, feels like typing | <500ms | Discord commands, iMessage |
| **Semi-sync** | Quick but brief wait acceptable | <1s | Telegram, Signal, WhatsApp |
| **Asynchronous** | Threaded/conversational, wait OK | <2-3s | Slack threads, MS Teams, Matrix |
| **Batch/Passive** | No real-time expectation | <10s | Email integrations, scheduled tasks |

### Channel-Specific Requirements

| Channel | TTFT Target | Streaming | Recommended Model | Notes |
|---------|-------------|-----------|-------------------|-------|
| **Discord (commands)** | <500ms | Yes | Llama 3.3 70B | Slash commands expect instant feedback |
| **Discord (DMs)** | <1s | Yes | Llama 3.3 70B | Typing indicator helps |
| **iMessage** | <500ms | No | Llama 3.3 70B | No streaming; fast TTFT critical |
| **Telegram** | <1s | Yes | Llama 3.3 70B | Typing action + edit for streaming |
| **Signal** | <1s | No | Llama 3.3 70B | No streaming; moderate TTFT OK |
| **WhatsApp** | <1s | No | Llama 3.3 70B | No streaming; typing indicator helps |
| **Slack (threads)** | <2s | Yes | Llama 3.3 70B / Qwen 3 235B | Threads tolerate longer waits; use 235B for complex |
| **Slack (DMs)** | <1s | Yes | Llama 3.3 70B | Direct messages need faster response |
| **MS Teams** | <2s | Yes | Llama 3.3 70B / Qwen 3 235B | Enterprise context often complex |
| **Matrix** | <2s | Yes | Llama 3.3 70B | Federated; latency varies by server |
| **Google Chat** | <2s | Yes | Llama 3.3 70B | Workspace integration |
| **Zalo** | <1s | No | Llama 3.3 70B | Popular in Vietnam; mobile-first |
| **Twitch** | <500ms | No | GPT OSS 120B / Llama 8B | Chat speed critical; simple responses |
| **Voice Call** | <300ms | Yes (audio) | Llama 3.3 70B | Real-time audio; lowest latency required |

### Streaming Capability by Channel

**Full Streaming Support:**
- Discord (edit messages progressively)
- Telegram (edit messages progressively)
- Slack (update message blocks)
- MS Teams (adaptive cards update)
- Matrix (message replacement)

**No Streaming (Single Response):**
- iMessage (Apple limitation)
- Signal (protocol limitation)
- WhatsApp (API limitation)
- Zalo (API limitation)
- Twitch (chat format)

### Model Selection Strategy by Channel Type

#### Sync Channels (Discord commands, iMessage, Voice)
```text
Priority: TTFT > Throughput > Capability
Default:  Llama 3.3 70B (170ms TTFT)
Fallback: GPT OSS 120B (280ms TTFT, 3000 t/s)
Complex:  Qwen 3 32B (fast + capable)
```

#### Semi-Sync Channels (Telegram, Signal, WhatsApp)
```text
Priority: Balance TTFT and capability
Default:  Llama 3.3 70B (170ms TTFT, 128K context)
Complex:  Qwen 3 235B (deeper reasoning acceptable)
```

#### Async Channels (Slack threads, MS Teams, Matrix)
```text
Priority: Capability > TTFT
Default:  Llama 3.3 70B (fast enough for most)
Complex:  Qwen 3 235B (full reasoning for enterprise)
Long ctx: Gemini 2.5 Flash (1M context for documents)
```

### Implementation Recommendations

1. **Route by channel type**: Implement channel-aware model routing
   ```typescript
   function getModelForChannel(channel: string, complexity: 'simple' | 'complex'): string {
     const syncChannels = ['discord-command', 'imessage', 'voice'];
     const asyncChannels = ['slack-thread', 'msteams', 'matrix'];

     if (syncChannels.includes(channel)) {
       return 'llama-3.3-70b'; // Always prioritize TTFT
     }
     if (asyncChannels.includes(channel) && complexity === 'complex') {
       return 'qwen-3-235b'; // Can afford deeper reasoning
     }
     return 'llama-3.3-70b'; // Default
   }
   ```

2. **Adaptive streaming**: Enable streaming only where supported
   ```typescript
   const streamingChannels = ['discord', 'telegram', 'slack', 'msteams', 'matrix'];
   const shouldStream = streamingChannels.includes(channel);
   ```

3. **Fallback for non-streaming**: For channels without streaming, consider:
   - Sending "typing" indicator immediately
   - Using faster models to compensate for no progressive display
   - Breaking long responses into multiple messages

4. **Voice-specific optimization**: Voice channels need special handling:
   - Target <300ms TTFT for natural conversation flow
   - Consider audio-optimized streaming (chunked TTS)
   - Use shorter system prompts to reduce latency

---

## 8. No-Code Quick Wins (Do These First)

These optimizations require no code changes - just configuration or usage patterns. **Start here before implementing anything.**

### 1. Switch Default Model
Change default model to **Cerebras Llama 3.3 70B (Pro)** for 10-20x faster responses (170ms TTFT). Pro subscription unlocks full 128K context on all Cerebras models.

### 2. Prompt Structure Optimization
Place static content FIRST (system prompt, tool definitions) for prefix cache hits:
```text
[STATIC: System prompt, tools] → cacheable
[DYNAMIC: Conversation history, RAG] → at end
```

### 3. Enable Provider Caching
Enable Anthropic prefix caching for up to 85% latency reduction on repeated prompts.

### 4. Output Token Limits
Add explicit constraints to reduce generation time:
- "Answer in under 50 words"
- "Provide a brief summary"
- Reduces latency proportionally to token reduction

### 5. Avoid JSON Where Possible
JSON responses are slower to generate than plain text.
Use structured output only when parsing is required.

### 6. Parallel Tool Calls
When multiple tools are needed, execute in parallel (Moltbot already supports this).

---

## 9. Implementation Roadmap (Requires Code Changes)

After applying the no-code optimizations above, consider these tiered improvements.

### Tier 1: Near-Term (Next Sprint)

| Technique | Improvement | Effort |
|-----------|-------------|--------|
| Simple model routing | 40-70% cost reduction | New code |
| Semantic caching | 31% cache hits | Integration |
| Prompt compression (long inputs) | 5-20x compression | Integration |

### Tier 2: Advanced (Future)

| Technique | Improvement | Effort |
|-----------|-------------|--------|
| Speculative decoding (local) | 2-4x speedup | Complex |
| KV cache optimization | 3-10x latency | Infrastructure |
| Custom routing classifier | Higher accuracy | Training needed |

---

## 10. Moltbot Integration Points

This section maps research recommendations to specific Moltbot codebase locations for implementation.

### Configuration File Locations

| Config Purpose | File Path |
|----------------|-----------|
| Main config | `~/.clawdbot/config.json` (or `moltbot.json` in workspace) |
| Auth profiles (credentials) | `~/.clawdbot/auth-profiles.json` |
| Model definitions | `~/.clawdbot/agents/<agentId>/models.json` (auto-generated) |

### Key Source Files

| Component | File | Purpose |
|-----------|------|---------|
| Default model/provider | `src/agents/defaults.ts` | `DEFAULT_PROVIDER`, `DEFAULT_MODEL` constants |
| Model selection logic | `src/agents/model-selection.ts` | `resolveConfiguredModelRef()`, alias resolution |
| Model config types | `src/config/types.agent-defaults.ts` | `AgentModelListConfig`, `AgentModelEntryConfig` |
| Provider config types | `src/config/types.models.ts` | `ModelProviderConfig`, `ModelDefinitionConfig` |
| Config defaults | `src/config/defaults.ts` | Model aliases, default model resolution |
| Auth profiles | `src/agents/auth-profiles/*.ts` | Provider auth management |
| Prefix caching | `src/agents/pi-embedded-runner/extra-params.ts` | `cacheControlTtl` param handling |
| System prompt builder | `src/agents/system-prompt.ts` | `buildAgentSystemPrompt()` |
| Embedded runner | `src/agents/pi-embedded-runner/run.ts` | Main agent execution loop |

### Implementation Checklist

#### 1. Switch to Cerebras Llama 3.3 70B (Config Change)

**Step 1: Add Cerebras auth profile**

Edit `~/.clawdbot/config.json`:
```json
{
  "auth": {
    "profiles": {
      "cerebras-pro": {
        "provider": "cerebras",
        "mode": "api_key"
      }
    },
    "order": {
      "cerebras": ["cerebras-pro"]
    }
  }
}
```

Then store the API key:
```bash
moltbot auth add cerebras-pro --provider cerebras --mode api_key
# Enter your Cerebras API key when prompted
```

**Step 2: Configure Cerebras as a provider**

Add to `~/.clawdbot/config.json`:
```json
{
  "models": {
    "providers": {
      "cerebras": {
        "baseUrl": "https://api.cerebras.ai/v1",
        "api": "openai-completions",
        "models": [
          {
            "id": "llama-3.3-70b",
            "name": "Llama 3.3 70B",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 128000,
            "maxTokens": 8192,
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
          },
          {
            "id": "llama-3.1-8b",
            "name": "Llama 3.1 8B",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 128000,
            "maxTokens": 8192,
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
          },
          {
            "id": "qwen-3-32b",
            "name": "Qwen 3 32B",
            "reasoning": true,
            "input": ["text"],
            "contextWindow": 131000,
            "maxTokens": 8192,
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
          }
        ]
      }
    }
  }
}
```

**Step 3: Set Cerebras as default model**

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "cerebras/llama-3.3-70b",
        "fallbacks": ["cerebras/llama-3.1-8b", "anthropic/claude-sonnet-4-5"]
      },
      "models": {
        "cerebras/llama-3.3-70b": { "alias": "cerebras" },
        "cerebras/llama-3.1-8b": { "alias": "cerebras-fast" },
        "cerebras/qwen-3-32b": { "alias": "cerebras-reason" }
      }
    }
  }
}
```

**Complete config example:**
```json
{
  "auth": {
    "profiles": {
      "cerebras-pro": { "provider": "cerebras", "mode": "api_key" }
    },
    "order": {
      "cerebras": ["cerebras-pro"]
    }
  },
  "models": {
    "providers": {
      "cerebras": {
        "baseUrl": "https://api.cerebras.ai/v1",
        "api": "openai-completions",
        "models": [
          {
            "id": "llama-3.3-70b",
            "name": "Llama 3.3 70B",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 128000,
            "maxTokens": 8192,
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "cerebras/llama-3.3-70b",
        "fallbacks": ["anthropic/claude-sonnet-4-5"]
      },
      "models": {
        "cerebras/llama-3.3-70b": { "alias": "cerebras" }
      }
    }
  }
}
```

#### 2. Enable Anthropic Prefix Caching (Config Change)

Anthropic prefix caching is automatic when using `cacheControlTtl`. Set in config:

```json
{
  "agents": {
    "defaults": {
      "models": {
        "anthropic/claude-sonnet-4-5": {
          "alias": "sonnet",
          "params": {
            "cacheControlTtl": "1h"
          }
        },
        "anthropic/claude-opus-4-5": {
          "alias": "opus",
          "params": {
            "cacheControlTtl": "1h"
          }
        }
      }
    }
  }
}
```

**How it works in code:**
- `src/agents/pi-embedded-runner/extra-params.ts` reads `params.cacheControlTtl`
- Valid values: `"5m"` or `"1h"`
- Only applies to `anthropic` provider (or `openrouter` with `anthropic/` prefix)
- The param is passed to `@mariozechner/pi-ai` `streamSimple()` which sets Anthropic cache headers

#### 3. Configure Model Routing (New Code Required)

Model routing would require new code. Suggested implementation path:

**Option A: Simple directive-based routing (config-only)**

Use existing `/model` directive with aliases:
```json
{
  "agents": {
    "defaults": {
      "models": {
        "cerebras/llama-3.3-70b": { "alias": "fast" },
        "cerebras/qwen-3-32b": { "alias": "reason" },
        "anthropic/claude-opus-4-5": { "alias": "deep" }
      }
    }
  }
}
```

Users can switch with `/model fast`, `/model reason`, `/model deep`.

**Option B: Automatic routing (requires code)**

Would need a new module at `src/agents/model-router.ts`:
```typescript
// Pseudo-code for future implementation
import type { MoltbotConfig } from "../config/config.js";
import type { ModelCatalogEntry } from "./model-catalog.js";

export type RoutingDecision = {
  provider: string;
  model: string;
  reason: string;
};

export function routeQuery(params: {
  query: string;
  cfg: MoltbotConfig;
  catalog: ModelCatalogEntry[];
}): RoutingDecision {
  // Simple heuristics:
  // - Short queries (<50 chars) -> fast model
  // - "reason", "analyze", "explain" keywords -> reasoning model
  // - Code/math patterns -> reasoning model
  // - Default -> primary model
}
```

Integration point: `src/agents/pi-embedded-runner/run.ts` at line ~95-97 where `provider` and `modelId` are resolved.

#### 4. Prompt Structure Optimization (System Prompt)

Moltbot already optimizes prompt structure for caching. The system prompt is built with static content first:

From `src/agents/system-prompt.ts`:
```text
[STATIC: Identity, Skills, Memory, User Identity sections]
[STATIC: Time, Reply Tags, Messaging, Voice, Docs sections]
[STATIC: Tool definitions and summaries]
[DYNAMIC: Workspace notes, context files]
```

**To verify cache hits** (Anthropic), check logs for:
- `response.usage.prompt_tokens_details.cached_tokens`

The cache trace diagnostic can be enabled:
```json
{
  "diagnostics": {
    "cacheTrace": {
      "enabled": true,
      "filePath": "~/.clawdbot/logs/cache-trace.jsonl",
      "includePrompt": true
    }
  }
}
```

### CLI Commands Reference

```bash
# Check current model config
moltbot config get agents.defaults.model

# Set primary model
moltbot config set agents.defaults.model.primary "cerebras/llama-3.3-70b"

# Add model to allowlist with alias
moltbot config set 'agents.defaults.models["cerebras/llama-3.3-70b"].alias' "cerebras"

# Check auth profiles
moltbot auth list

# Add Cerebras auth
moltbot auth add cerebras-pro --provider cerebras --mode api_key

# Verify model is accessible
moltbot models list

# Test model directly
moltbot message send --model cerebras/llama-3.3-70b "Hello, test!"
```

### Verification Steps

After configuration:

1. **Check config is valid:**
   ```bash
   moltbot doctor
   ```

2. **Verify model resolution:**
   ```bash
   moltbot status --deep
   ```

3. **Test Cerebras connectivity:**
   ```bash
   moltbot message send --model cerebras/llama-3.3-70b "What is 2+2?"
   ```

4. **Monitor latency:**
   - Check gateway logs for TTFT
   - Enable diagnostics for detailed timing

---

## Sources

### Vendor Documentation (Cerebras)
- Cerebras Pro 128K Context (all models): Verified via user's Pro subscription testing (Jan 2026); see https://cerebras.ai/pricing
- Cerebras Inference Docs: https://inference-docs.cerebras.ai/
- Cerebras MoE Guide & Calculator: https://www.cerebras.ai/blog/moe-guide-calculator
- Cerebras CePO (Test-Time Reasoning): https://inference-docs.cerebras.ai/capabilities/cepo
- Cerebras Batch API: https://inference-docs.cerebras.ai/capabilities/batch
- Cerebras Predicted Outputs: https://inference-docs.cerebras.ai/capabilities/predicted-outputs
- Cerebras Prompt Caching: https://inference-docs.cerebras.ai/capabilities/prompt-caching
- Cerebras Reasoning Models: https://inference-docs.cerebras.ai/capabilities/reasoning
- Cerebras Structured Outputs: https://inference-docs.cerebras.ai/capabilities/structured-outputs
- Cerebras Service Tiers: https://inference-docs.cerebras.ai/capabilities/service-tiers
- Cerebras Cookbook: https://inference-docs.cerebras.ai/cookbook

### Independent Benchmarks & Third-Party Analysis
- Artificial Analysis LLM Leaderboard: https://artificialanalysis.ai/leaderboards/models
- Artificial Analysis Methodology: https://artificialanalysis.ai/methodology
- Artificial Analysis Provider Comparison: https://artificialanalysis.ai/leaderboards/providers
- LLMPerf (Ray Project/Anyscale): https://github.com/ray-project/llmperf
- LLMPerf Leaderboard: https://github.com/ray-project/llmperf-leaderboard
- Anyscale LLMPerf Blog: https://www.anyscale.com/blog/reproducible-performance-metrics-for-llm-inference
- LLM-Stats Leaderboard: https://llm-stats.com/leaderboards/llm-leaderboard
- Hugging Face LLM-Perf Leaderboard: https://huggingface.co/spaces/optimum/llm-perf-leaderboard
- MLPerf Inference Benchmarks: https://www.redhat.com/en/blog/efficient-and-reproducible-llm-inference-red-hat-mlperf-inference-v51-results
- LLM-Inference-Bench (Academic): https://arxiv.org/html/2411.00136v1

### Competitor Comparisons
- Groq LPU Benchmark Results: https://groq.com/blog/groq-lpu-inference-engine-crushes-first-public-llm-benchmark
- Groq vs Artificial Analysis: https://groq.com/blog/artificialanalysis-ai-llm-benchmark-doubles-axis-to-fit-new-groq-lpu-inference-engine-performance-results
- Cerebras vs Groq Comparison: https://www.cerebras.ai/blog/cerebras-cs-3-vs-groq-lpu
- Cerebras vs SambaNova vs Groq (IntuitionLabs): https://intuitionlabs.ai/articles/cerebras-vs-sambanova-vs-groq-ai-chips
- Groq Inference Economics (SemiAnalysis): https://newsletter.semianalysis.com/p/groq-inference-tokenomics-speed-but
- AI Providers Comparison (AIMultiple): https://research.aimultiple.com/ai-providers/
- LLM Latency Benchmark by Use Cases: https://research.aimultiple.com/llm-latency-benchmark/

### Model Routing & Optimization
- RouteLLM: https://github.com/lm-sys/RouteLLM
- RouteLLM Blog: https://lmsys.org/blog/2024-07-01-routellm/
- Semantic Router: https://github.com/aurelio-labs/semantic-router
- vLLM Semantic Router (Iris): https://blog.vllm.ai/2026/01/05/vllm-sr-iris.html
- Cascade Routing (ICLR 2025): https://arxiv.org/abs/2410.10347
- Cascade Routing GitHub: https://github.com/eth-sri/cascade-routing
- RouterArena Evaluation: https://huggingface.co/blog/JerryPotter/who-routes-the-routers
- Anyscale LLM Router: https://github.com/anyscale/llm-router

### Caching & Optimization
- LLMLingua: https://www.llmlingua.com/
- vLLM: https://docs.vllm.ai/
- Gemini Pricing: https://ai.google.dev/gemini-api/docs/pricing
- Anthropic Prompt Caching: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- GPTCache: https://github.com/zilliztech/GPTCache
- KV Cache Optimization: https://introl.com/blog/kv-cache-optimization-memory-efficiency-production-llms-guide
- Speculative Decoding: https://docs.vllm.ai/en/latest/features/spec_decode/
