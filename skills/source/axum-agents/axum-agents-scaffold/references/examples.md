# Examples: axum-agents-scaffold

Two end-to-end worked scaffolds and the full minimal-to-production checklist.
These examples show the ORDER of decisions and which skill owns each one.
They introduce no new API claims; each named API is owned and verified by the
cross-referenced skill.

## Example 1: scaffolding a stateless JSON CRUD API

The request: "build me a REST API for managing tasks, Postgres-backed, with
JWT auth."

### Classify

The service-type decision tree puts this in the "Stateless JSON CRUD API"
branch. Because it touches a database, the service-shape rule pulls in
`axum-core-async-performance`. Because it has protected routes, it pulls in
`axum-impl-authorization`. Resulting skill list:

```
axum-core-architecture, axum-core-version-migration, axum-core-router,
axum-core-state, axum-core-async-performance, axum-syntax-extractors,
axum-syntax-handlers, axum-syntax-responses, axum-impl-middleware,
axum-impl-tower-stack, axum-errors-handling, axum-errors-handler-trait,
axum-impl-auth-jwt, axum-impl-authorization, axum-impl-database,
axum-impl-validation, axum-impl-tracing, axum-impl-testing,
axum-impl-deployment, axum-agents-api-review
```

### Sequence

| Step | Action | Owning skill |
|------|--------|--------------|
| 1 | Pin Axum 0.8.9 | `axum-core-version-migration` |
| 2 | `Router` with `/tasks` and `/tasks/{id}` routes, 404 fallback | `axum-core-router` |
| 3 | `AppState { pool: PgPool, jwt_secret: Arc<str> }`, `#[derive(FromRef)]` | `axum-core-state` |
| 4 | Handlers: `Json<NewTask>` create, `Path<i64>` fetch, list query | `axum-syntax-extractors`, `axum-syntax-handlers`, `axum-syntax-responses` |
| 5 | `TraceLayer`, allowlist `CorsLayer`, `TimeoutLayer`, body limit | `axum-impl-middleware`, `axum-impl-tower-stack` |
| 6 | `AppError` enum, `IntoResponse`, `From<sqlx::Error>` | `axum-errors-handling` |
| 7 | `Claims` JWT extractor, secret from env | `axum-impl-auth-jwt` |
| 8 | `route_layer` permission gate on write routes | `axum-impl-authorization` |
| 9 | One `PgPool` in `main`, `query_as!`, `ValidatedJson<NewTask>` | `axum-impl-database`, `axum-impl-validation` |
| 11 | `tracing-subscriber` registry, `#[instrument]` handlers | `axum-impl-tracing` |
| 12 | `app() -> Router` constructor, `TestServer` integration tests | `axum-impl-testing` |
| 13 | Release build, multi-stage Docker, graceful shutdown | `axum-impl-deployment` |
| 14 | Run the review checklist | `axum-agents-api-review` |

Step 10 (realtime) is skipped: this branch has no live connections.

### The skeleton `main`

```rust
// axum 0.8 - the assembly point; each piece is owned by its step's skill
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // step 11: observability first, so startup itself is traced
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .with(tracing_subscriber::fmt::layer())
        .init();

    // step 3: long-lived resources built once
    let pool = sqlx::PgPool::connect(&std::env::var("DATABASE_URL")?).await?;
    let state = AppState { pool, jwt_secret: std::env::var("JWT_SECRET")?.into() };

    // step 2 + 4 + 5: routes registered BEFORE layers
    let app = app(state);

    // step 13: bind 0.0.0.0, graceful shutdown
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;
    Ok(())
}

// step 12: the shared constructor, reused by tests
fn app(state: AppState) -> axum::Router {
    use axum::routing::{get, post};
    axum::Router::new()
        .route("/tasks", get(list_tasks).post(create_task))
        .route("/tasks/{id}", get(get_task))
        .layer(/* TraceLayer, CorsLayer, TimeoutLayer: step 5 */)
        .with_state(state)
}
```

## Example 2: scaffolding a realtime WebSocket service

The request: "build a chat backend, browser clients connect over WebSocket."

### Classify

The decision tree puts this in the "Realtime bidirectional" branch, chosen
over SSE because the client must SEND messages, not only receive. The
service-shape rule for realtime ALWAYS pulls in
`axum-core-async-performance`. Resulting skill list:

```
axum-core-architecture, axum-core-version-migration, axum-core-router,
axum-core-state, axum-core-async-performance, axum-impl-websockets,
axum-impl-middleware, axum-impl-tower-stack, axum-impl-auth-jwt,
axum-errors-handling, axum-errors-handler-trait, axum-impl-tracing,
axum-impl-testing, axum-impl-deployment, axum-agents-api-review
```

### Sequence

| Step | Action | Owning skill |
|------|--------|--------------|
| 1 | Pin Axum 0.8.9 (the `Message` type differs from 0.7) | `axum-core-version-migration` |
| 2 | `Router` with a `/ws` upgrade route | `axum-core-router` |
| 3 | `AppState` holding a broadcast channel sender | `axum-core-state` |
| 4 | `WebSocketUpgrade` handler returning `on_upgrade` | `axum-impl-websockets` |
| 5 | `TraceLayer`, `CorsLayer` allowlist | `axum-impl-middleware`, `axum-impl-tower-stack` |
| 6 | `AppError` for the upgrade path | `axum-errors-handling` |
| 7 | `Claims` JWT extractor validates the connecting client | `axum-impl-auth-jwt` |
| async | Split the socket, two `tokio::spawn` tasks, NEVER block | `axum-core-async-performance` |
| 11 | `tracing-subscriber`, `#[instrument]` on the upgrade handler | `axum-impl-tracing` |
| 12 | `app()` constructor, `TestServer` WebSocket test | `axum-impl-testing` |
| 13 | Release build, Docker, graceful shutdown | `axum-impl-deployment` |
| 14 | Review checklist | `axum-agents-api-review` |

The realtime branch makes `axum-core-async-performance` critical: blocking
the socket task or holding a `!Send` value across `.await` is the dominant
failure mode. Database steps (9) are skipped unless chat history is
persisted; if it is, the service-shape rule re-adds `axum-impl-database`.

## The minimal-to-production checklist

Build a service in tiers. ALWAYS complete the Hardened tier before any
deploy.

### Minimal (proof of life)

- [ ] `#[tokio::main] async fn main()`
- [ ] `Router::new().route("/", get(handler))`
- [ ] `TcpListener::bind` plus `axum::serve(listener, app).await`

### Functional (does real work)

- [ ] Typed handlers with `Path`, `Query`, `Json` extractors (one body
      extractor, last)
- [ ] Shared `State` for long-lived resources (DB pool, config)
- [ ] `AppError` enum implementing `IntoResponse`; no `.unwrap()` in handlers
- [ ] `Json<T>` request and response bodies with `serde`

### Hardened (before any deploy)

- [ ] `TraceLayer` plus `tracing-subscriber` initialised
- [ ] `CorsLayer` with an explicit origin allowlist (NEVER `permissive()`)
- [ ] `TimeoutLayer` so slow requests cannot pile up
- [ ] `DefaultBodyLimit` set to a deliberate ceiling
- [ ] All secrets read from environment variables, never literals
- [ ] Authentication on every non-public route
- [ ] Authorization gates applied with `route_layer`
- [ ] `with_graceful_shutdown` handling SIGTERM and Ctrl+C
- [ ] DB pool sized; `acquire_timeout` set
- [ ] A `tests/` suite using a shared `app()` constructor

### Operable (production lifecycle)

- [ ] Multi-stage Docker build with `cargo-chef` caching
- [ ] `cargo build --release`; optional `musl` static binary
- [ ] Binds `0.0.0.0`; all config from env
- [ ] Shutdown closes the DB pool and flushes tracing
- [ ] OpenAPI spec published (if a public API)
- [ ] `CompressionLayer` for bandwidth
- [ ] Health-check route for the orchestrator liveness probe

## Source

Synthesized from `docs/research/vooronderzoek-axum.md` section 6 and the
topic research at
`docs/research/topic-research/axum-agents-scaffold-research.md`. All skill
names are verbatim from the Skill Inventory in
`docs/masterplan/axum-masterplan.md`.
