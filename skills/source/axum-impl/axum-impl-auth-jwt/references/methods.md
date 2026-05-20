# Axum JWT Authentication : API Reference

All signatures verified against the `jsonwebtoken` v10.4.0 docs.rs reference and the
`tokio-rs/axum` `examples/jwt` crate on 2026-05-20.

## Crate dependencies

Verified from `examples/jwt/Cargo.toml`:

```toml
axum = "0.8"
axum-extra = { version = "0.10", features = ["typed-header"] }
jsonwebtoken = { version = "10", features = ["aws_lc_rs"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

- `jsonwebtoken` v10 selects its crypto backend by feature flag. `aws_lc_rs` is the
  example's choice; `rust_crypto` is the alternative. Exactly one must be enabled.
- `axum-extra` with the `typed-header` feature provides `Authorization<Bearer>`.

## jsonwebtoken: encode

```rust
pub fn encode<T: Serialize>(
    header: &Header,
    claims: &T,
    key: &EncodingKey,
) -> Result<String>
```

Serializes `header` and `claims`, signs the payload with `key`, and returns the compact
JWT string. `Result` is `jsonwebtoken::errors::Result`, whose error type is
`jsonwebtoken::errors::Error`.

## jsonwebtoken: decode

```rust
pub fn decode<T: DeserializeOwned>(
    token: impl AsRef<[u8]>,
    key: &DecodingKey,
    validation: &Validation,
) -> Result<TokenData<T>>
```

Decodes and validates a JWT. Returns `TokenData<T>` on success. The claims are read via
`token_data.claims` and the header via `token_data.header`. Validation (expiry, required
claims, algorithm) is applied per the `validation` argument.

## jsonwebtoken: Header

```rust
impl Default for Header // alg = HS256, typ = "JWT", all other fields None
```

`Header::default()` selects the HS256 algorithm. Use a non-default `Header` only to
change `alg` (for example to RS256) or to set `kid`.

## jsonwebtoken: EncodingKey and DecodingKey

Symmetric (HS256, HS384, HS512), used by this skill:

```rust
impl EncodingKey {
    pub fn from_secret(secret: &[u8]) -> Self;
}
impl DecodingKey {
    pub fn from_secret(secret: &[u8]) -> Self;
}
```

Both take the same shared secret because an HMAC signature is verified with the key that
produced it.

Asymmetric constructors (RS256, ES256, EdDSA), mentioned for completeness only:

```rust
EncodingKey::from_rsa_pem(&[u8]) -> Result<Self>
EncodingKey::from_ec_pem(&[u8])  -> Result<Self>
EncodingKey::from_ed_pem(&[u8])  -> Result<Self>
DecodingKey::from_rsa_pem(&[u8]) -> Result<Self>
DecodingKey::from_ec_pem(&[u8])  -> Result<Self>
DecodingKey::from_ed_pem(&[u8])  -> Result<Self>
```

DER variants (`from_rsa_der`, `from_ec_der`, `from_ed_der`) exist as well.

## jsonwebtoken: Validation

```rust
impl Default for Validation
```

Verified `Validation::default()` field values:

| Field | Default | Effect |
|-------|---------|--------|
| `validate_exp` | `true` | expiry IS checked |
| `required_spec_claims` | `{"exp"}` | a token with no `exp` claim is rejected |
| `leeway` | `60` | 60 seconds of clock-skew tolerance |
| `reject_tokens_expiring_in_less_than` | `0` | no early-expiry rejection |
| `validate_nbf` | `false` | `nbf` not checked unless opted in |
| `validate_aud` | `true` | audience checked when `aud` is set |
| `algorithms` | `vec![Algorithm::HS256]` | only HS256 accepted |
| `aud`, `iss`, `sub` | `None` | no audience/issuer/subject constraint |

Configuration methods:

```rust
impl Validation {
    pub fn new(alg: Algorithm) -> Self;
    pub fn set_audience<T: ToString>(&mut self, items: &[T]);
    pub fn set_issuer<T: ToString>(&mut self, items: &[T]);
    pub fn set_required_spec_claims<T: ToString>(&mut self, items: &[T]);
}
```

`Validation::default()` requires `exp`. The `Claims` struct MUST carry an `exp: usize`
field set to a real future Unix timestamp.

## jsonwebtoken: TokenData

```rust
pub struct TokenData<T> {
    pub header: Header,
    pub claims: T,
}
```

The success type of `decode`. The deserialized claims are `token_data.claims`.

## axum: FromRequestParts

```rust
// axum 0.8 : native async fn in trait
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;
    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection>;
}
```

```rust
// axum 0.7 : the impl block needs #[async_trait]
#[async_trait]
impl<S> FromRequestParts<S> for Claims { /* ... */ }
```

Axum 0.8 removed `#[async_trait]` from `FromRequest` and `FromRequestParts` in favor of
native return-position `impl Trait` in traits (RPITIT, MSRV Rust 1.75). The attribute is
required on 0.7 and rejected on 0.8.

`type Rejection` MUST implement `IntoResponse`. `AuthError` satisfies this.

## axum: RequestPartsExt::extract

```rust
pub trait RequestPartsExt {
    fn extract<E>(&mut self) -> impl Future<Output = Result<E, E::Rejection>>
    where
        E: FromRequestParts<()>;
}
```

`parts.extract::<TypedHeader<Authorization<Bearer>>>()` runs a sub-extractor against the
request parts inside another extractor. Requires `use axum::RequestPartsExt;`.

## axum-extra: typed Authorization header

```rust
use axum_extra::headers::{authorization::Bearer, Authorization};
use axum_extra::TypedHeader;

// after extraction:
let TypedHeader(Authorization(bearer)) = /* extracted */;
let token: &str = bearer.token();
```

`bearer.token()` returns the raw token string with the `Bearer ` prefix already
stripped. On Axum 0.7, `TypedHeader` is `axum::TypedHeader` behind the `headers`
feature; on Axum 0.8 it is `axum_extra::TypedHeader` with the `typed-header` feature.
