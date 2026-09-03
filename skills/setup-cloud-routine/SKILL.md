---
name: setup-cloud-routine
description: 'Set up or update a claude.ai scheduled cloud routine (RemoteTrigger) for a recurring task — correctly distinguishes it from session-only CronCreate and Claude Desktop''s local scheduled-tasks, diagnoses which MCP connectors are actually attached before assuming a data source works, picks a viable way to store any config that changes over time (Slack canvas > git file > repo checkout), writes a self-contained prompt, and verifies with a manual run before trusting the schedule. Trigger when the user wants something to run automatically every day/week/hour, wants a recurring Slack/report/reminder automation, or says things like "can we schedule this", "make this run automatically", "send this on a routine".'
argument-hint: <description of the recurring task — what to do, how often, which channels/services>
allowed-tools: Bash, ToolSearch, AskUserQuestion, RemoteTrigger
disable-model-invocation: true
---

# Setup Cloud Routine

Turns a one-off "can you do X for me" into a durable, unattended recurring automation using
claude.ai's cloud routines — not the local, session-scoped, or desktop-only alternatives that look
similar but aren't durable or reachable from here.

## Task

$ARGUMENTS

## Step 1: Understand the goal

Pin down, in plain terms:

- What should happen each time it fires (post a message, run a check, send a summary)?
- How often, and in the user's local timezone (they'll say "10:30 every morning" — you'll need to
  convert to UTC later)?
- Which inputs might change over time — a list of channels, a set of labels/tags, a threshold? These
  need a config-storage decision in Step 5; don't hardcode them yet.

## Step 2: Pick the right mechanism — don't confuse these three

| Mechanism                                                        | Where it lives                                                | Persistence                                                                                       |
| ---------------------------------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `CronCreate` (a Claude Code tool)                                | This CLI session's memory                                     | Dies when this session ends; auto-expires after 7 days regardless                                 |
| Claude Desktop routines (`~/.claude/scheduled-tasks/*/SKILL.md`) | Desktop app's own local-agent-mode                            | Persistent, but only creatable/manageable from inside the Desktop app — no tool here can touch it |
| **claude.ai routine** (`RemoteTrigger` API)                      | Hosted on claude.ai, independent of any local machine/session | Persistent, fires regardless of whether any laptop or session is open                             |

For anything that needs to survive past this conversation, use the **claude.ai routine**. Tell the
user explicitly if they described the Desktop-style routine — you can't create that one here, only
the claude.ai one, and it's worth naming the difference rather than silently substituting.

Load the tool first: `ToolSearch select:RemoteTrigger`. Auth is handled in-process — never use `curl`
against the API directly.

## Step 3: Check what's actually available before assuming it works

`RemoteTrigger action=list` to see existing routines (avoid duplicates; note the connected MCP
connectors and the default environment — these come back in the `schedule` skill's own instructions
when invoked, or infer from a prior `create`/`get` response).

If the task needs a data source (Jira, GitHub, a specific API) beyond what's already connected,
**say so before building around it**. A connector not being listed means the cloud routine cannot
call that tool at all — there is no partial/degraded access. Redesign around what IS connected (e.g.
source status from a Slack channel's own message history instead of querying a ticket tracker) rather
than building a prompt with a fallback branch for "if this tool fails" when it will always be absent.
Only use a fallback branch for genuine transient failures, not for a connector you already know isn't attached.

## Step 4: Write a fully self-contained prompt

The cloud session starts with **zero memory of this conversation**. Every ID, tone instruction, and
fallback behavior must be spelled out literally in the prompt text — never write "as discussed above"
or "the channel we talked about." Concretely:

- Resolve names to IDs yourself (channel IDs, not names) and embed the IDs.
- State the exact tone/format expected (e.g. "no greeting, no headers, match this person's existing
  terse style") rather than assuming the agent will infer it.
- State explicitly which tools it should and shouldn't call (e.g. "do not attempt to call any
  Jira/Atlassian tool — none is connected here").

## Step 5: Decide how to store config that changes over time

If Step 1 identified an input likely to change (a channel list, label mapping, threshold), don't
hardcode it into the prompt and don't point it at a local file or repo path — **cloud routines cannot
read local files, local services, or local environment variables.** Rank the options by cost:

1. **A Slack canvas** (or similarly a doc reachable through whatever connector is already attached) —
   the routine reads it fresh every run; the user edits it directly with no sync step and no code
   change. Prefer this whenever the routine already has a connector that can read documents/messages.
2. **A git-tracked file + manual sync** — best "file" semantics (history, PR review) but the routine
   can't read it; someone has to notice every edit and push it into the routine via
   `RemoteTrigger update`. Silent staleness is the built-in failure mode — only pick this if the user
   explicitly wants the git-review workflow and accepts the sync step.
   `RemoteTrigger update` sends the **entire** `job_config`, not a merge — always re-fetch with
   `action=get` first and edit its actual current state, never resend an old cached body, or the sync
   will silently revert unrelated fields (including a config-storage wiring already in place, see the
   warning in Step 8).
3. **A repo checkout attached to the routine's session** — true unattended file-based config, but the
   routine clones fresh on every single firing. Only worth it for a large/complex config or when the
   routine needs actual code, not for a handful of rows — and confirm auth is actually wired for that
   repo host before assuming it'll work.

Ask the user which tradeoff they want if it's not obvious from the task — this is a real fork with
different maintenance costs, not a detail to default silently.

## Step 6: Create the routine

`RemoteTrigger action=create` with:

- `cron_expression` in **UTC** — convert from the user's local time and double-check the current DST
  offset for their timezone (don't assume standard offset if the date falls in DST).
- `mcp_connections` for whatever the prompt actually calls.
- A fresh lowercase v4 UUID for the event (`uuidgen | tr '[:upper:]' '[:lower:]'` via Bash) — never
  reuse one across create/update calls for a changed prompt.

Report back the routine's claude.ai URL.

## Step 7: Verify with a manual run before trusting the schedule

`RemoteTrigger action=run` immediately. Then check the **actual output** — read the Slack channel,
check the file, whatever the routine was supposed to produce — rather than trusting a 200 response as
proof it worked. A cloud run is asynchronous; if the output isn't there yet, say so and check again
rather than assuming success or failure prematurely.

## Step 8: Warn about the round-trip gotcha

If the user says they edited the routine themselves (e.g. "I changed the cron time"), or before you
hand off, warn them: editing a routine from the claude.ai web UI (or calling `update` with a stale
snapshot) can **silently revert fields you didn't mean to touch** — including the entire prompt —
because the API takes a full `job_config` replacement, not a merge. After any external edit,
`RemoteTrigger action=get` and diff the returned prompt/config against what it should be before
assuming the edit was clean.

## Step 9: Report

Summarize: the routine's URL and schedule (human-readable, with timezone), what connectors it uses,
where its config lives and how to edit it, the manual-run verification result, and the round-trip
warning from Step 8.
