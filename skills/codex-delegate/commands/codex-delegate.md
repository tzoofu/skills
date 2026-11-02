---
description: Delegate work from Claude Code to OpenAI Codex CLI on the user's ChatGPT plan — headless edits, read-only reviews (codex review / exec read-only), plans, structured JSON answers, parallel worktree runs and resumed follow-ups — pick model, reasoning effort and speed tier, then verify the result yourself
argument-hint: "<task> [--review|--plan|--parallel] [--model <slug>] [--effort <level>]"
allowed-tools: [Bash, Read, Grep, Glob]
---

# Delegate to Codex (ChatGPT models)

Claude plans and verifies; `codex` runs a GPT model against the repo, billed to the ChatGPT plan.

## Key steps

1. Preflight: `codex login status` (ChatGPT), record `git status --short` baseline, list models with `codex debug models`.
2. Pick model (`gpt-6-luna` mechanical → `gpt-6.1-sol`/`gpt-6.1-terra` normal → `gpt-6-astra` hard), effort (`-c model_reasoning_effort='"low|medium|high|xhigh|max|ultra"'`), optional Fast tier (`-c service_tier='"priority"'`).
3. Write a self-contained brief (goal, exact files, literal result, no git/MCP/`.env`, restate rules — Codex reads `AGENTS.md`, not `CLAUDE.md`).
4. Run `codex exec --ephemeral --ignore-user-config --json -o <file> -m <slug> --sandbox workspace-write|read-only "<brief>" < /dev/null`. Review: `codex review --uncommitted|--base <branch>|--commit <sha> -c model=...`. Structured: `--output-schema`. Parallel: `--worktree`. Follow-up: `codex exec resume <thread_id>`.
5. Audit `command_execution` events with `jq`, verify vs baseline (`git diff`, greps, validation). One follow-up, then escalate.
6. Report model/effort, files or verified findings, tokens, validation; do not commit unless asked.

## Arguments

$ARGUMENTS
