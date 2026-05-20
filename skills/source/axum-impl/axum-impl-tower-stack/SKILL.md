---
name: axum-impl-tower-stack
description: >
  Use when adding middleware to an Axum app through tower and tower-http:
  TraceLayer, CorsLayer, CompressionLayer, TimeoutLayer, RequestBodyLimitLayer,
  a ServiceBuilder stack, or rate limiting.
  Prevents the silent layer-reordering bug from confusing ServiceBuilder
  ordering (first added is outermost) with stacked Router::layer() ordering
  (last added is outermost), prevents shipping CorsLayer::permissive to
  production, prevents the cannot-find-layer compile error from a missing
  tower-http Cargo feature, and prevents expecting a tower-http rate-limit
  layer that does not exist.
  Covers the tower Service and Layer traits, the poll_ready backpressure
  contract, the ServiceBuilder ordering rule, the tower-http layer set with
  feature flags, CorsLayer configuration, and rate limiting via tower::limit
  and the tower_governor crate.
  Keywords: axum tower, tower Service trait, tower Layer trait, ServiceBuilder,
  poll_ready, tower-http, TraceLayer, CorsLayer, CompressionLayer, TimeoutLayer,
  RequestBodyLimitLayer, layer ordering, middleware order wrong, layers run in
  wrong order, cannot find TraceLayer in scope, missing cargo feature, CORS
  not working, permissive CORS production, rate limit axum, tower_governor,
  GovernorLayer, how do I add middleware, what order do layers run.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-impl-tower-stack

## Overview

Everything routable in Axum is a `tower::Service`. A `Router`, a route, a
handler-turned-service: all are `Service` values. Because of this, Axum
inherits the entire tower and tower-http middleware ecosystem for free. A
middleware is a `tower::Layer` that wraps a `Service` and produces a new one.

This skill covers the tower foundation (the `Service` and `Layer` traits),
the `ServiceBuilder` ordering rule, the common tower-http layers and their
Cargo feature flags, `CorsLayer` configuration, and rate limiting. The tower
and tower-http API used here is stable across the Axum 0.7 and 0.8 era; code
is marked `// axum 0.7 / 0.8` unless a version-specific detail applies.

`.layer()` placement on the router (register routes before layering,
`.layer()` vs `.route_layer()`) is owned by `axum-impl-middleware`. This
skill references that rule, it does not re-teach it.

## Quick Reference

### The two tower traits

```rust
// tower 0.5 - docs.rs/tower/latest/tower/trait.Service.html
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}

// tower 0.5 - docs.rs/tower/latest/tower/trait.Layer.html
pub trait Layer<S> {
    type Service;
    fn layer(&self, inner: S) -> Self::Service;
}
```

`poll_ready` implements backpressure: it signals when the service can accept
a request. `call` consumes a request and returns the response future. A
`Layer` wraps an inner `Service` and returns a wrapped one.

### The ordering rule

| Construction | Reading direction | Outermost layer |
|--------------|-------------------|-----------------|
| `ServiceBuilder::new().layer(A).layer(B).service(svc)` | top-to-bottom | `A` (first added) |
| `router.layer(A).layer(B)` | bottom-to-top | `B` (last added) |

`ServiceBuilder` and stacked `Router::layer()` order layers in OPPOSITE
directions. Verified verbatim from the `ServiceBuilder` docs: "Layers that
are added first will be called with the request first. The argument to
`service` will be last to see the request."

### tower-http layers and Cargo features

All tower-http middleware is disabled by default. Each layer needs its Cargo
feature enabled. Verified from `docs.rs/tower-http`.

| Layer | Module | Cargo feature |
|-------|--------|---------------|
| `TraceLayer` | `tower_http::trace` | `trace` |
| `CorsLayer` | `tower_http::cors` | `cors` |
| `CompressionLayer` | `tower_http::compression` | `compression-gzip` / `-br` / `-deflate` / `-zstd` / `compression-full` |
| `TimeoutLayer` | `tower_http::timeout` | `timeout` |
| `RequestBodyLimitLayer` | `tower_http::limit` | `limit` |
| `ServeDir` / `ServeFile` | `tower_http::services` | `fs` |

`features = ["full"]` enables every layer at once.

### Core rules

- ALWAYS build the global middleware stack with one `ServiceBuilder` and
  apply it through a single `.layer()` call. It removes the bottom-to-top
  stacking ambiguity.
- ALWAYS enable the matching tower-http Cargo feature for every layer used.
  A layer behind a disabled feature is not in scope and does not compile.
- ALWAYS use `CorsLayer::new()` with an explicit `.allow_origin(...)`
  allowlist in production.
- NEVER ship `CorsLayer::permissive()` or `CorsLayer::very_permissive()` to a
  public production API. Both disable cross-origin protection.
- NEVER combine `.allow_credentials(true)` with a wildcard `Any` origin. That
  is invalid CORS and browsers reject it.
- NEVER expect a first-party tower-http rate-limit layer. None exists.
- When hand-writing a `Service`, ALWAYS drive `poll_ready` to
  `Poll::Ready(Ok(()))` before calling `call`. `call` is permitted to panic
  otherwise.

## Decision Trees

### ServiceBuilder or stacked layer calls

```
Applying more than one middleware?
  YES -> build ONE ServiceBuilder, apply with ONE .layer() call.
         Write the OUTERMOST layer FIRST (top-to-bottom).
  Applying exactly one middleware?
    -> a single router.layer(X) is fine.
  Already have several stacked router.layer(A).layer(B) calls?
    -> remember: B (last) is outermost, A (first) is innermost.
       NEVER mix this mental model with ServiceBuilder order.
```

### tower::middleware::from_fn or a hand-written Layer

```
Need custom middleware logic?
  Plain async/await, app-internal, no builder config
                                  -> axum::middleware::from_fn
                                     (see axum-impl-middleware)
  Need configurable builder methods, fine-grained poll_ready
  control, or publishing it as a reusable crate
                                  -> write a Layer + Service pair
```

### Which rate-limiting approach

```
Need rate limiting?
  Per-client / per-IP token bucket, burst + refill (production)
                                  -> tower_governor crate (GovernorLayer)
  Coarse GLOBAL concurrency ceiling (in-flight request cap)
                                  -> tower::limit::ConcurrencyLimitLayer
  Coarse GLOBAL request-rate ceiling (requests per window)
                                  -> tower::limit::RateLimitLayer
                                     (MUST wrap in tower::buffer::Buffer)
```

## Patterns

### Pattern: build the global stack with one ServiceBuilder

A request must hit `timeout` (outer), then `compression`, then `trace`, then
the handler. Write the outermost layer FIRST.

```rust
// axum 0.7 / 0.8 - one ServiceBuilder, one .layer() call
use tower::ServiceBuilder;
use tower_http::{
    trace::TraceLayer, compression::CompressionLayer, timeout::TimeoutLayer,
};
use std::time::Duration;

let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(TimeoutLayer::new(Duration::from_secs(30)))  // outermost
            .layer(CompressionLayer::new())                     // middle
            .layer(TraceLayer::new_for_http()),                 // innermost
    );
```

The identical onion built with stacked `.layer()` calls reads in REVERSE:

```rust
// axum 0.7 / 0.8 - same nesting, stacked calls, written backwards
let app = Router::new()
    .route("/", get(handler))
    .layer(TraceLayer::new_for_http())                  // innermost (first)
    .layer(CompressionLayer::new())                     // middle
    .layer(TimeoutLayer::new(Duration::from_secs(30))); // outermost (last)
```

Both produce: request flows `timeout` -> `compression` -> `trace` ->
`handler`; the response flows back in reverse. ALWAYS pick the
`ServiceBuilder` form so the reading order matches the onion.

### Pattern: tower-http layer configuration

```rust
// axum 0.7 / 0.8 - verified layer constructors
use tower_http::{
    trace::TraceLayer, compression::CompressionLayer,
    timeout::TimeoutLayer, limit::RequestBodyLimitLayer,
};
use std::time::Duration;

TraceLayer::new_for_http();                          // request/response spans
CompressionLayer::new();                             // Accept-Encoding negotiation
TimeoutLayer::new(Duration::from_secs(30));          // aborts slow requests
RequestBodyLimitLayer::new(10 * 1024 * 1024);        // 10 MB body ceiling
```

`RequestBodyLimitLayer` is the explicit ceiling that pairs with
`DefaultBodyLimit::disable()`: ALWAYS set a hard ceiling after disabling
Axum's 2 MB default, because an unbounded request body is a
memory-exhaustion DoS vector. See `axum-impl-file-upload`.

### Pattern: CorsLayer production allowlist

```rust
// axum 0.7 / 0.8 - production CORS: explicit origin allowlist
use tower_http::cors::CorsLayer;
use axum::http::{HeaderValue, Method, header};
use std::time::Duration;

let cors = CorsLayer::new()
    .allow_origin("https://app.example.com".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET, Method::POST])
    .allow_headers([header::CONTENT_TYPE])
    .max_age(Duration::from_secs(3600));

let app = Router::new().route("/api", get(handler)).layer(cors);
```

`CorsLayer::permissive()` sends a literal `*` origin and does NOT allow
credentials. `CorsLayer::very_permissive()` echoes the request's concrete
origin (legal alongside credentials) but exposes credentialed responses to
any origin. NEVER ship either to a public production API. Without
`.max_age()`, the browser issues a preflight `OPTIONS` for every request.

### Pattern: rate limiting

There is NO first-party tower-http rate-limit layer.

```rust
// axum 0.7 / 0.8 - coarse GLOBAL limits via tower::limit
// RateLimitLayer's service is NOT Clone; Axum needs Clone -> wrap in Buffer.
use tower::ServiceBuilder;
use tower::buffer::BufferLayer;
use tower::limit::{ConcurrencyLimitLayer, RateLimitLayer};
use std::time::Duration;

let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(BufferLayer::new(1024))                      // makes RateLimit Clone
            .layer(RateLimitLayer::new(100, Duration::from_secs(1)))
            .layer(ConcurrencyLimitLayer::new(64)),
    );
```

For production per-client (per-IP) rate limiting, ALWAYS use the
`tower_governor` ecosystem crate (`GovernorLayer`, token bucket). Its API is
version-sensitive; verify the current `GovernorConfigBuilder` /
`GovernorLayer` shape against the live `tower_governor` docs.rs page before
copying. See `references/examples.md` for the illustrative snippet.

### Pattern: the poll_ready readiness contract

Handler-level Axum code rarely touches `poll_ready` directly: Axum and
`tower::ServiceExt::oneshot` handle the readiness dance. A hand-written
`Service` MUST honour it: ALWAYS drive `poll_ready` to `Poll::Ready(Ok(()))`
before calling `call`. `call` is documented as permitted to panic when
invoked without a ready signal. See `references/methods.md`.

## Reference Links

- `references/methods.md`: full `Service` and `Layer` trait definitions, the
  `poll_ready` and `call` contracts, `ServiceBuilder` methods, the tower-http
  layer constructors, and `CorsLayer` builder methods.
- `references/examples.md`: a complete `Cargo.toml` feature set, the full
  ServiceBuilder stack, every tower-http layer configured, production CORS,
  and both rate-limiting approaches.
- `references/anti-patterns.md`: the layer-ordering confusion, permissive
  CORS in production, the credentials-plus-wildcard CORS error, the missing
  Cargo feature, and the `RateLimitLayer` `Clone` trap, each with root cause.

Related skills: `axum-impl-middleware` (`from_fn`, `.layer()` vs
`.route_layer()` placement), `axum-impl-tracing` (`TraceLayer` customization
and `tracing-subscriber`), `axum-impl-file-upload` (`RequestBodyLimitLayer`
with `DefaultBodyLimit`), `axum-impl-static-files` (`ServeDir` / `ServeFile`
as services).
