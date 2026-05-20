# Axum Core Architecture: Examples

Working, version-annotated code for the architectural surface of Axum. Every snippet
is verified against the SOURCES.md approved URLs. Code that is identical on Axum 0.7
and 0.8 is annotated as such; divergent code carries `// axum 0.8` or `// axum 0.7`.

## Example 1: the minimal app

```rust
// axum 0.7 and 0.8: byte-for-byte identical
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "Hello, World!" }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

The `#[tokio::main]` macro builds a tokio runtime and blocks on the async body.
`axum::serve` never returns, so `main` runs the server until the process is killed.
ALWAYS write this entry point once, with no version `cfg`.

`Cargo.toml`:

```toml
[dependencies]
axum = "0.8"                                  # or "0.7"
tokio = { version = "1", features = ["full"] }
```

## Example 2: graceful shutdown

```rust
// axum 0.7 and 0.8: identical
use axum::{routing::get, Router};
use tokio::signal;

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "ok" }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}
```

`with_graceful_shutdown` makes the `serve` future resolve `Ok(())` once the signal
future completes. SIGTERM is the stop signal Docker, Kubernetes, and systemd send.
ALWAYS handle both `ctrl_c()` and SIGTERM. The deeper deployment workflow, including
closing the database pool and flushing tracing on shutdown, lives in the
`axum-impl-deployment` skill.

## Example 3: a Router IS a tower::Service

```rust
// axum 0.7 and 0.8: identical
use axum::{routing::get, Router};
use axum::body::Body;
use axum::http::{Request, StatusCode};
use tower::ServiceExt; // brings `oneshot` into scope

fn app() -> Router {
    Router::new().route("/health", get(|| async { "alive" }))
}

#[tokio::test]
async fn health_route_responds_ok() {
    // `app()` is a Router, and a Router<()> implements tower::Service.
    // `oneshot` drives one request through it with no network and no `axum::serve`.
    let response = app()
        .oneshot(
            Request::builder()
                .uri("/health")
                .body(Body::empty())
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::OK);
}
```

Because the router is a `tower::Service`, it can be served, nested, merged, and tested
without a network socket. The full testing workflow is in the `axum-impl-testing`
skill. `oneshot` comes from the `tower` crate's `ServiceExt` trait.

## Example 4: a handler is just an async fn returning IntoResponse

```rust
// axum 0.7 and 0.8: identical
use axum::http::StatusCode;
use axum::response::IntoResponse;
use axum::Json;

// Valid handler: declared async, return type implements IntoResponse.
async fn hello() -> &'static str {
    "hello"
}

// Valid handler: a StatusCode plus a Json body, all IntoResponse.
async fn created() -> impl IntoResponse {
    (StatusCode::CREATED, Json(serde_json::json!({ "ok": true })))
}
```

There is no routing macro. A handler is registered as data:
`Router::new().route("/", get(hello))`. Eligibility is enforced entirely by the
`Handler` trait bounds. See `axum-syntax-handlers` for the full rules.

## Example 5: #[debug_handler] decodes the cryptic Handler error

```rust
// axum 0.7 and 0.8: identical. Requires the `macros` feature on axum.
use axum::debug_handler;
use axum::response::IntoResponse;
use axum::http::StatusCode;

// This function fails the Handler bound: a bare `bool` is not an extractor.
// Without #[debug_handler] the compiler says only "Handler is not satisfied".
// With #[debug_handler] the compiler names the offending argument precisely.
#[debug_handler]
async fn broken(flag: bool) -> impl IntoResponse {
    if flag { StatusCode::OK } else { StatusCode::BAD_REQUEST }
}
```

`Cargo.toml`:

```toml
[dependencies]
axum = { version = "0.8", features = ["macros"] }
```

Workflow: add `#[debug_handler]`, recompile, read the precise error, fix the root
cause, then remove `#[debug_handler]`. It is a development-only aid.

## Example 6: an axum-core library extractor

A reusable library that only implements a custom extractor depends on `axum-core`, not
on the full `axum` crate. This insulates it from router and extractor breaking changes.

```rust
// axum 0.8: a library crate. Cargo.toml -> axum-core = "0.5"
use axum_core::extract::FromRequestParts;
use axum_core::response::IntoResponse;
use http::request::Parts;
use http::StatusCode;

pub struct ExtractApiVersion(pub String);

impl<S: Send + Sync> FromRequestParts<S> for ExtractApiVersion {
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        parts
            .headers
            .get("x-api-version")
            .and_then(|v| v.to_str().ok())
            .map(|v| ExtractApiVersion(v.to_owned()))
            .ok_or((StatusCode::BAD_REQUEST, "missing x-api-version header"))
    }
}
```

```rust
// axum 0.7: the SAME extractor needs the #[async_trait] attribute.
// The library would depend on axum-core = "0.4" and the async-trait crate.
#[axum::async_trait]
impl<S: Send + Sync> FromRequestParts<S> for ExtractApiVersion {
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        parts
            .headers
            .get("x-api-version")
            .and_then(|v| v.to_str().ok())
            .map(|v| ExtractApiVersion(v.to_owned()))
            .ok_or((StatusCode::BAD_REQUEST, "missing x-api-version header"))
    }
}
```

The body is identical between versions; only the `#[async_trait]` attribute differs.
Axum 0.8 uses native `async fn` in traits (RPITIT, MSRV Rust 1.75). The full custom
extractor workflow is in the `axum-syntax-custom-extractors` skill.

## Example 7: migrating axum::Server to axum::serve

```rust
// axum 0.6 (pre-0.7): the removed axum::Server API. Do NOT use on 0.7 or 0.8.
#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "ok" }));
    axum::Server::bind(&"0.0.0.0:3000".parse().unwrap())
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

```rust
// axum 0.7 and 0.8: the correct entry point. axum::Server was removed in 0.8.
#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "ok" }));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

The migration: bind a `tokio::net::TcpListener` first, then pass the listener and the
router to `axum::serve`. A `Router<()>` no longer needs `.into_make_service()` for
plain serving. The complete 0.7 to 0.8 breaking-change matrix is in the
`axum-core-version-migration` skill.

## Example 8: extracting the peer address with ConnectInfo

```rust
// axum 0.7 and 0.8: identical
use axum::{routing::get, Router};
use axum::extract::ConnectInfo;
use std::net::SocketAddr;

async fn show_addr(ConnectInfo(addr): ConnectInfo<SocketAddr>) -> String {
    format!("peer is {addr}")
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/whoami", get(show_addr));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();

    // into_make_service_with_connect_info is REQUIRED for ConnectInfo to work.
    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .await
    .unwrap();
}
```

`ConnectInfo<SocketAddr>` is populated ONLY when the router is served via
`into_make_service_with_connect_info`. A plain `into_make_service` leaves it absent and
the extractor rejects the request.

## Sources verified

All fetched via WebFetch on 2026-05-20:

- https://docs.rs/axum/latest/axum/fn.serve.html
- https://docs.rs/axum/latest/axum/serve/struct.Serve.html
- https://docs.rs/axum/latest/axum/
- https://github.com/tokio-rs/axum/tree/main/examples (graceful-shutdown example)
