# Topic Research : axum-impl-tower-stack

> Skill : `axum-impl-tower-stack` (category `impl`)
> Phase : 4 (topic research)
> Target versions : Axum 0.7 and 0.8 (docs.rs current : axum 0.8.9, tower 0.5.x, tower-http 0.6.x)
> All signatures and quotations WebFetch-verified against docs.rs on 2026-05-20.
> Source fragments : `vooronderzoek-axum.md` section 2, `fragments/research-b-middleware-tower-realtime.md` sections 3-4.

This file scopes the `axum-impl-tower-stack` skill: the `tower::Service` and
`tower::Layer` traits, the `ServiceBuilder` ordering rule (and how it differs from
stacked `Router::layer()`), the common `tower-http` layers with their feature flags,
`CorsLayer` configuration, and rate-limiting options. Everything routable in Axum is
ultimately a `tower::Service`, so Axum inherits the entire tower / tower-http
middleware ecosystem for free.

---

## 1. The `tower::Service` Trait (verified)

`Service` is the universal Tower abstraction: conceptually
`async fn(Request) -> Result<Response, Error>`. Verified verbatim from
`docs.rs/tower/latest/tower/trait.Service.html`:

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(
        &mut self,
        cx: &mut Context<'_>,
    ) -> Poll<Result<(), Self::Error>>;

    fn call(&mut self, req: Request) -> Self::Future;
}
```

Associated types: `Response` (success type), `Error` (error type), `Future` (the
future `call` returns).

### `poll_ready` : backpressure / readiness (verified contract)

`poll_ready` implements **backpressure**. Verified verbatim quotes:

- "Returns `Poll::Ready(Ok(()))` when the service is able to process requests."
- "Before dispatching a request, `poll_ready` must be called and return
  `Poll::Ready(Ok(()))`."
- "Until a request is dispatched, repeated calls to `poll_ready` must return either
  `Poll::Ready(Ok(()))` or `Poll::Ready(Err(_))`."
- A service that cannot yet process requests returns `Poll::Pending`; the task is
  notified when it becomes ready again.
- `Poll::Ready(Err(_))` means the service can no longer handle requests.

### `call` : process the request (verified contract)

`call` consumes a request and returns a `Future` resolving to
`Result<Response, Error>`. Verified verbatim: "Implementations are permitted to panic
if `call` is invoked without obtaining `Poll::Ready(Ok(()))` from `poll_ready`."

DETERMINISTIC RULE: ALWAYS drive `poll_ready` to `Poll::Ready(Ok(()))` before calling
`call`. NEVER call `call` on a service whose `poll_ready` has not yet returned
`Poll::Ready(Ok(()))` — the implementation is permitted to panic. In handler-level
Axum code you rarely touch this directly (Axum and `tower::ServiceExt::oneshot`
handle the readiness dance), but a hand-written `Service` MUST honour it.

At the Axum level, every Router-backed service has `Error = Infallible`: a handler
returning `Result<T, E>` does not fail the service, because `Err(E)` is converted to a
valid HTTP response via `IntoResponse`.

---

## 2. The `tower::Layer` Trait (verified)

`Layer` decorates a `Service`, producing a new wrapped `Service`. Verified shape from
`docs.rs/tower/latest/tower/trait.Layer.html`:

```rust
pub trait Layer<S> {
    type Service;
    fn layer(&self, inner: S) -> Self::Service;
}
```

A `Layer` is the reusable, composable unit; calling `.layer(inner)` produces the
actual `Service`. `Router::layer(L)` accepts any `L: Layer<Route>` that is `Clone +
Send + Sync + 'static`. A hand-written middleware is a `Layer` + `Service` pair: the
`Layer` builds the `Service`, the `Service` carries the `poll_ready`/`call` behavior.

DECISION RULE: use `axum::middleware::from_fn` for app-internal middleware written in
familiar async/await; write a real `Layer` + `Service` pair when you need configurable
builder methods, fine-grained `poll_ready` control, or you intend to publish the
middleware as a reusable crate.

---

## 3. `ServiceBuilder` Ordering Rule (verbatim-verified)

`ServiceBuilder` gives a declarative API for stacking layers onto a leaf service.
The ordering semantics are the single most error-prone part of the topic, so they
were verified verbatim against `docs.rs/tower/latest/tower/builder/struct.ServiceBuilder.html`.

### Verbatim documentation wording

- "The order in which layers are added impacts how requests are handled. Layers that
  are added first will be called with the request first."
- "The argument to `service` will be last to see the request."

### Verbatim worked example

```rust
ServiceBuilder::new()
    .buffer(100)
    .concurrency_limit(10)
    .service(svc)
```

Verbatim: "In the above example, the buffer layer receives the request first followed
by `concurrency_limit`." When the two are reversed "the order of layers is reversed.
Now, `concurrency_limit` applies first and only allows 10 requests to be in-flight
total."

### THE DETERMINISTIC RULE

In `ServiceBuilder`, the **FIRST** `.layer()` added is the **OUTERMOST** layer
(top-to-bottom, first = outermost, first to see the request). This is the **OPPOSITE**
of stacked `Router::layer()` calls, where the **LAST** `.layer()` added is the
**OUTERMOST** layer (bottom-to-top, last = outermost).

| Construction | Reading direction | Which layer is outermost |
|--------------|-------------------|--------------------------|
| `ServiceBuilder::new().layer(A).layer(B).service(svc)` | top-to-bottom | `A` is outer, `B` is inner |
| `router.layer(A).layer(B)` | bottom-to-top | `B` is outer, `A` is inner |

### Worked example contrasting the two

Goal: a request must hit `timeout` (outer), then `compression`, then `trace`, then
the handler.

```rust
// (a) ServiceBuilder : write OUTERMOST FIRST, top-to-bottom
use tower::ServiceBuilder;

let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(TimeoutLayer::new(Duration::from_secs(30)))   // outermost
            .layer(CompressionLayer::new())                      // middle
            .layer(TraceLayer::new_for_http()),                  // innermost
    );
```

```rust
// (b) Stacked Router::layer() : same nesting, but written in REVERSE order
let app = Router::new()
    .route("/", get(handler))
    .layer(TraceLayer::new_for_http())                       // innermost (added first)
    .layer(CompressionLayer::new())                          // middle
    .layer(TimeoutLayer::new(Duration::from_secs(30)));      // outermost (added last)
```

Both `(a)` and `(b)` produce the IDENTICAL onion: request flows `timeout` ->
`compression` -> `trace` -> `handler`; response flows back in reverse. The trap is
mixing the two mental models. Writing `router.layer(timeout).layer(trace)` and
expecting `timeout` to be outermost is WRONG — there `trace` is outermost.

DETERMINISTIC RECOMMENDATION: ALWAYS build the global middleware stack with a single
`ServiceBuilder` and apply it through one `.layer()` call. The top-to-bottom reading
order matches how humans reason about an onion (outermost written first), and one
`.layer()` call eliminates the bottom-to-top stacking ambiguity entirely.

### `.layer()` placement rule (cross-referenced, not the core of this skill)

`.layer()` only wraps routes that already exist: ALWAYS register all `.route()` /
`.fallback()` calls BEFORE the `.layer()` call. This is owned by the
`axum-impl-middleware` skill; `axum-impl-tower-stack` should reference it, not
re-teach it.

---

## 4. `tower-http` Layers (feature flags verified)

`tower-http` middleware is feature-gated; every layer needs its Cargo feature enabled
in `Cargo.toml`. All middleware is disabled by default. The `full` feature enables all
middleware at once. Feature flags verified against
`docs.rs/tower-http/latest/tower_http/index.html`.

| Layer | Module | Cargo feature | Purpose |
|-------|--------|---------------|---------|
| `TraceLayer` | `tower_http::trace` | `trace` | High-level request/response tracing spans + events |
| `CorsLayer` | `tower_http::cors` | `cors` | Adds CORS headers, handles preflight |
| `CompressionLayer` | `tower_http::compression` | `compression-gzip`, `compression-br`, `compression-deflate`, `compression-zstd` (or `compression-full`) | Compresses response bodies via `Accept-Encoding` negotiation |
| `TimeoutLayer` | `tower_http::timeout` | `timeout` | Aborts requests exceeding a `Duration` |
| `RequestBodyLimitLayer` | `tower_http::limit` | `limit` | Caps request body length; over-limit becomes `413 Payload Too Large` |
| `ServeDir` / `ServeFile` | `tower_http::services` | `fs` | Static file `Service`s (NOT layers; mount via `nest_service`/`fallback_service`) |

`Cargo.toml` example:

```toml
[dependencies]
tower-http = { version = "0.6", features = [
    "trace", "cors", "compression-gzip", "timeout", "limit",
] }
```

### Config snippets (verified shapes)

```rust
// TraceLayer
use tower_http::trace::TraceLayer;
router.layer(TraceLayer::new_for_http());
```

```rust
// CompressionLayer
use tower_http::compression::CompressionLayer;
router.layer(CompressionLayer::new());
```

```rust
// TimeoutLayer
use tower_http::timeout::TimeoutLayer;
use std::time::Duration;
router.layer(TimeoutLayer::new(Duration::from_secs(30)));
```

```rust
// RequestBodyLimitLayer  ->  pub fn new(limit: usize) -> Self
use tower_http::limit::RequestBodyLimitLayer;
router.layer(RequestBodyLimitLayer::new(10 * 1024 * 1024)); // 10 MB ceiling
```

`RequestBodyLimitLayer` is the explicit body ceiling to pair with
`DefaultBodyLimit::disable()`: ALWAYS set a hard ceiling after disabling Axum's 2 MB
default, because an unbounded request body is a memory-exhaustion DoS vector.

---

## 5. `CorsLayer` Configuration (verified)

Verified against `docs.rs/tower-http/latest/tower_http/cors/struct.CorsLayer.html`.

### Builder methods (verified verbatim list)

`.allow_origin()`, `.allow_methods()`, `.allow_headers()`, `.allow_credentials()`,
`.expose_headers()`, `.max_age()`, `.allow_private_network()`, `.vary()`.

### Production config (explicit allowlist)

```rust
use tower_http::cors::CorsLayer;
use axum::http::{HeaderValue, Method};
use std::time::Duration;

let cors = CorsLayer::new()
    .allow_origin("https://app.example.com".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET, Method::POST])
    .allow_headers([axum::http::header::CONTENT_TYPE])
    .max_age(Duration::from_secs(3600));

let app = Router::new().route("/api", get(handler)).layer(cors);
```

Without `.max_age()`, the browser issues a preflight `OPTIONS` for every request.

### `permissive()` vs `very_permissive()` (verified, the footgun)

Verified verbatim, `CorsLayer::permissive()` provides:
"All request headers allowed", "All methods allowed", "All origins allowed",
"All headers exposed".

Verified verbatim, `CorsLayer::very_permissive()` provides:
"Credentials allowed", "The method received in `Access-Control-Request-Method` is sent
back as an allowed method", "The origin of the preflight request is sent back as an
allowed origin", "The header names received in `Access-Control-Request-Headers` are
sent back as allowed headers".

THE FOOTGUN: The CORS specification forbids the `*` wildcard origin when credentials
are allowed; browsers reject that combination. `permissive()` sends a literal wildcard
and does NOT allow credentials. `very_permissive()` exists precisely to sidestep the
wildcard-vs-credentials conflict: instead of sending `*`, it ECHOES the request's
concrete origin (and method / headers) back, which IS legal alongside
`allow_credentials(true)`.

DETERMINISTIC RULE: NEVER ship `CorsLayer::permissive()` or
`CorsLayer::very_permissive()` to a public production API — both disable the
cross-origin protection CORS exists to provide, and `very_permissive()` additionally
exposes credentialed responses to any origin. ALWAYS use `CorsLayer::new()` with an
explicit `.allow_origin(...)` allowlist in production. Reserve `permissive()` for
local development only. NEVER combine `.allow_credentials(true)` with an `Any`
wildcard origin — it is invalid CORS and browsers reject it.

VERIFICATION CAVEAT: the `CorsLayer` docs page does not print a verbatim
"do not use in production" sentence. The production warning is derived from the
documented behavior of `permissive`/`very_permissive` plus the well-established CORS
spec rule that a `*` origin is invalid with credentials. Flag this in the skill as
reasoned guidance, not a direct quote.

---

## 6. Rate Limiting (no first-party tower-http layer)

There is NO dedicated `tower-http` rate-limit layer. The skill must state this
explicitly and name the two real options.

### Option A : core `tower` limits (`tower::limit`, feature `limit`)

- `tower::limit::ConcurrencyLimitLayer` — caps the number of in-flight requests
  (concurrency, not rate).
- `tower::limit::RateLimitLayer` — caps requests per time window (true rate limiting).
  `RateLimitLayer` produces a `Service` that is NOT `Clone`, so it must be wrapped in a
  `tower::buffer::Buffer` (feature `buffer`) before it can be used where `Clone` is
  required (which Axum's `Route` is). These limits are COARSE and GLOBAL — they do not
  distinguish per-client / per-IP.

```rust
use tower::ServiceBuilder;
use tower::limit::{ConcurrencyLimitLayer, RateLimitLayer};
use std::time::Duration;

let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(BufferLayer::new(1024))                       // makes RateLimit Clone
            .layer(RateLimitLayer::new(100, Duration::from_secs(1)))
            .layer(ConcurrencyLimitLayer::new(64)),
    );
```

### Option B : `tower_governor` crate (RECOMMENDED for production)

`tower_governor` (`GovernorLayer`, built on the `governor` crate) is the community
standard for per-IP token-bucket rate limiting with Axum. It supports per-client key
extraction (per-IP by default), burst sizes, and refill rates — the things core
`tower` limits cannot do.

```rust
use tower_governor::{governor::GovernorConfigBuilder, GovernorLayer};
use std::sync::Arc;

let governor_conf = Arc::new(
    GovernorConfigBuilder::default()
        .per_second(2)
        .burst_size(5)
        .finish()
        .unwrap(),
);

let app = Router::new()
    .route("/", get(handler))
    .layer(GovernorLayer { config: governor_conf });
```

DETERMINISTIC RULE: For production per-client rate limiting ALWAYS use the
`tower_governor` ecosystem crate. Use `tower::limit` only for coarse global
concurrency / rate ceilings, and remember `RateLimitLayer` needs a `Buffer` in front
to satisfy the `Clone` bound. Document `tower_governor` as an ecosystem crate, not as
core Axum or core tower-http. (The exact `tower_governor` API is version-sensitive;
the skill should mark its snippet as illustrative and direct readers to verify against
the current `tower_governor` docs.rs page.)

---

## 7. Verified Code Snippets Summary

1. `tower::Service` trait definition (section 1) — verbatim from docs.rs.
2. `ServiceBuilder` buffer/concurrency_limit ordering example (section 3) — verbatim.
3. `ServiceBuilder` vs stacked `Router::layer()` contrast, both producing the same
   onion (section 3) — verified shapes.
4. `tower-http` layer config snippets: `TraceLayer`, `CompressionLayer`,
   `TimeoutLayer`, `RequestBodyLimitLayer` (section 4) — verified signatures.
5. Production `CorsLayer::new()` allowlist config (section 5) — verified builder
   methods.
6. Rate limiting via `tower::limit` + `Buffer` and via `tower_governor` `GovernorLayer`
   (section 6) — verified that no first-party tower-http layer exists.

---

## 8. Anti-Patterns This Skill Must Cover

1. **Confusing `ServiceBuilder` ordering with stacked `.layer()` ordering.**
   `ServiceBuilder` is top-to-bottom (first = outermost); stacked `Router::layer()` is
   bottom-to-top (last = outermost). Mixing the models silently reorders
   timeout/trace/compression. FIX: build the global stack with one `ServiceBuilder`,
   apply through one `.layer()`.
2. **`CorsLayer::permissive()` / `very_permissive()` in production.** Disables
   cross-origin protection. FIX: explicit `.allow_origin()` allowlist.
3. **`allow_credentials(true)` with a wildcard `Any` origin.** Invalid CORS, rejected
   by browsers. FIX: echo a concrete origin (`very_permissive()` does this) or use an
   explicit allowlist.
4. **Calling a `Service`'s `call` without driving `poll_ready` to
   `Poll::Ready(Ok(()))`.** The implementation is permitted to panic. FIX: honour the
   readiness contract; use `ServiceExt::oneshot` / `ready` helpers.
5. **Expecting a first-party tower-http rate-limit layer.** None exists. FIX: use
   `tower::limit` (coarse) or `tower_governor` (per-IP, recommended).
6. **`tower::limit::RateLimitLayer` without a `Buffer`.** `RateLimitLayer`'s service is
   not `Clone`; Axum requires `Clone`. FIX: wrap it in `tower::buffer::Buffer`.
7. **Forgetting the `tower-http` Cargo feature flag.** A layer behind a disabled
   feature fails to compile / is not in scope. FIX: enable `trace`, `cors`,
   `compression-*`, `timeout`, `limit`, `fs` as needed.

---

## Sources Verified (dated 2026-05-20)

All URLs fetched via WebFetch on **2026-05-20**. docs.rs current versions observed:
axum 0.8.9, tower 0.5.x, tower-http 0.6.x.

| URL | Verified | Used for |
|-----|----------|----------|
| https://docs.rs/tower/latest/tower/trait.Service.html | 2026-05-20 | Section 1 (`Service` trait definition, `poll_ready` backpressure contract, `call` panic rule) |
| https://docs.rs/tower/latest/tower/trait.Layer.html | 2026-05-20 | Section 2 (`Layer` trait shape) |
| https://docs.rs/tower/latest/tower/builder/struct.ServiceBuilder.html | 2026-05-20 | Section 3 (ServiceBuilder ordering rule, verbatim buffer/concurrency_limit example) |
| https://docs.rs/tower-http/latest/tower_http/index.html | 2026-05-20 | Section 4 (layer inventory + feature flags, `full` feature) |
| https://docs.rs/tower-http/latest/tower_http/cors/struct.CorsLayer.html | 2026-05-20 | Section 5 (CORS builder methods, `permissive` vs `very_permissive`) |
| https://docs.rs/tower-http/latest/tower_http/limit/struct.RequestBodyLimitLayer.html | 2026-05-20 | Section 4 (`RequestBodyLimitLayer::new` signature) — cross-referenced from fragment B |

### Verification notes / caveats

- The `CorsLayer` docs page did NOT print a verbatim "do not use in production"
  sentence. The production warning is reasoned from the documented behavior of
  `permissive`/`very_permissive` plus the CORS spec rule that a `*` origin is invalid
  with credentials. The skill should present this as reasoned guidance, not a quote.
- The `tower_governor` API in section 6 is illustrative; `tower_governor` is a
  third-party crate with a version-sensitive API. The skill should direct readers to
  verify the current `GovernorConfigBuilder` / `GovernorLayer` shape against the live
  `tower_governor` docs.rs page before copying.
- `tower` and `tower-http` versions (0.5.x / 0.6.x) are stated from docs.rs "latest";
  exact patch versions were not transcribed. The feature-flag names and trait
  definitions ARE verified against the fetched content.
