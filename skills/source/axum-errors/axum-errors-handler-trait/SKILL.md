---
name: axum-errors-handler-trait
description: >
  Use when the compiler rejects an Axum handler with
  "the trait bound ... Handler<_, _> is not satisfied", when a function will
  not register on a route, when the error block names Handler with inference
  placeholders and lists unrelated impls as help, or when a handler argument or
  return type silently breaks compilation.
  Prevents passing a non-extractor argument, placing a body extractor before a
  parts extractor, returning a type that is not IntoResponse, writing a plain
  synchronous fn, and producing a !Send future by holding a guard across await.
  Covers the opaque E0277 trait-bound error, the five root causes, the
  axum-macros #[debug_handler] diagnostic macro and its macros feature, the
  state= override, and a symptom-to-cause-to-fix decision tree.
  Keywords: Handler is not satisfied, trait bound is not satisfied, E0277,
  debug_handler, axum-macros, macros feature, FromRequestParts, FromRequest,
  IntoResponse, blanket impl, !Send future, MutexGuard across await,
  handlers must be async functions, cannot find __axum_macros_check, handler
  does not compile, function will not route, why does my handler not work,
  how do I fix the handler error, what does Handler not satisfied mean.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Axum Handler Trait Errors

## Overview

Axum has no handler macro. A function becomes a handler purely by satisfying the
`Handler<T, S>` trait, which has a single blanket implementation guarded by a long
conjunction of bounds. When a function fails ANY one bound, the compiler reports
the WHOLE `impl` as unsatisfied. It cannot point at the failing conjunct, so it
emits the opaque `error[E0277]: the trait bound ... Handler<_, _> is not
satisfied`, names the trait with inference placeholders, and lists unrelated
`Handler` impls as "help".

Every such error reduces to exactly one of five root causes. The diagnostic tool
is the `#[debug_handler]` attribute macro from `axum-macros`: it rewrites the
opaque error into a precise, plain-language one that points at the exact argument
or return type at fault.

The `Handler` trait, its blanket-impl criteria, and `#[debug_handler]` behave
identically in Axum 0.7 and 0.8. The diagnostic workflow in this skill applies
unchanged to both versions.

Three rules dominate correct diagnosis:

1. ALWAYS annotate the failing handler with `#[debug_handler]` before guessing.
   The macro converts the opaque error into a precise span.
2. ALWAYS read the precise error as one of the five causes, then apply the
   matching fix from the decision tree below.
3. NEVER treat `Handler is not satisfied` as a routing bug. The route call only
   surfaces the error. The fault is always in the handler's shape.

## Quick Reference

The five root causes of `Handler is not satisfied`:

| Cause | Symptom after `#[debug_handler]` | Fix |
|-------|----------------------------------|-----|
| 1: Non-extractor argument | Argument "does not implement `FromRequestParts` / `FromRequest`" | Replace with a real extractor (`Query`, `Path`, `State`, `Json`, ...) |
| 2: Body extractor not last | A body extractor sits before another argument, or two body extractors | Move the single body extractor to the last position |
| 3: Return type not `IntoResponse` | Return type "does not implement `IntoResponse`" | Return `Json<T>`, `String`, `StatusCode`, a tuple, or `impl IntoResponse` |
| 4: Function not `async` | "handlers must be async functions" | Add `async` to the `fn` |
| 5: Future is `!Send` | A `Send` bound on the future is unsatisfied | Drop `!Send` guards before `.await`, or use `tokio::sync::Mutex` |

The diagnostic macro:

```rust
// axum 0.7 and axum 0.8 - identical
use axum::debug_handler;

#[debug_handler]
async fn create_user(
    State(state): State<AppState>,
    Json(payload): Json<NewUser>,
) -> Result<Json<User>, AppError> {
    let user = state.repo.insert(payload).await?;
    Ok(Json(user))
}
```

The `macros` feature MUST be enabled to use `axum::debug_handler`:

```toml
# Cargo.toml - axum 0.8
axum = { version = "0.8", features = ["macros"] }

# Cargo.toml - axum 0.7
# axum = { version = "0.7", features = ["macros"] }
```

`#[debug_handler]` has NO effect under the release profile, so leaving it on a
handler permanently costs nothing in production builds.

## Decision Trees

### Symptom to cause to fix

```
SYMPTOM: error[E0277] "the trait bound `...{handler}: Handler<_, _>`
         is not satisfied" at a route() / get() / post() call.

STEP 0: Annotate the handler with #[debug_handler], recompile, read the
        precise error. Then branch:

|-- "handlers must be async functions"
|       CAUSE 4: the function is not async
|       FIX: add `async` to the `fn`
|
|-- Macro points at an ARGUMENT: "does not implement
|   FromRequestParts / FromRequest"
|       |-- argument is a primitive / domain struct / config type
|       |       CAUSE 1: non-extractor argument
|       |       FIX: use a real extractor; read the value through it
|       |
|       |-- argument IS an extractor, but a body extractor precedes
|       |   another argument, or there are two body extractors
|       |       CAUSE 2: body extractor not last
|       |       FIX: parts extractors first, one body extractor last
|
|-- Macro points at the RETURN TYPE: "does not implement IntoResponse"
|       CAUSE 3: return type not IntoResponse
|       FIX: return an IntoResponse type, or make the Result error
|            type implement IntoResponse
|
|-- Macro points at a `Send` bound on the future
|       CAUSE 5: the future is !Send
|       FIX: drop !Send guards before `.await`, or swap
|            std::sync::Mutex for tokio::sync::Mutex
|
|-- Macro emits E0425 "cannot find function
|   __axum_macros_check_handler_0_from_request_check"
        NOT a handler bug: #[debug_handler] was placed on an impl-block
        method that has no `self` parameter.
        FIX: extract a free `async fn`, or diagnose without the macro.

EDGE CASE: the handler has 17 or more arguments. No blanket impl exists
        for tuples that large.
        FIX: group extractors into one custom FromRequestParts /
             FromRequest struct (see axum-syntax-handlers).
```

### When to keep or remove `#[debug_handler]`

```
Handler compiles cleanly?
|-- yes: keep #[debug_handler] OR remove it. It is a release no-op,
|        so keeping it is safe and aids the next change.
|-- no:  keep #[debug_handler] until the precise error is resolved.

Handler is an impl-block method WITHOUT `self`?
|-- yes: do NOT use #[debug_handler] (E0425). Extract a free async fn.
|-- no:  #[debug_handler] is safe to apply.
```

## Patterns

### Pattern: the diagnostic workflow

ALWAYS follow this sequence. NEVER skip Step 1.

1. Add `#[debug_handler]` above the rejected handler. Add `features = ["macros"]`
   to the `axum` dependency if it is not already present.
2. Recompile. The macro generates synthetic check functions
   (`__axum_macros_check_*`) that assert each criterion independently, so the
   compiler pins the failure to one span.
3. Map the precise error to one of the five causes via the decision tree.
4. Apply the matching fix.
5. Recompile to confirm. Keep or remove `#[debug_handler]`; it is a release
   no-op either way.

### Cause 1: non-extractor argument

Every handler argument MUST implement `FromRequestParts<S>` (all but the last) or
`FromRequest<S, M>` (the last). Primitives (`bool`, `i32`, bare `String` used as a
plain argument), domain structs, and config types are NOT extractors.

```rust
// FAILS - `bool` is not an extractor
async fn handler(flag: bool) {}

// FIX - take a real extractor; read the value through it
async fn handler(Query(params): Query<Filter>) {}
// or, for shared config, put it in State
async fn handler(State(cfg): State<AppConfig>) {}
```

### Cause 2: body extractor not last

Exactly one argument may consume the request body, and it MUST be the last
argument. All preceding arguments MUST be parts-only (`FromRequestParts`). The
body is single-use, so two body extractors fail for the same reason.

```rust
// FAILS - `Bytes` (body extractor) precedes `Path` (parts extractor)
async fn handler(body: Bytes, Path(id): Path<u64>) {}

// FAILS - two body extractors
async fn handler(body: String, Json(p): Json<Payload>) {}

// FIX - parts extractors first, the single body extractor last
async fn handler(Path(id): Path<u64>, body: Bytes) {}
```

### Cause 3: return type not `IntoResponse`

The return type MUST implement `IntoResponse`. A `Result<T, E>` is a valid return
type ONLY when both `T: IntoResponse` and `E: IntoResponse`.

```rust
// FAILS - bare `i32` is not a response
async fn handler() -> i32 { 42 }

// FAILS - the Result error type does not implement IntoResponse
async fn handler() -> Result<String, std::io::Error> { /* ... */ }

// FIX - return an IntoResponse type
async fn handler() -> Json<i32> { Json(42) }
// or implement IntoResponse for the custom error used in Result
async fn handler() -> Result<String, AppError> { /* ... */ }
```

### Cause 4: function not `async`

A plain `fn` is NEVER a handler, even when it returns `impl Future`. The blanket
impl requires `async fn`.

```rust
// FAILS - synchronous fn
fn handler() -> &'static str { "hello" }

// FAILS - returns a Future but is not declared `async fn`
fn handler() -> impl Future<Output = &'static str> { async { "hello" } }

// FIX - declare the function `async`
async fn handler() -> &'static str { "hello" }
```

### Cause 5: `!Send` future

The handler future MUST be `Send`. Holding a `!Send` value across an `.await`
makes the whole future `!Send`. Common offenders: `Rc`, `RefCell` borrows held
across `.await`, and `std::sync::MutexGuard`.

```rust
// FAILS - std::sync::MutexGuard is held across the await, future is !Send
async fn handler(State(m): State<Arc<std::sync::Mutex<i32>>>) {
    let guard = m.lock().unwrap();
    do_async_work().await;          // guard still alive here -> !Send
    let _ = *guard;
}

// FIX A - narrow the lock scope so the guard drops before the await
async fn handler(State(m): State<Arc<std::sync::Mutex<i32>>>) {
    { let _v = *m.lock().unwrap(); }   // guard dropped at block end
    do_async_work().await;
}

// FIX B - use tokio::sync::Mutex, whose guard is Send across an await
async fn handler(State(m): State<Arc<tokio::sync::Mutex<i32>>>) {
    let guard = m.lock().await;
    do_async_work().await;             // tokio guard is Send
    let _ = *guard;
}
```

### Pattern: the `#[debug_handler]` state type

`#[debug_handler]` assumes a state type of `()` by default. If the handler has a
`State<T>` argument, the macro infers `T` automatically. For a handler with
multiple or ambiguous state arguments, set it explicitly:

```rust
// axum 0.7 and axum 0.8 - identical
#[debug_handler(state = AppState)]
async fn handler(State(state): State<AppState>) {}
```

### Edge case: 17 or more arguments

There is no `Handler` blanket impl for tuples of 17 or more arguments. The fix is
to group several extractors into one custom `FromRequestParts` or `FromRequest`
struct. This is owned by `axum-syntax-handlers`; consult that skill for the
struct-extractor pattern.

## Common Mistakes

- NEVER guess the cause from the opaque `E0277` block. ALWAYS run
  `#[debug_handler]` first; the five causes are indistinguishable without it.
- NEVER add `#[debug_handler]` without enabling the `macros` feature on `axum`.
  Without it, `axum::debug_handler` does not resolve.
- NEVER place `#[debug_handler]` on an impl-block method that lacks `self`. It
  emits `E0425 cannot find __axum_macros_check_handler_0_from_request_check`,
  which is a macro limitation, not a handler bug.
- NEVER assume the route call is broken. `Handler is not satisfied` always means
  the handler's shape is wrong, not the routing.

## Reference Links

- `references/methods.md`: the `Handler<T, S>` trait, the blanket-impl criteria,
  the `#[debug_handler]` macro, the `macros` feature, and synthetic check items.
- `references/examples.md`: working, version-annotated reproducers for all five
  causes and the full diagnostic workflow.
- `references/anti-patterns.md`: real mistakes that produce `Handler is not
  satisfied`, with the exact reason each one fails.

Related skills:

- `axum-syntax-handlers`: the handler shape, the blanket-impl criteria in depth,
  and the 17-plus-argument struct-extractor pattern.
- `axum-syntax-extractors`: which types implement `FromRequestParts` versus
  `FromRequest`, and extractor ordering.
- `axum-core-async-performance`: the `Send` requirement and avoiding `!Send`
  values across `.await`.
