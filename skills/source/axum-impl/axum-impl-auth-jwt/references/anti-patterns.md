# Axum JWT Authentication : Anti-Patterns

Each entry is a real mistake, the exact symptom it produces, and WHY it fails. Verified
against the `tokio-rs/axum` `examples/jwt` crate and `jsonwebtoken` v10.4.0 on
2026-05-20.

## 1: Hardcoded JWT secret (security hole)

```rust
// WRONG : the credential is now in the binary and in git history
static KEYS: LazyLock<Keys> = LazyLock::new(|| Keys::new(b"super-secret"));
```

Symptom: no compiler error, no runtime error. The code works, which is exactly what
makes this dangerous.

WHY it fails: a literal secret is compiled into the binary and committed to git history.
Anyone with read access to the repository or the binary can extract it and forge valid
tokens for any user. Rotating the secret requires a recompile and redeploy. This is the
single most severe mistake in JWT code.

Fix: read the secret from an environment variable, fail fast if absent, and add `.env`
to `.gitignore`:

```rust
static KEYS: LazyLock<Keys> = LazyLock::new(|| {
    let secret = std::env::var("JWT_SECRET").expect("JWT_SECRET must be set");
    Keys::new(secret.as_bytes())
});
```

ALWAYS read the signing secret from the environment. NEVER write it as a literal,
NEVER commit it, NEVER log it.

## 2: Claims struct without an exp field

```rust
// WRONG : no exp field
#[derive(Serialize, Deserialize)]
struct Claims {
    sub: String,
}
```

Symptom: every call to `decode::<Claims>` fails with
`ErrorKind::MissingRequiredClaim("exp")`, so every protected request returns
`400 Invalid token` even with a freshly issued token.

WHY it fails: `Validation::default()` sets `required_spec_claims = {"exp"}` and
`validate_exp = true`. A token whose payload has no `exp` claim is rejected before the
handler runs.

Fix: add `exp: usize` to `Claims` and set it to a real future Unix timestamp in the
login handler.

## 3: Stale or far-past exp value

```rust
// WRONG : exp is a fixed value already in the past
let claims = Claims { sub: "...".into(), company: "...".into(), exp: 1000000000 };
```

Symptom: `decode` fails with `ErrorKind::ExpiredSignature`; the token is rejected the
moment it is issued.

WHY it fails: `validate_exp = true` compares `exp` against the current time. A timestamp
in the past is an already-expired token.

Fix: compute `exp` as "now plus token lifetime", for example
`now_plus_seconds(3600)` using `SystemTime`, the `time` crate, or `chrono`.

## 4: Implementing FromRequest instead of FromRequestParts

```rust
// WRONG : Claims reads a header, not the body
impl<S> FromRequest<S> for Claims { /* ... */ }
```

Symptom: `Claims` can only appear as the last handler argument, and combining it with a
body extractor such as `Json<T>` fails to compile with conflicting-extractor errors.

WHY it fails: `FromRequest` consumes the whole request including the body and may only
be the final argument. The JWT token lives in the `Authorization` header, which is a
request *part*. `FromRequestParts` reads parts only and can sit in any argument
position.

Fix: implement `FromRequestParts` for `Claims`.

## 5: Wrong async_trait usage for the Axum version

```rust
// WRONG on axum 0.8 : #[async_trait] was removed
#[async_trait]
impl<S> FromRequestParts<S> for Claims { /* ... */ }
```

```rust
// WRONG on axum 0.7 : #[async_trait] is required and is missing here
impl<S> FromRequestParts<S> for Claims { /* ... */ }
```

Symptom on 0.8 with the attribute: a macro or trait-resolution compile error.
Symptom on 0.7 without it: `error: functions in traits cannot be declared async` or an
RPITIT-not-supported error.

WHY it fails: Axum 0.8 replaced `#[async_trait]` with native async fn in traits (RPITIT,
Rust 1.75). The attribute is required on 0.7 and rejected on 0.8.

Fix: use `#[async_trait]` plus `use axum::async_trait;` on 0.7; omit it entirely on 0.8.

## 6: Importing TypedHeader from the wrong crate on Axum 0.8

```rust
// WRONG on axum 0.8
use axum::TypedHeader;
```

Symptom: `error[E0432]: unresolved import axum::TypedHeader`.

WHY it fails: on Axum 0.8 `TypedHeader` and `Authorization<Bearer>` moved out of `axum`
core into `axum-extra`. On 0.7 they were in `axum` behind the `headers` feature.

Fix on 0.8: `use axum_extra::TypedHeader;` and
`use axum_extra::headers::{authorization::Bearer, Authorization};`, with
`axum-extra = { features = ["typed-header"] }` in `Cargo.toml`.

## 7: Leaking the raw jsonwebtoken error to the client

```rust
// WRONG : the crypto error type reaches the response
async fn protected(claims: Claims) -> Result<String, jsonwebtoken::errors::Error> {
    // ...
}
```

Symptom: no compiler error if `Error` implements `IntoResponse` somewhere, but the
response body exposes internal details such as the algorithm in use or the failure
kind, which helps an attacker.

WHY it fails: internal crypto error detail is not safe to expose. It narrows an
attacker's guessing space and couples the public API to a dependency's error type.

Fix: collapse every `encode` / `decode` error into an `AuthError` variant with
`.map_err(|_| AuthError::InvalidToken)` (or `TokenCreation` for `encode`). The client
sees only a generic message and a correct status code.

## 8: Adding an authentication check inside the handler body

```rust
// WRONG : redundant and easy to forget
async fn protected(headers: HeaderMap) -> Result<String, AuthError> {
    let token = headers.get("authorization").ok_or(AuthError::MissingCredentials)?;
    // manual decode here ...
}
```

Symptom: no compiler error. The check is duplicated on every protected handler, and a
handler that forgets it is silently unprotected.

WHY it fails: authentication belongs in the extractor, not in handler bodies. Manual
per-handler checks are repetitive and the type system cannot enforce that every
protected route includes one.

Fix: take `claims: Claims` as a handler argument and let the `FromRequestParts`
extractor enforce the token. A protected route is then visible in its signature, and a
route without the `Claims` argument is unambiguously public.

## 9: Forgetting the jsonwebtoken crypto backend feature

```toml
# WRONG : no backend feature selected
jsonwebtoken = "10"
```

Symptom: a link or compile error about a missing crypto backend, or unresolved symbols
from the signing functions.

WHY it fails: `jsonwebtoken` v10 selects its cryptography backend by feature flag.
Exactly one of `aws_lc_rs` or `rust_crypto` must be enabled.

Fix: `jsonwebtoken = { version = "10", features = ["aws_lc_rs"] }` (the official
example's choice), or `features = ["rust_crypto"]`.
