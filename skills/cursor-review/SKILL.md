---
name: cursor-review
description: Get a read-only second opinion, plan, or code review from a different model family through the Cursor CLI (agent / cursor-agent) in ask/plan mode, no edits. Use when the user wants a cross-model review of a diff, a plan sanity-check, or an investigation done without spending Claude usage.
argument-hint: "<what to review/plan> [--model <id>]"
allowed-tools: Bash, Read, Grep, Glob
disable-model-invocation: true
---

# Read-only second opinion via agent

Runs the Cursor CLI (`agent`; `cursor-agent` is a compatible alias) in `ask` or `plan` mode — it cannot edit files. Use it to review, plan, or investigate with a model that has different blind spots than Claude.

## Pick the mode

| Goal | Flag |
|---|---|
| Review a diff / explain code / find bugs | `--mode ask` |
| Produce an implementation plan | `--plan` |

## Pick the model (differ from the author)

Run `agent --list-models` (or `agent models`) first. `composer-2.5-fast` is billed Free; the others are billed against the plan (see `cursor-delegate` for the cost notes). Review quality depends on model strength, so don't go cheap on the review that matters:

| Review of | Model |
|---|---|
| Quick sanity pass, small diff (≤3 files), plan check | `composer-2.5-fast` (Free) |
| Normal MR / feature diff | `gpt-5.3-codex-high` |
| Claude-written code that needs a truly different perspective | `gpt-5.6-sol-high` or `gemini-4-argon-high` |
| Security-adjacent or high-risk diff | `gpt-5.6-sol-xhigh` |

If a Free-tier review returns nothing on a non-trivial diff, don't trust the silence — rerun one tier up. Name the files or diff range in the prompt and tell it not to explore beyond them (cuts token cost).

Avoid `(NO ZDR)` models for proprietary code.

## Run

Put the diff scope in the prompt rather than pasting code — Cursor can read the repo itself. Do **not** pass `--force` or `--approve-mcps` (ask/plan is read-only; MCP auto-approve fights the brief):

```bash
agent -p --trust --mode ask --model gpt-5.6-sol-high --output-format json \
  "Review the changes on this branch vs origin/dev (run git diff origin/dev...HEAD). Focus: <bugs | rule violations | missing tests>. Read CLAUDE.md and .claude/rules first. For each finding give file:line, why it is a real problem, and a concrete fix. Skip style nits. If you find nothing, say so."
```

Plan variant:

```bash
agent -p --trust --plan --model gpt-5.6-sol-high --output-format json "<goal + constraints + files involved>"
```

Add to every review prompt: "Do not use MCP tools, do not read `.env` files, do not edit anything." Ask/plan mode already blocks edits, but Cursor may have MCP servers configured that it calls on its own. Never use a Cursor review to run security scans. The CLI auto-loads `CLAUDE.md` and `.claude/rules`, so say "follow the repo rules" rather than pasting them.

Parse the answer with `jq -r '.result'` when using `--output-format json` (single object). For `stream-json` NDJSON use `jq -r 'select(.type=="result") | .result'`. On non-zero exit read stderr, and treat an empty result as a failed run, not as "no findings".

## After it returns

1. Treat every finding as a **claim to verify**: open the file:line yourself. Drop anything that does not reproduce.
2. Present only verified findings, ranked, with the model that found them.
3. Never post findings to GitLab/Jira/Slack without the user's explicit approval.
4. Fixing them is a separate step — hand to `cursor-delegate` or do it directly.
