# Masterplan : Axum

> Status : Phase 1 raw : pre-research
> Generated : 2026-05-19
> Updated : 2026-05-20

## Scope

- Technology : Axum (Rust async web framework)
- Versions : 0.7, 0.8
- Languages : Rust
- Prefix : axum
- License : MIT
- Runtime : tokio
- Companion stack : tower, tower-http, sqlx, serde, tracing, hyper

## Focus boundary

Axum ONLY. NOT Actix-Web, NOT Rocket. Companion crates covered where they integrate
directly with Axum (tower layers, sqlx pool as State, tracing-subscriber).

## Identified Topics (raw, pre-research)

Grouped by category. Counts are pre-research estimates, refined in Phase 3.

### Core (architecture, cross-cutting)

- axum-core-architecture : runtime model, tokio + hyper + tower, Service trait, request lifecycle
- axum-core-router : Router, nested routes, method-routers (get/post/put/delete/patch/head), merge, fallback, route ordering
- axum-core-state : Router::with_state, app-state pattern, Arc<...>, State vs Extension
- axum-core-performance : connection pooling, async pitfalls, tokio-runtime tuning, blocking-call traps
- axum-core-version-migration : v0.7 -> v0.8 breaking changes

### Syntax (API, code patterns)

- axum-syntax-extractors : Path<T>, Query<T>, Json<T>, Form<T>, State<T>, Extension<T>
- axum-syntax-custom-extractors : FromRequest, FromRequestParts traits, extractor ordering
- axum-syntax-handlers : async fn handlers, return types, IntoResponse trait, Handler trait bounds
- axum-syntax-responses : StatusCode, headers, cookies, Redirect, response builders, IntoResponseParts

### Implementation (workflows, use cases)

- axum-impl-middleware : tower::Layer, axum::middleware::from_fn, from_fn_with_state, ordering
- axum-impl-tower-stack : Service, Layer, ServiceBuilder, common layers (trace, cors, compression, timeout, rate-limit)
- axum-impl-websockets : WebSocketUpgrade, message handling, split sink/stream
- axum-impl-sse : Sse, Event stream, keep-alive
- axum-impl-file-upload : multipart extraction, large-file streaming
- axum-impl-static-files : tower_http ServeDir, ServeFile, SPA fallback
- axum-impl-auth : JWT, session, basic auth, OAuth2 flow
- axum-impl-authorization : role-based, permission-based middleware
- axum-impl-database : sqlx connection pool as State, transaction patterns
- axum-impl-validation : validator crate integration, custom validation extractors
- axum-impl-testing : axum-test, oneshot Request, response assertions
- axum-impl-tracing : tracing-subscriber, structured logging, OpenTelemetry
- axum-impl-graceful-shutdown : signal handling, with_graceful_shutdown
- axum-impl-openapi : utoipa OpenAPI generation
- axum-impl-deployment : release builds, statically linked binaries, Docker

### Errors (error handling patterns)

- axum-errors-handling : custom error types, IntoResponse for errors, ? operator ergonomics, anyhow vs thiserror
- axum-errors-extractor-rejections : extractor failures, rejection types, custom rejection responses
- axum-errors-trait-bounds : handler trait-bound errors, "Handler not satisfied" diagnostics

### Agents (orchestration)

- axum-agents-api-review : validation rules referencing other axum skills
- axum-agents-scaffold : orchestrate building a complete Axum service

## Estimated Skill Count

| Category | Estimate |
|----------|----------|
| core/    | 5        |
| syntax/  | 4        |
| impl/    | 14       |
| errors/  | 3        |
| agents/  | 2        |
| **Total** | **~28** |

Final count set in Phase 3 after research-driven MERGE/DROP/SPLIT decisions.

## Cross-tech boundaries

- Rust : language fundament (separate pkg). Axum skills assume Rust syntax is known.
- PostgreSQL / MariaDB : sqlx integration only, DB-specifics out of scope.
- Tauri 2 : Axum can run on the Tauri Rust side. Companion note only.
- OpenAEC : 4+ repos run Rust servers (openaec-server, openaec-cloud, openaec-reports-rs,
  openaec-docs). Skills should reflect production patterns.

## Next : Phase 2 Deep Research

Dispatch 3 parallel opus research agents per topic-cluster, write to
`docs/research/fragments/`, synthesize into `docs/research/vooronderzoek-axum.md`.
