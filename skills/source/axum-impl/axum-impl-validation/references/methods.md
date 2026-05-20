# axum-impl-validation : Methods and API Signatures

Complete API signatures for request-body validation in Axum. The `validator`
crate is version `0.20`; all attribute syntax verified against the `validator`
0.20 docs.rs page and crate README on 2026-05-20.

## The validator crate : setup

```toml
# Cargo.toml - the `derive` feature re-exports the Validate derive macro.
validator = { version = "0.20", features = ["derive"] }
```

Feature flags:

| Feature | Enables |
|---------|---------|
| `derive` | the `#[derive(Validate)]` macro (required for every example) |
| `card` | the `credit_card` validator |
| `unic` | the `non_control_character` validator |

The `Validate` derive macro lives in the companion crate `validator_derive`
0.20, but it is re-exported behind the `derive` feature; NEVER depend on
`validator_derive` directly.

## The Validate trait

```rust
// validator 0.20
pub trait Validate {
    fn validate(&self) -> Result<(), ValidationErrors>;
}
```

`#[derive(Validate)]` generates this impl for a struct. `.validate()` returns
`Ok(())` when every field passes and `Err(ValidationErrors)` when one or more
fail. The documented usage form:

```rust
// validator 0.20 - verified from docs.rs
match signup_data.validate() {
    Ok(_) => (),
    Err(e) => return e,
};
```

## Error types

### ValidationError

A single validation failure.

```rust
// validator 0.20
impl ValidationError {
    pub fn new(code: &'static str) -> ValidationError;
}
```

Constructed inside a `custom` validator function with
`ValidationError::new("code")`. The `code` is a stable string identifier for
the failure.

### ValidationErrors

The aggregate of every failure across every field, returned by `.validate()`.

```rust
// validator 0.20 - relevant trait impls and accessors
impl std::fmt::Display for ValidationErrors { /* human-readable summary */ }
impl serde::Serialize for ValidationErrors { /* field-name -> error list */ }

impl ValidationErrors {
    pub fn errors(&self) -> &BTreeMap<&'static str, ValidationErrorsKind>;
    pub fn field_errors(&self) -> BTreeMap<&'static str, &Vec<ValidationError>>;
}
```

Because `ValidationErrors` is `Display`, `e.to_string()` yields a flat message.
Because it is `Serialize`, `Json(e)` yields a structured per-field error map.
The structured form is the recommended response body.

## The #[validate(...)] attribute set

Every attribute below was verified against the `validator` 0.20 docs and the
crate README.

| Attribute | Full syntax | Applies to |
|-----------|-------------|------------|
| `email` | `#[validate(email)]` | `String` |
| `url` | `#[validate(url)]` | `String` |
| `length` | `#[validate(length(min = 1, max = 100))]`, also `equal = N`, const-name strings `min = "MIN"` | string / collection |
| `range` | `#[validate(range(min = 18, max = 65))]`, also `exclusive_min`, `exclusive_max`, const-name strings | numeric |
| `must_match` | `#[validate(must_match(other = "password2"))]` | any `PartialEq` field |
| `contains` | `#[validate(contains(pattern = "term"))]` | string / collection |
| `does_not_contain` | `#[validate(does_not_contain(pattern = "term"))]` | string / collection |
| `regex` | `#[validate(regex(path = *RE_STATIC))]` | `String` |
| `custom` | `#[validate(custom(function = "validate_fn"))]`, optional `use_context` | any field |
| `nested` | `#[validate(nested)]` | a field whose type derives `Validate` |
| `required` | `#[validate(required)]` | `Option<T>` |
| `credit_card` | `#[validate(credit_card)]` (needs `card` feature) | `String` |
| `non_control_character` | `#[validate(non_control_character)]` (needs `unic` feature) | `String` |

Syntax facts:

- `custom` takes `function = "..."` as a string identifier naming a free
  function; the path form `function = "::utils::validate_x"` is also accepted.
  The function signature is `fn(&T) -> Result<(), ValidationError>`. The
  optional `use_context` parameter passes a context value into the function.
- `regex` takes `path = *RE` pointing at a lazily-initialised `Regex` static
  (`LazyLock<Regex>` or `once_cell::sync::Lazy<Regex>`), NEVER an inline string.
- `must_match` takes `other = "field_name"` naming the field to compare against.
- `nested` is a bare flag; the annotated field's type MUST itself derive
  `Validate`. Its errors are folded into the parent `ValidationErrors`.
- Multiple validators stack on one field, comma-separated inside one attribute:
  `#[validate(length(min = 1), custom(function = "validate_x"))]`.

## A custom validator function

```rust
// validator 0.20
fn validate_name(value: &str) -> Result<(), validator::ValidationError> {
    // return Err on failure, Ok(()) on success
    Ok(())
}
```

The argument type matches the annotated field's type by reference. The return
is always `Result<(), ValidationError>`.

## The FromRequest trait : the ValidatedJson extractor

`ValidatedJson<T>` consumes the request body, so it implements `FromRequest`,
NOT `FromRequestParts`. A body extractor MUST be the last handler argument.

### Axum 0.8 form

```rust
// axum 0.8 - native async fn in trait, NO #[async_trait]
pub trait FromRequest<S, M = private::ViaRequest>: Sized {
    type Rejection: IntoResponse;
    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection>;
}
```

### Axum 0.7 form

```rust
// axum 0.7 - the trait requires #[async_trait]
#[async_trait]
pub trait FromRequest<S, M = private::ViaRequest>: Sized {
    type Rejection: IntoResponse;
    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection>;
}
```

The skill's `ValidatedJson` impl mirrors this: `#[async_trait]` on the `impl`
block for 0.7, no attribute for 0.8. Axum 0.8 removed `#[async_trait]` from
`FromRequest` and `FromRequestParts` because return-position `impl Trait` in
traits stabilised in Rust 1.75.

## JsonRejection

The rejection type produced by the built-in `Json<T>` extractor when body
parsing fails. The `ValidatedJson` extractor delegates to `Json` and forwards
this rejection.

```rust
// axum 0.7 / 0.8 - JsonRejection accessors used by the extractor
impl JsonRejection {
    pub fn status(&self) -> StatusCode;   // a 400-class status
    pub fn body_text(&self) -> String;    // a human-readable reason
}
```

`JsonRejection` covers a malformed body, a missing or wrong `Content-Type`, and
a missing body. It keeps its own 4xx status; it is NOT remapped to `422`.

## The garde crate (context-aware alternative)

```toml
# garde - reach for this only when validation needs runtime context.
garde = { version = "0.x", features = ["derive"] }
```

```rust
// garde - context is a required argument of validate
trait Validate {
    type Context;
    fn validate_with(&self, ctx: &Self::Context) -> Result<(), Report>;
}
```

`garde` uses `#[garde(...)]` attributes (`#[garde(email)]`,
`#[garde(length(min = 1))]`, `#[garde(dive)]` for nested validation) and takes
a validation context as a required argument, so a validator may depend on a
database handle, a tenant, or a feature flag without global statics. Use
`validator` for stateless validation; use `garde` when context is required.

## Sources

All verified via WebFetch on 2026-05-20.

- https://docs.rs/validator/latest/validator/ : `validator` 0.20.0, the
  `Validate` trait and `.validate()` return type, `ValidationError` /
  `ValidationErrors`, the full attribute set, feature flags `derive` / `card` /
  `unic`.
- https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html : the
  `FromRequest` trait shape and the 0.7 versus 0.8 `#[async_trait]` difference.
- https://docs.rs/axum/latest/axum/extract/rejection/struct.JsonRejection.html :
  `JsonRejection`, `.status()`, `.body_text()`.
