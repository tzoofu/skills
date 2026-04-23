---
name: capture-skill
description: Distill the current session's actions into a new, reusable agent skill. Use when the user asks to "save this as a skill", "capture this workflow", or "turn this into a skill".
---

# Capture Session as Skill

Look at everything done in the **current conversation** and synthesize it into a new, reusable skill file.

---

## Step 1: Reflect on the session

Review the current conversation from the start. Identify:

- **What was the goal?** (one sentence)
- **What repeated pattern or workflow emerged?** (the thing worth reusing)
- **Which tools were used?** (Bash, Read, Edit, MCP tools, etc.)
- **What inputs did the user provide?** (file paths, ticket IDs, config values — these become `$ARGUMENTS`)
- **What was the output?** (files written, commands run, reports produced)

If the session was exploratory with no reusable pattern, tell the user: "This session doesn't contain a repeatable workflow. Nothing to capture." Then stop.

## Step 2: Confirm with the user

Before writing anything, present a summary:

```
Proposed skill: <name>
Description: <one sentence — what it does and when to use it>
Workflow steps: <numbered list of 3–7 high-level steps>
Tools needed: <list>
Arguments: <what the user would pass, or "none">
```

Ask: **"Does this look right? Adjust anything before I write the files."**

Wait for the user to confirm or redirect. Apply any changes they request.

## Step 3: Locate the skills repo

Find the skills repo. Try in order:
1. `~/private/skills`
2. `~/skills`
3. Ask the user: "Where is your skills repo?"

## Step 4: Write SKILL.md

Create `<skills-repo>/skills/<skill-name>/SKILL.md` with:

```yaml
---
name: <kebab-case>
description: "<one sentence describing what it does and when to trigger>"
---
```

Body:
- A `# Title` heading
- A short description paragraph
- Numbered `## Step N:` sections derived from the session workflow — specific enough to reproduce, general enough to reuse
- End with a `## Task` section if the skill takes arguments

Keep the skill under 200 lines. If it needs more, suggest splitting into two focused skills.

## Step 5: Update README.md

Add a row to the Skills table in `<skills-repo>/README.md`:

```
| `<name>` | <one sentence description> |
```

## Step 6: Summary

Print:

```
Skill captured: <name>

Files written:
  ✅ skills/<skill-name>/SKILL.md

Updated:
  README.md

Install with:
  npx skills add <github-username>/skills
```

Remind the user to commit and push to GitHub so the skill is publicly available on skills.sh.
