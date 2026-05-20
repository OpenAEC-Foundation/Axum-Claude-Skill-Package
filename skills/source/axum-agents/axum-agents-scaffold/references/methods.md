# Build sequence and dependency baseline: axum-agents-scaffold

This file is the long form of the 14-step build sequence and the annotated
dependency baseline. It introduces no new Axum API claims; every API named
here is owned and verified by the cross-referenced skill. The Axum version
(0.8.9 current) and the `tokio ^1.44` constraint were WebFetch-verified
against docs.rs on 2026-05-20.

## The ordered build sequence

The package dependency chain is `core -> syntax -> impl -> errors -> agents`.
A service is built in the same order. Each step names the EXACT skill that
owns the decision.

### Step 0: classify what you are building

Run the service-type decision tree before scaffolding. The branch determines
which optional steps apply. ALWAYS classify first.
Consult: `axum-core-architecture` (when to choose Axum, the request
lifecycle, the `axum-core` vs `axum` crate boundary).

### Step 1: pin the version

Decide Axum 0.7 or 0.8 before writing any route. Path syntax, the WebSocket
`Message` type, and custom-extractor syntax all diverge. New services ALWAYS
target 0.8 (current 0.8.9).
Consult: `axum-core-version-migration`.

### Step 2: routing

Build the `Router`, attach method-routers (`get`, `post`, and the rest),
decide `.nest()` vs `.merge()`, set the 404 fallback and the 405
`method_not_allowed_fallback`. Use 0.8 path syntax `/{id}` and `/{*rest}`.
Register all routes BEFORE any `.layer()` call.
Consult: `axum-core-router`.

### Step 3: state

Identify the long-lived resources (DB pool, config, HTTP client, template
engine). Build them once in `main`, inject with `.with_state(...)`. Use
`#[derive(FromRef)]` substates so each handler extracts only the field it
needs. Choose `State` (compile-time checked) over `Extension` (runtime).
Consult: `axum-core-state`.

### Step 4: extractors and handlers

Write the `async fn` handlers. Pick extractors per input: `Path<T>`,
`Query<T>`, `Json<T>`, `State<T>`. Enforce the ordering rule: at most one
body extractor, and it MUST be the last argument. Return any `IntoResponse`
type.
Consult: `axum-syntax-extractors`, `axum-syntax-handlers`,
`axum-syntax-responses`. For a domain-specific extractor (a validated body,
an auth claim): `axum-syntax-custom-extractors`.

### Step 5: middleware and the tower stack

Add cross-cutting layers: `TraceLayer`, `CorsLayer` (explicit allowlist,
NEVER `permissive()` in production), `TimeoutLayer`, `CompressionLayer`,
`RequestBodyLimitLayer`. Decide `.layer()` (wraps all routes, runs before
routing) vs `.route_layer()` (only matched routes, so 404 stays 404).
Consult: `axum-impl-middleware`, `axum-impl-tower-stack`. For static assets
or SPA hosting: `axum-impl-static-files`.

### Step 6: errors

Define an `AppError` enum, implement `IntoResponse` for it, implement
`From<...>` for each source error so `?` converts automatically. Use
`thiserror` for known status-mapped variants plus an `anyhow` newtype
catch-all. Replace every handler `.unwrap()` and `.expect()` with
`Result<T, AppError>`.
Consult: `axum-errors-handling`, `axum-errors-extractor-rejections`. If the
compiler rejects a handler with "the trait bound ... Handler is not
satisfied": `axum-errors-handler-trait`.

### Step 7: authentication

Choose ONE primary scheme. Stateless API token: `axum-impl-auth-jwt`
(`jsonwebtoken`, a `Claims` `FromRequestParts` extractor). Browser session:
`axum-impl-auth-session` (`tower-sessions` or `axum-login`, HTTP Basic,
OAuth2). Read every secret from an environment variable, NEVER a literal.
Consult: `axum-impl-auth-jwt`, `axum-impl-auth-session`.

### Step 8: authorization

Add role and permission gates as `from_fn_with_state` middleware applied with
`route_layer`, so an unauthenticated hit on a protected path returns the
correct status and does not leak a 401 on a non-existent route. Prefer
permission checks over coarse roles.
Consult: `axum-impl-authorization`.

### Step 9: database and validation

Create ONE `sqlx` pool (`PgPool`) in `main`, share it via state. `Pool` is a
cheap reference-counted `Clone`: NO `Arc` wrapper. Use the compile-time
checked `query!` and `query_as!` macros. Size `max_connections` so
`instances x max_connections` stays under the server limit; set
`acquire_timeout` to fail fast.
Consult: `axum-impl-database`. For request-body validation:
`axum-impl-validation` (a `ValidatedJson<T>` extractor returning 422).

### Step 10: realtime (optional, service-type dependent)

WebSockets: `axum-impl-websockets` (`WebSocketUpgrade`, `.split()`, two
`tokio::spawn` tasks). Server-push streams: `axum-impl-sse` (`Sse`, `Event`,
always attach `KeepAlive`). File uploads: `axum-impl-file-upload`
(`Multipart` last argument, raise `DefaultBodyLimit`).

### Step 11: observability and API docs

Initialise `tracing-subscriber` (`registry()` plus `EnvFilter` plus
`fmt::layer()`), add `TraceLayer::new_for_http()`, annotate handlers with
`#[instrument]`. NEVER hold a span guard across `.await`.
Consult: `axum-impl-tracing`. For an OpenAPI spec: `axum-impl-openapi`
(`utoipa`, `utoipa-axum` `OpenApiRouter`).

### Step 12: testing

Extract a shared `app() -> Router` constructor. Test with `axum-test`
`TestServer` or `tower::ServiceExt::oneshot` against the `Router` directly.
Consult: `axum-impl-testing`.

### Step 13: deployment

`cargo build --release`, multi-stage Docker with `cargo-chef` dependency
caching, a minimal runtime image (`debian:bookworm-slim`, `distroless`, or
`scratch`). Wire `axum::serve(...).with_graceful_shutdown(...)` selecting
`ctrl_c()` and SIGTERM. On shutdown close the DB pool and flush tracing.
Bind `0.0.0.0`, config from env.
Consult: `axum-impl-deployment`.

### Step 14: self-review

Run the `axum-agents-api-review` checklist before declaring the service
done. It audits extractor ordering, path syntax, handler validity,
middleware ordering, blocking calls, per-request pools, error handling,
secret sources, and graceful shutdown.
Consult: `axum-agents-api-review`.

## Cargo.toml dependency baseline (Axum 0.8)

Verified against docs.rs: axum 0.8.9 current, axum depends on `tokio ^1.44`.
Pin minor versions; let the patch float.

```toml
[dependencies]
# Core (always present)
axum = "0.8"                                    # the web framework
tokio = { version = "1", features = ["full"] }  # async runtime: macros + rt-multi-thread + net + signal + fs
tower = "0.5"                                   # Service/Layer abstraction; ServiceBuilder
tower-http = { version = "0.6", features = ["trace", "cors", "timeout", "compression-full"] }  # HTTP layers
serde = { version = "1", features = ["derive"] }  # (de)serialization for Json/Query/Form payloads
serde_json = "1"                                # JSON value type and error rendering

# Errors
thiserror = "2"                                 # structured, status-mapped AppError enum
anyhow = "1"                                    # opaque catch-all error for the AppError newtype

# Observability
tracing = "0.1"                                 # structured logging facade; #[instrument]
tracing-subscriber = { version = "0.3", features = ["env-filter"] }  # subscriber + EnvFilter

# Database (optional: data-backed services)
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "macros"] }  # async DB, checked queries

# Auth (optional: protected services)
jsonwebtoken = "10"                             # JWT encode/decode (aws_lc_rs backend)

# Extras (optional: cookies, typed headers, WithRejection)
axum-extra = { version = "0.10", features = ["cookie", "typed-header"] }

# Validation (optional)
validator = { version = "0.20", features = ["derive"] }  # #[derive(Validate)] body validation

[dev-dependencies]
axum-test = "17"                                # TestServer integration-test helper
http-body-util = "0.1"                          # BodyExt::collect for oneshot test bodies
```

For Axum 0.7, change `axum = "0.7"` and `axum-extra = "0.9"`. The remaining
crates are unchanged.

## Feature-flag guidance

- `tokio` feature `full` is acceptable for an application binary. A library
  ALWAYS enables only the precise features it uses (`rt-multi-thread`,
  `macros`, `net`, `signal`).
- `tower-http` layers are individually feature-gated: add `fs` for
  `ServeDir`, `limit` for `RequestBodyLimitLayer`. Enable only what is used.
- `axum` enables `tokio`, `http1`, `json` by default. Add the `ws` feature
  for WebSockets, `multipart` for file uploads, `macros` for
  `#[debug_handler]` and `#[derive(FromRequest)]`.
- Companion-crate major versions reflect the current published lines as of
  2026-05. Re-confirm minor versions at build time. Only the axum 0.8.x line
  itself is independently version-verified (0.8.9).

## Source

Synthesized from `docs/research/vooronderzoek-axum.md` (all API claims
WebFetch-verified during Phase 2 against docs.rs, the tokio-rs/axum
repository, and official examples) and the topic research at
`docs/research/topic-research/axum-agents-scaffold-research.md`. The Axum
version and `tokio` constraint were WebFetch-checked against
`https://docs.rs/axum/latest/axum/` on 2026-05-20. All skill names are
verbatim from the Skill Inventory in `docs/masterplan/axum-masterplan.md`.
