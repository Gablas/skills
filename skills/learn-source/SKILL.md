---
name: learn-source
description: >-
  Use when researching an external API, SaaS, or webhook-delivered source before
  integrating with it (e.g. Hubspot, Stripe, Fireflies, Slack, Attio).

  Triggered by /learn-source <name> or phrases like "research X api",
  "learn X api", "understand X api".

  Produces a ground-truth reference bundle at `docs/sources/<source>/` combining
  official docs, OpenAPI/SDK, community findings, and live sandbox probes (read
  and write) plus captured webhook payloads. Intended as a handoff artifact for
  the next agent who implements the integration — this skill does NOT write
  integration code, propose implementation plans, or inspect repo layout.
---

# learn-source

## Overview

Produce a ground-truth reference for one external source at `docs/sources/<source>/` by combining parallel research (docs, OpenAPI, SDK, community) with live sandbox probes (including webhook capture). Next agent reads the artifact and implements. Skill never writes integration code.

**Hard scope boundary:** external knowledge only. No code under `apps/` or `libs/`. No Nx project placement. No impl plans. If tempted, stop.

## When to Use

Use when:
- User invokes `/learn-source <name>`
- User says "research X api", "learn X api", "understand X api"
- Integrating a source with no current `docs/sources/<source>/` doc, or doc > 90 days old

Do NOT use when:
- Task is "implement X" / "add X endpoint" — skip research, go integrate
- Source already documented with recent `last_probed` and unchanged scope
- User wants unblocking, not thorough research

## Output Layout

```
docs/sources/<source>/
  README.md      # overview, auth, envs, pagination, rate-limits, completeness table, frontmatter
  endpoints.md   # per-endpoint request/response shapes
  webhooks.md    # event catalog, signing, captured real payloads
  quirks.md      # landmines, doc↔probe discrepancies
  schemas.ts     # Zod, importable, types derived via z.infer
  .skill-hash    # per-section hashes for re-run diffing (hidden)
```

README frontmatter:

```yaml
---
source: hubspot
api_version: v3
last_probed: 2026-04-24
probed_by: gabriel
---
```

## Completeness Table (core deliverable)

Every README opens with a table covering these categories, each in exactly one state:

| State | Meaning |
|-------|---------|
| `verified-probe` | Confirmed via live sandbox call |
| `verified-docs`  | Confirmed via official docs / OpenAPI / SDK source |
| `n/a`            | Source has no such concept (e.g. no webhooks) |
| `unresolved`     | Research incomplete |

**Categories (fixed):** `auth` · `environments` · `endpoints` · `pagination` · `rate-limits` · `webhooks` · `idempotency` · `bulk-batch` · `errors` · `quirks` · `sdk`

ReACT loop runs while `unresolved > 0`; cap at 3 iterations, then prompt user to accept or extend.

## Flow

### 1. Scope
Ask user (or confirm from slash args):
- Source name
- Which operations they plan to integrate (drives probe scope — skill does not exhaust the API)

### 2. Parallel research burst
Dispatch 4 subagents concurrently via the Task tool (general-purpose). Each returns a short report. Prompts below are the exact briefs.

**docs subagent:**
> Find the official API reference for `<source>`. Enumerate base URLs (prod + sandbox), auth mechanism (API key / OAuth / JWT / other), pagination style, rate limit policy, error shape, SDK availability (official only). For the use-case scope `<list>`, enumerate endpoints with request/response structure. Use WebFetch/WebSearch. Under 600 words. Return markdown with section per category.

**openapi subagent:**
> Search for an OpenAPI/Swagger/Postman spec for `<source>` (common locations: github.com/<vendor>, developers.<source>.com, apidog.com, postman.com/<vendor>). If found, extract: endpoint list for scope `<list>`, exact schemas, error codes. Return raw schema fragments, not prose. If no spec exists, say so in one line.

**sdk subagent:**
> Find official SDK repo for `<source>` (language: TypeScript/JavaScript preferred). Read types, client module, error classes. Extract: typed request/response interfaces, retry/idempotency helpers, pagination helpers, rate-limit handling. Return relevant type blocks + one-line notes on non-obvious patterns. Skip README marketing.

**community subagent:**
> Search GitHub issues on the official SDK repo and top 10 StackOverflow results for `<source> api gotchas`. Extract: undocumented behaviors, type coercion quirks (null vs missing vs "null"), field-name inconsistencies, rate-limit surprises, pagination edge cases, deprecations in flight. Return bullet list with source URL per item.

### 3. Synthesize + write initial files
Merge the four reports. Build draft completeness table (mark categories `verified-docs` or `unresolved`). Write all files — atomic post-synthesis write guarantees a stable doc even if probe phase aborts.

### 4. Live probe (interactive)

**Cred setup:**
- Check `.gitignore` contains `.env.probe.local`; add line if missing.
- If source uses **API key**: prompt user to paste sandbox key → write to `.env.probe.local` → install cleanup trap in skill's shell session:
  ```
  trap 'rm -f .env.probe.local' EXIT INT TERM HUP
  ```
- If source uses **OAuth**: walk user through dev-app creation (print exact URL + required redirect URI `http://localhost:<port>/callback`), start a minimal Node/Bun listener on a free port, open auth URL in browser, capture code on redirect, exchange for access token, write token to `.env.probe.local`.
- Refuse prod creds: flag if env var name lacks `SANDBOX`/`TEST`/`DEV` marker or if user-supplied key appears to be prod. Confirm before proceeding.

**Endpoint probes:**
- One sample call per resource type in scope (not every endpoint).
- Reads always executed. Writes executed against sandbox only, no cleanup.
- Record real response bodies. Update `endpoints.md` and `schemas.ts` from observed shapes.
- On conflict with docs: probe wins for `schemas.ts`, both versions logged in `quirks.md` with quotes.

**Webhook probes (if scope includes webhooks):**
- Detect tunnel tool: prefer `ngrok` (check `command -v ngrok`); fall back to `smee.io` if absent.
- Start local listener + tunnel. Register webhook in source sandbox UI (print exact click-path for user). Instruct user to fire test events. Capture 5 min max, user can interrupt sooner.
- Write captured payloads verbatim to `webhooks.md` (redact PII if present).

### 5. Update completeness + discrepancy log
Each probed category flips `verified-docs` → `verified-probe`. Probe-vs-doc deltas always land in `quirks.md` (one line + one evidence block: doc quote AND probe response snippet).

### 6. ReACT loop
Recompute completeness. While `unresolved > 0`:
- Identify the gap (which category, what's missing).
- Dispatch one targeted subagent or run one targeted probe.
- Re-score.

Cap: 3 iterations. If still unresolved, ask user: accept `unresolved` for those categories or extend the loop.

### 7. Finalize
Update frontmatter: `last_probed`, `api_version`. Recompute `.skill-hash` for each section. Print final completeness table.

## Re-run Behavior

If `docs/sources/<source>/` exists:

1. Read `.skill-hash`. For each section, compare current content hash vs stored hash.
2. Sections unchanged since last skill run → regenerate freely.
3. Sections with human edits (hash mismatch) → show diff, ask user per section before overwrite.
4. When the source itself has changed (new API version, deprecated endpoints), delete and replace stale content confidently. Do not preserve outdated fields just because they exist in the old file.

Staleness check: if `last_probed` > 90 days ago, print warning + offer re-probe scoped to endpoints named in the existing doc.

## Source Types

| Type | Probe strategy |
|------|----------------|
| REST / JSON | Direct HTTP via fetch; capture status + body per call |
| GraphQL | Schema introspection query + one representative query + one mutation per resource in scope |
| Webhook-only | Skip endpoint probes; tunnel + capture only |

Not supported: gRPC, WebSocket, SSE. Document doc-only, flag in `quirks.md` as "probe-infeasible".

## Execution Shape

- Parallel research phase: 4 Task subagents, independent context, run in one Task batch.
- Sequential main-agent work: synthesis → probe → ReACT → finalize. Needs full picture + interactive creds, stays on main.
- Progress tracking: TodoWrite list covering every phase + every category. Update live. Print completeness table after each ReACT iteration.

## `quirks.md` — the landmine log

Load-bearing for the next agent. Every entry = one line + one evidence block. Categories:

- Undocumented fields seen in probes
- Doc-vs-probe discrepancies (show both verbatim)
- Type coercions (string `"null"` vs `null` vs missing)
- Rate-limit behavior absent from docs
- Field-name inconsistencies (snake_case ↔ camelCase, plural mismatches)
- Pagination edge cases (off-by-one, empty last page, cursor reuse)
- Webhook replay / delivery order behavior

## Credentials Discipline

- Sandbox/test creds only. Refuse if ambiguous — confirm with user.
- Creds live in `.env.probe.local` only. Never in Keychain, never committed, never logged.
- Install trap on first cred write: `trap 'rm -f .env.probe.local' EXIT INT TERM HUP`.
- Add `.env.probe.local` to `.gitignore` on first run (check before append to avoid duplicates).

## Out of Scope (red flags — STOP if tempted)

- Writing code under `apps/` or `libs/` — STOP, handoff is docs only.
- Suggesting Nx project placement or directory layout — STOP, no codebase awareness.
- Running `pnpm add <sdk>` — STOP, document SDK recommendation only.
- Proposing implementation steps or architecture — STOP, next agent's job.
- Grepping the monorepo for existing patterns — STOP, external-only focus.
- Cleaning up sandbox data after probes — STOP, sandbox is disposable.
- Trusting docs without probing (when creds available) — STOP, docs lie.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Skipping probe because "docs look clean" | Probe anyway. Docs vs reality is the whole point. |
| Probing every endpoint | Probe only what's in user's integration scope. |
| Silent conflict resolution | Both versions in `quirks.md`, probe wins schema. |
| Writing impl plan | Delete. Out of scope. |
| Persisting creds anywhere but `.env.probe.local` | Move. Trap-clean on exit. |
| Leaving `unresolved` categories hidden | Every one appears in completeness table. |
| Over-preserving human edits on re-run | Ask, then replace confidently when stale. |
| Skipping webhook capture for webhook-only sources | That's the only signal — always capture. |

## Quick Reference

| Phase | Tools | Output |
|-------|-------|--------|
| Scope parse | user prompt | scope list |
| Research burst | Task × 4 (parallel) | 4 subagent reports |
| Synthesize | main agent | initial files written |
| Auth setup | Bash, WebFetch, local listener | `.env.probe.local` |
| Probe | Bash (curl/fetch), ngrok/smee | updated endpoints.md, schemas.ts, webhooks.md |
| ReACT | Task (targeted) | resolved categories |
| Finalize | frontmatter + hash | stable doc |
