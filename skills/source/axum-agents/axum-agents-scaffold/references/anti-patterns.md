# Anti-patterns: axum-agents-scaffold

Real scaffolding mistakes, the symptom each produces, the root cause, and the
fix. These are process and ordering mistakes specific to building a whole
Axum service. Each fix points to the skill that owns the correct decision.

## Anti-pattern 1: building before classifying

Symptom: a service is half built, then a requirement appears (live updates,
file upload, public API) that forces a redesign of routing and state.

Root cause: scaffolding started before the service type was identified. The
optional steps (WebSockets, SSE, file upload, OpenAPI) change the routing and
state shape, so they cannot be bolted on cleanly afterwards.

Fix: ALWAYS run the service-type decision tree as step 0. Write down the
branch and its skill list before touching code. Consult
`axum-core-architecture` to confirm Axum is the right framework and to
understand the request lifecycle first.

## Anti-pattern 2: layering before routing

```rust
// WRONG - layer applied, then a route added after it
let app = Router::new()
    .layer(TraceLayer::new_for_http())
    .route("/late", get(handler)); // not wrapped by the layer
```

Symptom: a route added after a `.layer()` call is not covered by that layer.
Tracing, CORS, or timeout silently does not apply to it.

Root cause: `.layer()` wraps only the routes registered before it. Step 5
(middleware) runs after step 2 (routing) for exactly this reason.

Fix: ALWAYS register every route before any `.layer()` call. Follow the
sequence order: routing is step 2, middleware is step 5. Consult
`axum-core-router` and `axum-impl-middleware`.

## Anti-pattern 3: skipping the version pin

Symptom: route paths written as `/:id` fail to match on Axum 0.8, or `/{id}`
fails on 0.7. The WebSocket `Message` type or custom-extractor syntax does
not compile against the installed version.

Root cause: code was written before the Axum version was decided. Path
syntax, the WebSocket `Message` type, and `FromRequest` syntax all diverge
between 0.7 and 0.8.

Fix: ALWAYS pin the version as step 1, before writing the first route. New
services target 0.8. Consult `axum-core-version-migration` for the full list
of 0.7-to-0.8 differences.

## Anti-pattern 4: permissive CORS in production

```rust
// WRONG - allows every origin, every method, every header
.layer(CorsLayer::permissive())
```

Symptom: the service accepts cross-origin requests from any site. A browser
on an attacker page can call authenticated endpoints.

Root cause: `CorsLayer::permissive()` is a development convenience copied
into production untouched.

Fix: ALWAYS configure `CorsLayer` with an explicit origin allowlist. NEVER
ship `permissive()`. This is a Hardened-tier item; the service is not
deploy-ready without it. Consult `axum-impl-middleware` and
`axum-impl-tower-stack`.

## Anti-pattern 5: no graceful shutdown

```rust
// WRONG - no shutdown signal wired
axum::serve(listener, app).await?;
```

Symptom: on a container stop or deploy, in-flight requests are dropped
mid-response and the DB pool is not closed cleanly. Tracing buffers are lost.

Root cause: `axum::serve` was called without `.with_graceful_shutdown(...)`.
The container runtime sends SIGTERM and the process is killed before drain.

Fix: ALWAYS wire `axum::serve(...).with_graceful_shutdown(signal)` where the
signal selects over `ctrl_c()` and Unix `SignalKind::terminate()`. On
shutdown close the DB pool and flush tracing. This is a Hardened-tier item.
Consult `axum-impl-deployment`.

## Anti-pattern 6: blocking the runtime in a database-backed service

Symptom: under load, unrelated requests hang. A few slow queries make the
whole service unresponsive.

Root cause: a synchronous or CPU-bound call runs directly on the async
runtime, or a blocking driver is used instead of `sqlx`. The service-shape
rule that ANY database-touching service pulls in
`axum-core-async-performance` was ignored.

Fix: ALWAYS pull `axum-core-async-performance` into any branch that touches a
database or does realtime work. Use async I/O; move CPU-bound work to
`spawn_blocking`. Consult `axum-core-async-performance` and
`axum-impl-database`.

## Anti-pattern 7: Extension for application-wide state

```rust
// WRONG - app-wide resource passed as a runtime-resolved Extension
.layer(Extension(pool))
```

Symptom: a forgotten or mis-mounted `Extension` layer compiles fine and
returns 500 on every request that extracts it.

Root cause: `Extension<T>` is runtime-resolved. It is the right tool for
per-request values produced by middleware, not for resources known at
startup.

Fix: ALWAYS use `State` for resources known at startup (DB pool, config,
clients). A missing `with_state` is a compile error caught before deploy.
Step 3 owns this decision. Consult `axum-core-state`.

## Anti-pattern 8: skipping the self-review

Symptom: a service is declared done, then review later finds a body
extractor that is not last, a leaked secret literal, or a missing timeout.

Root cause: step 14 was skipped. The service was shipped on the author's
confidence rather than against a checklist.

Fix: ALWAYS run the `axum-agents-api-review` checklist as the final step.
NEVER declare a service done before it. The review audits extractor
ordering, path syntax, handler validity, middleware ordering, blocking
calls, per-request pools, error handling, secret sources, and graceful
shutdown.

## Anti-pattern 9: per-request resource construction

Symptom: throughput collapses; the database refuses connections; latency
grows with load.

Root cause: a `PgPool`, HTTP client, or other expensive resource is built
inside the handler instead of once in `main`. Every request opens and
discards its own.

Fix: ALWAYS build long-lived resources once in `main` and share them through
`State`. `Pool` is a cheap reference-counted `Clone`; it needs no `Arc`.
Step 3 owns this. Consult `axum-core-state` and `axum-impl-database`.

## Source

Derived from `docs/research/vooronderzoek-axum.md` section 9 (16 verified
anti-patterns) and the topic research at
`docs/research/topic-research/axum-agents-scaffold-research.md`. Skill names
are verbatim from the Skill Inventory in
`docs/masterplan/axum-masterplan.md`.
