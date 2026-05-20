# axum-impl-tower-stack: examples

Working, version-annotated code for the tower / tower-http middleware stack.
Verified against docs.rs (tower, tower-http) on 2026-05-20. The tower and
tower-http API is stable across the Axum 0.7 and 0.8 era; examples are marked
`// axum 0.7 / 0.8`. Where a snippet is from a version-sensitive third-party
crate, that is called out explicitly.

## Cargo.toml: tower-http feature set

All tower-http middleware is disabled by default. Enable the feature for
every layer used.

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower = { version = "0.5", features = ["util", "limit", "buffer", "timeout"] }
tower-http = { version = "0.6", features = [
    "trace",              # TraceLayer
    "cors",               # CorsLayer
    "compression-gzip",   # CompressionLayer (gzip)
    "timeout",            # TimeoutLayer
    "limit",              # RequestBodyLimitLayer
    "fs",                 # ServeDir / ServeFile
] }
```

`features = ["full"]` on tower-http enables every layer at once. A layer
behind a disabled feature is not in scope and does not compile.

## The global middleware stack

A request must hit `timeout` (outer), then `compression`, then `trace`, then
the handler. Build it with one `ServiceBuilder`, outermost layer FIRST.

```rust
// axum 0.7 / 0.8 - one ServiceBuilder applied through one .layer() call
use axum::{Router, routing::get};
use tower::ServiceBuilder;
use tower_http::{
    trace::TraceLayer,
    compression::CompressionLayer,
    timeout::TimeoutLayer,
};
use std::time::Duration;

async fn handler() -> &'static str { "ok" }

fn app() -> Router {
    Router::new()
        .route("/", get(handler))
        .layer(
            ServiceBuilder::new()
                .layer(TimeoutLayer::new(Duration::from_secs(30)))  // outermost
                .layer(CompressionLayer::new())                     // middle
                .layer(TraceLayer::new_for_http()),                 // innermost
        )
}
```

The same onion expressed with stacked `.layer()` calls reads in REVERSE
(last added is outermost):

```rust
// axum 0.7 / 0.8 - identical behavior, stacked calls written backwards
fn app_stacked() -> Router {
    Router::new()
        .route("/", get(handler))
        .layer(TraceLayer::new_for_http())                  // innermost
        .layer(CompressionLayer::new())                     // middle
        .layer(TimeoutLayer::new(Duration::from_secs(30)))  // outermost
}
```

ALWAYS prefer the `ServiceBuilder` form: the top-to-bottom reading order
matches the onion, and one `.layer()` call removes the stacking ambiguity.

## tower-http layers configured

```rust
// axum 0.7 / 0.8 - verified layer constructors
use tower_http::{
    trace::TraceLayer,
    compression::CompressionLayer,
    timeout::TimeoutLayer,
    limit::RequestBodyLimitLayer,
};
use std::time::Duration;

// Request and response tracing spans + events.
let trace = TraceLayer::new_for_http();

// Compresses response bodies via Accept-Encoding negotiation.
let compression = CompressionLayer::new();

// Aborts a request that exceeds the duration.
let timeout = TimeoutLayer::new(Duration::from_secs(30));

// Caps the request body length; an over-limit body becomes 413.
let body_limit = RequestBodyLimitLayer::new(10 * 1024 * 1024); // 10 MB
```

`RequestBodyLimitLayer` is the explicit ceiling that pairs with
`DefaultBodyLimit::disable()`. ALWAYS keep a hard ceiling after disabling
Axum's 2 MB default; an unbounded body is a memory-exhaustion DoS vector.

## CorsLayer: production allowlist

```rust
// axum 0.7 / 0.8 - production CORS with an explicit origin allowlist
use axum::{Router, routing::get};
use axum::http::{HeaderValue, Method, header};
use tower_http::cors::CorsLayer;
use std::time::Duration;

async fn handler() -> &'static str { "ok" }

fn app() -> Router {
    let cors = CorsLayer::new()
        .allow_origin("https://app.example.com".parse::<HeaderValue>().unwrap())
        .allow_methods([Method::GET, Method::POST])
        .allow_headers([header::CONTENT_TYPE])
        .max_age(Duration::from_secs(3600));

    Router::new().route("/api", get(handler)).layer(cors)
}
```

`CorsLayer::permissive()` is acceptable for LOCAL DEVELOPMENT only: it sends
a literal `*` origin and does not allow credentials. `very_permissive()`
echoes the request origin and allows credentials. NEVER ship either to a
public production API. Without `.max_age()`, the browser sends a preflight
`OPTIONS` for every request.

## Rate limiting: coarse global limits via tower::limit

There is NO first-party tower-http rate-limit layer. `tower::limit` provides
coarse, GLOBAL limits (not per-client).

```rust
// axum 0.7 / 0.8 - global concurrency + rate ceiling
// RateLimitLayer's service is NOT Clone; Axum needs Clone -> wrap in Buffer.
use axum::{Router, routing::get};
use tower::ServiceBuilder;
use tower::buffer::BufferLayer;
use tower::limit::{ConcurrencyLimitLayer, RateLimitLayer};
use std::time::Duration;

async fn handler() -> &'static str { "ok" }

fn app() -> Router {
    Router::new()
        .route("/", get(handler))
        .layer(
            ServiceBuilder::new()
                .layer(BufferLayer::new(1024))   // makes RateLimit Clone again
                .layer(RateLimitLayer::new(100, Duration::from_secs(1)))
                .layer(ConcurrencyLimitLayer::new(64)),
        )
}
```

Without `BufferLayer`, the build fails: `RateLimitLayer` produces a non-`Clone`
service and Axum's `Route` requires `Clone`.

## Rate limiting: per-IP via tower_governor (recommended for production)

`tower_governor` is the community standard for per-IP token-bucket rate
limiting. It is a third-party crate with a version-sensitive API. This
snippet is ILLUSTRATIVE: verify `GovernorConfigBuilder` and `GovernorLayer`
against the current `docs.rs/tower-governor` page before copying.

```rust
// axum 0.7 / 0.8 - per-IP token bucket (tower_governor, version-sensitive API)
use axum::{Router, routing::get};
use tower_governor::{governor::GovernorConfigBuilder, GovernorLayer};
use std::sync::Arc;

async fn handler() -> &'static str { "ok" }

fn app() -> Router {
    let governor_conf = Arc::new(
        GovernorConfigBuilder::default()
            .per_second(2)
            .burst_size(5)
            .finish()
            .unwrap(),
    );

    Router::new()
        .route("/", get(handler))
        .layer(GovernorLayer { config: governor_conf })
}
```

## Static files: ServeDir is a Service, not a Layer

```rust
// axum 0.7 / 0.8 - ServeDir mounts as a service, never via .layer()
use axum::Router;
use tower_http::services::ServeDir;

let app = Router::new()
    .nest_service("/assets", ServeDir::new("static"));
```

`ServeDir` and `ServeFile` implement `Service`, not `Layer`. Mount them with
`nest_service` or `fallback_service`. Full coverage: `axum-impl-static-files`.

## Hand-written Service: honour the readiness contract

When implementing a `Service` by hand, `poll_ready` must report readiness
before `call` runs.

```rust
// axum 0.7 / 0.8 - a minimal hand-written Service honouring poll_ready
use std::task::{Context, Poll};

impl<S, B> tower::Service<http::Request<B>> for MyService<S>
where
    S: tower::Service<http::Request<B>>,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = S::Future;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // ALWAYS delegate readiness to the inner service.
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: http::Request<B>) -> Self::Future {
        // Reached only after poll_ready returned Poll::Ready(Ok(())).
        self.inner.call(req)
    }
}
```

For most app-internal middleware, prefer `axum::middleware::from_fn` instead
of a hand-written `Service`. See `axum-impl-middleware`.
