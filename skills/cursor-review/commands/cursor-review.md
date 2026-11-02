---
description: Get a read-only second opinion, plan, or code review from a different model family through the Cursor CLI (agent / cursor-agent) in ask/plan mode, no edits
argument-hint: "<what to review/plan> [--model <id>]"
allowed-tools: [Bash, Read, Grep, Glob]
---

# Read-only second opinion via agent

Run `agent -p --trust --mode ask` (or `--plan`) with a different model family. Do not pass `--force` or `--approve-mcps`.

## Key steps

1. Pick ask vs plan; pick model from the ladder (start `composer-2.5-fast` for small diffs).
2. Scope files/diff in the prompt; forbid MCP / `.env` / edits.
3. Parse with `jq -r '.result'` for `--output-format json`.
4. Verify every finding at file:line before presenting; never post to GitLab/Jira/Slack without approval.

## Arguments

$ARGUMENTS
