# Model Optimization Implementation Plan

**Created:** 2026-01-30
**Updated:** 2026-01-30
**Deepened:** 2026-01-30
**Research:** `.claude/research/model-optimization/research-findings.md`
**Goal:** Switch to Cerebras Llama 3.3 70B as default for faster responses

---

## Enhancement Summary

**Research agents used:** 6 parallel agents
- Cerebras-inference skill application
- LLM latency optimization learning
- LLM fallback patterns research
- Configuration management best practices
- LLM observability patterns
- Model selection strategies

### Key Improvements Added
1. Cerebras-specific API parameters (service tier, prompt caching)
2. Fallback trigger conditions and circuit breaker patterns
3. Monitoring metrics and alerting thresholds
4. Config validation and rollback automation
5. Future model routing strategies (Phase 2+)

---

## Prerequisites

- [ ] Cerebras Pro subscription active
- [ ] API key from https://cloud.cerebras.ai/
- [ ] Moltbot CLI installed (`moltbot --version`)
- [ ] Gateway running (menubar app or `moltbot gateway status`)
- [ ] Config file exists (`~/.clawdbot/config.json`)

---

## Expected Improvements

| Metric | Current | Target | Notes |
|--------|---------|--------|-------|
| TTFT (p50) | ~600ms | <200ms | Best-case 170ms; may queue during US peak hours |
| TTFT (p95) | Unknown | <500ms | Account for Cerebras queue times |
| Generation | ~80 t/s | ~2000 t/s | 25x improvement |
| Cost | Variable | <$10/mo | Cerebras Pro subscription covers usage |

### Research Insights: Performance Validation

The performance claims are **realistic and well-sourced**:
- Cerebras Llama 3.3 70B: 170ms TTFT, 2,100-2,500 t/s (official benchmarks)
- Claude Sonnet baseline: ~600ms TTFT, 77-82 t/s (matches "~80 t/s current")
- For 100-token response: Current ~1850ms → Cerebras ~220ms = **8.4x faster**
- For 500-token response: Current ~6850ms → Cerebras ~420ms = **16.3x faster**

**Caveats:**
- TTFT can degrade during US peak hours (Cerebras notes "near 100% utilization")
- Network latency to Cerebras API adds 50-200ms depending on region
- These are p50 numbers; add p95/p99 monitoring

---

## Phase 1: Configuration (Effort: ~30 minutes)

### Step 1: Backup Current Config

```bash
cp ~/.clawdbot/config.json ~/.clawdbot/config.json.bak
```

#### Research Insights: Backup Strategy

Beyond simple backup, consider:
- **Timestamped backups**: `config.json.2026-01-30T10-30-00.bak`
- **Verify backup is valid**: Parse JSON before proceeding
- **Named backups before migrations**: `config.json.pre-cerebras.bak`

Moltbot already implements 5-file backup rotation internally.

### Step 2: Add Cerebras Credentials

```bash
moltbot models auth paste-token --provider cerebras
# Paste your Cerebras Pro API key when prompted
```

Or use environment variable:
```bash
export CEREBRAS_API_KEY="your-api-key-here"
# Add to ~/.zshrc or ~/.bashrc for persistence
```

#### Research Insights: Credential Security

- **Prefer auth profile over env var** - integrates with Moltbot's cooldown system
- **Never store keys in config directly** - use env var references or auth profiles
- **Key rotation procedure**:
  1. Add new key as secondary profile
  2. Test new key works
  3. Remove old profile

### Step 3: Test Cerebras Connectivity (Before Changing Default)

```bash
moltbot message send --model cerebras/llama-3.3-70b "Hello, this is a test"
```

**Expected:** Response in <1 second. If this fails, do not proceed.

#### Research Insights: Pre-flight Validation

Run these additional checks:
```bash
# Validate config syntax
moltbot doctor

# Check provider is reachable (if available)
# moltbot models test --provider cerebras --model llama-3.3-70b
```

### Step 4: Add Cerebras Provider Config

Edit `~/.clawdbot/config.json` and add/merge the following:

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
            "contextWindow": 128000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

#### Research Insights: Cerebras API Configuration

**Service Tier Selection** - Use `priority` for user-facing requests:
```json
{
  "providers": {
    "cerebras": {
      "defaultParams": {
        "service_tier": "priority"
      }
    }
  }
}
```

Service tier options:
| Tier | Priority | Use Case |
|------|----------|----------|
| `priority` | Highest | User-facing, time-critical |
| `default` | Standard | Production workloads |
| `flex` | Lowest | Batch tasks, higher limits |

**Prompt Structure for Cache Hits** - Cerebras caches in 100-600 token blocks:
```
[System prompt + tool definitions] ← Cached (5min+ TTL)
[Conversation history]             ← Dynamic
[Current query]                    ← Dynamic
```

Check cache effectiveness via `response.usage.prompt_tokens_details.cached_tokens`.

### Step 5: Set Cerebras as Default Model

Add/merge into `~/.clawdbot/config.json`:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "cerebras/llama-3.3-70b",
        "fallbacks": ["anthropic/claude-sonnet-4-5"]
      }
    }
  }
}
```

#### Research Insights: Fallback Configuration

**Fallback Trigger Conditions:**
| Trigger | Action |
|---------|--------|
| HTTP 429 (Rate Limit) | Immediate fallback |
| HTTP 5xx errors | Retry once, then fallback |
| HTTP 503 (Unavailable) | Immediate fallback |
| Request timeout (>10-15s) | Fallback |
| Network errors | Retry once, then fallback |

**Do NOT fallback on:**
- HTTP 400/401/403 (client errors) - fix the request
- HTTP 413 (payload too large) - reduce input

**Retry Strategy Before Fallback:**
```
max_retries: 2
initial_delay: 1 second
max_delay: 4 seconds
jitter: ±20%
```

Timeline: 0s → retry at 1s → retry at 2-4s → fallback at ~5s total

### Step 6: Restart Gateway

**Option A:** Restart via menubar app (click Moltbot icon → Restart)

**Option B:** Command line:
```bash
scripts/restart-mac.sh
```

### Step 7: Verify

```bash
# Validate config
moltbot doctor

# Check model status
moltbot models status

# Test default model (should use Cerebras)
moltbot message send "What model are you?"
```

**Expected output from `moltbot models status`:**
- Primary: `cerebras/llama-3.3-70b`
- Fallbacks: `anthropic/claude-sonnet-4-5`
- Cerebras auth: configured

#### Research Insights: Verification Checklist

**Before/After Latency Test:**
```bash
# Baseline (if not already switched)
time moltbot message send --model anthropic/claude-sonnet-4-5 "Hello"

# After changes
time moltbot message send --model cerebras/llama-3.3-70b "Hello"

# Expected: TTFT drops from ~600ms to ~170ms
```

**Fallback Verification:**
```bash
# Temporarily break Cerebras to test fallback
export CEREBRAS_API_KEY="invalid"
moltbot message send "Hello"  # Should fall back to Claude
unset CEREBRAS_API_KEY        # Restore (uses auth profile)
```

---

## Phase 2: Add More Models (Optional, Future)

Once Phase 1 is validated, add additional Cerebras models:

```json
{
  "models": {
    "providers": {
      "cerebras": {
        "models": [
          { "id": "llama-3.3-70b", "name": "Llama 3.3 70B", "contextWindow": 128000, "maxTokens": 8192 },
          { "id": "llama-3.1-8b", "name": "Llama 3.1 8B", "contextWindow": 128000, "maxTokens": 8192 },
          { "id": "qwen-3-32b", "name": "Qwen 3 32B", "reasoning": true, "contextWindow": 131000, "maxTokens": 8192 }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "models": {
        "cerebras/llama-3.3-70b": { "alias": "cerebras" },
        "cerebras/llama-3.1-8b": { "alias": "fast" },
        "cerebras/qwen-3-32b": { "alias": "reason" }
      }
    }
  }
}
```

Usage: `/model fast`, `/model reason`, `/model cerebras`

#### Research Insights: Model Selection Guide

| Model | Speed | Best For |
|-------|-------|----------|
| `llama-3.3-70b` | 2,100-2,500 t/s | Default, all tasks |
| `llama-3.1-8b` | ~2,000 t/s | Simple Q&A, greetings |
| `qwen-3-32b` | 2,600 t/s | Complex reasoning, code |
| `qwen-3-235b` | 1,400 t/s | Maximum capability |
| `gpt-oss-120b` | 3,000 t/s | Ultra-fast short responses |

**Reasoning Model Parameters:**
```typescript
// GPT-OSS with reasoning effort
{ model: "gpt-oss-120b", reasoning_effort: "high" }

// Qwen 3 with reasoning format
{ model: "qwen-3-32b", reasoning_format: "parsed" }
```

---

## Monitoring (Recommended)

### Key Metrics to Track

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| TTFT p50/p95 | Time to first token | p95 > 500ms |
| TPS | Tokens per second | < 500 t/s |
| Error rate | Failed requests | > 1% |
| Fallback rate | Primary failures | > 5% |
| Cost per day | Token usage | > budget/30 |

### Alerting Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| TTFT p95 | > 500ms | > 1,000ms |
| Error rate | > 1% | > 5% |
| Fallback rate | > 5% | > 20% |
| Timeout rate | > 1% | > 3% |

### Logging Best Practices

Log for each LLM request:
- Request ID, timestamp
- Provider, model
- TTFT, total latency
- Input/output tokens
- Status (success/error/fallback)
- Cost estimate

```bash
# Check logs for issues
./scripts/clawlog.sh -f

# Or check gateway log
tail -f /tmp/moltbot-gateway.log
```

---

## Troubleshooting

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `401 Unauthorized` | API key invalid or expired | Re-run `moltbot models auth paste-token --provider cerebras` |
| `429 Too Many Requests` | Rate limit hit | Wait and retry; check Cerebras Pro limits |
| `503 Service Unavailable` | Cerebras down | Fallback should auto-activate to Claude |
| `ECONNREFUSED` | Gateway not running | Restart gateway via menubar or `scripts/restart-mac.sh` |
| `context_length_exceeded` | Prompt too long | Truncate input or use model with larger context |

### Check Logs

```bash
# macOS unified logs
./scripts/clawlog.sh -f

# Or check gateway log directly
tail -f /tmp/moltbot-gateway.log
```

### Verify Fallback Works

Temporarily break Cerebras to test fallback:
```bash
# Set invalid key
export CEREBRAS_API_KEY="invalid"

# Send message - should fall back to Claude
moltbot message send "Hello"

# Restore valid key
unset CEREBRAS_API_KEY  # Uses auth profile instead
```

#### Research Insights: Error Handling

**Cerebras-Specific Error Patterns:**
```typescript
// Rate limiting - use retry-after header
if (error.status === 429) {
  const retryAfter = error.headers?.['retry-after'] || 60;
  await sleep(retryAfter * 1000);
}

// Context exhausted - reasoning models can consume context
if (error.code === 'context_length_exceeded') {
  // Option 1: Use /no_think suffix to disable reasoning
  // Option 2: Fall back to smaller model
  // Option 3: Truncate input
}
```

---

## Rollback Plan

If Cerebras has issues:

**Immediate (fallback auto-activates):**
- The fallback chain (`anthropic/claude-sonnet-4-5`) activates automatically on Cerebras errors

**Manual rollback:**
```bash
# Restore backup
cp ~/.clawdbot/config.json.bak ~/.clawdbot/config.json

# Restart gateway
scripts/restart-mac.sh

# Verify
moltbot doctor
```

**Or set Claude as primary:**
```bash
moltbot models set anthropic/claude-sonnet-4-5
```

#### Research Insights: Circuit Breaker Pattern

For automated protection, consider implementing:
```
CLOSED (normal) → OPEN (failing) → HALF-OPEN (testing)
```

Parameters:
- Failure threshold: 5 failures in 5-minute window
- Cooldown period: 60 seconds
- Success threshold: 2-3 successes to close

When OPEN, route 100% to fallback without attempting primary.

---

## Future Work (Separate Planning Required)

These optimizations require code changes and should be planned separately:

1. **Query-based model routing** - Route simple queries to fast models, complex to capable models
2. **Channel-aware routing** - Use faster models for sync channels (Discord, iMessage), allow slower for async (Slack threads)
3. **Semantic response cache** - Cache similar queries for instant responses
4. **Prompt compression** - Use LLMLingua for long context inputs

See `.claude/research/model-optimization/research-findings.md` for details.

### Research Insights: Model Routing Strategies

**Query Classification (No LLM Call):**
| Pattern | Model | Reason |
|---------|-------|--------|
| Greetings (`hi`, `hello`) | 8B fast | Simple acknowledgment |
| Reasoning keywords (`why`, `explain`) | 32B reason | Complex analysis |
| Code patterns (```, `function`) | 32B reason | Code context |
| Default | 70B balanced | General use |

**Channel-Based Defaults:**
| Channel | Latency Expectation | Default Model |
|---------|---------------------|---------------|
| Voice | Very Low (<300ms) | 8B fast only |
| iMessage, Discord cmd | Low (<500ms) | 70B balanced |
| Telegram, Signal | Medium (<1s) | 70B balanced |
| Slack threads | Tolerant (<2s) | 70B or 32B |

**Gradual Rollout Pattern:**
1. Shadow mode (1-2 weeks): Run all models, compare outputs
2. Opt-in (2 weeks): Users can `/fast`, `/think`
3. Smart routing (gradual): 20% → 50% → 100%

---

## Notes

- **Anthropic prefix caching** is automatic for Anthropic models - no config needed
- **Cerebras Pro** subscription unlocks 128K context on all models
- **Cost**: Cerebras Pro included in subscription; per-token charges only apply to fallback (Anthropic)
- **Peak hours**: Cerebras may queue requests during US business hours; TTFT can increase

---

## References

**Cerebras Documentation:**
- Cerebras Inference Docs: https://inference-docs.cerebras.ai/
- Cerebras Prompt Caching: https://inference-docs.cerebras.ai/capabilities/prompt-caching
- Cerebras Service Tiers: https://inference-docs.cerebras.ai/capabilities/service-tiers

**Best Practices:**
- LLM Fallback Patterns: https://portkey.ai/blog/retries-fallbacks-and-circuit-breakers-in-llm-apps/
- LLM Observability: https://langfuse.com/docs/observability/features/token-and-cost-tracking
- Configuration Management: https://blog.logrocket.com/creating-configuration-files-node-js-using-node-config/
