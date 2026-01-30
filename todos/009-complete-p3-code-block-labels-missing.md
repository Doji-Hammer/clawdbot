---
status: complete
priority: p3
issue_id: "009"
tags: [code-review, documentation, polish]
dependencies: []
---

# Code Block Language Labels Missing

## Problem Statement

Some code blocks lack language labels (```python, ```bash, etc.), reducing readability.

**What's broken:** Inconsistent syntax highlighting in rendered markdown.

**Why it matters:** Minor UX issue but reduces professionalism of documentation.

## Findings

**Code blocks needing labels:**
- Lines 103-106: bash (optillm command)
- Lines 454-457: plaintext (prompt structure diagram)
- Python blocks are generally labeled

**Location:** `.claude/research/model-optimization/research-findings.md`

## Proposed Solutions

### Option 1: Add Language Labels (Recommended)
- **Pros:** Consistent syntax highlighting
- **Cons:** None
- **Effort:** Small (10 min)
- **Risk:** Low

## Recommended Action

<!-- Fill during triage -->

## Technical Details

**Affected files:**
- `.claude/research/model-optimization/research-findings.md`

## Acceptance Criteria

- [ ] All code blocks have language labels
- [ ] Consistent highlighting when rendered

## Work Log

| Date | Action | Outcome |
|------|--------|---------|
| 2026-01-30 | Review identified formatting issue | Created todo |

## Resources

- Research document: `.claude/research/model-optimization/research-findings.md`
