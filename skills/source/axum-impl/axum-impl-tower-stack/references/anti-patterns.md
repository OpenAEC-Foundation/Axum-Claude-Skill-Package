# axum-impl-tower-stack: anti-patterns

Real mistakes when stacking tower / tower-http middleware on an Axum app,
each with root-cause analysis and the fix. Verified against docs.rs (tower,
tower-http) on 2026-05-20. These apply identically to Axum 0.7 and 0.8.

## AP-1: Confusing ServiceBuilder ordering with stacked layer ordering

```rust
// WRONG MENTAL MODEL - axum 0.7 / 0.8
// The author wants `timeout` outermost. With stacked .layer() calls,
// the LAST one is outermost -> here `trace` is outermost, not `timeout`.
let app = Router::new()
    .route("/", get(handler))
    .layer(TimeoutLayer::new(Duration::from_secs(30)))  // innermost (added first)
    .layer(TraceLayer::new_for_http());                 // outermost (added last)
```

Why it fails: `ServiceBuilder` and stacked `Router::layer()` order layers in
OPPOSITE directions. In `ServiceBuilder`, the first `.layer()` is outermost
(top-to-bottom). In stacked `Router::layer()` calls, the last `.layer()` is
outermost (bottom-to-top). An author who learned one model and applies it to
the other silently reorders the onion. The code compiles; the middleware just
runs in the wrong order. A timeout that should wrap tracing ends up wrapped
BY tracing, so trace spans no longer cover the timeout path.

Fix: build the global stack with ONE `ServiceBuilder` and apply it through a
single `.layer()` call. Write the outermost layer first.

```rust
// CORRECT - axum 0.7 / 0.8 - timeout is genuinely outermost
let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(TimeoutLayer::new(Duration::from_secs(30)))  // outermost
            .layer(TraceLayer::new_for_http()),                 // innermost
    );
```

## AP-2: Shipping CorsLayer::permissive() to production

```rust
// WRONG - axum 0.7 / 0.8 - permissive CORS on a public API
let app = Router::new()
    .route("/api", get(handler))
    .layer(CorsLayer::permissive());
```

Why it fails: `CorsLayer::permissive()` allows all origins (literal `*`), all
methods, and all request headers. It disables exactly the cross-origin
protection CORS exists to provide. Any website can call the API from a user's
browser. `very_permissive()` is worse for a credentialed API: it echoes the
caller's origin and allows credentials, exposing credentialed responses to
any origin.

Fix: in production ALWAYS use `CorsLayer::new()` with an explicit
`.allow_origin(...)` allowlist. Reserve `permissive()` for local development.

```rust
// CORRECT - axum 0.7 / 0.8 - explicit origin allowlist
let cors = CorsLayer::new()
    .allow_origin("https://app.example.com".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET, Method::POST]);
```

## AP-3: allow_credentials(true) with a wildcard origin

```rust
// WRONG - axum 0.7 / 0.8 - invalid CORS, browsers reject it
use tower_http::cors::{Any, CorsLayer};

let cors = CorsLayer::new()
    .allow_origin(Any)              // wildcard `*`
    .allow_credentials(true);       // credentials -> wildcard is forbidden
```

Why it fails: the CORS specification forbids the `*` wildcard origin when
credentials are allowed. A browser that receives
`Access-Control-Allow-Origin: *` together with
`Access-Control-Allow-Credentials: true` rejects the response. The request
fails in the browser even though the server returned `200`.

Fix: with credentials, the response must name a concrete origin. Either use
an explicit `.allow_origin(...)` allowlist, or use
`CorsLayer::very_permissive()`, which echoes the request's concrete origin
back instead of sending `*` (only acceptable outside production).

## AP-4: Forgetting the tower-http Cargo feature flag

```text
error[E0432]: unresolved import `tower_http::trace`
  --> src/main.rs
   |
   | use tower_http::trace::TraceLayer;
   |                 ^^^^^ could not find `trace` in `tower_http`
```

Why it fails: all tower-http middleware is disabled by default. `TraceLayer`,
`CorsLayer`, `CompressionLayer`, `TimeoutLayer`, `RequestBodyLimitLayer`,
`ServeDir` each live behind a Cargo feature. Without the feature, the module
is not compiled and the import does not resolve.

Fix: enable the matching feature in `Cargo.toml`.

```toml
# CORRECT
tower-http = { version = "0.6", features = ["trace", "cors", "timeout"] }
```

| Layer | Feature |
|-------|---------|
| `TraceLayer` | `trace` |
| `CorsLayer` | `cors` |
| `CompressionLayer` | `compression-gzip` / `-br` / `-deflate` / `-zstd` / `compression-full` |
| `TimeoutLayer` | `timeout` |
| `RequestBodyLimitLayer` | `limit` |
| `ServeDir` / `ServeFile` | `fs` |

`features = ["full"]` enables every layer.

## AP-5: Expecting a first-party tower-http rate-limit layer

```rust
// WRONG - axum 0.7 / 0.8 - this layer does not exist
use tower_http::limit::RateLimitLayer; // there is no such item
```

Why it fails: tower-http has NO rate-limit layer. `tower_http::limit` holds
`RequestBodyLimitLayer` (a body-size cap), not a request-rate limiter.

Fix: use one of the two real options.

- Coarse, GLOBAL limits: `tower::limit::ConcurrencyLimitLayer` (in-flight cap)
  or `tower::limit::RateLimitLayer` (requests per window). Not per-client.
- Per-client / per-IP token bucket: the `tower_governor` ecosystem crate
  (`GovernorLayer`). Recommended for production.

## AP-6: tower::limit::RateLimitLayer without a Buffer

```rust
// WRONG - axum 0.7 / 0.8 - RateLimitLayer's service is not Clone
let app = Router::new()
    .route("/", get(handler))
    .layer(RateLimitLayer::new(100, Duration::from_secs(1)));
// build fails: the resulting service does not implement Clone,
// which Axum's Route requires.
```

Why it fails: `RateLimitLayer` produces a `Service` that is NOT `Clone`.
Axum's `Route` requires the layered service to be `Clone + Send + Sync +
'static`. The trait bound is unsatisfied.

Fix: wrap `RateLimitLayer` in `tower::buffer::Buffer` (`BufferLayer`, Cargo
feature `buffer`). `Buffer` makes the inner service `Clone` by funneling
requests through an internal channel.

```rust
// CORRECT - axum 0.7 / 0.8 - Buffer in front of RateLimit restores Clone
let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(BufferLayer::new(1024))
            .layer(RateLimitLayer::new(100, Duration::from_secs(1))),
    );
```

## AP-7: Calling a Service's call without driving poll_ready

```rust
// WRONG - axum 0.7 / 0.8 - call without a ready signal
let fut = my_service.call(request); // poll_ready never returned Ready(Ok(()))
```

Why it fails: the `Service` contract is documented verbatim: "Implementations
are permitted to panic if `call` is invoked without obtaining
`Poll::Ready(Ok(()))` from `poll_ready`." Skipping `poll_ready` bypasses
backpressure (rate limits, concurrency limits, buffering) and may panic.

Fix: honour the readiness contract. ALWAYS drive `poll_ready` to
`Poll::Ready(Ok(()))` first. In practice, call a service through
`tower::ServiceExt::oneshot`, which performs the readiness dance, or through
`tower::ServiceExt::ready`. Handler-level Axum code rarely touches this
directly; a hand-written `Service` MUST honour it.

## Summary table

| Anti-pattern | Root cause | Fix |
|--------------|-----------|-----|
| AP-1 ordering confusion | ServiceBuilder is top-to-bottom, stacked `.layer()` is bottom-to-top | one ServiceBuilder, one `.layer()` |
| AP-2 permissive CORS in prod | `permissive()` allows all origins | explicit `.allow_origin()` allowlist |
| AP-3 credentials + wildcard | CORS spec forbids `*` with credentials | name a concrete origin |
| AP-4 missing Cargo feature | tower-http middleware disabled by default | enable the layer's feature |
| AP-5 expecting a rate-limit layer | tower-http has none | `tower::limit` or `tower_governor` |
| AP-6 RateLimitLayer not Clone | its service is not `Clone` | wrap in `tower::buffer::Buffer` |
| AP-7 call without poll_ready | violates the readiness contract | use `ServiceExt::oneshot` / `ready` |
