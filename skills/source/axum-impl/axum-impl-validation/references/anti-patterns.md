# axum-impl-validation : Anti-Patterns

Real mistakes in Axum request-body validation, each with a root-cause analysis
and the correct form. The `validator` crate is version `0.20`.

## AP-1 : returning 400 for a validation failure

### Wrong

```rust
// WRONG - a domain-rule failure is not a 400.
value.validate().map_err(|e| {
    (StatusCode::BAD_REQUEST, e.to_string())
})?;
```

### Why it fails

`400 Bad Request` means the server could not parse the request. A body that
deserialized cleanly into `T` parsed fine; the server understood it perfectly.
An `age` of 9000 or an empty `first_name` is well-formed input that breaks a
domain rule. Reporting that as `400` tells the client "your JSON is broken"
when the JSON is correct, so the client cannot tell a syntax error from a
semantic one and retries blindly.

### Correct

```rust
// CORRECT - 422 means well-formed but semantically invalid.
value.validate().map_err(|e| {
    (StatusCode::UNPROCESSABLE_ENTITY, e.to_string())
})?;
```

`422 Unprocessable Entity` is the status for a request that parsed but whose
content violates the rules. The `ValidatedJson` extractor keeps the two apart:
a `JsonRejection` (real parse failure) passes through with its own 400-class
status, a `ValidationErrors` maps to `422`.

## AP-2 : validating inside the handler

### Wrong

```rust
// WRONG - validation logic spread across every handler.
async fn signup(Json(data): Json<SignupData>) -> Result<StatusCode, Response> {
    data.validate().map_err(|e| {
        (StatusCode::UNPROCESSABLE_ENTITY, e.to_string()).into_response()
    })?;
    // ... use data ...
    Ok(StatusCode::CREATED)
}
```

### Why it fails

The check works, but nothing enforces it. The handler signature
`Json<SignupData>` says "deserialized", not "validated". Every new handler that
takes `Json<SignupData>` must remember to call `.validate()`; the first one
that forgets ships unvalidated data straight into the database, and the
compiler stays silent. Validation logic is also duplicated across handlers.

### Correct

```rust
// CORRECT - the extractor validates; the handler signature proves it.
async fn signup(ValidatedJson(data): ValidatedJson<SignupData>) -> StatusCode {
    // `data` cannot be unvalidated here: the extractor rejected the request
    // before this function was ever called.
    let _ = data;
    StatusCode::CREATED
}
```

Putting `.validate()` in the `ValidatedJson<T>` extractor makes "this value is
validated" a type-level guarantee. A handler taking `ValidatedJson<T>` cannot
run on bad data.

## AP-3 : compiling a regex on every request

### Wrong

```rust
// WRONG - Regex::new runs on every single request.
fn validate_slug(slug: &str) -> Result<(), ValidationError> {
    let re = Regex::new(r"^[a-z0-9-]+$").unwrap();
    if re.is_match(slug) { Ok(()) } else { Err(ValidationError::new("bad_slug")) }
}
```

### Why it fails

`Regex::new` parses and compiles the pattern into a state machine. That is
expensive work, and placing it inside a per-request validator repeats it on
every call on the hot path. The `validator` `regex` attribute is built for the
correct form: it takes `path = ...` pointing at an already-compiled `Regex`.

### Correct

```rust
// CORRECT - compiled once via a LazyLock static.
use std::sync::LazyLock;
use regex::Regex;

static SLUG_RE: LazyLock<Regex> =
    LazyLock::new(|| Regex::new(r"^[a-z0-9-]+$").unwrap());

#[derive(Validate, serde::Deserialize)]
struct Article {
    #[validate(regex(path = *SLUG_RE))]
    slug: String,
}
```

The `Regex` is compiled once, on first use, and reused for every request.

## AP-4 : ValidatedJson placed before another extractor

### Wrong

```rust
// WRONG - a body extractor is not the last argument.
async fn create_order(
    ValidatedJson(order): ValidatedJson<Order>,
    State(state): State<AppState>,
) -> impl IntoResponse {
    StatusCode::CREATED
}
```

### Why it fails

`ValidatedJson<T>` consumes the request body, so it implements `FromRequest`,
not `FromRequestParts`. The request body is a single-use stream: only one
`FromRequest` extractor can run, and it must be last. This code does not
compile; the error is a `Handler`-trait-bound failure that does not obviously
point at argument order.

### Correct

```rust
// CORRECT - FromRequestParts extractors first, the body extractor last.
async fn create_order(
    State(state): State<AppState>,        // FromRequestParts
    ValidatedJson(order): ValidatedJson<Order>,   // FromRequest -> last
) -> impl IntoResponse {
    let _ = (state, order);
    StatusCode::CREATED
}
```

## AP-5 : omitting the derive feature

### Wrong

```toml
# WRONG - the `derive` feature is missing.
validator = "0.20"
```

```rust
use validator::Validate;

#[derive(Validate)]   // error: cannot find derive macro `Validate`
struct SignupData { /* ... */ }
```

### Why it fails

The `Validate` derive macro lives in the companion crate `validator_derive`. It
is re-exported by the `validator` crate ONLY when the `derive` feature is on.
Without the feature, `validator::Validate` resolves to the trait but not the
derive macro, and `#[derive(Validate)]` fails to resolve.

### Correct

```toml
# CORRECT - the derive feature re-exports the macro.
validator = { version = "0.20", features = ["derive"] }
```

NEVER add `validator_derive` as a separate dependency to work around this;
enable the `derive` feature on `validator` instead.

## AP-6 : nested on a field whose type does not derive Validate

### Wrong

```rust
// WRONG - Address does not derive Validate.
#[derive(serde::Deserialize)]
struct Address {
    street: String,
}

#[derive(Validate, serde::Deserialize)]
struct Order {
    #[validate(nested)]   // error: Address does not implement Validate
    address: Address,
}
```

### Why it fails

`#[validate(nested)]` recursively calls `.validate()` on the field. That
requires the field's type to implement `Validate`. A plain struct that only
derives `Deserialize` has no `.validate()` method, so the macro-generated code
fails to compile.

### Correct

```rust
// CORRECT - the nested type also derives Validate.
#[derive(Validate, serde::Deserialize)]
struct Address {
    #[validate(length(min = 1, max = 80))]
    street: String,
}

#[derive(Validate, serde::Deserialize)]
struct Order {
    #[validate(nested)]
    address: Address,
}
```

A `nested` field's type MUST derive `Validate`, and its own fields should
carry their own `#[validate(...)]` rules; otherwise nesting validates nothing.

## AP-7 : ignoring the validate() Result

### Wrong

```rust
// WRONG - the Result is discarded; validation does nothing.
async fn signup(Json(data): Json<SignupData>) -> StatusCode {
    data.validate();          // warning ignored, errors thrown away
    StatusCode::CREATED
}
```

### Why it fails

`.validate()` returns `Result<(), ValidationErrors>`. It never panics and never
mutates anything; the ONLY signal of a failure is the returned `Err`. Calling
it and dropping the value runs the checks and then discards the verdict, so
invalid data proceeds exactly as if it had passed. The compiler emits an
`unused must_use` warning, which is easy to miss.

### Correct

Run validation in the `ValidatedJson<T>` extractor (see AP-2), where the `?`
operator propagates the `Err` into a `422` rejection and the request never
reaches the handler with unvalidated data.

## AP-8 : keeping #[async_trait] when porting to Axum 0.8

### Wrong

```rust
// WRONG on axum 0.8 - #[async_trait] was removed from FromRequest.
#[async_trait]
impl<S, T> FromRequest<S> for ValidatedJson<T>
where /* ... */
{ /* ... */ }
```

### Why it fails

Axum 0.8 changed `FromRequest` to use native return-position `impl Trait` in
traits (stable since Rust 1.75) and removed the `#[async_trait]` requirement.
Applying `#[async_trait]` to an impl of the 0.8 trait produces a desugared
`Pin<Box<dyn Future>>` signature that no longer matches the trait, so the impl
fails to satisfy `FromRequest`.

### Correct

On Axum 0.8, delete the `#[async_trait]` attribute and drop the `async-trait`
crate from `Cargo.toml`. On Axum 0.7, keep both. This single line is the only
difference in the `ValidatedJson` extractor between the two versions.

## Sources

All verified via WebFetch on 2026-05-20.

- https://docs.rs/validator/latest/validator/ : `validator` 0.20, the `derive`
  feature requirement, the `regex(path = ...)` and `nested` attribute syntax,
  `.validate()` returning `Result<(), ValidationErrors>`.
- https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html : the
  single-`FromRequest`-extractor rule, body extractors last, the 0.7-to-0.8
  `#[async_trait]` removal.
