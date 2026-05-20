# Topic Research : axum-core-async-performance

> Skill : `axum-core-async-performance`
> Phase : 4 (topic research)
> Researched : 2026-05-20
> Target versions : Axum 0.7 and 0.8 (runs on tokio 1.x; docs.rs tokio current at research time)
> Source fragments consulted : `vooronderzoek-axum.md` (sections 8, 9), `research-c-errors-auth-db-ops.md` (section 11), `research-b-middleware-tower-realtime.md` (section 6).

All signatures below were WebFetch-verified against docs.rs on 2026-05-20. No
signature is paraphrased; each is quoted verbatim from the official page.

---

## 1. Exact Signatures : `tokio::spawn` and `tokio::task::spawn_blocking`

### `tokio::spawn`

Verified verbatim from `https://docs.rs/tokio/latest/tokio/task/fn.spawn.html`:

```rust
pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
```

Both the future and its `Output` MUST be `Send + 'static`. The future starts
running in the background immediately when `spawn` is called. `tokio::spawn`
PANICS if called from outside a Tokio runtime context. It returns a
`JoinHandle<F::Output>`; awaiting that handle yields `Result<F::Output, JoinError>`.

Verified usage example (from the same page):

```rust
tokio::spawn(async move {
    // Process each socket concurrently.
    process(socket).await
});
```

Use `tokio::spawn` for **concurrent async work** that should run independently
of the request that started it (a background job, a fan-out of async tasks, a
WebSocket send/receive split). It does NOT make blocking code safe; the spawned
future still runs on a worker thread.

### `tokio::task::spawn_blocking`

Verified verbatim from `https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html`:

```rust
pub fn spawn_blocking<F, R>(f: F) -> JoinHandle<R>
where
    F: FnOnce() -> R + Send + 'static,
    R: Send + 'static,
```

It takes a synchronous closure (`FnOnce`, NOT a future), runs it on the
**blocking thread pool**, and returns a `JoinHandle<R>` you `.await` from async
code. The docs state Tokio "will spawn more blocking threads when they are
requested through this function until the upper limit configured on the
`Builder` is reached," and that the limit is "very large by default."

Verified usage example (from the same page):

```rust
let mut v = "Hello, ".to_string();
let res = task::spawn_blocking(move || {
    v.push_str("world");
    v
}).await?;
```

Use `spawn_blocking` for **CPU-bound or synchronous blocking work**: a slow
`std::fs` read, a `rusqlite` query, an `image` resize, a password hash
(`bcrypt`/`argon2`), a `std::thread::sleep`, or any third-party library that
has no async API.

---

## 2. The Cardinal Rule : NEVER Block the Tokio Runtime

A Tokio multi-threaded runtime has a fixed pool of **worker threads**, and each
worker runs many futures cooperatively. A future yields control only at an
`.await` point. A synchronous blocking call has no `.await` inside it, so it
never yields. While that call runs, the worker thread is frozen and EVERY other
future scheduled on that worker is starved. Under concurrent traffic this
collapses throughput and inflates tail latency. The `tokio-rs/axum`
discussions (#2045, #2321, #2508) document exactly this: a synchronous
`save_to_file()` under a `Mutex`, and handlers running blocking `rusqlite`
queries, stalling whole workers.

### Blocking vs non-blocking : a concrete pair

WRONG : blocks the worker thread for the full sleep / read.

```rust
async fn report(State(pool): State<PgPool>) -> Result<String, AppError> {
    // std::thread::sleep blocks the worker — every other future on it stalls.
    std::thread::sleep(std::time::Duration::from_secs(2));
    // std::fs::read blocks the worker until the disk returns.
    let template = std::fs::read_to_string("report.tmpl")?;
    Ok(template)
}
```

RIGHT : async I/O yields the worker at each `.await`.

```rust
async fn report(State(pool): State<PgPool>) -> Result<String, AppError> {
    // tokio::time::sleep yields; the worker runs other futures meanwhile.
    tokio::time::sleep(std::time::Duration::from_secs(2)).await;
    // tokio::fs yields the worker while the OS performs the read.
    let template = tokio::fs::read_to_string("report.tmpl").await?;
    Ok(template)
}
```

### Async I/O equivalents (the substitution table)

| Blocking (NEVER in an async handler) | Async replacement (ALWAYS use) |
|--------------------------------------|--------------------------------|
| `std::thread::sleep`                 | `tokio::time::sleep(d).await`  |
| `std::fs::read` / `write`            | `tokio::fs::read` / `write`    |
| `std::fs::File`                      | `tokio::fs::File`              |
| blocking SQL driver (`rusqlite`)     | `sqlx` (async), or wrap in `spawn_blocking` |
| `std::net::TcpStream`                | `tokio::net::TcpStream`        |
| CPU loop / hashing / image work      | `tokio::task::spawn_blocking` (or `rayon`, awaited) |

For genuine CPU-bound work there is no async equivalent: the work IS the
computation. Move it off the worker with `spawn_blocking` and `.await` the
`JoinHandle`. For sustained heavy CPU work, prefer a `rayon` thread pool and
bridge it to async with a `oneshot` channel rather than monopolising
`spawn_blocking` threads.

---

## 3. The `!Send` Future Trap

The Axum `Handler` trait requires the handler future to be `Send`, because the
multi-threaded scheduler moves tasks between worker threads. A future is `Send`
only if every value held **across an `.await` point** is `Send`. Three common
values are NOT `Send`:

- `std::rc::Rc<T>` — non-atomic refcount.
- `std::cell::RefCell<T>` (and `Cell<T>`) — non-thread-safe interior mutability.
- `std::sync::MutexGuard<'_, T>` — the guard returned by `std::sync::Mutex::lock`.

If any of these is alive at an `.await`, the whole handler future becomes
`!Send` and the `Handler` trait bound fails with a cryptic compiler error.
`#[debug_handler]` (from `axum-macros`) turns that into a precise message.

### Why `std::sync::MutexGuard` across `.await` breaks

WRONG : the guard is held across `.await`, so the future is `!Send` — and even
if it compiled, the lock would be held during suspension, serialising every
request.

```rust
async fn handler(State(state): State<AppState>) -> String {
    let guard = state.data.lock().unwrap(); // std::sync::MutexGuard, !Send
    some_async_call().await;                // guard still alive here -> !Send
    guard.clone()
}
```

FIX A : scope the `std::sync::Mutex` guard so it drops BEFORE any `.await`.

```rust
async fn handler(State(state): State<AppState>) -> String {
    let value = {
        let guard = state.data.lock().unwrap();
        guard.clone()                        // guard dropped at end of block
    };
    some_async_call().await;                 // no guard alive -> future is Send
    value
}
```

FIX B : use `tokio::sync::Mutex`, whose guard IS `Send` and may legitimately be
held across `.await`. Use this only when the lock genuinely must span an await;
otherwise FIX A is cheaper because `std::sync::Mutex` is faster when uncontended.

```rust
use tokio::sync::Mutex;

#[derive(Clone)]
struct AppState { data: std::sync::Arc<Mutex<String>> }

async fn handler(State(state): State<AppState>) -> String {
    let mut guard = state.data.lock().await;  // tokio guard is Send
    *guard = fetch_value().await;             // safe to hold across .await
    guard.clone()
}
```

NEVER hold a `tracing` span guard (`span.enter()`) across `.await` either — it
produces interleaved, incorrect traces; use `#[instrument]` instead.

---

## 4. Tokio Runtime : Worker Threads and the Blocking Pool

Two distinct thread pools exist in a multi-threaded Tokio runtime:

- **Worker threads** run async futures cooperatively. `#[tokio::main]` builds a
  multi-threaded runtime by default. Verified from
  `https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html`: the default
  worker count is *"the number of cores available to the system."*
- **Blocking threads** run the closures passed to `spawn_blocking` (and Tokio's
  own blocking file I/O). Verified from the same page: `max_blocking_threads`
  has a *"default value is 512."* Blocking threads are spawned lazily on demand
  up to that ceiling.

Verified `Builder` signatures (all from the docs.rs `Builder` page, 2026-05-20):

```rust
pub fn new_multi_thread() -> Builder
pub fn new_current_thread() -> Builder
pub fn worker_threads(&mut self, val: usize) -> &mut Self
pub fn max_blocking_threads(&mut self, val: usize) -> &mut Self
pub fn thread_name(&mut self, val: impl Into<String>) -> &mut Self
pub fn thread_stack_size(&mut self, val: usize) -> &mut Self
pub fn enable_all(&mut self) -> &mut Self
pub fn build(&mut self) -> Result<Runtime>
```

Tuning principle: the `core` count default for `worker_threads` is correct for
almost every Axum service — Axum is I/O-bound, not CPU-bound, so adding worker
threads beyond core count adds context-switching overhead, not throughput.
Tune deliberately, not by reflex. `max_blocking_threads` (512) only matters if
you run a very large number of concurrent `spawn_blocking` closures; the real
fix for that is usually a bounded `rayon` pool, not raising the ceiling.

`#[tokio::main]` is sufficient for the default case. Drop to an explicit
`Builder` only when you have a measured reason (a custom thread name for
profiling, `worker_threads(1)` for a deterministic test, a non-default stack
size).

---

## 5. Connection Pool Sizing

A `sqlx` `PgPool` is built ONCE at startup with `PgPoolOptions` and shared as
Axum state; `Pool` is a cheap reference-counted `Clone`, so no `Arc` is needed.
Two settings govern its behaviour under load:

- `max_connections(n)` — the hard upper bound on open database connections.
- `acquire_timeout(d)` — how long `pool.acquire()` waits for a free connection
  before failing instead of hanging.

Sizing principle: `(number of app instances) x max_connections` MUST stay
comfortably below the database server's global `max_connections` limit, leaving
headroom for migrations and admin tooling. Oversizing the pool increases lock
contention on the database rather than throughput; a common starting point is
the number of CPU cores on the database server. ALWAYS set `acquire_timeout` so
a saturated pool **fails fast** (return a 503) instead of leaving requests
hanging until the client times out. NEVER create a new pool per request — that
exhausts the database's connection slots (each fresh Postgres-over-TLS connect
is 50-100 ms) and discards the entire benefit of pooling.

---

## 6. Lift-Ready Verified Code Snippets

### 6.1 `spawn_blocking` for CPU work inside a handler

```rust
use tokio::task;

async fn hash_password(Json(req): Json<NewUser>) -> Result<String, AppError> {
    let password = req.password;
    // bcrypt is CPU-bound and synchronous -> move it off the worker thread.
    let hashed = task::spawn_blocking(move || bcrypt::hash(password, 12))
        .await            // JoinError if the blocking task panicked
        .map_err(|_| AppError::Internal)??;
    Ok(hashed)
}
```

### 6.2 `tokio::spawn` for fire-and-forget concurrent async work

```rust
async fn create_order(State(state): State<AppState>) -> StatusCode {
    let notifier = state.notifier.clone();
    // Respond immediately; the email send runs concurrently in the background.
    tokio::spawn(async move {
        if let Err(e) = notifier.send_confirmation().await {
            tracing::error!(error = %e, "confirmation email failed");
        }
    });
    StatusCode::ACCEPTED
}
```

### 6.3 Async file I/O instead of `std::fs`

```rust
use tokio::io::AsyncWriteExt;

async fn save_upload(mut field: axum::extract::multipart::Field<'_>)
    -> Result<(), AppError>
{
    let mut file = tokio::fs::File::create("/data/upload.bin").await?;
    while let Some(chunk) = field.chunk().await? {
        file.write_all(&chunk).await?;   // never std::io::Write here
    }
    file.flush().await?;
    Ok(())
}
```

### 6.4 Fail-fast connection pool

```rust
use sqlx::postgres::PgPoolOptions;
use std::time::Duration;

let pool = PgPoolOptions::new()
    .max_connections(5)
    .acquire_timeout(Duration::from_secs(3))  // saturated pool fails fast
    .connect(&std::env::var("DATABASE_URL")?)
    .await?;

let app = Router::new()
    .route("/", axum::routing::get(handler))
    .with_state(pool);
```

### 6.5 Explicit runtime tuning (only when measured)

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let runtime = tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .thread_name("axum-worker")           // names threads for profiling
        // worker_threads defaults to core count — leave it unless measured.
        .max_blocking_threads(512)            // default; raise only if needed
        .build()?;

    runtime.block_on(async {
        let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
        axum::serve(listener, app()).await?;
        Ok::<(), Box<dyn std::error::Error>>(())
    })
}
```

---

## 7. Sources Verified

All URLs fetched and verified on **2026-05-20**:

- `https://docs.rs/tokio/latest/tokio/task/fn.spawn.html` — exact `tokio::spawn`
  signature, `Send + 'static` bounds, `JoinHandle<F::Output>` return,
  "panics if called from outside a Tokio runtime", verbatim usage example.
- `https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html` — exact
  `spawn_blocking` signature, `FnOnce() -> R + Send + 'static` bound,
  `JoinHandle<R>` return, blocking thread pool behaviour, verbatim usage example.
- `https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html` —
  `new_multi_thread`, `new_current_thread`, `worker_threads`,
  `max_blocking_threads`, `thread_name`, `thread_stack_size`, `enable_all`,
  `build` signatures; default worker count = number of cores; default
  `max_blocking_threads` = 512.
- `https://docs.rs/sqlx/latest/sqlx/struct.Pool.html` — `Pool` is `Clone` and
  reference-counted, `acquire()`/`begin()` semantics (via fragment C, verified
  2026-05-20).
- `https://github.com/tokio-rs/axum/discussions/2045`,
  `https://github.com/tokio-rs/axum/discussions/2321`,
  `https://github.com/tokio-rs/axum/discussions/2508` — real-world
  blocking-call-starves-the-runtime anti-patterns (via fragment C, verified
  2026-05-20).
- `https://docs.rs/axum/latest/axum/extract/struct.State.html` — `std::sync::Mutex`
  guard across `.await` makes the handler future `!Send` (via fragment B,
  verified 2026-05-20).
</content>
</invoke>
