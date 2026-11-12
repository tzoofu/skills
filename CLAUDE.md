# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A public catalog of reusable [Agent Skills](https://agentskills.io) published under `tzoof/skills`. It is not a runnable application: there is no build, lint, or test suite. Consumers install skills via `/plugin marketplace add tzoof/skills` (Claude Code) or `npx skills add tzoof/skills` (other agents).

## Structure

```
groups/<plugin>/skills/<name>/SKILL.md   # grouped skills: one plugin folder per group
skills/<name>/SKILL.md                   # standalone skills: each is its own plugin
template/SKILL.md                        # blank starting point for new skills
.claude-plugin/marketplace.json          # marketplace manifest: one entry per group or standalone skill
README.md                                # public catalog: one section per group + standalone table
```

Claude Code always shows plugin skills as `/<plugin>:<skill>`, and there is no setting to hide the prefix. A single-skill plugin therefore shows doubled (`/capture-skill:capture-skill`), so related skills are grouped: group `delegate` with skill `cursor` shows as `/delegate:cursor`. Leave always-on rule skills and one-offs standalone.

**A plugin source folder is auto-scanned.** A `skills` list in a marketplace entry does NOT restrict what loads. Never point a group's `source` at a folder that also holds other skills or a `commands/` folder: everything in it loads and shows up, often duplicated. A group folder contains only `skills/<name>/` for that group's skills; no `commands/`, no stray skills.

## Adding or Renaming a Skill

Standalone skill: three files must stay in sync.

1. **`skills/<name>/SKILL.md`**: the skill content. The folder name, the frontmatter `name`, and the plugin `name` must all match.
2. **`.claude-plugin/marketplace.json`**: append an entry (edit the JSON as text so unrelated entries are not reformatted):
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
3. **`README.md`**: add a row to the standalone table, keeping it alphabetical.

Grouped skill: add `groups/<plugin>/skills/<name>/SKILL.md` (folder name == frontmatter `name`; do not repeat the group word in the name). The group's marketplace entry (`{name, source: "./groups/<plugin>", description, category, keywords, "strict": false}`, no `skills` list) picks it up automatically; update its description and add a row to the group's README section.

Slash references: inside skills write `/<plugin>:<name>` (for example `/delegate:cursor`). If a skill calls a skill in a different plugin, say so in the skill and the README, and note that both plugins must be installed. Prefer putting tightly coupled skills in the same group.

Validate after editing:

```bash
python3 -m json.tool .claude-plugin/marketplace.json > /dev/null
git diff --check
claude plugin validate .
for g in groups/*/; do claude plugin validate "$g"; done
```

Consistency check (prints nothing when everything lines up):

```bash
python3 - <<'PY'
import json, os, glob, re
m = json.load(open('.claude-plugin/marketplace.json'))
sources = [os.path.normpath(p['source']) for p in m['plugins']]
for src in sources:
    if not os.path.isdir(src): print('missing source:', src)
dirs = glob.glob('skills/*/') + glob.glob('groups/*/skills/*/')
for d in dirs:
    d = os.path.normpath(d)
    if not any(d == s or d.startswith(s + os.sep) for s in sources): print('uncovered skill:', d)
    f = os.path.join(d, 'SKILL.md')
    if not os.path.isfile(f): print('no SKILL.md:', d); continue
    n = re.search(r'^name:\s*(.+)$', open(f).read(), re.M)
    if not n or n.group(1).strip().strip('"\'') != os.path.basename(d): print('name mismatch:', d)
for g in glob.glob('groups/*/'):
    for e in os.listdir(g):
        if e != 'skills': print('unexpected in group:', os.path.join(g, e))
PY
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
