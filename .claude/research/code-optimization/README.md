# Code Optimization Research for Moltbot Gateway

**Research Date:** 2026-01-30

This directory contains comprehensive research on optimizing the Moltbot Gateway's AI-powered messaging system. The research covers five interconnected areas that together provide a roadmap for achieving faster response times, lower costs, and better scalability.

---

## Research Documents

| Document | Focus Area | Key Takeaway |
|----------|------------|--------------|
| [context-window-management.md](./context-window-management.md) | Memory & Context | Hybrid memory systems reduce tokens by 90%+ while maintaining quality |
| [parallel-agent-execution.md](./parallel-agent-execution.md) | Concurrency & Orchestration | Lane-based queues + parallel sub-agents enable 3x throughput |
| [llm-response-optimization.md](./llm-response-optimization.md) | Latency & Throughput | Streaming + semantic caching delivers sub-250ms TTFT |
| [model-routing.md](./model-routing.md) | Cost & Model Selection | Smart routing achieves 76% cost reduction |
| [gateway-profiling.md](./gateway-profiling.md) | Performance Monitoring | Event loop monitoring catches bottlenecks before users notice |

---

## Quick Reference: Key Metrics to Target

| Metric | Target | Critical | Current Baseline |
|--------|--------|----------|------------------|
| Time to First Token (TTFT) | <250ms | <500ms | Measure with profiling |
| Tokens/second | >50 | >20 | Provider-dependent |
| P99 Latency | <2s | <5s | Measure with APM |
| Cache Hit Rate | >70% | >50% | Implement caching first |
| Cost per Request | -50% vs baseline | -30% | Measure before routing |

---

## Implementation Roadmap

### Phase 1: Quick Wins (Week 1)

These optimizations require minimal code changes and provide immediate benefits:

1. **Enable Streaming Responses**
   - Immediate first-token delivery improves perceived latency
   - Users start reading before generation completes
   - See: [llm-response-optimization.md#streaming](./llm-response-optimization.md)

2. **Implement Response Caching**
   - Start with exact-match cache for deterministic outputs
   - Target FAQ-style queries and repeated requests
   - Expected: 50-70% latency reduction on cache hits

3. **Set Output Length Constraints**
   - Add `max_tokens` to all LLM calls
   - 40% processing time reduction for bounded outputs

4. **Compress System Prompts**
   - Audit current system prompts for verbosity
   - LLMLingua-style compression: 20x reduction possible
   - See: [context-window-management.md#compression](./context-window-management.md)

### Phase 2: Core Infrastructure (Weeks 2-4)

These require more substantial changes but deliver major improvements:

1. **Deploy Semantic Caching**
   ```typescript
   // Recommended: Redis LangCache or GPTCache
   const cached = await cache.lookup(query, 0.95);
   if (cached) return cached.response;
   ```
   - 95% similarity threshold to start
   - Expected: 250x faster for repetitive queries, 95% cost reduction

2. **Implement Model Routing**
   ```typescript
   const routerConfig = {
     simple: "claude-haiku",      // FAQ, greetings, extraction
     standard: "claude-sonnet",   // Most tasks
     complex: "claude-opus"       // Architecture, security, planning
   };
   ```
   - Route 60% to Haiku, 30% to Sonnet, 10% to Opus
   - Expected: 76% cost reduction

3. **Enable Continuous Batching**
   - Use vLLM or TensorRT-LLM for self-hosted models
   - 23x throughput vs static batching
   - See: [llm-response-optimization.md#batching](./llm-response-optimization.md)

4. **Add Summarization to Long Conversations**
   ```typescript
   if (messageCount > 15) {
     const summary = await summarize(oldMessages);
     context = [summaryMessage, ...recentMessages];
   }
   ```
   - ConversationSummaryBufferMemory pattern
   - 70-90% token reduction

### Phase 3: Advanced Optimization (Weeks 5-12)

For production-scale deployments:

1. **Deploy Speculative Decoding**
   - Pair large model with 1-7B draft model
   - 2-4x speedup on compatible hardware
   - See: [llm-response-optimization.md#speculative](./llm-response-optimization.md)

2. **Implement KV Cache Optimization**
   - PagedAttention for <4% memory waste
   - Prefix caching for shared system prompts (90% cost reduction)

3. **Set Up Lane-Based Command Queue**
   ```typescript
   const LANE_CONCURRENCY = {
     main: 1,       // Serialize user interactions
     subagent: 3,   // Parallel sub-agents
     cron: 2,       // Background tasks
   };
   ```
   - Already in Moltbot codebase: `src/process/command-queue.ts`

4. **Build Automatic Prompt Optimization Pipeline**
   - Measure prompt token usage
   - A/B test compressed variants
   - Continuous improvement loop

---

## Architecture Recommendations

### For Moltbot Gateway (TypeScript/Node.js)

```
┌─────────────────────────────────────────────────────────────┐
│                     Gateway Architecture                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│   Inbound Message                                             │
│        │                                                      │
│        ▼                                                      │
│   ┌─────────────┐    ┌─────────────┐                         │
│   │  Semantic   │───▶│   Cache     │──▶ Return cached        │
│   │   Cache     │    │    Hit?     │                         │
│   └─────────────┘    └──────┬──────┘                         │
│                             │ Miss                            │
│                             ▼                                 │
│   ┌─────────────┐    ┌─────────────┐                         │
│   │  Complexity │───▶│   Route     │                         │
│   │  Classifier │    │   Model     │                         │
│   └─────────────┘    └──────┬──────┘                         │
│                             │                                 │
│              ┌──────────────┼──────────────┐                 │
│              ▼              ▼              ▼                 │
│         ┌────────┐    ┌────────┐    ┌────────┐              │
│         │ Haiku  │    │ Sonnet │    │  Opus  │              │
│         │  60%   │    │  30%   │    │  10%   │              │
│         └────┬───┘    └────┬───┘    └────┬───┘              │
│              └──────────────┼──────────────┘                 │
│                             ▼                                 │
│   ┌─────────────┐    ┌─────────────┐                         │
│   │  Context    │◀──▶│   Memory    │                         │
│   │  Manager    │    │   System    │                         │
│   └─────────────┘    └─────────────┘                         │
│        │                                                      │
│        ▼                                                      │
│   Stream Response to Channel                                  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Concurrency Configuration

```typescript
// Recommended starting configuration for Moltbot
const CONCURRENCY_CONFIG = {
  lanes: {
    main: 1,        // Serialize main user interactions
    subagent: 3,    // Allow parallel sub-agent execution
    cron: 2,        // Background tasks
    nested: 1,      // Serialize nested calls
  },
  rateLimits: {
    anthropic: 45,  // Under 50 concurrent limit
    openai: 40,
    cerebras: 100,  // High throughput
  },
  contextBudgets: {
    whatsapp: 4000,
    telegram: 8000,
    discord: 8000,
    slack: 8000,
  }
};
```

---

## Channel-Specific Recommendations

| Channel | Context Budget | Summarize After | Memory Persistence |
|---------|---------------|-----------------|-------------------|
| WhatsApp | 4K tokens | 10 messages | Yes |
| Telegram | 8K tokens | 15 messages | Yes |
| Discord | 8K tokens | 20 messages | Yes |
| Slack | 8K tokens | 20 messages | Yes |
| Signal | 4K tokens | 10 messages | Yes |
| iMessage | 4K tokens | 10 messages | Yes |

---

## Cost Optimization Summary

| Technique | Cost Reduction | Implementation Effort |
|-----------|---------------|----------------------|
| Model Routing (Haiku/Sonnet/Opus) | 76% | Medium |
| Semantic Caching | 60-73% | Medium |
| Prompt Compression | 20-40% | Low |
| Prompt Caching (Anthropic) | Up to 90% | Low |
| Summarization | 30-50% | Medium |
| **Combined** | **Up to 90%** | High |

---

## Monitoring Checklist

Essential metrics to track after implementing optimizations:

- [ ] TTFT (Time to First Token) - P50, P95, P99
- [ ] End-to-end latency by channel
- [ ] Cache hit rate (semantic + exact)
- [ ] Model routing distribution (% to each tier)
- [ ] Token usage per request
- [ ] Cost per request by model
- [ ] Event loop lag (Node.js)
- [ ] Memory usage trends
- [ ] Error rates by provider

---

## Related Resources

### Moltbot-Specific Files

- `src/process/command-queue.ts` - Lane-based command queue implementation
- `src/media-understanding/concurrency.ts` - `runWithConcurrency` helper
- `src/agents/pi-embedded-runner/` - Agent runtime with model selection

### External Tools

- [RouteLLM](https://github.com/lm-sys/RouteLLM) - Open-source model routing
- [GPTCache](https://github.com/zilliztech/GPTCache) - Semantic caching
- [LLMLingua](https://github.com/microsoft/LLMLingua) - Prompt compression
- [vLLM](https://docs.vllm.ai/) - High-throughput LLM serving
- [OpenTelemetry](https://opentelemetry.io/) - Observability framework

---

## Next Steps

1. **Measure Current Baseline**
   - Set up profiling with OpenTelemetry or Datadog
   - Capture TTFT, latency, and cost metrics

2. **Implement Quick Wins**
   - Enable streaming if not already active
   - Add exact-match caching for repeated queries

3. **Plan Model Routing**
   - Analyze query complexity distribution
   - Design classification strategy

4. **Design Context Management**
   - Choose summarization approach
   - Implement per-channel token budgets

---

*Research compiled from web sources, academic papers, and industry best practices. See individual documents for complete source citations.*
