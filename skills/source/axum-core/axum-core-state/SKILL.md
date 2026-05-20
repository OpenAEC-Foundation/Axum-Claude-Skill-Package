---
name: axum-core-state
description: >
  Use when wiring shared application state into an Axum service: database
  pools, configuration, caches, or any value handlers must reach through the
  State extractor. Prevents the runtime-500 Extension trap, the cryptic
  "Handler is not satisfied" error caused by a missing with_state call, and
  the non-Send handler future caused by holding a std::sync::Mutex guard
  across an .await point.
  Covers Router::with_state, the State extractor, the Router<S> typing
  model, FromRef substate composition, the derive(FromRef) macro, State vs
  Extension, the missing-state compile error, and std vs tokio Mutex for
  mutable shared state.
  Keywords: axum state, with_state, State extractor, FromRef, derive FromRef,
  substate composition, Router type parameter, Extension vs State, shared
  application state, AppState, Arc, tokio Mutex, std Mutex, trait bound
  Handler is not satisfied, missing state, expected Router unit type, 500
  missing extension, how do I share a database pool, global config, app-wide
  cache, mutable shared state.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-core-state

## Overview

Axum delivers shared application state through one type parameter on `Router`
and one extractor. `Router<S>` means a router that is still missing a state
value of type `S`. `Router::with_state(value)` supplies that value and the
router type becomes `Router<()>`. Only `Router<()>` can be served. State
wiring is therefore checked entirely at compile time, never at runtime.

State APIs are identical across Axum 0.7 and 0.8: `State`, `with_state`,
`FromRef`, and the `Router<S>` model did not change at the 0.7 to 0.8
boundary. Every pattern in this skill applies to both versions.

## Quick Reference

| Item | Signature | Purpose |
|------|-----------|---------|
| `State<S>` | `pub struct State<S>(pub S);` | `FromRequestParts` extractor that yields the state |
| `Router::with_state` | `pub fn with_state<S2>(self, state: S) -> Router<S2>` | Supplies state, turns `Router<S>` into `Router<()>` |
| `Router::new` | `pub fn new() -> Self` | Creates an empty router |
| `FromRef<T>` | `fn from_ref(input: &T) -> Self;` | Reference-to-value conversion for substates |
| `#[derive(FromRef)]` | macro applied to the parent state struct | Generates one `FromRef<Parent>` impl per field |

Core rules:

- ALWAYS make application state `#[derive(Clone)]`. Axum clones the state into
  every request.
- ALWAYS wrap heavy or large state data in `Arc` so the per-request clone is a
  refcount bump, not a deep copy.
- ALWAYS use `State<S>` for data known at startup (pools, config, caches).
- NEVER use `Extension<T>` for application-wide resources. A missing
  `Extension` compiles and fails as a runtime 500 on every request.
- NEVER hold a `std::sync::MutexGuard` across an `.await` point. The handler
  future becomes `!Send` and stops satisfying `Handler`.

## Decision Trees

### State vs Extension

```
Is the value known when the server starts (pool, config, cache)?
  YES: use State<S> with Router::with_state(value).
       Wiring is checked at COMPILE time.
  NO, the value is produced per request by middleware
      (auth claims, authenticated user, request id):
       use Extension<T> with .layer(Extension(value)).
       Wiring is checked at RUNTIME.
```

NEVER reach for `Extension<AppState>` for app-wide resources. A forgotten or
mis-mounted `Extension` layer compiles fine and returns 500 on every request.
A forgotten `with_state` is a compile error caught before deployment.

### Which mutex for mutable shared state

```
Does the critical section contain an .await point?
  YES: use tokio::sync::Mutex.
       Its guard is Send, lock() is async and yields.
  NO (purely synchronous and short): std::sync::Mutex is fine and faster,
     but ALWAYS scope the guard so it drops before any later .await.
```

The same split applies to `RwLock`: `std::sync::RwLock` vs
`tokio::sync::RwLock`, with the identical rule.

### derive(FromRef) vs manual impl FromRef

```
Is each substate a plain field stored on the parent struct?
  YES: use #[derive(FromRef)] on the parent (the default choice).
  NO (the substate is computed, not stored): hand-write impl FromRef.
```

## Patterns

### Pattern: basic application state

State must implement `Clone`. A `sqlx` pool is itself a cheap `Clone`; wrap
heavy immutable data in `Arc`.

```rust
// axum 0.7 / 0.8 - identical
use axum::{Router, routing::get, extract::State};
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    pool: sqlx::PgPool,        // Pool is a cheap refcounted Clone, no Arc needed
    config: Arc<AppConfig>,    // heavy and immutable, Arc keeps the clone cheap
}

async fn handler(State(state): State<AppState>) -> String {
    format!("connections: {}", state.pool.size())
}

let app: Router = Router::new()
    .route("/", get(handler))
    .with_state(AppState { pool, config: Arc::new(cfg) });
// Router<AppState> turns into Router<()>
```

### Pattern: the Router<S> typing model

Read `Router<S>` as a debt: the router still owes a value of type `S`.

```rust
// axum 0.7 / 0.8 - verbatim from the Router docs
let router: Router<AppState> = Router::new()
    .route("/", get(|_: State<AppState>| async {}));

let router: Router<()> = router.with_state(AppState {});

let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, router).await.unwrap();
```

A handler that takes `State<AppState>` makes its enclosing router
`Router<AppState>`. `with_state` pays the debt and produces `Router<()>`. Only
`Router<()>` has `into_make_service` and can be passed to `axum::serve`.

### Pattern: FromRef substate composition

`FromRef` lets each handler depend only on the field it needs.
`#[derive(FromRef)]` generates the per-field impls automatically.

```rust
// axum 0.7 / 0.8 - identical
use axum::{Router, routing::get, extract::{State, FromRef}};

#[derive(Clone)]
struct ApiState { /* ... */ }

#[derive(Clone)]
struct DbState { pool: sqlx::PgPool }

#[derive(Clone, FromRef)]            // generates FromRef<AppState> per field
struct AppState {
    api: ApiState,
    db: DbState,
}

async fn api_handler(State(api): State<ApiState>) { /* ... */ }
async fn db_handler(State(db): State<DbState>) { /* ... */ }

let app: Router = Router::new()
    .route("/api", get(api_handler))
    .route("/db", get(db_handler))
    .with_state(AppState { api, db });
```

Every field type extracted as its own `State<Field>` MUST be `Clone`, because
`from_ref` returns it by value. `#[derive(FromRef)]` comes from `axum-macros`
and is re-exported as `axum::extract::FromRef`.

### Pattern: mutable shared state

State is immutable-shared by default (it is cloned per request). For mutable
shared state use interior mutability, and choose the mutex by whether the lock
is held across an `.await`.

```rust
// axum 0.7 / 0.8 - CORRECT: tokio::sync::Mutex, guard is Send, lock() is async
use tokio::sync::Mutex;
use std::sync::Arc;

#[derive(Clone)]
struct AppState { data: Arc<Mutex<String>> }

async fn append(State(state): State<AppState>) {
    let mut guard = state.data.lock().await;  // async lock, Send guard
    some_async_db_call().await;               // fine: tokio guard is Send
    guard.push_str("x");
}
```

See `references/examples.md` for the scoped `std::sync::Mutex` variant and the
failing `!Send` case.

### Pattern: reading the missing-state compile error

Because state lives in the `Router<S>` type parameter, two mistakes produce a
compile-time error that mentions `Handler` and `State` together.

| Symptom in the error | Root cause | Fix |
|----------------------|------------|-----|
| `expected Router<()>, found Router<AppState>` and/or `Handler<_, ()> is not satisfied` | `.with_state(...)` was never called | Add `.with_state(value)` before `axum::serve` |
| type mismatch naming `AppState` against another state type | `.with_state(WrongType)` was called | Pass the state type the handlers extract |

Whenever a confusing trait-bound error references `Handler` together with
`State`, the fix is almost always to add the missing `.with_state(...)` or to
correct the type passed to it. This is a deliberate design strength: the bug
is caught at compile time, never as a runtime failure.

## Reference Links

- `references/methods.md`: complete verified signatures for `State`,
  `with_state`, `Router::new`, `FromRef`, the `FromRef` blanket impl, and
  `#[derive(FromRef)]`.
- `references/examples.md`: full version-annotated working code, including the
  manual `impl FromRef`, both missing-state non-compiling cases, and the three
  mutable-state variants.
- `references/anti-patterns.md`: real mistakes with root-cause analysis,
  including the `Extension` runtime-500 trap and the `std::sync::Mutex` held
  across `.await` failure.

Related skills: `axum-syntax-extractors` (the full extractor set),
`axum-impl-database` (sharing a `sqlx` pool through state),
`axum-errors-handler-trait` (diagnosing the `Handler is not satisfied` error).
