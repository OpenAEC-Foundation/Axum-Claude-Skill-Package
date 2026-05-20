# axum-impl-database: Working Examples

Every example is verified against the `tokio-rs/axum` `examples/sqlx-postgres`
crate and `https://docs.rs/sqlx/latest/sqlx/` (2026-05-20). Postgres is shown;
substitute `MySql` or `Sqlite` for the other databases.

## Example 1: Full main.rs with the pool in state

```rust
use axum::{routing::get, Router};
use sqlx::postgres::{PgPool, PgPoolOptions};
use std::time::Duration;

#[tokio::main]
async fn main() {
    // NEVER a string literal. Fail fast when DATABASE_URL is absent.
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
        .with_state(pool); // PgPool directly: no Arc, no Mutex

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    tokio::signal::ctrl_c()
        .await
        .expect("failed to install ctrl_c handler");
}
```

When the application owns the pool and needs to close it explicitly, keep a
clone and call `pool.close().await` after `axum::serve` returns.

## Example 2: Query against the pool reference

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
        .map_err(internal_error)?;
    Ok(format!("{count} users"))
}

// Shared 500-mapping helper, verified from examples/sqlx-postgres.
fn internal_error<E>(err: E) -> (StatusCode, String)
where
    E: std::error::Error,
{
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}
```

## Example 3: query_as CRUD with FromRow

```rust
use axum::extract::{Path, State};
use axum::http::StatusCode;
use axum::Json;
use sqlx::postgres::PgPool;

#[derive(serde::Serialize, sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
}

#[derive(serde::Deserialize)]
struct NewUser {
    name: String,
}

// Read one row: fetch_optional distinguishes "not found" from an error.
async fn get_user(
    State(pool): State<PgPool>,
    Path(id): Path<i64>,
) -> Result<Json<User>, (StatusCode, String)> {
    let user = sqlx::query_as::<_, User>(
        "SELECT id, name FROM users WHERE id = $1",
    )
    .bind(id) // bound parameter: never concatenate input into SQL
    .fetch_optional(&pool)
    .await
    .map_err(internal_error)?;

    match user {
        Some(user) => Ok(Json(user)),
        None => Err((StatusCode::NOT_FOUND, "user not found".to_string())),
    }
}

// Insert one row and return the generated id.
async fn create_user(
    State(pool): State<PgPool>,
    Json(body): Json<NewUser>,
) -> Result<Json<User>, (StatusCode, String)> {
    let user = sqlx::query_as::<_, User>(
        "INSERT INTO users (name) VALUES ($1) RETURNING id, name",
    )
    .bind(body.name)
    .fetch_one(&pool)
    .await
    .map_err(internal_error)?;

    Ok(Json(user))
}

// Delete: execute returns the affected-row count.
async fn delete_user(
    State(pool): State<PgPool>,
    Path(id): Path<i64>,
) -> Result<StatusCode, (StatusCode, String)> {
    let result = sqlx::query("DELETE FROM users WHERE id = $1")
        .bind(id)
        .execute(&pool)
        .await
        .map_err(internal_error)?;

    if result.rows_affected() == 0 {
        return Err((StatusCode::NOT_FOUND, "user not found".to_string()));
    }
    Ok(StatusCode::NO_CONTENT)
}
```

The compile-time-checked form of the read query, used when the SQL is a static
literal:

```rust
let user = sqlx::query_as!(
    User,
    "SELECT id, name FROM users WHERE id = $1",
    id,
)
.fetch_optional(&pool)
.await
.map_err(internal_error)?;
```

## Example 4: The complete DatabaseConnection extractor

Use this when several statements in one handler must share a single physical
connection. The impl block differs by Axum version.

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
// Same struct and same method body; the impl block carries #[async_trait].
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

The handler destructures the extractor and uses the connection as `&mut *conn`:

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

## Example 5: A transaction handler

```rust
use axum::extract::State;
use axum::http::StatusCode;
use sqlx::postgres::PgPool;

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

    // If either statement fails, `?` returns early. `tx` is dropped here and
    // the transaction rolls back automatically: no half-applied transfer.
    tx.commit().await.map_err(internal_error)?;
    Ok(StatusCode::OK)
}
```

For a non-waiting attempt, `pool.try_begin()` returns
`Result<Option<Transaction>, Error>`: `Ok(None)` means a connection could not be
acquired without waiting.
