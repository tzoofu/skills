# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A public catalog of reusable [Agent Skills](https://agentskills.io) published under `tzoof/skills`. It is not a runnable application: there is no build, lint, or test suite. Consumers install skills via `/plugin marketplace add tzoof/skills` (Claude Code) or `npx skills add tzoof/skills` (other agents).

## Structure

```
skills/<name>/SKILL.md           # one folder per skill; each skill is its own plugin
template/SKILL.md                # blank starting point for new skills
.claude-plugin/marketplace.json  # marketplace manifest: one plugin entry per skill
README.md                        # public Skills table
```

A skill folder may also carry a `commands/<name>.md` slash-command variant (see `skills/spec-to-monorepo/`).

## Adding or Renaming a Skill

Three files must stay in sync. Skipping one means the skill is either undiscoverable or undocumented:

1. **`skills/<name>/SKILL.md`**: the skill content. The folder name, the frontmatter `name`, and the plugin `name` must all match.
2. **`.claude-plugin/marketplace.json`**: append an entry to the `plugins` array:
   ```json
   {
     "name": "<name>",
     "source": "./skills/<name>",
     "description": "Command: /<name> — <one-sentence summary>",
     "category": "universal",
     "keywords": ["...", "..."]
   }
   ```
   Begin `description` with `Command: /<name> — ` when the skill is user-invoked (`disable-model-invocation: true`). Model-invoked skills like `quick-codex-pet` omit that prefix.
3. **`README.md`**: add a row to the Skills table, keeping it alphabetical:
   ```
   | [`<name>`](./skills/<name>/) | One sentence description |
   ```

Validate the manifest after editing:

```bash
python3 -m json.tool .claude-plugin/marketplace.json > /dev/null
```

## SKILL.md Conventions

Only `name` and `description` are required by the spec. The existing skills follow these conventions:

- **`description`** drives auto-activation, so say both what the skill does and when to trigger it. Quote it if it contains `:` or `"`.
- **`disable-model-invocation: true`** on workflow skills that should only run as an explicit `/command`. Most skills here use it.
- **`argument-hint`** documents the slash-command arguments, e.g. `<path or module to audit — optional>`.
- **`allowed-tools`** is a tight allowlist, including specific MCP tool names when needed (e.g. `mcp__playwright__browser_*`, Context7, Snyk).
- The body is organized as numbered `## Step N:` sections. Keep skills well under ~200 lines.
- Never pin model IDs (`claude-opus-5-5`) in skills or generated agent frontmatter. Use the aliases `inherit`/`haiku`/`sonnet`/`opus`.

`skills/capture-skill/SKILL.md` is the meta-skill that writes new skills into this repo. Keep its instructions consistent with this file when conventions change.
