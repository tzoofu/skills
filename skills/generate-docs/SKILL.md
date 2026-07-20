---
name: generate-docs
description: Explore a codebase and generate a set of subsystem reference docs (one Markdown file per major subsystem) inside a `docs/` folder, then wire them into CLAUDE.md with a reference table. Trigger when the user says "generate docs", "create docs", "document this codebase", or "write reference docs".
allowed-tools: Read, Write, Edit, Bash, Glob
disable-model-invocation: true
---

# Generate Subsystem Reference Docs

Explore this codebase and produce a reference doc for each major subsystem under `docs/`, then link them from `CLAUDE.md`.

---

## Step 1: Discover subsystems

Read the codebase — entry points, `lib/`, `app/`, `src/`, `server/`, `services/`, config files, and any `CLAUDE.md` / `README.md` already present. Identify the major subsystems. Typical ones:

- Authentication & user management
- Authorization / protection layers (middleware, route guards, role checks)
- Database / ORM access patterns
- Caching strategy
- API layer (REST, GraphQL, RPC — routes, conventions, error handling)
- Background jobs / queues / workers
- File / object storage
- Real-time / push (WebSockets, FCM, SSE, webhooks)
- Third-party integrations (one doc per integration if non-trivial)
- Analytics / event tracking
- Configuration & environment variables

Skip subsystems that don't exist. Add others that are clearly present.

Produce a short list: `<slug> — <one-line summary>` for each subsystem you will document. Present it to the user and wait for a "looks good" or adjustments before writing any files.

---

## Step 2: Write one doc per subsystem

Create `docs/<slug>.md` for each confirmed subsystem. Follow this structure:

**Required structure:**

1. `# Title` — one sentence describing what this subsystem is
2. Short paragraph: purpose, where it lives in the codebase, what it touches
3. `---` separator
4. Sections that cover, where relevant:
   - **Overview table** — key concepts / components in a Markdown table
   - **Flow diagram** — ASCII art for multi-step flows (auth handshakes, data pipelines, upload paths, job lifecycles). Show the exact function/route/class names at each step.
   - **Key files** — table of `file` → what it exports / does
   - **Patterns & rules** — how to use the subsystem correctly; what must never be done; gotchas a new contributor would hit
   - **DB / storage layout** — table of collection/table/bucket paths → contents (if the subsystem persists data)
   - **Environment variables** — table of `VAR_NAME` → required/optional → purpose
   - **How to extend** — numbered steps to add a new item (new provider, new field, new event type, new route, etc.)

**Style rules — these are non-negotiable:**
- No filler sentences ("This document explains…", "As you can see…"). Start every section with substance.
- Every table row must convey a fact not immediately obvious from reading the variable/function name alone.
- Code blocks must use real function, file, and variable names from this repo — no invented placeholders.
- Prefer tables over prose for lists. Prefer ASCII diagrams over prose for flows.
- Write for a senior engineer who is new to this specific repo but needs to be productive in 10 minutes.
- Keep each doc under 300 lines.

---

## Step 3: Update CLAUDE.md

Check if `CLAUDE.md` exists at the repo root.

**If it exists:** find the `## Architecture` section (or the first `##` section after `## Commands`) and insert before it:

```markdown
## Reference docs (`docs/`)

Deep-dives into individual subsystems — read these when working on the relevant area:

| File | Topic |
|---|---|
| [`docs/<slug>.md`](docs/<slug>.md) | One-line description |
```

**If it doesn't exist:** create a minimal `CLAUDE.md`:

```markdown
# CLAUDE.md

## Commands

<!-- fill in run/build/test commands here -->

## Reference docs (`docs/`)

Deep-dives into individual subsystems — read these when working on the relevant area:

| File | Topic |
|---|---|
| [`docs/<slug>.md`](docs/<slug>.md) | One-line description |
```

---

## Step 4: Quality check

Before reporting done, re-read each doc and verify:

- [ ] Every table row adds information not visible in the code symbol name itself
- [ ] All filenames, function names, env var names, and route paths are real (grep-verifiable in the repo)
- [ ] Every flow diagram matches the actual call chain — verify by tracing it in the code
- [ ] "Patterns & rules" sections are specific enough that a reader doesn't need to look at the code to follow them
- [ ] No doc exceeds 300 lines

Fix anything that fails before finishing.

---

## Step 5: Summary

Print:

```
Docs generated:
  docs/<slug>.md — <one-line topic>
  ...

CLAUDE.md updated with reference table.

To keep docs fresh: re-run /generate-docs after major subsystem changes.
```

---

## Task

$ARGUMENTS
