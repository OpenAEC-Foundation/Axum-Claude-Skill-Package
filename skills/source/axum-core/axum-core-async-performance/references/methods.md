# Axum Core Async Performance: API Signatures

Complete API signatures for the async-performance surface: the offload primitives, the
async I/O substitutes, the synchronization primitives, the runtime builder, and the
connection pool. All signatures verified against the SOURCES.md approved URLs via
WebFetch on 2026-05-20.

These are Tokio and sqlx APIs. They are identical on Axum 0.7 and 0.8. This skill has
no version-divergent code.

## tokio::spawn

```rust
pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static;
```

- Both the future and its `Output` MUST be `Send + 'static`.
- The future starts running in the background immediately.
- It runs on the SAME worker thread pool that handlers run on. A blocking closure
  inside a spawned future still freezes a worker.
- It PANICS if called from outside a Tokio runtime context. Inside an Axum handler
  this never happens, because the handler already runs on the runtime.
- Returns `JoinHandle<F::Output>`.

## tokio::task::spawn_blocking

```rust
pub fn spawn_blocking<F, R>(f: F) -> JoinHandle<R>
where
    F: FnOnce() -> R + Send + 'static,
    R: Send + 'static;
```

- Takes a synchronous closure (`FnOnce`), NOT a future.
- Runs the closure on the separate blocking thread pool, never on a worker thread.
- Returns `JoinHandle<R>`.
- Per the official docs, Tokio "will spawn more blocking threads when they are
  requested through this function until the upper limit configured on the `Builder`
  is reached. After reaching the upper limit, the tasks are put in a queue." The
  default limit is intentionally large (512).

## JoinHandle

```rust
pub struct JoinHandle<T> { /* ... */ }

// JoinHandle<T> implements Future<Output = Result<T, JoinError>>
impl<T> Future for JoinHandle<T> {
    type Output = Result<T, JoinError>;
}

impl<T> JoinHandle<T> {
    pub fn abort(&self);
    pub fn is_finished(&self) -> bool;
}
```

- Awaiting a `JoinHandle<T>` yields `Result<T, JoinError>`. An `Err(JoinError)` means
  the task panicked or was aborted.
- A handler that calls `spawn_blocking` and then `.await??` propagates first the
  `JoinError`, then the closure's own `Result`.
- `abort()` requests cancellation. It is used to stop the surviving half of a
  WebSocket send/receive split.

## Async I/O substitutes

```rust
// Timer: yields the worker instead of freezing it.
pub async fn tokio::time::sleep(duration: Duration);

// File I/O: each call yields the worker while the OS performs the operation.
pub async fn tokio::fs::read<P: AsRef<Path>>(path: P) -> io::Result<Vec<u8>>;
pub async fn tokio::fs::read_to_string<P: AsRef<Path>>(path: P) -> io::Result<String>;
pub async fn tokio::fs::write<P, C>(path: P, contents: C) -> io::Result<()>;

// tokio::fs::File mirrors std::fs::File with async methods.
impl tokio::fs::File {
    pub async fn create<P: AsRef<Path>>(path: P) -> io::Result<File>;
    pub async fn open<P: AsRef<Path>>(path: P) -> io::Result<File>;
}
// Writing requires the AsyncWriteExt trait in scope:
//   use tokio::io::AsyncWriteExt;
//   file.write_all(&bytes).await?;
//   file.flush().await?;
```

ALWAYS use these in an async handler. NEVER use the `std::thread::sleep`,
`std::fs::*`, or `std::io::Write` equivalents in an async handler.

## tokio::sync::Mutex

```rust
pub struct Mutex<T: ?Sized> { /* ... */ }

impl<T: ?Sized> Mutex<T> {
    pub async fn lock(&self) -> MutexGuard<'_, T>;
    pub fn try_lock(&self) -> Result<MutexGuard<'_, T>, TryLockError>;
}
```

- `tokio::sync::Mutex::lock` is `async` and its `MutexGuard` is `Send`, so the guard
  MAY be held across an `.await`.
- Use it ONLY when the lock genuinely must span an `.await`. For a purely synchronous
  critical section, `std::sync::Mutex` with a scoped guard is faster when uncontended.

```rust
// std::sync::Mutex for comparison: its guard is NOT Send.
pub fn std::sync::Mutex::lock(&self)
    -> Result<MutexGuard<'_, T>, PoisonError<MutexGuard<'_, T>>>;
```

A `std::sync::MutexGuard` held across `.await` makes the handler future `!Send` and
breaks the `Handler` trait bound.

## tokio::runtime::Builder

```rust
pub fn new_multi_thread() -> Builder;    // requires the rt-multi-thread feature
pub fn new_current_thread() -> Builder;
pub fn worker_threads(&mut self, val: usize) -> &mut Self;
pub fn max_blocking_threads(&mut self, val: usize) -> &mut Self;
pub fn thread_name(&mut self, val: impl Into<String>) -> &mut Self;
pub fn thread_stack_size(&mut self, val: usize) -> &mut Self;
pub fn enable_all(&mut self) -> &mut Self;
pub fn build(&mut self) -> Result<Runtime>;
```

Thread-pool defaults verified from the docs:

- `worker_threads` default: "the number of cores available to the system". These run
  async futures cooperatively.
- `max_blocking_threads` default: 512. These run `spawn_blocking` closures and Tokio's
  own blocking file I/O. They are spawned lazily on demand up to this ceiling.

`#[tokio::main]` builds a multi-threaded runtime with these defaults. ALWAYS keep
`worker_threads` at the default for an Axum service. Drop to an explicit `Builder`
only for a measured reason.

## sqlx PgPoolOptions and Pool

```rust
impl PgPoolOptions {
    pub fn new() -> Self;
    pub fn max_connections(self, max: u32) -> Self;
    pub fn acquire_timeout(self, timeout: Duration) -> Self;
    pub async fn connect(self, url: &str) -> Result<Pool<Postgres>, Error>;
}

impl<DB: Database> Pool<DB> {
    pub async fn acquire(&self) -> Result<PoolConnection<DB>, Error>;
    pub async fn close(&self);
}
```

- `PgPool` is the alias for `Pool<Postgres>`. `MySqlPool` and `SqlitePool` are the
  analogues.
- The official docs state cloning a `Pool` "is cheap as it is simply a
  reference-counted handle to the inner pool state". NEVER wrap a `Pool` in `Arc`.
- `acquire()` waits up to `acquire_timeout` for a free connection, then fails.
- `close()` is called on graceful shutdown to drain connections cleanly.

Full database API detail is in the `axum-impl-database` skill.

## Sources verified

All fetched via WebFetch on 2026-05-20:

- https://docs.rs/tokio/latest/tokio/task/fn.spawn.html : `spawn` signature, the
  `Send + 'static` bounds, the runtime-context panic.
- https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html : `spawn_blocking`
  signature, the `FnOnce` bound, the blocking-pool behavior and queue note.
- https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html : the `Builder`
  method signatures, the core-count `worker_threads` default, the 512
  `max_blocking_threads` default.
- https://docs.rs/sqlx/latest/sqlx/struct.Pool.html : `Pool` is `Clone` and
  reference-counted, `acquire` and `close` semantics.
