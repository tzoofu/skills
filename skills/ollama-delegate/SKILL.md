---
name: ollama-delegate
description: Delegate a well-scoped edit or a read-only review from Claude Code to a local Ollama model (qwen, gemma, …) running headless Claude Code in --bare mode — free, offline, no data leaves the machine — then verify the result yourself. Use when the user says "do it with ollama", "use qwen locally", or wants to save Claude/Cursor usage on mechanical work.
argument-hint: "<task description> [--review] [--model <ollama-model>]"
allowed-tools: Bash, Read, Grep, Glob
disable-model-invocation: true
---

# Delegate a task to a local Ollama model

Claude plans and verifies; a local model does the bulk edit or a second-opinion review. The harness is **Claude Code itself** pointed at Ollama's Anthropic-compatible API (`/v1/messages`), so the local model gets the same Read/Edit/Write/Glob/Grep tools and the same `stream-json` audit trail.

Cost is zero and nothing leaves the machine, but local models are slower and weaker than Cursor/Claude: use them for Tier-0 work only (see `cursor-delegate` for the ladder) and escalate on the first failed verification.

## When to delegate

- Mechanical, literal changes in ≤3 named files: text/constants, renames, imports, simple tests, config, docs.
- Read-only second opinion on a small diff (≤ a few hundred lines), or a plan sanity-check.
- Not for: ambiguous design, security/auth/payments logic, anything touching live systems, or large multi-file refactors — use `cursor-delegate` (Tier 1+) or do it directly.

## Why `--bare` (the context-window fix)

Ollama loads models with a 32K context by default (`ollama ps` → CONTEXT column). A normal Claude Code run sends its full system prompt + CLAUDE.md + rules + MCP tool schemas — that overflows 32K and the model silently loses the start of the prompt.

Two fixes, use both:

1. **Shrink the prompt** — `--bare` skips CLAUDE.md auto-discovery, hooks, plugins, memory and LSP; `--strict-mcp-config` loads no MCP servers; `--tools` exposes only the tools named. A measured edit run used ~2.7K tokens in total.
2. **Raise the window** — make a large-context variant once (shares the weights, no extra disk):

   ```bash
   printf 'FROM qwen3.8:27b\nPARAMETER num_ctx 65536\n' > "$SCRATCH/Modelfile" && ollama create qwen3.8-64k -f "$SCRATCH/Modelfile"
   ```

   Verify with `ollama ps` after a run: CONTEXT should read `65536` and PROCESSOR `100% GPU`. If PROCESSOR shows a CPU split the window is too big for RAM — drop to 49152. For 128K+, start the server with `OLLAMA_FLASH_ATTENTION=1 OLLAMA_KV_CACHE_TYPE=q8_0` (halves KV-cache memory). Server-wide alternative: `OLLAMA_CONTEXT_LENGTH=65536` (Ollama app: `launchctl setenv OLLAMA_CONTEXT_LENGTH 65536`, then restart Ollama).

Because `--bare` drops CLAUDE.md, the brief **must** restate the 2–3 repo rules that matter. To inject a short rules file, add `--append-system-prompt-file <file>` (keep it small).

## Step 1 — Preflight

```bash
curl -sf localhost:11434/api/version || echo "Ollama not running — ask the user to open Ollama"
ollama list                    # pick a model; full tag required (e.g. qwen3.8-64k:latest)
ollama ps                      # what's loaded and with what context
git status --short > "$SCRATCH/ollama-before.txt"
```

### Model choice

| Model | Use for |
|---|---|
| `qwen3.8-64k:latest` (qwen3.8:27b @ 64K) | Default — edits and reviews |
| `gemma4:26b` (make a 64K variant the same way) | Review from a different family than Qwen |
| `qwen2.5-coder:7b` | Trivial one-file edits when speed matters; weak at multi-step tool use |

Always pass the **full tag** (`name:tag`). Only one 27B model fits in memory comfortably — run delegated tasks sequentially, never in parallel.

## Step 2 — Write the brief

The local model sees none of this conversation. Same shape as `cursor-delegate`:

1. **Goal** — one sentence.
2. **Exact files** it may touch. Everything else is off-limits.
3. **Exact expected result** — literal strings, names, signatures. If you can't state it literally, don't delegate to a local model.
4. **Constraints** — "do not touch other files, do not explore the repo, no code comments, don't reformat untouched code" + the repo rules that apply.
5. **Report** — "list the files you changed and anything you could not do".

## Step 3 — Run

Common env + flags (put in a shell variable to avoid repetition):

```bash
OLLAMA_ENV=(ANTHROPIC_BASE_URL=http://localhost:11434 ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_API_KEY=ollama)
BASE=(claude -p --bare --strict-mcp-config --no-session-persistence --model qwen3.8-64k:latest)
```

**Edit** (from the repo root):

```bash
env "${OLLAMA_ENV[@]}" "${BASE[@]}" --tools "Read,Edit,Write,Glob,Grep" --permission-mode acceptEdits \
  --output-format stream-json --verbose "<brief>" < /dev/null \
  > "$SCRATCH/ollama.ndjson" 2> "$SCRATCH/ollama.err"
```

**Review** (read-only — no Edit/Write tools exist in the run; pipe the diff in so it doesn't need git):

```bash
git diff origin/dev...HEAD -- <paths> | env "${OLLAMA_ENV[@]}" "${BASE[@]}" --tools "Read,Glob,Grep" --output-format json \
  "Review the diff in <stdin>. Report only real bugs as file:line, why it is a problem, concrete fix. Skip style nits. If nothing, say so. Do not edit anything." \
  2> "$SCRATCH/ollama.err" | jq -r '"\(.subtype) error=\(.is_error)\n\(.result)"'
```

Notes:

- No `Bash` tool by default: the local model can't run git, tests or arbitrary shell. Claude validates afterwards. If a task truly needs one command, allow it narrowly: `--allowedTools "Bash(npx jest path/to/file.test.ts)"` plus `Bash` in `--tools`.
- `acceptEdits` auto-approves edits inside the working directory only; writes elsewhere are denied in headless mode.
- `< /dev/null` stops it waiting on stdin in edit mode.
- Expect 30 s – several minutes per run (first call also loads the model, ~10–20 s). Long tasks: `run_in_background`, wait for the notice, don't poll.
- The stderr line `[claude-code:unrecognized_model]` is expected for local models — ignore it.

Read the edit result and audit what it actually did:

```bash
jq -r 'select(.type=="result") | "\(.subtype) error=\(.is_error) turns=\(.num_turns)\n\(.result)\nusage=\(.usage.input_tokens)+\(.usage.cache_read_input_tokens) in / \(.usage.output_tokens) out"' "$SCRATCH/ollama.ndjson"
jq -r 'select(.type=="assistant") | .message.content[] | select(.type=="tool_use") | "\(.name)\t\(.input.file_path // .input.pattern // "")"' "$SCRATCH/ollama.ndjson"
```

Failure handling: no `result` event → read `ollama.err`. `is_error=true`, an empty `result`, or zero `Edit`/`Write` calls on an edit task means nothing was done — don't report success. Context overflow shows up as the model ignoring the brief or looping on reads: check `ollama ps` CONTEXT and shorten the brief.

## Step 4 — Verify (never trust the report)

1. `git status --short` vs the baseline + the tool-call audit — only allowed files touched?
2. `git diff` the touched files (read new files in full).
3. Grep for the expected literal and for leftovers of the old one.
4. Run the project validation. Separate failures it caused from pre-existing ones.
5. Review findings are claims: open each file:line yourself and drop what doesn't reproduce.
6. If wrong: one re-run with a tighter brief that says what went wrong. Still wrong → escalate to `cursor-delegate` or fix it yourself. Don't loop on a local model.

## Step 5 — Report

Model used, files changed (one line each) or verified findings, run time, validation result, and whether it escalated. Do not commit unless the user asks.

## Alternative harness: Codex

`codex exec --oss --local-provider ollama -m <model:tag>` also targets Ollama, but as of codex-cli 0.166 with `qwen3.8` it fails: unknown models get fallback metadata, the model calls tools Codex doesn't expose (`read_file`), and the turn dies with `no user query found in messages`. Re-test before relying on it; when using it with `--ignore-user-config`, also pass `-c model_provider='"ollama"'`.

## Task

$ARGUMENTS
