---
name: regression-check
description: "Given the current branch diff, produce a manual testing board and a Playwright test plan scoped to what changed. Run before /audit."
---

# Regression Check

Analyzes what changed in this session or branch and produces:
1. A **manual testing board** — exact steps to verify the change works and nothing regressed
2. A **Playwright test plan** — numbered scenarios ready to be turned into test code

## Step 0: Delegate to the agent (if available)

If the `regression-check` agent is installed (`.claude/agents/regression-check.md` in the current repo), delegate the entire workflow to it via the `Task` tool with the inputs below. The agent returns a single report — print it. Skip the remaining steps.

If the agent is NOT available, execute the steps below inline.

## Step 1: Get the diff

```bash
git diff origin/dev...HEAD --name-only   # changed files vs dev
git diff origin/dev...HEAD --stat        # summary
```

If on dev branch with staged changes, use `git diff --staged --name-only` instead.

## Step 2: Map files to flows

Use this knowledge map to identify affected flows:

| File pattern | Affected flows |
|---|---|
| `businessSettings.core.ts` / `.server.ts` / `.tsx` | Order form, admin settings, tracking page, all settings consumers |
| `api/admin/settings/` | Admin settings save + cache bust |
| `api/orders/` | Order submission, flavor/payment/cutoff validation |
| `FlavorTile.tsx` | Flavor display everywhere (order form, admin, tracking, hero) |
| `OrderRulesSettings.tsx` | Admin settings UI (flavor catalog, cutoffs, payment toggles) |
| `PackageCard.tsx` | Order form flavor selection |
| `OrderForm.tsx` | Full order wizard (all 3 steps + submit) |
| `order/[slug]/TrackingView.tsx` | Customer tracking page |
| `admin/orders/[orderId]/` | Admin order detail + status buttons |
| `admin/page.tsx` | Admin dashboard + live order feed |
| `status.ts` | Status state machine, forward/rollback transitions |
| `schemas/order.ts` | Order form validation + API defense |
| `pricing.ts` | Package pricing, flavor limits |
| `orderCutoffs.ts` | Delivery day availability + cutoff labels |
| `notifications.ts` / `useNewOrderAlerts.ts` | Admin sound + OS notification alerts |

## Step 3: Identify regression paths

For each affected flow, list what ELSE could break — shared helpers, downstream consumers, API contracts.

## Step 4: Generate Manual Testing Board

Output a markdown checklist grouped by area:

```
## Manual Testing Board

### [Area name]
- [ ] [URL or entry point] → [Action] → [Expected result]
- [ ] ...

### Regression checks
- [ ] ...
```

Be specific: include exact URLs (`/order`, `/admin`, `/admin/settings`), button labels, and observable outcomes.

## Step 5: Generate Playwright Test Plan

Output numbered scenarios:

```
## Playwright Test Plan

### Scenario N: [Name]
Entry: [URL]
Steps:
  1. [action] → [expected]
  2. ...
Key assertions: [what must be true at the end]
```

Cover both the happy path and the most likely regression (e.g. if flavor validation changed, include a scenario where an invalid flavor is submitted).

## Task

$ARGUMENTS
