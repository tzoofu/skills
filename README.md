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

| Skill                                                          | Description                                                                                                                                                                                                                                                  |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [`bug-bounty`](./skills/bug-bounty/)                           | Audit a codebase for real, substantiated bugs using parallel specialized agents, rigorously verify every finding, then optionally plan and execute the fixes via parallel scoped implementation agents                                                      |
| [`capture-skill`](./skills/capture-skill/)                     | Distill the current session's actions into a new, reusable agent skill                                                                                                                                                                                       |
| [`clean-branch-history`](./skills/clean-branch-history/)       | Untangle a contaminated or stale feature branch into a single clean commit on a fresh base, with diff verification before a safe force-push                                                                                                                  |
| [`generate-docs`](./skills/generate-docs/)                     | Explore a codebase and generate one reference doc per major subsystem under `docs/`, then wire them into CLAUDE.md with a reference table                                                                                                                    |
| [`quick-codex-pet`](./skills/quick-codex-pet/)                 | Create a Codex custom pet package from a person or character reference image, with `pet.json`, `spritesheet.webp`, and atlas validation                                                                                                                      |
| [`regression-check`](./skills/regression-check/)               | Analyze the current branch diff and produce a manual testing board + Playwright test plan scoped to what changed. Run before /audit.                                                                                                                         |
| [`setup-cloud-routine`](./skills/setup-cloud-routine/)         | Set up or update a claude.ai scheduled cloud routine (RemoteTrigger) for a recurring task — picks the right scheduling mechanism, checks available MCP connectors, chooses a live config-storage approach, and verifies with a manual run                    |
| [`solid-audit-refactor`](./skills/solid-audit-refactor/)       | Audit a codebase (or a scoped area) against SOLID/DRY and its own documented conventions, implement a user-scoped refactor of the confirmed violations, and verify with the project's own gates (lean) or a two-track adversarial regression pass (thorough) |
| [`spec-to-monorepo`](./skills/spec-to-monorepo/)               | Takes a rough AI-generated spec or plan and turns it into a working, secure, documented multi-service monorepo — architecture review, service scaffolding, cross-service wiring, Snyk scanning, and a project README + CLAUDE.md with real gotchas           |
| [`squash-unpushed-commits`](./skills/squash-unpushed-commits/) | Squash every local commit ahead of the remote tracking branch into one commit with a given message — no rebase, no force-push                                                                                                                                |
| [`ui-bug-sweep`](./skills/ui-bug-sweep/)                       | Given a report that a UI control doesn't work, find the true root cause, generalize it into a bug signature, sweep the codebase for the same signature, fix every instance, and verify live with Playwright                                                  |

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
