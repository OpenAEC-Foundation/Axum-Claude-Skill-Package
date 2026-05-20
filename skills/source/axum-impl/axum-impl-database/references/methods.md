# axum-impl-database: API Signatures

All signatures verified 2026-05-20 against `https://docs.rs/sqlx/latest/sqlx/`,
`https://docs.rs/sqlx/latest/sqlx/struct.Pool.html`,
`https://docs.rs/sqlx/latest/sqlx/pool/struct.PoolOptions.html`, and the
`tokio-rs/axum` `examples/sqlx-postgres` crate. Postgres types are shown;
substitute `MySql` or `Sqlite` for the other databases.

## Pool and its type aliases

```rust
pub struct Pool<DB>(/* private fields */)
where
    DB: Database;
```

`Pool<DB>` implements `Clone`, `Send`, `Sync`, and `Unpin`. Cloning returns
"a new Pool tied to the same shared connection pool"; it is a cheap
reference-counted handle, so every clone shares the same connections.

Type aliases:

```rust
type PgPool      = Pool<Postgres>;
type MySqlPool   = Pool<MySql>;
type SqlitePool  = Pool<Sqlite>;
```

## Pool methods

```rust
// Connection checkout.
pub async fn acquire(&self) -> Result<PoolConnection<DB>, Error>
pub fn try_acquire(&self) -> Option<PoolConnection<DB>>

// Transactions.
pub async fn begin(&self) -> Result<Transaction<'static, DB>, Error>
pub async fn try_begin(&self) -> Result<Option<Transaction<'static, DB>>, Error>

// Construction and lifecycle.
pub async fn connect(url: &str) -> Result<Pool<DB>, Error>
pub fn close(&self) -> impl Future<Output = ()>

// Introspection.
pub fn size(&self) -> u32
pub fn num_idle(&self) -> usize
```

`acquire`'s wait time is bounded by the pool's `acquire_timeout`. When the pool
is at capacity, callers queue fairly. `try_acquire` never waits: it returns
`None` immediately if no connection is free.

## PgPoolOptions builder

`PgPoolOptions` is the Postgres specialization of `PoolOptions<DB>`. The builder
is move-based (`self -> PoolOptions<DB>`), so calls chain fluently.

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

Notes:

- `new()` returns a "sane" default configuration suitable for testing or
  light-duty applications. The exact numeric default for `max_connections` is
  not documented, so ALWAYS set `max_connections` explicitly.
- `acquire_timeout` takes a plain `Duration`.
- `idle_timeout` and `max_lifetime` take `impl Into<Option<Duration>>`: pass a
  `Duration` to enable or `None` to disable.
- `connect` is async and resolves once at least one connection is established.
- `connect_lazy` is synchronous and returns immediately; the first connection
  is opened on the first `acquire`.

## Compile-time query macros

Connect to a real database at build time (via `DATABASE_URL` or an offline
`.sqlx` query cache) and verify SQL syntax plus input and output types.

```rust
sqlx::query!("SELECT id, name FROM users WHERE id = $1", id)
sqlx::query_as!(User, "SELECT id, name FROM users WHERE id = $1", id)
sqlx::query_scalar!("SELECT count(*) FROM users")
```

- `query!` static query; the returned anonymous record has typed columns.
- `query_as!` first argument is a path to an explicitly defined output struct.
- `query_scalar!` expects a single column from the query.

## Runtime query functions

No compile-time checking; used for SQL assembled at runtime. They are prepared
statements and are transparently cached.

```rust
pub fn query(sql: &str) -> Query<'_, DB, ...>
pub fn query_as<O>(sql: &str) -> QueryAs<'_, DB, O, ...>     // O: FromRow
pub fn query_scalar<O>(sql: &str) -> QueryScalar<'_, DB, O, ...>
```

Bind inputs with `.bind(value)`. `query_as` maps each row onto a type deriving
`#[derive(sqlx::FromRow)]`.

```rust
#[derive(sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
}

sqlx::query_as::<_, User>("SELECT id, name FROM users WHERE id = $1")
    .bind(user_id)
```

## Execution methods

Chained after a query, against `&pool`, `&mut *conn`, or `&mut *tx`.

```rust
.fetch_one(executor)      // -> Result<Row, Error>; errors if zero rows
.fetch_optional(executor) // -> Result<Option<Row>, Error>; None if zero rows
.fetch_all(executor)      // -> Result<Vec<Row>, Error>
.execute(executor)        // -> Result<DB::QueryResult, Error>; affected-row count
```

`DB::QueryResult` exposes `.rows_affected() -> u64` for `INSERT`, `UPDATE`, and
`DELETE` statements.

## Transaction

```rust
pub struct Transaction<'c, DB: Database> { /* ... */ }

pub async fn commit(self) -> Result<(), Error>
pub async fn rollback(self) -> Result<(), Error>
```

`pool.begin()` returns `Transaction<'static, DB>` (Postgres alias
`PgTransaction`). Run queries against `&mut *tx`. If the `Transaction` is
dropped without `commit`, it rolls back automatically.

## DatabaseConnection extractor

A custom `FromRequestParts` extractor that checks out one connection per
request. The bound `PgPool: FromRef<S>` makes it usable with any state that
exposes a `PgPool`.

```rust
struct DatabaseConnection(sqlx::pool::PoolConnection<sqlx::Postgres>);

// axum 0.8: native async fn, no attribute.
impl<S> FromRequestParts<S> for DatabaseConnection
where
    PgPool: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);
    async fn from_request_parts(
        _parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> { /* ... */ }
}

// axum 0.7: identical body, plus #[async_trait] on the impl block.
```

The checked-out `PoolConnection<Postgres>` is used as `&mut *conn`.
