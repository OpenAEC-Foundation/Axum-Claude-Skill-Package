# axum-impl-tower-stack: methods

Complete API signatures for the tower and tower-http surface used with Axum.
Verified via WebFetch against docs.rs on 2026-05-20. docs.rs versions
observed: axum 0.8.9, tower 0.5.x, tower-http 0.6.x. The tower and tower-http
API is stable across the Axum 0.7 and 0.8 era.

## The tower::Service trait

```rust
// docs.rs/tower/latest/tower/trait.Service.html
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

| Item | Meaning |
|------|---------|
| `Response` | The success type produced by the service. |
| `Error` | The error type. Every Router-backed Axum service has `Error = Infallible`. |
| `Future` | The future `call` returns; resolves to `Result<Response, Error>`. |

### poll_ready: the backpressure contract

`poll_ready` implements backpressure and readiness. Verified verbatim from
the trait docs:

- "Returns `Poll::Ready(Ok(()))` when the service is able to process requests."
- "Before dispatching a request, `poll_ready` must be called and return
  `Poll::Ready(Ok(()))`."
- "Until a request is dispatched, repeated calls to `poll_ready` must return
  either `Poll::Ready(Ok(()))` or `Poll::Ready(Err(_))`."
- `Poll::Pending`: the service cannot yet process requests; the task is
  notified when it becomes ready again.
- `Poll::Ready(Err(_))`: the service can no longer handle requests.

### call: process the request

`call` consumes a request and returns a future resolving to
`Result<Response, Error>`. Verified verbatim: "Implementations are permitted
to panic if `call` is invoked without obtaining `Poll::Ready(Ok(()))` from
`poll_ready`."

DETERMINISTIC RULE: ALWAYS drive `poll_ready` to `Poll::Ready(Ok(()))` before
calling `call`. In handler-level Axum code Axum and
`tower::ServiceExt::oneshot` handle this; a hand-written `Service` MUST
honour it.

## The tower::Layer trait

```rust
// docs.rs/tower/latest/tower/trait.Layer.html
pub trait Layer<S> {
    type Service;
    fn layer(&self, inner: S) -> Self::Service;
}
```

A `Layer` decorates a `Service`: `.layer(inner)` produces a wrapped
`Service`. `Router::layer(L)` accepts any `L: Layer<Route>` that is
`Clone + Send + Sync + 'static`. A hand-written middleware is a
`Layer` + `Service` pair: the `Layer` builds the `Service`, the `Service`
carries the `poll_ready` / `call` behavior.

DECISION RULE: use `axum::middleware::from_fn` for app-internal middleware in
familiar async/await. Write a real `Layer` + `Service` pair when configurable
builder methods, fine-grained `poll_ready` control, or a publishable reusable
crate are required.

## ServiceBuilder

```rust
// docs.rs/tower/latest/tower/builder/struct.ServiceBuilder.html
ServiceBuilder::new()           // -> ServiceBuilder<Identity>
    .layer(layer)               // add any tower::Layer
    .timeout(duration)          // convenience: add a timeout layer
    .buffer(bound)              // convenience: add a Buffer layer
    .concurrency_limit(max)     // convenience: add a ConcurrencyLimit layer
    .rate_limit(num, per)       // convenience: add a RateLimit layer
    .service(svc);              // wrap the leaf service, return the composed Service
```

### The ordering rule, verified verbatim

"The order in which layers are added impacts how requests are handled.
Layers that are added first will be called with the request first. The
argument to `service` will be last to see the request."

Verified worked example:

```rust
ServiceBuilder::new()
    .buffer(100)             // added first  -> outermost, sees the request first
    .concurrency_limit(10)   // added second -> inner
    .service(svc);           // the leaf, sees the request last
```

THE RULE: in `ServiceBuilder`, the FIRST `.layer()` added is the OUTERMOST
layer (top-to-bottom). This is the OPPOSITE of stacked `Router::layer()`
calls, where the LAST `.layer()` added is the outermost (bottom-to-top).

| Construction | Outermost |
|--------------|-----------|
| `ServiceBuilder::new().layer(A).layer(B).service(svc)` | `A` |
| `router.layer(A).layer(B)` | `B` |

## tower-http layer constructors

All tower-http middleware is disabled by default; each layer needs its Cargo
feature. Verified from `docs.rs/tower-http/latest/tower_http/index.html`.

| Layer | Constructor | Feature |
|-------|-------------|---------|
| `TraceLayer` | `TraceLayer::new_for_http()` | `trace` |
| `CorsLayer` | `CorsLayer::new()` / `permissive()` / `very_permissive()` | `cors` |
| `CompressionLayer` | `CompressionLayer::new()` | `compression-gzip` / `-br` / `-deflate` / `-zstd` / `compression-full` |
| `TimeoutLayer` | `TimeoutLayer::new(Duration)` | `timeout` |
| `RequestBodyLimitLayer` | `RequestBodyLimitLayer::new(limit: usize)` | `limit` |
| `ServeDir` | `ServeDir::new(path)` | `fs` |
| `ServeFile` | `ServeFile::new(path)` | `fs` |

`features = ["full"]` enables every layer.

`ServeDir` and `ServeFile` are `Service`s, NOT `Layer`s. Mount them with
`Router::nest_service` or `Router::fallback_service`, never `.layer()`. Full
coverage is in `axum-impl-static-files`.

## CorsLayer builder methods

Verified from `docs.rs/tower-http/latest/tower_http/cors/struct.CorsLayer.html`.

| Method | Purpose |
|--------|---------|
| `.allow_origin(...)` | Set the allowed origin(s). Accepts a `HeaderValue`, a list, or `Any`. |
| `.allow_methods(...)` | Set the allowed HTTP methods. |
| `.allow_headers(...)` | Set the allowed request headers. |
| `.allow_credentials(bool)` | Allow credentialed requests (cookies, auth headers). |
| `.expose_headers(...)` | Headers the browser may expose to JS. |
| `.max_age(Duration)` | How long a preflight result is cached. |
| `.allow_private_network(bool)` | Allow private-network preflight. |
| `.vary(...)` | Customize the `Vary` response header. |

Verified preset behavior:

- `CorsLayer::permissive()`: all request headers allowed, all methods
  allowed, all origins allowed (literal `*`), all headers exposed. Does NOT
  allow credentials.
- `CorsLayer::very_permissive()`: credentials allowed; the requested method,
  origin, and headers are echoed back as allowed. Legal alongside
  `allow_credentials(true)` because it never sends a literal `*`.

DETERMINISTIC RULE: NEVER ship `permissive()` or `very_permissive()` to a
public production API. ALWAYS use `CorsLayer::new()` with an explicit
`.allow_origin(...)` allowlist. NEVER combine `.allow_credentials(true)` with
an `Any` wildcard origin: the CORS spec forbids `*` with credentials and
browsers reject it.

## Rate-limiting API (no first-party tower-http layer)

There is NO tower-http rate-limit layer. The two real options:

```rust
// core tower (feature `limit`, plus `buffer` for the Buffer)
tower::limit::ConcurrencyLimitLayer::new(max: usize);   // coarse global in-flight cap
tower::limit::RateLimitLayer::new(num: u64, per: Duration); // coarse global rate cap
```

`RateLimitLayer` produces a `Service` that is NOT `Clone`. Axum's `Route`
requires `Clone`, so `RateLimitLayer` MUST be wrapped in
`tower::buffer::Buffer` (`BufferLayer`, feature `buffer`) before use. Both
`tower::limit` layers are coarse and GLOBAL; they do not distinguish
per-client / per-IP.

```rust
// ecosystem crate `tower_governor` - per-IP token bucket (RECOMMENDED)
tower_governor::GovernorLayer { config: Arc<GovernorConfig> };
```

`tower_governor` is a third-party crate with a version-sensitive API. Verify
`GovernorConfigBuilder` / `GovernorLayer` against the live `tower_governor`
docs.rs page before copying.

## Sources

Verified via WebFetch on 2026-05-20:

- https://docs.rs/tower/latest/tower/trait.Service.html
- https://docs.rs/tower/latest/tower/trait.Layer.html
- https://docs.rs/tower/latest/tower/builder/struct.ServiceBuilder.html
- https://docs.rs/tower-http/latest/tower_http/index.html
- https://docs.rs/tower-http/latest/tower_http/cors/struct.CorsLayer.html
- https://docs.rs/tower-http/latest/tower_http/limit/struct.RequestBodyLimitLayer.html
