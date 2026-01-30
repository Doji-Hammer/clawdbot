---
status: complete
priority: p1
issue_id: "002"
tags: [code-review, research, accuracy]
dependencies: []
---

# Qwen 3 Context Lengths Incorrect

## Problem Statement

The research document contains incorrect Qwen 3 model context length specifications that could lead to wrong model selection decisions.

**What's broken:** Document states Qwen 3 32B has "65K (free)" and Qwen 3 235B has "64K (free)" context, but verified specs show different values.

**Why it matters:** Context length is critical for model routing. Incorrect values could cause truncation errors or suboptimal model selection.

## Findings

**Document claims:**
- Qwen 3 32B: 65K (free), 131K (pro)
- Qwen 3 235B: 64K (free), 131K (pro)

**Verified specs:**
- Qwen3-32B: Native 32,768 tokens (extendable to 131K via YaRN)
- Qwen3-235B-A22B: Native 262,144 tokens (extendable to 1M)

**Location:** `.claude/research/model-optimization/research-findings.md` lines 54-59

## Proposed Solutions

### Option 1: Correct to Verified Specs (Recommended)
- **Pros:** Accurate data
- **Cons:** May need to verify Cerebras-specific limits vs native model limits
- **Effort:** Small (20 min research + update)
- **Risk:** Low

### Option 2: Distinguish Native vs Provider Limits
- **Pros:** Clarifies that Cerebras may impose different limits than native model
- **Cons:** More complex table
- **Effort:** Medium (30 min)
- **Risk:** Low

## Recommended Action

<!-- Fill during triage -->

## Technical Details

**Affected files:**
- `.claude/research/model-optimization/research-findings.md`

**Lines to update:**
- Lines 54-59 (Context Length table)

**Sources to verify:**
- https://inference-docs.cerebras.ai/models/qwen-3-32b
- Cerebras official model docs

## Acceptance Criteria

- [x] Context lengths match official Qwen3 specs OR Cerebras provider limits
- [x] Table clearly indicates if values are native vs provider-limited
- [x] Sources cited for each value

## Work Log

| Date | Action | Outcome |
|------|--------|---------|
| 2026-01-30 | Review identified discrepancy | Created todo |

## Resources

- [Qwen3-32B Specs](https://apxml.com/models/qwen3-32b)
- [Qwen3-235B Specs](https://apxml.com/models/qwen3-235b-a22b-thinking)
- Research document: `.claude/research/model-optimization/research-findings.md`
