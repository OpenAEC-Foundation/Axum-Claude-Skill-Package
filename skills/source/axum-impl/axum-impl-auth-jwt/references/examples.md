# Axum JWT Authentication : Working Examples

Every example is version-annotated. Code is verified against the `tokio-rs/axum`
`examples/jwt` crate and the `jsonwebtoken` v10.4.0 docs on 2026-05-20. Examples target
Axum 0.8 unless marked otherwise.

## Example 1: Complete end-to-end JWT service

`Cargo.toml`:

```toml
[dependencies]
axum = "0.8"
axum-extra = { version = "0.10", features = ["typed-header"] }
jsonwebtoken = { version = "10", features = ["aws_lc_rs"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1.0", features = ["full"] }
```

`src/main.rs`:

```rust
// axum 0.8
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
    response::{IntoResponse, Response},
    routing::{get, post},
    Json, RequestPartsExt, Router,
};
use axum_extra::{
    headers::{authorization::Bearer, Authorization},
    TypedHeader,
};
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use serde_json::json;
use std::sync::LazyLock;

// 1: secret read from the environment, NEVER a literal
static KEYS: LazyLock<Keys> = LazyLock::new(|| {
    let secret = std::env::var("JWT_SECRET").expect("JWT_SECRET must be set");
    Keys::new(secret.as_bytes())
});

struct Keys {
    encoding: EncodingKey,
    decoding: DecodingKey,
}

impl Keys {
    fn new(secret: &[u8]) -> Self {
        Self {
            encoding: EncodingKey::from_secret(secret),
            decoding: DecodingKey::from_secret(secret),
        }
    }
}

// 2: Claims, exp is mandatory under Validation::default()
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims {
    sub: String,
    company: String,
    exp: usize,
}

#[derive(Deserialize)]
struct AuthPayload {
    client_id: String,
    client_secret: String,
}

#[derive(Serialize)]
struct AuthBody {
    access_token: String,
    token_type: String,
}

impl AuthBody {
    fn new(access_token: String) -> Self {
        Self { access_token, token_type: "Bearer".to_string() }
    }
}

// 3: error enum, maps cleanly to HTTP responses
enum AuthError {
    WrongCredentials,
    MissingCredentials,
    TokenCreation,
    InvalidToken,
}

impl IntoResponse for AuthError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            AuthError::WrongCredentials => (StatusCode::UNAUTHORIZED, "Wrong credentials"),
            AuthError::MissingCredentials => (StatusCode::BAD_REQUEST, "Missing credentials"),
            AuthError::TokenCreation => (StatusCode::INTERNAL_SERVER_ERROR, "Token creation error"),
            AuthError::InvalidToken => (StatusCode::BAD_REQUEST, "Invalid token"),
        };
        (status, Json(json!({ "error": msg }))).into_response()
    }
}

// 4: the extractor that makes any handler taking Claims protected
impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;

        let token_data = decode::<Claims>(bearer.token(), &KEYS.decoding, &Validation::default())
            .map_err(|_| AuthError::InvalidToken)?;

        Ok(token_data.claims)
    }
}

// 5: protected handler, authentication enforced by the Claims argument
async fn protected(claims: Claims) -> Result<String, AuthError> {
    Ok(format!("Welcome to the protected area. Your data: {claims:?}"))
}

// 6: login handler, issues a token
async fn authorize(Json(payload): Json<AuthPayload>) -> Result<Json<AuthBody>, AuthError> {
    if payload.client_id.is_empty() || payload.client_secret.is_empty() {
        return Err(AuthError::MissingCredentials);
    }
    if payload.client_id != "foo" || payload.client_secret != "bar" {
        return Err(AuthError::WrongCredentials);
    }

    let claims = Claims {
        sub: "b@b.com".to_owned(),
        company: "ACME".to_owned(),
        exp: now_plus_seconds(3600),
    };

    let token = encode(&Header::default(), &claims, &KEYS.encoding)
        .map_err(|_| AuthError::TokenCreation)?;

    Ok(Json(AuthBody::new(token)))
}

fn now_plus_seconds(seconds: u64) -> usize {
    use std::time::{SystemTime, UNIX_EPOCH};
    let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs();
    (now + seconds) as usize
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/protected", get(protected))
        .route("/authorize", post(authorize));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

Run it:

```bash
JWT_SECRET="$(openssl rand -hex 32)" cargo run
```

## Example 2: The extractor on Axum 0.7

The body is identical to Example 1; the 0.7 form differs only in the `#[async_trait]`
attribute and the `TypedHeader` import path.

```rust
// axum 0.7
use axum::{async_trait, extract::FromRequestParts, http::request::Parts, RequestPartsExt};
use axum::TypedHeader; // 0.7 : behind the `headers` feature, not axum-extra
use axum::headers::{authorization::Bearer, Authorization};

#[async_trait]
impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;

        let token_data = decode::<Claims>(bearer.token(), &KEYS.decoding, &Validation::default())
            .map_err(|_| AuthError::InvalidToken)?;

        Ok(token_data.claims)
    }
}
```

`Cargo.toml` for the 0.7 case enables the `headers` feature on `axum` instead of pulling
`Authorization<Bearer>` from `axum-extra`:

```toml
# axum 0.7
axum = { version = "0.7", features = ["headers"] }
```

## Example 3: Validating audience and issuer

`Validation::default()` checks only expiry. To require a specific audience and issuer,
build a non-default `Validation`:

```rust
// axum 0.8 (jsonwebtoken API is version-independent)
use jsonwebtoken::{decode, Validation};

fn decode_with_audience(token: &str) -> Result<Claims, AuthError> {
    let mut validation = Validation::default();
    validation.set_audience(&["my-api"]);
    validation.set_issuer(&["https://auth.example.com"]);

    let token_data = decode::<Claims>(token, &KEYS.decoding, &validation)
        .map_err(|_| AuthError::InvalidToken)?;
    Ok(token_data.claims)
}
```

The `Claims` struct must then carry matching `aud` and `iss` fields for the validation
to succeed.

## Example 4: Protected and public routes side by side

```rust
// axum 0.8
use axum::{routing::get, Router};

async fn public_health() -> &'static str {
    "ok" // no Claims argument, no token required
}

async fn private_profile(claims: Claims) -> Result<String, AuthError> {
    Ok(format!("profile for {}", claims.sub)) // Claims argument, token required
}

let app = Router::new()
    .route("/health", get(public_health))
    .route("/profile", get(private_profile));
```

Protection is per-handler and visible in the signature: a `claims: Claims` argument
means the route requires a valid token; its absence means the route is public.
