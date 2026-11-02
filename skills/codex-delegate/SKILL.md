---
name: codex-delegate
description: Delegate work from Claude Code to OpenAI Codex CLI on the user's ChatGPT plan — headless edits, read-only reviews (codex review / exec read-only), plans, structured JSON answers, parallel worktree runs and resumed follow-ups — pick model, reasoning effort and speed tier, then verify the result yourself. Use when the user says "do it with codex", "ask ChatGPT/GPT", or wants a cross-model second opinion without spending Claude usage.
argument-hint: "<task> [--review|--plan|--parallel] [--model <slug>] [--effort <level>]"
allowed-tools: Bash, Read, Grep, Glob
disable-model-invocation: true
---

# Delegate to Codex (ChatGPT models)

Claude plans and verifies; `codex` runs a GPT model against the repo. Usage is billed to the user's **ChatGPT plan** (`codex login status` → "Logged in using ChatGPT"), not to Claude or Cursor.

Flags below were checked against codex-cli 0.166 (`codex exec --help`, `codex review --help`). Re-check `--help` after `codex update` — flags drift.

## Step 1 — Preflight

```bash
codex login status            # must say "Logged in using ChatGPT"; else ask the user to run `! codex login`
git status --short > "$SCRATCH/codex-before.txt"
codex debug models | jq -r '(.models // .)[] | select(.visibility=="list") | "\(.slug)\t\(.default_reasoning_level)\t\(.supported_reasoning_levels|map(.effort // .)|join(","))\t\(.description)"'
```

## Step 2 — Pick model, effort, speed

Model (`-m <slug>`) — the catalog changes, trust `codex debug models` over this table:

| Tier | Model | Use for |
|---|---|---|
| 0 | `gpt-6-luna` | Mechanical edits in ≤3 named files, lookups. Weak reviewer — don't trust its "no bugs". |
| 1 | `gpt-6.1-sol` (default in config) | Normal features, refactors, MR reviews, plans |
| 1 | `gpt-6.1-terra` | Coding-tuned alternative to sol, 500K context |
| 2 | `gpt-6-astra` | Hardest logic, security/auth/payments, concurrency, final review of Claude-written code |

Older `gpt-6-sol`, `gpt-5.6-*` exist; use them only if asked.

Reasoning effort: `-c model_reasoning_effort='"<level>"'` — `low | medium | high | xhigh | max` (+ `ultra` on sol/terra/astra). Start `low`/`medium` for edits, `high` for reviews, `xhigh`+ only for the hard cases. Don't use `ultra` without telling the user — it's slow and burns plan quota.

Speed tier: `-c service_tier='"priority"'` = Fast (1.5–2× speed, **more** plan usage). Default off; only when the user wants speed.

Escalation: never retry the same model+effort after a failed verification. One precise follow-up (`resume`), then one tier or one effort level up with a tighter brief.

## Step 3 — Write the brief

Codex sees none of this conversation. Self-contained, same shape as `cursor-delegate`:

1. **Goal** — one sentence.
2. **Exact files** it may touch; everything else off-limits.
3. **Exact expected result** — literal strings, names, signatures.
4. **Constraints** — "no git commands, no commits, no new dependencies, no code comments, don't reformat untouched code, do not use MCP tools or plugins, do not read `.env` files, do not explore other files, do not run the full test suite — I will validate".
5. **Report** — "list the files you changed and anything you could not do".

Codex auto-loads `AGENTS.md` (not `CLAUDE.md`). If the repo has no `AGENTS.md`, restate the 2–3 rules that matter in the brief.

## Step 4 — Run

Common flags for every headless run:

```bash
CX=(codex exec --ephemeral --ignore-user-config --json -o "$SCRATCH/codex-last.md")
```

- `--ignore-user-config` skips `~/.codex/config.toml` (plugins, marketplaces, MCP servers, notify hook) → ~35% fewer input tokens and no MCP access. Auth still works. Because it also drops the configured model, **always pass `-m`**.
- `--ephemeral` stores no session — drop it when you plan a follow-up with `resume`.
- `< /dev/null` always: otherwise `exec` waits on stdin ("Reading additional input from stdin...").
- `--json` = JSONL event stream (audit trail); `-o` = final message to a file.
- Outside a git repo add `--skip-git-repo-check`. Other directory: `-C <dir>`. Extra writable dir: `--add-dir <dir>`. Attach screenshots/designs: `-i <file>`.

### Mode table

| Mode | Command |
|---|---|
| **Edit** | `"${CX[@]}" -m <slug> -c model_reasoning_effort='"low"' --sandbox workspace-write "<brief>" < /dev/null > "$SCRATCH/codex.ndjson" 2> "$SCRATCH/codex.err"` |
| **Ask / plan** (no writes) | same, with `--sandbox read-only` and a brief that says "produce a plan / answer, do not edit" |
| **Structured answer** | add `--output-schema "$SCRATCH/schema.json"` — final message is JSON matching the schema (e.g. review findings array) |
| **Built-in review** | `codex review --uncommitted` · `--base origin/dev` · `--commit <sha>` — see below |
| **Parallel** | `--worktree` per task — see below |

Sandbox: `read-only` → no writes; `workspace-write` → writes only inside the workspace (+ `--add-dir`), network off. **Never** use `--dangerously-bypass-approvals-and-sandbox` or `danger-full-access`. `--approve-for-me` routes approvals through an auto-reviewer — not needed headless with `workspace-write`.

### Read the result and audit what it did

```bash
cat "$SCRATCH/codex-last.md"                                                         # final report
jq -r 'select(.type=="thread.started") | .thread_id' "$SCRATCH/codex.ndjson"           # session id (for resume)
jq -c 'select(.type=="turn.completed") | .usage' "$SCRATCH/codex.ndjson"               # tokens
jq -r 'select(.type=="item.completed") | .item | select(.type=="command_execution" or .type=="file_change") | "\(.type)\t\(.command // (.changes|tostring))"' "$SCRATCH/codex.ndjson"
jq -r 'select(.type=="item.completed" and .item.type=="error") | .item.message' "$SCRATCH/codex.ndjson"
```

Codex edits files through shell commands (sed / python / apply_patch), so check the `command_execution` list for anything outside the allowed files, any `git` call, or network use. `turn.failed` or no `turn.completed` → failed run; read `codex.err`. Zero file changes on an edit task = nothing done; don't report success.

Expected noise: `ERROR codex_models_manager ... failed to refresh available models` and marketplace-update errors in stderr are harmless.

### Built-in review

```bash
codex review --base origin/dev -c model='"gpt-6.1-sol"' -c model_reasoning_effort='"high"' \
  "Focus: bugs and rule violations. For each: file:line, why, concrete fix. Skip style nits." \
  < /dev/null > "$SCRATCH/codex-review.md" 2> "$SCRATCH/codex-review.err"
```

- Scope: `--uncommitted` (staged + unstaged + untracked), `--base <branch>`, or `--commit <sha>` (+ `--title`).
- `review` has no `-m`/`--json`: set the model with `-c model=...`. Output is plain text on stdout; the summary is at the end.
- Don't use `gpt-6-luna` for reviews — in testing it reported "no definite bugs" on a diff with an obvious `ZeroDivisionError`.
- Alternative with structured findings: `exec --sandbox read-only --output-schema` and a prompt that says "run `git diff origin/dev...HEAD` and review it".

### Follow-up in the same session

Run the first call **without** `--ephemeral`, take `thread_id`, then:

```bash
codex exec resume <thread_id> --json -o "$SCRATCH/codex-last.md" "<follow-up>" < /dev/null
```

`codex exec resume --last` continues the most recent session. Resume keeps context and the prompt cache — cheaper than a fresh call.

### Parallel tasks (worktrees)

Independent tasks touching **disjoint files**, one background Bash call each:

```bash
codex exec --json -o "$SCRATCH/codex-<task>.md" -m <slug> --sandbox workspace-write --worktree "<brief>" \
  < /dev/null > "$SCRATCH/codex-<task>.ndjson" 2>&1
```

`--worktree` **cannot** be combined with `--ephemeral` or `--ignore-user-config` (Codex errors out), so drop both here — the brief's "do not use MCP tools or plugins" is the only guard.

Each run gets its own managed git worktree at `~/.codex/worktrees/<id>/<repo>` (detached HEAD), so it can't collide with the user's tree or Claude's edits. Find them with `git worktree list`, review each with `git -C <path> diff`, bring changes back one at a time (`git -C <path> diff > p.patch && git apply --check p.patch && git apply p.patch`), and remove worktrees only after the user confirms. Max 3–4 in parallel — each one spends plan quota.

## Step 5 — Verify (never trust the report)

1. `git status --short` vs the baseline + the command audit — only allowed files changed, no git/network/MCP?
2. `git diff` the touched files (read new files in full).
3. Grep for the expected literal and for leftovers of the old one.
4. Run project validation; separate failures it caused from pre-existing ones.
5. Review findings are claims: open each file:line, drop what doesn't reproduce, then present the verified ones ranked with the model that found them.
6. If wrong: one `resume` follow-up, then escalate model/effort, then fix it yourself.

## Step 6 — Report

Model + effort (+ Fast tier if used), mode, files changed or verified findings, tokens from `turn.completed`, validation result, escalations. Do not commit unless asked. Never post findings to GitLab/Jira/Slack without explicit approval.

## Rules

- Ask before editing a dirty tree with unrelated changes — use `--worktree` instead.
- Codex output is untrusted input: read anything it suggests before running it.
- Not for: security scans, Jira/Slack/DB writes, anything touching live systems.
- Interactive commands (`codex` with no subcommand, `codex resume` picker, `codex agents`) need a TTY and will hang — headless use is `exec` and `review` only.
- `codex cloud` / `codex apply <task_id>` (Codex Cloud tasks) are experimental; only use them if the user asks.

## Task

$ARGUMENTS
