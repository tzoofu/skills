---
name: spec-to-monorepo
description: "Takes a rough AI-generated spec or plan and turns it into a working, secure, documented multi-service monorepo — architecture review, service scaffolding, cross-service wiring, Snyk scanning, and a project README + CLAUDE.md with real gotchas."
argument-hint: <spec-path-or-text> <target-dir>
disable-model-invocation: true
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, mcp__claude_ai_Context7__resolve-library-id, mcp__claude_ai_Context7__query-docs, mcp__Snyk__snyk_code_scan, mcp__Snyk__snyk_sca_scan
---

# spec-to-monorepo

Turn a rough spec (from Gemini, ChatGPT, a doc, or a napkin sketch) into a running multi-service monorepo. Covers architecture review, service scaffolding, wiring, secrets hardening, Snyk security scanning, and documentation.

## Task

```
<spec>     — the plan text or path to a file containing it
<target>   — directory to scaffold into (must exist or be creatable)
```

---

## Step 1 — Ingest the spec

Read the spec in full. Extract:
- Services and their responsibilities
- Data flow between services (what crosses which boundary)
- Storage policy (what is persisted, what is ephemeral)
- Auth requirements (now or later)
- Network scope (local-only vs internet-facing)
- Tech stack choices — note which are fixed vs open

Do not scaffold anything yet.

---

## Step 2 — Surface blockers first

Use `AskUserQuestion` for 2–4 architectural decisions that cascade downstream. Typical questions:

- "Persist audio, transcripts, or both?"
- "Local network only, or internet-accessible?"
- "Authentication now or later?"
- "Which framework for the BFF / frontend?" (if not specified)

Do not scaffold until these are locked. Wrong answers here ripple into every file.

---

## Step 3 — Lock the architecture

Finalize:
- Service names, ports, responsibilities
- What data crosses each service boundary and in what shape
- What is stored vs discarded after each request
- Which secrets each service needs

---

## Step 4 — Scaffold in dependency order

Build services from the inside out — no-upstream-dependency service first, then those that depend on it, frontend last.

For each service:
1. Create the directory and install dependencies
2. Use Context7 (`mcp__claude_ai_Context7__resolve-library-id` + `mcp__claude_ai_Context7__query-docs`) to fetch current framework docs — do not rely on training data for API shapes
3. Write config and entry point
4. Implement one working route end-to-end before moving to the next service

Typical order: **ML/inference service → BFF → frontend**

---

## Step 5 — Wire services together

- CORS headers: lock each service to accept requests only from the next hop (not `*`)
- Env vars: each service reads its upstream URL from an env var, never hardcoded
- Proxy routes: BFF forwards to inference service, frontend calls only the BFF
- Type shapes: define the JSON contract at each boundary explicitly

---

## Step 6 — Harden secrets and gitignore

Write `.gitignore` covering:
- Language-specific build/cache dirs (`venv/`, `node_modules/`, `.next/`)
- All `.env` files (real values)
- Any local databases or model caches

Write `.env.example` files (committed) with placeholder values for every secret. Real `.env` files are gitignored and never committed.

Secrets go only in `.env` files — never hardcoded, never in comments.

---

## Step 7 — Snyk scan loop

Run `mcp__Snyk__snyk_code_scan` on all first-party source files.
Run `mcp__Snyk__snyk_sca_scan` on each package manifest (`requirements.txt`, `package.json`, etc.).

For each finding:
- If a fix is available: apply it and rescan
- If no `fixedIn` version exists: document the accepted risk in a comment in the relevant file

Repeat until no new fixable issues remain.

---

## Step 8 — Write README.md

The README must explain everything a new user needs to configure before running the project. Include:

- What the project does (one paragraph)
- Architecture diagram (ASCII)
- Prerequisites (system deps, language versions, external accounts)
- **Configuration section** — step-by-step: where to get credentials, what files to create, what values to set, any one-time setup
- How to run each service
- Supported input formats / constraints

Write for someone who has never seen the codebase.

---

## Step 9 — Write CLAUDE.md

Document what future Claude Code sessions need to know that isn't obvious from the code:

- Commands to run each service
- One-time setup steps (model downloads, license acceptance pages, native module rebuilds)
- **Known Gotchas** — real issues discovered during this scaffolding session: API quirks, version constraints, browser behavior, anything that caused a bug

Do not repeat what is already in README.md. CLAUDE.md is for the AI; README.md is for the human.

---

## Step 10 — Verify the golden path

Trace one real request end-to-end:
1. Start all services
2. Submit a real input through the frontend
3. Confirm the response appears correctly in the UI
4. Confirm any persistence (DB rows, files) is correct
5. Confirm ephemeral data (temp files, audio) is cleaned up

Only declare done after the golden path passes.
