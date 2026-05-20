# Topic Research : axum-impl-testing

Focused, verified research for the single skill `axum-impl-testing`. Every API
name, method signature, and code snippet below was confirmed via WebFetch
against official documentation and the canonical `tokio-rs/axum` example crate
on **2026-05-20**. Target Axum versions: **0.7** and **0.8**.

`axum-test` version at time of research: **20.0.0** (crate license: **MIT**).
Version 20.0.0 declares an axum dependency of `^0.8.8`, so it tracks Axum 0.8.
For an Axum 0.7 project, pin an older `axum-test` release whose axum dependency
matches 0.7. The `oneshot`-based approach below has NO crate-version coupling
and works identically on 0.7 and 0.8.

---

## 1. Two Verified Testing Approaches

Axum testing splits into two complementary approaches, and a skill MUST present
both because they serve different needs.

1. **`tower::ServiceExt::oneshot`** : the zero-extra-crate approach. A `Router`
   already implements `tower::Service<Request<Body>>`, so it can be driven
   directly. This is what the official `examples/testing` crate uses. Best for
   fast, in-process unit-style tests with full control over the raw
   `Request`/`Response`.
2. **`axum-test` `TestServer`** : an ergonomic E2E wrapper. It builds requests
   with chainable helpers (`.json()`, `.add_header()`), awaits them directly,
   and exposes fluent assertion methods (`assert_status_ok`, `assert_json`,
   `assert_text`). Best for readable integration tests and for features the raw
   approach lacks (cookie persistence across requests, real-socket transport,
   WebSocket testing).

Neither approach replaces the other. The skill SHOULD recommend `oneshot` for
lightweight handler-level tests and `axum-test` for higher-level integration
tests, both built on a shared `app()` constructor (section 5).

---

## 2. The `oneshot` Pattern (Verified, No Extra Crate)

The official `examples/testing/src/main.rs` test module imports exactly:

```rust
use super::*;
use axum::{
    body::Body,
    extract::connect_info::MockConnectInfo,
    http::{self, Request, StatusCode},
};
use http_body_util::BodyExt;          // brings `.collect()` into scope
use serde_json::{json, Value};
use tokio::net::TcpListener;
use tower::{Service, ServiceExt};     // brings `.oneshot()` / `.ready()` into scope
```

`ServiceExt::oneshot` consumes the service and serves exactly one request.
`http_body_util::BodyExt::collect` reads a streaming body to completion;
`.to_bytes()` then yields the buffered `Bytes`.

### Snippet 1 : verified GET test (`hello_world`)

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

`Request::builder().uri("/").body(Body::empty()).unwrap()` is the verbose form;
the convenience constructor `Request::get("/").body(Body::empty()).unwrap()` is
equivalent and also valid.

### Snippet 2 : verified JSON POST test (`json`)

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
                    serde_json::to_vec(&json!([1, 2, 3, 4])).unwrap(),
                ))
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::OK);

    let body = response.into_body().collect().await.unwrap().to_bytes();
    let body: Value = serde_json::from_slice(&body).unwrap();
    assert_eq!(body, json!({ "data": [1, 2, 3, 4] }));
}
```

Two facts the skill MUST encode as ALWAYS rules: the JSON body MUST be built
with `serde_json::to_vec` into a `Body::from(...)`, and the request MUST set
`CONTENT_TYPE` to `application/json`, otherwise Axum's `Json` extractor rejects
the request with `415 Unsupported Media Type` / a `MissingJsonContentType`
rejection.

### Snippet 3 : verified 404 test (`not_found`)

```rust
#[tokio::test]
async fn not_found() {
    let app = app();

    let response = app
        .oneshot(
            Request::builder()
                .uri("/does-not-exist")
                .body(Body::empty())
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::NOT_FOUND);
    let body = response.into_body().collect().await.unwrap().to_bytes();
    assert!(body.is_empty());
}
```

### Snippet 4 : verified multi-request pattern (`multiple_request`)

`oneshot` consumes the service, so it cannot serve a second request. When a
test needs several requests against one app instance without rebuilding it, the
official example turns the router into a service and drives it manually with
`ServiceExt::ready` followed by `Service::call`:

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
the service to be ready to accept a request before `.call(...)`.

The example also includes `the_real_deal`, which binds a real `TcpListener`,
`tokio::spawn`s `axum::serve`, and issues HTTP requests with a real client
(`hyper_util::client::legacy::Client`); and a `MockConnectInfo` test for
handlers that extract `ConnectInfo<SocketAddr>`. These are out of core scope
for this skill but worth a one-line mention.

---

## 3. The `axum-test` `TestServer` API (Verified)

`axum-test` 20.0.0 provides these main types: `TestServer` (runs the app),
`TestRequest` (builds and executes one request), `TestResponse` (the result),
plus `TestServerConfig` and `TestServerBuilder` for configuration.

### Constructors

- `TestServer::new(app)` : builds and runs the app immediately. Panics if
  construction fails; use `TestServer::try_new(app)` for a `Result`.
- `TestServer::builder()` : returns a `TestServerBuilder` for customized
  configuration before instantiation.

`new` accepts a `Router`, `IntoMakeService`, `IntoMakeServiceWithConnectInfo`,
`Serve`, `WithGracefulShutdown`, or an actix-web `App`.

### Request methods

`TestServer` exposes one method per HTTP verb plus a generic one. Each takes a
path and returns a `TestRequest`:

```rust
pub fn get(&self, path: &str) -> TestRequest
pub fn post(&self, path: &str) -> TestRequest
pub fn put(&self, path: &str) -> TestRequest
pub fn patch(&self, path: &str) -> TestRequest
pub fn delete(&self, path: &str) -> TestRequest
pub fn method(&self, method: Method, path: &str) -> TestRequest
```

A `TestRequest` is chainable (`.json(&body)`, `.add_header(...)`, etc.) and is
**awaited directly** to execute the request and yield a `TestResponse`.

### Snippet 5 : verified `TestServer` usage

```rust
use axum_test::TestServer;
use serde_json::json;

#[tokio::test]
async fn put_user() {
    let app = app();                       // shared constructor, section 5
    let server = TestServer::new(app).unwrap();

    let response = server
        .put("/users")
        .json(&json!({ "username": "Terrance Pencilworth" }))
        .await;

    response.assert_status_ok();
}
```

### TestResponse assertions and body readers

Verified assertion methods on `TestResponse` (all chainable, return `&Self`):

- `assert_status_ok()` : asserts status is `200`.
- `assert_status(StatusCode)` : asserts an exact status.
- `assert_status_not_found()` : asserts `404`.
- `assert_status_success()` : asserts a `2xx` status.
- `assert_status_failure()` : asserts a non-`2xx` status.
- `assert_json::<T>(&expected)` : deserializes the body as JSON and asserts it
  equals `expected`.
- `assert_text(expected)` : asserts the body equals the given text exactly.
- `assert_text_contains(expected)` : asserts the body contains the substring.

Verified body-reading methods: `json::<T>()` (deserialize body as JSON into
`T`), `text()` (body as a UTF-8 `String`), `as_bytes()` (`&Bytes`),
`into_bytes()` (consumes response, returns `Bytes`).

### Snippet 6 : verified assertion chaining

```rust
let response = server.get("/users/1").await;

response
    .assert_status_ok()
    .assert_json(&json!({ "id": 1, "username": "Terrance Pencilworth" }));

// Or extract and inspect:
let user: User = server.get("/users/1").await.json::<User>();
```

### Transport modes (verified)

`TestServerConfig` / `TestServerBuilder` control how the server runs:

- **Mock HTTP (default)** : no real network socket; requests are dispatched
  in-process. `server_address()` returns `None`. This is the fast default for
  unit and integration tests.
- **Real HTTP transport** : enabled via `.http_transport()` on the builder.
  Variants are `HttpRandomPort` (the OS assigns a free port) and `HttpIpPort`
  (an explicit IP and port). `server_address()` then returns `Some(Url)`. A
  real socket is REQUIRED for WebSocket testing.

`TestServerConfig` implements `Default`; the default uses mock HTTP transport
with no special expectations. The builder also configures default
success/failure expectations and cookie persistence across requests.

---

## 4. Choosing Between the Two Approaches

| Need | Use |
|------|-----|
| Fast handler-level unit test, raw `Request`/`Response` control | `oneshot` |
| Multiple requests against one app instance | `oneshot` + `.into_service()` + `ready`/`call`, or `TestServer` |
| Readable integration tests with fluent assertions | `axum-test` `TestServer` |
| Cookie persistence across a request sequence | `axum-test` `TestServer` |
| WebSocket testing | `axum-test` with real HTTP transport |
| Zero extra dependency | `oneshot` |

---

## 5. Shared `app()` Constructor and `tests/` Layout

The single most important structural rule: define ONE `app()` function that
returns the fully configured `Router`, and call it from every test. The
official example does exactly this:

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

Why this matters: tests that build their own ad-hoc router drift from the real
application wiring (missing layers, missing routes, different state), so a
passing test no longer proves the shipped app works. A shared constructor keeps
unit tests and integration tests building the *same* app. When the app needs
runtime state (a database pool, config), give the constructor a parameter:
`fn app(pool: PgPool) -> Router` and pass a test pool from each test.

### Directory layout

Two test locations, both verified as standard Cargo layout:

- **In-module unit tests** : a `#[cfg(test)] mod tests { ... }` block in the
  same `src/` file as the handlers and the `app()` function. `use super::*;`
  brings `app()` and the handlers into scope. The official example uses this
  form. Ideal for `oneshot`-based handler tests.
- **Integration tests** : separate files under a top-level `tests/` directory
  (`tests/api.rs`, `tests/users.rs`). Each file is compiled as its own crate
  and may only use the library's *public* API. For these to reach `app()`, the
  crate MUST expose it from `src/lib.rs` (for example `pub fn app() -> Router`),
  with `src/main.rs` a thin binary that calls into the library.

```
my-axum-app/
  Cargo.toml
  src/
    lib.rs        # pub fn app() -> Router; handlers; #[cfg(test)] unit tests
    main.rs       # fn main() { ... app() ... }
  tests/
    api.rs        # integration tests: use my_axum_app::app;
    users.rs
```

`Cargo.toml` dev-dependencies for the testing approaches:

```toml
[dev-dependencies]
axum-test       = "20"          # for the TestServer approach
tower           = { version = "0.5", features = ["util"] }  # ServiceExt::oneshot
http-body-util  = "0.1"         # BodyExt::collect
mime            = "0.3"         # mime::APPLICATION_JSON in the JSON test
serde_json      = "1"
tokio           = { version = "1", features = ["full"] }
```

`tower` and `http-body-util` are dev-dependencies only when using the `oneshot`
approach; `axum-test` is the dev-dependency for the `TestServer` approach.
`#[tokio::test]` (from `tokio`'s `macros`/`full` feature) makes each test an
async runtime entry point.

---

## Anti-Patterns (Testing)

- **Reusing a `oneshot` service for a second request.** `oneshot` consumes the
  service. A second `oneshot` call needs a fresh `app()`, or switch to
  `.into_service()` + `ready`/`call`.
- **Posting JSON without `Content-Type: application/json`.** Axum's `Json`
  extractor rejects the request with `415` / `MissingJsonContentType`. ALWAYS
  set the header in raw `oneshot` tests; `axum-test`'s `.json()` sets it
  automatically.
- **Building an ad-hoc router inside each test.** It diverges from the real
  app wiring. ALWAYS route every test through one shared `app()` constructor.
- **Putting integration tests under `tests/` while `app()` is private.** Files
  in `tests/` are separate crates and see only the public API. Expose `app()`
  from `src/lib.rs`.

---

## Sources Verified (2026-05-20)

All URLs below were fetched and verified on **2026-05-20**:

- https://docs.rs/axum-test/latest/axum_test/ — `axum-test` version **20.0.0**,
  MIT license, axum dependency `^0.8.8`, main types (`TestServer`,
  `TestRequest`, `TestResponse`, `TestServerConfig`, `TestServerBuilder`),
  basic `TestServer::new` + `.put().json().await` example.
- https://docs.rs/axum-test/latest/axum_test/struct.TestServer.html — verified
  signatures for `new`, `try_new`, `builder`, `get`/`post`/`put`/`patch`/
  `delete`/`method`; mock vs real HTTP transport (`HttpRandomPort`,
  `HttpIpPort`); `server_address()` behavior; `TestServerConfig` defaults.
- https://docs.rs/axum-test/latest/axum_test/struct.TestResponse.html —
  verified assertion methods (`assert_status_ok`, `assert_status`,
  `assert_status_not_found`, `assert_status_success`, `assert_status_failure`,
  `assert_json`, `assert_text`, `assert_text_contains`) and body readers
  (`json`, `text`, `as_bytes`, `into_bytes`).
- https://github.com/tokio-rs/axum/blob/main/examples/testing/src/main.rs and
  https://raw.githubusercontent.com/tokio-rs/axum/main/examples/testing/src/main.rs
  — verified `app()` constructor, test-module imports
  (`tower::{Service, ServiceExt}`, `http_body_util::BodyExt`,
  `serde_json::{json, Value}`, `axum::extract::connect_info::MockConnectInfo`),
  `oneshot` usage, `Request::builder()`/`Body::empty()`/`Body::from()`,
  `.into_body().collect().await.unwrap().to_bytes()`, the `hello_world`,
  `json`, `not_found`, `multiple_request` (`.into_service()` + `ready`/`call`),
  `the_real_deal` (real `TcpListener` + client), and `MockConnectInfo` tests.
- Research fragment C, section 7
  (`docs/research/fragments/research-c-errors-auth-db-ops.md`) — prior verified
  summary of the `axum-test` and `oneshot` testing patterns.
