---
description: Turn a rough AI-generated spec into a working, secure, documented multi-service monorepo with architecture review, scaffolding, cross-service wiring, Snyk scanning, and README + CLAUDE.md.
argument-hint: <spec-path-or-text> <target-dir>
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep, mcp__claude_ai_Context7__resolve-library-id, mcp__claude_ai_Context7__query-docs, mcp__Snyk__snyk_code_scan, mcp__Snyk__snyk_sca_scan]
---

# spec-to-monorepo

Takes a rough spec and produces a running multi-service monorepo with security hardening and full documentation.

## Steps

1. Ingest spec — extract services, data flow, storage, auth, tech stack
2. Ask 2–4 architectural blocking questions before scaffolding
3. Scaffold services in dependency order, using Context7 for current docs
4. Wire services — CORS locks, env-var URLs, explicit JSON contracts
5. Harden secrets — `.gitignore`, `.env.example`, no hardcoded values
6. Snyk scan loop — fix all fixable findings, document accepted risks
7. Write README.md (human-facing) and CLAUDE.md (AI-facing, gotchas)
8. Verify golden path end-to-end before declaring done

## Arguments

$ARGUMENTS
