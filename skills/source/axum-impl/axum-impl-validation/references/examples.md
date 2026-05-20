# axum-impl-validation : Examples

Working, version-annotated code for request-body validation in Axum. The
`validator` crate is version `0.20`. Code marked `// axum 0.7 / 0.8` is
identical on both versions; code marked `// axum 0.8` or `// axum 0.7` diverges.

## Example 1 : Cargo.toml

```toml
[dependencies]
axum = "0.8"                 # or "0.7"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
validator = { version = "0.20", features = ["derive"] }
regex = "1"                  # only when a `regex` validator is used

# Axum 0.7 ONLY - the custom extractor needs this; Axum 0.8 does not.
async-trait = "0.1"
```

## Example 2 : a struct with stacked validators

```rust
// validator 0.20 - verified from docs.rs
use serde::Deserialize;
use validator::{Validate, ValidationError};

#[derive(Debug, Validate, Deserialize)]
struct SignupData {
    #[validate(email)]
    mail: String,

    #[validate(url)]
    site: String,

    #[validate(length(min = 1), custom(function = "validate_unique_username"))]
    #[serde(rename = "firstName")]
    first_name: String,

    #[validate(range(min = 18, max = 20))]
    age: u32,
}

// validator 0.20 - verified from docs.rs. A custom validator is a free
// function returning Result<(), ValidationError>.
fn validate_unique_username(username: &str) -> Result<(), ValidationError> {
    if username == "xXxShad0wxXx" {
        return Err(ValidationError::new("terrible_username"));
    }
    Ok(())
}
```

## Example 3 : a password-confirm with must_match

```rust
// validator 0.20 - must_match names the field to compare against
use serde::Deserialize;
use validator::Validate;

#[derive(Debug, Validate, Deserialize)]
struct ChangePassword {
    #[validate(length(min = 8))]
    password1: String,

    #[validate(must_match(other = "password1"))]
    password2: String,
}
```

## Example 4 : nested struct validation

```rust
// validator 0.20 - the `nested` flag recurses into a sub-struct.
// The sub-struct's type MUST also derive Validate.
use serde::Deserialize;
use validator::Validate;

#[derive(Debug, Validate, Deserialize)]
struct Address {
    #[validate(length(min = 1, max = 80))]
    street: String,

    #[validate(length(equal = 6))]
    postcode: String,
}

#[derive(Debug, Validate, Deserialize)]
struct Order {
    #[validate(length(min = 1))]
    sku: String,

    #[validate(range(min = 1, max = 999))]
    quantity: u32,

    #[validate(nested)]              // recursively validates `address`
    address: Address,
}
```

## Example 5 : a regex validator with a static Regex

```rust
// validator 0.20 - the regex MUST point at a lazily-compiled static.
use std::sync::LazyLock;            // Rust 1.80+
use regex::Regex;
use serde::Deserialize;
use validator::Validate;

// Compiled once, on first use. NEVER per request.
static SLUG_RE: LazyLock<Regex> =
    LazyLock::new(|| Regex::new(r"^[a-z0-9]+(?:-[a-z0-9]+)*$").unwrap());

#[derive(Debug, Validate, Deserialize)]
struct Article {
    #[validate(regex(path = *SLUG_RE))]
    slug: String,

    #[validate(length(min = 1, max = 200))]
    title: String,
}
```

On a Rust version before 1.80, use `once_cell::sync::Lazy<Regex>` instead of
`std::sync::LazyLock`; the `regex(path = ...)` syntax is unchanged.

## Example 6 : the ValidatedJson<T> extractor (Axum 0.8)

```rust
// axum 0.8 + validator 0.20 - native async fn in trait, NO #[async_trait]
use axum::extract::{FromRequest, Request};
use axum::extract::rejection::JsonRejection;
use axum::http::StatusCode;
use axum::Json;
use serde_json::{json, Value};

/// A drop-in replacement for `Json<T>` that also runs `T::validate()`.
pub struct ValidatedJson<T>(pub T);

impl<S, T> FromRequest<S> for ValidatedJson<T>
where
    S: Send + Sync,
    T: serde::de::DeserializeOwned + validator::Validate,
    Json<T>: FromRequest<S, Rejection = JsonRejection>,
{
    type Rejection = (StatusCode, Json<Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        // 1. delegate JSON deserialization; a JsonRejection keeps its own status.
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|rej| (rej.status(), Json(json!({ "error": rej.body_text() }))))?;

        // 2. run the validator; a ValidationErrors maps to 422.
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

## Example 7 : the ValidatedJson<T> extractor (Axum 0.7)

```rust
// axum 0.7 + validator 0.20 - identical body, plus #[async_trait]
use async_trait::async_trait;
use axum::extract::{FromRequest, Request};
use axum::extract::rejection::JsonRejection;
use axum::http::StatusCode;
use axum::Json;
use serde_json::{json, Value};

pub struct ValidatedJson<T>(pub T);

#[async_trait]
impl<S, T> FromRequest<S> for ValidatedJson<T>
where
    S: Send + Sync,
    T: serde::de::DeserializeOwned + validator::Validate,
    Json<T>: FromRequest<S, Rejection = JsonRejection>,
{
    type Rejection = (StatusCode, Json<Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|rej| (rej.status(), Json(json!({ "error": rej.body_text() }))))?;
        value.validate().map_err(|e| {
            (StatusCode::UNPROCESSABLE_ENTITY, Json(json!({ "error": e.to_string() })))
        })?;
        Ok(ValidatedJson(value))
    }
}
```

The only line that differs from Example 6 is `#[async_trait]` on the `impl`
block. Porting 0.7 to 0.8: delete that line and drop the `async-trait`
dependency from `Cargo.toml`.

## Example 8 : the structured 422 body

`ValidationErrors` is `Serialize`, so returning it directly produces a
per-field error map instead of one flat string. Swap step 2 of the extractor:

```rust
// axum 0.7 / 0.8 - structured 422 body keyed by field name
use axum::Json;
use axum::http::StatusCode;

// Replace the `value.validate().map_err(...)` block with:
value.validate().map_err(|e: validator::ValidationErrors| {
    (StatusCode::UNPROCESSABLE_ENTITY, Json(e))
})?;
```

The `Self::Rejection` type then becomes
`(StatusCode, Json<validator::ValidationErrors>)`. The response JSON is an
object whose keys are field names and whose values list each failed code, for
example `{"age":[{"code":"range", ...}]}`.

## Example 9 : wiring the extractor into a router

```rust
// axum 0.7 / 0.8 - the handler signature is version-stable
use axum::extract::State;
use axum::http::StatusCode;
use axum::response::IntoResponse;
use axum::routing::post;
use axum::Router;

#[derive(Clone)]
struct AppState { /* db pool, config, ... */ }

// ValidatedJson is the LAST argument; it consumes the body.
async fn signup(ValidatedJson(data): ValidatedJson<SignupData>) -> impl IntoResponse {
    // `data` is guaranteed deserialized AND validated here.
    let _ = data;
    StatusCode::CREATED
}

// State (FromRequestParts) comes first, ValidatedJson (FromRequest) last.
async fn create_order(
    State(_state): State<AppState>,
    ValidatedJson(order): ValidatedJson<Order>,
) -> impl IntoResponse {
    let _ = order;
    StatusCode::CREATED
}

fn app(state: AppState) -> Router {
    Router::new()
        .route("/signup", post(signup))
        .route("/orders", post(create_order))
        .with_state(state)
}
```

A request with valid JSON that fails a rule (an `age` of 9000, an empty
`first_name`) gets `422 Unprocessable Entity`. A request with malformed JSON or
a missing `Content-Type` gets the `JsonRejection` status (a 400-class code).

## Example 10 : the garde context-aware alternative

```rust
// garde - reach for this when a validator needs runtime context.
use garde::Validate;

struct DbContext {
    reserved_names: Vec<String>,
}

#[derive(Validate)]
#[garde(context(DbContext))]
struct CreateTeam {
    #[garde(length(min = 1, max = 40))]
    name: String,

    #[garde(custom(name_is_free))]
    slug: String,
}

fn name_is_free(slug: &str, ctx: &DbContext) -> garde::Result {
    if ctx.reserved_names.iter().any(|r| r == slug) {
        return Err(garde::Error::new("slug is reserved"));
    }
    Ok(())
}

// Validation then takes the context as a required argument:
//   team.validate_with(&db_context)?;
```

`garde` makes the context a required argument of validation, so a check can
depend on a database handle or tenant without global statics. For stateless
APIs, `validator` remains the default.
