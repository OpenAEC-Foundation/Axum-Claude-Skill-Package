---
name: axum-agents-api-review
description: >
  Use when auditing or reviewing existing Axum code, a diff, or a single file
  for correctness, reliability, and security defects, when a code review must
  flag Axum-specific mistakes, or when checking an Axum service against the
  package best practices before a merge or a deploy.
  Prevents shipping the 16 known Axum anti-patterns: 0.7 path syntax on 0.8,
  body extractor not last, non-IntoResponse handler returns, blocking calls in
  async handlers, per-request database pools, hardcoded secrets, missing
  graceful shutdown, and more.
  Covers a 10-category review checklist, a three-level severity model
  (blocker, warning, suggestion), the structured findings output format, and
  cross-references to all 28 other Axum skills that own each fix.
  Keywords: axum code review, audit axum, review checklist, anti-pattern,
  severity, blocker warning suggestion, structured findings, is my axum code
  correct, what is wrong with this handler, find bugs in axum code, check an
  axum service before deploy, how do I review axum code.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Axum API Review

## Overview

`axum-agents-api-review` is an orchestration skill. The other 28 skills in this
package teach how to WRITE correct Axum code. This skill teaches how to AUDIT
existing Axum code against that combined knowledge.

The agent reads a codebase, a diff, or a single file, runs the 10-category
checklist, classifies each finding by severity, and emits structured findings.
Every finding names the exact skill that owns the corrective guidance.

Two rules define this skill:

1. This skill DETECTS and DELEGATES. It NEVER teaches the fix inline. Each
   finding cites the owning skill (for example `axum-impl-database`); the user
   reads that skill for the fix. Keeping fix knowledge in the owning skill is
   what keeps this skill short and prevents drift.
2. This skill introduces NO new Axum API claims. Every check traces to a
   verified anti-pattern already owned by another skill.

## Quick Reference

### Severity model

The agent MUST assign exactly one severity per finding.

| Severity | Definition | Example |
|----------|------------|---------|
| `blocker` | Will not compile, panics at startup or runtime, or a confirmed security hole. Ship-stopping. | 0.7 path syntax on 0.8, body extractor not last, hardcoded JWT secret, `.unwrap()` on external input |
| `warning` | Compiles and runs, but has a latent correctness, reliability, or operational defect that WILL bite under load or in production. | Blocking call in an async handler, new DB pool per request, missing graceful shutdown, `CorsLayer::permissive()` in production |
| `suggestion` | Works correctly, but diverges from a package best practice. | `Extension` where `State` would be compile-checked, coarse roles instead of permission checks, missing `acquire_timeout` |

ALWAYS round up when uncertain between two levels. A false blocker costs a
review comment; a missed blocker costs an outage.

### Review process

1. Determine the target Axum version (0.7 or 0.8) from `Cargo.toml`. If
   unknown, report it as `unknown` and flag version-sensitive checks.
2. Run all 10 checklist categories (Pattern 1) against the code.
3. Assign exactly one severity per finding (Pattern 2).
4. Map the finding to one of the 16 anti-patterns where it fits (Pattern 3).
5. Emit the summary header, then ordered structured findings (Pattern 4).

### Output format

Summary header first, then one block per finding:

```
Axum API Review : <N> findings
  blockers : <b>   warnings : <w>   suggestions : <s>
  Target Axum version : <0.7 | 0.8 | unknown>

[<SEVERITY>] <short title>
  Location : <file>:<line> (or "design-level")
  Symptom  : <observable behavior that goes wrong>
  Anti-pattern : AP-<n> (omit if it maps to no AP)
  Fix skill : axum-<category>-<topic>
  Action   : <one-line corrective direction; the fix skill has the detail>
```

## Decision Trees

### Assigning severity

```
Classify the finding.
|
+- Does the code fail to compile, panic at startup or runtime, or have a
|  confirmed security hole?
|  `- YES: blocker.
|
+- Does it compile and run, but carry a latent defect that fails under load
|  or in production?
|  `- YES: warning.
|
+- Does it work correctly and only diverge from a best practice?
|  `- YES: suggestion.
|
`- Uncertain between two levels?
   `- Choose the HIGHER severity. Never invent a level outside
      {blocker, warning, suggestion}.
```

### Should this be a finding at all?

```
Considering reporting something.
|
+- Does it tie to a specific check in one of the 10 categories, or to one
|  of the 16 anti-patterns?
|  `- YES: report it, cite the check category or the AP number.
|
`- NO: it ties to no check.
   `- NEVER report it. This skill only reports checklist-anchored findings.
      A finding with no owning check and no AP number is out of scope.
```

### Empty review result

```
Finished the checklist, found nothing.
|
`- ALWAYS still emit the summary header with all counts at 0 and an explicit
   "No issues found" line. An empty result is never silent or ambiguous.
```

## Patterns

### Pattern 1: Run the 10-category checklist

Run every category. Each category lists its headline checks and the skills
that own the fix. The complete per-check tables (what to look for, failure
symptom, owning skill) are in `references/methods.md`.

1. Routing: path syntax matches the target version (`{id}` on 0.8, `:id` on
   0.7); routes registered before `.layer()`; 405 distinguished from 404; no
   double-fallback panic after `.merge()`. Owners: `axum-core-router`,
   `axum-core-version-migration`, `axum-impl-middleware`.

2. Extractors: a body extractor (`Json`, `Form`, `Multipart`, `Bytes`,
   `String`, `Request`) is the LAST argument; at most one body extractor per
   handler; `Option<Path<T>>` treated with 0.8 semantics; custom extractors use
   the right base trait and a `Rejection` that implements `IntoResponse`.
   Owners: `axum-syntax-extractors`, `axum-syntax-custom-extractors`,
   `axum-core-version-migration`, `axum-errors-extractor-rejections`,
   `axum-errors-handler-trait`.

3. Handlers: the return type is `IntoResponse`; the function is `async`; the
   future is `Send` (no `Rc`, `RefCell`, `std::sync::MutexGuard` across
   `.await`); at most 16 extractor arguments. Owners: `axum-syntax-handlers`,
   `axum-errors-handler-trait`, `axum-core-async-performance`.

4. Middleware: layer ordering is intentional (stacked `Router::layer()` is
   bottom-to-top); `ServiceBuilder` is top-to-bottom, the opposite direction;
   auth middleware uses `.route_layer()` not `.layer()`; `from_fn` argument
   order is correct (`Next` last). Owners: `axum-impl-middleware`,
   `axum-impl-tower-stack`, `axum-impl-authorization`.

5. State: `State` is preferred over `Extension` for application data; state is
   `Clone` and cheap to clone; `FromRef` is used for substate composition;
   interior mutability uses `tokio::sync::Mutex`, never `std::sync::Mutex`
   across `.await`. Owners: `axum-core-state`,
   `axum-core-async-performance`.

6. Errors: no `.unwrap()` or `.expect()` on fallible values in handlers; a
   typed `AppError` with `IntoResponse` exists; `From` impls wire the `?`
   operator; extractor rejections are customized where the default leaks
   internal detail; fallible `tower::Service` middleware is adapted with
   `HandleError`. Owners: `axum-errors-handling`,
   `axum-errors-extractor-rejections`.

7. Auth: JWT and session secrets are read from the environment, never a
   literal; `Claims` mandates an `exp` field; auth is enforced through a
   `FromRequestParts` extractor; authorization is permission-based, not
   coarse-role string comparison; CORS uses an explicit allowlist in
   production, never `CorsLayer::permissive()`; session auth uses an
   established crate. Owners: `axum-impl-auth-jwt`, `axum-impl-auth-session`,
   `axum-impl-authorization`, `axum-impl-tower-stack`,
   `axum-syntax-custom-extractors`.

8. Database: exactly one pool, built in `main`; the pool is shared via
   `with_state` with no extra `Arc`; `acquire_timeout` is set;
   `max_connections` is sized against the database server limit; transactions
   rely on drop-rollback or an explicit `commit`. Owners:
   `axum-impl-database`, `axum-core-state`, `axum-core-async-performance`.

9. Performance: no blocking or CPU-bound work in an async handler; blocking
   work uses `spawn_blocking`; WebSocket and SSE tasks never block;
   `DefaultBodyLimit` is raised for large uploads but a hard ceiling remains.
   Owners: `axum-core-async-performance`, `axum-impl-websockets`,
   `axum-impl-sse`, `axum-impl-file-upload`.

10. Deployment: graceful shutdown is wired; the shutdown signal covers
    SIGTERM, not only `ctrl_c()`; the pool is closed and tracing flushed on
    shutdown; the service binds `0.0.0.0` in a container; config comes from
    environment variables; no `tracing` span guard is held across `.await`.
    Owners: `axum-impl-deployment`, `axum-impl-tracing`,
    `axum-impl-database`, `axum-core-async-performance`.

### Pattern 2: Assign exactly one severity

Apply the severity decision tree to every finding. Compilation failures and
startup panics (0.7 syntax on 0.8, body extractor not last, non-`IntoResponse`
return, `!Send` future) are blockers. Confirmed security holes (hardcoded
secret) are blockers. `.unwrap()` in a handler is a blocker, because it
converts a recoverable error into a crashed request task. Operational defects
that survive compilation are warnings. Pure best-practice divergences are
suggestions. NEVER emit a severity outside `{blocker, warning, suggestion}`.

### Pattern 3: Map to the 16 anti-patterns

Where a finding matches a known anti-pattern, cite its `AP-<n>` number. The
full table is in `references/methods.md`. Headline mapping:

| AP | Anti-pattern | Severity |
|----|--------------|----------|
| AP-1 | Body extractor not last | blocker |
| AP-2 | 0.7 path syntax on 0.8 (startup panic) | blocker |
| AP-3 | Non-`IntoResponse` handler return | blocker |
| AP-4 | `!Send` value held across `.await` | blocker |
| AP-13 | `.unwrap()` or `.expect()` in a handler | blocker |
| AP-15 | Hardcoded JWT or DB secret | blocker |
| AP-6 | Assuming `Option<Path<T>>` swallows errors | warning |
| AP-7 | `.layer()` before routes registered | warning |
| AP-8 | `ServiceBuilder` vs `.layer()` ordering | warning |
| AP-9 | `CorsLayer::permissive()` in production | warning |
| AP-10 | Blocking or CPU work in an async handler or WS task | warning |
| AP-11 | `DefaultBodyLimit` silently blocking uploads | warning |
| AP-12 | New DB pool per request | warning |
| AP-14 | Missing graceful shutdown | warning |
| AP-5 | `State` vs `Extension` confusion | suggestion |
| AP-16 | `tracing` span guard across `.await` | suggestion |

A finding that ties to a checklist category but not to a numbered
anti-pattern omits the `Anti-pattern` line; it still cites a `Fix skill`.

### Pattern 4: Emit ordered, structured findings

Emit the summary header first, then the finding blocks. Findings MUST be
ordered: all blockers, then all warnings, then all suggestions. Within a
severity, order by file then by line. A review with zero findings MUST still
emit the header with all counts at 0 and an explicit `No issues found` line.
A full worked review (input code plus the exact output) is in
`references/examples.md`.

### Pattern 5: Delegate the fix, never teach it inline

Each finding's `Action` line is ONE corrective sentence plus the `Fix skill`
name. NEVER paste the corrected code or the full explanation into a finding:
that duplicates the owning skill and drifts from it over time. The owning
skill is the single source of the fix. The reviewer's job is detection,
classification, and routing.

## Cross-Reference Map: all 28 skills

Every other skill in the package, and the checklist categories that route to
it. Skills with no dedicated check are still audit-relevant context.

| Skill | Checklist categories |
|-------|----------------------|
| `axum-core-architecture` | context only (see `axum-agents-scaffold`) |
| `axum-core-router` | 1 Routing |
| `axum-core-state` | 5 State, 8 Database |
| `axum-core-async-performance` | 3 Handlers, 5 State, 6 Errors, 8 Database, 9 Performance, 10 Deployment |
| `axum-core-version-migration` | 1 Routing, 2 Extractors |
| `axum-syntax-extractors` | 2 Extractors, 3 Handlers |
| `axum-syntax-custom-extractors` | 2 Extractors, 7 Auth |
| `axum-syntax-handlers` | 3 Handlers |
| `axum-syntax-responses` | covered via 6 Errors |
| `axum-impl-middleware` | 1 Routing, 4 Middleware |
| `axum-impl-tower-stack` | 4 Middleware, 7 Auth |
| `axum-impl-static-files` | context only (low audit risk) |
| `axum-impl-websockets` | 9 Performance |
| `axum-impl-sse` | 9 Performance |
| `axum-impl-file-upload` | 9 Performance |
| `axum-impl-auth-jwt` | 7 Auth |
| `axum-impl-auth-session` | 7 Auth |
| `axum-impl-authorization` | 4 Middleware, 7 Auth |
| `axum-impl-database` | 8 Database |
| `axum-impl-validation` | covered via 2 Extractors |
| `axum-impl-testing` | context only (review audits code, not tests) |
| `axum-impl-tracing` | 10 Deployment |
| `axum-impl-openapi` | context only |
| `axum-impl-deployment` | 10 Deployment |
| `axum-errors-handling` | 6 Errors |
| `axum-errors-extractor-rejections` | 2 Extractors, 6 Errors |
| `axum-errors-handler-trait` | 2 Extractors, 3 Handlers |
| `axum-agents-scaffold` | the build-side counterpart of this skill |

## Reference Links

- `references/methods.md`: the complete 10-category checklist with one row per
  check (what to look for, failure symptom, owning skill), and the full
  16-anti-pattern table with severity and owning skill.
- `references/examples.md`: a full worked review, an input Axum file with
  multiple defects and the exact structured-findings output, plus an empty
  `No issues found` review.
- `references/anti-patterns.md`: mistakes the REVIEWER agent makes (teaching
  the fix inline, inventing a severity, skipping the summary header,
  unordered findings, reporting an unanchored finding) and why each fails.

Related skills: `axum-agents-scaffold` (the build-side counterpart that
orchestrates writing a new Axum service). Every finding routes to one of the
27 implementation and reference skills listed in the cross-reference map.

Sources: this skill introduces no new API claims. It synthesizes
`docs/research/vooronderzoek-axum.md` section 9 (the 16 verified
anti-patterns) and the masterplan Skill Inventory (the 28 cross-reference
target names).
