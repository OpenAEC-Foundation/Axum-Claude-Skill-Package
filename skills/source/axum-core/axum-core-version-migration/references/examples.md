# axum-core-version-migration : Examples

Before-and-after migration examples for the Axum 0.7 to 0.8 upgrade. Every code
block is annotated `// axum 0.7` or `// axum 0.8`. APIs verified against the
official CHANGELOG and the 0.8.0 announcement blog on 2026-05-20 (see
`methods.md` for source URLs).

## Example: Cargo.toml dependency bump

```toml
# axum 0.7
[dependencies]
axum = { version = "0.7", features = ["ws", "macros", "headers"] }
axum-extra = "0.9"
async-trait = "0.1"
```

```toml
# axum 0.8 - "headers" feature dropped, TypedHeader moves to axum-extra,
# async-trait no longer needed for extractors
[dependencies]
axum = { version = "0.8", features = ["ws", "macros"] }
axum-extra = { version = "0.10", features = ["typed-header"] }
```

Set the toolchain to Rust 1.75 or newer, the 0.8 minimum supported version:

```toml
# rust-toolchain.toml
[toolchain]
channel = "1.75"
```

## Example: route path conversion

```rust
// axum 0.7 - colon and asterisk capture syntax
use axum::{Router, routing::get};

let app = Router::new()
    .route("/users/:id", get(show_user))
    .route("/users/:id/posts/:post_id", get(show_post))
    .route("/static/*path", get(serve_static))
    .nest("/api/:version", versioned_routes());
```

```rust
// axum 0.8 - curly-brace capture syntax
use axum::{Router, routing::get};

let app = Router::new()
    .route("/users/{id}", get(show_user))
    .route("/users/{id}/posts/{post_id}", get(show_post))
    .route("/static/{*path}", get(serve_static))
    .nest("/api/{version}", versioned_routes());
```

A literal brace in a path is escaped by doubling:

```rust
// axum 0.8 - matching a literal "{" then "}" segment
let app = Router::new().route("/template/{{id}}", get(handler));
// this route matches the path "/template/{id}" literally
```

## Example: a literal colon path with without_v07_checks

```rust
// axum 0.8 - the route genuinely needs a literal ":" segment
use axum::{Router, routing::get};

let app = Router::new()
    .without_v07_checks()                       // disable the 0.7-syntax panic
    .route("/ratio/:value", get(ratio_handler)); // ":value" is literal here
```

Use `without_v07_checks()` ONLY for a genuinely literal `:` or `*` segment. It is
not a migration shortcut: with it enabled, `:value` is a literal path segment,
not a captured parameter.

## Example: custom extractor, async_trait removal

```rust
// axum 0.7 - the #[async_trait] macro is required
use axum::{async_trait, extract::FromRequestParts, http::request::Parts};

#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for AuthToken {
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S)
        -> Result<Self, Self::Rejection>
    {
        parts.headers
            .get("authorization")
            .and_then(|v| v.to_str().ok())
            .map(|s| AuthToken(s.to_owned()))
            .ok_or((StatusCode::UNAUTHORIZED, "missing authorization header"))
    }
}
```

```rust
// axum 0.8 - identical body, NO #[async_trait], no async-trait import
use axum::{extract::FromRequestParts, http::request::Parts};

impl<S: Send + Sync> FromRequestParts<S> for AuthToken {
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S)
        -> Result<Self, Self::Rejection>
    {
        parts.headers
            .get("authorization")
            .and_then(|v| v.to_str().ok())
            .map(|s| AuthToken(s.to_owned()))
            .ok_or((StatusCode::UNAUTHORIZED, "missing authorization header"))
    }
}
```

The only change is deleting the `#[async_trait]` attribute and its import. The
`async fn` body is unchanged.

## Example: WebSocket Message construction

```rust
// axum 0.7 - Message::Text holds String, Binary holds Vec<u8>
use axum::extract::ws::Message;

socket.send(Message::Text("hello".to_string())).await?;
socket.send(Message::Binary(vec![0x01, 0x02])).await?;

while let Some(Ok(msg)) = socket.recv().await {
    match msg {
        Message::Text(text) => { /* text: String */ }
        Message::Binary(buf) => { /* buf: Vec<u8> */ }
        _ => {}
    }
}
socket.close().await?;   // removed in 0.8
```

```rust
// axum 0.8 - Message::Text holds Utf8Bytes, Binary holds Bytes
use axum::extract::ws::Message;

socket.send(Message::Text("hello".into())).await?;
socket.send(Message::Binary(vec![0x01, 0x02].into())).await?;

while let Some(Ok(msg)) = socket.recv().await {
    match msg {
        Message::Text(text) => { /* text: Utf8Bytes; &text is &str */ }
        Message::Binary(buf) => { /* buf: Bytes; &buf is &[u8] */ }
        _ => {}
    }
}
socket.send(Message::Close(None)).await?;   // replaces socket.close()
```

## Example: Option extractor migration

```rust
// axum 0.7 - a malformed path id silently became None
use axum::extract::Path;

async fn handler(maybe_id: Option<Path<u64>>) -> String {
    match maybe_id {
        Some(Path(id)) => format!("id is {id}"),
        None => "no valid id".to_string(),  // also reached on a MALFORMED id
    }
}
```

```rust
// axum 0.8 - use Result to keep the "inspect the failure" behavior
use axum::extract::{Path, rejection::PathRejection};

async fn handler(id: Result<Path<u64>, PathRejection>) -> String {
    match id {
        Ok(Path(id)) => format!("id is {id}"),
        Err(_rejection) => "id missing or malformed".to_string(),
    }
}
```

On 0.8, `Option<Path<u64>>` rejects a malformed id with `400` before the handler
runs, so the `None` branch is no longer reached for a malformed value. This is a
SILENT change: the 0.7 code still compiles on 0.8 but behaves differently.

## Example: tuple Path arity

```rust
// axum 0.8 - the tuple length MUST equal the number of {captures}

// route: "/users/{uid}/teams/{tid}"  has 2 captures
async fn ok(Path((uid, tid)): Path<(u64, u64)>) { /* compiles and matches */ }

// route: "/users/{uid}"  has 1 capture
async fn bad(Path((uid, tid)): Path<(u64, u64)>) { /* 0.8 rejects: arity 2 != 1 */ }
```

On 0.7 a tuple-length mismatch was lenient. On 0.8 it rejects the request. Audit
every `Path<(A, B, ...)>` against its route's capture count.

## Example: replacing axum::Server

```rust
// axum 0.7 - the old hyper-backed server type
#[tokio::main]
async fn main() {
    let app = build_app();
    axum::Server::bind(&"0.0.0.0:3000".parse().unwrap())
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

```rust
// axum 0.8 - serve over an explicit tokio TcpListener
#[tokio::main]
async fn main() {
    let app = build_app();
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## Example: moving TypedHeader and Host to axum-extra

```rust
// axum 0.7 - TypedHeader and Host live in the axum crate
use axum::extract::Host;
use axum::TypedHeader;
use axum::headers::UserAgent;

async fn handler(
    Host(host): Host,
    TypedHeader(agent): TypedHeader<UserAgent>,
) { /* ... */ }
```

```rust
// axum 0.8 - both moved to axum-extra
use axum_extra::extract::Host;
use axum_extra::TypedHeader;
use axum_extra::headers::UserAgent;

async fn handler(
    Host(host): Host,
    TypedHeader(agent): TypedHeader<UserAgent>,
) { /* ... */ }
```

The extractor usage is identical; only the import paths and the crate that
provides them change. Add `axum-extra` with the `typed-header` feature.

## Example: replacing the RawBody extractor

```rust
// axum 0.7 - RawBody gave direct access to the body
async fn handler(RawBody(body): RawBody) { /* body: the request body */ }
```

```rust
// axum 0.8 - RawBody is removed; take Bytes, String, or the whole Request
use axum::body::Bytes;

async fn handler(body: Bytes) { /* the full body as Bytes */ }

// or, when streaming access is needed
use axum::extract::Request;

async fn streaming_handler(req: Request) {
    let body = req.into_body();
    // consume body via http_body_util or Body::into_data_stream()
}
```
