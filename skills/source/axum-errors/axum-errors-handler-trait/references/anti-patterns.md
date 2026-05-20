# Anti-Patterns: axum-errors-handler-trait

Real mistakes that produce `the trait bound ... Handler<_, _> is not satisfied`,
each with the exact reason it fails and the corrected approach. All behavior is
identical in Axum 0.7 and Axum 0.8.

## Anti-pattern: guessing the cause from the opaque error

NEVER try to read the cause out of the raw `E0277` block.

```rust
// The compiler reports the whole Handler impl as unsatisfied:
//   error[E0277]: the trait bound `...{handler}: Handler<_, _>`
//                 is not satisfied
async fn handler(flag: bool) {}
```

WHY this fails as a diagnostic approach: the blanket impl is one `impl` with a
conjunction of seven bounds. Rust reports the conjunction, not the failing
conjunct. The `help` block lists unrelated `Handler` impls (`MethodRouter`,
`Layered`); it is not a hint. The five causes are indistinguishable from this
block alone.

CORRECT: ALWAYS annotate the handler with `#[debug_handler]` first. The macro
asserts each criterion in its own item and pins the failure to one span.

## Anti-pattern: using `#[debug_handler]` without the `macros` feature

```rust
use axum::debug_handler;   // does not resolve

#[debug_handler]
async fn handler() {}
```

```toml
# Cargo.toml - the macros feature is missing
axum = { version = "0.8" }
```

WHY this fails: `axum::debug_handler` is "Available on crate feature `macros`
only". Without the feature the import does not resolve and the macro is
unavailable.

CORRECT: enable the feature.

```toml
axum = { version = "0.8", features = ["macros"] }
```

## Anti-pattern: a primitive or domain type as a handler argument

```rust
// FAILS - bool is not an extractor
async fn handler(flag: bool) {}

// FAILS - a plain domain struct is not an extractor
struct Filter { active: bool }
async fn handler2(filter: Filter) {}
```

WHY this fails: every handler argument MUST implement `FromRequestParts<S>` (all
but the last) or `FromRequest<S, M>` (the last). Primitives and bare domain
structs implement neither, so criterion 3 or 4 of the blanket impl fails.

CORRECT: take a real extractor and read the value through it. Use `Query` or
`Path` for request data, and `State` for shared application data.

```rust
use axum::extract::{Query, State};

#[derive(serde::Deserialize)]
struct Filter { active: bool }

async fn handler(Query(filter): Query<Filter>) { let _ = filter.active; }
```

## Anti-pattern: a body extractor before another argument

```rust
// FAILS - Bytes (body) precedes Path (parts)
use axum::body::Bytes;
use axum::extract::Path;

async fn handler(body: Bytes, Path(id): Path<u64>) {}
```

WHY this fails: exactly one argument may consume the request body, and it MUST be
the last argument. Every argument before the last MUST implement
`FromRequestParts`. A body extractor in any earlier position breaks criterion 3.

CORRECT: order parts extractors first, the single body extractor last.

```rust
async fn handler(Path(id): Path<u64>, body: Bytes) { let _ = (id, body); }
```

## Anti-pattern: two body extractors in one handler

```rust
// FAILS - both String and Json consume the body
use axum::Json;

async fn handler(body: String, Json(p): Json<Payload>) {}

#[derive(serde::Deserialize)]
struct Payload { name: String }
```

WHY this fails: the request body is single-use. Only the last argument may be a
`FromRequest` body extractor. A second body extractor cannot satisfy
`FromRequestParts`, so criterion 3 fails.

CORRECT: use exactly one body extractor. Choose `Json` OR `String` OR `Bytes`,
never two.

## Anti-pattern: returning a type that is not `IntoResponse`

```rust
// FAILS - bare i32 is not a response
async fn handler() -> i32 { 42 }

// FAILS - the Result error type does not implement IntoResponse
async fn handler2() -> Result<String, std::io::Error> { Ok(String::new()) }
```

WHY this fails: criterion 5 requires the return type to implement
`IntoResponse`. A bare `i32` does not. For `Result<T, E>`, BOTH `T` and `E` must
implement `IntoResponse`; `std::io::Error` does not, so the `Result` does not
either.

CORRECT: return an `IntoResponse` type, and give any `Result` error type its own
`IntoResponse` impl.

```rust
use axum::Json;
async fn handler() -> Json<i32> { Json(42) }
```

## Anti-pattern: a synchronous handler function

```rust
// FAILS - plain fn
fn handler() -> &'static str { "hello" }

// FAILS - returns a Future but is not declared async fn
use std::future::Future;
fn handler2() -> impl Future<Output = &'static str> { async { "hello" } }
```

WHY this fails: criterion 1 requires `async fn`. A plain `fn` is never a handler,
even one that returns `impl Future`. The blanket impl matches the `async fn`
shape, not an arbitrary function returning a future. With `#[debug_handler]` this
produces the precise error `handlers must be async functions`.

CORRECT: declare the function `async`.

```rust
async fn handler() -> &'static str { "hello" }
```

## Anti-pattern: holding a `!Send` guard across `.await`

```rust
// FAILS - std::sync::MutexGuard held across the await makes the future !Send
use std::sync::{Arc, Mutex};
use axum::extract::State;

async fn handler(State(m): State<Arc<Mutex<i32>>>) {
    let guard = m.lock().unwrap();
    some_async_call().await;        // guard alive here -> future is !Send
    let _ = *guard;
}

async fn some_async_call() {}
```

WHY this fails: criterion 7 requires the handler future to be `Send`. A
`std::sync::MutexGuard` is `!Send`. Holding it (or an `Rc`, or a `RefCell`
borrow) across an `.await` makes the entire future `!Send`, so the blanket impl
no longer applies.

CORRECT: drop the guard before the `.await` by narrowing its scope, or use
`tokio::sync::Mutex`, whose guard is `Send` and may be held across an `.await`.

```rust
use std::sync::Arc;
use tokio::sync::Mutex;
use axum::extract::State;

async fn handler(State(m): State<Arc<Mutex<i32>>>) {
    let guard = m.lock().await;
    some_async_call().await;        // tokio guard is Send
    let _ = *guard;
}

async fn some_async_call() {}
```

## Anti-pattern: `#[debug_handler]` on an impl-block method without `self`

```rust
// FAILS - associated function with no self parameter
struct Api;

impl Api {
    #[axum::debug_handler]
    async fn handler() {}
    // error[E0425]: cannot find function
    //   `__axum_macros_check_handler_0_from_request_check` in this scope
}
```

WHY this fails: `#[debug_handler]` cannot be applied to a method inside an `impl`
block that lacks a `self` parameter. The synthetic check function it generates
cannot be resolved, producing `E0425`. This is a macro limitation, NOT a handler
bug.

CORRECT: move the handler to a free `async fn`, then apply `#[debug_handler]`
there.

```rust
#[axum::debug_handler]
async fn handler() {}
```

## Anti-pattern: blaming the route call

```rust
// The error span points at the route() call, but the route is correct.
Router::new().route("/", get(handler));
```

WHY this misleads: the `route` / `get` / `post` call is where the `Handler`
bound is required, so the error surfaces there. The fault is never the routing
call itself; it is always the shape of `handler`.

CORRECT: leave the route call alone. Annotate `handler` with `#[debug_handler]`
and fix the cause the macro reveals.

## Anti-pattern: removing `#[debug_handler]` to "save build time"

```rust
// Removing the attribute on the belief it slows release builds.
async fn handler() {}
```

WHY this is unnecessary: `#[debug_handler]` "has no effect when compiled with the
release profile". It adds zero cost to `cargo build --release`.

CORRECT: keep `#[debug_handler]` on handlers under active development if it
helps. Removal is a style choice, not a performance requirement. Optionally gate
it with `#[cfg_attr(debug_assertions, axum::debug_handler)]`.
