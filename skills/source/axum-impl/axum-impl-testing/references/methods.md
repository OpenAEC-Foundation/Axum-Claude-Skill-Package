# Axum Testing : API Reference

All signatures verified on 2026-05-20 against the official Axum `examples/testing`
crate and the `axum-test` 20.0.0 documentation on docs.rs. Approaches are valid
for Axum 0.7 and 0.8 unless a version annotation states otherwise.

## 1. The oneshot approach (tower)

### tower::ServiceExt::oneshot

```rust
fn oneshot(self, req: Request) -> Oneshot<Self, Request>
```

A `Router` implements `tower::Service<Request<Body>>`, so `oneshot` is callable
directly on it. `oneshot` CONSUMES the service and serves exactly one request.
The returned future resolves to `Result<Response, Infallible>`; in tests it is
unwrapped with `.await.unwrap()`.

Brought into scope by `use tower::ServiceExt;`. Requires the `util` feature on
the `tower` dev-dependency.

### Router::into_service

```rust
pub fn into_service(self) -> IntoService<Self, B>
```

Converts a `Router` into a reusable `Service`. Use it when one app instance
must serve multiple requests, because `oneshot` cannot be called twice.

### tower::ServiceExt::ready and Service::call

```rust
fn ready(&mut self) -> Ready<'_, Self, Request>          // tower::ServiceExt
fn call(&mut self, req: Request) -> Self::Future          // tower::Service
```

`ready(&mut app).await` waits until the service can accept a request, then
`call(request).await` dispatches it. Both are needed for the multi-request
pattern. Disambiguate the request type when the compiler needs it:

```rust
ServiceExt::<Request<Body>>::ready(&mut app).await.unwrap().call(request).await.unwrap()
```

`Service` and `ServiceExt` must both be in scope: `use tower::{Service, ServiceExt};`.

### Request construction (http crate)

```rust
Request::builder()                       // verbose builder
    .method(http::Method::POST)
    .uri("/path")
    .header(http::header::CONTENT_TYPE, mime::APPLICATION_JSON.as_ref())
    .body(Body::from(bytes))
    .unwrap()

Request::get("/path").body(Body::empty()).unwrap()    // convenience constructor
Request::post("/path").body(Body::from(bytes)).unwrap()
```

`Request::get`, `Request::post`, etc. are shortcuts for
`Request::builder().method(...).uri(...)`. Both forms are valid.

### Body construction (axum::body)

```rust
Body::empty()                                    // no body
Body::from(serde_json::to_vec(&value).unwrap())  // JSON body, build with to_vec
Body::from("plain text")                         // text body
```

### http_body_util::BodyExt::collect

```rust
fn collect(self) -> Collect<Self>
```

Reads a streaming body to completion. The result exposes `.to_bytes()` which
yields the buffered `Bytes`. Standard body read:

```rust
let bytes = response.into_body().collect().await.unwrap().to_bytes();
```

Brought into scope by `use http_body_util::BodyExt;`.

### #[tokio::test]

Replaces `#[test]` for `async` test functions; it starts a Tokio runtime as the
test entry point. Requires the `macros` or `full` feature on the `tokio`
dev-dependency.

## 2. The axum-test approach

`axum-test` 20.0.0, crate license MIT. Depends on `axum ^0.8.8` (tracks Axum
0.8). For an Axum 0.7 project, pin an older `axum-test` release whose axum
dependency matches 0.7.

### TestServer constructors

```rust
pub fn new(app: A) -> Result<TestServer>          // panics via .unwrap() if build fails
pub fn try_new(app: A) -> Result<TestServer>      // returns Result, no panic
pub fn builder() -> TestServerBuilder             // configure before building
```

`new` accepts a `Router`, `IntoMakeService`, `IntoMakeServiceWithConnectInfo`,
`Serve`, `WithGracefulShutdown`, or an actix-web `App`.

### TestServer request methods

Each takes a path and returns a `TestRequest`:

```rust
pub fn get(&self, path: &str) -> TestRequest
pub fn post(&self, path: &str) -> TestRequest
pub fn put(&self, path: &str) -> TestRequest
pub fn patch(&self, path: &str) -> TestRequest
pub fn delete(&self, path: &str) -> TestRequest
pub fn method(&self, method: Method, path: &str) -> TestRequest
```

### TestServer transport

```rust
pub fn server_address(&self) -> Option<Url>
```

Returns `None` in mock HTTP mode (the default, in-process dispatch, no socket)
and `Some(Url)` when real HTTP transport is enabled. `server_address()` is the
way to obtain the bound URL for a real-socket test.

### TestRequest

A `TestRequest` is chainable and is AWAITED DIRECTLY to execute the request and
yield a `TestResponse`. Verified chainable builders:

```rust
.json(&body)          // serialize body as JSON, sets Content-Type: application/json
.add_header(name, value)
.content_type(mime_str)
```

### TestResponse assertion methods

All assertion methods take `&self` and return `&Self`, so they chain. They
panic on failure.

```rust
pub fn assert_status(&self, expected: StatusCode) -> &Self
pub fn assert_not_status(&self, expected: StatusCode) -> &Self
pub fn assert_status_ok(&self) -> &Self            // 200
pub fn assert_status_not_ok(&self) -> &Self
pub fn assert_status_success(&self) -> &Self       // any 2xx
pub fn assert_status_failure(&self) -> &Self       // any non-2xx
pub fn assert_status_not_found(&self) -> &Self     // 404
pub fn assert_json<T>(&self, expected: &T) -> &Self
    where T: Serialize + DeserializeOwned + PartialEq + Debug
pub fn assert_text<C>(&self, expected: C) -> &Self
    where C: AsRef<str>
pub fn assert_text_contains<C>(&self, expected: C) -> &Self
    where C: AsRef<str>
```

### TestResponse body readers

These extract content without asserting:

```rust
pub fn json<T>(&self) -> T where T: DeserializeOwned   // deserialize body as JSON
pub fn text(&self) -> String                           // body as UTF-8 String
pub fn as_bytes(&self) -> &Bytes                        // borrow raw bytes
pub fn into_bytes(self) -> Bytes                        // consume, return raw bytes
```

### TestServerBuilder

Obtained from `TestServer::builder()`. Configures behavior before the server is
built. Verified methods:

```rust
pub fn build<A>(self, app: A) -> TestServer        // panics if build fails
pub fn try_build<A>(self, app: A) -> Result<TestServer>
pub fn http_transport(self) -> Self                // real HTTP socket
pub fn http_transport_with_ip_port(self, ip: Option<IpAddr>, port: Option<u16>) -> Self
pub fn mock_transport(self) -> Self                // default in-process transport
pub fn save_cookies(self) -> Self                  // cookie persistence across requests
pub fn do_not_save_cookies(self) -> Self
pub fn default_content_type(self, content_type: &str) -> Self
pub fn expect_success_by_default(self) -> Self
```

A real socket via `http_transport()` is REQUIRED for WebSocket testing.

Typical usage:

```rust
let server = TestServer::builder()
    .http_transport()
    .save_cookies()
    .build(app());
```

### TestServerConfig and new_with_config

`TestServerConfig` implements `Default`. The default uses mock HTTP transport
(in-process, no socket) with no special success or failure expectations.

```rust
pub fn new_with_config<A, C>(app: A, config: C) -> TestServer
    where C: Into<TestServerConfig>
```

`new_with_config` builds a `TestServer` from an explicit config; the builder
above is the more common path.

## 3. The shared app() constructor

```rust
fn app() -> Router { /* full router with routes, layers, state */ }
fn app(pool: PgPool) -> Router { /* state-carrying variant */ }
```

ONE constructor used by both unit tests and integration tests guarantees every
test exercises the same wiring as the shipped binary. For integration tests
under `tests/`, declare it `pub fn app(...)` and export it from `src/lib.rs`.

## Sources

- https://github.com/tokio-rs/axum/blob/main/examples/testing/src/main.rs
- https://docs.rs/axum-test/latest/axum_test/
- https://docs.rs/axum-test/latest/axum_test/struct.TestServer.html
- https://docs.rs/axum-test/latest/axum_test/struct.TestResponse.html
- https://docs.rs/tower/latest/tower/
