# Axum Core Async Performance: Anti-Patterns

Real async and performance mistakes, each with the symptom, why it fails at the root,
and the deterministic fix. These are observed in the Axum issue tracker and
discussions (#2045, #2321, #2508) and the official Tokio and sqlx documentation.

## AP-1: a blocking call inside an async handler starves the runtime

Symptom: the service is fine with one client but throughput collapses and tail latency
explodes under concurrent load. Requests that should be fast become slow for no
visible reason.

Why it fails: a Tokio multi-threaded runtime runs many request futures cooperatively
on a fixed pool of worker threads. A future yields control only at an `.await` point.
A synchronous blocking call (`std::thread::sleep`, `std::fs`, a blocking `rusqlite`
query, a CPU-heavy loop) has no `.await` inside it, so it never yields. While it runs,
the worker thread is frozen and every other request future scheduled on that worker is
starved. The Axum discussions document exactly this: a synchronous file save under a
`Mutex`, and handlers running blocking SQLite queries, each stalling whole workers.

Fix: ALWAYS use the async equivalent for I/O (`tokio::time::sleep`, `tokio::fs`,
`sqlx`). ALWAYS move CPU-bound or unavoidable synchronous work onto the blocking pool
with `tokio::task::spawn_blocking` and `.await` the `JoinHandle`. NEVER call a
blocking function directly in an async handler.

## AP-2: using tokio::spawn to "escape" a blocking call

Symptom: a developer notices a blocking call, wraps it in `tokio::spawn(async move {
... })`, and the runtime still stalls under load.

Why it fails: `tokio::spawn` schedules the future onto the SAME worker thread pool
that handlers run on. A blocking closure inside a spawned future freezes a worker
exactly like a blocking call in the handler itself. `tokio::spawn` changes which task
owns the work, not which pool runs it.

Fix: ALWAYS use `tokio::task::spawn_blocking` for blocking or CPU-bound work. Only
`spawn_blocking` runs the closure on the separate blocking thread pool. Reserve
`tokio::spawn` for genuinely async concurrent work.

## AP-3: holding a std::sync::MutexGuard across .await

Symptom: the cryptic `the trait bound ... Handler is not satisfied` compiler error
appears after adding an `.await` to a handler that locks a `std::sync::Mutex`.

Why it fails: a `std::sync::MutexGuard` is `!Send`. The Axum `Handler` trait requires
a `Send` future, because Tokio's multi-threaded scheduler can migrate a task between
worker threads. A `!Send` value alive across an `.await` makes the whole future
`!Send`, so the `Handler` blanket impl stops applying. Even if it compiled, holding
the lock across suspension would serialise every request that needs the lock.

Fix: scope the `std::sync::Mutex` guard inside a block so it drops BEFORE any `.await`.
When an `.await` genuinely must happen while the lock is held, use
`tokio::sync::Mutex`, whose guard is `Send`. ALWAYS prefer scoping: `std::sync::Mutex`
is faster when uncontended. Add `#[debug_handler]` to decode the error precisely.

## AP-4: holding a tracing span guard across .await

Symptom: traces show interleaved, overlapping spans, and recorded timing or
parent-child structure is wrong.

Why it fails: `let _g = span.enter();` keeps the span entered on the current thread.
When the future yields at an `.await`, the worker runs other futures while the span is
still considered entered, so spans from unrelated tasks interleave. The official
`tracing` docs explicitly warn this "may produce incorrect traces".

Fix: NEVER hold a `Span::enter()` guard across `.await`. Annotate the async function
with `#[instrument]`, or attach the span with `Future::instrument`, which enters the
span only while the future is actually being polled. See the `axum-impl-tracing` skill.

## AP-5: creating a new database pool per request

Symptom: the service works at low traffic, then under load every request fails with a
database connection error.

Why it fails: calling `PgPoolOptions::new()...connect()` inside a handler builds a
brand-new connection pool on every request. A database has a finite global connection
limit; a per-request pool opens fresh TCP and TLS connections (50 to 100 ms each for
Postgres over TLS) and quickly exhausts the server's connection slots. It also
discards the entire benefit of pooling: connection reuse.

Fix: ALWAYS build one pool in `main` and share it via `.with_state(pool)`. A `Pool` is
a cheap reference-counted `Clone`, so it needs no `Arc` wrapper and no per-request
allocation. See the `axum-impl-database` skill.

## AP-6: no acquire_timeout, so a saturated pool hangs requests

Symptom: under load, requests do not error, they just hang indefinitely until the
client itself times out.

Why it fails: `pool.acquire()` waits for a free connection. Without `acquire_timeout`
set, that wait has no bound. When the pool is saturated, every additional request
queues forever instead of failing, so latency grows without limit and the service
appears frozen even though it is technically still running.

Fix: ALWAYS set `acquire_timeout` on the pool. A saturated pool then fails fast: map
the resulting error to a `503 Service Unavailable` so clients get a quick, honest
answer and can retry or back off.

## AP-7: raising worker_threads to "go faster"

Symptom: a developer raises `worker_threads` well above the CPU core count expecting
more throughput, and sees no gain or a regression.

Why it fails: an Axum service is I/O-bound, not CPU-bound. Worker threads run async
futures cooperatively, and adding more of them than there are cores does not create
more parallelism for I/O-bound work. It only adds context-switching overhead and
cache pressure. The Tokio default for `worker_threads` is the core count precisely
because that is correct for almost every async service.

Fix: ALWAYS leave `worker_threads` at the default. Drop to an explicit
`tokio::runtime::Builder` only for a measured reason: a profiling thread name, a
deterministic single-worker test, or a non-default stack size. NEVER tune the runtime
by reflex.

## AP-8: monopolising the spawn_blocking pool with sustained CPU work

Symptom: I/O offload (a `spawn_blocking` file read, a blocking driver call) starts
queueing and slowing down after the service runs heavy CPU work.

Why it fails: `spawn_blocking` and Tokio's own blocking file I/O share one blocking
thread pool (default ceiling 512). Sustained, parallel CPU work submitted through
`spawn_blocking` occupies those threads, so other `spawn_blocking` callers queue
behind it. Raising `max_blocking_threads` does not fix the root cause, it just
oversubscribes the CPU.

Fix: use `spawn_blocking` for occasional CPU work and for synchronous I/O. For
sustained, parallel CPU work use a bounded `rayon` thread pool, and bridge it to async
with a `tokio::sync::oneshot` channel so the handler still `.await`s the result.

## Sources verified

All fetched via WebFetch on 2026-05-20:

- https://docs.rs/tokio/latest/tokio/task/fn.spawn.html : `tokio::spawn` runs on the
  worker pool.
- https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html : the separate
  blocking pool and its 512 default ceiling.
- https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html : the core-count
  `worker_threads` default.
- https://docs.rs/sqlx/latest/sqlx/struct.Pool.html : `Pool` is a cheap
  reference-counted `Clone`, `acquire` and `acquire_timeout` semantics.
- https://docs.rs/tracing/latest/tracing/ : the async span-guard warning.
- https://github.com/tokio-rs/axum/discussions/2045, /2321, /2508 : real-world
  blocking-call-starves-the-runtime reports.
