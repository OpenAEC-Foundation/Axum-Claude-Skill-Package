---
name: axum-errors-extractor-rejections
description: >
  Use when an Axum extractor fails and you need control over the HTTP error:
  customizing a Json/Path/Query rejection, returning a JSON error body instead
  of Axum's plain text, building a unified API error type, or upgrading 0.7 to
  0.8 and Option extractors suddenly return 4xx.
  Prevents the non-exhaustive match compile error on rejection enums, inventing
  a wrong status code instead of reusing the rejection's own, and the silent
  0.8 behavior change where Option no longer swallows malformed input.
  Covers the Rejection associated type, the axum::extract::rejection module,
  Result handler arguments, axum-extra WithRejection, derive(FromRequest) with
  rejection(...), hand-written custom extractors, and OptionalFromRequest.
  Keywords: axum rejection, extractor rejection, JsonRejection, PathRejection,
  QueryRejection, FormRejection, WithRejection, from_request rejection,
  derive FromRequest, OptionalFromRequest, non_exhaustive match error,
  body_text, status, customize extractor error, 400 instead of JSON,
  plain text error body, Option Path no longer swallows, 422 vs 400,
  how do I return a JSON error, why is my Option extractor rejecting,
  what is a rejection, custom error response for bad request body.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-errors-extractor-rejections

## Overview

Every Axum extractor that can fail carries an associated `Rejection` type. When
extraction fails, Axum does not bubble a `Service` error: it turns the
rejection straight into an HTTP response. Both extractor traits bound the
rejection with `type Rejection: IntoResponse`, which is the mechanism behind
Axum's infallible-handler model.

Built-in extractors ship predefined rejections in the `axum::extract::rejection`
module. The default rejection response is plain text with a sensible status
code. This skill covers the four ways to replace that default with your own
shape, and the Axum 0.8 change to `Option<T>` extractor behavior.

The rejection module and customization APIs are identical across Axum 0.7 and
0.8. The single version divergence is `Option<T>` extractor behavior (see the
decision tree below).

## Quick Reference

| Built-in extractor | Rejection enum | Default status family |
|--------------------|----------------|-----------------------|
| `Json<T>` | `JsonRejection` | 400 / 415 / 422 |
| `Path<T>` | `PathRejection` | 400 |
| `Query<T>` | `QueryRejection` | 400 |
| `Form<T>` | `FormRejection` | 400 / 415 |
| `Bytes` | `BytesRejection` | 400 |
| `String` | `StringRejection` | 400 |
| `Extension<T>` | `ExtensionRejection` | 500 |

Every rejection enum exposes the same two inherent methods:

| Method | Signature | Purpose |
|--------|-----------|---------|
| `status` | `pub fn status(&self) -> StatusCode` | The status Axum would have used |
| `body_text` | `pub fn body_text(&self) -> String` | The plain-text error message |

Customization techniques, lightest to heaviest:

| Technique | Crate / feature | Use when |
|-----------|-----------------|----------|
| `Result<T, T::Rejection>` handler arg | core `axum` | One or two handlers need bespoke handling |
| `WithRejection<E, R>` | `axum-extra` + `with-rejection` | Remap a built-in rejection across many handlers |
| `#[derive(FromRequest)]` + `rejection(...)` | `axum` + `macros` | Remap with zero boilerplate, wrapper also a named type |
| Hand-written custom extractor | core `axum` | Extraction must read other request parts |

Core rules:

- ALWAYS include a `_ =>` catch-all arm when matching a rejection enum. Every
  rejection enum is `#[non_exhaustive]`.
- ALWAYS reuse `rejection.status()` and `rejection.body_text()` to preserve
  Axum's correct status and message. NEVER invent your own status code.
- NEVER assume `Option<Extractor>` swallows all errors on Axum 0.8. It rejects
  on malformed input.
- A custom extractor's `Rejection` type MUST implement `IntoResponse`.
- NEVER implement both `FromRequest` and `FromRequestParts` for the same
  concrete type, except for a generic wrapper around another extractor.

## Decision Trees

### Which customization technique

```
Does extraction itself need to read other request parts
(MatchedPath, headers) to build the error?
  YES: hand-write a custom extractor. Set its Rejection type
       and build the response in from_request / from_request_parts.
  NO: does more than one or two handlers need the same custom error?
    NO:  type the handler argument as Result<Extractor, Rejection>
         and match on the rejection inside the handler body.
    YES: do you also want the wrapper usable as a named type
         and zero manual trait impls?
      YES: #[derive(FromRequest)] with #[from_request(via(...), rejection(R))].
      NO:  WithRejection<Extractor, R> from axum-extra.
```

### Option<T> extractor behavior: 0.7 vs 0.8

```
On which Axum version does the Option<Extractor> handler argument run?
  0.7: Option<T> swallows ALL failure conditions and yields None.
  0.8: Option<T> is backed by OptionalFromRequest(Parts).
       Genuinely absent value  -> None.
       Present but malformed value -> rejection (4xx).
```

After a 0.7 to 0.8 upgrade, ALWAYS audit every `Option<Extractor>` handler
argument. Code that relied on swallow-everything now returns 4xx where it
previously returned `None`. To restore the old behavior, use
`Result<T, T::Rejection>` and map `Err(_)` to your own default.

## Patterns

### Pattern: match a rejection enum with the mandatory catch-all

Every rejection enum is `#[non_exhaustive]`, so Axum can add failure variants
in a minor release. A `match` without a `_ =>` arm does not compile.

```rust
// axum 0.7 / 0.8 - identical
use axum::extract::rejection::JsonRejection;
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};

fn render(rejection: JsonRejection) -> Response {
    match rejection {
        JsonRejection::MissingJsonContentType(_) =>
            (StatusCode::UNSUPPORTED_MEDIA_TYPE, "expected application/json")
                .into_response(),
        JsonRejection::JsonSyntaxError(_) =>
            (StatusCode::BAD_REQUEST, "invalid JSON syntax").into_response(),
        JsonRejection::JsonDataError(_) =>
            (StatusCode::UNPROCESSABLE_ENTITY, "JSON does not match schema")
                .into_response(),
        // #[non_exhaustive] makes this arm mandatory
        rejection => (rejection.status(), rejection.body_text()).into_response(),
    }
}
```

### Pattern: Result<T, T::Rejection> handler argument

The lightest customization. No custom extractor, no extra crate. Type the
handler argument as `Result<Extractor, Rejection>`; Axum hands you the
`Result` so you `match` inside the handler.

```rust
// axum 0.7 / 0.8 - identical
use axum::{extract::rejection::JsonRejection, Json, response::Response};
use serde_json::Value;

async fn create(payload: Result<Json<Value>, JsonRejection>) -> Response {
    match payload {
        Ok(Json(value)) => Json(value).into_response(),
        Err(rejection)  => render(rejection), // see catch-all pattern above
    }
}
```

ALWAYS use this when only one or two handlers need bespoke handling. NEVER
scatter it across every handler in a large API; use `WithRejection` or a
derived rejection instead.

### Pattern: WithRejection<E, R> for codebase-wide remapping

`axum-extra`'s `WithRejection<E, R>` runs extractor `E` and, on failure,
converts its rejection into your type `R`. `R` MUST implement `IntoResponse`
and `From<E::Rejection>`. Requires the `with-rejection` feature on `axum-extra`.

```rust
// axum 0.7 / 0.8 - identical, requires axum-extra "with-rejection" feature
use axum::{extract::rejection::JsonRejection, Json, response::{IntoResponse, Response}};
use axum_extra::extract::WithRejection;
use serde_json::{json, Value};
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ApiError {
    #[error(transparent)]
    JsonRejection(#[from] JsonRejection), // #[from] generates From<JsonRejection>
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            ApiError::JsonRejection(r) => (r.status(), r.body_text()),
        };
        (status, Json(json!({ "message": message }))).into_response()
    }
}

async fn handler(WithRejection(Json(value), _): WithRejection<Json<Value>, ApiError>) {
    let _ = value; // the successfully extracted value
}
```

The second tuple field is a `PhantomData<R>` placeholder; discard it with `_`.

### Pattern: derive(FromRequest) with rejection(...)

`#[derive(FromRequest)]` builds an extractor that delegates to another via
`via(...)` and routes its rejection through `rejection(...)`, with no manual
trait `impl`. Requires the `macros` feature on `axum`.

```rust
// axum 0.7 / 0.8 - identical, requires axum "macros" feature
use axum::extract::FromRequest;
use axum::extract::rejection::JsonRejection;
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use serde_json::json;

#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct Json<T>(pub T);

pub struct ApiError { status: StatusCode, message: String }

impl From<JsonRejection> for ApiError {
    fn from(r: JsonRejection) -> Self {
        ApiError { status: r.status(), message: r.body_text() }
    }
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        (self.status, axum::Json(json!({ "message": self.message }))).into_response()
    }
}
```

`via(...)` names the inner extractor; `rejection(...)` names the type the inner
rejection is converted into via `From`. A body-consuming extractor derives
`FromRequest`; a parts-only one derives `FromRequestParts`.

### Pattern: hand-written custom extractor

Use this only when extraction must read other request parts. The wrapper
declares its own `Rejection` type, which MUST implement `IntoResponse`.

```rust
// axum 0.7 / 0.8 - identical
use axum::extract::{FromRequest, MatchedPath, Request, rejection::JsonRejection};
use axum::http::StatusCode;
use serde::de::DeserializeOwned;
use serde_json::{json, Value};

pub struct Json<T>(pub T);

impl<S, T> FromRequest<S> for Json<T>
where
    axum::Json<T>: FromRequest<S, Rejection = JsonRejection>,
    T: DeserializeOwned,
    S: Send + Sync,
{
    type Rejection = (StatusCode, axum::Json<Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let path = req.extensions().get::<MatchedPath>()
            .map(|p| p.as_str().to_owned());
        match axum::Json::<T>::from_request(req, state).await {
            Ok(axum::Json(value)) => Ok(Self(value)),
            Err(rejection) => {
                let payload = json!({ "message": rejection.body_text(), "path": path });
                Err((rejection.status(), axum::Json(payload)))
            }
        }
    }
}
```

See `references/examples.md` for the full verified version of every pattern.

## Reference Links

- `references/methods.md`: verified `FromRequest` / `FromRequestParts` trait
  definitions, the full `axum::extract::rejection` module inventory (11 enums,
  14 structs), `JsonRejection` detail, `WithRejection<E, R>` signature, the
  `#[from_request]` attribute keys, and the `OptionalFromRequest(Parts)` traits.
- `references/examples.md`: complete version-annotated working code for all six
  patterns, including the `Option<Path<T>>` 0.7-vs-0.8 contrast.
- `references/anti-patterns.md`: real mistakes with root-cause analysis: the
  missing catch-all arm, inventing a wrong status, the 0.8 `Option` trap,
  scattering `Result` arguments, the double-trait-impl error, and missing
  feature flags.

Related skills: `axum-syntax-extractors` (the full extractor set),
`axum-syntax-custom-extractors` (writing `FromRequest` by hand),
`axum-errors-handling` (application-level error types and `IntoResponse`),
`axum-errors-handler-trait` (diagnosing `Handler is not satisfied`).
