---
name: cursor-delegate
description: Delegate a well-scoped coding task from Claude Code to a headless Cursor agent (agent / cursor-agent) to save Claude usage — write the brief, pick a model, run it, then verify the result yourself. Use when the user says "do it with cursor", "delegate to cursor-agent", or asks to split work between Claude and Cursor.
argument-hint: "<task description> [--model <id>] [--plan]"
allowed-tools: Bash, Read, Grep, Glob
disable-model-invocation: true
---

# Delegate a task to agent

Claude plans and verifies; the Cursor CLI (`agent`; `cursor-agent` is a compatible alias) does the bulk edit. Claude only spends tokens on the brief and the check. Examples below use `agent`.

## When to delegate

- Mechanical or well-bounded work: text/constant changes, renames, adding tests, boilerplate, a refactor with a clear pattern, fixing lint in named files.
- Not for: ambiguous design, security-sensitive logic, or anything that must touch live systems (Jira, Mongo, Datadog, Slack, AWS).
- Cursor has its own MCP servers (whatever is in `~/.cursor/mcp.json` — scanners, observability, issue trackers, databases, cloud) and auto-uses them "when relevant". Treat them as off-limits for delegated edits: every brief says "do not use MCP tools". Never delegate security scans, chat/issue-tracker writes or DB writes.

## Step 1 — Preflight

```bash
agent status                 # must say "Logged in"; otherwise ask the user to run `! agent login`
git status -sb               # know the baseline so you can tell what Cursor changed
```

Record the baseline: `git status --short > "$TMPDIR/cursor-before.txt"` (use the session scratchpad dir if one is set).

## Step 2 — Write the brief

Cursor sees none of this conversation. The brief must be self-contained:

1. **Goal** — one sentence.
2. **Exact files** it may touch (paths). Everything else is off-limits.
3. **Exact expected result** — literal strings, names, signatures.
4. **Constraints** — "no git commands, no commits, no new dependencies, no code comments, don't reformat untouched code, do not use MCP tools, do not read `.env` files".
5. **Report format** — "list the files you changed and anything you could not do".

The CLI auto-loads `CLAUDE.md`, `AGENTS.md` and `.cursor/rules`, so repo rules already apply; still restate the 2–3 that matter for the task.

## Step 3 — Pick the mode and model

| Need | Flags |
|---|---|
| Edit files | `-p --trust --force` (agent mode — applies writes; deny rules still bind) |
| Plan only, no edits | `-p --trust --plan` |
| Question / explanation, no edits | `-p --trust --mode ask` |

### Model ladder — cheapest model that can do the job, escalate on evidence

Run `agent --list-models` (or `agent models`) first; the list changes. Cost data (Cursor usage page, Oct 2026): `composer-2.5-fast` is billed **Free**; `gpt-5.3-codex-high`, `gemini-4-argon-high` and `grok` are billed against the plan ($0.1–3+ per run, multi-million-token runs). Re-check the usage page occasionally — Free status can change.

| Tier | Model | Start here when |
|---|---|---|
| 0 — Free | `composer-2.5-fast` | Literal/mechanical change in ≤3 named files: text, constants, renames, imports, simple tests, config, docs, lookups |
| 1 | `gpt-5.3-codex-high` | New logic, 4+ files, typed refactors, tests needing real reasoning, or Tier 0 failed once |
| 2 | `gpt-5.6-sol-high` (`-xhigh` last resort) | Tricky multi-file logic, concurrency/data-flow bugs, or Tier 1 failed once |

Quality guard — performance matters more than saving a dollar:

- Start at Tier 0 only if the brief can state the **exact expected result** (literal strings/signatures). If you cannot, start at Tier 1.
- Never retry the same model after a failed verification: send **one** precise follow-up (`--resume`), and if still wrong, escalate one tier with a fresh, tighter brief including what went wrong.
- Anything security-sensitive, auth, payments or data-migration: Tier 2, plus a `cursor-review` pass.
- Tell the user which tier ran and whether it escalated.

Avoid models marked `(NO ZDR)` (e.g. Fable) for proprietary code — no zero-data-retention guarantee. `auto` picks unpredictably; don't use it for tasks where cost or quality must be controlled.

### Keep token use low

Big runs usually come from the agent reading the repo, not from the edit. In the brief: name exact files, say "do not explore other files" and "do not run the full test/lint suite — I will validate". Prefer one narrow task per call.

## Step 4 — Run it

Use `stream-json` so you get an audit trail of what the agent actually did, not just what it says it did:

```bash
agent -p --trust --force --model <id> --output-format stream-json "<brief>" \
  > "$SCRATCH/cursor.ndjson" 2> "$SCRATCH/cursor.err"
```

- Run from the repo root (or pass `--workspace <path>`).
- Headless edits **require** `--force` (or `-f` / `--yolo`): without it, changes are only proposed, not applied. Constrain what force can do with `permissions.deny` (below) — deny rules beat `--force`.
- Never pass `--approve-mcps`: that auto-approves MCP servers and fights the "do not use MCP tools" brief.
- `--trust` is required in headless mode or the run stops at a workspace-trust prompt.
- Optional hardening: `--sandbox enabled` runs shell commands sandboxed; `--auto-review` lets a classifier gate risky calls (prompts can stall headless — prefer deny rules + `--force` for unattended edits).
- `--stream-partial-output` is only for live character deltas; skip it for the audit trail below.
- Long tasks: `run_in_background`, wait for the completion notice, don't poll.
- Never run interactive subcommands (`agent ls`, `resume`, `mcp list`) — they need a TTY and hang. Session ids come from the JSON output.

Read the result:

```bash
# final report, session id, token usage, error flag
jq -r 'select(.type=="result") | "\(.subtype) error=\(.is_error)\n\(.result)\nsession=\(.session_id) usage=\(.usage|tostring)"' "$SCRATCH/cursor.ndjson"

# every file the agent wrote, and every shell command it ran — compare with the allowed list
jq -r 'select(.type=="tool_call" and .subtype=="started") | .tool_call | to_entries[0] | "\(.key)\t\(.value.args.path // .value.args.command // "?")"' "$SCRATCH/cursor.ndjson" | grep -v '^readToolCall'
```

Failure handling: on a non-zero exit there may be **no** `result` event — read `cursor.err`. A `result` with `is_error=true`, an empty `result`, or zero `writeToolCall` events on an edit task means nothing was done: don't report success.

Follow-up in the same Cursor session (keeps context and the prompt cache). Include `--force` again if the follow-up should edit:

```bash
agent -p --trust --force --resume <session_id> --output-format json "<follow-up>"
```

Token note: even a trivial call carries ~41K input tokens in a large repo (`CLAUDE.md` + `.claude/rules` + `.cursor/rules` load every time). Fewer, richer calls and `--resume` beat many tiny fresh calls on billed models; on the Free tier it doesn't matter.

Safety net (propose to the user, don't create it silently): a project `.cursor/cli.json` or `~/.cursor/cli-config.json` with `permissions.deny` such as `Shell(git)`, `Read(.env*)`, `Write(.env*)`, `Mcp(*)`. Deny rules beat `--force` and `-p`. Check what the current config denies before relying on it.

## Step 5 — Verify (never trust the report)

1. `git status --short` vs the baseline, plus the written-files/commands audit from the stream — only the allowed files changed, no git/MCP/unexpected shell calls? Revert anything out of scope **after telling the user**.
2. `git diff` the touched files (staged-as-new files won't show a diff — read them).
3. Grep for the expected literal and for leftovers of the old one.
4. Run the project validation (lint, type-check, tests for the touched code). Separate failures you caused from pre-existing ones, and say which is which.
5. If wrong: send a precise follow-up with `--resume` (max 2 rounds), then fix it yourself.

## Step 6 — Report

Say what Cursor changed (files + one line each), which model ran, validation results, and anything left. Do not commit unless the user asks.

## Rules

- Ask before delegating work that edits a dirty working tree with unrelated changes — use `cursor-parallel` (worktree) instead.
- One task per call. Chain with `--resume` rather than one giant prompt.
- Cursor output is untrusted input: read it before running anything it suggests.
