---
name: axum-impl-testing
description: >
  Use when writing tests for an Axum application, when a handler test panics
  with MissingJsonContentType or returns 415, when a second request against the
  same app fails to compile, or when integration tests under tests/ cannot see
  the app() function.
  Prevents reusing a consumed oneshot service, posting JSON without a
  Content-Type header, and building ad-hoc routers that drift from the real app
  wiring.
  Covers the tower ServiceExt::oneshot pattern, the axum-test TestServer API,
  http_body_util BodyExt::collect for reading response bodies, the shared app()
  constructor, and the src/lib.rs plus tests/ directory layout.
  Keywords: axum testing, TestServer, oneshot, ServiceExt, axum-test, BodyExt
  collect, into_service, ready call, MissingJsonContentType, 415 Unsupported
  Media Type, assert_status_ok, assert_json, tokio::test, how do I test an axum
  handler, my test will not compile a second request, integration test cannot
  find app, test panics on json post.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Axum Testing

## Overview

Axum testing has two complementary approaches. Both drive a `Router` without
binding a real network socket, so tests are fast and deterministic.

1. `tower::ServiceExt::oneshot` : the zero-extra-crate approach. A `Router`
   already implements `tower::Service<Request<Body>>`, so it can be called
   directly. This is what the official `examples/testing` crate uses.
2. `axum-test` `TestServer` : an ergonomic wrapper. It builds requests with
   chainable helpers (`.json()`, `.add_header()`) and exposes fluent assertion
   methods (`assert_status_ok`, `assert_json`, `assert_text`).

Neither replaces the other. ALWAYS build both kinds of test on top of ONE
shared `app()` constructor so every test exercises the real app wiring.

The `oneshot` pattern has NO crate-version coupling and works identically on
Axum 0.7 and 0.8. `axum-test` 20.0.0 depends on `axum ^0.8.8`; for an Axum 0.7
project pin an older `axum-test` release whose axum dependency matches 0.7.

## Quick Reference

| Approach | Dev-dependency | Best for |
|----------|----------------|----------|
| `oneshot` | `tower` (`util` feature), `http-body-util` | Fast handler-level unit tests, raw `Request`/`Response` control, zero extra crate |
| `TestServer` | `axum-test` | Readable integration tests, fluent assertions, cookie persistence, WebSockets |

| Need | API |
|------|-----|
| Serve exactly one request | `app.oneshot(request).await` |
| Reuse one app for many requests | `app.into_service()` then `ready(&mut app).await?.call(req).await` |
| Read a response body | `response.into_body().collect().await.unwrap().to_bytes()` |
| Async test entry point | `#[tokio::test]` |
| Build a `TestServer` | `TestServer::new(app).unwrap()` or `TestServer::try_new(app)?` |
| Execute a `TestServer` request | `server.get("/path").await` (await the `TestRequest` directly) |
| Send JSON with `TestServer` | `server.post("/path").json(&body).await` (sets `Content-Type`) |
| Assert a `TestServer` status | `response.assert_status_ok()` |

Required imports for the `oneshot` approach:

```rust
use axum::body::Body;
use axum::http::{self, Request, StatusCode};
use http_body_util::BodyExt;       // brings .collect() into scope
use tower::{Service, ServiceExt}; // brings .oneshot() / .ready() into scope
```

## Decision Trees

### Which testing approach

```
Need to test an Axum app?
├─ Want zero extra dependency, raw Request/Response control?
│  └─ Use oneshot (tower::ServiceExt).
├─ Want fluent assertions, readable integration tests?
│  └─ Use axum-test TestServer.
├─ Need cookie persistence across a request sequence?
│  └─ Use TestServer (oneshot has no cookie jar).
├─ Need to test a WebSocket route?
│  └─ Use TestServer with real HTTP transport (.http_transport()).
└─ Need many requests against one app instance?
   ├─ Using oneshot:    app.into_service() + ready/call (oneshot consumes the service).
   └─ Using TestServer: one server, call .get()/.post() repeatedly.
```

### Where the test goes

```
Writing a test?
├─ Unit test of a handler in the same src/ file?
│  └─ #[cfg(test)] mod tests block, use super::*; reaches app() directly.
└─ Integration test in a separate tests/ file?
   ├─ tests/ files are separate crates. They see ONLY the public API.
   └─ app() MUST be pub and exported from src/lib.rs, or the test cannot find it.
```

## Patterns

### Pattern 1: Shared app() constructor

Define ONE function that returns the fully configured `Router`. Call it from
every test. A test that builds its own ad-hoc router drifts from the real app
wiring (missing layers, routes, or state), so a passing test stops proving the
shipped app works.

```rust
fn app() -> Router {
    Router::new()
        .route("/", get(|| async { "Hello, World!" }))
        .route(
            "/json",
            post(|payload: Json<serde_json::Value>| async move {
                Json(serde_json::json!({ "data": payload.0 }))
            }),
        )
        .layer(TraceLayer::new_for_http())
}
```

When the app needs runtime state, give the constructor a parameter and pass a
test value from each test:

```rust
fn app(pool: PgPool) -> Router { /* ... .with_state(pool) */ }
```

### Pattern 2: oneshot unit test

`ServiceExt::oneshot` consumes the service and serves exactly one request.

```rust
#[tokio::test]
async fn hello_world() {
    let app = app();

    let response = app
        .oneshot(Request::builder().uri("/").body(Body::empty()).unwrap())
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::OK);

    let body = response.into_body().collect().await.unwrap().to_bytes();
    assert_eq!(&body[..], b"Hello, World!");
}
```

`Request::builder().uri("/").body(Body::empty()).unwrap()` and the convenience
form `Request::get("/").body(Body::empty()).unwrap()` are equivalent. Reading
the body ALWAYS goes `into_body()` then `.collect().await.unwrap().to_bytes()`;
`collect()` comes from `http_body_util::BodyExt`.

### Pattern 3: oneshot JSON test

Two rules are mandatory for a raw JSON request, because Axum's `Json` extractor
rejects a request that omits the content type with `415 Unsupported Media Type`
(`MissingJsonContentType` rejection):

- ALWAYS set `CONTENT_TYPE` to `application/json`.
- ALWAYS build the body with `serde_json::to_vec` wrapped in `Body::from`.

```rust
#[tokio::test]
async fn json() {
    let app = app();

    let response = app
        .oneshot(
            Request::builder()
                .method(http::Method::POST)
                .uri("/json")
                .header(http::header::CONTENT_TYPE, mime::APPLICATION_JSON.as_ref())
                .body(Body::from(
                    serde_json::to_vec(&serde_json::json!([1, 2, 3, 4])).unwrap(),
                ))
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::OK);

    let body = response.into_body().collect().await.unwrap().to_bytes();
    let body: serde_json::Value = serde_json::from_slice(&body).unwrap();
    assert_eq!(body, serde_json::json!({ "data": [1, 2, 3, 4] }));
}
```

### Pattern 4: Many requests against one app (oneshot)

`oneshot` consumes the service, so it cannot serve a second request. To drive
several requests against one app instance, turn the router into a service and
call it manually with `ServiceExt::ready` followed by `Service::call`:

```rust
#[tokio::test]
async fn multiple_request() {
    let mut app = app().into_service();

    let request = Request::builder().uri("/").body(Body::empty()).unwrap();
    let response = ServiceExt::<Request<Body>>::ready(&mut app)
        .await
        .unwrap()
        .call(request)
        .await
        .unwrap();
    assert_eq!(response.status(), StatusCode::OK);

    let request = Request::builder().uri("/").body(Body::empty()).unwrap();
    let response = ServiceExt::<Request<Body>>::ready(&mut app)
        .await
        .unwrap()
        .call(request)
        .await
        .unwrap();
    assert_eq!(response.status(), StatusCode::OK);
}
```

`.into_service()` produces a reusable `Service`; `ready(...).await` waits for
the service to accept a request before `.call(...)`.

### Pattern 5: TestServer integration test

`TestServer::new` builds and runs the app immediately. `.json()` sets the
`Content-Type` header automatically, so the Pattern 3 rules do not apply here.
Await the `TestRequest` directly to execute it.

```rust
use axum_test::TestServer;
use serde_json::json;

#[tokio::test]
async fn put_user() {
    let app = app();
    let server = TestServer::new(app).unwrap();

    let response = server
        .put("/users")
        .json(&json!({ "username": "Terrance Pencilworth" }))
        .await;

    response.assert_status_ok();
}
```

Assertion methods return `&Self`, so they chain:

```rust
let response = server.get("/users/1").await;
response
    .assert_status_ok()
    .assert_json(&json!({ "id": 1, "username": "Terrance Pencilworth" }));

// Or extract and inspect a typed value:
let user: User = server.get("/users/1").await.json::<User>();
```

### Pattern 6: tests/ directory layout

Files under a top-level `tests/` directory are each compiled as a separate
crate and may use ONLY the library's public API. For an integration test to
reach `app()`, the crate MUST expose it from `src/lib.rs`.

```
my-axum-app/
  Cargo.toml
  src/
    lib.rs        // pub fn app() -> Router; handlers; #[cfg(test)] unit tests
    main.rs       // fn main() { ... app() ... }  thin binary
  tests/
    api.rs        // use my_axum_app::app;  integration tests
    users.rs
```

`Cargo.toml` dev-dependencies:

```toml
[dev-dependencies]
tower          = { version = "0.5", features = ["util"] } # oneshot approach
http-body-util = "0.1"                                    # oneshot approach
mime           = "0.3"                                    # JSON oneshot tests
serde_json     = "1"
tokio          = { version = "1", features = ["full"] }   # #[tokio::test]
axum-test      = "20"  // axum 0.8 ; for axum 0.7 pin an older axum-test
```

## ALWAYS / NEVER

- ALWAYS route every test through one shared `app()` constructor. NEVER build
  an ad-hoc `Router` inside an individual test.
- ALWAYS create a fresh `app()` for each `oneshot` call, OR switch to
  `.into_service()` plus `ready`/`call`. NEVER call `oneshot` twice on the same
  service value.
- ALWAYS set `Content-Type: application/json` on a raw `oneshot` JSON request.
  NEVER post a JSON body without it; the `Json` extractor returns `415`.
- ALWAYS read a body with `into_body()` then `.collect().await.unwrap()
  .to_bytes()`. NEVER assume a body type implements `.bytes()` directly.
- ALWAYS expose `app()` as `pub` from `src/lib.rs` when integration tests live
  under `tests/`. NEVER leave it private; a `tests/` crate cannot see it.
- ALWAYS annotate each test with `#[tokio::test]`. NEVER use `#[test]` for an
  `async` test function.

## Reference Links

- `references/methods.md` : complete verified API signatures for `oneshot`,
  `into_service`, `BodyExt::collect`, `TestServer`, `TestRequest`,
  `TestResponse`, and `TestServerConfig`.
- `references/examples.md` : full working test files for both approaches,
  including state-carrying `app(pool)`, real-socket transport, and
  `MockConnectInfo`.
- `references/anti-patterns.md` : real testing mistakes with the exact failure
  each one produces and the fix.

Cross-references:

- `axum-core-architecture` : why a `Router` is a `tower::Service` and can be
  driven directly.
- `axum-impl-database` : building a test `PgPool` to pass into `app(pool)`.
