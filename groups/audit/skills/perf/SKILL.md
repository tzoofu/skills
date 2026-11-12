---
name: perf
description: "Audit a web or mobile app for speed and perceived performance against a fixed, deduplicated checklist and output a prioritized markdown plan with code snippets. Works on a repo or on a short project description. Use instead of ad-hoc perf advice for a consistent report with overkill guardrails; for Next.js on Vercel also consider the vercel:performance-optimizer agent."
argument-hint: <project path or short description — optional>
allowed-tools: Bash, Read, Grep, Glob, Agent, mcp__datadog-mcp__search_datadog_rum_events, mcp__datadog-mcp__aggregate_rum_events, mcp__plugin_vercel_vercel__aggregate_pageviews, mcp__plugin_vercel_vercel__get_observability_schema, mcp__plugin_vercel_vercel__create_observability_query, mcp__expo-mcp__observe_metrics, mcp__expo-mcp__observe_metrics_summary
disable-model-invocation: true
---

# Performance Audit

Audit a web or mobile project against the checklist below and produce a prioritized plan. Never invent numbers: every gain is an estimate tied to a measured baseline, or marked `unmeasured`.

## Task

$ARGUMENTS

## Step 1: Detect input mode and stack

- **Repo present** (path given, or cwd has `package.json` / `app.json` / `pyproject.toml`): detect the stack from files (Next.js, React SPA, Node/Express/Nest, Expo/React Native, ORM such as Prisma/TypeORM/Mongoose, cache, CDN config).
- **Description only**: ask the user for the stack and current metrics, then audit from their answers. Mark findings as `from description, not verified in code`.
- **Web vs mobile**: web targets are LCP < 2.5s, TTFB < 200ms, INP < 200ms, CLS < 0.1. For React Native / Expo use cold start + TTI, JS bundle size, `FlatList`/`FlashList` virtualization, Hermes, image caching instead.

## Step 2: Measure first

Look for real numbers before asking the user, in this order:
1. Datadog RUM, Vercel Speed Insights / analytics, Expo observe metrics (the MCP tools in `allowed-tools`).
2. `web-vitals` or Lighthouse CI already wired in the repo.
3. `npx lighthouse <url> --only-categories=performance --output=json` if a URL is reachable.
4. Ask the user for LCP / TTFB only if none of the above works.

Record the baseline. If nothing is measurable, say so in the report and make "add measurement" the first action.

## Step 3: Scan the code

For a repo, delegate the heavy reading to one read-only subagent (`Explore` or general-purpose) with an inline brief: the detected stack, the checklist IDs from Step 4 that apply, and "return a findings list of `ID | file:line | evidence`, no file dumps". Skip groups that do not apply (no DB checks for a static site, no SSR for a pure API).

## Step 4: Checklist (about 29 checks, deduplicated)

**UX / perceived speed**
- U1 Skeleton screens for async content
- U2 Optimistic UI for low-risk mutations
- U3 Streaming for LLM responses
- U4 Background jobs for genuinely heavy work (report generation, video, bulk import)
- U5 Debounce search input (about 300ms)

**Assets**
- A1 WebP/AVIF instead of PNG/JPEG
- A2 Lazy-load below-the-fold images; priority/preload for the LCP image
- A3 `width`/`height` (or aspect-ratio) on images
- A4 CDN for static assets
- A5 Brotli/gzip on text responses
- A6 Only the font weights actually used; `font-display: swap`

**Network / API**
- N1 HTTP/2 or HTTP/3
- N2 Pagination (about 20 per page)
- N3 Independent requests in parallel (`Promise.all`), no waterfalls
- N4 Select only needed columns / fields
- N5 Client timeout (about 3s) with `AbortController`
- N6 Code splitting by route, dynamic import for heavy components
- N7 Defer or lazy-load third-party scripts

**DB / cache**
- D1 N+1 queries
- D2 Indexes on filtered / sorted columns
- D3 Redis (or equivalent) for hot, repeated reads
- D4 Connection pooling
- D5 `Cache-Control` (immutable for hashed statics, stale-while-revalidate for data)

**Rendering**
- R1 SSR / ISR / static generation for critical pages
- R2 Virtualize lists over about 100 rows
- R3 Above-the-fold content first, no render-blocking CSS/JS

**Metrics**
- M1 Web Vitals reporting / OpenTelemetry traces
- M2 Real User Monitoring
- M3 Tested on throttled slow 3G and a low-end device

## Step 5: Overkill guardrails

Do not recommend these unless the evidence demands it, and say why when you do:
- Timeouts under 1s on non-critical APIs
- Redis for everything (only hot, expensive, repeated reads)
- Streaming for non-AI responses
- Background jobs for simple, fast tasks
- Migrating to SSR/ISR when the page is already fast or auth-gated

## Step 6: Report

Output one markdown report, ranked by impact on LCP / TTFB / perceived speed:

```
## Baseline
LCP x / TTFB y / source (or "unmeasured")

## Critical (LCP / TTFB)
1. <finding> — `file:line` — <why it hurts>
   Estimated gain: <range, label "estimate"> | `<snippet>`

## High-impact UX
...

## Technical debt
...

## Skipped (not applicable / overkill)
- <ID>: <reason>
```

Rules for each item:
- Cite the evidence (`file:line`) or mark it as from description.
- Include a minimal, stack-correct code snippet (e.g. `next/image` with `priority`, Prisma `include`/`select` to fix N+1, `AbortController` with a 3s timeout, `FlashList`).
- Express gains as ranges with units (ms, KB, queries) and the assumption behind them. Without a baseline write `unmeasured` instead of a number.
- End by offering to implement the top items; do not edit code during the audit.
