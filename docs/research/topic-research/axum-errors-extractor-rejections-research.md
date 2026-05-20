# Topic Research : axum-errors-extractor-rejections

> Skill : `axum-errors-extractor-rejections`
> Category : errors
> Target versions : Axum 0.7 and 0.8 (axum-extra 0.9/0.10 for `WithRejection`)
> Researched : 2026-05-20
> Status : Phase 4 topic research, verified

This file supports a single skill covering extractor rejections in Axum: the
`Rejection` associated type, the `axum::extract::rejection` module inventory,
the `Result<T, T::Rejection>` handler-argument pattern, `axum-extra`'s
`WithRejection<E, R>`, derive-based custom rejections via
`#[from_request(rejection(...))]`, and the Axum 0.8 `Option<T>` extractor
behavior. All API claims below are WebFetch-verified against the official
sources listed at the bottom.

---

## 1. The `Rejection` associated type

Every extractor that can fail carries an associated **rejection type**. Both
extractor traits declare it, and the docs.rs trait definitions confirm the
rejection MUST itself implement `IntoResponse` so Axum can turn a failed
extraction directly into an HTTP response:

```rust
// axum 0.8 - verified verbatim from docs.rs
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;
    fn from_request_parts(parts: &mut Parts, state: &S)
        -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}

pub trait FromRequest<S, M = ViaRequest>: Sized {
    type Rejection: IntoResponse;
    fn from_request(req: Request<Body>, state: &S)
        -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

The `IntoResponse: Rejection` bound is the mechanism behind Axum's
infallible-handler model: a failed extraction never bubbles up as a
`tower::Service` error, it becomes a normal HTTP response. Built-in extractors
ship with predefined rejection types; a custom extractor declares its own
(commonly `(StatusCode, String)` or a JSON tuple).

---

## 2. The `axum::extract::rejection` module inventory

The `axum::extract::rejection` module holds every built-in rejection type.
The module mixes per-extractor **enums** (the top-level rejection returned by
an extractor) and granular **structs** (the individual failure variants).

**Enums (top-level extractor rejections):** `BytesRejection`,
`ExtensionRejection`, `FailedToBufferBody`, `FormRejection`, `JsonRejection`
(behind the `json` feature), `MatchedPathRejection` (behind the `matched-path`
feature), `PathRejection`, `QueryRejection`, `RawFormRejection`,
`RawPathParamsRejection`, `StringRejection`.

**Structs (granular failure types):** `FailedToDeserializeForm`,
`FailedToDeserializeFormBody`, `FailedToDeserializeQueryString`,
`InvalidFormContentType`, `InvalidUtf8`, `JsonDataError`, `JsonSyntaxError`,
`LengthLimitError`, `MatchedPathMissing`, `MissingExtension`,
`MissingJsonContentType`, `MissingPathParams`, `NestedPathRejection`,
`UnknownBodyError`.

Per-extractor mapping (verified):

| Extractor | Rejection enum | Notable inner types |
|-----------|----------------|---------------------|
| `Json<T>` | `JsonRejection` | `JsonDataError`, `JsonSyntaxError`, `MissingJsonContentType`, `BytesRejection` |
| `Path<T>` | `PathRejection` | failed-deserialize, invalid-UTF8, `MissingPathParams` |
| `Query<T>` | `QueryRejection` | `FailedToDeserializeQueryString` |
| `Form<T>` | `FormRejection` | `FailedToDeserializeForm`, `FailedToDeserializeFormBody`, `InvalidFormContentType` |
| `Bytes` | `BytesRejection` | `FailedToBufferBody` |
| `String` | `StringRejection` | `InvalidUtf8`, `FailedToBufferBody` |
| `Extension<T>` | `ExtensionRejection` | `MissingExtension` |
| `RawForm` | `RawFormRejection` | `InvalidFormContentType` |
| `MatchedPath` | `MatchedPathRejection` | `MatchedPathMissing` |
| `NestedPath` | `NestedPathRejection` | (struct, used directly) |

**CRITICAL RULE: every rejection enum is `#[non_exhaustive]`.** A `match` on
`JsonRejection`, `PathRejection`, or any other rejection enum MUST include a
catch-all `_ =>` arm, otherwise the code does not compile. The
`#[non_exhaustive]` marker exists so Axum can add new failure variants in a
minor release without it being a breaking change.

All rejection enums implement `Debug`, `Display`, `std::error::Error`,
`IntoResponse`, and (per variant) `From<InnerType>`. Each also exposes two
inherent methods used to drive a custom response:

- `pub fn status(&self) -> StatusCode` — the HTTP status Axum would have used.
- `pub fn body_text(&self) -> String` — the plain-text error message.

---

## 3. `JsonRejection` verified in detail

`JsonRejection` is the most-handled rejection and the canonical teaching
example. Verified facts:

- Marked `#[non_exhaustive]`.
- Variants (verbatim): `JsonDataError(JsonDataError)`,
  `JsonSyntaxError(JsonSyntaxError)`,
  `MissingJsonContentType(MissingJsonContentType)`,
  `BytesRejection(BytesRejection)`.
- Methods: `body_text(&self) -> String`, `status(&self) -> StatusCode`.
- Implements `Debug`, `Display`, `Error`, `IntoResponse`, plus
  `From<JsonDataError>`, `From<JsonSyntaxError>`,
  `From<MissingJsonContentType>`, `From<BytesRejection>`.

Variant meanings: `MissingJsonContentType` — the `Content-Type` header was not
`application/json`. `JsonSyntaxError` — the body is not syntactically valid
JSON. `JsonDataError` — valid JSON but it does not match the target type. The
`Error::source()` chain on `JsonSyntaxError` / `JsonDataError` leads to the
underlying `serde_json::Error`; you can walk that chain and `downcast` for
fine-grained diagnostics, though the official `customize-extractor-error`
example does NOT do this and relies on `body_text()` instead.

---

## 4. Customizing a rejection: the `Result<T, T::Rejection>` handler argument

The lightest-weight way to customize a rejection, requiring no custom extractor
and no extra crate: type the handler argument as `Result<Extractor, Rejection>`
instead of bare `Extractor`. Axum then hands you the `Result`, so you `match`
on the rejection inside the handler body:

```rust
// axum 0.8 - handle the rejection inline, no custom extractor needed
use axum::{extract::rejection::JsonRejection, Json};
use serde_json::Value;

async fn create(payload: Result<Json<Value>, JsonRejection>) -> Response {
    match payload {
        Ok(Json(value)) => /* use value */ todo!(),
        // #[non_exhaustive] -> the catch-all arm is mandatory
        Err(JsonRejection::MissingJsonContentType(_)) =>
            (StatusCode::UNSUPPORTED_MEDIA_TYPE, "expected application/json")
                .into_response(),
        Err(JsonRejection::JsonSyntaxError(_)) =>
            (StatusCode::BAD_REQUEST, "invalid JSON syntax").into_response(),
        Err(JsonRejection::JsonDataError(_)) =>
            (StatusCode::UNPROCESSABLE_ENTITY, "JSON does not match schema")
                .into_response(),
        Err(rejection) =>
            (rejection.status(), rejection.body_text()).into_response(),
    }
}
```

ALWAYS use this pattern when only one or two handlers need bespoke rejection
handling. NEVER scatter it across every handler in a large API — for codebase-
wide remapping use `WithRejection` (section 6) or a derived rejection
(section 7).

---

## 5. The verified `customize-extractor-error` example

The official `tokio-rs/axum` `examples/customize-extractor-error` crate is the
authoritative reference. Its `src/main.rs` declares three modules and registers
one POST route per module:

```rust
// examples/customize-extractor-error/src/main.rs - verified structure
mod custom_extractor;
mod derive_from_request;
mod with_rejection;

let app = Router::new()
    .route("/with-rejection",     post(with_rejection::handler))
    .route("/custom-extractor",   post(custom_extractor::handler))
    .route("/derive-from-request",post(derive_from_request::handler));
```

It binds `127.0.0.1:3000` and sets up `tracing_subscriber`. Each module shows a
different technique for the same goal — replacing Axum's default plain-text
rejection with a custom JSON error shape:

**`custom_extractor.rs`** — a hand-written wrapper extractor
`pub struct Json<T>(pub T)` with `Rejection = (StatusCode, axum::Json<Value>)`.
Its `from_request` extracts `MatchedPath` first, delegates to
`axum::Json::<T>::from_request`, and on failure builds a JSON payload:

```rust
// verified verbatim from custom_extractor.rs
let payload = json!({
    "message": rejection.body_text(),
    "origin": "custom_extractor",
    "path": path,
});
Err((rejection.status(), axum::Json(payload)))
```

This is the most flexible option (it can read other request parts such as
`MatchedPath`) but the most verbose.

**`with_rejection.rs`** and **`derive_from_request.rs`** are covered in
sections 6 and 7.

---

## 6. `axum-extra` `WithRejection<E, R>` for global rejection remapping

`axum-extra`'s `WithRejection<E, R>` is an extractor wrapper that runs extractor
`E` and, on failure, converts `E`'s rejection into your own type `R`. It
requires the `with-rejection` feature on `axum-extra`.

Generic parameters and bounds (verified):

- `E` — an extractor implementing `FromRequest<S>` or `FromRequestParts<S>`.
- `R` — the custom rejection: it MUST implement `IntoResponse` and
  `From<E::Rejection>`.

Used as a handler argument, you destructure both tuple fields; the first is the
inner `E`, the second a `PhantomData` placeholder discarded with `_`:

```rust
// verified verbatim shape from with_rejection.rs
use axum_extra::extract::WithRejection;
use axum::{extract::rejection::JsonRejection, Json};
use serde_json::{json, Value};
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ApiError {
    #[error(transparent)]
    JsonExtractorRejection(#[from] JsonRejection),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            ApiError::JsonExtractorRejection(json_rejection) =>
                (json_rejection.status(), json_rejection.body_text()),
        };
        let payload = json!({ "message": message, "origin": "with_rejection" });
        (status, Json(payload)).into_response()
    }
}

async fn handler(
    WithRejection(Json(value), _): WithRejection<Json<Value>, ApiError>,
) {
    // value is the successfully extracted Value
}
```

The `#[from]` attribute on the `thiserror` variant auto-generates the required
`From<JsonRejection> for ApiError` impl. ALWAYS prefer `WithRejection` over a
hand-written extractor when the only goal is remapping a built-in extractor's
rejection to a unified error type across many handlers.

---

## 7. Derive-based custom rejections (`#[from_request(rejection(...))]`)

`axum-macros` (re-exported as `axum::extract::FromRequest` /
`FromRequestParts` derives) supports building an extractor from another
extractor and routing its rejection through a custom type, with NO manual
trait `impl`. This is the `derive_from_request.rs` technique:

```rust
// verified shape from derive_from_request.rs
use axum::extract::FromRequest;

#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct Json<T>(pub T);

// ApiError is a plain struct here: { status: StatusCode, message: String }
impl From<JsonRejection> for ApiError {
    fn from(rejection: JsonRejection) -> Self {
        ApiError { status: rejection.status(), message: rejection.body_text() }
    }
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let payload = json!({
            "message": self.message,
            "origin": "derive_from_request",
        });
        (self.status, axum::Json(payload)).into_response()
    }
}
```

Two attribute keys matter: `via(...)` names the inner extractor the derive
delegates to, and `rejection(...)` names the rejection type that the inner
rejection is converted into (via `From`). The named rejection type MUST
implement `IntoResponse` and `From<InnerExtractor::Rejection>`. This derive is
also how a struct that bundles several extractors declares one unified
rejection type. Note: a body-consuming derived extractor uses
`#[derive(FromRequest)]`; a parts-only one uses `#[derive(FromRequestParts)]`.

**Manual-impl warning (verified from docs):** when implementing an extractor by
hand, do NOT implement both `FromRequest` and `FromRequestParts` for the same
concrete type — the official docs state this makes the extractor unusable. The
sole exception is a generic wrapper around another extractor, where
implementing both is correct and intended.

---

## 8. `OptionalFromRequest` and `Option<T>` extractor behavior (0.8)

In Axum 0.7, wrapping an extractor as `Option<T>` swallowed **all** failure
conditions and yielded `None`. Axum 0.8 changed this — verified verbatim 0.8.0
breaking change: "`Option<Path<T>>` no longer swallows all error conditions,
instead rejecting the request in many cases."

The mechanism is two dedicated traits, `OptionalFromRequestParts` and
`OptionalFromRequest`, that back `Option<T>` as an extractor. The semantics are
extractor-specific: a value that is genuinely **absent** yields `None`, while a
value that is **present but malformed** produces a rejection. For example, a
missing optional header is `None`, but a header present with an
undeserializable value rejects.

`OptionalFromRequest` impls for `Json` and `Extension` were added in a 0.8.x
patch release. MIGRATION RULE: after upgrading 0.7 → 0.8, audit every
`Option<Extractor>` handler argument — code that relied on the
swallow-everything behavior now returns a `4xx` rejection where it previously
returned `None`. To recover the old "treat any failure as absent" behavior,
explicitly use `Result<T, T::Rejection>` and map `Err(_)` to your own default.

---

## 9. Verified snippet inventory (for skill use)

The skill should carry these 6 verified snippets:

1. The `FromRequest` / `FromRequestParts` trait definitions with the
   `Rejection: IntoResponse` bound (section 1).
2. The `Result<Json<Value>, JsonRejection>` inline-match handler with the
   mandatory `#[non_exhaustive]` catch-all arm (section 4).
3. The `custom_extractor.rs` hand-written wrapper with
   `Rejection = (StatusCode, axum::Json<Value>)` and the `json!` payload
   (section 5).
4. The `WithRejection<Json<Value>, ApiError>` handler plus the `thiserror`
   `ApiError` enum with `#[from]` and `IntoResponse` (section 6).
5. The `#[derive(FromRequest)]` + `#[from_request(via(...), rejection(...))]`
   derived extractor with `From` and `IntoResponse` impls (section 7).
6. An `Option<Path<T>>` example contrasting 0.7 swallow-all vs 0.8
   reject-on-malformed behavior (section 8).

---

## 10. Skill content guidance

- **Decision tree** the skill must encode: one-off handler customization →
  `Result<T, T::Rejection>`; reusable remap of a built-in extractor →
  `WithRejection<E, R>`; reusable remap with zero boilerplate and the wrapper
  also usable as a named type → `#[derive(FromRequest)]` with
  `rejection(...)`; needs to read other request parts (e.g. `MatchedPath`)
  during extraction → hand-written custom extractor.
- ALWAYS include a `_ =>` catch-all arm when matching any rejection enum;
  every rejection enum is `#[non_exhaustive]`.
- ALWAYS use `rejection.status()` + `rejection.body_text()` to preserve Axum's
  correct status code and message instead of inventing your own.
- NEVER assume `Option<Extractor>` swallows errors on Axum 0.8 — it rejects on
  malformed input.
- `WithRejection` requires the `axum-extra` `with-rejection` feature; the
  derive macros require the `axum` `macros` feature (or `axum-macros`).

---

## Sources verified (2026-05-20)

All URLs below were fetched and verified via WebFetch on **2026-05-20**:

- https://docs.rs/axum/latest/axum/extract/rejection/index.html — full module
  inventory: 11 rejection enums, 14 rejection structs, per-extractor mapping.
- https://docs.rs/axum/latest/axum/extract/rejection/enum.JsonRejection.html —
  `JsonRejection` `#[non_exhaustive]`, four variants (`JsonDataError`,
  `JsonSyntaxError`, `MissingJsonContentType`, `BytesRejection`),
  `status()` / `body_text()` methods, `IntoResponse` + `Error` impls.
- https://docs.rs/axum-extra/latest/axum_extra/extract/struct.WithRejection.html
  — `WithRejection<E, R>` generic params, `IntoResponse` + `From<E::Rejection>`
  bounds on `R`, `with-rejection` feature flag, handler-argument shape.
- https://github.com/tokio-rs/axum/blob/main/examples/customize-extractor-error/src/main.rs
  — module declarations (`custom_extractor`, `derive_from_request`,
  `with_rejection`), three POST routes, server config.
- https://raw.githubusercontent.com/tokio-rs/axum/main/examples/customize-extractor-error/src/custom_extractor.rs
  — hand-written `Json<T>` wrapper, `Rejection = (StatusCode, axum::Json<Value>)`,
  `MatchedPath` extraction, `json!` payload with `body_text()` / `status()`.
- https://raw.githubusercontent.com/tokio-rs/axum/main/examples/customize-extractor-error/src/with_rejection.rs
  — `WithRejection(Json(value), _)` handler, `thiserror` `ApiError` enum with
  `#[from] JsonRejection`, `IntoResponse` impl.
- https://raw.githubusercontent.com/tokio-rs/axum/main/examples/customize-extractor-error/src/derive_from_request.rs
  — `#[derive(FromRequest)]` with `#[from_request(via(axum::Json), rejection(ApiError))]`,
  `From<JsonRejection>` and `IntoResponse` for `ApiError`.

Cross-referenced (verified earlier 2026-05-20, not re-fetched this session):
- https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html and
  `.../trait.FromRequestParts.html` — `type Rejection: IntoResponse` bound,
  native `impl Future` (no `#[async_trait]` in 0.8).
- https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0 — 0.8.0 breaking
  change: `Option<Path<T>>` reject-on-malformed, `OptionalFromRequest` /
  `OptionalFromRequestParts` traits.
