---
name: ui-bug-sweep
description: "Given a report that some UI control (\"the X button doesn't work\") is broken, find its true root cause, generalize it into a precise bug signature, sweep the whole codebase for that same signature, fix every instance, and verify live in a real browser."
argument-hint: <description of the reported-broken control>
allowed-tools: Task, Read, Edit, Write, Grep, Glob, Bash, mcp__playwright__browser_navigate, mcp__playwright__browser_click, mcp__playwright__browser_snapshot, mcp__playwright__browser_resize, mcp__playwright__browser_tabs
disable-model-invocation: true
---

# UI Bug Sweep

A user reports a control that doesn't do what it should ("the top login button doesn't work", "the save button on X does nothing"). The naive fix patches that one spot. This skill instead treats the report as evidence of a *bug class*, fixes every instance of that class, and proves the fix works by driving a real browser — not just by passing a type-check.

## Task

$ARGUMENTS

---

## Step 1: Find the real root cause

Don't guess from the symptom. Read the actual implementation of the reported control:

- If the scope is uncertain or spans multiple files/components, delegate to an `Explore` agent (via the `Task` tool) with a **specific** brief: find the exact component, its handler/href, and quote the surrounding code. Ask it to also check for the obvious suspects — missing `onClick`/`href`, disabled state, event propagation stopped, wrong/self-referential navigation target, stale condition — but don't let it stop at "clean sweep" before you've named the bug class (see Step 2).
- Confirm with `git log -p` on the file whether this is a recent regression or has been latent since introduction — it changes how urgently related code needs auditing.

## Step 2: State the bug as a precise, generalized signature

Do not write "button X is broken." Write the **shape** of the defect, e.g.:

> "A control whose label implies performing an action (login, save, submit, report, add) but which is implemented as navigation to a route — where that route can equal the route the control is already rendered on, making the click a no-op."

A vague signature ("check if buttons are broken") produces a sweep that clears everything and misses the real pattern — this is the single most common failure mode of this skill. Be as mechanically specific as the actual bug: name the exact code shape (a hook call, a prop, a comparison), not just the symptom.

## Step 3: Consult advisor before committing to the sweep

If an `advisor`/second-opinion tool is available, call it now, before running the codebase-wide sweep. Its job is to catch a signature that's too narrow (misses siblings of the same bug) or too broad (turns into a generic "any button that could look weird" review that reports false positives). Adjust the signature based on its answer before proceeding — don't sweep on a signature you haven't sanity-checked.

## Step 4: Sweep the codebase for the corrected signature

Grep/Explore for every occurrence of the pattern named in Step 2 (not just the component family the original report came from). For each candidate, explicitly record the verdict — hit, or "checked, clean, because ___". A report that only lists positives can't be trusted; the "nothing else found" conclusion is only credible when it names what was checked and why it doesn't match.

Watch for the trap of running this sweep in parallel with Step 1 before the signature is known — a generic "is any button dead" sweep answers a different, weaker question than "does this same specific bug shape exist elsewhere," and will confidently report false negatives.

**When findings conflict, don't average them — verify.** If sweeping via multiple agents or passes and two sources disagree on whether a candidate matches the signature, read the file yourself and settle it before deciding to fix or skip it. A wrong "hit" wastes a fix; a wrong "clean" leaves the bug live.

## Step 5: Fix

- If the same fix needs to apply in more than one place (e.g. a nav copy of a hero button, desktop + mobile variants), extract the shared logic (a hook, a helper) rather than patching each call site by hand — match however this codebase already shares logic across similar call sites (grep for an existing analogous pattern first).
- Move/extract code **verbatim** where possible in the same step you extract it; don't refactor-while-moving the one thing that currently works.
- Keep the fix scoped to the named signature — don't opportunistically refactor unrelated code nearby.

## Step 6: Type-check and run tests

Run the project's type-check and test commands (check `CLAUDE.md`/`package.json` for the exact ones — e.g. `tsc --noEmit`, `npm test`). This proves the code compiles and doesn't regress existing coverage — it does **not** prove the UI behavior is fixed.

## Step 7: Verify live, in a real browser

A passing test suite doesn't confirm the button now does the right thing. Start the project's dev server, then use the Playwright MCP tools to drive it.

**Check for an auth gate first.** If the affected page requires login, get a valid session before attempting any navigation — ask the user for a way in (a session cookie from an already-logged-in tab, a test account, or a saved auth-state file) rather than discovering the wall mid-verification and stalling the whole step.

1. `browser_navigate` to the affected page(s).
2. `browser_snapshot` to confirm the control's accessible role/label matches the fix (e.g. it's now a `button` with an `onClick`, not a dead `link`).
3. `browser_click` it — **drive the fix through the actual control, not a shortcut to its destination.** For a bug that's specifically about *how* a control navigates (client-side `Link` vs. full navigation, `router.back()` vs. a fresh href, etc.), jumping straight to the destination URL with `browser_navigate` bypasses the exact mechanism under test and can pass even when the real control is still broken. Use direct URL navigation only for setup or as a supplementary check — never as the primary confirmation. Confirm the **actual expected side effect** happens (a popup opens, a request fires, state visibly changes) — not just "no console error."
4. If the control has responsive variants (a mobile drawer copy, a different breakpoint layout), `browser_resize` and repeat the click there too.
5. Re-verify any button whose *code path* you touched during extraction (e.g. the original hero/reference implementation), even if the report wasn't about it — shared-logic extraction risks breaking the thing that already worked.
6. **If you edit code during this step, don't re-test immediately.** Dev servers using Fast Refresh/HMR take a moment to apply a change — testing too soon silently exercises the stale bundle and produces a false "still broken" result. Confirm the rebuild actually finished (a `[Fast Refresh] done` console entry, or a fresh compile line) before trusting the next test, or force a hard reload to be certain.
7. **If live behavior contradicts code that type-checks and reads correctly, don't re-derive the fix by reading harder.** Add temporary `console.log` statements at each relevant effect/branch, reproduce the exact same steps, and read the actual execution order and values from the browser console. This surfaces hidden second-order causes (e.g. unrelated shared state being reset by something else entirely) far faster than re-theorizing from source. Remove the instrumentation once the real cause is confirmed and fixed.
8. Clean up: close any extra tabs/popups the flow opened, stop the dev server.

## Step 8: Report

Summarize: the confirmed root cause, the generalized signature, what the sweep found (including explicitly "checked, clean" areas), the fix (files touched, and whether logic was extracted/shared), and what was verified live vs. only type-checked.
