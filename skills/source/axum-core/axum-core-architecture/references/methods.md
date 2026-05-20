# Axum Core Architecture: API Signatures

Complete API signatures for the architectural surface of Axum: the server entry point,
the make-service conversion, the diagnostic macro, and the `axum-core` crate boundary.
All signatures verified against the SOURCES.md approved URLs via WebFetch on 2026-05-20.

Unless a signature is explicitly annotated, it is identical on Axum 0.7 and 0.8.

## axum::serve

The glue that binds a listener to a service.

```rust
// axum 0.7 and 0.8: identical
pub fn serve<L, M, S>(listener: L, make_service: M) -> Serve<L, M, S>
where
    L: Listener,
    M: for<'a> Service<IncomingStream<'a, L>, Error = Infallible, Response = S>,
    S: Service<Request, Response = Response, Error = Infallible> + Clone + Send + 'static,
    S::Future: Send;
```

Behavior and constraints:

- Available ONLY when the `tokio` feature and at least one of `http1` or `http2` are
  enabled. All three are on by default; they matter when `default-features = false`.
- `L` is a `Listener`. In practice this is `tokio::net::TcpListener`.
- `M` is the make-service. A `Router<()>` is accepted directly. A `MethodRouter` or a
  bare `Handler` becomes a make-service via `.into_make_service()`.
- The returned `Serve<L, M, S>` is a future that resolves to `io::Result<()>` but, per
  the official docs, "will never actually complete or return an error". A socket error
  is handled by sleeping briefly (currently one second) and continuing.

## The Serve struct

`axum::serve::Serve<L, M, S>` is the future returned by `axum::serve`. It also exposes:

```rust
// axum 0.7 and 0.8: identical
impl<L, M, S> Serve<L, M, S> {
    pub fn local_addr(&self) -> io::Result<L::Addr>;

    pub fn with_graceful_shutdown<F>(self, signal: F) -> WithGracefulShutdown<L, M, S, F>
    where
        F: Future<Output = ()> + Send + 'static;
}
```

- `local_addr()` returns the address the listener is bound to. It is the correct way
  to learn the port when binding to `0.0.0.0:0`.
- `with_graceful_shutdown(signal)` returns a `WithGracefulShutdown` future. That future
  resolves to `Ok(())` ONLY after the `signal` future completes. The `signal` future
  must be `Send + 'static`.

`Serve` and `WithGracefulShutdown` implement `IntoFuture`, which is why
`axum::serve(listener, app).await` and
`axum::serve(listener, app).with_graceful_shutdown(sig).await` both compile.

## Turning a service into a make-service

```rust
// axum 0.7 and 0.8: identical

// On Router:
impl<S> Router<S> {
    pub fn into_make_service(self) -> IntoMakeService<Self>;
    pub fn into_make_service_with_connect_info<C>(self)
        -> IntoMakeServiceWithConnectInfo<Self, C>;
}

// On a bare handler, via the HandlerWithoutStateExt extension trait:
pub trait HandlerWithoutStateExt<T>: Handler<T, ()> {
    fn into_make_service(self) -> IntoMakeService<HandlerService<Self, T, ()>>;
    fn into_make_service_with_connect_info<C>(self)
        -> IntoMakeServiceWithConnectInfo<HandlerService<Self, T, ()>, C>;
}
```

- A `Router<()>` is accepted by `axum::serve` directly, so `.into_make_service()` is
  optional for a plain router.
- `.into_make_service()` is REQUIRED to serve a bare `Handler` without a `Router`.
- `into_make_service_with_connect_info::<SocketAddr>()` is REQUIRED before a handler
  can extract `ConnectInfo<SocketAddr>` (the peer address).

## #[tokio::main]

```rust
// from the `tokio` crate, NOT axum
#[tokio::main]
async fn main() { /* ... */ }
```

`#[tokio::main]` expands to a synchronous `fn main()` that builds a multi-threaded
tokio runtime and blocks on the async body. It is the runtime entry point and the only
macro a minimal Axum app uses. ALWAYS attribute its presence to tokio, not Axum.

The expanded equivalent, useful when explicit runtime tuning is needed:

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            // async main body
        });
}
```

## #[debug_handler]

```rust
// from the `axum-macros` crate, re-exported as axum::debug_handler
// requires the `macros` feature
use axum::debug_handler;

#[debug_handler]
async fn handler(/* extractors */) -> impl axum::response::IntoResponse { /* ... */ }
```

- `#[debug_handler]` generates code that produces a precise, plain-language compiler
  error when a function fails the `Handler` trait bounds, instead of the cryptic
  `Handler is not satisfied` message.
- It is a development-only diagnostic aid. ALWAYS remove it once the handler compiles.
- A companion `#[debug_middleware]` exists for `from_fn` middleware functions.

## The axum-core crate boundary

`axum-core` is a separate crate. The `axum` crate depends on it and re-exports its
traits. `axum-core` holds the foundational traits and types:

```rust
// re-exported by `axum` from `axum-core`
axum_core::extract::FromRequest        // body-consuming extractor trait
axum_core::extract::FromRequestParts   // parts-only extractor trait
axum_core::response::IntoResponse      // value to response conversion
axum_core::response::IntoResponseParts // headers and extensions, no status or body
axum_core::body::Body                  // the response body type
axum_core::Error                       // the boxed error type
```

What lives in `axum` and NOT in `axum-core`: `Router`, `MethodRouter`, the concrete
extractors (`Path`, `Query`, `Json`, `State`, and the rest), the `Handler` trait, and
`axum::serve`.

Dependency rule:

- An application binary: `axum = "0.8"`.
- A library whose only Axum surface is `FromRequest` / `FromRequestParts` /
  `IntoResponse` implementations: `axum-core = "0.5"` (the `axum-core` version that
  pairs with `axum` 0.8; `axum` 0.7 pairs with `axum-core` 0.4).

## Feature flags relevant to architecture

| Feature | Default | Effect |
|---------|---------|--------|
| `tokio` | on | enables `axum::serve` and the tokio integration |
| `http1` | on | HTTP/1 support, one of `http1` / `http2` is required by `serve` |
| `http2` | on | HTTP/2 support |
| `macros` | off | enables `#[debug_handler]` and the derive macros |

When `default-features = false` is set, ALWAYS re-enable `tokio` and at least one of
`http1` / `http2`, or `axum::serve` will not be available.

## Version note: axum::Server

`axum::Server` was the pre-0.7 entry point. It was deprecated in 0.7 and REMOVED in
0.8. The only correct entry point for both 0.7 and 0.8 is `axum::serve`. See
`references/examples.md` for the migration.

## Sources verified

All fetched via WebFetch on 2026-05-20:

- https://docs.rs/axum/latest/axum/fn.serve.html : the `serve` signature, trait
  bounds, feature gates, and the "never completes" note.
- https://docs.rs/axum/latest/axum/serve/struct.Serve.html : `local_addr` and
  `with_graceful_shutdown` signatures.
- https://docs.rs/axum/latest/axum/ : crate overview, the `axum-core` library-author
  recommendation, the `#[debug_handler]` description, hyper and http version
  requirements.
- https://docs.rs/axum-macros/latest/axum_macros/ : `debug_handler` and the derive
  macros.
