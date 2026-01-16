# Parallel Subagent-Driven Development

## Overview

Extends subagent-driven-development to run independent tasks concurrently, each in its own git worktree. Maintains the two-stage review process (spec compliance, then code quality) while parallelizing where possible.

**Core principle:** Dependency-aware scheduling + worktree isolation + per-task review pipeline = parallel execution without conflicts.

## Design Decisions

| Decision | Choice |
|----------|--------|
| Dependency handling | Analyze plan upfront, only parallelize independent tasks |
| Review timing | Immediately when each task completes (while others continue) |
| Merge target | Directly to main/target branch |
| Rebase timing | Before merge (after review passes) |
| Conflict handling | Resolver subagent fixes, re-run quality review |
| Worktree cleanup | Immediately after successful merge |

## The Process

### Phase 1: Plan Analysis

Before dispatching any implementers, the controller:

1. Reads the plan and extracts all tasks
2. Analyzes dependencies between tasks:
   - Explicit: "This task requires Task 2's API"
   - Implicit: Two tasks modify the same file
   - Inferred: Task B uses a function Task A creates
3. Builds a dependency graph
4. Identifies **parallel groups** - sets of tasks with no dependencies on each other

```
Example plan with 5 tasks:
- Task 1: Add user model (independent)
- Task 2: Add auth middleware (depends on Task 1)
- Task 3: Add logging utility (independent)
- Task 4: Add rate limiter (independent)
- Task 5: Add auth routes (depends on Task 1, Task 2)

Parallel groups:
  Group A: [Task 1, Task 3, Task 4]  <- run in parallel
  Group B: [Task 2]                   <- after Task 1
  Group C: [Task 5]                   <- after Task 2
```

The controller creates a TodoWrite with all tasks, annotated with their group.

### Phase 2: Worktree Setup & Dispatch

For each parallel group, the controller:

1. Creates one worktree per task in the group:
   ```bash
   git worktree add .worktrees/task-1-user-model -b task-1-user-model
   git worktree add .worktrees/task-3-logging -b task-3-logging
   git worktree add .worktrees/task-4-rate-limiter -b task-4-rate-limiter
   ```

2. Dispatches implementer subagents **in parallel** (single message, multiple Task tool calls):
   ```
   Task("Implement Task 1: Add user model", work_dir=".worktrees/task-1-user-model")
   Task("Implement Task 3: Add logging utility", work_dir=".worktrees/task-3-logging")
   Task("Implement Task 4: Add rate limiter", work_dir=".worktrees/task-4-rate-limiter")
   ```

3. Each implementer:
   - Works in its isolated worktree
   - Follows TDD, implements, tests, commits
   - Self-reviews and reports back

**Key constraint:** Implementers in the same parallel group must not have overlapping file changes. The dependency analysis should catch this, but if a conflict is detected during worktree creation (same files in scope), the controller falls back to sequential execution for those tasks.

### Phase 3: Per-Task Review Pipeline

As implementers finish (in any order), the controller immediately starts that task's review pipeline:

```
Task 3 finishes first:
  -> Dispatch spec reviewer for Task 3
  -> Spec passes -> Dispatch code quality reviewer for Task 3
  -> Quality passes -> Task 3 ready to merge

(Meanwhile, Tasks 1 and 4 still running)

Task 1 finishes:
  -> Dispatch spec reviewer for Task 1
  -> ...
```

**Review loops still apply:** If spec reviewer finds issues, implementer fixes them (same worktree), then re-review. Same for code quality.

**Parallelism in reviews:** Reviews for different tasks can run in parallel too. Task 3's code quality review can happen while Task 1's spec review is running.

```
Timeline:
  Task 3: [implement] [spec-review] [quality-review] [merge] done
  Task 1: [implement.....] [spec-review] [fix] [spec-review] [quality-review] [merge] done
  Task 4: [implement...........] [spec-review] [quality-review] [merge] done
```

### Phase 4: Merge & Cleanup

When a task passes both reviews, it's ready to merge:

1. **Rebase onto latest main:**
   ```bash
   cd .worktrees/task-3-logging
   git fetch origin main
   git rebase origin/main
   ```

2. **If rebase succeeds (no conflicts):**
   ```bash
   git checkout main
   git merge task-3-logging --ff-only
   git push origin main
   git worktree remove .worktrees/task-3-logging
   git branch -d task-3-logging
   ```

3. **If rebase has conflicts:**
   - Dispatch resolver subagent with conflict details
   - Resolver fixes conflicts, commits
   - Re-run code quality review (spec still valid)
   - Then merge as above

**Cleanup is immediate:** Worktree deleted right after successful merge. No lingering branches.

### Phase 5: Next Parallel Group

Once all tasks in a parallel group have merged, the controller moves to the next group:

```
Group A complete (Tasks 1, 3, 4 merged to main)
  |
  v
Group B starts: Task 2 (depends on Task 1)
  - Create worktree from latest main (now includes Task 1)
  - Dispatch implementer
  - Review pipeline
  - Merge to main
  |
  v
Group C starts: Task 5 (depends on Tasks 1, 2)
  - Create worktree from latest main (now includes Tasks 1, 2)
  - ...
```

**Why sequential between groups:** Tasks in Group B depend on Group A's output. They need that code in main before they can start.

### Phase 6: Final Review & Finish

After all groups complete:
1. Dispatch final code reviewer for entire implementation
2. Use `finishing-a-development-branch` skill to wrap up

## Complete Flow Diagram

```
+-------------------------------------------------------------+
| 1. Analyze plan -> build dependency graph -> identify groups |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| 2. For each parallel group:                                 |
|    a. Create worktrees for all tasks in group               |
|    b. Dispatch implementers in parallel                     |
|    c. As each finishes -> spec review -> quality review     |
|    d. When reviews pass -> rebase -> merge to main -> cleanup|
|    e. Wait for all tasks in group to merge                  |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| 3. Final code review -> finishing-a-development-branch      |
+-------------------------------------------------------------+
```

## Error Handling

| Scenario | Action |
|----------|--------|
| Implementer fails | Dispatch fix subagent in same worktree |
| Review fails | Implementer fixes, re-review |
| Rebase conflicts | Resolver subagent fixes, re-run quality review |
| Unrecoverable error | Stop, report to user, preserve worktree for debugging |

## Comparison with Existing Skills

| Aspect | subagent-driven-development | parallel-subagent-driven-development |
|--------|----------------------------|-------------------------------------|
| Execution | Sequential (one task at a time) | Parallel within groups |
| Isolation | Single working directory | One worktree per task |
| Review | After each task | After each task (in parallel) |
| Merge | Single branch | Direct to main per task |
| Speed | Slower | Faster for independent tasks |

## Integration

**Required skills:**
- `writing-plans` - Creates the plan this skill executes
- `using-git-worktrees` - Worktree creation patterns
- `requesting-code-review` - Review templates
- `finishing-a-development-branch` - Completion workflow

**Subagents use:**
- `test-driven-development` - Implementation approach

## When to Use

**Use parallel-subagent-driven-development when:**
- Plan has 3+ tasks
- Many tasks are independent (can parallelize)
- Want faster execution
- Comfortable with direct-to-main merges

**Use regular subagent-driven-development when:**
- Tasks are tightly coupled
- Want simpler debugging (single timeline)
- Plan has mostly sequential dependencies
