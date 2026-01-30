---
module: Gateway
date: 2026-01-30
problem_type: performance_issue
component: service_object
symptoms:
  - "LLM API latency is 85-95% of response time"
  - "Users waiting too long for bot responses"
  - "High token costs on production workloads"
root_cause: config_error
resolution_type: config_change
severity: high
tags: [llm, latency, cerebras, caching, optimization]
---

# LLM Latency Optimization Techniques

## Problem Statement

Moltbot gateway's primary bottleneck is LLM API latency (85-95% of total response time). Users expect fast responses on messaging channels.

## Key Findings

### Target Metrics

| Metric | Target | Critical |
|--------|--------|----------|
| Time to First Token (TTFT) | <250ms | <500ms |
| Tokens/second | >50 | >20 |
| P99 Latency | <2s | <5s |
| Cache Hit Rate | >70% | >50% |

### Optimization Techniques by Impact

#### Tier 1: Immediate (Config Changes)

| Technique | Impact | Effort |
|-----------|--------|--------|
| **Switch to Cerebras Llama 3.3 70B** | 170ms TTFT, 128K context | Config |
| **Enable streaming** | Perceived latency → 0 | Already done |
| **Anthropic prefix caching** | 85% latency, 90% cost reduction | Config |

#### Tier 2: Medium-Term (New Code)

| Technique | Impact | Effort |
|-----------|--------|--------|
| **Semantic caching** | 250x faster on cache hits | Integration |
| **Model routing** | 76% cost reduction | New code |
| **Continuous batching** | 23x throughput | Framework |

#### Tier 3: Advanced

| Technique | Impact | Effort |
|-----------|--------|--------|
| **Speculative decoding** | 2-4x speedup | Complex |
| **KV cache optimization** | 3-10x latency | Infrastructure |

## Solution: Model Selection for Moltbot

### With Cerebras Pro Subscription

```
Default       → Llama 3.3 70B (170ms TTFT, 128K context)
Ultra-fast    → GPT OSS 120B (3,000 t/s for short responses)
Deep reasoning → Qwen 3 235B (131K context, complex tasks)
Long documents → Gemini 2.5 Flash (1M context)
```

### Channel-Specific Recommendations

| Channel | TTFT Target | Model |
|---------|-------------|-------|
| Discord commands | <500ms | Llama 3.3 70B |
| iMessage | <500ms | Llama 3.3 70B |
| WhatsApp | <1s | Llama 3.3 70B |
| Slack threads | <2s | Qwen 3 235B (if complex) |

## Implementation Checklist

- [ ] Set Cerebras Llama 3.3 70B as default model
- [ ] Enable prefix caching in system prompt construction
- [ ] Implement semantic cache (start with exact-match)
- [ ] Add simple model router based on query complexity
- [ ] Monitor TTFT in production logs

## Related

- `.claude/research/code-optimization/llm-response-optimization.md` - Full research
- `.claude/research/model-optimization/research-findings.md` - Model benchmarks
- `.claude/research/code-optimization/model-routing.md` - Routing strategies
