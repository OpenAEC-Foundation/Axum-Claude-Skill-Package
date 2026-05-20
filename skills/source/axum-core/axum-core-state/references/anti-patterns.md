# axum-core-state: Anti-Patterns

Real, recurring mistakes with state in Axum, each with a root-cause analysis.
Verified against docs.rs and the Axum issue tracker on 2026-05-20. State APIs
are identical across Axum 0.7 and 0.8.

## AP-1: Using Extension for application-wide resources

```rust
// WRONG: app-wide state injected as a runtime extension
let app = Router::new()
    .route("/", get(handler))
    .layer(Extension(AppState { /* ... */ }));

async fn handler(Extension(state): Extension<AppState>) { /* ... */ }
```

WHY IT FAILS: `Extension` is dynamically typed. It stores values in a
type-keyed map. If the layer is forgotten, mounted on the wrong sub-router, or
shadowed, the handler still compiles and every request returns
`500 Internal Server Error` with "missing extension", discovered only at
runtime in production.

FIX: use `State<AppState>` plus `Router::with_state(AppState { ... })`. A
`Router<AppState>` that never receives its state does not type-check, so the
bug is caught at compile time before deployment. Reserve `Extension` for
per-request data that middleware injects (auth claims, the authenticated user,
a request id).

## AP-2: Holding a std::sync::Mutex guard across .await

```rust
// WRONG: std guard alive across an await point
async fn bad(State(state): State<AppState>) {
    let mut guard = state.data.lock().unwrap(); // std MutexGuard is !Send
    some_async_db_call().await;                 // guard still alive here
    guard.push_str("x");
}
```

WHY IT FAILS: `std::sync::MutexGuard` is `!Send`. A future that keeps a `!Send`
value alive across an `.await` is itself `!Send`. Tokio's multi-threaded
scheduler moves tasks between worker threads, and Axum's `Handler` blanket impl
requires the handler future to be `Send`. The handler stops satisfying
`Handler` and the compiler rejects it with a cryptic `Handler is not satisfied`
trait-bound error that never mentions the mutex.

FIX: use `tokio::sync::Mutex` when the lock must stay held across `.await`; its
guard is `Send` and `lock()` is an `async fn` that yields instead of blocking
the worker thread. If the critical section is purely synchronous, scope the
`std::sync::Mutex` guard in a block so it drops before any later `.await`. The
same rule applies to `RwLock` and to `Rc` or `RefCell` held in state.

## AP-3: Forgetting with_state

```rust
// WRONG: handler extracts State, router never given the state
async fn handler(State(s): State<AppState>) {}

let app = Router::new().route("/", get(handler)); // type stays Router<AppState>
axum::serve(listener, app).await.unwrap();        // does not compile
```

WHY IT FAILS: a handler that takes `State<AppState>` makes its router
`Router<AppState>`, which means "still missing an AppState". `axum::serve`
accepts only `Router<()>`, because only `Router<()>` has `into_make_service`.
The call fails to type-check with an error tying the handler's
`State<AppState>` to the router's `()` state.

FIX: call `.with_state(AppState { ... })` so the type becomes `Router<()>`.
This compile error is a design strength: the missing wiring can never reach
production.

## AP-4: Passing the wrong type to with_state

```rust
// WRONG: handler wants AppState, router given a different state type
async fn handler(State(s): State<AppState>) {}

let app = Router::new()
    .route("/", get(handler))
    .with_state(OtherState {}); // type mismatch
```

WHY IT FAILS: the handler's extracted state type and the value passed to
`with_state` must unify. A mismatch is a compile-time type error.

FIX: pass the exact state type every handler in the router extracts. When
handlers need different slices of one larger state, use `#[derive(FromRef)]`
substates instead of multiple distinct state types.

## AP-5: Heavy state without Arc

```rust
// WRONG: large data deep-copied on every request
#[derive(Clone)]
struct AppState {
    big_lookup_table: Vec<Record>,  // cloned in full per request
}
```

WHY IT FAILS: Axum clones the state into every request. A `Clone` that deep
copies a large `Vec`, `HashMap`, or `String` runs that copy on every single
request, wasting CPU and memory under load.

FIX: wrap heavy or large data in `Arc`: `big_lookup_table: Arc<Vec<Record>>`.
The per-request clone becomes a refcount increment. Connection pools such as
`sqlx::PgPool` are already cheap refcounted clones and need no `Arc`.

## AP-6: A FromRef substate field that is not Clone

```rust
// WRONG: substate field type does not derive Clone
#[derive(Clone, FromRef)]
struct AppState {
    api: ApiState,   // ApiState does NOT derive Clone
}
```

WHY IT FAILS: `#[derive(FromRef)]` generates `from_ref` impls that return each
field by value via `.clone()`. A field type that is not `Clone` makes the
generated code fail to compile, and the parent `#[derive(Clone)]` fails too.

FIX: derive `Clone` on every substate type. If a field genuinely cannot be
`Clone`, wrap it in `Arc` so the field itself becomes `Clone`.
