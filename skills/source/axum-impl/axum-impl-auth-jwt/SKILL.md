---
name: axum-impl-auth-jwt
description: >
  Use when adding stateless JWT authentication to an Axum service, when a login
  route must issue a bearer token, when handlers must require a valid token, or
  when migrating a JWT extractor from Axum 0.7 to 0.8.
  Prevents the hardcoded-secret security hole, the missing exp claim that makes
  every token fail to decode, implementing Claims as FromRequest instead of
  FromRequestParts, and leaking raw jsonwebtoken errors to the client.
  Covers the jsonwebtoken crate (v10, aws_lc_rs), the Claims struct, encode and
  decode, Validation::default, EncodingKey and DecodingKey from an env-var secret,
  the FromRequestParts extractor that makes any handler protected, the AuthError
  enum with IntoResponse, and the async_trait difference between 0.7 and 0.8.
  Keywords: axum jwt, jsonwebtoken, encode, decode, Claims, EncodingKey,
  DecodingKey, Validation, FromRequestParts, Authorization Bearer, JWT_SECRET,
  401 unauthorized, InvalidToken, token expired, every token rejected, how do I
  protect a route, how do I add login to axum, what is a bearer token.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Axum JWT Authentication

## Overview

Stateless JWT authentication in Axum is built from six pieces, mirroring the official
`tokio-rs/axum` `examples/jwt` crate: a `Claims` struct, a `Keys` struct holding an
`EncodingKey` and a `DecodingKey`, an `AuthError` enum that implements `IntoResponse`,
a `FromRequestParts` extractor for `Claims`, a protected handler, and a login handler
that issues a token.

The core mechanism: `Claims` implements `FromRequestParts`, so any handler that takes
`claims: Claims` as an argument is automatically protected. A missing, malformed, or
expired token makes the extractor return `Err(AuthError::InvalidToken)`, which becomes
the HTTP response, and the handler body never runs. Authentication is enforced by the
type system, not by a guard call inside the handler.

Two rules dominate correct JWT code:

1. The signing secret MUST be read from an environment variable, NEVER a string
   literal. A literal bakes the credential into the binary and git history.
2. The `Claims` struct MUST carry an `exp` field. `Validation::default()` checks
   expiry and requires the `exp` claim; without it every token fails `decode`.

## Quick Reference

| Need | API | Notes |
|------|-----|-------|
| Sign a token | `encode(&Header::default(), &claims, &keys.encoding)` | `Header::default()` is HS256 |
| Verify a token | `decode::<Claims>(token, &keys.decoding, &Validation::default())` | expiry checked automatically |
| Symmetric key (HS256) | `EncodingKey::from_secret(&[u8])` / `DecodingKey::from_secret(&[u8])` | same shared secret both sides |
| Read the bearer header | `TypedHeader<Authorization<Bearer>>`, then `bearer.token()` | `axum-extra`, `typed-header` feature |
| Protect a handler | give it a `claims: Claims` argument | extractor enforces the token |
| Read the secret | `std::env::var("JWT_SECRET")` | NEVER a string literal |
| Map errors to responses | `AuthError` enum + `impl IntoResponse` | never leak `jsonwebtoken` errors |

Required crates (verified against `examples/jwt/Cargo.toml`):

```toml
axum = "0.8"
axum-extra = { version = "0.10", features = ["typed-header"] }
jsonwebtoken = { version = "10", features = ["aws_lc_rs"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1.0", features = ["full"] }
```

## Decision Trees

### Choosing the signing algorithm

```
Who signs tokens and who verifies them?
├── ONE service does both        -> HS256 symmetric, EncodingKey/DecodingKey::from_secret
└── one signs, others verify     -> RS256 / ES256 / EdDSA asymmetric,
                                    EncodingKey::from_rsa_pem + DecodingKey::from_rsa_pem
```

This skill targets the HS256 symmetric default. For the asymmetric case use the PEM/DER
constructors and set `Validation.algorithms` to match.

### FromRequest vs FromRequestParts for Claims

```
Where does the token live?
└── in the Authorization header  -> Claims implements FromRequestParts (ALWAYS)
```

ALWAYS implement `FromRequestParts` for `Claims`. The token is read from a header, never
the body, so `Claims` can sit in any handler argument position. `FromRequest` would
force `Claims` to be the last argument, consume the body, and is semantically wrong.

### Picking the extractor form by Axum version

```
Which Axum version?
├── 0.8  -> native async fn in trait, NO #[async_trait]
└── 0.7  -> #[async_trait] on the impl block, plus use axum::async_trait
```

## Patterns

### Pattern 1: The Claims struct, exp is mandatory

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims {
    sub: String,
    company: String,
    exp: usize, // Unix timestamp (seconds). MANDATORY.
}
```

`Validation::default()` sets `validate_exp = true` and `required_spec_claims = {"exp"}`.
ALWAYS include `exp` as a real future Unix timestamp. A `Claims` struct without `exp`,
or a token whose `exp` is in the past, is rejected by `decode`.

### Pattern 2: The Keys struct and the env-var secret

```rust
use jsonwebtoken::{DecodingKey, EncodingKey};
use std::sync::LazyLock;

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

// Secret read from the environment at first access. NEVER a string literal.
static KEYS: LazyLock<Keys> = LazyLock::new(|| {
    let secret = std::env::var("JWT_SECRET").expect("JWT_SECRET must be set");
    Keys::new(secret.as_bytes())
});
```

Both keys derive from the same shared secret because HS256 is a symmetric MAC. ALWAYS
read the secret from `JWT_SECRET` and fail fast with `.expect(...)` if it is absent.
NEVER write `Keys::new(b"super-secret")`. `LazyLock` is stable since Rust 1.80; on older
toolchains use `once_cell::sync::Lazy` with the same closure.

### Pattern 3: The AuthError enum

```rust
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use serde_json::json;

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
```

Because `AuthError` implements `IntoResponse`, it is a valid handler return type and the
extractor's `type Rejection`. ALWAYS map `jsonwebtoken` errors into an `AuthError`
variant; NEVER let a raw crypto error reach the response body.

### Pattern 4: The FromRequestParts extractor

This is what makes any handler taking `Claims` protected.

```rust
// axum 0.8 : native async fn in trait, NO #[async_trait]
use axum::{extract::FromRequestParts, http::request::Parts, RequestPartsExt};
use axum_extra::{
    headers::{authorization::Bearer, Authorization},
    TypedHeader,
};
use jsonwebtoken::{decode, Validation};

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

```rust
// axum 0.7 : #[async_trait] is REQUIRED
use axum::async_trait;

#[async_trait]
impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // body identical to the 0.8 version above
    }
}
```

The only difference between versions is the `#[async_trait]` attribute (required on 0.7,
rejected on 0.8) and the `TypedHeader` import path (`axum::TypedHeader` behind the
`headers` feature on 0.7, `axum_extra::TypedHeader` with the `typed-header` feature on
0.8). The extractor body, trait bounds, and method signature are otherwise identical.

### Pattern 5: The protected handler

```rust
async fn protected(claims: Claims) -> Result<String, AuthError> {
    Ok(format!("Welcome. Your data: {claims:?}"))
}
```

The `claims: Claims` argument alone makes this route require a valid token. No guard
call inside the body is needed or correct.

### Pattern 6: The login handler that issues a token

```rust
use jsonwebtoken::{encode, Header};

async fn authorize(Json(payload): Json<AuthPayload>) -> Result<Json<AuthBody>, AuthError> {
    if payload.client_id.is_empty() || payload.client_secret.is_empty() {
        return Err(AuthError::MissingCredentials);
    }
    if payload.client_id != "foo" || payload.client_secret != "bar" {
        return Err(AuthError::WrongCredentials); // real code queries a database here
    }

    let claims = Claims {
        sub: "b@b.com".to_owned(),
        company: "ACME".to_owned(),
        exp: now_plus_seconds(3600), // a real future timestamp, not a literal
    };

    let token = encode(&Header::default(), &claims, &KEYS.encoding)
        .map_err(|_| AuthError::TokenCreation)?;

    Ok(Json(AuthBody::new(token)))
}
```

`Header::default()` selects HS256. Compute `exp` as "now plus token lifetime" with the
`time` or `chrono` crate. An `encode` failure maps to `AuthError::TokenCreation`.

Router wiring:

```rust
use axum::{routing::{get, post}, Router};

let app = Router::new()
    .route("/protected", get(protected))
    .route("/authorize", post(authorize));
```

### Pattern 7: Validating audience or issuer

`Validation::default()` checks expiry only. To constrain audience or issuer, build a
non-default `Validation`:

```rust
let mut validation = Validation::default();
validation.set_audience(&["my-api"]);
validation.set_issuer(&["my-auth-server"]);
let token_data = decode::<Claims>(token, &KEYS.decoding, &validation)?;
```

## Reference Links

- `references/methods.md` : complete signatures for `encode`, `decode`, `Header`,
  `EncodingKey`, `DecodingKey`, `Validation`, `TokenData`, and the `FromRequestParts`
  trait, with 0.7 / 0.8 differences.
- `references/examples.md` : the full end-to-end JWT example (Cargo.toml, all six
  pieces, both extractor forms), version-annotated.
- `references/anti-patterns.md` : real mistakes with the exact symptom and WHY each
  fails, including the hardcoded-secret security hole.

## Related Skills

- `axum-syntax-custom-extractors` : the `FromRequestParts` trait in depth.
- `axum-impl-authorization` : role and permission checks on top of authentication.
- `axum-errors-handling` : the `IntoResponse` error-enum pattern in general.
