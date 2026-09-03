---
name: bug-bounty
description: "Audit a codebase (or a scoped area of it) for real, substantiated bugs using parallel specialized agents, rigorously verify every finding before reporting it, then optionally plan and execute the fixes via parallel scoped implementation agents."
argument-hint: <path or module to audit, and/or a target bug count — optional, defaults to the whole repo>
allowed-tools: Agent, AskUserQuestion, Read, Edit, Write, Bash, Grep, Glob
disable-model-invocation: true
---

# Bug Bounty: Parallel Audit → Verify → Plan → Fix

Finds real bugs by fanning audit work out across parallel agents scoped to distinct
subsystems, then treats every raw finding as a *claim to be checked*, not a fact to report.
The two phases — audit and fix — are independently useful: stop after the audit if that's all
that was asked for, or continue into planning and parallel fix execution when asked to.

## Step 1: Scope the audit

Read the project's own conventions doc first (CLAUDE.md or equivalent) — it distinguishes
*deliberate*, documented-as-correct-looking-weird design from actual defects, and every
audit agent needs this context so it doesn't flag an intentional tradeoff as a bug.

Partition the target (whole repo, or the path given in the task) into natural subsystem
boundaries — e.g. core domain logic, security/authorization rules, server/auth endpoints,
page-level UI state, shared components, data-access/repository layer. Aim for 4-8 partitions
with little to no file overlap between them; this is what makes Step 2 parallelizable and,
later, Step 6's fixes parallelizable too.

## Step 2: Fan out parallel audit agents

Launch one `Agent` (fork or general-purpose) per partition, all in a single message so they
run concurrently. Each agent's brief must include:

- The exact files/directories it owns.
- An instruction to actually open and read the code, not skim filenames or infer from
  structure.
- The documented intentional patterns from Step 1 that apply to its area, so it doesn't
  false-positive on them.
- A demand for **concrete failure scenarios**: `file:line — what input/state triggers what
  wrong output`, not style nits, hypotheticals, or "this could theoretically..." speculation.
- Instruction to check existing tests for the area — a gap in test coverage is often exactly
  where a real bug hides, and an existing test that already covers the concern rules it out.

## Step 3: Verify every finding — never trust raw agent output

This is the step that separates a real bug list from a padded one. For every finding:

1. If an `advisor`/second-opinion tool is available, call it with the full raw finding set
   before doing anything else. Its job is to catch overclaiming — a finding that conflicts
   with something already documented as intentional, a "latent risk" reported as if live, a
   missing-check finding where a broader gate actually already covers it elsewhere, or a
   count that was hit by *counting* rather than *verifying*.
2. Personally re-read the actual current code for every finding advisor flagged as
   questionable, and for anything security/authorization-related regardless of what advisor
   said. Read the *surrounding* code too (sibling rules, sibling functions) — a missing check
   is only real if there's no broader mechanism elsewhere that already covers it.
3. Run the project's type-check and full test suite now, even though no code has changed yet
   — a fresh, still-failing check or test is a free, already-confirmed bug that cost nothing
   to find. A clean pass here doesn't clear anything else; it just means don't expect free
   wins from this source.
4. For anything you can decide empirically (a query needing a database/emulator, a
   parity/format check across two files, a header-parsing edge case), actually run it rather
   than reasoning about it in the abstract.

## Step 4: Report — split Confirmed from Hardening/lower-confidence

Never present a single flat numbered list that mixes verified defects with speculative or
low-severity ones just to hit a target count. Use two buckets:

- **Confirmed bugs**: survived Step 3's re-verification, with a real, current, concrete
  failure scenario.
- **Hardening / consistency notes**: real gaps but latent, low-severity, or requiring
  specific conditions not yet shown to be reachable — say so explicitly, don't dress them up
  as live exploits.

Also state what you checked and found clean (a parity check, a rule that turned out to have a
broader existing gate) — a "nothing else found here" claim is only credible when it names what
was checked and why it came back clean. Save the full list to a durable file (not just chat
output) before treating the audit as done — this is what Step 5 continues from, in a fresh
session if needed.

If the task was audit-only, stop here.

## Step 5: Plan the fixes (only if asked to fix)

Enter planning mode (the harness's native plan mode if available; otherwise a plain written
plan needing explicit approval before any edit):

1. Launch `Explore` agents (grouped the same way as Step 2's partitions) to pull the *exact*
   current code for every confirmed finding plus any existing helper/pattern in the codebase
   that a fix should reuse — a plan built on remembered snippets from Step 2/3 drifts from
   what's actually on disk by the time fixes are written.
2. For the highest-risk or most-subtle findings (security rules, anything with more than one
   plausible fix shape), launch a `Plan` agent and require it to validate its proposed fix
   against **every real call site** in the codebase, not just the one that triggered the
   finding — a fix that's correct for the reported case but breaks a sibling call site is a
   regression, not a fix.
3. Some findings won't have a clean, contained fix — the real fix needs a new endpoint, a
   schema change, or a behavior decision with tradeoffs. Don't quietly downgrade or force a
   partial fix; use `AskUserQuestion` to present the tradeoff and let the user decide, then
   fold the answer into the plan as its own work package.
4. Group the fixes into work packages with **disjoint file sets** — this is what lets Step 6
   run them in parallel. Note the one thing solid-single-agent execution can't: if two
   findings can only be fixed by touching the same file, they belong to the *same* package
   (sequenced internally), or to packages explicitly ordered one-after-the-other — never to
   two packages meant to run concurrently.
5. Write the plan (context, each finding's exact fix, the package split, ordering
   constraints, and an end-to-end verification section) and get explicit approval before any
   code changes.

## Step 6: Execute — one agent per package, respecting ordering

Dispatch every package with no ordering dependency in one message so they run concurrently.
Each dispatch must tell the agent, explicitly:

- The exact file list it owns.
- That other named packages are editing other, disjoint files in this same working tree
  concurrently — so it knows not to "helpfully" touch something outside its lane, and knows
  a type-check failure appearing in a file it doesn't own is not its problem to fix.
- To run the project's type-check/test (filtered to what it touched) itself before reporting
  back, so regressions are caught immediately rather than discovered later in aggregate.

Hold any package with an ordering dependency until its prerequisite package's completion
notification has actually arrived — don't launch it speculatively. When a completed package's
report flags a same-bug-class instance outside its declared scope, note it as a follow-up for
the final report; don't have it (or a different agent) fix it unprompted.

## Step 7: Final verification and report

After every package lands: run the full repo-wide type-check and test suite yourself (not
trusting the per-package filtered runs alone), check `git status` for the complete set of
changed/new files, and run lint. If lint fails, distinguish pre-existing failures from
regressions by re-running it against a `git stash` (with `-u`) baseline before attributing
anything to this work — restore the stash immediately after.

Final report: what changed per package, any deliberate scope decisions an implementing agent
made on its own judgment, anything explicitly skipped and why (e.g. a suggested fix that would
itself have introduced a regression), and any out-of-scope follow-ups surfaced along the way.
Do not commit unless asked.

## Task

$ARGUMENTS
