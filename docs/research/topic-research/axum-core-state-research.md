# Topic Research : axum-core-state

> Skill : `axum-core-state`
> Category : core
> Target versions : Axum 0.7 and 0.8 (docs.rs current at research time : 0.8.9)
> Researched : 2026-05-20
> All signatures WebFetch-verified against docs.rs on 2026-05-20.

This file is the verified research basis for the `axum-core-state` skill. It covers
`Router::with_state`, the `State<S>` extractor, the `Router<S>` typing model, `FromRef`
substate composition, `State` vs `Extension`, the "missing state" compile error, and
the `std` vs `tokio` Mutex rule for state held across `.await`.

State management identical between Axum 0.7 and 0.8 : the `State` extractor, `FromRef`,
`with_state`, and the `Router<S>` model did not change across the 0.7 -> 0.8 boundary.
All examples below apply to both versions unless annotated.

---

## 1. Exact signatures (verified verbatim, 2026-05-20)

`State<S>` is a tuple struct with one public field :

```rust
// axum 0.8 - verbatim from docs.rs/axum/latest/axum/extract/struct.State.html
pub struct State<S>(pub S);
```

It is a `FromRequestParts` extractor (no body access), so it may appear in any handler
argument position before the body extractor.

`Router::with_state` consumes the router and the state value :

```rust
// axum 0.8 - verbatim from docs.rs/axum/latest/axum/struct.Router.html
pub fn with_state<S2>(self, state: S) -> Router<S2>
```

The doc text : "Provide the state for the router. State passed to this method is global
and will be used for all requests this router receives." `Router::new` :

```rust
// verbatim
pub fn new() -> Self
```

The `FromRef` trait, used to derive substates :

```rust
// axum 0.8 - verbatim from docs.rs/axum/latest/axum/extract/trait.FromRef.html
pub trait FromRef<T> {
    fn from_ref(input: &T) -> Self;
}
```

`FromRef` is "Used to do reference-to-value conversions thus not consuming the input
value." There is a documented blanket impl : `impl<T> FromRef<T> for T where T: Clone`,
which is why a handler can extract `State<AppState>` directly from a router holding
`AppState` (the whole state implements `FromRef` for itself). `#[derive(FromRef)]` (from
`axum-macros`, re-exported as `axum::extract::FromRef`) generates one `FromRef<Parent>`
impl per field of the parent struct.

---

## 2. The `Router<S>` type-parameter mental model (verified)

The single most important fact, verbatim from the `Router` docs :

> "`Router<S>` means a router that is _missing_ a state of type `S` to be able to handle
> requests. Thus, only `Router<()>` (i.e. without missing state) can be passed to
> `serve`."

Read `Router<S>` as a debt : the router still owes a value of type `S` before it can run.
A handler that takes `State<AppState>` makes its enclosing router `Router<AppState>`.
Calling `.with_state(value)` pays the debt and changes the type to `Router<()>` :

> "Once we call `Router::with_state` the router isn't missing the state anymore ...
> Therefore the router type becomes `Router<()>`." Only `Router<()>` has the
> `into_make_service` method and can be passed to `axum::serve`.

Verified example from the docs :

```rust
// axum 0.8 - verbatim from docs.rs Router page
let router: Router<AppState> = Router::new()
    .route("/", get(|_: State<AppState>| async {}));

let router: Router<()> = router.with_state(AppState {});

let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, router).await.unwrap();
```

Consequence : the compiler enforces that every state-consuming handler is matched by a
`with_state` call of the correct type. State wiring is checked at compile time, never at
runtime. This is a deliberate Axum design strength.

---

## 3. Verified `with_state` and `FromRef` examples

State must implement `Clone` because Axum clones it into every request. Wrap heavy or
shared-mutable data in `Arc` so the clone is cheap (a pointer bump, not a deep copy) :

```rust
// axum 0.7 / 0.8 - basic AppState with with_state
use axum::{Router, routing::get, extract::State};
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    pool: sqlx::PgPool,           // Pool is itself a cheap Clone, no Arc needed
    config: Arc<AppConfig>,       // heavy/immutable -> Arc so clone is a refcount bump
}

async fn handler(State(state): State<AppState>) -> String {
    format!("connections: {}", state.pool.size())
}

let app: Router = Router::new()
    .route("/", get(handler))
    .with_state(AppState { pool, config: Arc::new(cfg) }); // Router<AppState> -> Router<()>
```

`FromRef` substate composition lets each handler depend only on the field it needs. The
`#[derive(FromRef)]` macro generates the per-field impls automatically :

```rust
// axum 0.7 / 0.8 - substate composition with #[derive(FromRef)]
use axum::extract::{State, FromRef};

#[derive(Clone)]
struct ApiState { /* ... */ }

#[derive(Clone)]
struct DbState { pool: sqlx::PgPool }

#[derive(Clone, FromRef)]               // generates FromRef<AppState> for each field
struct AppState {
    api: ApiState,
    db: DbState,
}

// each handler extracts only its substate
async fn api_handler(State(api): State<ApiState>) { /* ... */ }
async fn db_handler(State(db): State<DbState>) { /* ... */ }

let app: Router = Router::new()
    .route("/api", get(api_handler))
    .route("/db",  get(db_handler))
    .with_state(AppState { api, db });
```

The equivalent manual impl (what the derive expands to per field), verbatim from the
`State` docs :

```rust
// axum 0.7 / 0.8 - manual FromRef impl
impl FromRef<AppState> for ApiState {
    fn from_ref(app_state: &AppState) -> ApiState {
        app_state.api_state.clone()
    }
}
```

Rule : every field type that a handler wants to extract as its own `State<Field>` must be
`Clone`, because `from_ref` returns it by value. Use `#[derive(FromRef)]` by default;
hand-write `impl FromRef` only when the substate is computed rather than a plain field.

---

## 4. `State` vs `Extension` decision table

Both inject values reachable from handlers, but they differ in when the wiring is checked.

| Aspect | `State<S>` | `Extension<T>` |
|--------|-----------|----------------|
| Trait | `FromRequestParts` | `FromRequestParts` |
| Wiring | `.with_state(value)` on the `Router` | `.layer(Extension(value))` |
| Checked | Compile time : missing/wrong state is a type error | Runtime : missing extension is a `500` rejection |
| Type model | Lives in the `Router<S>` type parameter | Stored in a type-keyed map (`http::Extensions`) |
| Intended for | App-wide data known at startup : DB pools, config, caches | Per-request data injected by middleware : auth claims, user, request ID |
| Failure mode | Code does not compile | Code compiles, requests fail at runtime |

Documented guidance, verbatim : "State is global and used in every request a router with
state receives. For accessing data derived from requests, such as authorization data, see
`Extension`."

**Decision rule** : if the data is known at startup, use `State` (compile-time safety).
If the data is produced during request handling by middleware, use `Extension`. NEVER use
`Extension<AppState>` for application-wide resources : a forgotten or mis-mounted
`Extension` layer compiles fine and fails as a runtime `500` on every request, whereas a
forgotten `with_state` is a compile error caught before deployment.

---

## 5. The "missing state" compile error

Because state lives in the `Router<S>` type parameter, two mistakes produce compile-time
errors, never runtime errors. Both surface as a trait-bound error mentioning `Handler`
and `State`.

**Cause 1 : forgot `.with_state(...)` entirely.** The router keeps a non-`()` `S` type
parameter and cannot be served. `axum::serve` requires `Router<()>` (only `Router<()>` has
`into_make_service`), so the call fails to type-check :

```rust
// axum 0.8 - DOES NOT COMPILE: with_state was never called
async fn handler(State(s): State<AppState>) {}

let app = Router::new().route("/", get(handler)); // type is Router<AppState>, not Router<()>
axum::serve(listener, app).await.unwrap();         // ERROR: expected Router<()>
```

The error reads roughly : `the trait bound 'fn(...) {handler}: Handler<_, ()>' is not
satisfied` and/or `expected 'Router<()>', found 'Router<AppState>'`. The handler's
`State<AppState>` and the router's `()` state cannot be unified.

**Cause 2 : `.with_state(WrongType)`.** The handler expects one state type, the router is
given another :

```rust
// axum 0.8 - DOES NOT COMPILE: handler wants AppState, router given OtherState
async fn handler(State(s): State<AppState>) {}

let app = Router::new()
    .route("/", get(handler))
    .with_state(OtherState {});  // ERROR: type mismatch, handler expects AppState
```

**How to read it** : whenever a confusing trait-bound error references `Handler` together
with `State`, the fix is almost always (a) add the missing `.with_state(...)`, or (b)
correct the type passed to `.with_state(...)` so it matches what the handlers extract.
This is a deliberate design strength : the bug is caught at compile time.

---

## 6. `std::sync::Mutex` vs `tokio::sync::Mutex` for state held across `.await`

State is immutable-shared by default (it is `Clone`d per request). For *mutable* shared
state you need interior mutability, and the choice of mutex is load-bearing.

**The rule** : NEVER hold a `std::sync::MutexGuard` across an `.await` point. Use
`tokio::sync::Mutex` when the lock must stay held across `.await`.

Why : `std::sync::MutexGuard` is `!Send`. A future that keeps a `!Send` value alive across
an await point is itself `!Send`. Axum's `Handler` blanket impl requires the handler
future to be `Send` (tokio's multi-threaded scheduler moves tasks between worker threads),
so the handler stops satisfying `Handler` and the compiler rejects it with the same
cryptic trait-bound error. `tokio::sync::Mutex`'s guard is `Send`, and its `lock()` is an
`async fn` that yields instead of blocking the worker thread.

```rust
// axum 0.8 - WRONG: std::sync::Mutex guard held across .await -> handler is !Send
use std::sync::Mutex;

#[derive(Clone)]
struct AppState { data: Arc<Mutex<String>> }

async fn bad(State(state): State<AppState>) {
    let mut guard = state.data.lock().unwrap(); // std MutexGuard is !Send
    some_async_db_call().await;                 // guard still alive across .await
    guard.push_str("x");                        // future becomes !Send -> compile error
}
```

```rust
// axum 0.8 - CORRECT option A: tokio::sync::Mutex (guard is Send, lock() is async)
use tokio::sync::Mutex;

#[derive(Clone)]
struct AppState { data: Arc<Mutex<String>> }

async fn ok_tokio(State(state): State<AppState>) {
    let mut guard = state.data.lock().await;    // async lock, Send guard
    some_async_db_call().await;                 // fine: tokio guard is Send
    guard.push_str("x");
}
```

```rust
// axum 0.8 - CORRECT option B: scope the std guard so it drops before any .await
use std::sync::Mutex;

async fn ok_std(State(state): State<AppState>) {
    {
        let mut guard = state.data.lock().unwrap();
        guard.push_str("x");
    } // guard dropped here, before the await below
    some_async_db_call().await;
}
```

Decision : if the critical section contains an `.await`, use `tokio::sync::Mutex`. If the
critical section is purely synchronous and short, `std::sync::Mutex` is fine and faster,
as long as the guard is dropped (scoped) before any `.await`. The same `!Send`-across-
`.await` reasoning applies to `Rc` and `RefCell` guards held in state. Note `RwLock` has
the identical split : `std::sync::RwLock` vs `tokio::sync::RwLock`, same rule.

---

## 7. Verified code snippets summary

The skill should ship these small verified snippets (all assembled from verbatim docs):

1. `pub struct State<S>(pub S);` and `pub fn with_state<S2>(self, state: S) -> Router<S2>`.
2. The `Router<AppState>` -> `Router<()>` example (Section 2).
3. The `AppState` + `with_state` + `Arc` example (Section 3).
4. The `#[derive(FromRef)]` substate example and the manual `impl FromRef` (Section 3).
5. The two "missing state" non-compiling examples (Section 5).
6. The `std` vs `tokio` Mutex wrong/right pair (Section 6).

---

## Sources verified

All URLs fetched and verified via WebFetch on **2026-05-20** (docs.rs current : axum 0.8.9).

- https://docs.rs/axum/latest/axum/extract/struct.State.html - `State<S>` struct definition
  (`pub struct State<S>(pub S);`), purpose, `FromRef` substate example, State vs Extension
  wording, `Clone` requirement, `Arc<Mutex<_>>` shared-mutable example.
- https://docs.rs/axum/latest/axum/extract/trait.FromRef.html - `FromRef<T>` trait
  definition, `from_ref(input: &T) -> Self` signature, "reference-to-value conversion"
  purpose, the `impl<T> FromRef<T> for T where T: Clone` blanket impl, `#[derive(FromRef)]`.
- https://docs.rs/axum/latest/axum/struct.Router.html - `Router::new` and
  `Router::with_state<S2>(self, state: S) -> Router<S2>` signatures, the `Router<S>` =
  "missing state" model, only `Router<()>` can be served, verbatim `with_state` example.

Cross-referenced internal research (already verified 2026-05-20) :
- `docs/research/vooronderzoek-axum.md` sections 2 and 4.
- `docs/research/fragments/research-a-core-routing-extractors.md` section 4 (State vs
  Extension, `Router<S>` semantics, `!Send`-across-`.await`).
- `docs/research/fragments/research-b-middleware-tower-realtime.md` section 5 (`with_state`
  signature, `Arc`, `FromRef` verbatim example, missing-state compile error, mutex rule).
