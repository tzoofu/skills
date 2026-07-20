---
name: solid-audit-refactor
description: "Audit a codebase (or a scoped area of it) against SOLID/DRY and its own documented conventions, implement a user-scoped refactor of the confirmed violations, and verify with the project's own gates or a two-track adversarial regression pass."
argument-hint: <path or module to audit — optional, defaults to the whole repo>
allowed-tools: Agent, AskUserQuestion, Read, Edit, Write, Bash, Grep, Glob
disable-model-invocation: true
---

# SOLID Audit → Pragmatic Refactor → Regression Pass

Audits the target codebase for SOLID/DRY violations with concrete evidence, lets the user
choose how aggressive the fix should be, implements a minimal-diff refactor that respects the
project's documented conventions, and closes with a regression review. Designed for
codebases that are functionally solid but grew duplication and god classes organically.

## Step 0: Choose audit mode

Ask (`AskUserQuestion`) — skip this ask if the user's request already implies one (e.g. "quick
pass over X" → Lean; "full SOLID audit" / whole unfamiliar repo / high-risk area → Thorough):

- **Lean** (recommended for a scoped area, a codebase you already know well, or a fast
  "tidy this up" ask): single audit agent, you personally verify every finding, skip the
  separate Plan-agent design step, verify with the project's own build/lint/test gates.
- **Thorough** (recommended for a whole-codebase sweep, an unfamiliar repo, or a high-risk
  change): parallel Explore agents by layer, an explicit refactor-depth choice, a dedicated
  Plan agent for design, and a two-track adversarial regression pass at the end.

Both modes share the same backbone (audit → verify → plan → approve → implement → test →
regress → close out) and the same non-negotiables: ground the audit in the project's own
documented conventions, never trust a finding you haven't personally confirmed against the
current source, and never implement before the user has approved a concrete plan.

## Step 1: Evidence-gathering audit

Read the project's CLAUDE.md (or equivalent docs) first — it distinguishes *deliberate*
conventions from accidental debt, and gives the audit something concrete to check drift
against (a documented "always route through X" rule is a much stronger finding than a generic
DRY nit).

**Thorough:** launch 1–3 `Explore` agents in one message, split by layer (e.g.
services/managers vs UI/entry-points). **Lean:** launch a single `code-reviewer` (or
`Explore`) agent scoped to the target area, asked to ground findings in the project's own
conventions and rank them most-important-first (8–15 substantive findings, not an exhaustive
nitpick list). Either way, each agent must report with `file:line`:

- **God classes**: classes mixing persistence + business logic + system callbacks + UI.
- **Duplicated logic**: the same function/check copy-pasted across files — and critically,
  whether the copies have **diverged** (a missing guard in one copy is a real bug, not style).
  For anything reported as a duplicate, search by **value/content, not just symbol name**, and
  outside the obvious layer too — a fourth copy often hides in a utility/renderer/script dir
  under a different name.
- **Scattered constants**: shared string keys/IDs re-declared or re-typed as literals per file
  (SharedPreferences keys, notification IDs, action strings, env names).
- **Dependency shape**: what each class constructs inline vs receives; existing abstractions.
- **Test coverage**: what is/isn't unit-testable and why.

Then — **both modes, non-skippable** — personally re-read every finding's actual file before
building on it. An agent's summary describes what it *believes* it found, not verified ground
truth; a finding whose line number or exact behavior doesn't match the current source doesn't
survive into the plan. Drop or correct anything that doesn't hold up under a real read.

While agents run, get project scale yourself (`find ... | xargs wc -l | sort -rn`), and
**capture the clean-tree baseline now, before any edit**: save lint/type-check/build output
(error list, warning count) to a scratch file. Every later "did I break this?" diff is only
meaningful against this snapshot — capturing it after edits begin is too late. (Lean mode on
a small scoped area can substitute the project's own already-documented zero-warning/expected-
warnings policy for this baseline, if one exists — no need to hand-capture what's already
written down.)

## Step 2: Ask the user for depth (Thorough) / go straight to planning (Lean)

**Thorough:** use `AskUserQuestion` with these standard options (recommend the first for
small/solo projects):

1. **Pragmatic cleanup** — centralize constants, extract shared helpers/pure decision
   functions, fix behavioral divergences. No new interfaces or DI framework.
2. **Medium restructure** — pragmatic + split god classes, repository abstraction.
3. **Full architecture** — interfaces everywhere, DI framework, use-case layer.

Also ask whether to add unit tests for newly extracted pure logic. Warn when full SOLID would
be over-engineering for the codebase size.

**Lean:** skip this ask — verified findings are already concrete, scoped fixes (not an
architecture choice), so proceed straight to Step 3 with all of them included, ordered
cheapest/lowest-risk first.

## Step 3: Design and get approval

**Thorough:** launch a `Plan` agent (see below), then spot-verify its output yourself.
**Lean:** you already hold full context from Step 1 — write the plan directly rather than
dispatching a separate agent for it. Either way the plan must contain: a **Context** section
(why this pass, what prompted it), the verified findings ranked most-important-first (each
with file:line, the failure scenario, and a concrete fix), the exact files touched, and a
verification section listing the project's own build/test/lint commands in their documented
order. If the harness's Plan Mode is active, write this to the plan file and use its native
approval flow (`ExitPlanMode`); otherwise present the plan as text and get explicit
`AskUserQuestion`/confirmation before touching any code. Do not implement before approval.

### Thorough: Plan-agent design detail

Launch a `Plan` agent with: the audit findings (verbatim, with line numbers), the user's depth
choice, and the explicit constraint list of conventions that must NOT change. Require from it:
new file names + API sketches, per-file call-site changes, deliberate behavior changes called
out separately, edge-case "must-not-change" checklist, test cases, verification commands.

When splitting design across **multiple Plan agents**, assign cross-cutting infrastructure
(test-runner setup, tooling tasks, the shared-constants module's home) to exactly one agent
and tell the others to reference it — otherwise both spec it and you reconcile by hand.
If an agent dies mid-run (session limit, API error), **resume it** rather than relaunching:
its partial verification work survives; a fresh launch repeats it.

Before finalizing: **read the riskiest call sites yourself** to confirm the plan's claims
(guards it says are missing, ordering quirks it says exist), and **byte-compare each "copy"
slated for unification** — a lookalike whose values genuinely differ (a denser variant, a
different cap) is not a copy; keep it local and record it as a deliberate non-unification
rather than force-unifying or silently changing it. If the repo changed mid-planning
(`git log`), re-check the plan against the new commits and patch it.

## Step 4: Implement — shared code first, then call sites

1. Create all new shared files first (constants object, permission/validation gate, pure
   decision functions as `internal` for testability, shared builders). Pure functions take
   plain values in and return sealed-type decisions out — callers execute the decision their
   own way, so caller-specific mechanics (which start method, optimistic writes) stay put.
2. Convert call sites **one file at a time**, applying all changes to that file in one pass.
3. Preserve exact semantics on mechanical moves: key strings byte-identical, same defaults,
   same branch order — except the deliberate changes the plan lists.
4. Compile after the sweep (`compileDebugKotlin` / `tsc` / equivalent) before writing tests.

## Step 5: Tests for extracted logic

Match the repo's existing test conventions. Highest-value test: the full decision matrix of
any extracted guard logic (loop over all boolean combinations), including "inconsistent state
still does something sane" cases and any deliberate ordering decisions (encode them so drift
fails a test). Run the whole suite and confirm the new classes actually executed
(check test-results XML counts, not just BUILD SUCCESSFUL).

## Step 6: Regression pass

**Lean:** run the project's own documented verification gates, in their documented order
(cheapest first — e.g. a project's own `build.md`/CLAUDE.md rule file often specifies this
exactly), and confirm real pass/fail from the actual output — test-result XML counts or
equivalent, not just a zero exit code or "BUILD SUCCESSFUL." Cross-check the project's own
list of already-expected/documented warnings so you don't misreport a known, accepted warning
as a new regression. This substitutes for Track A/B below when the change is small and scoped
enough that a full adversarial pass would be overkill; escalate to Thorough's two tracks if
anything in Step 1 hinted at wider blast radius than expected.

**Thorough (two independent tracks, in parallel):**

**Track A — adversarial review agent** (`code-reviewer` or similar, read-only): give it the
exact refactor intent, the list of deliberate behavior changes (so it doesn't flag them), and
category-by-category hunting instructions: key-spelling drift vs `git show HEAD:`, semantic
drift in extracted helpers, lost call-site behavior line-by-line, edge semantics of new guards,
missed call sites, unused imports. Tell it to be adversarial and report file:line + severity.

**Track B — your own mechanical checks**:
- Byte-compare every migrated constant value against `git show HEAD:<file>` in a shell loop.
- Grep-sweep for leftover literals/old const names/duplicate checks — expect hits only in the
  new central file. Sweep by **value** too (the alphabet string, the color pair, the magic
  number), repo-wide: name-based greps miss copies that renamed the symbol.
- Full recompile with `--rerun-tasks`; require zero **new** warnings vs the Step 1 baseline.
- For any newly added guard, verify UI reachability: is the new "blocked" branch actually
  reachable, or already gated upstream? (Prevents silently stranding a user path.)
- **"Behavior-preserving" also preserves latent bugs.** For every moved/extracted write that
  must satisfy an external validator (DB security rules, schema constraints, API contracts),
  check it against the *deployed* validator config and against legacy/edge data shapes
  (absent fields, empty arrays — e.g. an `arrayRemove` on a field older docs never had).
  Report what you find as **pre-existing findings**, don't silently carry it forward: the
  refactor gets blamed for whatever breaks next, even bugs it faithfully preserved.

## Step 7: Fix findings and close out

Fix real regressions and refactor-introduced nits; leave the user's own concurrent edits
alone. Re-run build + tests. Update CLAUDE.md wherever it documented the old duplication as
convention, and update any other doc the change made stale (e.g. a hard-coded test-count
comment in a build rule file). Final report: what changed, deliberate behavior changes, what
was deliberately NOT done, and a manual/on-device (or otherwise untestable-by-CI) verification
checklist for paths the test suite can't cover. Do not commit unless asked.

## Task

$ARGUMENTS
