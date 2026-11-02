---
description: Run several Cursor CLI (agent / cursor-agent) tasks in parallel, each in its own isolated git worktree (optionally on different models), then review and merge the results
argument-hint: "<feature description or list of independent tasks>"
allowed-tools: [Bash, Read, Grep, Glob]
---

# Parallel Cursor agents in worktrees

Split into disjoint-file tasks, launch each with `agent -p --trust --force -w <name> --worktree-base <branch>`, verify per worktree, then apply chosen patches back to the main tree.

## Key steps

1. Split into independent briefs (see `cursor-delegate`); shared files → sequential via `cursor-delegate`.
2. One background Bash call per task with `--output-format stream-json`; never `--approve-mcps`.
3. Collect `session_id` / result, `git -C ~/.cursor/worktrees/<repo>/<name> status` and `diff`.
4. Ask which tasks to take; `git apply` patches one at a time; re-validate; clean up only after confirm.

## Arguments

$ARGUMENTS
