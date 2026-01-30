---
status: complete
priority: p2
issue_id: "004"
tags: [code-review, documentation, structure]
dependencies: []
---

# Missing Table of Contents

## Problem Statement

The 500-line research document has no table of contents, making navigation difficult.

**What's broken:** 8 major sections with no TOC or anchor links for quick navigation.

**Why it matters:** Reduces usability of the reference document. Readers must scroll to find sections.

## Findings

- Document has 8 numbered sections plus "Constraints Summary"
- No table of contents after header
- Sections could be linked with standard Markdown anchors
- Mintlify docs pattern: root-relative, no .md extension

**Location:** `.claude/research/model-optimization/research-findings.md` (top of document)

## Proposed Solutions

### Option 1: Add TOC with Anchor Links (Recommended)
- **Pros:** Standard solution, clickable navigation
- **Cons:** Needs maintenance when sections change
- **Effort:** Small (15 min)
- **Risk:** Low

## Recommended Action

<!-- Fill during triage -->

## Technical Details

**Affected files:**
- `.claude/research/model-optimization/research-findings.md`

**Insert location:** After line 6 (after Purpose)

**Example TOC:**
```markdown
## Table of Contents
1. [Cerebras Inference Benchmarks](#1-cerebras-inference-benchmarks)
2. [Local LLM (Mac Studio M2 Ultra)](#2-local-llm-mac-studio-m2-ultra-128gb)
3. [Gemini Free Tier](#3-gemini-free-tier)
4. [Model Routing Strategies](#4-model-routing-strategies)
5. [Prompt Optimization Techniques](#5-prompt-optimization-techniques)
6. [Recommended Model Stack](#6-recommended-model-stack-for-moltbot-pro-subscription)
7. [Actionable Optimization Techniques](#7-actionable-optimization-techniques-priority-order)
8. [Quick Wins for Moltbot](#8-quick-wins-for-moltbot)
```

## Acceptance Criteria

- [x] TOC added after document header
- [x] All section links work (test in preview)
- [x] Follows Mintlify anchor format (no em-dashes, no apostrophes)

## Work Log

| Date | Action | Outcome |
|------|--------|---------|
| 2026-01-30 | Review identified navigation issue | Created todo |

## Resources

- Research document: `.claude/research/model-optimization/research-findings.md`
