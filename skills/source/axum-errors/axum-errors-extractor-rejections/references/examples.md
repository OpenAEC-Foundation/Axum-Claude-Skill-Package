# Examples: axum-errors-extractor-rejections

Every example is verified against the official `tokio-rs/axum`
`examples/customize-extractor-error` crate and docs.rs (fetched 2026-05-20).
Each block is annotated with the Axum versions it applies to.

## Example 1: the extractor trait definitions

The `Rejection: IntoResponse` bound is the contract every custom extractor
must satisfy.

```rust
// axum 0.8 - verbatim from docs.rs
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;
    fn from_request_parts(parts: &mut Parts, state: &S)
        -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}

pub trait FromRequest<S, M = ViaRequest>: Sized {
    type Rejection: IntoResponse;
    fn from_request(req: Request<Body>, state: &S)
        -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

```rust
// axum 0.7 - same methods, declared under #[async_trait]
#[async_trait]
pub trait FromRequest<S, M = ViaRequest>: Sized {
    type Rejection: IntoResponse;
    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection>;
}
```

## Example 2: Result<T, T::Rejection> with the mandatory catch-all arm

Every rejection enum is `#[non_exhaustive]`, so a `match` without a `_ =>` arm
fails to compile. This handler needs no custom extractor and no extra crate.

```rust
// axum 0.7 / 0.8 - identical
use axum::{extract::rejection::JsonRejection, Json, http::StatusCode};
use axum::response::{IntoResponse, Response};
use serde_json::Value;

async fn create(payload: Result<Json<Value>, JsonRejection>) -> Response {
    match payload {
        Ok(Json(value)) => Json(value).into_response(),

        Err(JsonRejection::MissingJsonContentType(_)) =>
            (StatusCode::UNSUPPORTED_MEDIA_TYPE, "expected application/json")
                .into_response(),
        Err(JsonRejection::JsonSyntaxError(_)) =>
            (StatusCode::BAD_REQUEST, "invalid JSON syntax").into_response(),
        Err(JsonRejection::JsonDataError(_)) =>
            (StatusCode::UNPROCESSABLE_ENTITY, "JSON does not match schema")
                .into_response(),

        // #[non_exhaustive] makes this arm mandatory. It also preserves
        // Axum's own status and message for any future variant.
        Err(rejection) =>
            (rejection.status(), rejection.body_text()).into_response(),
    }
}
```

## Example 3: hand-written custom extractor (custom_extractor.rs)

Verified structure from the official `customize-extractor-error` example. Use
this only when extraction must read other request parts, here `MatchedPath`.

```rust
// axum 0.7 / 0.8 - identical
use axum::extract::{FromRequest, MatchedPath, Request, rejection::JsonRejection};
use axum::http::StatusCode;
use serde::de::DeserializeOwned;
use serde_json::{json, Value};

pub struct Json<T>(pub T);

impl<S, T> FromRequest<S> for Json<T>
where
    axum::Json<T>: FromRequest<S, Rejection = JsonRejection>,
    T: DeserializeOwned,
    S: Send + Sync,
{
    // the Rejection type MUST implement IntoResponse; this tuple does
    type Rejection = (StatusCode, axum::Json<Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        // read another request part before delegating
        let path = req.extensions().get::<MatchedPath>()
            .map(|p| p.as_str().to_owned());

        match axum::Json::<T>::from_request(req, state).await {
            Ok(axum::Json(value)) => Ok(Self(value)),
            Err(rejection) => {
                let payload = json!({
                    "message": rejection.body_text(),
                    "origin": "custom_extractor",
                    "path": path,
                });
                Err((rejection.status(), axum::Json(payload)))
            }
        }
    }
}
```

This is the most flexible option and the most verbose. It can read headers,
`MatchedPath`, or extensions while building the error.

## Example 4: WithRejection<E, R> with a thiserror enum (with_rejection.rs)

Verified shape from the official example. `WithRejection` requires the
`with-rejection` feature on `axum-extra`. `R` must implement `IntoResponse`
and `From<E::Rejection>`.

```rust
// axum 0.7 / 0.8 - identical, requires axum-extra "with-rejection" feature
use axum::{extract::rejection::JsonRejection, Json};
use axum::response::{IntoResponse, Response};
use axum_extra::extract::WithRejection;
use serde_json::{json, Value};
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ApiError {
    // #[from] generates the required From<JsonRejection> for ApiError
    #[error(transparent)]
    JsonExtractorRejection(#[from] JsonRejection),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            ApiError::JsonExtractorRejection(json_rejection) =>
                (json_rejection.status(), json_rejection.body_text()),
        };
        let payload = json!({ "message": message, "origin": "with_rejection" });
        (status, Json(payload)).into_response()
    }
}

async fn handler(
    // the second tuple field is PhantomData<ApiError>, discarded with _
    WithRejection(Json(value), _): WithRejection<Json<Value>, ApiError>,
) {
    let _ = value; // the successfully extracted Value
}
```

## Example 5: derive(FromRequest) with rejection(...) (derive_from_request.rs)

Verified shape from the official example. Requires the `macros` feature on
`axum`. No manual `FromRequest` impl is written.

```rust
// axum 0.7 / 0.8 - identical, requires axum "macros" feature
use axum::extract::{FromRequest, rejection::JsonRejection};
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use serde_json::json;

#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct Json<T>(pub T);

// here ApiError is a plain struct
pub struct ApiError {
    status: StatusCode,
    message: String,
}

// rejection(ApiError) requires From<JsonRejection> for ApiError
impl From<JsonRejection> for ApiError {
    fn from(rejection: JsonRejection) -> Self {
        ApiError {
            status: rejection.status(),
            message: rejection.body_text(),
        }
    }
}

// rejection(ApiError) also requires IntoResponse for ApiError
impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let payload = json!({
            "message": self.message,
            "origin": "derive_from_request",
        });
        (self.status, axum::Json(payload)).into_response()
    }
}
```

## Example 6: Option<T> extractor, 0.7 swallow-all vs 0.8 reject-on-malformed

The same handler signature behaves differently across versions.

```rust
// axum 0.7 - Option<Path<T>> swallows EVERY failure, including malformed input
use axum::extract::Path;

async fn handler(id: Option<Path<u32>>) -> String {
    match id {
        Some(Path(id)) => format!("id is {id}"),
        // reached for a MISSING path param AND for a non-numeric one such as /abc
        None => "no id".to_owned(),
    }
}
```

```rust
// axum 0.8 - Option<Path<T>> rejects malformed input, yields None only when absent
use axum::extract::Path;

async fn handler(id: Option<Path<u32>>) -> String {
    match id {
        Some(Path(id)) => format!("id is {id}"),
        // reached ONLY when the param is genuinely absent.
        // a present-but-non-numeric param now returns a 4xx rejection instead.
        None => "no id".to_owned(),
    }
}
```

To keep the Axum 0.7 "treat any failure as absent" behavior on 0.8, replace
`Option<T>` with `Result<T, T::Rejection>` and map the error explicitly:

```rust
// axum 0.8 - explicit recovery of the old swallow-all behavior
use axum::extract::{Path, rejection::PathRejection};

async fn handler(id: Result<Path<u32>, PathRejection>) -> String {
    match id {
        Ok(Path(id)) => format!("id is {id}"),
        Err(_)       => "no id".to_owned(), // any failure is treated as absent
    }
}
```

## Cargo features used in these examples

```toml
# axum 0.8
axum = { version = "0.8", features = ["macros"] }       # for #[derive(FromRequest)]
axum-extra = { version = "0.10", features = ["with-rejection"] } # for WithRejection
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
```

```toml
# axum 0.7
axum = { version = "0.7", features = ["macros"] }
axum-extra = { version = "0.9", features = ["with-rejection"] }
```

## Sources verified (2026-05-20)

- https://github.com/tokio-rs/axum/blob/main/examples/customize-extractor-error/src/main.rs
- https://raw.githubusercontent.com/tokio-rs/axum/main/examples/customize-extractor-error/src/custom_extractor.rs
- https://raw.githubusercontent.com/tokio-rs/axum/main/examples/customize-extractor-error/src/with_rejection.rs
- https://raw.githubusercontent.com/tokio-rs/axum/main/examples/customize-extractor-error/src/derive_from_request.rs
- https://docs.rs/axum/latest/axum/extract/rejection/enum.JsonRejection.html
- https://docs.rs/axum-extra/latest/axum_extra/extract/struct.WithRejection.html
- https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0
