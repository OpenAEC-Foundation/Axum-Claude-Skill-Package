# axum-errors-handling: Examples

Complete, copy-ready code. Every example compiles against Axum 0.7 and Axum
0.8: the error-handling APIs are identical across both versions, so no example
here is version-divergent. The `Cargo.toml` lines mark which `axum` line to
pick.

## Example 1: The official error-handling service with AppJson

This is the full `examples/error-handling` shape: an `AppError` enum, a derived
`AppJson<T>` wrapper whose own JSON-extraction rejection routes through
`AppError`, and the `?` operator inside a handler.

```toml
[dependencies]
axum = "0.8"                                  # axum 0.8
# axum = "0.7"                                # axum 0.7
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
```

```rust
use axum::extract::rejection::JsonRejection;
use axum::extract::FromRequest;
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::{routing::post, Router};
use serde::{Deserialize, Serialize};
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let app = Router::new().route("/users", post(create_user));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}

// A JSON wrapper whose extraction rejection is remapped to AppError.
// #[from_request(via(axum::Json), rejection(AppError))] makes a failed
// deserialization produce AppError::JsonRejection instead of the default.
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);

impl<T> IntoResponse for AppJson<T>
where
    axum::Json<T>: IntoResponse,
{
    fn into_response(self) -> Response {
        axum::Json(self.0).into_response()
    }
}

#[derive(Deserialize)]
struct CreateUser {
    name: String,
}

#[derive(Serialize)]
struct User {
    id: u64,
    name: String,
}

// The handler takes AppJson<CreateUser>, so a malformed body becomes
// AppError::JsonRejection automatically. The handler returns Result<_, AppError>.
async fn create_user(
    AppJson(payload): AppJson<CreateUser>,
) -> Result<AppJson<User>, AppError> {
    let id = next_id()?; // IdError -> AppError via the ? operator
    Ok(AppJson(User {
        id,
        name: payload.name,
    }))
}

// A stand-in for a fallible third-party call.
#[derive(Debug)]
struct IdError;
impl std::fmt::Display for IdError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "id generation failed")
    }
}
impl std::error::Error for IdError {}

fn next_id() -> Result<u64, IdError> {
    Ok(42)
}

#[derive(Debug)]
enum AppError {
    // Invalid JSON in the request body.
    JsonRejection(JsonRejection),
    // A failure from a third-party library.
    IdError(IdError),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        #[derive(Serialize)]
        struct ErrorResponse {
            message: String,
        }

        let (status, message, err) = match &self {
            AppError::JsonRejection(rejection) => {
                (rejection.status(), rejection.body_text(), None)
            }
            AppError::IdError(_err) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "Something went wrong".to_owned(),
                Some(self),
            ),
        };

        let mut response =
            (status, AppJson(ErrorResponse { message })).into_response();
        if let Some(err) = err {
            // A logging middleware reads this; the client never sees it.
            response.extensions_mut().insert(Arc::new(err));
        }
        response
    }
}

impl From<JsonRejection> for AppError {
    fn from(rejection: JsonRejection) -> Self {
        Self::JsonRejection(rejection)
    }
}

impl From<IdError> for AppError {
    fn from(error: IdError) -> Self {
        Self::IdError(error)
    }
}
```

## Example 2: The anyhow newtype service

The full `examples/anyhow-error-response` shape. Every error collapses to a
single 500. The blanket `From` covers every error type, so `?` works on any
`anyhow`-compatible error.

```toml
[dependencies]
axum = "0.8"                                  # axum 0.8
# axum = "0.7"                                # axum 0.7
anyhow = "1"
tokio = { version = "1", features = ["full"] }
```

```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(handler));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Result<(), AppError> {
    try_thing()?;
    Ok(())
}

fn try_thing() -> Result<(), anyhow::Error> {
    anyhow::bail!("it failed!")
}

// A local newtype around anyhow::Error: the orphan-rule workaround.
struct AppError(anyhow::Error);

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        // In production return a generic body and log self.0 instead of
        // formatting the error into the response.
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        )
            .into_response()
    }
}

// Enables ? on any function returning Result<_, E> where E: Into<anyhow::Error>.
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
```

## Example 3: Combined thiserror enum with an anyhow catch-all

The production shape. Known errors keep their own status code; everything else
funnels into `Unexpected` and becomes a 500. This example also shows the
client-safe body rule: the `Unexpected` variant's text is logged, not returned.

```toml
[dependencies]
axum = "0.8"                                  # axum 0.8
# axum = "0.7"                                # axum 0.7
thiserror = "2"
anyhow = "1"
tokio = { version = "1", features = ["full"] }
tracing = "0.1"
```

```rust
use axum::extract::Path;
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/users/{id}", get(get_user));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}

#[derive(thiserror::Error, Debug)]
enum AppError {
    #[error("entity not found")]
    NotFound,

    #[error("invalid input: {0}")]
    Validation(String),

    // transparent: Display and source delegate to the inner anyhow::Error.
    // from: generates From<anyhow::Error> so ? funnels here.
    #[error(transparent)]
    Unexpected(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, body) = match &self {
            AppError::NotFound => {
                (StatusCode::NOT_FOUND, self.to_string())
            }
            AppError::Validation(_) => {
                (StatusCode::UNPROCESSABLE_ENTITY, self.to_string())
            }
            AppError::Unexpected(err) => {
                // Log the real cause; never serialize it to the client.
                tracing::error!(error = %err, "unhandled error");
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "internal server error".to_owned(),
                )
            }
        };
        (status, body).into_response()
    }
}

async fn get_user(Path(id): Path<i64>) -> Result<String, AppError> {
    if id < 0 {
        return Err(AppError::Validation("id must be non-negative".into()));
    }
    let user = lookup_user(id)?; // any anyhow error -> AppError::Unexpected
    user.ok_or(AppError::NotFound)
}

// A stand-in repository call returning an anyhow::Result.
fn lookup_user(id: i64) -> anyhow::Result<Option<String>> {
    Ok((id == 1).then(|| "Ada".to_owned()))
}
```

## Example 4: HandleErrorLayer wrapping a fallible timeout layer

A `TimeoutLayer` produces a fallible service: a timed-out request yields an
error, not a response. `Router::layer` rejects it. `HandleErrorLayer` adapts
it. The error handler reads `Method` and `Uri`, with the service error last.

```toml
[dependencies]
axum = "0.8"                                  # axum 0.8
# axum = "0.7"                                # axum 0.7
tower = { version = "0.5", features = ["timeout"] }
tokio = { version = "1", features = ["full"] }
```

```rust
use axum::error_handling::HandleErrorLayer;
use axum::http::{Method, StatusCode, Uri};
use axum::{routing::get, Router};
use std::time::Duration;
use tower::timeout::TimeoutLayer;
use tower::{BoxError, ServiceBuilder};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/slow", get(slow_handler)).layer(
        ServiceBuilder::new()
            // ServiceBuilder applies top-to-bottom: HandleErrorLayer must sit
            // ABOVE TimeoutLayer so it catches the timeout error.
            .layer(HandleErrorLayer::new(handle_timeout_error))
            .layer(TimeoutLayer::new(Duration::from_secs(5))),
    );

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn slow_handler() -> &'static str {
    tokio::time::sleep(Duration::from_secs(10)).await;
    "done"
}

// Extractors first, the service error (BoxError) last. The output implements
// IntoResponse, so the adapted service is infallible.
async fn handle_timeout_error(
    method: Method,
    uri: Uri,
    err: BoxError,
) -> (StatusCode, String) {
    (
        StatusCode::REQUEST_TIMEOUT,
        format!("`{method} {uri}` failed: {err}"),
    )
}
```

## Example 5: A no-extractor error handler

When the error handler does not need request data, it takes only the error.
Verified shape from the `error_handling` module docs:

```rust
use axum::http::StatusCode;

async fn handle_anyhow_error(err: anyhow::Error) -> (StatusCode, String) {
    (
        StatusCode::INTERNAL_SERVER_ERROR,
        format!("Something went wrong: {err}"),
    )
}
```

Pass it to `HandleError::new(inner_service, handle_anyhow_error)` to adapt a
single fallible service, or to `HandleErrorLayer::new(handle_anyhow_error)` to
adapt a layer stack.
