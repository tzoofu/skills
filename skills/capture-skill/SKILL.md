---
name: capture-skill
description: Distill the current session's actions into a reusable artifact — a skill, an agent, or both (skill as slash entrypoint + agent as isolated implementation).
argument-hint: <name>
allowed-tools: Bash, Read, Write, Edit, Glob
disable-model-invocation: true
---

# Capture Session as Skill or Agent

Look at everything done in the **current conversation** and synthesize it into a reusable artifact: a skill, an agent, or both.

## Task

$ARGUMENTS

The argument (if provided) is the desired name in kebab-case. If omitted, infer a name from the session work.

---

## Step 1: Reflect on the session

Review the current conversation from the start. Identify:

- **What was the goal?** (one sentence)
- **What repeated pattern or workflow emerged?** (the thing worth reusing)
- **Which tools were used?** (Bash, Read, Edit, MCP tools, etc.)
- **What inputs did the user provide?** (file paths, ticket IDs, config values — these become `$ARGUMENTS`)
- **What was the output?** (files written, commands run, reports produced)
- **How heavy was the intermediate output?** (many files read, docs fetched, logs parsed — or a small linear flow?)

If the session was exploratory with no reusable pattern, tell the user: "This session doesn't contain a repeatable workflow. Nothing to capture." Then stop.

## Step 2: Choose the shape — skill, agent, or both

Match the workflow against these signals:

**Skill** — user-invocable slash command, runs in main context:
- Mostly orchestration / step-by-step user guidance
- The tool surface is broad / hard to enumerate
- Benefits from `AskUserQuestion` calls mid-flow
- The final result IS the workflow itself

**Agent** — subprocess via `Task` tool, returns one summary:
- **Heavy intermediate output, slim final result** (reads many files / fetches docs / parses logs, then reports)
- The tool surface is **naturally narrow** (e.g. read + grep + specific MCPs, no Edit/Write)
- Tool isolation matters — structurally prevent unwanted tool use, not rely on convention
- No mid-flow user interaction needed

**Both** — skill as entrypoint, agent as implementation:
- User wants the slash command (`/<name>`) AND the work has agent-shape characteristics
- Skill body delegates to the agent via the `Task` tool
- Best of both: portable slash entrypoint + isolated/efficient implementation

Default to **skill** when uncertain. Promote to agent later if real usage shows narrow tools / heavy output.

> **Note:** Skills repos distribute only skills + commands, not agents. Agent files go to `~/.claude/agents/` (user-global) or `<repo>/.claude/agents/` (per-project) — not into the skills repo itself. If the captured artifact is **agent** or **both**, the skill portion still publishes to the skills repo; the agent ships separately.

## Step 3: Confirm with the user

Present a one-paragraph summary:

```
Proposed: <name>
Shape: skill | agent | both
Description: <one sentence — what it does and when to trigger>
Workflow steps: <numbered list of 3–7 steps>
Tools needed: <list>
Arguments: <what the user would pass, or "none">
Agent location (if agent or both): user-global (~/.claude/agents) | per-project (.claude/agents)
```

Ask: **"Does this look right? Adjust shape, scope, tools, or agent location before I write the files."**

Wait for confirmation or redirect.

## Step 4: Locate the skills repo (if shape is "skill" or "both")

Find the skills repo root. Try in order:
1. Current working directory (check for `skills/` folder and `.claude-plugin/marketplace.json`)
2. Ask the user: "Where is your skills repo?"

Confirm it's valid by checking `README.md` and `.claude-plugin/marketplace.json` exist at root.

For **agent** only: skip this step.

## Step 5: Write SKILL.md (if shape is "skill" or "both")

Create `<skills-repo>/skills/<name>/SKILL.md`:

```yaml
---
name: <kebab-case>
description: "<one sentence>"
argument-hint: <what to pass, e.g. <TICKET_ID>>   # omit if no arguments
allowed-tools: <comma-separated list>
disable-model-invocation: true
---
```

Body:
- `# Title` heading
- Short description paragraph
- Numbered `## Step N:` sections derived from the session workflow
- End with:

```markdown
## Task

$ARGUMENTS
```

**For "both" shape**, prepend Step 0 to the workflow:

```markdown
## Step 0: Delegate to the agent (if available)

If the `<name>` agent is installed (`~/.claude/agents/<name>.md` or `.claude/agents/<name>.md` in the current repo), delegate the entire workflow to it via the `Task` tool with the inputs described below. The agent returns a single summary — that becomes your output. Skip the remaining steps.

If the agent is NOT available, execute the steps below inline.
```

Keep the skill under 200 lines.

## Step 6: Write the agent file (if shape is "agent" or "both")

Write to the location chosen in Step 3:
- User-global: `~/.claude/agents/<name>.md`
- Per-project: `<repo-root>/.claude/agents/<name>.md`

Frontmatter:

```yaml
---
name: <kebab-case>
description: Use this agent when [trigger context]. Examples: <example>Context: <realistic scenario> user: '<example prompt>' assistant: 'I'll use the <name> agent — <why>' <commentary><why this is the right fit></commentary></example>
model: sonnet | haiku
color: <green | blue | cyan | yellow | purple | red | orange | pink>
tools: <comma-separated allowlist>
---
```

**Critical fields:**

- **`description`** must be **trigger-rich**: include 1–2 `<example>` blocks with realistic user prompts. Claude uses this to decide when to invoke the agent. A bare description without examples often won't auto-trigger.
- **`tools`** is an **allowlist** — the agent cannot use anything not listed. Explicitly OMIT tools that would let the agent do things the workflow shouldn't (e.g. omit `Edit`/`Write` for read-only audits to structurally enforce read-only). Never include `Agent` unless you genuinely want sub-delegation.
- **`model`**:
  - `haiku` — fast mechanical work (batched edits, simple verifications)
  - `sonnet` — workflows needing judgment (rule audits, code review, planning)

Body structure:

1. **Input contract** — describe the JSON shape the caller passes via the agent prompt. Required vs optional fields.
2. **Numbered `## Step N:` sections** — same level of detail as a skill, but assume no mid-flow user interaction (the agent returns once).
3. **Final report shape** — what the agent returns. Keep it tight (≤ 250 words). End with a terminal-state sentence (e.g. "Ready to commit." / "Audit found N issues.").

## Step 7: Update README.md (if shape is "skill" or "both")

Add a row to the Skills table in `<skills-repo>/README.md`:

```
| [`<name>`](./skills/<name>/) | <one sentence description> |
```

Insert in alphabetical order.

## Step 8: Update marketplace.json (if shape is "skill" or "both")

Read `<skills-repo>/.claude-plugin/marketplace.json`. Append to the `plugins` array:

```json
{
  "name": "<name>",
  "source": "./skills/<name>",
  "description": "<description>",
  "category": "universal",
  "keywords": ["<2–5 relevant tags>"]
}
```

If an entry with this name already exists, update it in place. Validate JSON after writing:

```bash
python3 -m json.tool <skills-repo>/.claude-plugin/marketplace.json > /dev/null
```

## Step 9: Summary

Print:

```
Captured: <name>
Shape: skill | agent | both

Files written:
  ✅ skills/<name>/SKILL.md                 (if skill or both)
  ✅ <agent-path>                           (if agent or both)

Updated:
  📝 README.md                              (if skill or both)
  📝 .claude-plugin/marketplace.json        (if skill or both)

Use it:
  /<name>                                   (if skill or both — slash command)
  Task → <name>                             (if agent or both — main Claude delegates via Task tool)

To share in another project:
  Skill — copy skills/<name>/ → .claude/skills/<name>/
  Agent — copy <agent-path> → ~/.claude/agents/<name>.md (or <repo>/.claude/agents/<name>.md)
```

Remind the user to commit the changes so the skill is available to others. Agent files in `~/.claude/agents/` are local-only and don't need a commit; per-project agents should be committed to that repo.
