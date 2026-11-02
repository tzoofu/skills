---
name: cursor-parallel
description: Run several Cursor CLI (agent / cursor-agent) tasks in parallel, each in its own isolated git worktree (optionally on different models), then review and merge the results. Use when the user wants to build with Claude and Cursor at the same time, or split a feature into independent pieces.
argument-hint: "<feature description or list of independent tasks>"
allowed-tools: Bash, Read, Grep, Glob
disable-model-invocation: true
---

# Parallel Cursor agents in worktrees

Cursor can build in an isolated worktree (`-w`) so it never collides with the user's working tree or with Claude's own edits. Use the Cursor CLI as `agent` (`cursor-agent` is a compatible alias).

## Split the work first

- Tasks must touch **disjoint files**. If two tasks share a file, run them sequentially via `cursor-delegate`.
- Write one self-contained brief per task (goal, allowed files, expected result, constraints) — see `cursor-delegate` step 2.
- Claude may keep one slice for itself in the main tree; Cursor takes the rest.

## Launch

One background Bash call per task, so they run concurrently:

```bash
agent -p --trust --force -w <task-name> --worktree-base <branch> \
  --model <id> --output-format stream-json "<brief>"
```

- Worktrees land in `~/.cursor/worktrees/<repo>/<task-name>`. `--worktree-base` defaults to current HEAD; set it explicitly (`origin/dev` or the feature branch).
- Choose each task's model with the ladder in `cursor-delegate` (start at the cheapest tier that fits; `composer-2.5-fast` is Free, the rest are billed). Mixing is the point: Free tier for mechanical slices, `gpt-5.3-codex-high` or `gpt-5.6-sol-high` only for slices with real logic. Check `agent --list-models` (or `agent models`) first. Avoid `(NO ZDR)` models for proprietary code.
- Parallel runs multiply cost, so keep each brief narrow (exact files, "do not explore", "I will validate").
- Use `--skip-worktree-setup` only if the repo's `.cursor/worktrees.json` setup is slow and not needed.
- Headless edits **require** `--force` (or `-f` / `--yolo`); without it, changes are only proposed. Constrain with `permissions.deny` (`Shell(git)`, `Read(.env*)`, `Write(.env*)`, `Mcp(*)`) — deny beats `--force`. Never pass `--approve-mcps`.
- Wait for completion notices; do not poll. Never run interactive subcommands (`agent ls`, `mcp list`) — they hang without a TTY.
- Use `--output-format stream-json` and the audit `jq` from `cursor-delegate` per task, so each worktree's written files and shell commands can be checked. Every brief must say "do not use MCP tools, no git commands, do not read `.env`" — Cursor may have MCP servers configured that it calls on its own.
- New worktrees are missing untracked files (local `.claude/` rules, `node_modules`, `.env`) — copy what's needed or run `npm install` if the task needs validation there, or validate after merging.

## Collect and verify

For each finished task:

1. Note `session_id`, `result` and `usage` from the stream/JSON (`select(.type=="result")` for `stream-json`; `.result` for single-object `json`).
2. `git -C ~/.cursor/worktrees/<repo>/<task-name> status --short` and `git diff` — files stay within the allowed set?
3. Run validation in the worktree if it has dependencies; otherwise after bringing the changes over.

## Bring changes back

Ask the user which tasks to take. Then apply one at a time:

```bash
git -C ~/.cursor/worktrees/<repo>/<task-name> diff > "$TMPDIR/<task-name>.patch"
git apply --check "$TMPDIR/<task-name>.patch" && git apply "$TMPDIR/<task-name>.patch"
```

Re-run validation on the combined result. Do not commit unless asked. Clean up worktrees only after the user confirms (`git worktree remove <path>`).

## Rules

- Max 3–4 parallel agents — each one costs Cursor usage and review time.
- Report per task: model, files changed, validation result.
