---
name: axum-impl-validation
description: >
  Use when validating an incoming JSON request body in an Axum handler: an email
  field, a string length bound, a numeric range, a password-confirm match, or any
  domain rule the value must satisfy before the handler trusts it.
  Prevents accepting semantically invalid input, returning a 400 where a 422
  belongs, scattering validate() calls across every handler, and the runtime
  panic from compiling a regex on every request.
  Covers the validator crate #[derive(Validate)] attributes, a ValidatedJson<T>
  custom FromRequest extractor that deserializes then runs .validate() and
  returns 422, the structured ValidationErrors JSON body, garde as a
  context-aware alternative, and the Axum 0.7 versus 0.8 #[async_trait]
  difference on the extractor impl.
  Keywords: axum validation, validator crate, derive Validate, ValidationError,
  ValidationErrors, ValidatedJson, FromRequest extractor, JsonRejection,
  422 Unprocessable Entity, validate email, length range custom regex nested,
  must_match, garde, async_trait, invalid input accepted, returned 400 instead
  of 422, validation not running, bad data reaches the database, how do I
  validate a request body, what is request validation.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-impl-validation

## Overview

Axum has no built-in request-body validation. A handler argument of `Json<T>`
guarantees only that the body was syntactically valid JSON and deserialized
into `T`. It does NOT guarantee the values make sense: an empty name, an email
with no `@`, an age of 9000 all deserialize cleanly.

The idiomatic solution is the `validator` crate plus a small custom extractor.
`validator` supplies a `#[derive(Validate)]` macro and a `.validate()` method;
the custom `ValidatedJson<T>` extractor deserializes the body, runs
`.validate()`, and returns `422 Unprocessable Entity` on failure. Putting the
check in the extractor makes "this value is validated" a type-level guarantee:
if the handler argument is `ValidatedJson<T>`, the data is already correct.

The `validator` crate is version `0.20`. The `garde` crate is a context-aware
alternative covered in the decision tree below. The `ValidatedJson` extractor
diverges between Axum 0.7 and 0.8 by exactly one line (`#[async_trait]`).

## Quick Reference

Cargo dependency (the `derive` feature re-exports the `Validate` macro):

```toml
# validator 0.20 - the derive feature re-exports the Validate derive macro
validator = { version = "0.20", features = ["derive"] }
```

`#[validate(...)]` attributes, all verified against the `validator` 0.20 docs:

| Attribute | Syntax | Checks |
|-----------|--------|--------|
| `email` | `#[validate(email)]` | HTML5-spec email format |
| `url` | `#[validate(url)]` | a parseable URL |
| `length` | `#[validate(length(min = 1, max = 100))]` | string / collection length |
| `range` | `#[validate(range(min = 18, max = 65))]` | numeric bounds |
| `must_match` | `#[validate(must_match(other = "password2"))]` | two fields are equal |
| `contains` | `#[validate(contains(pattern = "x"))]` | substring / element present |
| `does_not_contain` | `#[validate(does_not_contain(pattern = "x"))]` | substring / element absent |
| `regex` | `#[validate(regex(path = *RE))]` | matches a compiled `Regex` |
| `custom` | `#[validate(custom(function = "validate_fn"))]` | a user function |
| `nested` | `#[validate(nested)]` | recursively validates a sub-struct |
| `required` | `#[validate(required)]` | an `Option` field is `Some` |
| `credit_card` | `#[validate(credit_card)]` (needs `card` feature) | a card number |
| `non_control_character` | needs `unic` feature | no control chars |

Core rules:

- ALWAYS validate a request body through a custom extractor, NEVER inside the
  handler body.
- ALWAYS return `422 Unprocessable Entity` for a validation failure, NEVER
  `400`. `400` is for malformed JSON; `422` is for well-formed-but-invalid.
- ALWAYS make a body extractor (`ValidatedJson<T>`, `Json<T>`) the LAST handler
  argument. It consumes the single-use body stream.
- ALWAYS point a `regex` validator at a `LazyLock<Regex>` static, NEVER compile
  the regex inline per request.
- ALWAYS add `#[async_trait]` to the `FromRequest` impl on Axum 0.7; NEVER add
  it on Axum 0.8.
- NEVER call `.validate()` and ignore the `Result`. It returns
  `Result<(), ValidationErrors>` and must be propagated.

## Decision Trees

### Which validation crate to use

```
Does a validator need runtime state (a DB handle, a tenant, a feature flag)?
  NO  -> validator 0.20    (the widely-adopted default; every example here)
  YES -> garde             (first-class context: validate(&ctx) takes a context)
```

`validator` can pass context via the optional `use_context` parameter on a
`custom` validator, but `garde` makes a validation context a required argument
of `validate`, so a check can depend on runtime state without global statics.
For a stateless API, ALWAYS pick `validator`: it is the default the rest of
this skill builds on.

### Where to run validation

```
Where should .validate() be called?
  inside each handler body    -> WRONG: logic is duplicated and easily forgotten
  inside a ValidatedJson<T>   -> CORRECT: validation is a type-level guarantee
      custom FromRequest extractor
```

A handler taking `ValidatedJson<SignupData>` cannot run with unvalidated data:
the extractor rejected the request before the handler was ever called. A
handler taking `Json<SignupData>` and calling `data.validate()?` works, but the
check is invisible in the signature and one forgotten call ships bad data.

### Which status code for a validation failure

```
What stage failed?
  body is not valid JSON, wrong Content-Type, missing body
      -> JsonRejection keeps its own 4xx (a 400-class status)
  body parsed into T fine, but a value breaks a domain rule
      -> 422 Unprocessable Entity
```

`400 Bad Request` means the server could not parse the request. `422` means
the request parsed but its content is semantically wrong. The `ValidatedJson`
extractor MUST keep the two apart: a `JsonRejection` passes through with its
own status, a `ValidationErrors` maps to `422`.

## Patterns

### Pattern: a struct with #[derive(Validate)]

`#[derive(Validate)]` implements the `Validate` trait. Stack multiple
validators on one field by comma-separating them inside one `#[validate(...)]`.
Derive `Deserialize` on the same struct so it doubles as the request body.

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
```

### Pattern: a custom validator function

A `custom` validator is a free function returning
`Result<(), ValidationError>`. Construct the error with
`ValidationError::new("code")`. The string code identifies the failure to the
client.

```rust
// validator 0.20 - verified from docs.rs
fn validate_unique_username(username: &str) -> Result<(), ValidationError> {
    if username == "xXxShad0wxXx" {
        return Err(ValidationError::new("terrible_username"));
    }
    Ok(())
}
```

### Pattern: the ValidatedJson<T> extractor (Axum 0.8)

Axum exposes no validation hook, so wrap `Json` in a custom `FromRequest`
extractor. It delegates body parsing to the built-in `Json` extractor (and
forwards any `JsonRejection` with its own status), then runs `.validate()` and
maps a `ValidationErrors` to `422`. Because it consumes the body it implements
`FromRequest`, NOT `FromRequestParts`, so it must be the last handler argument.

```rust
// axum 0.8 + validator 0.20 - native async fn in trait, NO #[async_trait]
use axum::extract::{FromRequest, Request};
use axum::extract::rejection::JsonRejection;
use axum::http::StatusCode;
use axum::Json;
use serde_json::{json, Value};

pub struct ValidatedJson<T>(pub T);

impl<S, T> FromRequest<S> for ValidatedJson<T>
where
    S: Send + Sync,
    T: serde::de::DeserializeOwned + validator::Validate,
    Json<T>: FromRequest<S, Rejection = JsonRejection>,
{
    type Rejection = (StatusCode, Json<Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        // 1. delegate JSON deserialization; a JsonRejection keeps its own status
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|rej| (rej.status(), Json(json!({ "error": rej.body_text() }))))?;

        // 2. run the validator; a ValidationErrors maps to 422
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

### Pattern: the ValidatedJson<T> extractor (Axum 0.7)

Axum 0.7's `FromRequest` trait uses `#[async_trait]`; Axum 0.8 removed it in
favour of native return-position `impl Trait` in traits. The ONLY difference is
the attribute on the `impl` block. The body of `from_request` is byte-for-byte
identical.

```rust
// axum 0.7 + validator 0.20 - identical body, plus #[async_trait]
use async_trait::async_trait;

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

Migration rule: porting `ValidatedJson<T>` from 0.7 to 0.8 means deleting the
`#[async_trait]` line and dropping the `async-trait` dependency. Nothing else
changes.

### Pattern: using the extractor in a handler

`data` is guaranteed deserialized AND validated inside the handler. Because
`ValidatedJson` is a body extractor it MUST come last; a `FromRequestParts`
extractor such as `State` comes before it.

```rust
// axum 0.7 / 0.8 - the handler signature is version-stable
use axum::response::IntoResponse;
use axum::http::StatusCode;

async fn signup(ValidatedJson(data): ValidatedJson<SignupData>) -> impl IntoResponse {
    // `data` is already deserialized and validated here
    StatusCode::CREATED
}

// State first (FromRequestParts), ValidatedJson last (FromRequest):
async fn signup_with_state(
    State(db): State<Db>,
    ValidatedJson(data): ValidatedJson<SignupData>,
) -> impl IntoResponse {
    StatusCode::CREATED
}
```

Placing `ValidatedJson` before `State` fails to compile: only the last
argument may be a `FromRequest` extractor.

### Pattern: a structured 422 body

`ValidationErrors` implements `Serialize`, so returning it directly produces a
per-field error map instead of one flat string. This is far more actionable
for an API client. Replace step 2's `map_err` with:

```rust
// axum 0.7 / 0.8 - structured 422 body keyed by field name
value.validate().map_err(|e: validator::ValidationErrors| {
    (StatusCode::UNPROCESSABLE_ENTITY, Json(e))
})?;
```

The response JSON is an object whose keys are field names and whose values
list each failed validation code.

### Pattern: nested struct validation

`#[validate(nested)]` recursively validates a sub-struct field. The annotated
field's type MUST itself derive `Validate`; its errors are folded into the
parent `ValidationErrors`.

```rust
// validator 0.20 - the `nested` flag recurses into a sub-struct
#[derive(Validate, serde::Deserialize)]
struct Address {
    #[validate(length(min = 1, max = 80))]
    street: String,
}

#[derive(Validate, serde::Deserialize)]
struct Order {
    #[validate(length(min = 1))]
    sku: String,
    #[validate(nested)]
    address: Address,
}
```

### Pattern: a regex validator with a static Regex

A `regex` validator takes `path = ...` pointing at a compiled `Regex`, NOT an
inline string. ALWAYS store it in a `LazyLock<Regex>` static so it is compiled
once, never per request.

```rust
// validator 0.20 - regex points at a LazyLock<Regex> static
use std::sync::LazyLock;          // Rust 1.80+
use regex::Regex;

static POSTCODE_RE: LazyLock<Regex> =
    LazyLock::new(|| Regex::new(r"^\d{4}\s?[A-Z]{2}$").unwrap());

#[derive(Validate, serde::Deserialize)]
struct Delivery {
    #[validate(regex(path = *POSTCODE_RE))]
    postcode: String,
}
```

## Reference Links

- `references/methods.md`: the `Validate` trait and `.validate()` signature,
  `ValidationError` and `ValidationErrors`, every `#[validate(...)]` attribute
  with its full parameter syntax, and the `FromRequest` trait shape the
  `ValidatedJson` extractor implements.
- `references/examples.md`: a complete `Cargo.toml`, a full multi-rule struct,
  custom validators, nested and regex validation, both versions of the
  `ValidatedJson` extractor end to end, the structured 422 body, the handler
  wiring, and the `garde` context-aware alternative.
- `references/anti-patterns.md`: real mistakes with root-cause analysis,
  including returning `400` for a validation failure, validating inside the
  handler, compiling a regex per request, putting `ValidatedJson` before
  `State`, omitting the `derive` feature, and `nested` on a non-`Validate`
  type.

Related skills: `axum-syntax-custom-extractors` (the `FromRequest` versus
`FromRequestParts` split and writing custom extractors), `axum-errors-extractor-rejections`
(`JsonRejection` and remapping rejection types), `axum-core-version-migration`
(the 0.7-to-0.8 `#[async_trait]` removal).
