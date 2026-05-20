# Topic Research : axum-errors-handler-trait

> Skill : `axum-errors-handler-trait`
> Category : errors/
> Target versions : Axum 0.7 and 0.8 (docs.rs current : 0.8.x)
> Researched : 2026-05-20
> Status : verified against official docs.rs content

This research file scopes the `axum-errors-handler-trait` skill. It covers the
opaque `Handler is not satisfied` compiler error, the five root causes that
produce it, the `#[debug_handler]` diagnostic macro, and a symptom-to-cause-to-fix
decision tree. It builds directly on `axum-syntax-handlers-research.md` (the
blanket-impl criteria) and adds the diagnostic angle.

---

## 1. The Error and Why It Is Opaque

Axum's no-macros design enforces handler eligibility entirely through the
`Handler<T, S>` trait and its blanket implementation. When a function fails ANY
of the blanket-impl criteria, the compiler does not say which criterion failed.
Instead it emits a variant of:

```
error[E0277]: the trait bound `fn(bool) -> impl Future<Output = ()> {handler}:
              Handler<_, _>` is not satisfied
   --> src/main.rs:8:25
    |
  8 |     Router::new().route("/", get(handler))
    |                              --- ^^^^^^^ the trait `Handler<_, _>` is not
    |                              |           implemented for fn item `...`
    |                              required by a bound introduced by this call
    |
    = help: the following other types implement trait `Handler<T, S>`:
              `MethodRouter<S>` implements `Handler<(), S>`
              `axum::handler::Layered<L, H, T, S>` implements `Handler<T, S>`
note: required by a bound in `axum::routing::get`
```

The error is opaque because the blanket impl is a single `impl` with a long
conjunction of bounds (`async fn`, every argument `FromRequestParts`/`FromRequest`,
return `IntoResponse`, future `Send`, at most 16 arguments). Rust reports the
whole `impl` as unsatisfied; it cannot point at the one conjunct that failed.
The error names `Handler<_, _>` with inference placeholders, lists unrelated
`Handler` impls as "help", and gives no hint about the argument or return type
at fault. This is the single most-reported pain point of Axum's design and the
sole reason the `#[debug_handler]` macro exists.

---

## 2. The Five Root Causes

Every `Handler is not satisfied` error reduces to one of five causes. Each is
shown below with a minimal reproducer and the fix.

### Cause 1 — A non-extractor argument

An argument type implements neither `FromRequestParts<S>` nor `FromRequest<S, M>`.
Primitive types (`bool`, `i32`, `String` as a bare argument), domain structs, and
config types are not extractors.

```rust
// FAILS - `bool` is not an extractor
async fn handler(flag: bool) {}

// FIX - take an extractor; read the value through it
async fn handler(Query(params): Query<Filter>) {}
// or, for shared config: put it in State
async fn handler(State(cfg): State<AppConfig>) {}
```

### Cause 2 — A body extractor is not the last argument

Exactly one argument may consume the request body (`FromRequest`); it MUST be the
last argument. All preceding arguments must be parts-only (`FromRequestParts`).
Placing `Json`, `Bytes`, `String`, `Form`, or `Multipart` before another argument
breaks the rule. Two body extractors break it for the same reason — the body is
single-use.

```rust
// FAILS - `Bytes` (body extractor) is before `Path` (parts extractor)
async fn handler(body: Bytes, Path(id): Path<u64>) {}

// FAILS - two body extractors
async fn handler(body: String, Json(p): Json<Payload>) {}

// FIX - parts extractors first, the single body extractor last
async fn handler(Path(id): Path<u64>, body: Bytes) {}
```

### Cause 3 — The return type does not implement `IntoResponse`

The handler's return type must implement `IntoResponse`. Bare primitives,
arbitrary structs, and `Result<T, E>` where `E` is not `IntoResponse` all fail.

```rust
// FAILS - bare `i32` is not a response
async fn handler() -> i32 { 42 }

// FAILS - `Result` whose error type does not implement IntoResponse
async fn handler() -> Result<String, std::io::Error> { /* ... */ }

// FIX - return a type that implements IntoResponse
async fn handler() -> Json<i32> { Json(42) }
// or implement IntoResponse for a custom AppError used in Result
async fn handler() -> Result<String, AppError> { /* ... */ }
```

### Cause 4 — The function is not `async`

A plain `fn` is never a handler, even one that returns `impl Future`. The blanket
impl requires `async fn`.

```rust
// FAILS - synchronous fn
fn handler() -> &'static str { "hello" }

// FAILS - returns a Future but is not `async fn`
fn handler() -> impl Future<Output = &'static str> { async { "hello" } }

// FIX - declare the function `async`
async fn handler() -> &'static str { "hello" }
```

### Cause 5 — The produced future is `!Send`

The handler future must be `Send`. Holding a non-`Send` value across an `.await`
makes the whole future `!Send`, so the blanket impl no longer applies. Common
offenders: `Rc`, `RefCell` borrows held across `.await`, and
`std::sync::MutexGuard`.

```rust
// FAILS - std::sync::MutexGuard is held across the await, future is !Send
async fn handler(State(m): State<Arc<std::sync::Mutex<i32>>>) {
    let guard = m.lock().unwrap();
    do_async_work().await;          // guard still alive here -> !Send
    let _ = *guard;
}

// FIX A - drop the guard before the await (narrow the lock scope)
async fn handler(State(m): State<Arc<std::sync::Mutex<i32>>>) {
    { let _v = *m.lock().unwrap(); }   // guard dropped at block end
    do_async_work().await;
}

// FIX B - use tokio::sync::Mutex, whose guard is Send and held across await
async fn handler(State(m): State<Arc<tokio::sync::Mutex<i32>>>) {
    let guard = m.lock().await;
    do_async_work().await;             // tokio guard is Send
    let _ = *guard;
}
```

Note: the 16-argument limit is a sixth, far rarer failure mode (no blanket impl
exists for 17+ argument tuples). The fix is to group extractors into one custom
`FromRequestParts`/`FromRequest` struct. It is covered fully in
`axum-syntax-handlers`; this errors skill should mention it only briefly.

---

## 3. The `#[debug_handler]` Macro — The Diagnostic Workflow

`#[debug_handler]` is an attribute macro that transforms the opaque trait-bound
error into a precise, plain-language one pointing at the exact argument or return
type at fault. It is THE remedy for diagnosing all five causes.

**Where it lives.** The macro is defined in the `axum-macros` crate
(`axum_macros::debug_handler`, verified at axum-macros 0.5.1). It is also
re-exported from the main `axum` crate as `axum::debug_handler`. The re-export is
"Available on crate feature `macros` only" (verified verbatim) — the `macros`
feature must be enabled on the `axum` dependency:

```toml
# Cargo.toml - enable the macros feature for axum::debug_handler
axum = { version = "0.8", features = ["macros"] }
```

**How it produces precise errors.** Applied to a handler, the macro generates
extra checking code (synthetic functions such as
`__axum_macros_check_handler_0_from_request_check`) that asserts each criterion
independently. Because each assertion is its own item, the compiler can pin the
failure to a single argument or the return type. Example output for the
not-async case (verified verbatim from the `axum-macros` docs):

```
error: handlers must be async functions
  --> main.rs:xx:1
   |
xx | fn handler() -> &'static str {
   | ^^
```

For a non-extractor argument, the macro reports that the argument does not
implement `FromRequestParts`/`FromRequest`; for a bad return type, it reports
that the return type does not implement `IntoResponse`; for a `!Send` future, it
reports the `Send` violation — each pointing at the precise span rather than the
whole `route(...)` call.

**State inference.** By default the macro assumes a state type of `()`. If the
handler has a `State<T>` argument the macro infers `T` automatically. For handlers
with multiple or ambiguous state arguments, set it explicitly with
`#[debug_handler(state = AppState)]` (verified).

**Dev-only behavior.** Verified verbatim: "This macro has no effect when compiled
with the release profile." It is safe to leave on a handler permanently — it adds
zero release-build cost. For strict hygiene it may be gated with
`#[cfg_attr(debug_assertions, debug_handler)]`, though this is not required since
the macro is already a release no-op.

**Limitation.** The macro does not work for methods inside `impl` blocks that lack
a `self` parameter; it then emits
`error[E0425]: cannot find function __axum_macros_check_handler_0_from_request_check`
(verified). For such associated functions, move the logic to a free `async fn`
or add diagnostics another way.

```rust
// The diagnostic workflow: annotate the failing handler, recompile, read the
// precise error, fix, then optionally remove or leave the attribute.
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

---

## 4. Symptom-to-Cause-to-Fix Decision Tree

```
SYMPTOM: error[E0277] "the trait bound `...{handler}: Handler<_, _>` is not satisfied"

STEP 0  Annotate the handler with #[debug_handler] and recompile.
        The macro turns the opaque error into a precise one. Then:

|-- Macro says "handlers must be async functions"
|       CAUSE 4 : function is not async
|       FIX     : add `async` to the `fn`
|
|-- Macro points at an argument: "does not implement FromRequestParts / FromRequest"
|       |-- argument type is a primitive / domain struct / config type
|       |       CAUSE 1 : non-extractor argument
|       |       FIX     : use a real extractor (Query, Path, State, Json, ...)
|       |
|       |-- argument IS an extractor but a body extractor sits before another arg,
|       |   or there are two body extractors
|       |       CAUSE 2 : body extractor not last
|       |       FIX     : move the single body extractor to the last position;
|       |                 keep at most one body extractor
|
|-- Macro points at the return type: "does not implement IntoResponse"
|       CAUSE 3 : return type not IntoResponse
|       FIX     : return Json<T>/String/StatusCode/impl IntoResponse, or make
|                 the Result error type implement IntoResponse
|
|-- Macro points at a `Send` bound on the future
|       CAUSE 5 : future is !Send
|       FIX     : drop !Send guards before `.await`, or swap
|                 std::sync::Mutex for tokio::sync::Mutex; avoid Rc/RefCell
|
|-- Macro emits E0425 cannot find `__axum_macros_check_handler_..._check`
        NOT a handler bug : #[debug_handler] on an impl-block method without `self`
        FIX : extract a free async fn, or diagnose without the macro

EDGE CASE: handler has 17+ arguments -> no blanket impl exists at all.
        FIX : group extractors into one custom FromRequestParts/FromRequest struct.
```

---

## 5. Verified Snippets Summary

The skill may reuse these six verified snippets:

1. The opaque `E0277` error block (Section 1).
2. Cause 1 — non-extractor argument: `async fn handler(flag: bool)` fails; fix
   with `Query`/`State`.
3. Cause 2 — body extractor before parts extractor: `async fn handler(body: Bytes,
   Path(id): Path<u64>)` fails; fix by reordering.
4. Cause 4 — non-async: `fn handler() -> &'static str` fails; fix by adding
   `async`.
5. Cause 5 — `!Send` future from `std::sync::MutexGuard` across `.await`; fix by
   scoping the guard or using `tokio::sync::Mutex`.
6. The `#[debug_handler]` usage block plus the verbatim "handlers must be async
   functions" macro error and the `macros` feature in `Cargo.toml`.

---

## 6. Sources Verified

All URLs below were fetched and verified via WebFetch on **2026-05-20**.

- https://docs.rs/axum/latest/axum/handler/trait.Handler.html — `Handler<T, S>`
  trait signature, supertraits (`Clone + Send + Sync + Sized + 'static`),
  associated `Future` type (`Future<Output = Response> + Send + 'static`), the
  `call` / `with_state` / `layer` methods, and the six blanket-impl requirements:
  `async fn`, "Take no more than 16 arguments that all implement `Send`", "All
  except the last argument implement `FromRequestParts`", "The last argument
  implements `FromRequest`", "Returns something that implements `IntoResponse`",
  closures "`Clone + Send` and be `'static`", "Returns a future that is `Send`".
- https://docs.rs/axum-macros/latest/axum_macros/attr.debug_handler.html — the
  `#[debug_handler]` macro purpose ("Generates better error messages"), the
  verbatim "handlers must be async functions" example error, default `()` state
  with `State`-argument inference, the `#[debug_handler(state = ...)]` override,
  the `impl`-block-without-`self` limitation
  (`E0425 cannot find __axum_macros_check_handler_0_from_request_check`), and the
  verbatim "This macro has no effect when compiled with the release profile".
- https://docs.rs/axum/latest/axum/attr.debug_handler.html — confirms
  `debug_handler` is re-exported by the `axum` crate from `axum-macros` (0.5.1)
  and is "Available on crate feature `macros` only".

Supporting prior research (already verified, see `axum-syntax-handlers-research.md`,
`vooronderzoek-axum.md` section 7, and
`fragments/research-a-core-routing-extractors.md` section 6, all verified
2026-05-20):
- The no-macros design and the five root causes of an invalid handler are
  enumerated in `research-a-core-routing-extractors.md` section 6 and
  `axum-syntax-handlers-research.md` section 5.
- The blanket-impl criteria and the `Handler<T, S>` shape are verified in
  `axum-syntax-handlers-research.md` sections 1 and 5.
</content>
</invoke>
