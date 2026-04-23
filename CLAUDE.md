# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A public catalog of reusable [Agent Skills](https://agentskills.io) published to [skills.sh](https://skills.sh) under `tzoof/skills`. It is not a runnable application — consumers install skills via `npx skills add tzoof/skills` or `/plugin marketplace add tzoof/skills` in Claude Code.

## Structure

```
skills/                          # one folder per skill
  <name>/SKILL.md
template/SKILL.md                # blank starting point for new skills
.claude-plugin/marketplace.json  # skills.sh discovery manifest
```

## Adding a Skill

Every new skill requires two things kept in sync:

1. **`skills/<name>/SKILL.md`** — the skill content
2. **`.claude-plugin/marketplace.json`** — add the path to the `skills` array of the relevant plugin collection

### SKILL.md format

Only `name` and `description` are required in frontmatter:

```yaml
---
name: kebab-case-name
description: What this skill does and when to trigger it. This drives auto-activation.
---
```

Copy `template/SKILL.md` as a starting point.

### marketplace.json

Add the new skill path to the `skills` array of the matching plugin entry (or create a new plugin entry if it belongs to a different collection):

```json
"skills": [
  "./skills/existing-skill",
  "./skills/new-skill"
]
```

Validate JSON before committing:

```bash
python3 -m json.tool .claude-plugin/marketplace.json > /dev/null
```

### README.md

Add a row to the Skills table:

```
| [`<name>`](./skills/<name>/) | One sentence description |
```
