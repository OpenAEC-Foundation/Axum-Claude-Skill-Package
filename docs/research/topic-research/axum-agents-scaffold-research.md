# Topic Research : axum-agents-scaffold

> Skill : axum-agents-scaffold (agents/, batch 10)
> Status : Phase 4 topic-research
> Generated : 2026-05-20
> Research basis : `docs/research/vooronderzoek-axum.md` (whole file, especially §6)
> Version check : axum 0.8.9 current on docs.rs (WebFetch-verified 2026-05-20)

This skill is an ORCHESTRATION skill. It does not teach a single API; it teaches the
ordered sequence of decisions for building a complete Axum service and points to the
exact skill that owns each decision. It is the entry point a user reaches for when the
request is "build me an Axum service" rather than "fix this extractor".

---

## 1. The Ordered Build Sequence

The dependency chain of the package is `core -> syntax -> impl -> errors -> agents`.
Building a service follows the SAME order. Each step below names the EXACT skill(s)
from the masterplan inventory to consult.

**Step 0 : Decide what you are building.**
Run the decision tree in §4 first. The service type determines which optional steps
(WebSockets, SSE, file upload, OpenAPI) apply. ALWAYS classify before scaffolding.
Consult : `axum-core-architecture` (when to choose Axum, request lifecycle, the
`axum-core` vs `axum` boundary).

**Step 1 : Pin the version.**
Decide Axum 0.7 or 0.8 BEFORE writing any route. Path syntax, the WebSocket `Message`
type, and custom-extractor syntax all diverge. New services ALWAYS target 0.8 (current
0.8.9). Consult : `axum-core-version-migration`.

**Step 2 : Routing.**
Build the `Router`, attach method-routers (`get`/`post`/...), decide `.nest()` vs
`.merge()`, set the 404 fallback and the 405 `method_not_allowed_fallback`. Use 0.8
path syntax `/{id}` and `/{*rest}`. Register all routes BEFORE any `.layer()` call.
Consult : `axum-core-router`.

**Step 3 : State.**
Identify the long-lived resources (DB pool, config, HTTP client, template engine).
Build them once in `main`, inject with `.with_state(...)`. Use `#[derive(FromRef)]`
substates so each handler extracts only the field it needs. Choose `State` (compile-time
checked) over `Extension` (runtime-resolved). Consult : `axum-core-state`.

**Step 4 : Extractors and handlers.**
Write the `async fn` handlers. Pick extractors per input : `Path<T>`, `Query<T>`,
`Json<T>`, `State<T>`. Enforce the ordering rule : at most one body extractor and it
MUST be the last argument. Return any `IntoResponse` type. Consult :
`axum-syntax-extractors`, `axum-syntax-handlers`, `axum-syntax-responses`. For a
domain-specific extractor (e.g. a validated body, an auth claim), consult
`axum-syntax-custom-extractors`.

**Step 5 : Middleware and the tower stack.**
Add cross-cutting layers : `TraceLayer`, `CorsLayer` (explicit allowlist, NEVER
`permissive()` in production), `TimeoutLayer`, `CompressionLayer`,
`RequestBodyLimitLayer`. Decide `.layer()` (wraps all routes, runs before routing) vs
`.route_layer()` (only matched routes, so 404 stays 404). Consult :
`axum-impl-middleware`, `axum-impl-tower-stack`. For static assets / SPA hosting :
`axum-impl-static-files`.

**Step 6 : Errors.**
Define an `AppError` enum, implement `IntoResponse` for it, implement `From<...>` for
each source error so `?` converts automatically. Use `thiserror` for known
status-mapped variants plus an `anyhow` newtype catch-all. Replace every handler
`.unwrap()`/`.expect()` with `Result<T, AppError>`. Consult : `axum-errors-handling`,
`axum-errors-extractor-rejections`. If the compiler rejects a handler with "the trait
bound ... Handler is not satisfied", consult `axum-errors-handler-trait`.

**Step 7 : Authentication.**
Choose ONE primary scheme via the decision tree in `axum-impl-auth-session`. Stateless
API token : `axum-impl-auth-jwt` (`jsonwebtoken`, a `Claims` `FromRequestParts`
extractor). Browser session : `axum-impl-auth-session` (`tower-sessions` / `axum-login`,
HTTP Basic, OAuth2). Read every secret from an environment variable, NEVER a literal.

**Step 8 : Authorization.**
Add role/permission gates as `from_fn_with_state` middleware applied with
`route_layer` (so an unauthenticated hit on a protected path returns the correct
status, not a leaked 401 on a non-existent route). Prefer permission checks over coarse
roles. Consult : `axum-impl-authorization`.

**Step 9 : Database.**
Create ONE `sqlx` pool (`PgPool`) in `main`, share it via state. `Pool` is a cheap
reference-counted `Clone` : NO `Arc` wrapper. Use the compile-time-checked `query!` /
`query_as!` macros. Size `max_connections` so `instances x max_connections` stays under
the server limit; set `acquire_timeout` to fail fast. Consult : `axum-impl-database`.
For request-body validation : `axum-impl-validation` (a `ValidatedJson<T>` extractor
returning 422).

**Step 10 : Realtime (optional, service-type dependent).**
WebSockets : `axum-impl-websockets` (`WebSocketUpgrade`, `.split()`, two `tokio::spawn`
tasks). Server-push streams : `axum-impl-sse` (`Sse`, `Event`, always attach
`KeepAlive`). File uploads : `axum-impl-file-upload` (`Multipart` last argument, raise
`DefaultBodyLimit`).

**Step 11 : Observability.**
Initialise `tracing-subscriber` (`registry()` + `EnvFilter` + `fmt::layer()`), add
`TraceLayer::new_for_http()`, annotate handlers with `#[instrument]`. NEVER hold a span
guard across `.await`. Consult : `axum-impl-tracing`. For an OpenAPI spec :
`axum-impl-openapi` (`utoipa`, `utoipa-axum` `OpenApiRouter`).

**Step 12 : Testing.**
Extract a shared `app() -> Router` constructor. Test with `axum-test` `TestServer` or
`tower::ServiceExt::oneshot` against the `Router` directly. Consult :
`axum-impl-testing`.

**Step 13 : Deployment.**
`cargo build --release`, multi-stage Docker with `cargo-chef` dependency caching, a
minimal runtime image (`debian:bookworm-slim` / `distroless` / `scratch`). Wire
`axum::serve(...).with_graceful_shutdown(...)` selecting `ctrl_c()` and SIGTERM. On
shutdown close the DB pool and flush tracing. Bind `0.0.0.0`, config from env. Consult :
`axum-impl-deployment`.

**Step 14 : Self-review.**
Run the `axum-agents-api-review` checklist before declaring the service done.

---

## 2. Cargo.toml Dependency Baseline (Axum 0.8)

Verified against docs.rs : axum 0.8.9 current, axum depends on `tokio ^1.44`. Pin minor
versions; let patch float.

```toml
[dependencies]
# Core (always present)
axum = "0.8"                                    # the web framework
tokio = { version = "1", features = ["full"] }  # async runtime; "full" = macros + rt-multi-thread + net + signal + fs
tower = "0.5"                                   # Service/Layer abstraction; ServiceBuilder
tower-http = { version = "0.6", features = ["trace", "cors", "timeout", "compression-full"] }  # ready-made HTTP layers
serde = { version = "1", features = ["derive"] }  # (de)serialization for Json/Query/Form payloads
serde_json = "1"                                # JSON value type and error rendering

# Errors
thiserror = "2"                                 # structured, status-mapped AppError enum
anyhow = "1"                                    # opaque catch-all error for the AppError newtype

# Observability
tracing = "0.1"                                 # structured logging facade; event macros, #[instrument]
tracing-subscriber = { version = "0.3", features = ["env-filter"] }  # subscriber + EnvFilter

# Database (optional : data-backed services)
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "macros"] }  # async DB, compile-time-checked queries

# Auth (optional : protected services)
jsonwebtoken = "10"                             # JWT encode/decode (aws_lc_rs backend)

# Extras (optional : cookies, typed headers, WithRejection)
axum-extra = { version = "0.10", features = ["cookie", "typed-header"] }  # CookieJar, TypedHeader, WithRejection

# Validation (optional)
validator = { version = "0.20", features = ["derive"] }  # #[derive(Validate)] body validation

[dev-dependencies]
axum-test = "17"                                # TestServer integration-test helper
http-body-util = "0.1"                          # BodyExt::collect for reading response bodies in oneshot tests
```

Notes :
- `tokio` feature `full` is acceptable for an application binary. A library should
  enable only the precise features it uses (`rt-multi-thread`, `macros`, `net`,
  `signal`).
- `tower-http` layers are individually feature-gated : add `fs` for `ServeDir`,
  `limit` for `RequestBodyLimitLayer`. Only enable what the service uses.
- `axum` enables `tokio`, `http1`, `json` by default; add the `ws` feature for
  WebSockets, `multipart` for file uploads, `macros` for `#[debug_handler]`.
- Crate versions above are the current major lines as of 2026-05; skill authors should
  re-confirm minor versions at build time. The axum 0.8.x line itself is WebFetch-
  verified at 0.8.9.

---

## 3. Minimal-Server-to-Production Checklist

A minimal Axum server is roughly fifteen lines : a `Router`, one route, a
`TcpListener`, `axum::serve`. Production adds layers around that core. Add each item
as the service grows; ALWAYS complete the "before any deploy" tier.

**Minimal (proof of life)**
- [ ] `#[tokio::main] async fn main()`
- [ ] `Router::new().route("/", get(handler))`
- [ ] `TcpListener::bind` + `axum::serve(listener, app).await`

**Functional (does real work)**
- [ ] Typed handlers with `Path`/`Query`/`Json` extractors (one body extractor, last)
- [ ] Shared `State` for long-lived resources (DB pool, config)
- [ ] `AppError` enum implementing `IntoResponse`; no `.unwrap()` in handlers
- [ ] `Json<T>` request/response bodies with `serde`

**Hardened (before any deploy)**
- [ ] `TraceLayer` + `tracing-subscriber` initialised
- [ ] `CorsLayer` with an explicit origin allowlist (NEVER `permissive()`)
- [ ] `TimeoutLayer` so slow requests cannot pile up
- [ ] `DefaultBodyLimit` set to a deliberate ceiling
- [ ] All secrets read from environment variables, never literals
- [ ] Authentication on every non-public route
- [ ] Authorization gates applied with `route_layer`
- [ ] `with_graceful_shutdown` handling SIGTERM and Ctrl+C
- [ ] DB pool sized; `acquire_timeout` set
- [ ] A `tests/` suite using a shared `app()` constructor

**Operable (production lifecycle)**
- [ ] Multi-stage Docker build with `cargo-chef` caching
- [ ] `cargo build --release`; optional `musl` static binary
- [ ] Binds `0.0.0.0`; all config from env
- [ ] Shutdown closes the DB pool and flushes tracing
- [ ] OpenAPI spec published (if a public API)
- [ ] `CompressionLayer` for bandwidth
- [ ] Health-check route for the orchestrator's liveness probe

---

## 4. Decision Tree : What Kind of Axum Service Am I Building

```
What does the service primarily do?
|
+-- Stateless JSON CRUD API
|   -> core-router, core-state, syntax-extractors, syntax-handlers, syntax-responses,
|      impl-database, impl-validation, errors-handling, impl-auth-jwt, impl-testing
|
+-- Server-rendered / browser app (cookies, sessions, forms)
|   -> core-router, core-state, syntax-responses (CookieJar), impl-auth-session,
|      impl-static-files, impl-validation, errors-handling, impl-testing
|
+-- Realtime bidirectional (chat, live dashboard)
|   -> core-router, core-state, impl-websockets, core-async-performance,
|      impl-auth-jwt, errors-handling, impl-testing
|
+-- One-way server push (notifications, progress streams)
|   -> core-router, core-state, impl-sse, core-async-performance, impl-testing
|
+-- File / media service (upload, static hosting)
|   -> core-router, impl-file-upload, impl-static-files, impl-tower-stack,
|      core-async-performance, errors-handling
|
+-- Public API for third parties
    -> all of "Stateless JSON CRUD API" PLUS impl-openapi, impl-tracing,
       impl-authorization, impl-deployment
```

ALWAYS-applicable skills regardless of branch : `core-architecture`,
`core-version-migration`, `core-async-performance`, `impl-middleware`,
`impl-tower-stack`, `impl-tracing`, `impl-deployment`, `errors-handler-trait`,
`agents-api-review`.

Service-shape rules :
- ANY service touching a database pulls in `impl-database` and `core-async-performance`
  (a blocking query starves the runtime).
- ANY service with non-public routes pulls in an auth skill and `impl-authorization`.
- A realtime branch ALWAYS pulls in `core-async-performance` because blocking the
  socket task or holding a `!Send` value across `.await` is the dominant failure mode.
- WebSockets vs SSE : choose WebSockets only when the client must also SEND messages;
  one-way push is simpler and more proxy-friendly with SSE.

---

## Sources

Synthesized from the verified package research in `docs/research/vooronderzoek-axum.md`
(all API claims WebFetch-verified during Phase 2 against docs.rs, the tokio-rs/axum
repository, official examples, and the CHANGELOG). The current axum version (0.8.9) and
the `tokio ^1.44` dependency constraint were WebFetch-checked against
`https://docs.rs/axum/latest/axum/` on 2026-05-20. All skill names are taken verbatim
from the Skill Inventory in `docs/masterplan/axum-masterplan.md`. Companion-crate major
versions reflect the current published lines and should be re-confirmed at skill-build
time; only the axum 0.8.x line itself is independently version-verified.
