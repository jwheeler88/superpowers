---
name: parallel-subagent-driven-development
description: Use when executing implementation plans with many independent tasks - runs tasks concurrently in isolated worktrees
---

# Parallel Subagent-Driven Development

Execute plan by running independent tasks concurrently, each in its own git worktree. Maintains two-stage review process (spec compliance, then code quality) while parallelizing where possible.

**Core principle:** Dependency-aware scheduling + worktree isolation + per-task review pipeline = parallel execution without conflicts.

**Announce at start:** "I'm using parallel-subagent-driven-development to execute this plan with concurrent task execution."

## When to Use

```dot
digraph when_to_use {
    "Have implementation plan?" [shape=diamond];
    "3+ tasks?" [shape=diamond];
    "Many independent tasks?" [shape=diamond];
    "Want faster execution?" [shape=diamond];
    "parallel-subagent-driven-development" [shape=box];
    "subagent-driven-development" [shape=box];
    "Manual or brainstorm first" [shape=box];

    "Have implementation plan?" -> "3+ tasks?" [label="yes"];
    "Have implementation plan?" -> "Manual or brainstorm first" [label="no"];
    "3+ tasks?" -> "Many independent tasks?" [label="yes"];
    "3+ tasks?" -> "subagent-driven-development" [label="no - use sequential"];
    "Many independent tasks?" -> "Want faster execution?" [label="yes"];
    "Many independent tasks?" -> "subagent-driven-development" [label="no - tightly coupled"];
    "Want faster execution?" -> "parallel-subagent-driven-development" [label="yes"];
    "Want faster execution?" -> "subagent-driven-development" [label="no - simpler debugging"];
}
```

**Use parallel-subagent-driven-development when:**
- Plan has 3+ tasks
- Many tasks are independent (can parallelize)
- Want faster execution
- Comfortable with direct-to-main merges

**Use regular subagent-driven-development when:**
- Tasks are tightly coupled (most depend on each other)
- Want simpler debugging (single timeline)
- Plan has mostly sequential dependencies
