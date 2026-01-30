---
module: Gateway
date: 2026-01-30
problem_type: workflow_issue
component: tooling
symptoms:
  - "Knowledge compounding only happens when manually triggered"
  - "Easy to forget /compound-docs after solving problems"
  - "Institutional knowledge gaps when documentation is skipped"
root_cause: missing_tooling
resolution_type: tooling_addition
severity: medium
tags: [hooks, compound-docs, automation, knowledge-management]
---

# Enforce Knowledge Compounding at Commit Time

## Problem Statement

The compound-docs skill effectively captures solved problems, but relies on manual invocation (`/compound-docs` or detection of "that worked" phrases). This leads to:
- Inconsistent documentation coverage
- Knowledge gaps when developers forget to compound
- Lost institutional knowledge from solved problems

## Proposed Solution: PreToolUse Hook for Git Commits

Add a Claude Code hook that triggers before `git commit` commands to:
1. Detect if significant problem-solving occurred in the session
2. Prompt user to document the solution before committing
3. Optionally block commit until documentation is complete

### Implementation Approach

**Option 1: Soft Reminder (Recommended)**

Create a `PreToolUse` hook that:
- Fires when Bash tool is invoked with `git commit`
- Checks if session contains problem-solving patterns (error messages, debugging, fixes)
- Injects a reminder prompt if undocumented solutions detected

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": {
          "tool_name": "Bash",
          "input_contains": "git commit"
        },
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/compound-reminder.py",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

**Option 2: Hard Gate**

Block commits until compound-docs is invoked:
- Maintain session state tracking documented vs undocumented fixes
- Return `block: true` from hook if undocumented solutions exist
- Provide clear message: "Run /compound-docs first to capture this solution"

### Detection Heuristics

The hook should detect problem-solving sessions by looking for:
- Error messages in conversation (`error:`, `failed`, `exception`)
- Debugging patterns (`tried`, `didn't work`, `fixed by`)
- Multiple investigation attempts on same topic
- Code changes after error investigation

### Integration with compound-docs Skill

The hook should:
1. Set `COMPOUND_DOCS_PENDING=true` environment variable when problem-solving detected
2. Clear flag after `/compound-docs` is successfully run
3. Check flag before allowing commit

## Acceptance Criteria

- [ ] Hook fires before `git commit` commands
- [ ] Detection heuristics identify problem-solving sessions
- [ ] User receives clear prompt to document before committing
- [ ] Hook respects user override (don't block indefinitely)
- [ ] Integration with existing hookify plugin infrastructure

## Related

- Hookify plugin: `.claude/plugins/marketplaces/claude-code-plugins/plugins/hookify/`
- compound-docs skill: `~/.claude/skills/compound-docs/`
- Hook development skill: `.claude/skills/hook-development/`

## Implementation Status

**Status:** Proposal (not yet implemented)

This document captures the idea for future implementation. To proceed:
1. Create the hook script in compound-docs skill
2. Register hook in hooks.json
3. Test with various commit scenarios
4. Document in compound-docs SKILL.md
