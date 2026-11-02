---
description: Delegate a well-scoped edit or a read-only review from Claude Code to a local Ollama model (qwen, gemma, …) running headless Claude Code in --bare mode — free, offline, no data leaves the machine — then verify the result yourself
argument-hint: "<task description> [--review] [--model <ollama-model>]"
allowed-tools: [Bash, Read, Grep, Glob]
---

# Delegate a task to a local Ollama model

Claude plans and verifies; a local Ollama model does the edit or review through headless Claude Code (`ANTHROPIC_BASE_URL=http://localhost:11434`).

## Key steps

1. Preflight: Ollama up (`curl localhost:11434/api/version`), `ollama list`/`ollama ps`, record `git status --short` baseline.
2. Context fix: run with `--bare --strict-mcp-config --tools ...` (tiny prompt) and a 64K variant (`FROM qwen3.8:27b` + `PARAMETER num_ctx 65536` → `ollama create qwen3.8-64k`). Full model tag required.
3. Write a self-contained brief (goal, exact files, literal expected result, constraints + restated repo rules — `--bare` skips CLAUDE.md).
4. Edit: `--tools "Read,Edit,Write,Glob,Grep" --permission-mode acceptEdits --output-format stream-json --verbose`. Review: pipe the diff in, `--tools "Read,Glob,Grep" --output-format json`.
5. Audit tool calls with `jq`, verify vs baseline (`git diff`, greps, project validation). One retry, then escalate to `cursor-delegate` or fix directly.
6. Report model, files changed / verified findings, validation; do not commit unless asked.

## Arguments

$ARGUMENTS
