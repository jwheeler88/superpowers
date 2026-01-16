# Resolver Subagent Prompt Template

Use this template when dispatching a resolver subagent to fix rebase conflicts.

**Purpose:** Resolve merge conflicts that occur when rebasing a task branch onto main.

```
Task tool (general-purpose):
  description: "Resolve conflicts for Task N"
  prompt: |
    You are resolving merge conflicts from rebasing a task branch onto main.

    ## Context

    Task N: [task name] was implemented and passed both reviews.
    During rebase onto latest main, conflicts occurred.

    Work from: [worktree path, e.g., .worktrees/task-N-name]

    ## Conflict Details

    [Output from git rebase showing conflicts]

    Files with conflicts:
    [List of conflicting files]

    ## Your Job

    1. Understand what the task implemented (from commit messages/diff)
    2. Understand what changed in main (the other side of conflict)
    3. Resolve conflicts preserving both:
       - The task's new functionality
       - Main's updates from other merged tasks
    4. Complete the rebase
    5. Run tests to verify resolution works

    ## Resolution Guidelines

    - Preserve the intent of both changes
    - If both added to same location, include both (in logical order)
    - If both modified same code, merge the logic correctly
    - Never silently drop either side's changes

    ## Report Format

    When done, report:
    - What conflicts you resolved
    - How you merged the changes
    - Test results after resolution
    - Any concerns about the resolution
```
