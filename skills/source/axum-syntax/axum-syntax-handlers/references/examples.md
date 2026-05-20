# axum-syntax-handlers: examples

Working, version-annotated handler code. Every example is verified against
`docs.rs/axum/latest/axum/handler/` (fetched 2026-05-20). The handler model
is identical on Axum 0.7 and 0.8; examples are marked `// axum 0.7 / 0.8`
unless an example uses a version-specific API.

## Minimal handlers

The smallest valid handlers take no arguments. The return type alone decides
the response.

```rust
// axum 0.7 / 0.8 - zero-argument handlers, every one valid
use axum::{Json, http::StatusCode, response::Html};

// () -> empty 200 OK
async fn unit() {}

// &'static str -> text/plain; charset=utf-8 body
async fn text() -> &'static str { "hello" }

// String -> text/plain body
async fn owned_text() -> String { format!("now serving") }

// StatusCode -> that status, empty body
async fn no_content() -> StatusCode { StatusCode::NO_CONTENT }

// Html<T> -> text/html body
async fn page() -> Html<&'static str> { Html("<h1>home</h1>") }

// Json<T> -> application/json body
async fn config() -> Json<serde_json::Value> {
    Json(serde_json::json!({ "version": 1 }))
}
```

## A handler with state, a parts extractor, and a body extractor

The canonical shape: every `FromRequestParts` extractor first, the single
`FromRequest` body extractor last.

```rust
// axum 0.7 / 0.8 - full handler signature
use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
};
use serde::{Deserialize, Serialize};

#[derive(Clone)]
struct AppState { /* db pool, config, ... */ }

#[derive(Deserialize)]
struct NewUser { name: String }

#[derive(Serialize)]
struct User { id: u64, name: String }

async fn create_user(
    State(_state): State<AppState>,  // FromRequestParts
    Path(team_id): Path<u64>,        // FromRequestParts
    Json(payload): Json<NewUser>,    // FromRequest - LAST argument
) -> (StatusCode, Json<User>) {
    let user = User { id: team_id, name: payload.name };
    (StatusCode::CREATED, Json(user))
}
```

## Result handlers and the ? operator

A handler returning `Result<T, E>` produces a valid HTTP response in both the
`Ok` and `Err` arm, because both `T` and `E` implement `IntoResponse`. Axum
services never "fail": their error type is `Infallible`.

```rust
// axum 0.7 / 0.8 - Result with a built-in status as the error type
use axum::{body::Bytes, http::StatusCode};

async fn echo(body: Bytes) -> Result<String, StatusCode> {
    String::from_utf8(body.to_vec()).map_err(|_| StatusCode::BAD_REQUEST)
}
```

```rust
// axum 0.7 / 0.8 - Result with a custom error type; `?` converts via From
use axum::{extract::Json, response::{IntoResponse, Response}, http::StatusCode};
use serde::Deserialize;

#[derive(Deserialize)]
struct Payload { value: i64 }

enum AppError { OutOfRange }

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        match self {
            AppError::OutOfRange => {
                (StatusCode::UNPROCESSABLE_ENTITY, "value out of range").into_response()
            }
        }
    }
}

fn process(p: Payload) -> Result<String, AppError> {
    if p.value > 1000 { return Err(AppError::OutOfRange); }
    Ok(format!("accepted {}", p.value))
}

async fn fallible(Json(p): Json<Payload>) -> Result<String, AppError> {
    let result = process(p)?;   // `?` short-circuits with Err(AppError)
    Ok(result)
}
```

The `AppError` design (variants, `From` impls, status mapping) is covered in
depth by `axum-errors-handling`. This example shows only the handler-side
shape: a `Result` return type whose `E` is `IntoResponse`.

## Registering handlers on a Router

Handlers become routes through the method-router constructors. The handler
itself is unchanged; the version difference is route path syntax only.

```rust
// axum 0.8 - curly-brace path syntax
use axum::{Router, routing::{get, post}};

let app: Router = Router::new()
    .route("/", get(text))
    .route("/users/{team_id}", post(create_user)); // 0.8: {team_id}
```

```rust
// axum 0.7 - colon path syntax, same handlers
use axum::{Router, routing::{get, post}};

let app: Router = Router::new()
    .route("/", get(text))
    .route("/users/:team_id", post(create_user)); // 0.7: :team_id
```

## Async closures as handlers

A closure is a handler when it meets the six criteria plus `Clone + Send +
'static`. The closure body must be `async`.

```rust
// axum 0.7 / 0.8 - async closures as handlers
use axum::{Router, routing::{get, post}};

let app: Router = Router::new()
    .route("/", get(|| async { "Hello, World!" }))
    .route("/echo", post(|body: String| async move { body }))
    .route("/whoami", get(|method: axum::http::Method| async move {
        format!("method was {method}")
    }));
```

```rust
// axum 0.7 / 0.8 - a closure capturing a Send + 'static value is valid
use std::sync::Arc;
use axum::{Router, routing::get};

let greeting = Arc::new(String::from("hi"));
let app: Router = Router::new().route("/", get({
    let greeting = Arc::clone(&greeting);
    move || async move { (*greeting).clone() }
}));
```

A closure that captures a non-`Send` value (an `Rc`, a `RefCell` borrow) or a
non-`'static` borrow fails the `Handler` bound exactly as a named function
would.

## Serving one handler without a Router

`HandlerWithoutStateExt::into_make_service` serves a single stateless handler
as the entire service. It answers every request on every path and method.

```rust
// axum 0.8 - one handler, no Router
use axum::handler::HandlerWithoutStateExt;

async fn handler() -> &'static str { "Hello from a bare handler" }

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, handler.into_make_service())
        .await
        .unwrap();
}
```

```rust
// axum 0.8 - bare handler that needs the peer address
use axum::{extract::ConnectInfo, handler::HandlerWithoutStateExt};
use std::net::SocketAddr;

async fn handler(ConnectInfo(addr): ConnectInfo<SocketAddr>) -> String {
    format!("you are {addr}")
}

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    let service = handler.into_make_service_with_connect_info::<SocketAddr>();
    axum::serve(listener, service).await.unwrap();
}
```

For Axum 0.7 the same code applies; `axum::serve` and
`HandlerWithoutStateExt` exist in both versions. (Axum 0.7 also still exposed
`axum::Server`, removed in 0.8; that boot path is covered by
`axum-core-version-migration`.)

## Working around the 16-extractor limit

A handler with 17 or more arguments has no blanket impl. Group related
extractors into one struct that implements `FromRequestParts`. The struct
counts as a single argument.

```rust
// axum 0.7 / 0.8 - bundle several pieces of request data into one extractor
use axum::{
    extract::{FromRequestParts, Json},
    http::{request::Parts, HeaderMap, Method, Uri},
};
use std::convert::Infallible;

// One extractor that replaces three separate handler arguments.
struct RequestContext {
    method: Method,
    uri: Uri,
    headers: HeaderMap,
}

// axum 0.8 - native async fn in trait, no #[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for RequestContext {
    type Rejection = Infallible;

    async fn from_request_parts(parts: &mut Parts, _state: &S)
        -> Result<Self, Self::Rejection>
    {
        Ok(RequestContext {
            method: parts.method.clone(),
            uri: parts.uri.clone(),
            headers: parts.headers.clone(),
        })
    }
}

async fn report(ctx: RequestContext, Json(_body): Json<serde_json::Value>) {
    let _ = (ctx.method, ctx.uri, ctx.headers);
}
```

For Axum 0.7 the same extractor needs the `#[async_trait]` attribute on the
`impl` block. The full custom-extractor pattern, including the 0.7-vs-0.8
`#[async_trait]` difference, is covered by `axum-syntax-custom-extractors`.

## Diagnosing a rejected handler

When a function will not compile as a handler, apply `#[debug_handler]`.

```rust
// axum 0.7 / 0.8 - debug_handler turns the opaque error into a precise one
#[axum::debug_handler]
async fn broken(value: i32) -> i32 { value }
// Without debug_handler: "the trait bound ... Handler<_, _> is not satisfied".
// With debug_handler: a message naming `value: i32` as a non-extractor
// argument and `-> i32` as a non-IntoResponse return type.
```
