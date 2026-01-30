---
module: Gateway
date: 2026-01-30
problem_type: best_practice
component: documentation
symptoms:
  - "Research documents with unverified claims"
  - "Missing actionable integration guidance"
  - "Source bias toward single vendor"
root_cause: inadequate_documentation
resolution_type: workflow_improvement
severity: medium
tags: [research, review, documentation, quality]
---

# Research Document Review Pattern

## Problem Statement

Research documents created from web searches may contain:
- Outdated information (API rate limits change)
- Unverified claims (vendor marketing vs reality)
- Missing integration guidance (Python examples for TypeScript codebase)
- Source bias (too many sources from single vendor)

## Solution: Multi-Agent Review Pattern

Use parallel review agents to evaluate research documents across multiple dimensions.

### Review Dimensions

| Agent | Focus | Key Questions |
|-------|-------|---------------|
| **Structure Reviewer** | Organization | Is there a TOC? Are sections logical? |
| **Technical Accuracy** | Correctness | Do benchmarks match verified sources? |
| **Actionability** | Usability | Can developers act on recommendations? |
| **Missing Techniques** | Completeness | What's not covered? |
| **Source Quality** | Credibility | Diverse sources? Recent? Authoritative? |

### Findings Categories

**P1 (Critical)** - Blocks usage:
- Incorrect technical specifications
- Outdated rate limits/pricing
- Unverified claims stated as fact

**P2 (Important)** - Should fix:
- Missing navigation (TOC)
- No integration path for codebase
- Source bias (>50% single vendor)

**P3 (Nice-to-have)** - Polish:
- Code block language labels
- Inline citations
- Cross-section references

### Output: File-Based Todos

Create todo files in `todos/` directory:

```
todos/
├── 001-pending-p1-{finding}.md
├── 002-pending-p2-{finding}.md
└── 003-pending-p3-{finding}.md
```

### Example Findings from Today

| Finding | Severity | Issue |
|---------|----------|-------|
| Gemini rate limits outdated | P1→P3 | User has paid tier (not free) |
| Qwen3 context lengths incorrect | P1 | Stated 65K, actual 32K native |
| Missing TOC | P2 | 500-line doc needs navigation |
| No TypeScript examples | P2 | All code is Python |
| Source bias | P2 | 11/20 sources are Cerebras |

## Key Learning: Context Matters

**Before:** Flagged Gemini rate limits as P1 (critical)
**After:** User clarified they have paid account → downgraded to P3

**Lesson:** Always confirm user's actual subscription/environment before prioritizing accuracy issues.

## Prevention / Best Practice

When reviewing research documents:

1. **Clarify user context first** (subscriptions, environment)
2. **Run 5 parallel review agents** (structure, accuracy, actionability, completeness, sources)
3. **Create todos immediately** (don't ask for approval on each finding)
4. **Adjust priorities** based on user feedback
5. **Focus on actionability** - Can the team actually use this?

## Related

- `todos/` - File-based todo tracking
- `.claude/research/model-optimization/research-findings.md` - Reviewed document
- `/workflows:review` - Full review workflow
