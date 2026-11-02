---
description: Delegate coding tasks to the Claude Code CLI while the managing agent scopes the work, reviews changes, runs validation, and sends corrective follow-ups; use when asked to have Claude write code or manage development through Claude CLI
argument-hint: "<task description> [--model <id>] [--plan]"
allowed-tools: [Bash, Read, Grep, Glob]
---

# Manage development through Claude Code CLI

Use the installed skill's `SKILL.md` for the full workflow and CLI flags.

1. Check CLI/authentication and record existing changes; read project instructions.
2. Write a self-contained brief with scope, acceptance criteria, and constraints.
3. Run `claude -p` with read/edit tools, MCP exclusion, and JSON output; use read-only tools for a plan.
4. Inspect results, review actual diffs, and run project validation yourself.
5. Send corrections through the exact session ID with `--resume`, preserving tool restrictions.
6. Report changes, validation, and limitations; preserve unrelated work and do not publish without authorization.

## Arguments

$ARGUMENTS
