---
name: solid-audit-refactor
description: "Audits a codebase (or a scoped area of it) against SOLID/DRY and its own documented conventions, implements a user-approved refactor of the confirmed violations, and verifies it with the project's own gates plus a bug-class sweep (lean) or a two-track adversarial regression pass (thorough). Use when a working codebase has grown duplicated or diverged logic, god classes, or drift from its own documented conventions."
argument-hint: <path or module to audit — optional, defaults to the whole repo>
allowed-tools: Agent, AskUserQuestion, Read, Edit, Write, Bash, Grep, Glob
disable-model-invocation: true
effort: high
---

# SOLID Audit → Pragmatic Refactor → Regression Pass

Audits the target codebase for SOLID/DRY violations with concrete evidence, implements a
minimal-diff refactor that respects the project's documented conventions, and closes with a
regression review. Designed for codebases that are functionally solid but grew duplication
and god classes organically.

Copy this checklist into your response and tick it off as you go — every step is required in
both modes:

```
Audit progress:
- [ ] 0. Mode + scope chosen
- [ ] 1. Baseline captured (before any edit); audit agents launched; every finding re-read
- [ ] 2. Findings triaged into in-scope batches + deferred list
- [ ] 3. Plan approved
- [ ] 4. Implemented shared code → call sites, bug-class sibling sweep per fix
- [ ] 5. Tests for extracted logic
- [ ] 6. Gates pass + bug-class sweep loop clean
- [ ] 7. Doc sweep for deleted/renamed symbols; final report
```

## Step 0: Choose mode and scope

These are **two independent choices** — ask both in one `AskUserQuestion` call (skip either
when the request already implies it):

- **Scope** — which part of the repo to audit (a path/module, a layer, or the whole repo).
  Scope decides audit *breadth*: how many audit agents to launch in Step 1.
- **Mode** — how much *rigor* to spend on design and verification:
  - **Lean** (recommended when the user knows the codebase or wants a fast pass): you write
    the plan yourself and verify with the project's own gates plus the bug-class sweep.
  - **Thorough** (recommended for an unfamiliar repo or a high-risk change): adds a
    refactor-depth choice, a dedicated Plan agent, and a two-track adversarial regression
    pass. See [thorough.md](thorough.md) — load it only in this mode.

Lean + whole repo is a valid combination: it still fans out by layer in Step 1, it just skips
the Plan agent and the adversarial tracks.

Non-negotiables in both modes: ground the audit in the project's own documented conventions,
never trust a finding you haven't personally confirmed against the current source, and never
implement before the user has approved a concrete plan.

## Step 1: Evidence-gathering audit

Read the project's CLAUDE.md (or equivalent docs) first — it distinguishes *deliberate*
conventions from accidental debt, and gives the audit something concrete to check drift
against (a documented "always route through X" rule is a much stronger finding than a generic
DRY nit).

**Capture the clean-tree baseline now, before any edit**: run the project's type-check/build
and test gates and save their output (error list, warning count, pass/fail counts). Every
later "did I break this?" is only meaningful against this snapshot. If the baseline already
has failures, record them and check `git log -p` on the failing test's subject for the cause.
Fix one only when it's clearly a stale test left behind by a deliberate code change **and**
you're touching that file anyway; otherwise report it as pre-existing and leave it alone.

Launch audit agents in one message: **1 agent for a scoped area, 1–3 split by layer** (e.g.
server/data vs UI vs scripts) for a wide scope. Use `Explore` by default — the harness's Plan
Mode only allows Explore agents during research; `code-reviewer` is fine outside Plan Mode.
Ask each agent to rank 8–15 substantive findings most-important-first (not a nitpick list),
each with `file:line`, a short quoted snippet, a concrete failure scenario, and whether it's a
real bug or pure duplication:

- **Duplicated logic**: the same function/check copy-pasted across files — and critically,
  whether the copies have **diverged** (a missing guard in one copy is a real bug, not style).
  Search by **value/content, not just symbol name**, and outside the obvious layer too — a
  fourth copy often hides in a utility/renderer/script dir under a different name.
- **Rule drift**: code that breaks a convention the project's own docs/rules state.
- **God classes/modules**: persistence + business logic + system callbacks + UI mixed together.
- **Scattered constants**: shared keys/IDs/magic numbers re-typed as literals per file.
- **Dependency shape** and **test coverage**: what's constructed inline vs injected; what
  pure logic is trapped in an untestable module.

Then **personally re-read every finding's actual file before building on it**. An agent's
summary describes what it *believes* it found; a finding whose line or behavior doesn't match
the current source doesn't survive into the plan. Prefer LSP (`findReferences`) over grep for
"is this dead code / who calls this" claims.

## Step 2: Triage

**Thorough:** first ask the refactor-depth question in [thorough.md](thorough.md).

Both modes: order the verified findings cheapest/lowest-risk first. If there are more than
~15, don't try to do them all in one pass — split them into:

- **Batch A — real bugs in diverged copies** (a guard missing in one copy, data loss,
  security gaps), and
- **Batch B — DRY consolidation** (no behavior change beyond what's called out), plus
- **Deferred, with a reason each**: high blast radius (god-file splits), needs a policy
  decision from the user, or needs live data/devices to validate.

## Step 3: Design and get approval

**Lean:** you already hold full context — write the plan yourself. **Thorough:** use the
Plan-agent flow in [thorough.md](thorough.md), then spot-verify its output yourself.

The plan must contain: a **Context** section (why this pass, the baseline), the in-scope
findings ranked (each with file:line, failure scenario, concrete fix), the deferred list with
reasons, the files touched, and a verification section with the project's own gates in their
documented order. If Plan Mode is active, write it to the plan file and use `ExitPlanMode`;
otherwise present it and get explicit confirmation. Do not implement before approval.

## Step 4: Implement — shared code first, then call sites

1. Create all new shared files first (constants, validation gate, pure decision functions,
   shared builders). Pure functions take plain values in and return decisions out — callers
   execute the decision their own way, so caller-specific mechanics stay put.
2. Convert call sites **one file at a time**, applying all changes to that file in one pass.
3. Preserve exact semantics on mechanical moves: key strings byte-identical, same defaults,
   same branch order, same call order — except the deliberate changes the plan lists.
   **Response shapes count too:** when an extracted helper's result replaces a value the
   caller serializes to a client (HTTP/MCP JSON, CLI output, an event payload), compare the
   serialized shape before and after — a helper that returns a "cleaner" object can silently
   drop a field an external client reads.
4. **Bug-class sibling sweep, per fix.** Every time you fix a real bug, name its pattern in
   one line (e.g. "date formatted without a pinned timezone", "fetch result used without an
   `ok` check", "read-then-rewrite of an array field") and grep the whole repo for that
   pattern — by value/shape, not just by the symbol you edited. Each hit is either fixed now
   or added to the deferred list with a reason. The audit only names the copies it happened
   to find; the pattern tells you where the rest are.
5. Run the compile/type-check gate after the sweep, before writing tests.

## Step 5: Tests for extracted logic

Match the repo's existing test conventions (runner, file layout, mocking style). Highest-value
tests: the full decision matrix of any extracted guard logic (loop over all combinations),
"inconsistent state still does something sane" cases, and any deliberate ordering decision
(encode it so drift fails a test). When a fix changes an existing test's mocking boundary
(e.g. a call site now goes through a use case), move the mock to the new boundary rather
than mocking deeper internals. Confirm the new tests actually ran from the runner's own
pass/fail counts, not just a zero exit code.

## Step 6: Regression pass

Both modes:

1. Run the project's own documented gates in their documented order (cheapest first) and
   compare against the Step 1 baseline: zero new type/compile errors, zero new warnings, test
   counts up by exactly the tests you added, no previously passing test now failing. Don't
   report a baseline warning or failure as a regression.
2. **Bug-class sweep loop:** re-run every pattern grep from Step 4.4, plus a grep for every
   old name/literal you consolidated. Fix anything new, re-run, and repeat until every hit is
   either the new central definition, fixed, or on the deferred list. Only then move on.

**Thorough:** additionally run the two-track pass in [thorough.md](thorough.md).

## Step 7: Close out

Fix real regressions and refactor-introduced nits; leave the user's own concurrent edits
alone. Re-run the gates. Then:

- **Doc sweep:** grep the project's docs (CLAUDE.md, `docs/`, rule files) for every deleted
  file, renamed/removed symbol, and renamed event/key — deleted code tends to leave stale
  references behind. Update CLAUDE.md wherever it documented the old duplication as a
  convention.
- **Final report:** what changed (grouped by batch), deliberate behavior changes, the deferred
  list, pre-existing issues found but not fixed, and a manual verification checklist for
  paths the test suite can't cover.

Do not commit unless asked; follow the project's own staging/commit policy.

## Task

$ARGUMENTS
