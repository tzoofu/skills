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

Related skills ship as one plugin per group, so they show in the picker as `/<group>:<action>`. Install a group with `claude plugin install <group>@tzoof-skills`.

### `delegate` — hand work to another coding agent

Each skill writes the brief, runs the other agent, then verifies the result itself. Skills in this group reference each other (for example the cost ladder in `/delegate:cursor`), so install the whole group.

| Command | What it does |
| --- | --- |
| [`/delegate:claude`](./groups/delegate/skills/claude/) | Delegate coding tasks to the Claude Code CLI while the managing agent scopes the work, reviews changes, runs validation, and sends corrective follow-ups |
| [`/delegate:codex`](./groups/delegate/skills/codex/) | Hand edits, reviews, plans, structured answers or parallel worktree tasks to OpenAI Codex CLI (ChatGPT models), pick model/effort/speed, verify the result |
| [`/delegate:cursor`](./groups/delegate/skills/cursor/) | Hand a scoped task to a headless Cursor agent (`agent` / `cursor-agent`), pick a model from a cost/quality ladder, verify the result |
| [`/delegate:cursor-parallel`](./groups/delegate/skills/cursor-parallel/) | Run several Cursor CLI tasks in parallel git worktrees (optionally on different models), then verify and merge |
| [`/delegate:cursor-review`](./groups/delegate/skills/cursor-review/) | Read-only second opinion, plan, or code review from a different model family via the Cursor CLI in ask/plan mode |
| [`/delegate:ollama`](./groups/delegate/skills/ollama/) | Hand a scoped edit or read-only review to a local Ollama model (qwen, gemma) via headless Claude Code `--bare` — free, offline — and verify the result |

### `audit` — find and fix quality problems

| Command | What it does |
| --- | --- |
| [`/audit:bugs`](./groups/audit/skills/bugs/) | Audit a codebase for real, substantiated bugs using parallel specialized agents, rigorously verify every finding, then optionally plan and execute the fixes via parallel scoped implementation agents |
| [`/audit:detangle`](./groups/audit/skills/detangle/) | Untangle a messy or AI-generated codebase without behavior changes: map real coupling (imports + git co-change), get plan approval, then reorganize into feature folders, split mixed-responsibility files, merge only true duplicates, and clean comments and tests, with typecheck/lint/tests gating every step |
| [`/audit:perf`](./groups/audit/skills/perf/) | Audit a web/mobile app for speed and perceived performance and get a prioritized, measured action plan with code snippets |
| [`/audit:regression`](./groups/audit/skills/regression/) | Analyze the current branch diff and produce a manual testing board + Playwright test plan scoped to what changed |
| [`/audit:solid`](./groups/audit/skills/solid/) | Audit a codebase (or a scoped area) against SOLID/DRY and its own documented conventions, implement a user-approved refactor of the confirmed violations, and verify with the project's own gates plus a bug-class sweep (lean) or a two-track adversarial regression pass (thorough) |
| [`/audit:ui-bugs`](./groups/audit/skills/ui-bugs/) | Given a report that a UI control doesn't work, find the true root cause, generalize it into a bug signature, sweep the codebase for the same signature, fix every instance, and verify live with Playwright |

### `git` — history cleanup

`/git:squash` points to `/git:clean-history` for pushed or stale branches, so install the whole group.

| Command | What it does |
| --- | --- |
| [`/git:clean-history`](./groups/git/skills/clean-history/) | Untangle a contaminated or stale feature branch into a single clean commit on a fresh base, with diff verification before a safe force-push |
| [`/git:squash`](./groups/git/skills/squash/) | Squash every local commit ahead of the remote tracking branch into one commit with a given message — no rebase, no force-push |

### Standalone skills

Single-skill plugins; Claude Code shows them as `/<name>:<name>`.

| Skill | Description |
| --- | --- |
| [`capture-skill`](./skills/capture-skill/) | Distill the current session's actions into a new, reusable agent skill |
| [`generate-docs`](./skills/generate-docs/) | Explore a codebase and generate one reference doc per major subsystem under `docs/`, then wire them into CLAUDE.md with a reference table |
| [`quick-codex-pet`](./skills/quick-codex-pet/) | Create a Codex custom pet package from a person or character reference image, with `pet.json`, `spritesheet.webp`, and atlas validation |
| [`setup-cloud-routine`](./skills/setup-cloud-routine/) | Set up or update a claude.ai scheduled cloud routine (RemoteTrigger) for a recurring task — picks the right scheduling mechanism, checks available MCP connectors, chooses a live config-storage approach, and verifies with a manual run |
| [`spec-to-monorepo`](./skills/spec-to-monorepo/) | Takes a rough AI-generated spec or plan and turns it into a working, secure, documented multi-service monorepo — architecture review, service scaffolding, cross-service wiring, Snyk scanning, and a project README + CLAUDE.md with real gotchas |

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
