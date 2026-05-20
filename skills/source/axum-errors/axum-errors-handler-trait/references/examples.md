# Examples: axum-errors-handler-trait

Working, version-annotated reproducers for the opaque `Handler is not satisfied`
error, all five root causes, and the full `#[debug_handler]` diagnostic workflow.
Every snippet is verified against the sources listed in `methods.md`.

The `Handler` blanket-impl criteria and `#[debug_handler]` behavior are identical
in Axum 0.7 and Axum 0.8. Snippets are marked `// axum 0.7 and axum 0.8` where
they apply unchanged to both, and split with `// axum 0.8` / `// axum 0.7`
markers only where the surrounding code diverges.

## Setup: enabling the macros feature

`axum::debug_handler` requires the `macros` feature on the `axum` dependency.

```toml
# Cargo.toml - axum 0.8
[dependencies]
axum = { version = "0.8", features = ["macros"] }
tokio = { version = "1", features = ["full"] }

# Cargo.toml - axum 0.7
# axum = { version = "0.7", features = ["macros"] }
```

## The opaque error and the diagnostic fix

A primitive argument breaks the handler. Without `#[debug_handler]` the compiler
reports the whole `Handler` impl:

```rust
// axum 0.7 and axum 0.8
use axum::{routing::get, Router};

async fn handler(flag: bool) {}   // bool is not an extractor

fn app() -> Router {
    Router::new().route("/", get(handler))
    // error[E0277]: the trait bound `fn(bool) -> ... {handler}:
    //               Handler<_, _>` is not satisfied
}
```

Annotate the handler and recompile. The macro turns the opaque error into a
precise one pointing at the `flag` argument:

```rust
// axum 0.7 and axum 0.8
use axum::debug_handler;

#[debug_handler]
async fn handler(flag: bool) {}
// error now names `flag: bool` and states it does not implement
// FromRequestParts / FromRequest.
```

## Cause 1: non-extractor argument

```rust
// axum 0.7 and axum 0.8 - FAILS
async fn handler(flag: bool) {}

// FAILS - a domain struct is not an extractor either
struct AppConfig { limit: u32 }
async fn handler2(cfg: AppConfig) {}
```

```rust
// axum 0.7 and axum 0.8 - FIX: use a real extractor
use axum::extract::{Query, State};
use serde::Deserialize;

#[derive(Deserialize)]
struct Filter { active: bool }

async fn handler(Query(params): Query<Filter>) {
    let _ = params.active;
}

// FIX - shared config belongs in State
#[derive(Clone)]
struct AppConfig { limit: u32 }

async fn handler2(State(cfg): State<AppConfig>) {
    let _ = cfg.limit;
}
```

## Cause 2: body extractor not last

A body extractor (`Json`, `Bytes`, `String`, `Form`, `Multipart`) consumes the
request body and MUST be the last argument. At most one body extractor is
allowed.

```rust
// axum 0.7 and axum 0.8 - FAILS: Bytes (body) precedes Path (parts)
use axum::body::Bytes;
use axum::extract::Path;

async fn handler(body: Bytes, Path(id): Path<u64>) {}

// FAILS - two body extractors
use axum::Json;
async fn handler2(body: String, Json(p): Json<Payload>) {}

#[derive(serde::Deserialize)]
struct Payload { name: String }
```

```rust
// axum 0.7 and axum 0.8 - FIX: parts extractors first, one body extractor last
use axum::body::Bytes;
use axum::extract::Path;

async fn handler(Path(id): Path<u64>, body: Bytes) {
    let _ = (id, body.len());
}
```

Note: `Path` extraction uses route syntax `/{id}` in axum 0.8 and `/:id` in
axum 0.7. The handler-argument ordering rule shown here is the same in both.

## Cause 3: return type not `IntoResponse`

```rust
// axum 0.7 and axum 0.8 - FAILS: bare i32 is not a response
async fn handler() -> i32 { 42 }

// FAILS - the Result error type does not implement IntoResponse
async fn handler2() -> Result<String, std::io::Error> {
    Ok(String::from("ok"))
}
```

```rust
// axum 0.7 and axum 0.8 - FIX: return an IntoResponse type
use axum::Json;

async fn handler() -> Json<i32> { Json(42) }

// FIX - make the Result error type implement IntoResponse
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};

struct AppError(String);

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (StatusCode::INTERNAL_SERVER_ERROR, self.0).into_response()
    }
}

async fn handler2() -> Result<String, AppError> {
    Ok(String::from("ok"))
}
```

## Cause 4: function not `async`

```rust
// axum 0.7 and axum 0.8 - FAILS: synchronous fn
fn handler() -> &'static str { "hello" }

// FAILS - returns a Future but is not declared async fn
use std::future::Future;
fn handler2() -> impl Future<Output = &'static str> {
    async { "hello" }
}
```

```rust
// axum 0.7 and axum 0.8 - FIX: declare the function async
async fn handler() -> &'static str { "hello" }
```

With `#[debug_handler]` the non-async case produces the verbatim error:

```
error: handlers must be async functions
  --> main.rs:xx:1
   |
xx | fn handler() -> &'static str {
   | ^^
```

## Cause 5: `!Send` future

```rust
// axum 0.7 and axum 0.8 - FAILS: std::sync::MutexGuard held across .await
use std::sync::{Arc, Mutex};
use axum::extract::State;

async fn do_async_work() {}

async fn handler(State(m): State<Arc<Mutex<i32>>>) {
    let guard = m.lock().unwrap();
    do_async_work().await;          // guard still alive -> future is !Send
    let _ = *guard;
}
```

```rust
// axum 0.7 and axum 0.8 - FIX A: narrow the lock scope
use std::sync::{Arc, Mutex};
use axum::extract::State;

async fn do_async_work() {}

async fn handler(State(m): State<Arc<Mutex<i32>>>) {
    {
        let _v = *m.lock().unwrap();   // guard dropped at block end
    }
    do_async_work().await;             // no !Send value across the await
}
```

```rust
// axum 0.7 and axum 0.8 - FIX B: tokio::sync::Mutex, guard is Send across await
use std::sync::Arc;
use tokio::sync::Mutex;
use axum::extract::State;

async fn do_async_work() {}

async fn handler(State(m): State<Arc<Mutex<i32>>>) {
    let guard = m.lock().await;
    do_async_work().await;             // tokio guard is Send
    let _ = *guard;
}
```

## The full diagnostic workflow

```rust
// axum 0.7 and axum 0.8
use axum::debug_handler;
use axum::extract::State;
use axum::Json;
use axum::response::{IntoResponse, Response};
use axum::http::StatusCode;

#[derive(Clone)]
struct AppState { /* repo, config, ... */ }

#[derive(serde::Deserialize)]
struct NewUser { name: String }

#[derive(serde::Serialize)]
struct User { id: u64, name: String }

struct AppError(String);

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (StatusCode::INTERNAL_SERVER_ERROR, self.0).into_response()
    }
}

// Step 1: annotate the rejected handler with #[debug_handler].
// Step 2: recompile and read the precise error.
// Step 3: map it to one of the five causes.
// Step 4: apply the fix. Step 5: recompile to confirm.
#[debug_handler]
async fn create_user(
    State(_state): State<AppState>,         // parts extractor, not last
    Json(payload): Json<NewUser>,           // body extractor, last
) -> Result<Json<User>, AppError> {         // both arms are IntoResponse
    Ok(Json(User { id: 1, name: payload.name }))
}
```

## Explicit state for `#[debug_handler]`

```rust
// axum 0.7 and axum 0.8
use axum::debug_handler;
use axum::extract::State;

#[derive(Clone)]
struct AppState;

// The macro infers () by default; give it the real state explicitly
// when the handler has multiple or ambiguous state arguments.
#[debug_handler(state = AppState)]
async fn handler(State(_state): State<AppState>) {}
```

## Edge case: 17 or more arguments

```rust
// axum 0.7 and axum 0.8 - FIX: group extractors into one struct.
// See axum-syntax-handlers for the custom FromRequestParts pattern.
use axum::extract::{Path, Query, State};

struct AppState;

// Instead of 17 separate arguments, accept one aggregate extractor.
// The aggregate struct implements FromRequestParts by deriving or
// hand-writing the impl (covered in axum-syntax-handlers).
async fn handler(State(_s): State<AppState>, /* one aggregate extractor */) {}
```
