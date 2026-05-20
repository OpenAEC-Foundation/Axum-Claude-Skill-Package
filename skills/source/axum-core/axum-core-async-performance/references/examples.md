# Axum Core Async Performance: Examples

Working code for keeping Axum handlers non-blocking and offloading slow work. Every
snippet is verified against the SOURCES.md approved URLs. These Tokio and sqlx APIs
are identical on Axum 0.7 and 0.8, so each snippet carries the
`// axum 0.7 and 0.8: identical` annotation.

## Example 1: the blocking-versus-async pair

```rust
// axum 0.7 and 0.8: identical
use axum::extract::State;

// WRONG: both calls block the worker thread for their full duration.
// Every other request future on that worker is frozen meanwhile.
async fn report_blocking(State(pool): State<AppState>) -> Result<String, AppError> {
    std::thread::sleep(std::time::Duration::from_secs(2));      // freezes the worker
    let template = std::fs::read_to_string("report.tmpl")?;    // freezes the worker
    Ok(template)
}

// RIGHT: each .await yields the worker so it can run other futures.
async fn report_async(State(pool): State<AppState>) -> Result<String, AppError> {
    tokio::time::sleep(std::time::Duration::from_secs(2)).await;          // yields
    let template = tokio::fs::read_to_string("report.tmpl").await?;       // yields
    Ok(template)
}
```

ALWAYS prefer the async form. The blocking form is correct only outside an async
context (in `main` before the runtime starts, or inside a `spawn_blocking` closure).

## Example 2: spawn_blocking for CPU-bound work

```rust
// axum 0.7 and 0.8: identical
use axum::Json;
use tokio::task;

async fn hash_password(Json(req): Json<NewUser>) -> Result<String, AppError> {
    let password = req.password;
    // bcrypt is CPU-bound and synchronous: move it onto the blocking pool.
    // .await yields the worker; the closure runs on a separate blocking thread.
    let hashed = task::spawn_blocking(move || bcrypt::hash(password, 12))
        .await                              // outer ?: JoinError if it panicked
        .map_err(|_| AppError::Internal)?
        ?;                                  // inner ?: the bcrypt Result
    Ok(hashed)
}
```

`spawn_blocking` takes a closure, not a future. The double `?` unwraps first the
`JoinError`, then the closure's own `Result`.

## Example 3: tokio::spawn for fire-and-forget concurrent async work

```rust
// axum 0.7 and 0.8: identical
use axum::extract::State;
use axum::http::StatusCode;

async fn create_order(State(state): State<AppState>) -> StatusCode {
    let notifier = state.notifier.clone();
    // Respond immediately. The confirmation email runs concurrently afterwards.
    tokio::spawn(async move {
        if let Err(e) = notifier.send_confirmation().await {
            tracing::error!(error = %e, "confirmation email failed");
        }
    });
    StatusCode::ACCEPTED
}
```

Use `tokio::spawn` ONLY for async work. It schedules onto the worker pool, so a
blocking closure inside it would still freeze a worker. For blocking work use
`spawn_blocking` instead.

## Example 4: async file I/O instead of std::fs

```rust
// axum 0.7 and 0.8: identical
use tokio::io::AsyncWriteExt;          // brings write_all and flush into scope

async fn save_upload(
    mut field: axum::extract::multipart::Field<'_>,
) -> Result<(), AppError> {
    let mut file = tokio::fs::File::create("/data/upload.bin").await?;
    while let Some(chunk) = field.chunk().await? {
        file.write_all(&chunk).await?;   // NEVER std::io::Write inside an async fn
    }
    file.flush().await?;
    Ok(())
}
```

Streaming the field chunk by chunk also keeps memory bounded. Buffering a large upload
with `.bytes()` is a separate memory-exhaustion risk covered by `axum-impl-file-upload`.

## Example 5: the !Send mutex, two correct fixes

```rust
// axum 0.7 and 0.8: identical
use axum::extract::State;

// WRONG: the std guard is alive across .await, so the future is !Send and the
// Handler trait bound fails with the cryptic "Handler is not satisfied" error.
async fn bad(State(state): State<AppState>) -> String {
    let guard = state.data.lock().unwrap();   // std::sync::MutexGuard, !Send
    some_async_call().await;                  // guard still alive here
    guard.clone()
}

// FIX A: scope the std guard so it drops before any .await. Preferred: a
// std::sync::Mutex is faster than a tokio::sync::Mutex when uncontended.
async fn fix_a(State(state): State<AppState>) -> String {
    let value = {
        let guard = state.data.lock().unwrap();
        guard.clone()                         // guard dropped at the block end
    };
    some_async_call().await;                  // no guard alive: the future is Send
    value
}
```

```rust
// FIX B: use tokio::sync::Mutex when an .await genuinely must happen while the
// lock is held. Its guard is Send and may legitimately cross .await.
use tokio::sync::Mutex;
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    data: Arc<Mutex<String>>,
}

async fn fix_b(State(state): State<AppState>) -> String {
    let mut guard = state.data.lock().await;  // tokio guard is Send
    *guard = fetch_value().await;             // safe to hold across .await
    guard.clone()
}
```

ALWAYS prefer FIX A. Holding any lock across `.await`, even a `Send` one, serialises
every request that needs that lock.

## Example 6: a fail-fast connection pool

```rust
// axum 0.7 and 0.8: identical
use axum::{routing::get, Router};
use sqlx::postgres::PgPoolOptions;
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Build the pool ONCE at startup. Never inside a handler.
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))   // saturated pool fails fast
        .connect(&std::env::var("DATABASE_URL")?)
        .await?;

    // Share it as state. PgPool is a cheap reference-counted Clone: no Arc.
    let app = Router::new()
        .route("/", get(handler))
        .with_state(pool);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app).await?;
    Ok(())
}
```

Without `acquire_timeout`, a request that cannot get a connection waits forever. With
it, `acquire()` fails after 3 seconds and the handler can map that to a 503.

## Example 7: explicit runtime tuning, only when measured

```rust
// axum 0.7 and 0.8: identical
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let runtime = tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .thread_name("axum-worker")          // names threads for a profiler
        // worker_threads defaults to the core count. Leave it unless measured.
        .max_blocking_threads(512)           // the default; raise only if needed
        .build()?;

    runtime.block_on(async {
        let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
        axum::serve(listener, app()).await?;
        Ok::<(), Box<dyn std::error::Error>>(())
    })
}
```

Use `#[tokio::main]` for the default case. The explicit `Builder` is justified only
by a measured reason: profiling thread names, a deterministic single-worker test, or
a non-default stack size. NEVER raise `worker_threads` above the core-count default by
reflex: Axum is I/O-bound, and extra workers add context switching, not throughput.

## Example 8: a rayon pool for sustained CPU work

```rust
// axum 0.7 and 0.8: identical
// For sustained heavy CPU work, a rayon pool avoids monopolising the
// spawn_blocking pool. Bridge it to async with a tokio oneshot channel.
async fn render_thumbnail(image: Vec<u8>) -> Result<Vec<u8>, AppError> {
    let (tx, rx) = tokio::sync::oneshot::channel();
    rayon::spawn(move || {
        let result = expensive_cpu_render(&image);   // runs on a rayon thread
        let _ = tx.send(result);
    });
    rx.await.map_err(|_| AppError::Internal)?
}
```

`spawn_blocking` is correct for occasional CPU work. A bounded `rayon` pool is the
right tool when CPU work is sustained and parallel, so it does not starve the
blocking pool that I/O offload also depends on.

## Sources verified

All fetched via WebFetch on 2026-05-20:

- https://docs.rs/tokio/latest/tokio/task/fn.spawn.html
- https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html
- https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html
- https://docs.rs/sqlx/latest/sqlx/struct.Pool.html
- https://github.com/tokio-rs/axum/discussions/2045, /2321, /2508 (real-world
  blocking-call-starves-the-runtime reports)
