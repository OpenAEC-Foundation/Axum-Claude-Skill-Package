# axum-core-state: Examples

Every example is assembled from signatures and code verified via WebFetch
against docs.rs on 2026-05-20. State APIs are identical across Axum 0.7 and
0.8, so examples are annotated `// axum 0.7 / 0.8`. No version difference
applies to core state.

## 1. Basic state with with_state

```rust
// axum 0.7 / 0.8
use axum::{Router, routing::get, extract::State};
use std::sync::Arc;

struct AppConfig { name: String }

#[derive(Clone)]
struct AppState {
    pool: sqlx::PgPool,         // Pool is a cheap refcounted Clone, no Arc needed
    config: Arc<AppConfig>,     // heavy and immutable, Arc keeps the clone cheap
}

async fn handler(State(state): State<AppState>) -> String {
    format!("app: {}", state.config.name)
}

#[tokio::main]
async fn main() {
    let pool: sqlx::PgPool = todo!("build the pool once at startup");
    let cfg = AppConfig { name: "demo".to_owned() };

    let app: Router = Router::new()
        .route("/", get(handler))
        .with_state(AppState { pool, config: Arc::new(cfg) });

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## 2. The Router<S> to Router<()> transition

```rust
// axum 0.7 / 0.8 - verbatim from the Router docs
#[derive(Clone)]
struct AppState {}

let router: Router<AppState> = Router::new()
    .route("/", get(|_: State<AppState>| async {}));

let router: Router<()> = router.with_state(AppState {});

let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, router).await.unwrap();
```

A handler taking `State<AppState>` forces the router to `Router<AppState>`.
`with_state` pays the debt and produces `Router<()>`, the only router type
`axum::serve` accepts.

## 3. FromRef substate composition with the derive macro

```rust
// axum 0.7 / 0.8
use axum::{Router, routing::get, extract::{State, FromRef}};

#[derive(Clone)]
struct ApiState { api_key: String }

#[derive(Clone)]
struct DbState { pool: sqlx::PgPool }

#[derive(Clone, FromRef)]              // generates FromRef<AppState> per field
struct AppState {
    api: ApiState,
    db: DbState,
}

async fn api_handler(State(api): State<ApiState>) -> String {
    format!("key len: {}", api.api_key.len())
}

async fn db_handler(State(db): State<DbState>) -> String {
    format!("connections: {}", db.pool.size())
}

let app: Router = Router::new()
    .route("/api", get(api_handler))
    .route("/db", get(db_handler))
    .with_state(AppState {
        api: ApiState { api_key: "secret".to_owned() },
        db: DbState { pool: todo!("pool") },
    });
```

## 4. Manual impl FromRef for a computed substate

When the substate is not a plain field, hand-write the impl instead of
deriving it.

```rust
// axum 0.7 / 0.8 - shape verbatim from the State docs
use axum::extract::FromRef;

#[derive(Clone)]
struct ApiState { /* ... */ }

#[derive(Clone)]
struct AppState {
    api_state: ApiState,
}

impl FromRef<AppState> for ApiState {
    fn from_ref(app_state: &AppState) -> ApiState {
        app_state.api_state.clone()
    }
}
```

This is exactly what `#[derive(FromRef)]` expands to per field. Use the derive
by default; hand-write `impl FromRef` only when the substate is computed
rather than a plain stored field.

## 5. Missing state, cause 1: with_state never called

```rust
// axum 0.8 - DOES NOT COMPILE: with_state was never called
async fn handler(State(s): State<AppState>) {}

let app = Router::new().route("/", get(handler)); // type is Router<AppState>
axum::serve(listener, app).await.unwrap();        // ERROR: expected Router<()>
```

The error reads roughly `the trait bound 'fn(...) {handler}: Handler<_, ()>'
is not satisfied` and/or `expected 'Router<()>', found 'Router<AppState>'`.
Fix: call `.with_state(AppState { ... })` before `axum::serve`.

## 6. Missing state, cause 2: wrong type passed to with_state

```rust
// axum 0.8 - DOES NOT COMPILE: handler wants AppState, router given OtherState
async fn handler(State(s): State<AppState>) {}

let app = Router::new()
    .route("/", get(handler))
    .with_state(OtherState {});  // ERROR: type mismatch, handler expects AppState
```

Fix: pass the exact state type the handlers extract.

## 7. Mutable shared state: correct with tokio::sync::Mutex

```rust
// axum 0.7 / 0.8 - CORRECT: tokio guard is Send, lock() is async
use tokio::sync::Mutex;
use std::sync::Arc;

#[derive(Clone)]
struct AppState { data: Arc<Mutex<String>> }

async fn ok_tokio(State(state): State<AppState>) {
    let mut guard = state.data.lock().await;  // async lock, Send guard
    some_async_db_call().await;               // fine: tokio guard is Send
    guard.push_str("x");
}
```

## 8. Mutable shared state: correct with a scoped std::sync::Mutex

```rust
// axum 0.7 / 0.8 - CORRECT: std guard dropped before any .await
use std::sync::{Arc, Mutex};

#[derive(Clone)]
struct AppState { data: Arc<Mutex<String>> }

async fn ok_std(State(state): State<AppState>) {
    {
        let mut guard = state.data.lock().unwrap();
        guard.push_str("x");
    } // guard dropped here, before the await below
    some_async_db_call().await;
}
```

## 9. Mutable shared state: the failing case

```rust
// axum 0.8 - WRONG: std guard held across .await, handler future is !Send
use std::sync::{Arc, Mutex};

#[derive(Clone)]
struct AppState { data: Arc<Mutex<String>> }

async fn bad(State(state): State<AppState>) {
    let mut guard = state.data.lock().unwrap(); // std MutexGuard is !Send
    some_async_db_call().await;                 // guard alive across .await
    guard.push_str("x");                        // future is !Send, compile error
}
```

`std::sync::MutexGuard` is `!Send`. A future holding it across an `.await` is
itself `!Send`, and Axum's `Handler` blanket impl requires a `Send` future.
The handler stops satisfying `Handler` and the compiler rejects it with the
same cryptic trait-bound error. Use `tokio::sync::Mutex` (example 7) or scope
the std guard (example 8).
