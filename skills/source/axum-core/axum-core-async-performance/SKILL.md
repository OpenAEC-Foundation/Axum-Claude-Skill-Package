---
name: axum-core-async-performance
description: >
  Use when an Axum service is slow under concurrent load, has high tail latency,
  freezes, or stops responding, and when a handler must run CPU-bound work, a
  blocking library call, or long-running background work.
  Prevents blocking the Tokio runtime, the !Send future trap that breaks the
  Handler trait bound, creating a database pool per request, and pool exhaustion
  that hangs requests instead of failing fast.
  Covers tokio::spawn versus tokio::task::spawn_blocking, the async I/O
  substitution table, !Send values held across .await, the tokio worker and
  blocking thread pools, runtime tuning, and connection pool sizing.
  Keywords: tokio::spawn, spawn_blocking, blocking the runtime, async performance,
  tokio::sync::Mutex, std::sync::MutexGuard not Send, JoinHandle, worker threads,
  max_blocking_threads, acquire_timeout, connection pool sizing, server slow under
  load, high tail latency, requests hang, app freezes, handler future is not Send,
  Handler is not satisfied, why is my axum server slow, how do I run blocking code,
  what is spawn_blocking.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Axum Core Async Performance

## Overview

The cardinal rule of every Axum service: NEVER block the Tokio runtime in an async
handler. This single rule is the most important operational fact about Axum, broader
than any other performance concern.

A Tokio multi-threaded runtime has a fixed pool of worker threads, and each worker runs
many request futures cooperatively. A future yields control only at an `.await` point.
A synchronous blocking call has no `.await` inside it, so it never yields: while it
runs, the worker thread is frozen and EVERY other request future scheduled on that
worker is starved. Under concurrent traffic this collapses throughput and inflates tail
latency.

The Tokio APIs in this skill are identical on Axum 0.7 and 0.8. This skill has no
version-divergent code.

This skill covers how to keep handlers non-blocking, how to offload CPU and blocking
work, the `!Send` future trap, and connection pool sizing. For the database pool
itself see `axum-impl-database`. For the cryptic `Handler is not satisfied` error see
`axum-errors-handler-trait`.

## Quick Reference

### The two offload primitives

```rust
// Concurrent ASYNC work. Future and Output must be Send + 'static.
// Panics if called outside a Tokio runtime context.
pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static;

// CPU-bound or SYNCHRONOUS blocking work. Takes a closure, NOT a future.
// Runs on the separate blocking thread pool.
pub fn spawn_blocking<F, R>(f: F) -> JoinHandle<R>
where
    F: FnOnce() -> R + Send + 'static,
    R: Send + 'static;
```

ALWAYS `.await` the returned `JoinHandle` to get the result; awaiting it yields
`Result<T, JoinError>`, where the error means the task panicked.

### Blocking call to async substitution table

| Blocking, NEVER in an async handler | Async replacement, ALWAYS use |
|-------------------------------------|-------------------------------|
| `std::thread::sleep(d)` | `tokio::time::sleep(d).await` |
| `std::fs::read` / `write` | `tokio::fs::read` / `write` (`.await`) |
| `std::fs::File` | `tokio::fs::File` |
| blocking SQL driver such as `rusqlite` | `sqlx` async API, or wrap in `spawn_blocking` |
| `std::net::TcpStream` | `tokio::net::TcpStream` |
| CPU loop, hashing, image work | `tokio::task::spawn_blocking`, awaited |

For genuine CPU-bound work there is no async equivalent: the work IS the computation.
Move it off the worker with `spawn_blocking`.

### The !Send rule

The Axum `Handler` trait requires the handler future to be `Send`, because Tokio's
multi-threaded scheduler can move a task between worker threads. A future is `Send`
only if EVERY value held across an `.await` point is `Send`. NEVER hold these across
`.await`: `std::rc::Rc`, `std::cell::RefCell` borrows, `std::sync::MutexGuard`, and a
`tracing` span `enter()` guard.

## Decision Trees

### Which offload tool for which work

```
What kind of work is this?
|- Async I/O that has a tokio equivalent (file, network, timer, sqlx)?
|     -> call the tokio / sqlx async API directly with .await. No spawn needed.
|- CPU-bound or a synchronous blocking library call with NO async API?
|     -> tokio::task::spawn_blocking(closure), then .await the JoinHandle.
|- Concurrent async work that should outlive the request (fire-and-forget)?
|     -> tokio::spawn(async move { .. }). The handler can respond immediately.
|- Sustained heavy CPU work (many cores, long duration)?
|     -> a rayon thread pool, bridged to async with a oneshot channel.
```

NEVER reach for `tokio::spawn` to "fix" a blocking call. `tokio::spawn` schedules the
future onto the SAME worker pool, so a blocking closure inside it still freezes a
worker. Only `spawn_blocking` uses the separate blocking pool.

### Which mutex when state is shared

```
Does the lock need to be held across an .await point?
|- NO, the locked section is purely synchronous?
|     -> std::sync::Mutex. Scope the guard in a block so it drops BEFORE any .await.
|        std::sync::Mutex is faster than tokio::sync::Mutex when uncontended.
|- YES, an .await must happen while the lock is held?
|     -> tokio::sync::Mutex. Its guard is Send and may legitimately cross .await.
```

ALWAYS prefer scoping a `std::sync::Mutex` guard over switching to
`tokio::sync::Mutex`. Holding any lock across `.await` serialises every request that
needs that lock, even when the guard type is `Send`.

### Diagnosing a service that is slow or frozen under load

```
The service is fine with one client but collapses under concurrency.
|- STEP 1: scan every handler for a synchronous call with no .await:
|          std::thread::sleep, std::fs::*, a blocking driver, a CPU loop.
|          Found one -> apply the substitution table or spawn_blocking.
|- STEP 2: scan for a database pool built inside a handler.
|          Found one -> build the pool once in main, share via with_state.
|- STEP 3: check the pool has an acquire_timeout set.
|          Missing -> requests queue forever on a saturated pool; set it.
|- STEP 4: check spawn_blocking is not monopolised by long-running CPU work.
|          If it is -> move sustained CPU work to a bounded rayon pool.
```

## Patterns

### Pattern: the cardinal rule, never block the runtime

ALWAYS treat an async handler body as a place where every potentially slow operation
is an `.await` on an async API. A synchronous blocking call inside an async handler
freezes the entire worker thread it runs on, starving every other request future on
that worker. This is documented in the Axum discussions (#2045, #2321, #2508): a
synchronous file save under a `Mutex`, and handlers running blocking `rusqlite`
queries, each stalled whole workers under load.

NEVER call `std::thread::sleep`, `std::fs`, a blocking database driver, or a CPU-heavy
loop directly in an async handler or in a WebSocket or SSE task. See
`references/examples.md` for the blocking-versus-async pair.

### Pattern: spawn_blocking for CPU-bound and synchronous work

ALWAYS move CPU-bound work (password hashing with `bcrypt` or `argon2`, image resizing,
a heavy computation) and unavoidable synchronous library calls onto the blocking pool
with `tokio::task::spawn_blocking`. It takes a closure, runs it on a separate blocking
thread pool, and returns a `JoinHandle` to `.await`.

```rust
use tokio::task;

async fn hash_password(password: String) -> Result<String, AppError> {
    // bcrypt is CPU-bound and synchronous: move it off the worker thread.
    let hashed = task::spawn_blocking(move || bcrypt::hash(password, 12))
        .await                       // JoinError if the blocking task panicked
        .map_err(|_| AppError::Internal)??;
    Ok(hashed)
}
```

The blocking pool spawns threads on demand up to `max_blocking_threads`, which
defaults to 512. For sustained heavy CPU work across many cores, NEVER monopolise the
blocking pool: use a bounded `rayon` pool and bridge it to async with a `oneshot`
channel.

### Pattern: tokio::spawn for concurrent fire-and-forget async work

ALWAYS use `tokio::spawn` for async work that should run independently of the request
that started it: a background notification, a fan-out of async tasks, or the
send/receive split of a WebSocket. The handler can respond immediately while the
spawned future continues.

```rust
async fn create_order(State(state): State<AppState>) -> StatusCode {
    let notifier = state.notifier.clone();
    // Respond now; the email send runs concurrently in the background.
    tokio::spawn(async move {
        if let Err(e) = notifier.send_confirmation().await {
            tracing::error!(error = %e, "confirmation email failed");
        }
    });
    StatusCode::ACCEPTED
}
```

`tokio::spawn` does NOT make blocking code safe: the spawned future still runs on a
worker thread. `tokio::spawn` panics if called outside a Tokio runtime context, which
never happens inside an Axum handler because the handler already runs on the runtime.

### Pattern: the !Send future trap

The `Handler` trait requires a `Send` future. A value that is NOT `Send` and is alive
across an `.await` makes the whole future `!Send`, and the handler stops satisfying
`Handler`, producing the cryptic `Handler is not satisfied` error. The non-`Send`
values are `Rc`, `RefCell` borrows, and `std::sync::MutexGuard`.

```rust
// WRONG: the std guard is alive across .await, the future is !Send.
async fn bad(State(state): State<AppState>) -> String {
    let guard = state.data.lock().unwrap();   // std::sync::MutexGuard, !Send
    some_async_call().await;                  // guard still alive here
    guard.clone()
}

// RIGHT: scope the guard so it drops before any .await.
async fn good(State(state): State<AppState>) -> String {
    let value = {
        let guard = state.data.lock().unwrap();
        guard.clone()                         // guard dropped at block end
    };
    some_async_call().await;                  // no guard alive: future is Send
    value
}
```

ALWAYS add `#[debug_handler]` to decode the error when it appears. NEVER hold a
`tracing` span `enter()` guard across `.await`: it interleaves spans and corrupts
traces. Use `#[instrument]` instead.

### Pattern: connection pool sizing for fail-fast behavior

ALWAYS build a `sqlx` pool ONCE at startup and share it as Axum state. A `Pool` is a
cheap reference-counted `Clone`, so NEVER wrap it in `Arc` and NEVER build one per
request. Two settings govern behavior under load:

- `max_connections(n)`: the hard upper bound on open database connections.
- `acquire_timeout(d)`: how long `acquire()` waits for a free connection before
  failing instead of hanging.

```rust
use sqlx::postgres::PgPoolOptions;
use std::time::Duration;

let pool = PgPoolOptions::new()
    .max_connections(5)
    .acquire_timeout(Duration::from_secs(3))  // saturated pool fails fast
    .connect(&std::env::var("DATABASE_URL")?)
    .await?;
```

Sizing rule: `(app instances) x max_connections` MUST stay comfortably below the
database server's global connection limit, leaving headroom for migrations and admin
tooling. ALWAYS set `acquire_timeout` so a saturated pool returns an error fast
(map it to a 503) instead of leaving requests hanging. Full database detail is in the
`axum-impl-database` skill.

### Pattern: runtime tuning only when measured

`#[tokio::main]` builds a multi-threaded runtime whose worker count defaults to the
number of CPU cores. ALWAYS leave `worker_threads` at the default for an Axum service:
Axum is I/O-bound, not CPU-bound, so extra worker threads add context-switching
overhead, not throughput. NEVER raise `worker_threads` by reflex.

Drop to an explicit `tokio::runtime::Builder` only for a measured reason: a custom
thread name for profiling, `worker_threads(1)` for a deterministic test, or a
non-default stack size. See `references/examples.md` for the explicit-builder form.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `std::thread::sleep` or `std::fs` in an async handler | Use the `tokio` async equivalent with `.await` |
| `tokio::spawn` used to "escape" a blocking call | Use `spawn_blocking`, it uses the separate blocking pool |
| `std::sync::MutexGuard` held across `.await` | Scope the guard in a block, or use `tokio::sync::Mutex` |
| Building a `PgPool` inside a handler | Build it once in `main`, share via `.with_state` |
| No `acquire_timeout` on the pool | Set it so a saturated pool fails fast with a 503 |
| Raising `worker_threads` to "go faster" | Leave it at the core-count default unless measured |

Full anti-pattern analysis with root causes is in `references/anti-patterns.md`.

## Reference Links

- `references/methods.md`: complete API signatures for `tokio::spawn`,
  `spawn_blocking`, `JoinHandle`, `tokio::time::sleep`, `tokio::fs`,
  `tokio::sync::Mutex`, the `tokio::runtime::Builder`, and `PgPoolOptions`, with
  thread-pool defaults.
- `references/examples.md`: working code for the blocking-versus-async pair,
  `spawn_blocking`, `tokio::spawn`, async file I/O, the `!Send` mutex fixes, a
  fail-fast pool, and explicit runtime tuning.
- `references/anti-patterns.md`: real performance mistakes with "why this fails"
  explanations and fixes.

Related skills:

- `axum-impl-database`: the `sqlx` pool, query macros, and transactions.
- `axum-errors-handler-trait`: diagnosing `Handler is not satisfied`, including the
  `!Send` future cause.
- `axum-impl-websockets`: never block the WebSocket task.
- `axum-impl-tracing`: why a span guard must not cross `.await`.
- `axum-impl-deployment`: release builds and production runtime hygiene.
