# Thorough mode — extra steps

Load this only in Thorough mode. It adds a refactor-depth choice, a dedicated Plan agent,
and a two-track adversarial regression pass on top of the shared backbone in SKILL.md.

## Contents
- Depth choice (Step 2)
- Plan-agent design (Step 3)
- Two-track regression pass (Step 6)

## Depth choice (Step 2)

Use `AskUserQuestion` with these standard options (recommend the first for small/solo
projects):

1. **Pragmatic cleanup** — centralize constants, extract shared helpers/pure decision
   functions, fix behavioral divergences. No new interfaces or DI framework.
2. **Medium restructure** — pragmatic + split god classes, repository abstraction.
3. **Full architecture** — interfaces everywhere, DI framework, use-case layer.

Also ask whether to add unit tests for newly extracted pure logic. Warn when full SOLID would
be over-engineering for the codebase size.

## Plan-agent design (Step 3)

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

## Two-track regression pass (Step 6)

Run both tracks in parallel, in addition to the shared gates and bug-class sweep in SKILL.md.

**Track A — adversarial review agent** (`code-reviewer` or similar, read-only): give it the
exact refactor intent, the list of deliberate behavior changes (so it doesn't flag them), and
category-by-category hunting instructions: key-spelling drift vs `git show HEAD:`, semantic
drift in extracted helpers, lost call-site behavior line-by-line, changed response/return
shapes, edge semantics of new guards, missed call sites, unused imports. Tell it to be
adversarial and report file:line + severity.

**Track B — your own mechanical checks**:
- Byte-compare every migrated constant value against `git show HEAD:<file>` in a shell loop.
- Grep-sweep for leftover literals/old const names/duplicate checks — expect hits only in the
  new central file. Sweep by **value** too (the alphabet string, the color pair, the magic
  number), repo-wide: name-based greps miss copies that renamed the symbol.
- Full clean rebuild (bypass incremental caches — e.g. Gradle `--rerun-tasks`); require zero
  **new** warnings vs the Step 1 baseline.
- For any newly added guard, verify UI reachability: is the new "blocked" branch actually
  reachable, or already gated upstream? (Prevents silently stranding a user path.)
- **"Behavior-preserving" also preserves latent bugs.** For every moved/extracted write that
  must satisfy an external validator (DB security rules, schema constraints, API contracts),
  check it against the *deployed* validator config and against legacy/edge data shapes
  (absent fields, empty arrays — e.g. an `arrayRemove` on a field older docs never had).
  Report what you find as **pre-existing findings**, don't silently carry it forward: the
  refactor gets blamed for whatever breaks next, even bugs it faithfully preserved.
