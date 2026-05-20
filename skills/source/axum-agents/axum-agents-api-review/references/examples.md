# axum-agents-api-review: Worked Examples

Two complete reviews: one of a file with several defects, one of clean code.
Both show the exact structured-findings format the agent MUST emit.

## Example 1: A file with multiple defects

### Input

`Cargo.toml` declares `axum = "0.8"`, so the target version is 0.8.

`src/main.rs`:

```rust
use axum::{routing::get, routing::post, Router, Json, Extension};
use axum::extract::Path;
use serde::Deserialize;
use std::sync::Arc;
use tower_http::cors::CorsLayer;

const JWT_SECRET: &str = "super-secret-key-123";

#[derive(Clone)]
struct AppConfig {
    db_url: String,
}

#[derive(Deserialize)]
struct NewUser {
    name: String,
}

#[tokio::main]
async fn main() {
    let config = AppConfig {
        db_url: "postgres://localhost/app".to_string(),
    };

    let app = Router::new()
        .route("/users/:id", get(get_user))
        .route("/users", post(create_user))
        .layer(CorsLayer::permissive())
        .layer(Extension(Arc::new(config)));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn get_user(Path(id): Path<i64>) -> String {
    std::thread::sleep(std::time::Duration::from_millis(200));
    format!("user {id}")
}

async fn create_user(
    Json(body): Json<NewUser>,
    Extension(config): Extension<Arc<AppConfig>>,
) -> String {
    let pool = sqlx::PgPool::connect(&config.db_url).await.unwrap();
    let _ = sqlx::query("insert into users (name) values ($1)")
        .bind(&body.name)
        .execute(&pool)
        .await;
    format!("created {}", body.name)
}
```

### Output

```
Axum API Review : 9 findings
  blockers : 4   warnings : 4   suggestions : 1
  Target Axum version : 0.8

[BLOCKER] 0.7 path syntax on Axum 0.8
  Location : src/main.rs:27
  Symptom  : route "/users/:id" uses 0.7 syntax; on 0.8 the router panics
             at construction. Axum 0.8 requires "/users/{id}".
  Anti-pattern : AP-2
  Fix skill : axum-core-version-migration
  Action   : convert :id to {id}; see the path-syntax migration check.

[BLOCKER] Body extractor not in final position
  Location : src/main.rs:41
  Symptom  : create_user takes Json<NewUser> before Extension<...>. A body
             extractor must be the last argument; the Handler trait bound
             fails to compile.
  Anti-pattern : AP-1
  Fix skill : axum-syntax-extractors
  Action   : move Json<NewUser> to the last parameter position.

[BLOCKER] Hardcoded JWT secret in source
  Location : src/main.rs:7
  Symptom  : JWT_SECRET is a string literal, baked into the binary and the
             git history; anyone with the binary can forge tokens.
  Anti-pattern : AP-15
  Fix skill : axum-impl-auth-jwt
  Action   : read the secret from an environment variable at startup.

[BLOCKER] unwrap on a fallible value in a handler
  Location : src/main.rs:46
  Symptom  : PgPool::connect(...).unwrap() panics on any connection failure,
             crashing the request task instead of returning a response.
  Anti-pattern : AP-13
  Fix skill : axum-errors-handling
  Action   : return Result<_, AppError> and propagate the error with ?.

[WARNING] CorsLayer::permissive() shipped to production
  Location : src/main.rs:30
  Symptom  : permissive() allows every origin, method, and header,
             disabling cross-origin protection.
  Anti-pattern : AP-9
  Fix skill : axum-impl-tower-stack
  Action   : replace with an explicit origin allowlist.

[WARNING] Blocking call in an async handler
  Location : src/main.rs:37
  Symptom  : std::thread::sleep blocks the Tokio worker thread, adding
             latency to every other request on that worker.
  Anti-pattern : AP-10
  Fix skill : axum-core-async-performance
  Action   : use tokio::time::sleep, or spawn_blocking for CPU-bound work.

[WARNING] New database pool created per request
  Location : src/main.rs:46
  Symptom  : PgPool::connect runs inside create_user, opening a fresh pool
             on every request and exhausting the server connection limit.
  Anti-pattern : AP-12
  Fix skill : axum-impl-database
  Action   : build one pool in main and share it via with_state.

[WARNING] Missing graceful shutdown
  Location : src/main.rs:34
  Symptom  : axum::serve has no .with_graceful_shutdown(); a SIGTERM on
             deploy kills in-flight requests.
  Anti-pattern : AP-14
  Fix skill : axum-impl-deployment
  Action   : add .with_graceful_shutdown() with a SIGTERM-aware signal.

[SUGGESTION] Extension used where State would be compile-checked
  Location : src/main.rs:31
  Symptom  : AppConfig is passed via Extension; a missing extension is a
             runtime 500 with no compile-time check.
  Anti-pattern : AP-5
  Fix skill : axum-core-state
  Action   : move AppConfig into router state and extract it with State.
```

Note the ordering: all 4 blockers first, then all 4 warnings, then the single
suggestion. Within each severity, findings are ordered by file then by line.

## Example 2: A clean review

### Input

`Cargo.toml` declares `axum = "0.8"`. `src/main.rs` builds the pool once in
`main`, shares it via `with_state`, uses `{id}` path syntax, declares every
handler `async` with an `IntoResponse` return, returns `Result<_, AppError>`
with no `.unwrap()`, reads secrets from the environment, and wires
`.with_graceful_shutdown()` with a SIGTERM-aware signal.

### Output

```
Axum API Review : 0 findings
  blockers : 0   warnings : 0   suggestions : 0
  Target Axum version : 0.8

No issues found.
```

An empty result is NEVER silent. The summary header with all counts at 0 and
the explicit `No issues found.` line make a clean review unambiguous.

## Example 3: An unknown-version review

When no `Cargo.toml` is provided, or it pins no concrete `axum` version, the
agent reports the version as `unknown` and still runs every check. A
version-sensitive finding (path syntax, `Option<Path<T>>` behavior, the
`#[async_trait]` form of a custom extractor) is reported as a warning with the
version caveat stated in its `Symptom` line.

```
Axum API Review : 1 findings
  blockers : 0   warnings : 1   suggestions : 0
  Target Axum version : unknown

[WARNING] Path syntax cannot be validated without a known Axum version
  Location : src/routes.rs:12
  Symptom  : route "/items/:id" uses 0.7 syntax. On 0.8 this panics at
             construction; on 0.7 it is correct. The target version is
             unknown, so correctness cannot be confirmed.
  Anti-pattern : AP-2
  Fix skill : axum-core-version-migration
  Action   : pin the axum version in Cargo.toml, then confirm the syntax.
```
