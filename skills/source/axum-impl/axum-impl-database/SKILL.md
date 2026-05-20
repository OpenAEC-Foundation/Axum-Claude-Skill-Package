---
name: axum-impl-database
description: >
  Use when adding a SQL database to an Axum service, when wiring a sqlx
  connection pool into router state, when running queries or transactions
  inside handlers, or when migrating a DatabaseConnection extractor from
  Axum 0.7 to 0.8.
  Prevents creating a new pool per request, wrapping PgPool in Arc, wrapping
  PgPool in a Mutex, calling unwrap on query results, and omitting
  acquire_timeout so a saturated pool hangs every request forever.
  Covers sqlx PgPoolOptions, the Pool reference-counted Clone, with_state,
  compile-time query macros versus runtime query functions, fetch_one and
  fetch_optional and fetch_all and execute, transactions with drop-rollback,
  the DatabaseConnection FromRequestParts extractor, and pool sizing.
  Keywords: sqlx, PgPool, MySqlPool, SqlitePool, PgPoolOptions,
  max_connections, acquire_timeout, with_state, query macro, query_as,
  query_scalar, fetch_one, fetch_optional, fetch_all, execute, transaction,
  begin commit rollback, FromRequestParts, PoolConnection, FromRef,
  connection pool exhausted, too many connections, requests hang forever,
  database is slow, panic on missing row, how do I connect axum to postgres,
  what is a connection pool.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Axum Database Access with sqlx

## Overview

An Axum service talks to a SQL database through a `sqlx` connection pool. The
pool is an expensive resource: every connection is a real TCP socket (usually
with TLS) plus a server-side session. The pool MUST be created exactly once at
startup and shared with every handler through router state.

`sqlx` exposes the pool as `Pool<DB>`, with the type aliases `PgPool` for
Postgres, `MySqlPool` for MySQL, and `SqlitePool` for SQLite. This skill uses
Postgres; the patterns apply uniformly to the other databases by substituting
the type.

Three rules dominate correct database code in Axum:

1. Build the pool once in `main` and hand it to the router with
   `.with_state(pool)`. NEVER call `PgPoolOptions::new()...connect()` inside a
   handler.
2. `Pool` is already a cheap reference-counted handle. NEVER wrap a `PgPool` in
   `Arc`, and NEVER wrap it in a `Mutex`.
3. ALWAYS set `acquire_timeout`. Without it, a request that arrives when the
   pool is saturated hangs forever instead of failing fast.

## Quick Reference

| Need | API | Notes |
|------|-----|-------|
| Build the pool | `PgPoolOptions::new().max_connections(n).acquire_timeout(d).connect(url).await` | `connect` is async, resolves once connected |
| Share the pool | `Router::new()....with_state(pool)` | pass `PgPool` directly, no `Arc` |
| Read the pool in a handler | `State(pool): State<PgPool>` | `Pool` is cloned per request, cheaply |
| Run a static query | `query!` / `query_as!` / `query_scalar!` | SQL checked at compile time |
| Run a dynamic query | `query()` / `query_as()` / `query_scalar()` | runtime, no compile-time check |
| Exactly one row | `.fetch_one(&pool).await` | errors if zero rows |
| Zero or one row | `.fetch_optional(&pool).await` | returns `Option` |
| All rows | `.fetch_all(&pool).await` | returns `Vec` |
| INSERT, UPDATE, DELETE | `.execute(&pool).await` | returns affected-row count |
| Start a transaction | `pool.begin().await?` | returns `Transaction<'static, DB>` |
| Commit or roll back | `tx.commit().await?` / `tx.rollback().await?` | drop without commit auto-rolls-back |
| Dedicated connection | `DatabaseConnection` extractor | one checked-out connection per request |
| Connection string | `std::env::var("DATABASE_URL")` | NEVER a string literal |

Required crates:

```toml
[dependencies]
axum = "0.8"                                  # axum 0.8
# axum = "0.7"                                # axum 0.7
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres"] }
tokio = { version = "1", features = ["full"] }
```

For the Axum 0.7 extractor form, also add `async-trait`. See Pattern 3.

## Decision Trees

### Where do I create the pool?

```
Need a database connection in a handler?
|
+- Pool already built in main and in router state?
|  `- YES: read it with State(pool): State<PgPool>. Done.
|
`- NO pool in state yet
   `- Build it ONCE in main with PgPoolOptions, then .with_state(pool).
      NEVER build a pool inside a handler.
```

### Pool reference, dedicated connection, or transaction?

```
What does the handler do with the database?
|
+- One query, or several independent queries
|  `- State(pool): State<PgPool>; run each query against &pool.
|     Each .await acquires and returns a connection automatically.
|
+- Several statements that must share ONE connection
|  `- DatabaseConnection extractor; run against &mut *conn.
|
`- Several statements that must be atomic (all-or-nothing)
   `- Transaction: pool.begin(), run against &mut *tx, then commit.
```

### query! macro or query() function?

```
Is the SQL string known at compile time (a string literal)?
|
+- YES: use query! / query_as! / query_scalar!.
|       SQL syntax and input/output types are checked when you build.
|
`- NO, the SQL is assembled at runtime
   `- use query() / query_as() / query_scalar().
      No compile-time check; bind every input with .bind() so the
      value is never concatenated into the SQL string.
```

### Should I wrap the pool?

```
Tempted to write Arc<PgPool> or Mutex<PgPool>?
|
+- Arc<PgPool>   -> NEVER. Pool is already reference-counted. Pass PgPool.
+- Mutex<PgPool> -> NEVER. The pool is internally synchronized and hands
|                   out connections concurrently. A Mutex serializes every
|                   query and destroys concurrency.
`- Plain PgPool in state -> CORRECT.
```

## Patterns

### Pattern 1: Create the pool once, share via with_state

Build the pool in `main`, read `DATABASE_URL` from the environment, and pass the
`PgPool` straight to `.with_state(...)`.

```rust
use axum::{routing::get, Router};
use sqlx::postgres::{PgPool, PgPoolOptions};
use std::time::Duration;

#[tokio::main]
async fn main() {
    // Fail fast on a missing connection string. NEVER a string literal.
    let db_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(&db_url)
        .await
        .expect("can't connect to database");

    let app = Router::new()
        .route("/users/count", get(count_users))
        .with_state(pool); // PgPool passed directly: no Arc, no Mutex

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

`PgPool` is the state type. Axum clones the state once per request; cloning a
`Pool` is an `Arc`-style refcount bump, so every handler shares the same
connections. See `axum-core-state` for substate access with `#[derive(FromRef)]`.

### Pattern 2: Query against the pool reference

`&Pool<DB>` implements the `sqlx` `Executor` trait, so any query runs against
`&pool` directly. Each `.await` transparently acquires a connection, runs the
statement, and returns the connection to the pool.

```rust
use axum::extract::State;
use axum::http::StatusCode;
use sqlx::postgres::PgPool;

async fn count_users(
    State(pool): State<PgPool>,
) -> Result<String, (StatusCode, String)> {
    let count: i64 = sqlx::query_scalar("select count(*) from users")
        .fetch_one(&pool)
        .await
        .map_err(internal_error)?; // NEVER .unwrap() here
    Ok(format!("{count} users"))
}

// Maps any std::error::Error to a 500 response.
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
```

ALWAYS return `Result<T, E>` and propagate query errors with `?`. A `.unwrap()`
on a query result turns a missing row or a dropped connection into a panic. For
a typed application error that distinguishes 404 from 500, see
`axum-errors-handling`.

### Pattern 3: The DatabaseConnection extractor

When a handler runs several statements that must share one physical connection,
check out a dedicated connection per request with a custom `FromRequestParts`
extractor. The extractor is bounded by `PgPool: FromRef<S>`, so it works for any
state that exposes a `PgPool`.

The impl block differs by Axum version: Axum 0.8 uses a native `async fn` in the
trait; Axum 0.7 needs the `#[async_trait]` attribute on the impl block.

```rust
// axum 0.8
use axum::extract::{FromRef, FromRequestParts};
use axum::http::request::Parts;
use axum::http::StatusCode;
use sqlx::postgres::PgPool;

struct DatabaseConnection(sqlx::pool::PoolConnection<sqlx::Postgres>);

impl<S> FromRequestParts<S> for DatabaseConnection
where
    PgPool: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(
        _parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        let pool = PgPool::from_ref(state);
        let conn = pool.acquire().await.map_err(internal_error)?;
        Ok(Self(conn))
    }
}
```

```rust
// axum 0.7
#[async_trait::async_trait]
impl<S> FromRequestParts<S> for DatabaseConnection
where
    PgPool: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(
        _parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        let pool = PgPool::from_ref(state);
        let conn = pool.acquire().await.map_err(internal_error)?;
        Ok(Self(conn))
    }
}
```

The checked-out `PoolConnection<Postgres>` is used as `&mut *conn`:

```rust
async fn run_on_one_connection(
    DatabaseConnection(mut conn): DatabaseConnection,
) -> Result<String, (StatusCode, String)> {
    sqlx::query_scalar("select 'hello world from pg'")
        .fetch_one(&mut *conn)
        .await
        .map_err(internal_error)
}
```

`acquire`'s wait time is bounded by `acquire_timeout`. When the pool is at
capacity, callers queue fairly. The connection returns to the pool when `conn`
is dropped at the end of the handler. The full `DatabaseConnection` extractor is
in `references/examples.md`.

### Pattern 4: Transactions

`pool.begin().await?` starts a transaction and returns a
`Transaction<'static, DB>`. Run queries against `&mut *tx`, then call
`tx.commit().await?` to persist or `tx.rollback().await?` to discard.

The decisive safety property: if a `Transaction` is dropped without `commit`, it
rolls back automatically. An early `return` or a `?`-propagated error inside the
handler safely abandons the transaction with no half-applied state. This is why
transactions in Axum handlers combine safely with the `?` operator.

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

    // If either statement above fails, `?` returns early, `tx` is dropped,
    // and the transaction rolls back. No half-applied state ever persists.
    tx.commit().await.map_err(internal_error)?;
    Ok(StatusCode::OK)
}
```

`pool.try_begin()` returns `Result<Option<Transaction>, Error>` for a
non-waiting attempt.

### Pattern 5: Query macros versus runtime functions

Compile-time-checked macros connect to a real database at build time (via
`DATABASE_URL` or an offline `.sqlx` query cache) and verify SQL syntax plus
input and output types during compilation:

- `query!` static query with verified column types.
- `query_as!` static query mapped to an explicitly named output struct.
- `query_scalar!` static query returning a single column.

Runtime functions perform no compile-time checking and are used for dynamic
SQL. They are still prepared statements, transparently cached:

- `query()` runtime query; bind inputs with `.bind(value)`.
- `query_as()` runtime query mapped onto a type that derives
  `#[derive(sqlx::FromRow)]`.
- `query_scalar()` runtime query returning the first column.

ALWAYS PREFER the `!` macros when the SQL is a static string literal, because
errors surface at compile time. Use the runtime functions ONLY when the query
string is assembled at runtime, and bind every value with `.bind(...)` so input
is never concatenated into the SQL text. Full macro and function examples are in
`references/examples.md`.

### Pattern 6: Pool sizing

`max_connections` is the hard upper bound on concurrent database connections per
application instance. Size it so that
`(number of app instances) x max_connections` stays comfortably below the
database server's own global connection limit, leaving headroom for migrations,
admin sessions, and monitoring tools.

A typical starting point is near the database server's CPU core count, not a
large number. Oversizing the pool increases lock contention on the database and
rarely improves throughput.

ALWAYS pair `max_connections` with `acquire_timeout`. Without `acquire_timeout`,
a request that arrives when the pool is saturated waits indefinitely; with it,
`acquire` fails after the timeout and the handler can map the error to a `503`,
giving fast, observable backpressure instead of a silent hang. `min_connections`
keeps a warm floor of connections; `max_lifetime` and `idle_timeout` recycle
connections so stale or load-balancer-dropped sockets do not accumulate. On
graceful shutdown, call `pool.close().await` so in-flight connections drain
cleanly. See `axum-core-async-performance` for the broader concurrency picture.

## Reference Links

- `references/methods.md` complete API signatures: `PgPoolOptions` builder,
  `Pool` methods, query macros and functions, execution methods, `Transaction`.
- `references/examples.md` working version-annotated code: full `main.rs` with
  graceful shutdown, `query_as` CRUD with `FromRow`, the complete
  `DatabaseConnection` extractor for 0.7 and 0.8, a transaction handler.
- `references/anti-patterns.md` real mistakes and why each one fails:
  per-request pools, `Arc<PgPool>`, `Mutex<PgPool>`, `.unwrap()` on queries,
  missing `acquire_timeout`, the wrong extractor form per version.

Related skills: `axum-core-state` (router state and `FromRef`),
`axum-core-async-performance` (concurrency and blocking work),
`axum-errors-handling` (typed `AppError` with `IntoResponse`).

Sources: `https://docs.rs/sqlx/latest/sqlx/`,
`https://docs.rs/sqlx/latest/sqlx/struct.Pool.html`,
`https://docs.rs/sqlx/latest/sqlx/pool/struct.PoolOptions.html`,
`https://github.com/tokio-rs/axum/tree/main/examples/sqlx-postgres`.
