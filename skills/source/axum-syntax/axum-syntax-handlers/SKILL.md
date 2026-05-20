---
name: axum-syntax-handlers
description: >
  Use when writing or fixing an Axum handler function: deciding the argument
  list, the return type, or why a function is rejected as a handler.
  Prevents the "the trait bound Handler is not satisfied" error caused by a
  non-extractor argument, a body extractor that is not last, a return type
  that does not implement IntoResponse, a synchronous function, a non-Send
  future, or more than 16 arguments.
  Covers the Handler<T, S> trait and its blanket impl, the six eligibility
  criteria, valid return types including Result handlers, async closures as
  handlers, the 16-extractor limit, and serving one handler without a Router
  via HandlerWithoutStateExt::into_make_service.
  Keywords: axum handler, Handler trait, Handler<T, S>, blanket impl,
  IntoResponse return type, async fn handler, 16 argument limit, async
  closure handler, into_make_service, HandlerWithoutStateExt, debug_handler,
  Handler is not satisfied, trait bound is not satisfied, handler will not
  compile, function not accepted as handler, my route does not compile,
  how do I write a handler, what return type does a handler need, what is
  a handler in axum.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-syntax-handlers

## Overview

A handler is an `async fn` (or async closure) that accepts zero or more
extractors as arguments and returns a value that can be converted into a
response. Handlers are where request data becomes application logic.

Axum has no routing macros. A function is NOT marked as a handler with an
attribute. Eligibility is enforced entirely by the `Handler<T, S>` trait and
its blanket implementation. A function that meets every criterion IS a
handler; a function that fails any criterion is rejected with one opaque
error: `the trait bound ... Handler<_, _> is not satisfied`.

The handler model is identical across Axum 0.7 and 0.8. This skill is
version-stable; the only version-divergent handler-adjacent detail is route
path syntax, which belongs to `axum-core-router`.

## Quick Reference

### The six handler criteria

A function implements `Handler<T, S>` through the blanket impl when ALL of
these hold. Verified from `docs.rs/axum/latest/axum/handler/`:

| # | Criterion | Failure symptom |
|---|-----------|-----------------|
| 1 | Declared `async fn` | Plain `fn` returning a `Future` is rejected |
| 2 | At most 16 arguments, all `Send` | 17+ arguments has no blanket impl |
| 3 | Every argument except the last implements `FromRequestParts<S>` | A non-extractor or a misplaced body extractor is rejected |
| 4 | The last argument implements `FromRequest<S, M>` | A body extractor not in last position is rejected |
| 5 | The return type implements `IntoResponse` | A bare `i32`, an unwrapped struct, is rejected |
| 6 | The produced future is `Send` | An `Rc` or `MutexGuard` held across `.await` is rejected |

For a closure handler, criterion 7 is added: the closure "must implement
`Clone + Send` and be `'static`".

A handler with zero arguments trivially satisfies criteria 3 and 4.

### The Handler trait shape

```rust
// axum 0.7 / 0.8 - verified from docs.rs/axum/latest/axum/handler/trait.Handler.html
pub trait Handler<T, S>: Clone + Send + Sync + Sized + 'static {
    type Future: Future<Output = Response> + Send + 'static;

    fn call(self, req: Request, state: S) -> Self::Future;

    fn layer<L>(self, layer: L) -> Layered<L, Self, T, S>
    where
        L: Layer<HandlerService<Self, T, S>> + Clone,
        L::Service: Service<Request>;

    fn with_state(self, state: S) -> HandlerService<Self, T, S>;
}
```

`T` is a coherence-rule workaround: it carries the handler's argument types
so the blanket impl does not collide for functions of different arities. A
handler author NEVER names `T`; it is always inferred. `S` is the application
state type.

### Valid return types

ANY type that implements `IntoResponse` is a valid handler return type.

| Return type | Result |
|-------------|--------|
| `()` | empty `200 OK` |
| `&'static str`, `String` | `text/plain; charset=utf-8` body |
| `Vec<u8>`, `Bytes` | `application/octet-stream` body |
| `StatusCode` | response with that status, empty body |
| `(StatusCode, T)` | status plus a body from `T` |
| `Json<T>` | JSON body |
| `Html<T>` | `text/html` body |
| `Redirect` | redirect response |
| `impl IntoResponse` | any of the above, type erased |
| `Result<T, E>` | `Ok` or `Err`, both `IntoResponse` |

The full `IntoResponse` catalog belongs to `axum-syntax-responses`.

### Core rules

- ALWAYS declare a handler `async fn`. A synchronous `fn` is never a handler.
- ALWAYS place the single body extractor (`FromRequest`) LAST.
- ALWAYS make every argument an extractor. A plain `bool`, `i32`, or domain
  struct is not a `FromRequestParts` or `FromRequest` type.
- ALWAYS return a type that implements `IntoResponse`.
- NEVER hold an `Rc`, a `RefCell` borrow, or a `std::sync::MutexGuard` across
  an `.await` inside a handler. The future becomes `!Send` and stops being a
  handler.
- NEVER exceed 16 arguments. Group related extractors into one custom struct.
- When a handler will not compile, ALWAYS apply `#[debug_handler]` first.

## Decision Trees

### Is my function a valid handler

```
Function rejected with "Handler is not satisfied"?
  Is it declared `async fn`?
    NO  -> add `async`. A plain fn returning a Future is not a handler.
    YES -> continue
  Does every argument implement an extractor trait?
    NO  -> a `bool` / `i32` / domain struct argument is the cause.
           Replace it with an extractor, or remove it.
    YES -> continue
  Is at most one argument a body extractor (FromRequest)?
    NO  -> two body extractors. The body is single-use. Keep one.
    YES -> continue
  Is the body extractor the LAST argument?
    NO  -> move it to the end of the argument list.
    YES -> continue
  Does the return type implement IntoResponse?
    NO  -> wrap data in Json(...), return String, or use a StatusCode tuple.
    YES -> continue
  Does the handler hold a !Send value across an .await?
    YES -> the future is !Send. Scope the Rc / guard, or use
           tokio::sync::Mutex. See axum-core-async-performance.
    NO  -> more than 16 arguments. Group extractors into one struct.

Cannot tell which criterion failed? Apply `#[debug_handler]` from
axum-macros for a precise, plain-language diagnostic. Full coverage:
axum-errors-handler-trait.
```

### Which return type for which result

```
What does the handler produce?
  nothing meaningful, only a side effect      -> ()  (empty 200 OK)
  plain text                                  -> &'static str / String
  a serializable value as JSON                -> Json<T>
  an HTML document                            -> Html<T>
  only a status (204, 404, ...)               -> StatusCode
  a status plus a body                        -> (StatusCode, Json<T>) etc.
  a redirect                                  -> Redirect
  success OR a typed failure                  -> Result<T, E>  (E: IntoResponse)
  several different concrete types per branch -> impl IntoResponse
```

## Patterns

### Pattern: the handler signature

Parts extractors first, the optional body extractor last. The return type
implements `IntoResponse`. This shape is identical on Axum 0.7 and 0.8.

```rust
// axum 0.7 / 0.8 - canonical handler: parts extractors, then one body extractor
use axum::{extract::{Path, State, Json}, http::StatusCode};

async fn create_user(
    State(state): State<AppState>,   // FromRequestParts
    Path(team_id): Path<u64>,        // FromRequestParts
    Json(payload): Json<NewUser>,    // FromRequest - MUST be last
) -> Result<Json<User>, AppError> {
    let user = state.repo.insert(team_id, payload).await?;
    Ok(Json(user))
}
```

The smallest valid handlers take no arguments:

```rust
// axum 0.7 / 0.8 - minimal handlers, all valid
async fn unit() {}                              // -> empty 200 OK
async fn text() -> &'static str { "hello" }     // -> text/plain body
async fn created() -> impl IntoResponse {
    (StatusCode::CREATED, Json(serde_json::json!({ "ok": true })))
}
```

### Pattern: Result handlers and the ? operator

A handler returning `Result<T, E>` does NOT fail at the service level. Axum
services have `Infallible` as their error type. `Err(E)` becomes a valid HTTP
response because `E` implements `IntoResponse`. This is what makes the `?`
operator usable inside handlers.

```rust
// axum 0.7 / 0.8 - Result handler. Both T and E implement IntoResponse.
use axum::{extract::Json, http::StatusCode, body::Bytes};

async fn echo(body: Bytes) -> Result<String, StatusCode> {
    String::from_utf8(body.to_vec()).map_err(|_| StatusCode::BAD_REQUEST)
}

// axum 0.7 / 0.8 - `?` converts a source error through a custom AppError.
async fn fallible(Json(p): Json<Payload>) -> Result<String, AppError> {
    let result = process(p)?;   // `?` maps the error via From into AppError
    Ok(result)
}
```

`AppError` must implement `IntoResponse`, and `From` impls convert source
errors into it. The depth of this error-type design belongs to
`axum-errors-handling`, not this syntax skill.

### Pattern: async closures as handlers

A closure is a handler when it meets the six criteria plus the closure bound:
it implements `Clone + Send` and is `'static`. The closure body must be
`async` so it produces a `Send` future.

```rust
// axum 0.7 / 0.8 - closures as handlers
use axum::{Router, routing::{get, post}};

let app: Router = Router::new()
    .route("/", get(|| async { "Hello, World!" }))
    .route("/echo", post(|body: String| async move { body }));
```

A closure that captures a non-`Send` value or a non-`'static` borrow fails
the bound exactly as a named function would.

### Pattern: serving one handler without a Router

`HandlerWithoutStateExt` is an extension trait for handlers whose state type
is `()`. Its `into_make_service()` turns a single stateless handler into a
`MakeService` for `axum::serve`, with no `Router`. The handler then answers
every request, regardless of path or method.

```rust
// axum 0.8 - serving a bare handler, no Router
use axum::handler::HandlerWithoutStateExt;

#[tokio::main]
async fn main() {
    async fn handler() -> &'static str { "Hello from a bare handler" }

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, handler.into_make_service()).await.unwrap();
}
```

Use `into_make_service_with_connect_info::<C>()` instead when the handler
needs the peer address. For anything beyond a single catch-all responder,
use a `Router`.

### Pattern: the 16-extractor limit

The blanket impl is generated for tuples of 0 through 16 arguments. A handler
with 17 or more arguments has no matching impl and fails the `Handler` bound.
ALWAYS group related extractors into one struct that implements
`FromRequestParts` (or derives it with `#[derive(FromRequestParts)]`), which
counts as a single argument.

```rust
// axum 0.7 / 0.8 - one custom extractor replaces many arguments
async fn report(ctx: RequestContext, Json(body): Json<ReportInput>) {}
// RequestContext bundles method, headers, auth, locale into one extractor.
```

Hitting the limit is usually a sign the handler should be decomposed. See
`axum-syntax-custom-extractors` for implementing the bundling extractor.

## Reference Links

- `references/methods.md`: the full `Handler<T, S>` trait definition, the
  blanket-impl bounds, `call` / `layer` / `with_state`, the
  `HandlerWithoutStateExt` methods, and the `Layered` / `HandlerService` types.
- `references/examples.md`: version-annotated working handlers, Result
  handlers, async closures, the bare-handler service, and the custom-struct
  workaround for the 16-extractor limit.
- `references/anti-patterns.md`: the real causes of "Handler is not
  satisfied", each with root-cause analysis and the fix.

Related skills: `axum-syntax-extractors` (the extractor argument types),
`axum-syntax-responses` (the `IntoResponse` return-type catalog),
`axum-errors-handler-trait` (diagnosing the "Handler is not satisfied" error
in depth with `#[debug_handler]`), `axum-core-async-performance` (why a
`!Send` future stops a function being a handler).
