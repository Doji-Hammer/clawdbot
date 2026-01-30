---
status: complete
priority: p2
issue_id: "005"
tags: [code-review, actionability, integration]
dependencies: []
---

# No Moltbot Integration Path

## Problem Statement

The research document provides Python code examples but no TypeScript implementation guidance for Moltbot integration.

**What's broken:**
- All code examples are Python (RouteLLM, Cascade Routing, Cerebras SDK)
- No references to Moltbot codebase entry points
- Developer can't act without reverse-engineering the codebase

**Why it matters:** "Immediate Impact" recommendations can't be implemented without knowing where to change code.

## Findings

**Missing integration points:**
- Where model selection happens (`src/agents/pi-embedded-runner/`)
- How to configure Cerebras as default
- Where system prompts are constructed (for prefix caching)
- How to implement routing in TypeScript

**Location:** `.claude/research/model-optimization/research-findings.md` - all code examples

## Proposed Solutions

### Option 1: Add Moltbot Code Integration Map (Recommended)
- **Pros:** Directly actionable
- **Cons:** Requires codebase exploration
- **Effort:** Medium (1-2 hours to map code paths)
- **Risk:** Low

**Example addition:**
```markdown
## Moltbot Integration Points

| Recommendation | File | Function/Config |
|----------------|------|-----------------|
| Switch to Llama 3.3 70B | `src/config/` | Model default config |
| Enable prefix caching | Pi agent prompt builder | Cache control headers |
| Implement router | NEW: `src/routing/model-router.ts` | - |
```

### Option 2: Create Separate Implementation Guide
- **Pros:** Keeps research doc focused
- **Cons:** Another document to maintain
- **Effort:** Medium (1-2 hours)
- **Risk:** Low

## Recommended Action

<!-- Fill during triage -->

## Technical Details

**Affected files:**
- `.claude/research/model-optimization/research-findings.md`

**Moltbot files to reference:**
- `src/agents/pi-embedded-runner/` - Agent runtime
- `src/gateway/` - Gateway service
- `src/config/` - Configuration loading
- `src/routing/` - Routing logic

## Acceptance Criteria

- [x] TypeScript examples added OR code paths documented
- [x] Implementation checklist with specific file:line references
- [x] "Switch to Cerebras" has exact config change documented

## Work Log

| Date | Action | Outcome |
|------|--------|---------|
| 2026-01-30 | Review identified actionability gap | Created todo |

## Resources

- Research document: `.claude/research/model-optimization/research-findings.md`
- Moltbot architecture: `.claude/CLAUDE.md`
