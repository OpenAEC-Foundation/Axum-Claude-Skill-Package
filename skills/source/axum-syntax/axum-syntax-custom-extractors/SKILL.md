---
name: axum-syntax-custom-extractors
description: >
  Use when implementing a custom extractor in Axum: a type that builds itself
  from the request by implementing FromRequestParts or FromRequest, a wrapper
  that runs an inner extractor and remaps its rejection, or a struct derived
  with axum-macros #[derive(FromRequest)].
  Prevents the stale 0.7 habit of keeping #[async_trait] on a 0.8 extractor
  impl, prevents the ambiguous-extractor bug from implementing both
  FromRequest and FromRequestParts for one concrete type, and prevents the
  "Handler is not satisfied" error from a body extractor that is not last.
  Covers the two extractor traits and the Rejection associated type, the
  FromRequestParts vs FromRequest choice, native async fn in trait versus
  #[async_trait], wrapping and remapping an inner extractor, FromRef substate
  access, and the axum-macros derive attributes.
  Keywords: axum custom extractor, FromRequestParts, FromRequest, impl
  FromRequestParts, type Rejection, IntoResponse, async_trait removal, RPITIT,
  derive FromRequest, from_request via, from_request rejection, ViaRequest,
  ViaParts blanket impl, wrap Json extractor, ValidatedJson, Handler is not
  satisfied, conflicting trait implementation, extractor is ambiguous, how do
  I write my own extractor, build a type from the request, protect a handler
  with an extractor.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-syntax-custom-extractors

## Overview

A custom extractor is a type you define that builds itself from an incoming
request. You turn a type into an extractor by implementing one of two traits.
Which trait is decided by exactly one question: does the extractor read the
request body?

- `FromRequestParts<S>`: builds the value from request parts only (method,
  URI, headers, extensions, captured path params). It NEVER touches the body
  and may appear in any handler argument position.
- `FromRequest<S>`: builds the value by consuming the request body. A handler
  may have at most one `FromRequest` extractor and it MUST be the last
  argument.

Both traits carry a `Rejection` associated type: the error returned when
extraction fails. `Rejection` MUST implement `IntoResponse` so Axum converts a
failed extraction directly into an HTTP response.

The single version-divergent detail is the async mechanism. Axum 0.8 uses
native `async fn` in traits (return-position `impl Trait`, RPITIT, Rust 1.75).
Axum 0.7 requires the `#[async_trait]` attribute macro on the `impl` block.
Everything else in this skill is identical across 0.7 and 0.8.

## Quick Reference

| Trait | Reads body | Method | Handler position |
|-------|-----------|--------|------------------|
| `FromRequestParts<S>` | NO | `from_request_parts(&mut Parts, &S)` | any |
| `FromRequest<S>` | YES | `from_request(Request, &S)` | LAST only |

Trait definitions, Axum 0.8 native-async form (verified verbatim from
docs.rs):

```rust
// axum 0.8
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

Core rules:

- ALWAYS implement `FromRequestParts` when the extractor does not read the
  body. It composes more freely: it runs in any handler argument position.
- ALWAYS implement `FromRequest` when the extractor consumes the body. It must
  be the last handler argument.
- NEVER implement BOTH `FromRequest` and `FromRequestParts` for the same
  concrete type. A blanket impl already bridges them; a hand-written second
  impl makes the extractor ambiguous and unusable as a handler argument.
- ALWAYS make the `Rejection` associated type a value that implements
  `IntoResponse`. `(StatusCode, &'static str)` and `(StatusCode, Json<T>)`
  both qualify.
- NEVER write `#[async_trait]` on an extractor impl for Axum 0.8. It is
  required on Axum 0.7 and forbidden on 0.8.
- ALWAYS delegate body parsing to an existing extractor (`Json`, `Form`,
  `Bytes`) inside a `FromRequest` impl instead of reading bytes by hand.

## Decision Trees

### Which trait to implement

```
Does the extractor need to read the request BODY?
  NO  (reads method, URI, headers, extensions, path params, state)
                                  -> impl FromRequestParts<S>
  YES (parses JSON, form, bytes, or the raw body stream)
                                  -> impl FromRequest<S>   (must be last arg)
```

The blanket impl `impl<S, T> FromRequest<S, ViaParts> for T where
T: FromRequestParts<S>` means every `FromRequestParts` type is automatically a
`FromRequest` type. So a `FromRequestParts` extractor still works as the last
handler argument. The "must be last" constraint only bites a genuine body
extractor.

### Hand-written impl vs derive macro

```
What shape is the extractor?
  ONE value pulled from the request (a header, a token, a tenant)
                          -> hand-write impl FromRequestParts / FromRequest
  a STRUCT bundling several existing extractors into one argument
                          -> #[derive(FromRequestParts)] or #[derive(FromRequest)]
  wrap ONE existing extractor and remap only its rejection
                          -> hand-write a newtype impl, or derive with
                             #[from_request(via(...), rejection(...))]
```

### Reading application state inside an extractor

```
Does the extractor need application state?
  NO   -> impl<S: Send + Sync> FromRequestParts<S> for MyType   (ignore state)
  YES, the WHOLE state value
       -> bound the impl to a concrete state: impl FromRequestParts<AppState>
  YES, ONE field of a composite state
       -> bound with FromRef: where SomeResource: FromRef<S>, S: Send + Sync
          then SomeResource::from_ref(state)
```

## Patterns

### Pattern: a FromRequestParts extractor

The most common custom extractor reads a request part and builds a typed
value, rejecting the request when the data is missing or malformed. The
0.7-to-0.8 difference is exactly one line: the `#[async_trait]` attribute.

```rust
// axum 0.7 - custom parts extractor: REQUIRES #[async_trait]
use async_trait::async_trait;
use axum::extract::FromRequestParts;
use axum::http::{header::USER_AGENT, request::Parts, HeaderValue, StatusCode};

struct ExtractUserAgent(HeaderValue);

#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S)
        -> Result<Self, Self::Rejection>
    {
        parts
            .headers
            .get(USER_AGENT)
            .map(|v| ExtractUserAgent(v.clone()))
            .ok_or((StatusCode::BAD_REQUEST, "missing user-agent header"))
    }
}
```

```rust
// axum 0.8 - SAME extractor, NO #[async_trait], native async fn in trait
use axum::extract::FromRequestParts;
use axum::http::{header::USER_AGENT, request::Parts, HeaderValue, StatusCode};

struct ExtractUserAgent(HeaderValue);

impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S)
        -> Result<Self, Self::Rejection>
    {
        parts
            .headers
            .get(USER_AGENT)
            .map(|v| ExtractUserAgent(v.clone()))
            .ok_or((StatusCode::BAD_REQUEST, "missing user-agent header"))
    }
}
```

Facts that hold for both versions:

- `parts: &mut Parts` is `http::request::Parts`. It is `&mut` because some
  built-in extractors take their data out of `parts.extensions`. Reading
  headers does not need the `&mut`, but the trait signature requires it.
- `state: &S` is an immutable borrow. A parts extractor reads state, it never
  owns it. The bound `S: Send + Sync` is required in both versions.
- A handler that takes `ExtractUserAgent` as an argument is now protected: the
  request is rejected before the handler body runs if the header is absent.

### Pattern: a body extractor wrapping Json

A `FromRequest` extractor consumes the body, so it must be the last handler
argument. ALWAYS delegate the byte parsing to an existing extractor and remap
its rejection rather than reading the body stream by hand.

```rust
// axum 0.8 - ValidatedJson<T>: wraps Json<T>, runs validation, remaps rejection
// axum 0.7: identical body, add #[async_trait] above the impl block
use axum::extract::{FromRequest, Request};
use axum::extract::rejection::JsonRejection;
use axum::http::StatusCode;
use axum::Json;
use serde_json::json;

struct ValidatedJson<T>(T);

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
        // delegate body consumption to Json, remap its JsonRejection
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|rej| {
                (rej.status(), Json(json!({ "error": rej.body_text() })))
            })?;

        // run domain validation, remap ValidationErrors to a 422
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

Facts behind this pattern:

- `Request` is `axum::extract::Request`, an alias for
  `http::Request<axum::body::Body>`. It is consumed by value because reading
  the body takes ownership of the single-use stream.
- `JsonRejection` exposes `.status()` and `.body_text()`. Calling these and
  rebuilding a new tuple is how an inner rejection becomes the outer
  `Rejection`.
- `(StatusCode, Json<serde_json::Value>)` implements `IntoResponse`, so it
  satisfies the `Rejection: IntoResponse` bound.

### Pattern: reading a substate via FromRef

When the extractor needs one resource out of a composite application state,
bound the impl with `FromRef` instead of pinning the whole state type.

```rust
// axum 0.8 - extractor that pulls one resource out of a composite AppState
use axum::extract::{FromRef, FromRequestParts};
use axum::http::{header::HOST, request::Parts, StatusCode};

struct CurrentTenant(Tenant);

impl<S> FromRequestParts<S> for CurrentTenant
where
    TenantRegistry: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, state: &S)
        -> Result<Self, Self::Rejection>
    {
        let registry = TenantRegistry::from_ref(state);
        let host = parts
            .headers
            .get(HOST)
            .and_then(|v| v.to_str().ok())
            .ok_or((StatusCode::BAD_REQUEST, "missing host header"))?;
        registry
            .lookup(host)
            .map(CurrentTenant)
            .ok_or((StatusCode::NOT_FOUND, "unknown tenant"))
    }
}
```

This keeps the extractor reusable with any state type that can produce a
`TenantRegistry` through `FromRef`. See `axum-core-state` for the `FromRef`
substate model.

### Pattern: deriving an extractor with axum-macros

`axum-macros` provides `#[derive(FromRequest)]` and
`#[derive(FromRequestParts)]` so a struct that bundles several extractors
needs no hand-written `impl`. The derive macro syntax is identical on Axum 0.7
and 0.8.

```rust
// axum 0.7 / 0.8 - derive FromRequestParts, per-field extraction
use axum::extract::{FromRequestParts, Query};
use axum_extra::TypedHeader;
use axum_extra::headers::ContentType;
use std::collections::HashMap;

#[derive(FromRequestParts)]
struct MyExtractor {
    #[from_request(via(Query))]
    query_params: HashMap<String, String>,
    content_type: TypedHeader<ContentType>,
}
```

By default the derive calls `from_request` (or `from_request_parts`) on each
field. `#[from_request(via(Query))]` runs `Query` to populate that field; a
field with no attribute must already be an extractor type.

```rust
// axum 0.7 / 0.8 - derive FromRequest with a unified custom rejection
use axum::extract::{rejection::JsonRejection, FromRequest};
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::Json;

#[derive(FromRequest)]
#[from_request(rejection(ApiError))]
struct CreateUser {
    #[from_request(via(Json))]
    payload: NewUserPayload,
}

// ApiError MUST implement IntoResponse AND From<JsonRejection>
// (one From impl per via-field rejection type).
struct ApiError(StatusCode, String);

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        (self.0, self.1).into_response()
    }
}

impl From<JsonRejection> for ApiError {
    fn from(rej: JsonRejection) -> Self {
        ApiError(rej.status(), rej.body_text())
    }
}
```

Derive rules (verified on docs.rs):

- `#[from_request(via(...))]` extracts a field (or the whole type) through
  another extractor. The `via` extractor must be a generic newtype struct (a
  tuple struct with exactly one public field) that implements the trait.
- `#[from_request(rejection(MyRejection))]` sets one unified rejection type.
  `MyRejection` MUST implement `IntoResponse` and `From<R>` for every field
  rejection `R`.
- `#[from_request(state(MyState))]` pins a concrete state type instead of a
  generic `S`.
- With `FromRequest`, only the last field may consume the body.

### Pattern: porting a 0.7 extractor to 0.8

```
1. Delete the #[async_trait] attribute above every FromRequest /
   FromRequestParts impl block.
2. Remove the async-trait dependency from Cargo.toml.
3. Leave the async fn body unchanged. The signature stays the same.
4. Verify Rust toolchain >= 1.75 (Axum 0.8 MSRV: native RPITIT needs it).
```

The plain `async fn from_request_parts` / `async fn from_request` body is
byte-for-byte identical between versions. Only the attribute and the
dependency are removed.

## The Both-Traits Rule

NEVER implement both `FromRequest` and `FromRequestParts` for the same
concrete type. The blanket impl
`impl<S, T> FromRequest<S, ViaParts> for T where T: FromRequestParts<S>`
already makes every `FromRequestParts` type usable as a `FromRequest` through
the `ViaParts` marker. A hand-written `FromRequest<S>` impl resolves through
the default `M = ViaRequest` marker, so the type now has two `FromRequest`
impls reachable through two different markers. The compiler can no longer
infer which marker a handler argument needs, the extractor becomes ambiguous,
and the `Handler` trait bound fails.

The single legitimate exception is a generic wrapper extractor (for example
`axum-extra`'s `WithRejection<E, R>`). A generic wrapper implements the two
traits over different inner extractors `E`, so the impls never collide for one
concrete type. For a concrete (non-generic) extractor, pick exactly one trait.

## Reference Links

- `references/methods.md`: verbatim `FromRequestParts` and `FromRequest` trait
  definitions, the `ViaParts` blanket impl, the `Rejection` bound, the
  `axum-macros` derive attribute table, and `JsonRejection` accessors.
- `references/examples.md`: complete version-annotated working extractors,
  including the parts extractor, the `ValidatedJson` body extractor, the
  `FromRef` substate extractor, and the two derive forms.
- `references/anti-patterns.md`: real mistakes with root-cause analysis,
  including `#[async_trait]` left on a 0.8 impl, the both-traits ambiguity, a
  body extractor not placed last, and a `Rejection` type that is not
  `IntoResponse`.

Related skills: `axum-syntax-extractors` (the built-in extractor set and the
two traits), `axum-errors-extractor-rejections` (the rejection module and
`WithRejection`), `axum-core-version-migration` (the full 0.7 to 0.8 matrix),
`axum-impl-validation` (the `validator` crate and `ValidatedJson` in context),
`axum-core-state` (the `FromRef` substate model).
