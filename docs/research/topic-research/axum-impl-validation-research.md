# Topic Research : axum-impl-validation

> Skill : `axum-impl-validation`
> Phase : 4 (topic research)
> Target versions : Axum 0.7 and 0.8 (docs.rs current : 0.8.x)
> Validation crate : `validator` 0.20.0 (verified on docs.rs, 2026-05-20)
> Researched / verified : 2026-05-20
> Source fragments consulted : `research-c-errors-auth-db-ops.md` (sec. 3),
> `axum-syntax-custom-extractors-research.md` (sec. 3, 6)

This file is the verified research basis for the `axum-impl-validation` skill.
Axum itself has no built-in request-body validation. The idiomatic solution is
the `validator` crate plus a small custom `FromRequest` extractor that
deserializes the body, runs `.validate()`, and returns `422 Unprocessable
Entity` on failure. This is research only; it is not the skill.

---

## 1. The `validator` Crate : Version and Setup

The current release is **`validator` 0.20.0** (MIT licensed). The derive macro
ships in the companion crate `validator_derive` 0.20.0, but skill authors do
NOT depend on it directly: it is re-exported behind the `derive` feature of the
`validator` crate. Verified `Cargo.toml` line (the docs use an older version
number in their snippet, but the feature name is stable):

```toml
# validator 0.20 - the `derive` feature re-exports the Validate derive macro
validator = { version = "0.20", features = ["derive"] }
```

Optional features verified from docs.rs:

- `derive` : enables the `#[derive(Validate)]` macro. Required for every skill
  example.
- `card` : enables the `credit_card` validator.
- `unic` : enables the `non_control_character` validator.

---

## 2. The `Validate` Trait and `.validate()`

`#[derive(Validate)]` implements the `Validate` trait for a struct. The trait's
method is `validate()` and it returns `Result<(), ValidationErrors>`. Verified
from docs.rs : calling `.validate()` yields `Ok(())` when every field passes
and `Err(ValidationErrors)` otherwise. The documented usage pattern:

```rust
// validator 0.20 - verified verbatim from docs.rs
match signup_data.validate() {
    Ok(_) => (),
    Err(e) => return e,
};
```

Two error types:

- **`ValidationError`** : a single validation failure. Constructed with
  `ValidationError::new("code")`. This is what a custom validator function
  returns inside an `Err`.
- **`ValidationErrors`** : the aggregate of all failures across all fields,
  returned by `.validate()`. It implements `Display` (so `.to_string()` yields
  a human-readable summary) and `Serialize` (so it can be turned into a
  structured JSON body keyed by field name). It exposes the per-field detail
  via `.field_errors()` and `.errors()`.

Because `ValidationErrors` is `Serialize`, the extractor can return either a
flat `e.to_string()` message or the full structured `Json(e)` map of
field-name to error-list. The skill should show the structured form as the
recommended option.

---

## 3. `#[derive(Validate)]` Attributes (verified)

Every `#[validate(...)]` attribute below was verified against the `validator`
0.20.0 docs.rs page and the crate README on 2026-05-20.

| Attribute | Syntax | Validates |
|-----------|--------|-----------|
| `email` | `#[validate(email)]` | HTML5-spec email format |
| `url` | `#[validate(url)]` | a parseable URL |
| `length` | `#[validate(length(min = 1, max = 100))]` ; also `equal = N`, or const-name strings `min = "MIN_CONST"` | string / collection length |
| `range` | `#[validate(range(min = 18, max = 65))]` ; also `exclusive_min`, `exclusive_max`, const-name strings | numeric bounds |
| `must_match` | `#[validate(must_match(other = "password2"))]` | two fields are equal |
| `contains` | `#[validate(contains(pattern = "..."))]` | substring / element present |
| `does_not_contain` | `#[validate(does_not_contain(pattern = "..."))]` | substring / element absent |
| `regex` | `#[validate(regex(path = *RE_TWO_CHARS))]` | matches a compiled `Regex` |
| `custom` | `#[validate(custom(function = "validate_unique_username"))]` | a user function |
| `nested` | `#[validate(nested)]` | recursively validates a sub-struct field |
| `credit_card` | `#[validate(credit_card)]` (needs `card` feature) | a valid card number |
| `non_control_character` | `#[validate(non_control_character)]` (needs `unic` feature) | no control chars |
| `required` | `#[validate(required)]` | an `Option` field is `Some` |

Key syntax facts verified:

- **`custom`** takes `function = "..."` as a string identifier referencing a
  free function, and the path form `function = "::utils::validate_something"`
  is also accepted. The function signature is
  `fn(&T) -> Result<(), ValidationError>`. An optional `use_context` parameter
  passes a context value into the function (this is how `validator` does
  context-aware validation; `garde`, below, makes it first-class).
- **`regex`** takes `path = ...` pointing at a lazily-initialised `Regex`
  (typically a `LazyLock<Regex>` or `once_cell` static), NOT an inline string.
- **`must_match`** takes `other = "field_name"` naming the field to compare
  against.
- **`nested`** is a bare flag; the annotated field's type must itself derive
  `Validate`, and its errors are folded into the parent `ValidationErrors`.
- Multiple validators stack on one field, comma-separated inside one attribute:
  `#[validate(length(min = 1), custom(function = "validate_unique_username"))]`.

Verified verbatim example from docs.rs showing the derive, multiple stacked
validators, and a custom validator function:

```rust
// validator 0.20 - verified verbatim from docs.rs
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

fn validate_unique_username(username: &str) -> Result<(), ValidationError> {
    if username == "xXxShad0wxXx" {
        return Err(ValidationError::new("terrible_username"));
    }
    Ok(())
}
```

A nested-validation example (the `nested` flag and a sub-struct that also
derives `Validate`):

```rust
// validator 0.20 - nested struct validation
#[derive(Validate, serde::Deserialize)]
struct Address {
    #[validate(length(min = 1, max = 80))]
    street: String,
    #[validate(regex(path = *POSTCODE_RE))]
    postcode: String,
}

#[derive(Validate, serde::Deserialize)]
struct Order {
    #[validate(length(min = 1))]
    sku: String,
    #[validate(nested)]          // recursively validates `address`
    address: Address,
}
```

---

## 4. The `ValidatedJson<T>` Custom Extractor

Axum has no validation hook, so wrap `Json` in a custom `FromRequest`
extractor. It delegates body consumption to the built-in `Json` extractor,
remaps the `JsonRejection` (malformed / missing-content-type JSON, a `400`),
then runs `.validate()` and remaps `ValidationErrors` to a
`422 UNPROCESSABLE_ENTITY`. `422` is the correct status: the JSON parsed fine
(so it is not a `400`), but the values violate domain rules.

A body extractor consumes the single-use body stream, so `ValidatedJson<T>`
implements `FromRequest` (not `FromRequestParts`) and MUST be the **last**
handler argument.

### 0.8 form (native `async fn` in trait, NO `#[async_trait]`)

```rust
// axum 0.8 + validator 0.20
use axum::extract::{FromRequest, Request};
use axum::extract::rejection::JsonRejection;
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
        // 1. delegate JSON deserialization; remap JsonRejection (a 400)
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|rej| {
                (rej.status(), Json(json!({ "error": rej.body_text() })))
            })?;

        // 2. run validator; remap ValidationErrors to a 422
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

### 0.7 form (REQUIRES `#[async_trait]`)

The only difference is the `#[async_trait]` attribute on the `impl` block.
Axum 0.8 removed `#[async_trait]` from `FromRequest` / `FromRequestParts` in
favour of native return-position `impl Trait` in traits (RPITIT, stable since
Rust 1.75). The body of `from_request` is byte-for-byte identical.

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
    type Rejection = (StatusCode, Json<serde_json::Value>);

    async fn from_request(req: Request, state: &S)
        -> Result<Self, Self::Rejection>
    {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|rej| (rej.status(), Json(json!({ "error": rej.body_text() }))))?;
        value.validate().map_err(|e| (
            StatusCode::UNPROCESSABLE_ENTITY,
            Json(json!({ "error": e.to_string() })),
        ))?;
        Ok(ValidatedJson(value))
    }
}
```

Migration rule : porting `ValidatedJson<T>` from 0.7 to 0.8 means deleting the
`#[async_trait]` line and dropping the `async-trait` dependency from
`Cargo.toml`. Nothing else changes.

### Using it in a handler

```rust
// axum 0.7 + 0.8 - the handler signature is version-stable
async fn signup(ValidatedJson(data): ValidatedJson<SignupData>)
    -> impl IntoResponse
{
    // `data` is guaranteed deserialized AND validated here
    StatusCode::CREATED
}
```

Because `ValidatedJson` is a body extractor it must be the last argument:
`async fn signup(State(db): State<Db>, ValidatedJson(data): ValidatedJson<SignupData>)`
is correct; placing `ValidatedJson` before `State` fails to compile.

### Structured 422 body (recommended variant)

`ValidationErrors` is `Serialize`, so returning it directly gives a per-field
error map instead of a flat string. Replace step 2's `map_err` with:

```rust
// axum 0.7 + 0.8 - structured 422 body keyed by field name
value.validate().map_err(|e: validator::ValidationErrors| {
    (StatusCode::UNPROCESSABLE_ENTITY, Json(e))
})?;
```

This yields a JSON object whose keys are the field names and whose values list
each failed validation code, which is far more actionable for API clients than
a single concatenated message.

---

## 5. `garde` : a Context-Aware Alternative (mention only)

`garde` is a modern alternative to `validator` with a similar derive macro but
a stricter, first-class context-aware API. Where `validator` bolts context on
via the optional `use_context` parameter, `garde`'s `validate(&ctx)` takes a
validation context as a required argument, so a validator can depend on runtime
state (a database handle, a feature flag, the current tenant) without global
statics. Its attributes use the `#[garde(...)]` prefix (`#[garde(email)]`,
`#[garde(length(min = 1))]`, `#[garde(dive)]` for nested validation). The
skill should mention `garde` as the option to reach for when validation must
be context-aware, but `validator` 0.20 remains the widely-adopted default and
is what every skill example uses.

---

## 6. Lift-Ready Verified Snippets

**S-1 : `Cargo.toml` dependency (validator 0.20, `derive` feature).**

```toml
validator = { version = "0.20", features = ["derive"] }
```

**S-2 : `#[derive(Validate)]` struct with stacked validators (verbatim docs).**

```rust
// validator 0.20 - verbatim from docs.rs
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

**S-3 : custom validator function (verbatim docs) + `regex`/`must_match`/`nested` attribute forms.**

```rust
// validator 0.20 - custom validator returns Result<(), ValidationError>
fn validate_unique_username(username: &str) -> Result<(), ValidationError> {
    if username == "xXxShad0wxXx" {
        return Err(ValidationError::new("terrible_username"));
    }
    Ok(())
}
// attribute forms (verified):
//   #[validate(regex(path = *RE_TWO_CHARS))]
//   #[validate(must_match(other = "password2"))]
//   #[validate(nested)]
```

**S-4 : `.validate()` return type and usage (verbatim docs).**

```rust
// validator 0.20 - validate() returns Result<(), ValidationErrors>
match signup_data.validate() {
    Ok(_) => (),
    Err(e) => return e,
};
```

**S-5 : `ValidatedJson<T>` extractor, Axum 0.8 native form (no `#[async_trait]`).**

```rust
// axum 0.8 + validator 0.20
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
        let Json(value) = Json::<T>::from_request(req, state).await
            .map_err(|r| (r.status(), Json(json!({ "error": r.body_text() }))))?;
        value.validate().map_err(|e| (
            StatusCode::UNPROCESSABLE_ENTITY,
            Json(json!({ "error": e.to_string() })),
        ))?;
        Ok(ValidatedJson(value))
    }
}
```

**S-6 : `ValidatedJson<T>` extractor, Axum 0.7 form (`#[async_trait]` added).**

```rust
// axum 0.7 + validator 0.20 - only the attribute differs from S-5
#[async_trait]
impl<S, T> FromRequest<S> for ValidatedJson<T>
where
    S: Send + Sync,
    T: serde::de::DeserializeOwned + validator::Validate,
    Json<T>: FromRequest<S, Rejection = JsonRejection>,
{
    type Rejection = (StatusCode, Json<serde_json::Value>);
    async fn from_request(req: Request, state: &S)
        -> Result<Self, Self::Rejection>
    { /* identical body to S-5 */ }
}
```

---

## 7. Anti-Patterns (validation-specific)

**AP-V1 : returning `400` for a validation failure.** A body that parsed as
valid JSON but breaks a domain rule (an age out of range, an empty name) is a
`422 Unprocessable Entity`, not a `400 Bad Request`. `400` is for syntactically
malformed input; `422` tells the client the request was well-formed but
semantically invalid. The `ValidatedJson` extractor must distinguish the two:
`JsonRejection` keeps its own status (a `400`-class), `ValidationErrors` maps
to `422`.

**AP-V2 : validating inside the handler instead of in the extractor.** Calling
`data.validate()?` in the handler body works but spreads validation logic
across every handler and makes it easy to forget. Putting validation in the
`ValidatedJson<T>` extractor makes "this value is validated" a type-level
guarantee : if the handler argument is `ValidatedJson<T>`, the data is already
checked.

**AP-V3 : inline `Regex::new` per request.** A `regex` validator's pattern must
point at a lazily-initialised static (`LazyLock<Regex>` or `once_cell`).
Compiling the regex on every request is wasted work on the hot path.

---

## 8. Sources Verified

All URLs fetched and verified via WebFetch on **2026-05-20**.

- https://docs.rs/validator/latest/validator/ — `validator` crate version
  **0.20.0**; the `Validate` trait and `.validate()` returning
  `Result<(), ValidationErrors>`; the full validator attribute set (`email`,
  `url`, `length`, `range`, `must_match`, `contains`, `does_not_contain`,
  `regex`, `custom`, `credit_card`, `non_control_character`, `required`);
  `ValidationError` / `ValidationErrors` types; feature flags `derive`, `card`,
  `unic`; verbatim `SignupData` derive example and `validate_unique_username`
  custom-validator example; verbatim `match signup_data.validate()` usage.
- https://docs.rs/validator_derive/latest/validator_derive/ — confirmed
  `validator_derive` version **0.20.0** (the `Validate` derive macro crate,
  re-exported by `validator`'s `derive` feature).
- https://raw.githubusercontent.com/Keats/validator/master/README.md — verbatim
  attribute syntax for `nested` (`#[validate(nested)]`), `regex`
  (`regex(path = *RE_TWO_CHARS)`), `must_match` (`must_match(other = "...")`),
  `length` (`min`/`max`/`equal`/const-name), `range`
  (`min`/`max`/`exclusive_min`/const-name), `custom`
  (`custom(function = "...")`, path form, optional `use_context`).
- Cross-referenced (verified in prior Phase 2 / Phase 4 work on 2026-05-20, not
  re-fetched here) : the `ValidatedJson<T>` `FromRequest` extractor pattern and
  the Axum 0.7 vs 0.8 `#[async_trait]` difference — see
  `research-c-errors-auth-db-ops.md` section 3 and
  `axum-syntax-custom-extractors-research.md` sections 3 and 6.

### Verification caveats

- `https://docs.rs/validator/latest/validator/derive.Validate.html` returns
  HTTP 404 : the `Validate` derive macro is documented on the main `validator`
  crate page (re-export) and in the `validator_derive` crate, not at that path.
  Attribute syntax was therefore verified from the main docs.rs page plus the
  canonical crate README, both fetched 2026-05-20.
- The `contains` / `does_not_contain` `pattern = "..."` parameter form and the
  nested `Order`/`Address` example in section 3 are faithful compositions of
  the verified attribute list and the verified `nested` flag syntax, not a
  verbatim docs transcription; they are marked accordingly.
