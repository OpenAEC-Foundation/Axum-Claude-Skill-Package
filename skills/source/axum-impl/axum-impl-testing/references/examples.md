# Axum Testing : Working Examples

Every example below is verified against the official Axum `examples/testing`
crate and `axum-test` 20.0.0 (verified 2026-05-20). The `oneshot` examples work
identically on Axum 0.7 and 0.8. `axum-test` 20.0.0 tracks Axum 0.8; for Axum
0.7 pin an older `axum-test` release.

## Example 1: Complete oneshot unit-test module

A single `src/main.rs` (or `src/lib.rs`) with the app, its handlers, and a
`#[cfg(test)]` test module. `use super::*;` brings `app()` into the module.

```rust
use axum::{
    routing::{get, post},
    Json, Router,
};
use tower_http::trace::TraceLayer;

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

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app()).await.unwrap();
}

#[cfg(test)]
mod tests {
    use super::*;
    use axum::{
        body::Body,
        http::{self, Request, StatusCode},
    };
    use http_body_util::BodyExt;       // for .collect()
    use tower::{Service, ServiceExt}; // for .oneshot() / .ready()

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

    #[tokio::test]
    async fn json() {
        let app = app();

        let response = app
            .oneshot(
                Request::builder()
                    .method(http::Method::POST)
                    .uri("/json")
                    .header(
                        http::header::CONTENT_TYPE,
                        mime::APPLICATION_JSON.as_ref(),
                    )
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
}
```

## Example 2: TestServer integration test

`axum-test`'s `TestServer` wraps the `Router` and gives fluent assertions.
`.json()` sets the `Content-Type` header automatically.

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

#[tokio::test]
async fn get_user_chained_assertions() {
    let server = TestServer::new(app()).unwrap();

    let response = server.get("/users/1").await;

    response
        .assert_status_ok()
        .assert_json(&json!({ "id": 1, "username": "Terrance Pencilworth" }));
}

#[tokio::test]
async fn extract_typed_body() {
    let server = TestServer::new(app()).unwrap();

    // json::<T>() deserializes the body into a typed value.
    let user: User = server.get("/users/1").await.json::<User>();
    assert_eq!(user.id, 1);
}

#[tokio::test]
async fn missing_route_is_404() {
    let server = TestServer::new(app()).unwrap();

    server.get("/does-not-exist").await.assert_status_not_found();
}
```

## Example 3: State-carrying app(pool) constructor

When the app needs runtime state, give the constructor a parameter so each test
can pass a test value. See the `axum-impl-database` skill for building a test
`PgPool`.

```rust
use axum::{routing::get, Router};
use sqlx::PgPool;

// Library code: pub so tests/ integration tests can reach it.
pub fn app(pool: PgPool) -> Router {
    Router::new()
        .route("/users/{id}", get(get_user))
        .with_state(pool)
}

#[cfg(test)]
mod tests {
    use super::*;
    use axum_test::TestServer;

    async fn test_pool() -> PgPool {
        PgPool::connect(&std::env::var("TEST_DATABASE_URL").unwrap())
            .await
            .unwrap()
    }

    #[tokio::test]
    async fn get_user_uses_test_pool() {
        let pool = test_pool().await;
        let server = TestServer::new(app(pool)).unwrap();

        server.get("/users/1").await.assert_status_ok();
    }
}
```

## Example 4: tests/ directory integration test

`src/lib.rs` exposes `pub fn app()`. A file under `tests/` is its own crate and
imports the app by the crate name (here `my_axum_app`).

`src/lib.rs`:

```rust
use axum::{routing::get, Router};

pub fn app() -> Router {
    Router::new().route("/", get(|| async { "Hello, World!" }))
}
```

`src/main.rs`:

```rust
#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, my_axum_app::app()).await.unwrap();
}
```

`tests/api.rs`:

```rust
use axum::body::Body;
use axum::http::{Request, StatusCode};
use http_body_util::BodyExt;
use my_axum_app::app;            // the public app() from src/lib.rs
use tower::ServiceExt;

#[tokio::test]
async fn root_returns_hello() {
    let response = app()
        .oneshot(Request::builder().uri("/").body(Body::empty()).unwrap())
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::OK);
    let body = response.into_body().collect().await.unwrap().to_bytes();
    assert_eq!(&body[..], b"Hello, World!");
}
```

## Example 5: Real-socket transport for WebSocket tests

WebSocket testing needs a real network socket. Enable it through the builder:
`TestServer::builder()` returns a `TestServerBuilder`, `.http_transport()`
selects a real socket, and `.build(app)` produces the `TestServer`.

```rust
use axum_test::TestServer;

#[tokio::test]
async fn websocket_route() {
    let server = TestServer::builder()
        .http_transport()      // real socket instead of the default mock transport
        .build(app());

    // server.server_address() now returns Some(Url) for a real client.
    let _addr = server.server_address();
    // Drive the WebSocket handshake against that address.
}
```

`build(app)` panics if construction fails; `try_build(app)` returns a `Result`.
In mock HTTP mode (the default) `server_address()` returns `None`; with
`.http_transport()` it returns `Some(Url)`.

## Example 6: MockConnectInfo for ConnectInfo handlers

A handler that extracts `ConnectInfo<SocketAddr>` has no real peer address in a
`oneshot` test. The official example layers `MockConnectInfo` to supply one.

```rust
use axum::extract::connect_info::MockConnectInfo;
use std::net::SocketAddr;

#[tokio::test]
async fn with_into_make_service_with_connect_info() {
    let mut app = app()
        .layer(MockConnectInfo(SocketAddr::from(([0, 0, 0, 0], 3000))))
        .into_service();

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

## Cargo.toml dev-dependencies

```toml
[dev-dependencies]
tower          = { version = "0.5", features = ["util"] } # ServiceExt::oneshot
http-body-util = "0.1"                                    # BodyExt::collect
mime           = "0.3"                                    # JSON oneshot tests
serde_json     = "1"
tokio          = { version = "1", features = ["full"] }   # #[tokio::test]
axum-test      = "20"  // axum 0.8 ; for axum 0.7 pin an older axum-test
```

`tower` and `http-body-util` are needed only for the `oneshot` approach;
`axum-test` only for the `TestServer` approach. A project using both lists all
of them.
