# Topic Research : axum-impl-database

> Skill : `axum-impl-database`
> Phase : 4 (topic research)
> Researched : 2026-05-20
> Target versions : Axum 0.7 and 0.8 (the `DatabaseConnection` extractor differs by version)
> Database layer : `sqlx` (current docs.rs release), Postgres-focused, MySQL/SQLite analogous

This file is the verified research basis for the `axum-impl-database` skill. All
API signatures, the example structure, and the `Pool` Clone semantics were
WebFetch-verified on 2026-05-20 against `docs.rs/sqlx`, the `sqlx` `Pool` and
`PoolOptions` pages, and the canonical `tokio-rs/axum` `examples/sqlx-postgres`
crate. No API is reproduced from memory.

---

## 1. Core principle : one pool, created once, shared via `with_state`

A `sqlx` connection pool is an expensive resource (each connection is a real
TCP, and for Postgres usually TLS, socket plus a server-side session). It MUST
be created exactly once at application startup and shared. The verified
`examples/sqlx-postgres` pattern builds the pool in `main` and hands it to the
router with `.with_state(pool)`:

```rust
let pool = PgPoolOptions::new()
    .max_connections(5)
    .acquire_timeout(Duration::from_secs(3))
    .connect(&db_connection_str)
    .await
    .expect("can't connect to database");

let app = Router::new()
    .route(
        "/",
        get(using_connection_pool_extractor).post(using_connection_extractor),
    )
    .with_state(pool);
```

NEVER call `PgPoolOptions::new()...connect()` inside a handler. A per-request
pool opens fresh connections on every request, exhausts the database server's
`max_connections` limit, and discards the entire benefit of pooling.

The connection string comes from the environment, never a literal. The verified
example reads `DATABASE_URL`:

```rust
let db_connection_str = std::env::var("DATABASE_URL")
    .unwrap_or_else(|_| "postgres://postgres:password@localhost".to_string());
```

In production code, fail fast on a missing `DATABASE_URL` rather than falling
back to a localhost default.

---

## 2. `Pool` is a cheap reference-counted `Clone` : NO `Arc` needed

Verified from `docs.rs/sqlx/latest/sqlx/struct.Pool.html`. The struct is:

```rust
pub struct Pool<DB>(/* private fields */)
where
    DB: Database;
```

`Pool` implements `Clone`, and the documented Clone behavior is: it "Returns a
new `Pool` tied to the same shared connection pool." Cloning is cheap because it
is a reference-counted handle to the inner pool state; every clone shares the
same connections. `Pool` is also `Send`, `Sync`, and `Unpin`.

This is the decisive fact for Axum state. Axum clones the state value once per
request. Because cloning a `Pool` is just an `Arc`-style refcount bump, you pass
the `PgPool` directly to `.with_state(...)`. You do NOT wrap it in
`Arc<PgPool>`, and you do NOT wrap it in a `Mutex` (the pool is internally
synchronized and hands out connections concurrently). Wrapping it in `Arc` is a
redundant double indirection; wrapping it in a `Mutex` serializes all database
access and destroys concurrency.

`PgPool` is the type alias for `Pool<Postgres>`. `MySqlPool` is `Pool<MySql>`
and `SqlitePool` is `Pool<Sqlite>`; the patterns in this skill apply uniformly,
substituting the database type.

---

## 3. `PgPoolOptions` builder : exact signatures

`PgPoolOptions` is the Postgres specialization of `PoolOptions<DB>`. Verified
builder signatures from `docs.rs/sqlx/latest/sqlx/pool/struct.PoolOptions.html`:

```rust
pub fn new() -> PoolOptions<DB>
pub fn max_connections(self, max: u32) -> PoolOptions<DB>
pub fn min_connections(self, min: u32) -> PoolOptions<DB>
pub fn acquire_timeout(self, timeout: Duration) -> PoolOptions<DB>
pub fn idle_timeout(self, timeout: impl Into<Option<Duration>>) -> PoolOptions<DB>
pub fn max_lifetime(self, lifetime: impl Into<Option<Duration>>) -> PoolOptions<DB>
pub fn test_before_acquire(self, test: bool) -> PoolOptions<DB>
pub async fn connect(self, url: &str) -> Result<Pool<DB>, Error>
pub fn connect_lazy(self, url: &str) -> Result<Pool<DB>, Error>
```

Notes verified:

- The builder is move-based (`self -> PoolOptions<DB>`), so it chains fluently.
- `new()` returns a documented "default 'sane' configuration, suitable for
  testing or light-duty applications". The exact numeric default for
  `max_connections` is NOT stated on the docs page, so the skill must NOT claim
  a specific default number; it should instead instruct readers to set
  `max_connections` explicitly.
- `connect` is `async` and resolves once at least one connection is established.
  `connect_lazy` is synchronous and returns immediately, deferring the first
  connection until the first `acquire`.
- `acquire_timeout` takes a plain `Duration`. `idle_timeout` and `max_lifetime`
  take `impl Into<Option<Duration>>`, so you can pass a `Duration` directly or
  `None` to disable.

The two settings the skill should always show are `max_connections` (the hard
ceiling) and `acquire_timeout` (so a saturated pool fails fast instead of
hanging), exactly as the verified example does.

---

## 4. Running queries : the two patterns

### 4.1 Pool reference directly

`&Pool<DB>` implements the `sqlx` `Executor` trait, so any query can be executed
against `&pool`. This is the simplest pattern and the one the verified example
uses for its `GET` handler:

```rust
async fn using_connection_pool_extractor(
    State(pool): State<PgPool>,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&pool)
        .await
        .map_err(internal_error)
}
```

Each `.await` on a query executed against `&pool` transparently acquires a
connection, runs the statement, and returns it to the pool.

### 4.2 The `DatabaseConnection` `FromRequestParts` extractor

When a handler runs several statements that should share one connection (for
example because they depend on connection-local state), check out a dedicated
connection per request with a custom extractor. Verified verbatim from the
example (this is the **Axum 0.8** form, native `async fn` in trait, no
`#[async_trait]`):

```rust
struct DatabaseConnection(sqlx::pool::PoolConnection<sqlx::Postgres>);

impl<S> FromRequestParts<S> for DatabaseConnection
where
    PgPool: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(_parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let pool = PgPool::from_ref(state);
        let conn = pool.acquire().await.map_err(internal_error)?;
        Ok(Self(conn))
    }
}

async fn using_connection_extractor(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&mut *conn)
        .await
        .map_err(internal_error)
}
```

Verified mechanics:

- The extractor depends on `PgPool: FromRef<S>`, so it works for any state type
  that exposes a `PgPool` (the whole state being a `PgPool`, or a substate via
  `#[derive(FromRef)]`).
- The checked-out `PoolConnection<Postgres>` is used as `&mut *conn`.
- `acquire`'s signature is
  `pub async fn acquire(&self) -> Result<PoolConnection<DB>, Error>`. Its wait
  time is bounded by `acquire_timeout`; when the pool is at capacity, callers
  queue fairly.
- In **Axum 0.7** this same `impl` block additionally needs the
  `#[async_trait]` attribute on it; Axum 0.8 dropped `#[async_trait]` in favor
  of native RPITIT. The skill MUST show both forms with a version note.

The shared error helper from the verified example:

```rust
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
```

---

## 5. Query macros vs runtime functions

Verified descriptions from `docs.rs/sqlx`.

**Compile-time-checked macros** connect to a real database (via `DATABASE_URL`
at build time, or an offline `.sqlx` query cache) and verify SQL syntax plus
input/output types during compilation:

- `query!` — "Statically checked SQL query with `println!()` style syntax."
- `query_as!` — "A variant of `query!` which takes a path to an explicitly
  defined struct as the output type."
- `query_scalar!` — "A variant of `query!` which expects a single column from
  the query."

**Runtime functions** perform no compile-time checking and are used for dynamic
SQL. They are prepared statements, transparently cached:

- `query()` — "Execute a single SQL query as a prepared statement."
- `query_as()` — "... Maps rows to Rust types using `FromRow`." (The output
  type derives `#[derive(sqlx::FromRow)]`.)
- `query_scalar()` — "... extract the first column of each row."

Rule for the skill: PREFER the `!` macros whenever the SQL is static, because
errors are caught at compile time. Use the runtime functions only when the
query string is built dynamically.

**Execution methods** (chained after a query, against `&pool` or `&mut *conn`):

- `fetch_one` — exactly one row; errors if zero rows.
- `fetch_optional` — `Option<Row>`; `None` if zero rows.
- `fetch_all` — `Vec` of all rows.
- `execute` — runs an `INSERT`/`UPDATE`/`DELETE`, returns a result carrying the
  affected-row count.

---

## 6. Transactions

Verified signature from the `Pool` page:

```rust
pub async fn begin(&self) -> Result<Transaction<'static, DB>, Error>
```

`pool.begin().await?` starts a transaction and returns a
`Transaction<'static, DB>` (for Postgres, the alias `PgTransaction`). Run queries
against `&mut *tx`. Finish with `tx.commit().await?` to persist or
`tx.rollback().await?` to discard.

Critical safety property : **if a `Transaction` is dropped without `commit`, it
rolls back automatically.** This means an early `return` or a `?`-propagated
error inside the handler safely abandons the transaction with no half-applied
state. The skill should present this drop-rollback as the reason transactions
in Axum handlers are safe to combine with the `?` operator.

```rust
async fn transfer(
    State(pool): State<PgPool>,
) -> Result<StatusCode, (StatusCode, String)> {
    let mut tx = pool.begin().await.map_err(internal_error)?;

    sqlx::query("update accounts set balance = balance - 100 where id = 1")
        .execute(&mut *tx)
        .await
        .map_err(internal_error)?;
    sqlx::query("update accounts set balance = balance + 100 where id = 2")
        .execute(&mut *tx)
        .await
        .map_err(internal_error)?;

    // an error on either statement above returns early; `tx` drops and rolls back.
    tx.commit().await.map_err(internal_error)?;
    Ok(StatusCode::OK)
}
```

`Pool` also offers `try_begin` (`Result<Option<Transaction>, Error>`) for a
non-waiting attempt.

---

## 7. Pool sizing guidance

`max_connections` is the hard upper bound on concurrent database connections per
application instance. Size it so that
`(number of app instances) x max_connections` stays comfortably below the
database server's own global `max_connections` limit, leaving headroom for
migrations, admin sessions, and monitoring tools.

A typical starting point is in the range of the database server's CPU core
count rather than a large number. Oversizing the pool increases lock contention
on the database and rarely improves throughput. The verified example uses just
`max_connections(5)`.

Always pair `max_connections` with `acquire_timeout`. Without it, a request that
arrives when the pool is saturated waits indefinitely; with it, `acquire` fails
after the timeout and the handler can map the error to a `503`, giving fast,
observable backpressure instead of a silent hang. `min_connections` keeps a warm
floor of connections; `max_lifetime` and `idle_timeout` recycle connections so
stale or load-balancer-dropped sockets do not accumulate.

On graceful shutdown, call `pool.close().await` so in-flight connections drain
cleanly before the process exits.

---

## 8. Verified snippets summary (for the skill's Patterns section)

1. `PgPoolOptions::new().max_connections(5).acquire_timeout(...).connect(url)` —
   the verified pool builder.
2. `Router::new().route(...).with_state(pool)` — pool-as-state, no `Arc`.
3. `State(pool): State<PgPool>` + `query_scalar(...).fetch_one(&pool)` — direct
   pool query in a handler.
4. The `DatabaseConnection(PoolConnection<Postgres>)` `FromRequestParts`
   extractor with the `PgPool: FromRef<S>` bound.
5. `pool.begin()` / `&mut *tx` / `commit` with drop-rollback semantics.
6. The `internal_error<E>` `500`-mapping helper.

---

## 9. Anti-patterns this skill must cover

- New `PgPool` per request inside a handler — exhausts the database connection
  limit; build one pool in `main`.
- Wrapping `PgPool` in `Arc` — redundant; `Pool` is already a refcounted handle.
- Wrapping `PgPool` in a `Mutex` — serializes all database access, destroys
  concurrency; the pool is already internally synchronized.
- `.unwrap()` on a query result in a handler — turns a missing row or dropped
  connection into a panic; return `Result<T, AppError>` and use `?`.
- Omitting `acquire_timeout` — a saturated pool then hangs requests forever.
- Using the Axum 0.7 `#[async_trait]` form on Axum 0.8 (or vice versa) for the
  `DatabaseConnection` extractor — the skill must show both, version-flagged.

---

## Sources verified (2026-05-20)

- https://docs.rs/sqlx/latest/sqlx/ — `query!` / `query_as!` / `query_scalar!`
  macro descriptions, `query()` / `query_as()` / `query_scalar()` runtime
  function descriptions, `PgPool` / `MySqlPool` / `SqlitePool` type aliases,
  `fetch_one` / `fetch_optional` / `fetch_all` / `execute` execution methods.
- https://docs.rs/sqlx/latest/sqlx/struct.Pool.html — `Pool<DB>` struct
  definition, `Clone` = cheap reference-counted handle to the same shared pool,
  `acquire` signature, `begin` / `try_begin` signatures, `connect` / `close` /
  `size` / `num_idle`, `Send + Sync + Unpin`.
- https://docs.rs/sqlx/latest/sqlx/pool/struct.PoolOptions.html — exact builder
  signatures for `new`, `max_connections`, `min_connections`, `acquire_timeout`,
  `idle_timeout`, `max_lifetime`, `test_before_acquire`, `connect`,
  `connect_lazy`; the "sane default" note (no explicit numeric default stated).
- https://github.com/tokio-rs/axum/blob/main/examples/sqlx-postgres/src/main.rs
  — verbatim verified example: `PgPoolOptions` configuration,
  `.with_state(pool)`, the `DatabaseConnection` `FromRequestParts` extractor
  with the `PgPool: FromRef<S>` bound, both handler patterns, and the
  `internal_error` helper.
- Cross-referenced internal fragment :
  `docs/research/fragments/research-c-errors-auth-db-ops.md` section 6
  (verified 2026-05-20).
