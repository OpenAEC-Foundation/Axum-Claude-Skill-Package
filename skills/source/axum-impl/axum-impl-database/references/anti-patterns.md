# axum-impl-database: Anti-Patterns

Each entry is a real mistake, why it fails, and the corrected form. Verified
2026-05-20 against `https://docs.rs/sqlx/latest/sqlx/struct.Pool.html` and the
`tokio-rs/axum` `examples/sqlx-postgres` crate.

## 1. Creating a new pool inside a handler

### Wrong

```rust
async fn list_users() -> Result<String, (StatusCode, String)> {
    // A fresh pool on every request.
    let pool = PgPoolOptions::new()
        .connect(&std::env::var("DATABASE_URL").unwrap())
        .await
        .unwrap();
    sqlx::query_scalar("select count(*) from users")
        .fetch_one(&pool)
        .await
        .map_err(internal_error)
}
```

### Why it fails

A per-request pool opens fresh connections on every request. Each connection is
a TCP and TLS handshake plus a server-side session. Under load this exhausts the
database server's global connection limit, every connection setup adds latency,
and the entire benefit of pooling is discarded. The symptom is
`too many connections` errors from the database and requests that get slower as
traffic rises.

### Correct

Build the pool once in `main` and share it via `.with_state(pool)`. The handler
reads it with `State(pool): State<PgPool>`. See Pattern 1 in `SKILL.md`.

## 2. Wrapping PgPool in Arc

### Wrong

```rust
let pool = PgPoolOptions::new().connect(&db_url).await.unwrap();
let app = Router::new().route("/", get(handler))
    .with_state(Arc::new(pool)); // redundant wrapper
```

### Why it fails

`Pool` is documented as a cheap reference-counted handle: cloning it "returns a
new Pool tied to the same shared connection pool". Axum already clones the state
once per request, and cloning a `Pool` is an `Arc`-style refcount bump.
Wrapping it in `Arc` adds a second, redundant layer of indirection and forces
every handler signature to become `State<Arc<PgPool>>` for no benefit.

### Correct

Pass the `PgPool` directly: `.with_state(pool)` and `State(pool): State<PgPool>`.

## 3. Wrapping PgPool in a Mutex

### Wrong

```rust
let app = Router::new().route("/", get(handler))
    .with_state(Arc::new(Mutex::new(pool)));
```

### Why it fails

The pool is internally synchronized and is built to hand out connections to many
tasks concurrently. A `Mutex` around it forces every request to take the same
lock before touching the database, so all database access runs one request at a
time. This serializes the whole service and destroys the concurrency the pool
exists to provide. The symptom is a service that is slow under load even though
the database and the pool are nearly idle.

### Correct

Never lock the pool. Pass the bare `PgPool` as state; concurrent handlers share
it safely.

## 4. Calling unwrap on a query result in a handler

### Wrong

```rust
async fn get_user(State(pool): State<PgPool>, Path(id): Path<i64>) -> Json<User> {
    let user = sqlx::query_as::<_, User>("SELECT id, name FROM users WHERE id = $1")
        .bind(id)
        .fetch_one(&pool)
        .await
        .unwrap(); // panics on a missing row or a dropped connection
    Json(user)
}
```

### Why it fails

`fetch_one` returns `Err` when there are zero rows, and any query can fail if a
connection drops. `.unwrap()` turns that ordinary, expected outcome into a panic.
The panic aborts the request task; the client gets a bare connection reset or a
500 with no useful body, and a single missing row can take down request handling.

### Correct

Return `Result<T, E>` and propagate with `?`. Use `fetch_optional` when zero
rows is a valid outcome, and map it to `404`:

```rust
async fn get_user(
    State(pool): State<PgPool>,
    Path(id): Path<i64>,
) -> Result<Json<User>, (StatusCode, String)> {
    let user = sqlx::query_as::<_, User>(
        "SELECT id, name FROM users WHERE id = $1",
    )
    .bind(id)
    .fetch_optional(&pool)
    .await
    .map_err(internal_error)?;

    user.map(Json)
        .ok_or((StatusCode::NOT_FOUND, "user not found".to_string()))
}
```

For a typed application error, see `axum-errors-handling`.

## 5. Omitting acquire_timeout

### Wrong

```rust
let pool = PgPoolOptions::new()
    .max_connections(5)
    // no acquire_timeout
    .connect(&db_url)
    .await
    .unwrap();
```

### Why it fails

When all `max_connections` connections are checked out, the next `acquire` (or
the next query against `&pool`) waits for a connection to free up. Without
`acquire_timeout` that wait is unbounded: a saturated pool makes every new
request hang forever. The symptom is requests that never return and never error,
while the process looks healthy.

### Correct

Always set `acquire_timeout`. A saturated pool then makes `acquire` fail after
the timeout, and the handler can map that to a fast `503`:

```rust
let pool = PgPoolOptions::new()
    .max_connections(5)
    .acquire_timeout(Duration::from_secs(3))
    .connect(&db_url)
    .await
    .expect("can't connect to database");
```

## 6. Using the wrong DatabaseConnection extractor form per version

### Wrong

```rust
// On axum 0.8: #[async_trait] no longer belongs here.
#[async_trait::async_trait]
impl<S> FromRequestParts<S> for DatabaseConnection { /* ... */ }
```

```rust
// On axum 0.7: a native async fn in the trait without #[async_trait]
// does not compile.
impl<S> FromRequestParts<S> for DatabaseConnection { /* ... */ }
```

### Why it fails

Axum 0.7 defines `FromRequestParts` with `#[async_trait]`, so an impl block must
also carry `#[async_trait]`. Axum 0.8 dropped `#[async_trait]` in favor of native
`async fn` in traits, so the attribute is wrong there. Mixing the forms produces
trait-mismatch compile errors that point at the impl block rather than the real
cause.

### Correct

Match the form to the Axum version. Axum 0.8: native `async fn`, no attribute.
Axum 0.7: identical method body with `#[async_trait]` on the impl block. Both
forms are shown in full in `references/examples.md`.

## 7. Concatenating input into the SQL string

### Wrong

```rust
let sql = format!("SELECT id, name FROM users WHERE name = '{name}'");
sqlx::query_as::<_, User>(&sql).fetch_all(&pool).await
```

### Why it fails

Building the SQL text by string interpolation puts untrusted input directly into
the statement, which is a SQL injection vulnerability. It also bypasses prepared
statement caching.

### Correct

Use a bound parameter so the value travels separately from the SQL text:

```rust
sqlx::query_as::<_, User>("SELECT id, name FROM users WHERE name = $1")
    .bind(name)
    .fetch_all(&pool)
    .await
```

When the SQL is a static literal, prefer the `query_as!` macro so the query is
also checked at compile time.
