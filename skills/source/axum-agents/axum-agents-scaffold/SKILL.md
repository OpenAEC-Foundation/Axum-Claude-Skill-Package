---
name: axum-agents-scaffold
description: >
  Use when the request is to build a complete Axum service from scratch
  rather than to fix one isolated piece: "build me an Axum API", "scaffold a
  Rust web service", "set up a new Axum project", or when a half-built service
  needs the missing production layers identified and ordered.
  Prevents scaffolding in the wrong order (layering before routes, choosing a
  framework before pinning a version), shipping a service with no graceful
  shutdown or permissive CORS, and reinventing decisions that a dedicated
  skill in this package already owns.
  Covers the ordered core to syntax to impl to errors build sequence, a
  service-type classifier, the Cargo.toml dependency baseline, the
  minimal-to-production checklist, and which of the other 28 Axum skills owns
  each decision.
  Keywords: axum scaffold, build axum service, new axum project, axum project
  structure, scaffold rust web service, axum boilerplate, where do I start
  with axum, how do I structure an axum app, axum service checklist, axum
  production checklist, axum Cargo.toml, axum dependencies, missing graceful
  shutdown, service has no CORS, what skill do I need, build order, axum
  starter, full axum example, axum from requirement.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-agents-scaffold

## Overview

This is an orchestration skill. It does NOT teach an Axum API. It teaches the
ordered sequence of decisions for building a complete Axum service and names
the exact skill in this package that owns each decision.

Reach for this skill when the request is "build a service", not "fix this
extractor". A single isolated question goes straight to the owning skill. A
whole-service build runs the sequence below.

Core principle: an Axum service is built in the same order as the package
dependency chain, `core -> syntax -> impl -> errors -> agents`. Building out
of order produces rework. ALWAYS classify the service type first, ALWAYS pin
the version second, then follow the sequence.

## Quick Reference

The ordered build sequence. Each step names the skill that owns it.

| Step | Decision | Owning skill(s) |
|------|----------|-----------------|
| 0 | Classify the service type | `axum-core-architecture` |
| 1 | Pin Axum 0.7 or 0.8 (new: 0.8) | `axum-core-version-migration` |
| 2 | Build the `Router`, method-routers, fallbacks | `axum-core-router` |
| 3 | Define shared `State` for long-lived resources | `axum-core-state` |
| 4 | Write handlers, pick extractors | `axum-syntax-extractors`, `axum-syntax-handlers`, `axum-syntax-responses`, `axum-syntax-custom-extractors` |
| 5 | Add middleware and the tower stack | `axum-impl-middleware`, `axum-impl-tower-stack`, `axum-impl-static-files` |
| 6 | Define `AppError`, wire `IntoResponse` and `From` | `axum-errors-handling`, `axum-errors-extractor-rejections`, `axum-errors-handler-trait` |
| 7 | Add authentication | `axum-impl-auth-jwt`, `axum-impl-auth-session` |
| 8 | Add authorization gates | `axum-impl-authorization` |
| 9 | Wire the database pool and validation | `axum-impl-database`, `axum-impl-validation` |
| 10 | Realtime (optional, type-dependent) | `axum-impl-websockets`, `axum-impl-sse`, `axum-impl-file-upload` |
| 11 | Observability and API docs | `axum-impl-tracing`, `axum-impl-openapi` |
| 12 | Tests against a shared `app()` constructor | `axum-impl-testing` |
| 13 | Release build, Docker, graceful shutdown | `axum-impl-deployment` |
| 14 | Self-review before declaring done | `axum-agents-api-review` |

Skills that ALWAYS apply, regardless of service type:
`axum-core-architecture`, `axum-core-version-migration`,
`axum-core-async-performance`, `axum-impl-middleware`,
`axum-impl-tower-stack`, `axum-impl-tracing`, `axum-impl-deployment`,
`axum-errors-handler-trait`, `axum-agents-api-review`.

Core rules:

- ALWAYS run the service-type decision tree before writing any code.
- ALWAYS pin the Axum version before writing the first route. Path syntax,
  the WebSocket `Message` type, and custom-extractor syntax all diverge.
- ALWAYS follow the sequence order. Routing precedes state precedes handlers
  precedes middleware precedes errors.
- ALWAYS finish with the `axum-agents-api-review` checklist.
- NEVER skip the "Hardened" tier before a deploy. It is not optional.
- NEVER scaffold a database-backed service without also consulting
  `axum-core-async-performance`. A blocking query starves the runtime.

## Decision Trees

### Classify the service type

Run this FIRST. The branch determines which optional steps apply.

```
What does the service primarily do?

Stateless JSON CRUD API:
  axum-core-router, axum-core-state, axum-syntax-extractors,
  axum-syntax-handlers, axum-syntax-responses, axum-impl-database,
  axum-impl-validation, axum-errors-handling, axum-impl-auth-jwt,
  axum-impl-testing

Server-rendered browser app (cookies, sessions, forms):
  axum-core-router, axum-core-state, axum-syntax-responses,
  axum-impl-auth-session, axum-impl-static-files, axum-impl-validation,
  axum-errors-handling, axum-impl-testing

Realtime bidirectional (chat, live dashboard):
  axum-core-router, axum-core-state, axum-impl-websockets,
  axum-core-async-performance, axum-impl-auth-jwt, axum-errors-handling,
  axum-impl-testing

One-way server push (notifications, progress streams):
  axum-core-router, axum-core-state, axum-impl-sse,
  axum-core-async-performance, axum-impl-testing

File or media service (upload, static hosting):
  axum-core-router, axum-impl-file-upload, axum-impl-static-files,
  axum-impl-tower-stack, axum-core-async-performance, axum-errors-handling

Public API for third parties:
  everything in "Stateless JSON CRUD API" PLUS axum-impl-openapi,
  axum-impl-tracing, axum-impl-authorization, axum-impl-deployment
```

### WebSockets or SSE for the realtime branch

```
Must the client also SEND messages over the live connection?
  YES: WebSockets. Consult axum-impl-websockets.
  NO (server push only): SSE. Consult axum-impl-sse.
       SSE is simpler and more proxy-friendly.
```

### Single question or full build

```
Is the request about one isolated piece (an extractor, one error, one route)?
  YES: skip this skill. Go straight to the owning skill in the table above.
  NO (build or extend a whole service): run the full sequence here.
```

## Patterns

### Pattern: the orchestration workflow

ALWAYS run scaffolding as three explicit phases.

1. Classify. Run the service-type decision tree. Write down the branch and
   the skill list it produced. ALWAYS classify before scaffolding.
2. Sequence. Walk steps 0 to 14 in order. At each step, open the owning
   skill, apply it, then move on. NEVER jump ahead: a later step depends on
   the types fixed by an earlier one.
3. Review. Run the `axum-agents-api-review` checklist. NEVER declare a
   service done before this step.

### Pattern: service-shape rules

These rules pull extra skills into the branch regardless of the base type.

- ANY service touching a database pulls in `axum-impl-database` AND
  `axum-core-async-performance`. A blocking query on the async runtime
  starves every other request.
- ANY service with non-public routes pulls in an auth skill
  (`axum-impl-auth-jwt` or `axum-impl-auth-session`) AND
  `axum-impl-authorization`.
- ANY realtime branch ALWAYS pulls in `axum-core-async-performance`. Blocking
  the socket task or holding a `!Send` value across `.await` is the dominant
  realtime failure mode.
- ANY public-facing API pulls in `axum-impl-openapi` and `axum-impl-tracing`.

### Pattern: the minimal-to-production tiers

Build a service in tiers. ALWAYS complete the Hardened tier before any
deploy. The full per-item checklist is in `references/examples.md`.

| Tier | Goal | What it adds |
|------|------|--------------|
| Minimal | Proof of life | `#[tokio::main]`, one `Router` route, `axum::serve` |
| Functional | Does real work | Typed extractors, shared `State`, `AppError`, JSON bodies |
| Hardened | Before any deploy | `TraceLayer`, allowlist `CorsLayer`, `TimeoutLayer`, body limit, env secrets, auth, graceful shutdown, sized DB pool, tests |
| Operable | Production lifecycle | Multi-stage Docker, release build, health route, compression, OpenAPI |

### Pattern: the dependency baseline

Step 1 fixes the Axum version; the dependency set follows from the branch.
Core dependencies (`axum`, `tokio`, `tower`, `tower-http`, `serde`,
`serde_json`, `thiserror`, `anyhow`, `tracing`, `tracing-subscriber`) are
always present. Database, auth, validation, and extras crates are pulled in
per branch. The full annotated `Cargo.toml` with feature-flag notes is in
`references/methods.md`.

ALWAYS pin minor versions and let the patch float. ALWAYS enable only the
crate features the service uses; `tokio` feature `full` is acceptable for an
application binary but NEVER for a library.

## Reference Links

- `references/methods.md`: the full 14-step build sequence with the precise
  action and owning skill for each step, plus the complete annotated
  Axum 0.8 `Cargo.toml` dependency baseline and feature-flag guidance.
- `references/examples.md`: two end-to-end worked scaffolds (a stateless JSON
  CRUD API and a realtime WebSocket service), each walked step by step, plus
  the complete minimal-to-production checklist.
- `references/anti-patterns.md`: real scaffolding mistakes with root-cause
  analysis: building before classifying, layering before routing, skipping
  the version pin, permissive CORS, no graceful shutdown, and skipping the
  self-review.

Related skills: every skill in this package. The Quick Reference table maps
each build step to its owning skill. `axum-agents-api-review` is the
companion agents skill that audits a finished service.
