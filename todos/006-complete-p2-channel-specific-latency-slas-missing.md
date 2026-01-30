---
status: complete
priority: p2
issue_id: "006"
tags: [code-review, research, moltbot-specific]
dependencies: []
---

# Channel-Specific Latency SLAs Missing

## Problem Statement

Moltbot is a multi-channel messaging bot, but the research document provides no guidance on model selection per channel based on latency requirements.

**What's broken:** No differentiation between synchronous channels (Discord commands, iMessage - need <500ms TTFT) vs async channels (Slack threads, Telegram edits - can wait 3s).

**Why it matters:** Using a slower model on synchronous channels creates poor UX. Using a fast model on async channels wastes optimization opportunity.

## Findings

**Moltbot channels (from CLAUDE.md):**
- Core: Discord, Telegram, Slack, Signal, iMessage, WhatsApp
- Extensions: Matrix, MS Teams, Zalo, Google Chat, Twitch

**Missing in research:**
- Channel → TTFT requirement mapping
- Channel → model selection matrix
- Streaming vs non-streaming capabilities per channel

**Location:** `.claude/research/model-optimization/research-findings.md` - no section on channel-specific optimization

## Proposed Solutions

### Option 1: Add Channel Routing Matrix (Recommended)
- **Pros:** Directly actionable for Moltbot
- **Cons:** Requires understanding each channel's UX expectations
- **Effort:** Medium (1 hour)
- **Risk:** Low

**Example addition:**
```markdown
## Channel-Specific Model Selection

| Channel | TTFT Target | Streaming | Recommended Model |
|---------|-------------|-----------|-------------------|
| Discord (commands) | <500ms | Yes | Llama 3.3 70B |
| iMessage | <500ms | No | Llama 3.3 70B |
| Slack (threads) | <2s | Yes | Qwen 3 235B (if complex) |
| WhatsApp | <1s | No | Llama 3.3 70B |
```

### Option 2: Add to code-optimization research
- **Pros:** Keeps model research focused on models
- **Cons:** Information split across documents
- **Effort:** Medium
- **Risk:** Low

## Recommended Action

<!-- Fill during triage -->

## Technical Details

**Affected files:**
- `.claude/research/model-optimization/research-findings.md`

**Related Moltbot files:**
- `src/channels/` - Channel abstraction
- `src/discord/`, `src/telegram/`, etc. - Channel implementations

## Acceptance Criteria

- [x] Channel → TTFT requirement table added
- [x] Model selection guidance per channel type
- [x] Streaming capability documented per channel

## Work Log

| Date | Action | Outcome |
|------|--------|---------|
| 2026-01-30 | Review identified Moltbot-specific gap | Created todo |

## Resources

- Research document: `.claude/research/model-optimization/research-findings.md`
- Moltbot channels: `src/channels/`
- Code optimization research: `.claude/research/code-optimization/`
