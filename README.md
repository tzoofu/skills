# skills

A collection of reusable [Agent Skills](https://agentskills.io) for Claude Code and other AI agents. Skills are folders containing a `SKILL.md` file that gives an agent specialized knowledge and repeatable workflows.

## Getting Started

**Claude Code** — Register the marketplace and install:

```bash
/plugin marketplace add tzoof/skills
```

Then browse and install individual skill collections from the marketplace.

**Other agents** — Install via the CLI:

```bash
npx skills add tzoof/skills
```

Or copy any skill folder directly into your agent's skills directory (e.g. `.agents/skills/` or `.claude/skills/`).

## Skills

| Skill | Description |
|---|---|
| [`capture-skill`](./skills/capture-skill/) | Distill the current session's actions into a new, reusable agent skill |

## Creating Your Own Skills

Use the included [template](./template/SKILL.md) as a starting point. A skill requires only a folder with a `SKILL.md` file:

```
my-skill/
└── SKILL.md
```

Minimal `SKILL.md`:

```yaml
---
name: my-skill
description: What this skill does and when to use it.
---

# My Skill

Instructions for the agent...
```

See the [Agent Skills specification](https://agentskills.io/specification) for the full format reference.

To publish your own collection, fork this repo or create a new one with the same structure, then share it via `npx skills add <github-username>/skills`.

## Disclaimer

These skills are provided for demonstration and educational purposes. Test thoroughly before relying on them for critical workflows.
