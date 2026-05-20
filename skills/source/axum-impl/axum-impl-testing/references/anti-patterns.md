# Axum Testing : Anti-Patterns

Each entry is a real mistake, the exact failure it produces, and the fix. Verified
against the official Axum `examples/testing` crate and `axum-test` 20.0.0 on
2026-05-20.

## 1. Reusing a oneshot service for a second request

WRONG:

```rust
let app = app();
let r1 = app.clone().oneshot(req1).await.unwrap(); // forces a Clone bound
let r2 = app.oneshot(req2).await.unwrap();
```

WHY THIS FAILS: `ServiceExt::oneshot` CONSUMES the service (`self` by value) and
serves exactly one request. Calling `oneshot` on a value that was already moved
is a compile error (`use of moved value`). Cloning the router on every request
is wasteful and hides the real intent.

FIX: Build a fresh `app()` per `oneshot` call, or, for many requests against one
instance, use `.into_service()` plus `ready`/`call`:

```rust
let mut app = app().into_service();
let r1 = ServiceExt::<Request<Body>>::ready(&mut app).await.unwrap()
    .call(req1).await.unwrap();
let r2 = ServiceExt::<Request<Body>>::ready(&mut app).await.unwrap()
    .call(req2).await.unwrap();
```

## 2. Posting JSON without a Content-Type header

WRONG:

```rust
let response = app
    .oneshot(
        Request::builder()
            .method(http::Method::POST)
            .uri("/json")
            .body(Body::from(serde_json::to_vec(&payload).unwrap()))
            .unwrap(),
    )
    .await
    .unwrap();
assert_eq!(response.status(), StatusCode::OK); // FAILS
```

WHY THIS FAILS: Axum's `Json` extractor checks the request `Content-Type`. With
the header missing it rejects the request with `415 Unsupported Media Type`
(the `MissingJsonContentType` rejection), so the handler never runs and the
status assertion fails.

FIX: ALWAYS set `CONTENT_TYPE` to `application/json` on a raw `oneshot` JSON
request:

```rust
.header(http::header::CONTENT_TYPE, mime::APPLICATION_JSON.as_ref())
```

`axum-test`'s `.json(&body)` sets this header automatically, so the mistake
cannot happen with the `TestServer` approach.

## 3. Building an ad-hoc router inside each test

WRONG:

```rust
#[tokio::test]
async fn test_handler() {
    // A throwaway router, NOT the real app.
    let app = Router::new().route("/", get(my_handler));
    let response = app.oneshot(req).await.unwrap();
}
```

WHY THIS FAILS: The ad-hoc router drifts from the real application wiring. It
misses the layers, fallback routes, and shared state that the shipped app has.
The test then passes even when the real app is broken (for example an auth
layer that rejects the request in production is absent from the test router).
The test no longer proves anything about what ships.

FIX: Route every test through ONE shared `app()` constructor that returns the
fully configured `Router`. Unit tests and integration tests then build the same
app.

## 4. Integration tests under tests/ while app() is private

WRONG: `app()` defined as a private `fn app()` in `src/main.rs`, with an
integration test in `tests/api.rs` that does `use my_axum_app::app;`.

WHY THIS FAILS: Every file under the top-level `tests/` directory is compiled
as its OWN crate and can use ONLY the library crate's PUBLIC API. A private
`app()`, or an `app()` that exists only in `src/main.rs` (a binary, not the
library), is invisible to `tests/`. The result is a compile error:
`function app is private` or `unresolved import`.

FIX: Declare `pub fn app() -> Router` in `src/lib.rs`. Keep `src/main.rs` a
thin binary that calls `my_axum_app::app()`. Integration tests then import the
public function.

## 5. Using #[test] instead of #[tokio::test]

WRONG:

```rust
#[test]
async fn hello_world() { /* ... */ }
```

WHY THIS FAILS: A plain `#[test]` function cannot be `async`; there is no
runtime to poll the future. The compiler reports that the `async` test function
is not a valid test, or the test silently does nothing because the future is
never awaited.

FIX: ALWAYS annotate an async test with `#[tokio::test]`. It starts a Tokio
runtime as the test entry point. Requires the `macros` or `full` feature on the
`tokio` dev-dependency.

## 6. Forgetting to import ServiceExt or BodyExt

WRONG:

```rust
use tower::Service;          // ServiceExt missing
let response = app.oneshot(req).await.unwrap();      // error: no method oneshot
let body = response.into_body().collect().await;     // error: no method collect
```

WHY THIS FAILS: `.oneshot()` and `.ready()` are provided by the EXTENSION trait
`tower::ServiceExt`, and `.collect()` by `http_body_util::BodyExt`. An extension
method is only callable when its trait is in scope. Without the `use`, the
compiler reports `no method named oneshot` / `no method named collect`.

FIX: Import the extension traits:

```rust
use tower::{Service, ServiceExt}; // Service for call(), ServiceExt for oneshot()/ready()
use http_body_util::BodyExt;       // collect()
```

## 7. Missing the util feature on the tower dev-dependency

WRONG:

```toml
[dev-dependencies]
tower = "0.5"   # no features
```

WHY THIS FAILS: `ServiceExt::oneshot` and `ServiceExt::ready` live behind
`tower`'s `util` feature. Without it, importing `tower::ServiceExt` either fails
or the methods are absent, and `oneshot` does not resolve.

FIX: Enable the feature:

```toml
[dev-dependencies]
tower = { version = "0.5", features = ["util"] }
```

## 8. Reading a response body without collect()

WRONG:

```rust
let bytes = response.body().to_bytes();    // no such method on a streaming body
```

WHY THIS FAILS: An Axum response body is a STREAM, not an already-buffered
buffer. It has no direct `.to_bytes()`. The bytes exist only after the stream
is read to completion.

FIX: ALWAYS take ownership with `into_body()`, read the stream with
`BodyExt::collect`, then call `.to_bytes()`:

```rust
let bytes = response.into_body().collect().await.unwrap().to_bytes();
```

With `axum-test` use the `TestResponse` readers instead: `.text()`,
`.json::<T>()`, `.as_bytes()`, `.into_bytes()`.

## 9. Using oneshot for cookie or WebSocket tests

WRONG: Trying to assert that a cookie set by request 1 is sent back on request
2 using two separate `oneshot` calls, or attempting a WebSocket handshake
against a `oneshot` router.

WHY THIS FAILS: `oneshot` has no cookie jar; each request is independent, so a
`Set-Cookie` from the first response is never replayed. It also has no real
socket, so a WebSocket upgrade has nothing to connect to.

FIX: Use `axum-test` `TestServer`. `save_cookies()` on the builder persists
cookies across requests, and `.http_transport()` provides the real socket a
WebSocket handshake requires.

## 10. Mismatched axum-test and axum versions

WRONG: An Axum 0.7 project with `axum-test = "20"` in dev-dependencies.

WHY THIS FAILS: `axum-test` 20.0.0 declares an axum dependency of `^0.8.8`. Cargo
then pulls in Axum 0.8 alongside the project's Axum 0.7, producing two
incompatible `Router` types and trait-mismatch errors when the test passes the
0.7 `Router` to a `TestServer` expecting the 0.8 one.

FIX: Match the `axum-test` major version to the project's Axum version. For
Axum 0.8 use `axum-test = "20"`. For Axum 0.7 pin an older `axum-test` release
whose axum dependency resolves to 0.7. The `oneshot` approach has no such
coupling and works identically on both Axum 0.7 and 0.8.
