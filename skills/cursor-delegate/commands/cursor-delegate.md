---
description: Delegate a well-scoped coding task from Claude Code to a headless Cursor agent (agent / cursor-agent) to save Claude usage — write the brief, pick a model, run it, then verify the result yourself
argument-hint: "<task description> [--model <id>] [--plan]"
allowed-tools: [Bash, Read, Grep, Glob]
---

# Delegate a task to agent

Claude plans and verifies; the Cursor CLI (`agent`; `cursor-agent` is a compatible alias) does the bulk edit.

## Key steps

1. Preflight: `agent status`, record `git status --short` baseline.
2. Write a self-contained brief (goal, allowed files, expected result, constraints including no MCP/git/`.env`).
3. Pick mode/model: edits use `-p --trust --force`; plan = `--plan`; ask = `--mode ask`. Ladder starts at `composer-2.5-fast`.
4. Run with `--output-format stream-json`, audit tool calls with `jq`, never `--approve-mcps`.
5. Verify vs baseline (`git diff`, greps, project validation). Follow up with `--resume` (include `--force` if still editing).
6. Report files changed, model, validation; do not commit unless asked.

## Arguments

$ARGUMENTS
