# axum-errors-handling: Anti-Patterns

Each entry is a real mistake, why it fails, and the corrected form. Verified
2026-05-20 against `https://docs.rs/axum/latest/axum/error_handling/index.html`
and the `tokio-rs/axum` `examples/error-handling` and
`examples/anyhow-error-response` crates.

## 1. Calling unwrap, expect, or panic to report an error

### Wrong

```rust
async fn get_user(id: i64) -> Json<User> {
    let user = lookup_user(id).await.unwrap(); // panics on Err
    Json(user)
}
```

### Why it fails

`lookup_user` returns `Err` on a missing row or a dropped connection. Those are
ordinary, expected outcomes. `.unwrap()` turns them into a panic. A panic
aborts the request task: the client receives a dropped connection or an opaque
500 with no useful body, and the failure does not surface as a clean status
code. A handler must never use a panic as its error channel, because Axum's
whole model is that a handler produces a `Response`, it does not "fail".

### Correct

Return `Result<T, E>` and convert the error into a response:

```rust
async fn get_user(id: i64) -> Result<Json<User>, AppError> {
    let user = lookup_user(id).await?; // Err -> AppError -> Response
    Ok(Json(user))
}
```

`expect` is acceptable ONLY for startup invariants in `main` (binding the
listener, reading a mandatory env var), where refusing to start is correct.

## 2. impl IntoResponse for anyhow::Error directly

### Wrong

```rust
impl IntoResponse for anyhow::Error {
    fn into_response(self) -> Response {
        (StatusCode::INTERNAL_SERVER_ERROR, self.to_string()).into_response()
    }
}
```

### Why it fails

This does not compile. `anyhow::Error` is a foreign type (defined in the
`anyhow` crate) and `IntoResponse` is a foreign trait (defined in `axum`).
Rust's orphan rule forbids implementing a foreign trait for a foreign type:
neither is local to the current crate. The compiler reports "only traits
defined in the current crate can be implemented for types defined outside of
the crate".

### Correct

Wrap `anyhow::Error` in a local newtype and implement `IntoResponse` for the
newtype, which IS local:

```rust
struct AppError(anyhow::Error);

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (StatusCode::INTERNAL_SERVER_ERROR, "internal server error")
            .into_response()
    }
}
```

See SKILL.md Pattern 4 for the full newtype, including the blanket `From`.

## 3. Leaking internal error text to the client

### Wrong

```rust
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        // The same detailed message goes to every client, for every variant.
        (StatusCode::INTERNAL_SERVER_ERROR, self.to_string()).into_response()
    }
}
```

For a `Database(sqlx::Error)` variant, `self.to_string()` can expose table
names, column names, SQL fragments, and connection-string details.

### Why it fails

Internal error text is an information-disclosure vulnerability. Database driver
messages, file paths, and library internals help an attacker map the system and
reveal nothing useful to a legitimate client. The official `error-handling`
example deliberately returns `"Something went wrong"` for its internal variant
and stashes the real error in the response extensions for a logging layer.

### Correct

Return a generic body for internal variants; log the detail:

```rust
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        match self {
            AppError::NotFound => {
                (StatusCode::NOT_FOUND, "not found").into_response()
            }
            AppError::Database(err) => {
                tracing::error!(error = %err, "database failure");
                (StatusCode::INTERNAL_SERVER_ERROR, "internal server error")
                    .into_response()
            }
        }
    }
}
```

Client-safe variants (a validation message) MAY return their own text.

## 4. A missing From impl breaks the ? operator

### Wrong

```rust
#[derive(Debug)]
enum AppError {
    NotFound,
    Database(sqlx::Error),
}

// No From<sqlx::Error> for AppError.

async fn handler() -> Result<String, AppError> {
    let row = run_query().await?; // run_query returns Result<_, sqlx::Error>
    Ok(row)
}
```

### Why it fails

`?` desugars to `return Err(AppError::from(source_error))`. Without a
`From<sqlx::Error> for AppError` impl, that conversion does not exist. The
compiler reports "the trait bound `AppError: From<sqlx::Error>` is not
satisfied" and points at the `?`. The enum having a `Database(sqlx::Error)`
variant is not enough; the `From` impl that constructs it is also required.

### Correct

Add the `From` impl, or let `thiserror`'s `#[from]` generate it:

```rust
#[derive(thiserror::Error, Debug)]
enum AppError {
    #[error("entity not found")]
    NotFound,
    #[error("database error")]
    Database(#[from] sqlx::Error), // #[from] generates From<sqlx::Error>
}
```

## 5. Handing a fallible service to Router::layer without HandleError

### Wrong

```rust
let app = Router::new()
    .route("/", get(handler))
    .layer(TimeoutLayer::new(Duration::from_secs(10))); // fallible layer
```

### Why it fails

`TimeoutLayer` produces a service whose `Error` type is not `Infallible`: a
timed-out request yields an error. `Router::layer` requires the layered service
to have `Error = Infallible`, because every routable Axum service must produce
a `Response` and never "fail". The code does not compile; the error points at a
`Service` / `Error` trait bound mismatch on the `.layer(...)` call.

### Correct

Wrap the fallible layer with `HandleErrorLayer`, which converts the error into
a response and so makes the resulting service infallible:

```rust
let app = Router::new().route("/", get(handler)).layer(
    ServiceBuilder::new()
        .layer(HandleErrorLayer::new(handle_timeout_error))
        .layer(TimeoutLayer::new(Duration::from_secs(10))),
);
```

## 6. Placing HandleErrorLayer below the layer it protects

### Wrong

```rust
let app = Router::new().route("/", get(handler)).layer(
    ServiceBuilder::new()
        .layer(TimeoutLayer::new(Duration::from_secs(10)))
        .layer(HandleErrorLayer::new(handle_timeout_error)),
);
```

### Why it fails

`ServiceBuilder` applies layers top-to-bottom: the first `.layer(...)` is the
outermost. Here `TimeoutLayer` is outer and `HandleErrorLayer` is inner. The
error handler is wrapped inside the very layer whose error it must catch, so
the fallible error escapes past the adapter and the `Router` still sees a
non-`Infallible` service. The code does not compile.

### Correct

`HandleErrorLayer` must be the OUTER layer, above the fallible layer it
protects:

```rust
ServiceBuilder::new()
    .layer(HandleErrorLayer::new(handle_timeout_error)) // outer: catches
    .layer(TimeoutLayer::new(Duration::from_secs(10)))  // inner: fallible
```

## 7. Using HandleError for a normal handler error

### Wrong

```rust
// Trying to route an ordinary handler's Result error through HandleError.
let svc = HandleError::new(my_handler_service, handle_error);
```

### Why it fails

`HandleError` / `HandleErrorLayer` exist ONLY for a fallible `tower::Service`
used as middleware: a layer whose `Error` type is genuinely not `Infallible`.
A normal handler is already infallible at the `Service` level. Its error path
is the `Result<T, E>` return type plus `IntoResponse`. Reaching for
`HandleError` here adds a layer that solves a problem the handler does not
have, and obscures the real error model.

### Correct

A handler reports errors through its return type. Give the error type an
`IntoResponse` impl and return `Result<T, E>`:

```rust
async fn handler() -> Result<Json<User>, AppError> {
    let user = lookup().await?;
    Ok(Json(user))
}
```

Reserve `HandleError` / `HandleErrorLayer` for fallible tower layers
(`TimeoutLayer`, load-shed, a hand-written fallible `Service`).

## 8. The error argument not last in a HandleError handler

### Wrong

```rust
// The service error is placed before the extractors.
async fn handle_error(
    err: BoxError,
    method: Method,
    uri: Uri,
) -> (StatusCode, String) {
    (StatusCode::REQUEST_TIMEOUT, format!("{method} {uri}: {err}"))
}
```

### Why it fails

`HandleError`'s function-arity rules require the service error to be the FINAL
argument; the leading arguments are extractors implementing
`FromRequestParts<()>`. With the error first, the argument list does not match
any accepted `FnOnce` shape, and the `HandleErrorLayer::new(handle_error)` call
fails to compile with a trait-bound error on the handler function.

### Correct

Extractors first, the service error last:

```rust
async fn handle_error(
    method: Method,
    uri: Uri,
    err: BoxError,
) -> (StatusCode, String) {
    (StatusCode::REQUEST_TIMEOUT, format!("{method} {uri}: {err}"))
}
```
