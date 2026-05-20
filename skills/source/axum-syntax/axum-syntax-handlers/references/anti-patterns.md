# axum-syntax-handlers: anti-patterns

Real mistakes that make a function fail the `Handler<T, S>` trait bound, each
with root-cause analysis and the fix. Every one of these surfaces as the same
opaque compiler error:

```text
the trait bound `fn(...) -> ... {handler}: Handler<_, _>` is not satisfied
```

The error never names which criterion failed. That is the defining pain
point of Axum's no-macros design. The remedy for ALL of them is to apply
`#[debug_handler]` from `axum-macros` first, then read the precise message.

These causes are verified against
`docs.rs/axum/latest/axum/handler/` and the project research fragments
(fetched 2026-05-20). They apply identically to Axum 0.7 and 0.8.

## AP-1: An argument is not an extractor

```rust
// WRONG - axum 0.7 / 0.8 - `bool` is not an extractor
async fn handler(flag: bool) {}
```

Why it fails: handler criterion 3 and 4 require every argument to implement
`FromRequestParts<S>` or `FromRequest<S, M>`. A bare `bool`, `i32`, `String`
in a non-last position, or a domain struct implements neither. The function
has no matching blanket impl.

The trap: `String` IS a valid extractor (it reads the body), so
`async fn h(s: String)` compiles, but `async fn h(flag: bool)` does not. The
difference is not visible in the error message.

Fix: every argument must be an extractor. Read the value from the request
through `Query<T>`, `Path<T>`, `State<T>`, `Json<T>`, a header extractor, or
a custom extractor. If the value is not request data at all, it does not
belong in a handler signature; capture it in a closure or read it from state.

## AP-2: A body extractor is not the last argument

```rust
// WRONG - axum 0.7 / 0.8 - body extractor String is before another argument
async fn handler(body: String, headers: HeaderMap) {}

// WRONG - axum 0.7 / 0.8 - body extractor Bytes is before Path
async fn handler(body: Bytes, Path(id): Path<u64>) {}
```

Why it fails: the request body is a single-use async stream. The `Handler`
blanket impl requires every argument except the last to implement
`FromRequestParts` (which never touches the body) and only the last to
implement `FromRequest` (which consumes the body). A body extractor in a
non-final position has no matching impl.

Fix: move the body extractor to the end.

```rust
// CORRECT - axum 0.7 / 0.8 - body extractor last
async fn handler(headers: HeaderMap, body: String) {}
async fn handler(Path(id): Path<u64>, body: Bytes) {}
```

## AP-3: Two body extractors in one handler

```rust
// WRONG - axum 0.7 / 0.8 - String and Json both consume the body
async fn handler(body: String, json: Json<Payload>) {}
```

Why it fails: the same single-use-body rule as AP-2. Only one argument can
implement `FromRequest`; the body cannot be read twice. With two body
extractors, no position satisfies the bound.

Fix: keep one body extractor. To read the body two ways, take it once with
the richer extractor (`Bytes` or `Request`) and parse from there.

## AP-4: The return type does not implement IntoResponse

```rust
// WRONG - axum 0.7 / 0.8 - i32 is not a response
async fn handler() -> i32 { 42 }

// WRONG - axum 0.7 / 0.8 - a bare domain struct is not a response
async fn handler() -> User { User::default() }
```

Why it fails: handler criterion 5 requires the return type to implement
`IntoResponse`. A bare integer, or a struct that does not derive or implement
`IntoResponse`, does not. The error blames the `Handler` bound, never the
return type directly.

Fix: return a type that implements `IntoResponse`.

```rust
// CORRECT - axum 0.7 / 0.8
async fn handler() -> Json<i32> { Json(42) }
async fn handler() -> Json<User> { Json(User::default()) }
async fn handler() -> String { "42".to_string() }
```

## AP-5: The function is not async

```rust
// WRONG - axum 0.7 / 0.8 - a synchronous fn is never a handler
fn handler() -> &'static str { "hello" }

// WRONG - axum 0.7 / 0.8 - a sync fn returning a Future is still not a handler
fn handler() -> impl std::future::Future<Output = &'static str> {
    async { "hello" }
}
```

Why it fails: handler criterion 1 requires `async fn`. The blanket impl is
written for `async fn`, not for any function that happens to return a
`Future`. The second form looks equivalent but does not match the impl.

Fix: declare the function `async fn`.

```rust
// CORRECT - axum 0.7 / 0.8
async fn handler() -> &'static str { "hello" }
```

## AP-6: The handler future is !Send

```rust
// WRONG - axum 0.7 / 0.8 - std::sync::MutexGuard held across .await
use std::sync::Mutex;

async fn handler(State(state): State<Arc<Mutex<Counter>>>) {
    let mut guard = state.lock().unwrap(); // std MutexGuard is !Send
    guard.value += other_service().await;  // guard alive across .await -> !Send future
}
```

Why it fails: handler criterion 6 requires the produced future to be `Send`.
A future that keeps a `!Send` value alive across an `.await` point is itself
`!Send`. `std::sync::MutexGuard`, `Rc`, and a `RefCell` borrow are all
`!Send`. Tokio's multi-threaded runtime and the `Handler` bound both require
`Send` futures, so the function stops being a handler.

Fix: do not hold a `!Send` value across an `.await`. Either scope the guard
so it drops before the await, or use `tokio::sync::Mutex` whose guard IS
`Send`. See `axum-core-async-performance`.

```rust
// CORRECT - axum 0.7 / 0.8 - scope the std guard so it drops before .await
async fn handler(State(state): State<Arc<Mutex<Counter>>>) {
    let delta = other_service().await;
    let mut guard = state.lock().unwrap();
    guard.value += delta;            // guard never crosses an .await
}
```

## AP-7: More than 16 arguments

```rust
// WRONG - axum 0.7 / 0.8 - 17 arguments has no blanket impl
async fn handler(a: Method, b: Uri, /* ... 15 more extractors ... */) {}
```

Why it fails: the blanket impl is generated only for tuples of 0 through 16
argument types. A function with 17 or more arguments has no matching impl,
even when every argument is a valid extractor.

Fix: group related extractors into one struct that implements
`FromRequestParts`, reducing the argument count. A handler approaching the
limit is usually a sign it should be decomposed. See
`axum-syntax-custom-extractors`.

## AP-8: Ignoring #[debug_handler] when a handler will not compile

The mistake: staring at `the trait bound ... Handler<_, _> is not satisfied`
and guessing which criterion failed, instead of letting the compiler say so.

Why it costs time: the bare error is identical for all seven causes above. It
names neither the offending argument nor the return type. Guessing is slow
and unreliable.

Fix: ALWAYS apply `#[debug_handler]` from `axum-macros` (re-exported as
`axum::debug_handler`) as the FIRST diagnostic step. It generates code that
produces a precise, plain-language error pointing at the exact argument or
return type. Remove it once the handler compiles; it is a development aid.
The full diagnostic workflow is owned by `axum-errors-handler-trait`.

```rust
// axum 0.7 / 0.8 - the first move when a handler will not compile
#[axum::debug_handler]
async fn suspect(/* ... */) { /* ... */ }
```

## Summary table

| Anti-pattern | Failed criterion | Fix |
|--------------|------------------|-----|
| AP-1 non-extractor argument | 3 / 4 | make every argument an extractor |
| AP-2 body extractor not last | 3 / 4 | move the body extractor to the end |
| AP-3 two body extractors | 3 / 4 | keep one body extractor |
| AP-4 non-IntoResponse return | 5 | wrap in `Json`, return `String`, etc. |
| AP-5 not async | 1 | declare `async fn` |
| AP-6 !Send future | 6 | no `!Send` value across `.await` |
| AP-7 17+ arguments | 2 | bundle extractors into one struct |
| AP-8 not using debug_handler | (diagnosis) | apply `#[debug_handler]` first |
