---
name: detangle
description: "Untangle a messy, fast-grown or AI-generated codebase into a clean, feature-organized structure without changing behavior: maps real coupling (imports plus git co-change history), proposes a plan for approval, then reorganizes folders, splits mixed-responsibility files, merges only true duplicates, and cleans comments and tests, with typecheck/lint/tests gating every step. Use when the structure is the problem (everything dumped in src/, giant utils files, copy-pasted helpers after vibe-coding); for SOLID/DRY design drift use /audit:solid."
argument-hint: <path or module — optional, defaults to the whole repo>
allowed-tools: AskUserQuestion, Read, Edit, Write, Bash, Grep, Glob, Agent
disable-model-invocation: true
effort: high
---

# Detangle: Map → Plan → Restructure → Verify

Refactor only. No behavior changes; if one is unavoidable, stop, explain it, and list it in the
final summary. Never invent numbers: report counts you measured, or write `unmeasured`.

Copy this checklist into your response and tick it off as you go:

```
Detangle progress:
- [ ] 0. Pre-flight: stack, clean tree, gate commands, baseline
- [ ] 1. Coupling map
- [ ] 2. Plan approved
- [ ] 3. Reorganize (moves only) — gate green, committed
- [ ] 4. Split files — gate green, committed
- [ ] 5. Merge true duplicates — gate green, committed
- [ ] 6. Comments — gate green, committed
- [ ] 7. Tests (report → approval → act) — gate green, committed
- [ ] 8. Summary
```

## The gate (after every step from 3 to 7)

Run the project's typecheck, lint and tests. Green and no fewer passing tests than the baseline →
commit the step as a local checkpoint (`detangle: <step>`), never push. Red → fix it within the
step's intent; if that needs a behavior change or more than a small fix, `git revert` the step's
commit (or `git checkout -- .` if not committed yet), stop, and report. Never start a step on a
red tree.

## Step 0: Pre-flight

1. Scope = `$ARGUMENTS` or the whole repo.
2. Detect the stack from `package.json`, config files and folder layout: Next.js (App or Pages
   Router), React (Vite/CRA), Node, Expo / Expo Router, bare React Native. Monorepo → treat each
   package separately. Can't tell → ask with `AskUserQuestion`.
3. `git status --porcelain` must be empty. If not, stop and ask the user to commit or stash —
   checkpoint commits must contain only this skill's changes.
4. Find the gate commands (`package.json` scripts, Makefile, CI config). Missing typecheck or
   lint is fine; note it. Run them once and record the baseline: pass/fail per command and the
   test pass/fail/skip counts. A red baseline → stop and report; don't refactor on top of it.
5. Gauge test coverage roughly (which feature areas have tests). Where logic has no tests,
   restrict work to pure moves and import updates — no splits or merges of untested logic.

## Step 1: Map coupling

Build the real dependency picture, not just imports:

- **Import graph** — who imports whom; find cycles and modules everyone imports.
- **Co-change history** — files that change in the same commits are coupled even with no import:
  ```bash
  git log --since=12.months --name-only --pretty=format:'--' -- <scope> \
    | awk '/^--$/{n++; next} NF{print n"\t"$0}'
  ```
  Count file pairs per commit; skip huge commits (mass renames, formatting) as noise.
- **Hidden channels** — shared module-level state, globals, singletons, event emitters / pub-sub
  names, env vars, string-keyed registries, DB tables touched from several areas.

Group files into feature clusters (auth, billing, checkout, …) plus a small `shared/` for truly
cross-cutting code. For big repos, fan out one Explore agent per top-level area and merge results.

## Step 2: Plan and approval

Present, with evidence for each item (import counts, co-change pairs, line ranges):

- Target folder layout and the move list (old → new path).
- Files to split and the proposed parts.
- Duplicate groups to merge, with the co-change evidence that they change together.
- Comment and test cleanup approach.
- **Leave alone** — what you deliberately won't touch and why.
- Behavior changes, if any — should be "none".

Ask for approval with `AskUserQuestion` (approve / adjust / cancel). Don't edit before approval.

## Step 3: Reorganize (moves only)

Use `git mv` so history follows. Update imports, `tsconfig` `paths`, bundler/Babel/Jest/Metro
aliases together. No content edits other than import paths. Never move these:

| Stack | Frozen paths | Notes |
|-------|--------------|-------|
| Next.js App Router | `app/**`, `middleware.ts`, `instrumentation.ts`, `next.config.*` | Folders are URLs — move logic out of `app/` into features, keep route files in place |
| Next.js Pages Router | `pages/**`, `middleware.ts` | Same idea |
| Expo / Expo Router | `app/**`, `app.json` / `app.config.*` | File-based routing |
| Bare React Native | `index.js`, `ios/`, `android/` | Update Metro / `babel.config.js` aliases |
| React (Vite/CRA) | entry file, `public/`, `index.html` | |
| Node | files referenced by `package.json` `main` / `bin` / `exports` | Moving them breaks consumers |

Also leave generated code, migrations, and anything referenced by path from config or CI. Gate.

## Step 4: Split files

Split along responsibilities, not line counts. Over ~300 lines is a reason to look, not a rule;
a cohesive 400-line file stays. Aim for focused files (often ~150–200 lines). Keep public exports
stable: re-export from the original path only if external consumers import it, otherwise update
call sites. Gate.

## Step 5: Merge true duplicates

Follow `/audit:solid`'s rule: duplication is a problem only when copies change for the same
reason. Merge only groups backed by Step 1 co-change evidence or identical domain rules. Similar-
looking code with different owners stays duplicated. When in doubt, leave it and list it under
deferred. Gate.

## Step 6: Comments

Delete comments that restate the code (`// get user by id`), commented-out code, and stale
section banners. Keep comments explaining *why*, workarounds with links, TODO/FIXME with context,
license headers, public-API docs (JSDoc/TSDoc), and lint/type directives. Gate.

## Step 7: Tests (report first)

List candidates in two groups, with a reason per test:

- **Delete** — can't catch a real bug: asserts mocks only, tests a deleted feature, duplicates
  another test, snapshot of trivial markup.
- **Fix** — fails or flakes without a real bug: timing, order dependence, stale fixtures,
  brittle selectors.

Ask with `AskUserQuestion` which to apply. Act only on approved items; never delete a test that
covers code you changed in Steps 3–6. Gate.

## Step 8: Summary

Report in tables:

- Moved (old → new), split (file → parts), merged (copies → one), deleted (comments count,
  tests by name).
- Gate results: baseline vs final per command, test counts before → after.
- Behavior changes (expected: none).
- Deferred items and why, plus checkpoint commit SHAs so the user can review or revert each step.

Don't push. Remind the user the checkpoint commits are local and can be squashed (`/git:squash`).
