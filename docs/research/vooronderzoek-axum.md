# Vooronderzoek : Axum

> Status : Phase 2 complete
> Generated : 2026-05-19
> Researched : 2026-05-20
> Target versions : Axum 0.7 and 0.8 (docs.rs current : 0.8.9)
> Detailed source fragments : `docs/research/fragments/research-{a,b,c}-*.md`

This document synthesizes three verified research fragments (~10,500 words combined,
all API claims WebFetch-verified against docs.rs, the tokio-rs/axum repository, official
examples, and the CHANGELOG on 2026-05-20). It is the basis for Phase 3 masterplan
refinement.

---

## 1. Architecture, Runtime Model, Design Philosophy

Axum is an async web framework for Rust, built and maintained by the Tokio team. Its
single most important architectural fact: **Axum implements almost nothing itself**. It
composes three lower layers:

- **tokio** : the async runtime (executor, `TcpListener`, timers, `tokio::sync`).
  Handler futures must be `Send` because the multi-threaded scheduler moves tasks
  between worker threads.
- **hyper** : the HTTP/1 + HTTP/2 implementation. Axum 0.7+ requires hyper 1.0,
  `http` 1.0, `http-body` 1.0. Hyper parses bytes into `http::Request<Body>` and
  serializes `http::Response<Body>` back.
- **tower** : the `Service` trait, the universal `async fn(Request) -> Result<Response,
  Error>` abstraction. Everything routable in Axum is ultimately a `tower::Service`.
  Speaking `Service` means Axum inherits the entire tower / tower-http middleware
  ecosystem (timeouts, tracing, compression, CORS, auth, rate limiting) for free.

Request lifecycle: `TcpListener` -> hyper parses -> `axum::serve` drives the `Router`
as a `Service` -> Router matches path, selects `MethodRouter` -> MethodRouter selects
handler by HTTP method -> each extractor runs `FromRequestParts`/`FromRequest` ->
handler body executes -> return value runs `IntoResponse::into_response` -> response
flows back through hyper.

**The "no macros" design philosophy.** Axum deliberately exposes a macro-free routing
and handler API. No `#[get("/path")]` attributes (unlike Rocket/Actix). Routes are
ordinary values (`Router::new().route(...)`); handlers are ordinary `async fn`s whose
eligibility is enforced purely by trait bounds (the `Handler` trait). Benefits: routing
is data, so it composes, returns from functions, and unit-tests cleanly; no
macro-expansion obscures errors; IDE tooling works. Cost: a handler that fails the
trait bounds produces a notoriously cryptic compiler error, mitigated by the one macro
Axum ships, `#[debug_handler]` from `axum-macros`.

`axum-core` is a separate crate holding foundational traits (`FromRequest`,
`FromRequestParts`, `IntoResponse`, `Error`) so other crates can build extractors
without depending on all of Axum.

---

## 2. API Surface

**Router** (`Router<S>`, where `S` is state still required; a serveable router is
`Router<()>`): `Router::new()`, `.route(path, MethodRouter)`, `.route_service()`,
`.nest()`, `.nest_service()`, `.merge()`, `.fallback()`, `.fallback_service()`,
`.method_not_allowed_fallback()`, `.reset_fallback()`, `.layer()`, `.route_layer()`,
`.with_state()`, `.without_v07_checks()`.

**Method-routers** (`axum::routing`): `get`, `post`, `put`, `delete`, `patch`, `head`,
`options`, `any`. Each takes a handler, returns a `MethodRouter`, chains
(`get(list).post(create)`). `MethodRouter::fallback` handles unregistered methods (405).

**Extractors** : `Path<T>`, `Query<T>`, `Json<T>`, `Form<T>`, `State<T>`,
`Extension<T>`, `HeaderMap`, `Method`, `Uri`, `Bytes`, `String`, `Request`,
`Multipart`, `WebSocketUpgrade`, `NestedPath`, `MatchedPath`, `DefaultBodyLimit`.
Two traits govern them: `FromRequestParts<S>` (parts only, no body) and `FromRequest<S>`
(consumes the body). **Extractor ordering rule**: the body is a single-use async
stream, so a handler may have at most one `FromRequest` extractor and it MUST be the
last argument; all preceding arguments must be `FromRequestParts`.

**Handlers** : `async fn` with 0-16 extractor arguments returning `impl IntoResponse`.
The `Handler<T, S>` blanket impl requires: declared `async`, ≤16 `Send` arguments,
every argument except the last is `FromRequestParts`, last is `FromRequest`, return
type is `IntoResponse`, produced future is `Send`.

**Responses** : `IntoResponse` (the conversion trait) and `IntoResponseParts` (headers
/extensions without status/body). Built-in impls: `()`, `&str`/`String`, `Vec<u8>`/
`Bytes`, `StatusCode`, `(StatusCode, T)`, `(StatusCode, HeaderMap, T)`, `(StatusCode,
[(HeaderName, V); N], T)`, `Json<T>`, `Html<T>`, `Result<T, E>`. `Redirect::to` (303),
`::permanent` (308), `::temporary` (307). `AppendHeaders` preserves existing headers.
Cookies are not in core: use `axum-extra` `CookieJar`/`SignedCookieJar`/
`PrivateCookieJar`, or raw `Set-Cookie` headers.

**State** : `Router::with_state(value)` consumes `Router<S>` -> `Router<()>`. State
must be `Clone` (cloned per request). `FromRef` / `#[derive(FromRef)]` derives
substates so a handler extracts only the field it needs. `State` is compile-time
checked; `Extension` is runtime-resolved (missing extension = 500).

**Middleware** : `axum::middleware::from_fn`, `from_fn_with_state`, `map_request`,
`map_response` (+ `_with_state` variants), the `Next` type (`next.run(request).await`).
Or a hand-written `tower::Layer` + `tower::Service` pair.

**tower / tower-http** : `Service` (`poll_ready` for backpressure, `call`), `Layer`,
`ServiceBuilder`. tower-http layers (feature-gated): `TraceLayer`, `CorsLayer`,
`CompressionLayer`, `TimeoutLayer`, `RequestBodyLimitLayer`, services `ServeDir`/
`ServeFile`/`SetStatus`. No first-party rate-limit layer (use `tower::limit` or the
`tower_governor` crate).

**Realtime** : `axum::extract::ws` (`WebSocketUpgrade`, `WebSocket`, `Message`,
`on_upgrade`, `.split()`). `axum::response::sse` (`Sse`, `Event`, `KeepAlive`).

---

## 3. Version Matrix : v0.7 -> v0.8 Breaking Changes

Axum 0.7.0 released 2023-11-27; 0.8.0 released 2024-12-02. The breaking changes:

| Change | v0.7 | v0.8 |
|--------|------|------|
| Path capture syntax | `/:id`, `/*rest` | `/{id}`, `/{*rest}` (matchit 0.8). Old syntax PANICS at startup. |
| Custom extractor async | `#[async_trait]` on impl | native `async fn` in trait (RPITIT). MSRV Rust 1.75. |
| WebSocket `Message` | `Text(String)`, `Binary(Vec<u8>)` | `Text(Utf8Bytes)`, `Binary(Bytes)` |
| `Option<Path<T>>` | swallows all errors -> `None` | rejects on many errors (new `OptionalFromRequest` traits) |
| `RawBody`, `BodyStream` | available | removed (use `Bytes`/`String`/`Request`) |
| `axum::Server` | available | removed (use `axum::serve`) |
| `B` body type param | generic body param | removed; `IntoResponse` body must be `axum::body::Body` |
| `hyper::Body` re-export | re-exported | removed (use `axum::body::Body`) |
| `TypedHeader`, `headers` feature | in `axum` | moved to `axum-extra` |

`Router::without_v07_checks()` is the deliberate escape hatch for literal `:`/`*`
paths. Literal `{`/`}` are escaped by doubling (`{{`, `}}`).

---

## 4. Configuration Patterns

- **App entry** : `#[tokio::main] async fn main()`, build `Router`, bind
  `tokio::net::TcpListener`, `axum::serve(listener, app).await`.
- **State injection** : build resources (DB pool, config) once in `main`, pass via
  `.with_state(...)`. Substates via `FromRef`.
- **Feature flags** : tower-http layers are feature-gated (`trace`, `cors`,
  `compression-*`, `timeout`, `limit`, `fs`); Axum's `ws`, `multipart`, `macros`.
- **Global middleware stack** : build one `ServiceBuilder`, apply through a single
  `.layer()` call, register AFTER all routes.
- **Body limits** : `DefaultBodyLimit` is 2 MB by default; raise with
  `DefaultBodyLimit::max(n)` or `disable()` + `RequestBodyLimitLayer::new(n)`.
- **Env-driven config** : secrets and `DATABASE_URL` from environment variables, never
  literals; bind `0.0.0.0` in containers.

---

## 5. Security and Permissions Model

Axum has no built-in auth; it provides the building blocks (extractors, middleware).

- **Authentication** : implement token verification in a `FromRequestParts` extractor
  (e.g. a `Claims` type), so any handler taking that type is automatically protected.
  - **JWT** : `jsonwebtoken` crate (v10, `aws_lc_rs`), `Claims` struct with mandatory
    `exp`, `encode`/`decode`, keys from env secret.
  - **Session** : `tower-sessions` (`SessionManagerLayer` + pluggable stores),
    `axum-login` (full `AuthSession`, `login_required!` guard).
  - **Basic** : `axum-extra` `TypedHeader<Authorization<Basic>>`, TLS-only.
  - **OAuth2** : `oauth2` crate authorization-code flow, two routes (redirect +
    callback).
- **Authorization** : enforce role/permission checks in `from_fn_with_state`
  middleware applied with `route_layer` (so a 404 is not turned into a 401).
  Permission-based checks beat coarse roles. Per-route guards can also be a dedicated
  newtype extractor (`AdminClaims`) so the requirement shows in the signature.
- **CORS** : production uses `CorsLayer::new()` with an explicit `.allow_origin()`
  allowlist. `permissive()`/`very_permissive()` are dev-only; `allow_credentials(true)`
  is illegal with a wildcard origin.
- **Validation** : `validator` crate `#[derive(Validate)]` + a `ValidatedJson<T>`
  custom extractor returning 422; `garde` is an alternative.
- **Rejections** : every extractor has a typed `Rejection` implementing `IntoResponse`;
  customize with `Result<T, T::Rejection>` handler arguments or `axum-extra`
  `WithRejection`.

---

## 6. Common Workflows and Use Cases

- **CRUD JSON API** : `Router` + method-routers, `Json<T>` extractor, `Path<T>` for
  IDs, sqlx pool as state, typed `AppError` -> `IntoResponse`.
- **WebSocket service** : `WebSocketUpgrade` -> `on_upgrade` -> `.split()` socket,
  drive send/receive in separate `tokio::spawn` tasks, `tokio::select!` to abort.
- **SSE stream** : handler returns `Sse<impl Stream<Item = Result<Event, E>>>`, always
  attach `KeepAlive`.
- **File upload** : `Multipart` extractor (last argument), stream `.chunk()` to
  `tokio::fs::File`, raise `DefaultBodyLimit`.
- **Static / SPA hosting** : `ServeDir` via `nest_service`/`fallback_service`; SPA
  fallback wraps `ServeFile` in `SetStatus`.
- **Protected routes** : `Claims` extractor for auth + `route_layer` guard for authz.
- **Database access** : one `PgPool` in state (no `Arc` needed, `Pool` is a cheap
  reference-counted `Clone`); query macros (`query!`, `query_as!`) for compile-time
  checking; `pool.begin()` transactions auto-rollback on drop.
- **Observability** : `tracing_subscriber::registry()` + `EnvFilter` + `fmt::layer()`,
  `TraceLayer::new_for_http()`, `#[instrument]` on handlers, OpenTelemetry export.
- **Graceful shutdown** : `axum::serve(...).with_graceful_shutdown(shutdown_signal())`
  where the signal selects over `ctrl_c()` and Unix `SignalKind::terminate()`.
- **API docs** : `utoipa` annotations (`#[derive(ToSchema)]`, `#[utoipa::path]`,
  `#[derive(OpenApi)]`), `utoipa-swagger-ui`, or `utoipa-axum` `OpenApiRouter`.
- **Testing** : `axum-test` `TestServer`, or `tower::ServiceExt::oneshot` against the
  `Router` directly.

---

## 7. Error Patterns and Debugging Strategies

**The infallible-handler model** : at the `tower::Service` level all Axum services have
`Infallible` as their error type. A handler returning `Result<T, E>` does not "fail";
`Err(E)` is still turned into a valid HTTP response because `E: IntoResponse`.

**The canonical error pattern** : define an `AppError` enum, implement `IntoResponse`
for it (mapping variants to status codes + JSON body), implement `From<ForeignError>`
for each source error so the `?` operator converts automatically. `thiserror` for
structured status-mapped enums; `anyhow` via a newtype wrapper (`struct
AppError(anyhow::Error)`, orphan-rule workaround) for opaque catch-all glue. Production
services often combine both.

**Fallible middleware** : a `tower::Service` middleware that genuinely fails is adapted
with `HandleError` / `HandleErrorLayer`, which run an error handler converting the
error to a response.

**The "Handler not satisfied" error** : the no-macros design's main pain point. Causes:
an argument is not an extractor; a body extractor is not last; the return type is not
`IntoResponse`; the function is not `async`; the future is `!Send` (an `Rc`,
`RefCell`, or `std::sync::MutexGuard` held across `.await`). The diagnostic tool is
`#[debug_handler]`, which generates a precise plain-language error.

**Extractor rejections** : `axum::extract::rejection` module (`JsonRejection`,
`PathRejection`, etc., all `#[non_exhaustive]`). Inspect via `Result<T, Rejection>`
handler arguments, remap globally with `WithRejection`.

---

## 8. Build and Deployment

- **Release builds** : `cargo build --release` (the dev profile is far slower).
- **Static binaries** : target `x86_64-unknown-linux-musl` for a glibc-free binary
  that runs in `scratch`/`distroless` containers.
- **Docker** : multi-stage build, a `rust` builder stage (use `cargo-chef` to cache
  the dependency layer) + a minimal runtime stage (`debian:bookworm-slim`,
  `distroless`, or `scratch`). Images shrink from ~1.5 GB to tens of MB.
- **Runtime hygiene** : bind `0.0.0.0`, env-var config, SIGTERM-aware graceful
  shutdown, close the DB pool and flush tracing on shutdown.
- **Pool tuning** : size `max_connections` so `instances x max_connections` stays
  below the database server limit; `acquire_timeout` makes a saturated pool fail fast.

---

## 9. Anti-Patterns (verified, 16)

1. **Body extractor not last** : a `FromRequest` extractor in a non-final handler
   position fails the `Handler` trait bound (body is single-use).
2. **0.7 path syntax on 0.8** : `/:id` panics at router construction on 0.8.
3. **Non-`IntoResponse` return** : `async fn h() -> i32` fails the `Handler` bound.
4. **`!Send` across `.await`** : `std::sync::MutexGuard`/`Rc`/`RefCell` held across an
   await makes the future `!Send`; use `tokio::sync::Mutex` or scope the guard.
5. **`State` vs `Extension` confusion** : `Extension` for app-wide data risks runtime
   500s; `State` is compile-time checked.
6. **Assuming `Option<Path<T>>` swallows errors** : 0.8 changed this; it now rejects.
7. **`.layer()` before routes registered** : later routes are not wrapped.
8. **`ServiceBuilder` vs `.layer()` ordering** : `ServiceBuilder` is top-to-bottom
   (first = outermost), stacked `Router::layer()` is bottom-to-top (last = outermost).
9. **`CorsLayer::permissive()` in production** : disables cross-origin protection.
10. **Blocking / CPU work in an async handler or WS task** : starves the Tokio worker;
    use async I/O or `spawn_blocking`.
11. **`DefaultBodyLimit` (2 MB) silently blocking multipart uploads** : raise/disable
    it with an explicit ceiling.
12. **New DB pool per request** : exhausts database connections; build one pool in
    `main`.
13. **`.unwrap()`/`.expect()` in a handler** : turns runtime errors into panics;
    return `Result<T, AppError>`.
14. **Missing graceful shutdown** : SIGTERM kills in-flight requests on every deploy.
15. **Hardcoded JWT/DB secret** : bakes credentials into binary + git history; read
    from env.
16. **`tracing` span guard across `.await`** : produces incorrect interleaved traces;
    use `#[instrument]`.

---

## Newly Discovered Sub-Topics

Items surfaced by research that are NOT cleanly covered by the raw masterplan topics.
These drive the Phase 3 MERGE/DROP/SPLIT/ADD decisions.

1. **`MethodRouter` + method-not-allowed fallback** : the 404 (router fallback) vs 405
   (`MethodRouter::fallback` / `.method_not_allowed_fallback()`) distinction is subtle.
2. **`#[debug_handler]` diagnostic workflow** : the primary tool for the "Handler not
   satisfied" error. Belongs in an errors skill, not just syntax.
3. **`FromRef` / substate composition** : distinct from basic `State` usage.
4. **`NestedPath` extractor** : new in 0.8, recovers a nested router's mount prefix.
5. **`axum-extra` cookie extractors** (`CookieJar`, `SignedCookieJar`,
   `PrivateCookieJar`) : cookies are NOT in Axum core at all.
6. **`OptionalFromRequest` / `OptionalFromRequestParts`** : the 0.8 mechanism behind
   `Option<Extractor>`.
7. **`axum-core` crate boundary** : matters for library authors writing portable
   extractors.
8. **Complete removed-API migration checklist** : `RawBody`, `BodyStream`,
   `axum::Server`, `B` param, `hyper::Body` re-export, `headers`/`TypedHeader` move.
9. **WebSocket `Message` type change** (`String`/`Vec<u8>` -> `Utf8Bytes`/`Bytes`) :
   high-impact 0.7->0.8 break, needs a prominent version section.
10. **Rate limiting has no first-party tower-http layer** : needs `tower::limit` or
    the `tower_governor` crate.
11. **`route_layer` vs `layer`** : a distinct decision point (404-vs-401 auth).
12. **`SetStatus`** : required for the canonical SPA `index.html`-with-404 fallback.
13. **`anyhow` newtype wrapper pattern** : the orphan-rule workaround, distinct from
    the `thiserror` enum pattern.
14. **`tower-sessions` + `axum-login`** : the ecosystem standard for stateful auth,
    structurally different from JWT.
15. **`utoipa-axum` `OpenApiRouter`** : unifies routing + spec, distinct from plain
    `utoipa` + `utoipa-swagger-ui`.
16. **`cargo-chef`** : Docker dependency-layer caching for fast Rust container builds.
17. **`garde`** : a validation alternative to `validator`.
18. **OpenTelemetry export** (`opentelemetry` + `tracing-opentelemetry`) : distributed
    tracing, a clear sub-topic of tracing.
19. **`WithRejection` + derive-based custom rejections** : reusable rejection remapping.

---

## Verification Caveats

- WebFetch's summarization model could not reliably transcribe exact CHANGELOG release
  *dates* (returned chronologically impossible values). Release dates above
  (0.7.0 = 2023-11-27, 0.8.0 = 2024-12-02) are cross-checked knowledge; all API-shape
  and breaking-change claims ARE verified against fetched official content.
- The `async_trait` removal is confirmed via the live native-async trait definitions
  on docs.rs and the current `customize-extractor-error` example; skill authors should
  quote the trait definitions, not an exact CHANGELOG line.
- The CORS production warning is derived from documented `permissive`/`very_permissive`
  behavior plus the CORS spec rule, not a verbatim docs quote.
