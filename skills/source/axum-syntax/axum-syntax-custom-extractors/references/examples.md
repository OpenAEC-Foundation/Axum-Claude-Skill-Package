# Examples : axum-syntax-custom-extractors

Complete, version-annotated custom extractors. Every snippet is built from
signatures WebFetch-verified against docs.rs on 2026-05-20. Code that differs
between Axum 0.7 and 0.8 is annotated `// axum 0.7` / `// axum 0.8`.

## Example 1 : a FromRequestParts extractor, both versions

A header-reading extractor. Any handler that takes `ExtractUserAgent` as an
argument rejects the request before the body runs when the header is absent.

```rust
// axum 0.7 - custom parts extractor: #[async_trait] REQUIRED
use async_trait::async_trait;
use axum::extract::FromRequestParts;
use axum::http::{header::USER_AGENT, request::Parts, HeaderValue, StatusCode};

pub struct ExtractUserAgent(pub HeaderValue);

#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S)
        -> Result<Self, Self::Rejection>
    {
        parts
            .headers
            .get(USER_AGENT)
            .map(|v| ExtractUserAgent(v.clone()))
            .ok_or((StatusCode::BAD_REQUEST, "missing user-agent header"))
    }
}
```

```rust
// axum 0.8 - SAME extractor: NO #[async_trait], native async fn in trait
use axum::extract::FromRequestParts;
use axum::http::{header::USER_AGENT, request::Parts, HeaderValue, StatusCode};

pub struct ExtractUserAgent(pub HeaderValue);

impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S)
        -> Result<Self, Self::Rejection>
    {
        parts
            .headers
            .get(USER_AGENT)
            .map(|v| ExtractUserAgent(v.clone()))
            .ok_or((StatusCode::BAD_REQUEST, "missing user-agent header"))
    }
}
```

Using it in a handler (identical in both versions):

```rust
// axum 0.7 / 0.8
use axum::{routing::get, Router};

async fn show(ExtractUserAgent(ua): ExtractUserAgent) -> String {
    format!("your user-agent is {ua:?}")
}

fn app() -> Router {
    Router::new().route("/", get(show))
}
```

## Example 2 : an extractor with a custom IntoResponse rejection

When `(StatusCode, &'static str)` is too coarse, define a rejection type and
implement `IntoResponse` on it. The rejection still satisfies the trait bound.

```rust
// axum 0.8 - dedicated rejection type
use axum::extract::FromRequestParts;
use axum::http::{request::Parts, StatusCode};
use axum::response::{IntoResponse, Response};
use axum::Json;
use serde_json::json;

pub struct ApiKey(pub String);

pub enum ApiKeyRejection {
    Missing,
    Malformed,
}

impl IntoResponse for ApiKeyRejection {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            ApiKeyRejection::Missing => (StatusCode::UNAUTHORIZED, "missing x-api-key"),
            ApiKeyRejection::Malformed => (StatusCode::BAD_REQUEST, "malformed x-api-key"),
        };
        (status, Json(json!({ "error": msg }))).into_response()
    }
}

impl<S: Send + Sync> FromRequestParts<S> for ApiKey {
    type Rejection = ApiKeyRejection;

    async fn from_request_parts(parts: &mut Parts, _state: &S)
        -> Result<Self, Self::Rejection>
    {
        let raw = parts
            .headers
            .get("x-api-key")
            .ok_or(ApiKeyRejection::Missing)?;
        let text = raw.to_str().map_err(|_| ApiKeyRejection::Malformed)?;
        Ok(ApiKey(text.to_owned()))
    }
}
```

On Axum 0.7 the only change is `#[async_trait]` above the `impl` block.

## Example 3 : a body extractor wrapping Json

`ValidatedJson<T>` consumes the body through `Json`, runs `validator`
validation, and remaps both failure modes into one rejection.

```rust
// axum 0.8 - 0.7 needs #[async_trait] on the impl, otherwise identical
use axum::extract::{rejection::JsonRejection, FromRequest, Request};
use axum::http::StatusCode;
use axum::Json;
use serde_json::json;

pub struct ValidatedJson<T>(pub T);

impl<S, T> FromRequest<S> for ValidatedJson<T>
where
    S: Send + Sync,
    T: serde::de::DeserializeOwned + validator::Validate,
    Json<T>: FromRequest<S, Rejection = JsonRejection>,
{
    type Rejection = (StatusCode, Json<serde_json::Value>);

    async fn from_request(req: Request, state: &S)
        -> Result<Self, Self::Rejection>
    {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|rej| {
                (rej.status(), Json(json!({ "error": rej.body_text() })))
            })?;

        value.validate().map_err(|e| {
            (
                StatusCode::UNPROCESSABLE_ENTITY,
                Json(json!({ "error": e.to_string() })),
            )
        })?;

        Ok(ValidatedJson(value))
    }
}
```

Using it as the LAST handler argument:

```rust
// axum 0.7 / 0.8
use validator::Validate;

#[derive(serde::Deserialize, Validate)]
struct NewUser {
    #[validate(email)]
    email: String,
    #[validate(length(min = 8))]
    password: String,
}

async fn create(ValidatedJson(user): ValidatedJson<NewUser>) {
    // user is deserialized AND validated here
    let _ = user;
}
```

## Example 4 : a parts extractor reading a substate via FromRef

The extractor depends on `TenantRegistry`, one field of a composite
application state. The `FromRef` bound keeps it reusable with any state that
can produce that field.

```rust
// axum 0.8 - 0.7 needs #[async_trait] on the impl
use axum::extract::{FromRef, FromRequestParts};
use axum::http::{header::HOST, request::Parts, StatusCode};

#[derive(Clone)]
pub struct TenantRegistry { /* ... */ }

impl TenantRegistry {
    fn lookup(&self, _host: &str) -> Option<Tenant> { None }
}

#[derive(Clone)]
pub struct Tenant { /* ... */ }

pub struct CurrentTenant(pub Tenant);

impl<S> FromRequestParts<S> for CurrentTenant
where
    TenantRegistry: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, state: &S)
        -> Result<Self, Self::Rejection>
    {
        let registry = TenantRegistry::from_ref(state);
        let host = parts
            .headers
            .get(HOST)
            .and_then(|v| v.to_str().ok())
            .ok_or((StatusCode::BAD_REQUEST, "missing host header"))?;
        registry
            .lookup(host)
            .map(CurrentTenant)
            .ok_or((StatusCode::NOT_FOUND, "unknown tenant"))
    }
}
```

The composite state, with `#[derive(FromRef)]` so `TenantRegistry::from_ref`
is generated:

```rust
// axum 0.7 / 0.8
use axum::extract::FromRef;

#[derive(Clone, FromRef)]
struct AppState {
    tenants: TenantRegistry,
    // other fields ...
}
```

## Example 5 : #[derive(FromRequestParts)] bundling several extractors

The derive runs an extractor per field. The `via` attribute names the
extractor used for that field; a field with no attribute must already be an
extractor type.

```rust
// axum 0.7 / 0.8 - derive macro syntax is version-stable
use axum::extract::{FromRequestParts, Query};
use axum_extra::TypedHeader;
use axum_extra::headers::ContentType;
use std::collections::HashMap;

#[derive(FromRequestParts)]
struct PageContext {
    #[from_request(via(Query))]
    query_params: HashMap<String, String>,
    content_type: TypedHeader<ContentType>,
}

async fn handler(ctx: PageContext) {
    let _ = (ctx.query_params, ctx.content_type);
}
```

## Example 6 : #[derive(FromRequest)] with via and a unified rejection

`#[from_request(rejection(ApiError))]` makes one rejection type cover every
field. `ApiError` must implement `IntoResponse` and `From<R>` for the
rejection `R` of each `via` field.

```rust
// axum 0.7 / 0.8 - derive macro syntax is version-stable
use axum::extract::{rejection::JsonRejection, FromRequest};
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::Json;

#[derive(serde::Deserialize)]
struct NewUserPayload {
    email: String,
}

#[derive(FromRequest)]
#[from_request(rejection(ApiError))]
struct CreateUser {
    #[from_request(via(Json))]
    payload: NewUserPayload,
}

struct ApiError(StatusCode, String);

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        (self.0, self.1).into_response()
    }
}

// One From impl per via-field rejection type. Here the only via field is
// Json, so From<JsonRejection> is required.
impl From<JsonRejection> for ApiError {
    fn from(rej: JsonRejection) -> Self {
        ApiError(rej.status(), rej.body_text())
    }
}

async fn create(CreateUser { payload }: CreateUser) {
    let _ = payload.email;
}
```

## Example 7 : porting a 0.7 extractor to 0.8

Before, Axum 0.7:

```rust
// axum 0.7
#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for RequestId {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S)
        -> Result<Self, Self::Rejection>
    {
        parts.headers.get("x-request-id")
            .and_then(|v| v.to_str().ok())
            .map(|s| RequestId(s.to_owned()))
            .ok_or((StatusCode::BAD_REQUEST, "missing x-request-id"))
    }
}
```

After, Axum 0.8 (delete the attribute, drop the `async-trait` dependency, keep
the body):

```rust
// axum 0.8
impl<S: Send + Sync> FromRequestParts<S> for RequestId {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S)
        -> Result<Self, Self::Rejection>
    {
        parts.headers.get("x-request-id")
            .and_then(|v| v.to_str().ok())
            .map(|s| RequestId(s.to_owned()))
            .ok_or((StatusCode::BAD_REQUEST, "missing x-request-id"))
    }
}
```
