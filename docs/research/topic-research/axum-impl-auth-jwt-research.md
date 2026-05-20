# Topic Research : axum-impl-auth-jwt

> Skill : `axum-impl-auth-jwt`
> Category : impl
> Researched : 2026-05-20
> Target versions : Axum 0.7 and 0.8
> Verified against : the official `tokio-rs/axum` `examples/jwt` crate and the
> `jsonwebtoken` v10.4.0 docs.rs API reference. All API claims below are
> WebFetch-verified on 2026-05-20.

This file supports a single skill covering stateless JWT authentication in Axum.
It is scoped to the canonical pattern shipped in the official `examples/jwt`
crate: a `Claims` struct, a `Keys` struct holding an `EncodingKey` and a
`DecodingKey`, an `AuthError` enum, a `FromRequestParts` extractor that turns any
handler taking `Claims` into a protected handler, and a login route that issues a
token.

---

## 1. Crate dependencies (verified)

The official `examples/jwt/Cargo.toml` pins exactly these dependencies:

```toml
axum = { path = "../../axum" }
axum-extra = { path = "../../axum-extra", features = ["typed-header"] }
jsonwebtoken = { version = "10", features = ["aws_lc_rs"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

Key facts for the skill:

- `jsonwebtoken = "10"` with the `aws_lc_rs` feature. v10 selects the crypto
  backend by feature flag (`aws_lc_rs` is the example's choice; `rust_crypto` is
  the alternative). The crate's current published release is **v10.4.0**.
- `axum-extra` is required with the `typed-header` feature, because the
  `Authorization<Bearer>` typed header lives in `axum-extra`, not in `axum` core.
  In Axum 0.7, `TypedHeader` was in `axum` behind a `headers` feature; in 0.8 it
  moved to `axum-extra`. The skill MUST use the `axum-extra` path.
- The example references `axum` by local path, so it tracks the in-development
  version. The skill targets Axum 0.7 and 0.8.

---

## 2. The verified JWT example structure

The official example is built from six pieces. The skill should mirror this
exact decomposition.

### 2.1 The `Claims` struct — `exp` is mandatory

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims {
    sub: String,
    company: String,
    exp: usize,
}
```

The `exp` field is **not optional**. `Validation::default()` sets
`validate_exp = true` and `required_spec_claims = {"exp"}` (both verified, see
section 4). A `Claims` struct without an `exp` field, or a token whose `exp` is
in the past, is rejected by `decode`. ALWAYS include `exp` as a Unix timestamp
(seconds since epoch). `exp` is typically computed as "now + token lifetime",
for example with the `time` or `chrono` crate.

### 2.2 The `Keys` struct and `Keys::new`

```rust
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
```

Both keys are derived from the **same** shared secret because HS256 (the default
algorithm) is a symmetric MAC: the same secret signs and verifies.
`EncodingKey::from_secret` and `DecodingKey::from_secret` both take `&[u8]`.

### 2.3 The `KEYS` static — secret from an environment variable

```rust
static KEYS: LazyLock<Keys> = LazyLock::new(|| {
    let secret = std::env::var("JWT_SECRET").expect("JWT_SECRET must be set");
    Keys::new(secret.as_bytes())
});
```

This is the single most important security rule in the skill. The secret is read
from the `JWT_SECRET` environment variable at first access via
`std::sync::LazyLock`. NEVER pass a string literal such as
`Keys::new(b"super-secret")` — that bakes the credential into the compiled
binary and the git history, lets anyone with read access forge valid tokens, and
forces a recompile to rotate the secret. ALWAYS read the secret from the
environment, fail fast with `.expect(...)` if it is absent, and add `.env` to
`.gitignore`.

`LazyLock` is `std`-stable since Rust 1.80. On older toolchains use
`once_cell::sync::Lazy` with the same closure.

### 2.4 The `AuthError` enum and its `IntoResponse` impl

```rust
enum AuthError {
    WrongCredentials,
    MissingCredentials,
    TokenCreation,
    InvalidToken,
}

impl IntoResponse for AuthError {
    fn into_response(self) -> Response {
        let (status, error_message) = match self {
            AuthError::WrongCredentials => {
                (StatusCode::UNAUTHORIZED, "Wrong credentials")
            }
            AuthError::MissingCredentials => {
                (StatusCode::BAD_REQUEST, "Missing credentials")
            }
            AuthError::TokenCreation => {
                (StatusCode::INTERNAL_SERVER_ERROR, "Token creation error")
            }
            AuthError::InvalidToken => {
                (StatusCode::BAD_REQUEST, "Invalid token")
            }
        };
        let body = Json(json!({ "error": error_message }));
        (status, body).into_response()
    }
}
```

Verified status-code mapping from the official example:

| Variant | Status | Meaning |
|---|---|---|
| `WrongCredentials` | `401 UNAUTHORIZED` | login credentials did not match |
| `MissingCredentials` | `400 BAD_REQUEST` | login body missing fields |
| `TokenCreation` | `500 INTERNAL_SERVER_ERROR` | `encode` failed |
| `InvalidToken` | `400 BAD_REQUEST` | token absent or `decode` failed |

Because `AuthError` implements `IntoResponse`, it is a valid handler return type
and the extractor's `type Rejection`. This is the same error-handling pattern as
the general `AppError` enum: handlers stay infallible at the `tower::Service`
level while still returning typed errors.

### 2.5 The `FromRequestParts` extractor — this is what makes handlers protected

```rust
impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        // Extract the `Authorization: Bearer <token>` header.
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;

        // Decode and validate the token (expiry checked by Validation::default).
        let token_data = decode::<Claims>(
            bearer.token(),
            &KEYS.decoding,
            &Validation::default(),
        )
        .map_err(|_| AuthError::InvalidToken)?;

        Ok(token_data.claims)
    }
}
```

`Claims` implements `FromRequestParts` (not `FromRequest`): it reads only request
parts (the header), never the body, so it may appear in any handler argument
position. The single most important consequence: **any handler that takes
`claims: Claims` as an argument is automatically protected** — if the token is
missing, malformed, or expired, the extractor returns
`Err(AuthError::InvalidToken)`, which becomes the HTTP response and the handler
body never runs. Authentication is enforced by the type system, with no explicit
guard call inside the handler.

`parts.extract::<T>()` is the helper that runs a sub-extractor against the
request parts. The skill must show the full chain: extract the typed header,
then `decode` the bearer token.

### 2.6 The protected handler

```rust
async fn protected(claims: Claims) -> Result<String, AuthError> {
    Ok(format!(
        "Welcome to the protected area :)\nYour data:\n{claims:?}"
    ))
}
```

The signature alone communicates "this route requires a valid token". No body
logic is needed to authenticate.

### 2.7 The `authorize` (login) handler — issuing a token

```rust
async fn authorize(
    Json(payload): Json<AuthPayload>,
) -> Result<Json<AuthBody>, AuthError> {
    // Reject an empty login body.
    if payload.client_id.is_empty() || payload.client_secret.is_empty() {
        return Err(AuthError::MissingCredentials);
    }
    // Verify credentials (the example hardcodes a check; real code queries a DB).
    if payload.client_id != "foo" || payload.client_secret != "bar" {
        return Err(AuthError::WrongCredentials);
    }

    let claims = Claims {
        sub: "b@b.com".to_owned(),
        company: "ACME".to_owned(),
        // Mandatory `exp` — a real value, not a literal far-future constant.
        exp: 2000000000,
    };

    // Create the token, signing with the encoding key.
    let token = encode(&Header::default(), &claims, &KEYS.encoding)
        .map_err(|_| AuthError::TokenCreation)?;

    Ok(Json(AuthBody::new(token)))
}
```

The login route deserializes a credentials payload, validates it (returning
`MissingCredentials` or `WrongCredentials`), builds the `Claims`, calls `encode`
with `Header::default()` (HS256) and the encoding key, and returns the token in
a JSON body. An `encode` failure maps to `TokenCreation`.

Router wiring:

```rust
let app = Router::new()
    .route("/protected", get(protected))
    .route("/authorize", post(authorize));
```

---

## 3. Exact `jsonwebtoken` API (verified, v10.4.0)

```rust
// Encode the header and claims, sign the payload.
pub fn encode<T: Serialize>(
    header: &Header,
    claims: &T,
    key: &EncodingKey,
) -> Result<String>

// Decode and validate a JWT.
pub fn decode<T: DeserializeOwned>(
    token: impl AsRef<[u8]>,
    key: &DecodingKey,
    validation: &Validation,
) -> Result<TokenData<T>>
```

- `encode` returns `Result<String>` — the compact serialized JWT.
- `decode` returns `Result<TokenData<T>>`. `TokenData` is the success type of a
  `decode` call; the claims are read via `token_data.claims` and the header via
  `token_data.header`.
- Both `Result`s are `jsonwebtoken::errors::Result`, an alias whose error type
  is `jsonwebtoken::errors::Error`. The example collapses any error into
  `AuthError::InvalidToken` / `AuthError::TokenCreation` with `.map_err`.

**`Header`.** `Header::default()` sets `alg` to `HS256` and `typ` to `"JWT"`; all
other fields are optional. The example uses `Header::default()` unchanged.

**`EncodingKey` / `DecodingKey`.** For the HS256 symmetric case both are built
from the shared secret with `EncodingKey::from_secret(&[u8])` and
`DecodingKey::from_secret(&[u8])`. The crate also exposes asymmetric
constructors (`from_rsa_pem`, `from_ec_pem`, `from_ed_pem`, `from_rsa_der`,
etc.) for RS256/ES256/EdDSA; the skill targets the symmetric default and may
mention the asymmetric constructors as an aside only.

---

## 4. `Validation::default()` — why `exp` is mandatory (verified)

`Validation` holds the checks applied after a token is decoded. Verified default
field values for `Validation::default()`:

| Field | Default | Effect |
|---|---|---|
| `validate_exp` | `true` | expiry IS checked |
| `required_spec_claims` | `{"exp"}` | a token with no `exp` claim is rejected |
| `leeway` | `60` | 60 seconds of clock-skew tolerance |
| `reject_tokens_expiring_in_less_than` | `0` | no early-expiry rejection |
| `validate_nbf` | `false` | `nbf` not checked unless opted in |
| `validate_aud` | `true` | audience checked if `aud` is set |
| `algorithms` | `vec![Algorithm::HS256]` | only HS256 accepted |
| `aud`, `iss`, `sub` | `None` | no audience/issuer/subject constraint |

The skill MUST state plainly: with `Validation::default()`, expiry validation is
**on** and `exp` is a **required** claim. Therefore the `Claims` struct MUST
carry an `exp: usize` field and the login handler MUST set it to a real future
timestamp. Omitting `exp`, or reusing a fixed far-past value, makes every token
fail `decode`. To check a specific audience or issuer, build a non-default
`Validation` and call `.set_audience(...)` / `.set_issuer(...)`.

---

## 5. Version note : `FromRequestParts` impl differs between 0.7 and 0.8

Axum 0.8 removed the `#[async_trait]` macro from `FromRequest` and
`FromRequestParts`, replacing it with native return-position `impl Trait` in
traits (RPITIT, MSRV Rust 1.75). The JWT extractor is therefore written
differently on the two versions. The skill MUST show both forms.

**Axum 0.8 (native async — verified, this is what the current example uses):**

```rust
impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        // ... body identical ...
    }
}
```

**Axum 0.7 (`#[async_trait]` required):**

```rust
#[async_trait]
impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        // ... body identical ...
    }
}
```

The only difference is the `#[async_trait]` attribute on the `impl` block (and
the `use axum::async_trait;` import on 0.7). The extractor body, the trait
bounds (`S: Send + Sync`), the `type Rejection`, and the method signature are
otherwise identical. The `TypedHeader` import path also moved: `axum::TypedHeader`
behind the `headers` feature on 0.7, `axum_extra::TypedHeader` (with the
`typed-header` feature) on 0.8.

---

## 6. Imports (for completeness)

```rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
    response::{IntoResponse, Response},
    routing::{get, post},
    Json, Router,
};
use axum_extra::{
    headers::{authorization::Bearer, Authorization},
    typed_header::TypedHeaderRejectionReason, // optional, for granular header errors
    TypedHeader,
};
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use serde_json::json;
use std::sync::LazyLock;
```

---

## 7. Anti-patterns this skill must call out

1. **Hardcoded JWT secret.** `Keys::new(b"super-secret")` bakes the credential
   into the binary and git history; anyone can forge tokens. ALWAYS read
   `JWT_SECRET` from the environment.
2. **Missing `exp` field.** A `Claims` struct without `exp`, or a stale `exp`,
   fails `decode` under `Validation::default()`. ALWAYS include a real future
   `exp`.
3. **`Claims` implementing `FromRequest` instead of `FromRequestParts`.** The
   token lives in a header, not the body. `FromRequestParts` lets `Claims` sit in
   any argument position; `FromRequest` would force it to be the last argument
   and is semantically wrong.
4. **Forgetting `#[async_trait]` on Axum 0.7** (or leaving it on 0.8). The
   attribute is required on 0.7 and rejected on 0.8.
5. **`TypedHeader` from the wrong crate.** On Axum 0.8 it is in `axum-extra` with
   the `typed-header` feature; importing from `axum` fails to compile.
6. **Leaking the underlying `jsonwebtoken` error to the client.** Map every
   `decode`/`encode` error into an `AuthError` variant so internal crypto
   details never reach the response body.

---

## Sources verified (2026-05-20)

- https://github.com/tokio-rs/axum/blob/main/examples/jwt/src/main.rs —
  `Claims` struct (mandatory `exp`), `Keys` struct + `Keys::new`, the `KEYS`
  `LazyLock` static reading `JWT_SECRET`, the native-async `FromRequestParts`
  impl for `Claims`, the `AuthError` enum and its `IntoResponse` status mapping,
  the `protected` handler signature, the `authorize` login handler with
  `encode(&Header::default(), &claims, &KEYS.encoding)`.
- https://github.com/tokio-rs/axum/blob/main/examples/jwt/Cargo.toml —
  `jsonwebtoken = { version = "10", features = ["aws_lc_rs"] }`,
  `axum-extra` with the `typed-header` feature, `serde` with `derive`,
  `serde_json`, `tokio` with `full`.
- https://docs.rs/jsonwebtoken/latest/jsonwebtoken/ — crate version v10.4.0,
  `Header` defaults to `alg = HS256` / `typ = "JWT"`.
- https://docs.rs/jsonwebtoken/latest/jsonwebtoken/fn.encode.html —
  `pub fn encode<T: Serialize>(header: &Header, claims: &T, key: &EncodingKey) -> Result<String>`.
- https://docs.rs/jsonwebtoken/latest/jsonwebtoken/fn.decode.html —
  `pub fn decode<T: DeserializeOwned>(token: impl AsRef<[u8]>, key: &DecodingKey, validation: &Validation) -> Result<TokenData<T>>`.
- https://docs.rs/jsonwebtoken/latest/jsonwebtoken/struct.Validation.html —
  `Validation::default()` field values: `validate_exp = true`,
  `required_spec_claims = {"exp"}`, `leeway = 60`, `validate_nbf = false`,
  `validate_aud = true`, `algorithms = vec![Algorithm::HS256]`.
- Cross-referenced: `docs/research/vooronderzoek-axum.md` section 5 and
  `docs/research/fragments/research-c-errors-auth-db-ops.md` section 4 (the
  `#[async_trait]` 0.7-vs-0.8 distinction, the env-var secret rule, the
  `FromRequestParts`-makes-handlers-protected pattern).
