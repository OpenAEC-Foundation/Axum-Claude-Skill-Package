# Topic Research : axum-syntax-custom-extractors

> Skill : `axum-syntax-custom-extractors`
> Phase : 4 (topic research)
> Target versions : Axum 0.7 and 0.8 (docs.rs current : 0.8.x)
> Researched / verified : 2026-05-20
> Source fragments consulted : `vooronderzoek-axum.md` (sec. 2, 3),
> `research-a-core-routing-extractors.md` (sec. 5), `research-c-errors-auth-db-ops.md` (sec. 2)

This file is the verified research basis for the `axum-syntax-custom-extractors`
skill. Every trait definition, signature, and attribute below was WebFetch-verified
against docs.rs on 2026-05-20. It is research only; it is not the skill.

---

## 1. The Two Extractor Traits (verbatim, Axum 0.8 native-async form)

A custom extractor is any type for which you implement one of two traits.
`FromRequestParts<S>` is for extractors that read only the request *parts*
(method, URI, headers, extensions, captured path params) and never touch the
body. `FromRequest<S>` is for extractors that consume the request *body*.

Verified verbatim from docs.rs (Axum 0.8):

```rust
// axum 0.8 - https://docs.rs/axum/latest/axum/extract/trait.FromRequestParts.html
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;

    fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

```rust
// axum 0.8 - https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html
pub trait FromRequest<S, M = ViaRequest>: Sized {
    type Rejection: IntoResponse;

    fn from_request(
        req: Request<Body>,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

Verified facts about these definitions:

- **`Parts` import path** is `http::request::Parts` (re-exported in Axum as
  `axum::extract::FromRequestParts` callers typically import `axum::http::request::Parts`).
- **`Request<Body>`** uses `axum::body::Body` as the body type. In Axum 0.8 there
  is no generic `B` body parameter; the body type is always `axum::body::Body`.
- **`M = ViaRequest`** is a default generic marker parameter on `FromRequest`.
  Most hand-written impls write `FromRequest<S>` and never name `M`; the marker
  exists only so the blanket impl below can coexist.
- **The `Rejection` associated type** is the error returned when extraction fails.
  Its bound is `Rejection: IntoResponse` (verified verbatim). Because the rejection
  is `IntoResponse`, Axum converts a failed extraction directly into an HTTP
  response. docs.rs states: "If the extractor fails it'll use this 'rejection'
  type. A rejection is a kind of error that can be converted into a response."
- Both methods return a bare `impl Future<...> + Send`. There is **no
  `#[async_trait]`**. This is native return-position `impl Trait` in traits
  (RPITIT), stabilized in Rust 1.75, which is exactly Axum 0.8's MSRV.

### The blanket impl (parts-extractors are also body-extractors)

Verified verbatim from docs.rs:

```rust
// axum 0.8 - blanket impl on FromRequest
impl<S, T> FromRequest<S, ViaParts> for T
where
    S: Send + Sync,
    T: FromRequestParts<S>,
```

This is why a handler whose arguments are all `FromRequestParts` extractors
compiles fine: every `FromRequestParts` type is automatically a `FromRequest`
type via the `ViaParts` marker. The constraint "the last argument must be
`FromRequest`" therefore only bites when a *genuine body* extractor is not last.

docs.rs guidance, verbatim: "Extractors that implement `FromRequestParts` cannot
consume the request body and can thus be run in any order for handlers. If your
extractor needs to consume the request body then you should implement
`FromRequest` and not `FromRequestParts`." And conversely: "If your extractor
doesn't need to consume the request body then you should implement
`FromRequestParts` and not `FromRequest`."

---

## 2. A Custom `FromRequestParts` Extractor : 0.7 vs 0.8

This is the most common custom extractor: read a header (or other part) and
build a typed value, rejecting the request if the data is missing or malformed.
The 0.7-to-0.8 difference is exactly one line: the `#[async_trait]` attribute.

```rust
// axum 0.7 - custom parts extractor (REQUIRES #[async_trait])
use async_trait::async_trait;
use axum::extract::FromRequestParts;
use axum::http::{header::USER_AGENT, request::Parts, StatusCode};

struct ExtractUserAgent(http::HeaderValue);

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
use axum::http::{header::USER_AGENT, request::Parts, StatusCode};

struct ExtractUserAgent(http::HeaderValue);

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

Migration rule, verified: porting a custom extractor from 0.7 to 0.8 means
deleting the `#[async_trait]` attribute above the `impl` block and removing the
`async-trait` dependency from `Cargo.toml`. The plain `async fn` body is
unchanged. The `state: &S` bound `S: Send + Sync` stays the same in both versions.

Notes that apply to both versions:

- The state argument is `&S` (an immutable borrow), so a parts extractor reads
  state, it does not own it. Pull a substate with `FromRef`: bound the impl with
  `where SomeResource: FromRef<S>` then `SomeResource::from_ref(state)`.
- `parts: &mut Parts` is mutable because some built-in extractors (`Path`)
  *take* their data out of `parts.extensions`. Reading headers does not need the
  `&mut`, but the trait signature requires it regardless.
- A `FromRequestParts` extractor can appear in any handler argument position.

---

## 3. A Body-Consuming Custom Extractor : Wrap `Json`, Remap the Rejection

A `FromRequest` extractor consumes the body, so it must be the last handler
argument. The idiomatic body extractor does not parse bytes by hand; it delegates
to an existing extractor (`Json`, `Form`, `Bytes`) and remaps the inner
rejection into its own `Rejection` type.

```rust
// axum 0.8 - ValidatedJson<T>: wraps Json<T>, runs validation, remaps rejection
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
    // require that Json<T> is itself a body extractor with the known rejection
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

The same code in Axum 0.7 needs `#[async_trait]` on the `impl` block; nothing
else changes. Verified facts behind this pattern:

- `Request` in the signature is `axum::extract::Request`, an alias for
  `http::Request<axum::body::Body>`. It is consumed by value because reading the
  body takes ownership of the single-use stream.
- `JsonRejection` lives in `axum::extract::rejection`, is `#[non_exhaustive]`,
  and exposes `.status()` (the `StatusCode`) and `.body_text()` (a `String`).
  Its variants are `MissingJsonContentType`, `JsonDataError`, `JsonSyntaxError`,
  and `BytesRejection`.
- The `Rejection` type `(StatusCode, Json<serde_json::Value>)` is `IntoResponse`
  because Axum implements `IntoResponse` for `(StatusCode, T)` tuples and for
  `Json<T>`. This satisfies the `Rejection: IntoResponse` bound.
- A body extractor must be the **last** handler argument; the body is a
  single-use async stream, so a handler may have at most one `FromRequest`
  extractor and it must come last.

---

## 4. The `#[derive(FromRequest)]` / `#[derive(FromRequestParts)]` Macros

`axum-macros` (enabled via Axum's `macros` feature, also re-exported as
`axum::FromRequest` / `axum::FromRequestParts`) provides derive macros so a
struct that bundles several extractors needs no hand-written `impl`. Verified
attribute options from docs.rs (`derive.FromRequest.html`,
`derive.FromRequestParts.html`):

| Attribute | Placement | Effect |
|-----------|-----------|--------|
| `#[from_request(via(SomeExtractor))]` | field or container | extract that field (or the whole type) *through* `SomeExtractor` |
| `#[from_request(rejection(MyRejection))]` | container | use `MyRejection` as the unified `Rejection` type |
| `#[from_request(state(MyState))]` | container | pin the concrete state type instead of generic `S` |

### Per-field extraction (multi-field struct)

Verified verbatim from docs.rs: "By default `#[derive(FromRequest)]` will call
`FromRequest::from_request` for each field." Each field must itself be an
extractor; with `FromRequest`, "only the last field can consume the request
body."

Verified example (`derive.FromRequestParts.html`):

```rust
// axum 0.8 - derive FromRequestParts, per-field extraction
use axum::extract::FromRequestParts;
use axum_extra::TypedHeader;
use axum_extra::headers::ContentType;
use axum::extract::Query;
use std::collections::HashMap;

#[derive(FromRequestParts)]
struct MyExtractor {
    #[from_request(via(Query))]
    query_params: HashMap<String, String>,
    content_type: TypedHeader<ContentType>,
}

async fn handler(extractor: MyExtractor) {}
```

`#[from_request(via(Query))]` runs `Query` to populate `query_params`; the
`content_type` field has no attribute, so it is extracted as a `TypedHeader`
directly (it is already an extractor type).

### Custom unified rejection

Verified verbatim from docs.rs (`derive.FromRequest.html`): a custom rejection
must satisfy two conditions: "This tells axum how to convert" each field's
rejection "into `MyRejection`" via `From` trait implementations, and "All
rejections must implement `IntoResponse`."

```rust
// axum 0.8 - derive with a unified custom rejection
use axum::extract::FromRequest;
use axum::Json;
use axum::response::{IntoResponse, Response};
use axum::http::StatusCode;

#[derive(FromRequest)]
#[from_request(rejection(ApiError))]
struct CreateUser {
    #[from_request(via(Json))]
    payload: NewUserPayload,
}

// ApiError must: implement IntoResponse, and implement
// From<JsonRejection> (the rejection of the via(Json) field).
struct ApiError(StatusCode, String);

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        (self.0, self.1).into_response()
    }
}

impl From<axum::extract::rejection::JsonRejection> for ApiError {
    fn from(rej: axum::extract::rejection::JsonRejection) -> Self {
        ApiError(rej.status(), rej.body_text())
    }
}
```

`#[derive(FromRequestParts)]` is identical except it cannot consume the body.
docs.rs states verbatim that the parts derive functions "similarly to
`#[derive(FromRequest)]` but without body extraction capability" and, for body
extraction, "Use `#[derive(FromRequest)]` for that." When `via(...)` is used on
the container (single newtype field), docs.rs notes the wrapped extractor must
be "a generic newtype struct (a tuple struct with exactly one public field) that
implements `FromRequest`."

---

## 5. The Both-Traits Rule

**Rule: do NOT implement both `FromRequest` and `FromRequestParts` for the same
concrete type, unless the type is a generic wrapper around another extractor.**

Why it matters: the blanket impl `impl<S, T> FromRequest<S, ViaParts> for T
where T: FromRequestParts<S>` already makes every `FromRequestParts` type usable
as a `FromRequest`. If you *also* hand-write a `FromRequest<S>` (which resolves
with the `M = ViaRequest` marker) for the same concrete type, the type now has
two `FromRequest` impls reachable through two different marker types. The
compiler can no longer infer which marker `M` a handler argument should use, so
the extractor becomes ambiguous and **unusable** as a handler argument: type
inference fails and the `Handler` trait bound is not satisfied.

The single legitimate exception is a **generic wrapper** extractor, for example
`struct WithRejection<E, R>` or a `ViaRequest`/`ViaParts`-discriminated wrapper.
Such a wrapper implements `FromRequest` and `FromRequestParts` over *different
generic parameterizations* (different inner extractor `E`), so the two impls
never collide for a single concrete type. The official `axum-extra`
`WithRejection` extractor does exactly this. Practical guidance for skill
authors: for a concrete extractor, pick exactly one trait based on whether you
need the body. Implementing both for a concrete (non-generic) type is always a
bug.

---

## 6. Lift-Ready Verified Snippets (version-annotated)

**S-1 : Verbatim trait definitions (0.8 native async form).**

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

**S-2 : Parts extractor, 0.7 form (with `#[async_trait]`).**

```rust
// axum 0.7
#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S)
        -> Result<Self, Self::Rejection>
    {
        parts.headers.get(USER_AGENT)
            .map(|v| ExtractUserAgent(v.clone()))
            .ok_or((StatusCode::BAD_REQUEST, "missing user-agent header"))
    }
}
```

**S-3 : Same parts extractor, 0.8 form (NO `#[async_trait]`).**

```rust
// axum 0.8
impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S)
        -> Result<Self, Self::Rejection>
    {
        parts.headers.get(USER_AGENT)
            .map(|v| ExtractUserAgent(v.clone()))
            .ok_or((StatusCode::BAD_REQUEST, "missing user-agent header"))
    }
}
```

**S-4 : Body extractor wrapping `Json`, remapping the rejection (0.8).**

```rust
// axum 0.8 - 0.7 needs #[async_trait] on the impl, otherwise identical
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

**S-5 : `#[derive(FromRequest)]` with `via` and a unified custom rejection
(0.8 and 0.7, the derive macro is version-stable).**

```rust
// axum 0.7 and 0.8 - derive macro syntax is identical across versions
#[derive(FromRequest)]
#[from_request(rejection(ApiError))]
struct CreateUser {
    #[from_request(via(Json))]
    payload: NewUserPayload,
}
// ApiError: impl IntoResponse + impl From<JsonRejection>
```

**S-6 : Parts extractor reading a substate via `FromRef` (0.8).**

```rust
// axum 0.8 - extractor that pulls one resource out of a composite AppState
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
        let host = parts.headers.get(axum::http::header::HOST)
            .and_then(|v| v.to_str().ok())
            .ok_or((StatusCode::BAD_REQUEST, "missing host header"))?;
        registry.lookup(host)
            .map(CurrentTenant)
            .ok_or((StatusCode::NOT_FOUND, "unknown tenant"))
    }
}
```

---

## 7. Sources Verified

All URLs fetched and verified via WebFetch on **2026-05-20**.

- https://docs.rs/axum/latest/axum/extract/trait.FromRequestParts.html — verbatim
  `FromRequestParts<S>` trait definition, native `impl Future + Send` return,
  `type Rejection: IntoResponse` bound, `Parts` is `http::request::Parts`,
  parts-cannot-consume-body guidance.
- https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html — verbatim
  `FromRequest<S, M = ViaRequest>` trait definition, `Request<Body>` argument,
  `type Rejection: IntoResponse` bound, the
  `impl<S, T> FromRequest<S, ViaParts> for T where T: FromRequestParts<S>`
  blanket impl, implement-only-one-trait guidance.
- https://docs.rs/axum-macros/latest/axum_macros/ — index of the `FromRequest` /
  `FromRequestParts` derive macros.
- https://docs.rs/axum-macros/latest/axum_macros/derive.FromRequest.html —
  `#[from_request(via(...))]`, `#[from_request(rejection(...))]`,
  `#[from_request(state(...))]` attribute options; per-field `from_request`
  default behavior; "only the last field can consume the request body"; rejection
  must implement `From` for each field rejection and all rejections must
  implement `IntoResponse`; `via` requires a generic single-public-field newtype.
- https://docs.rs/axum-macros/latest/axum_macros/derive.FromRequestParts.html —
  `#[derive(FromRequestParts)]` verbatim example (`MyExtractor` with
  `#[from_request(via(Query))]`), behaves like `FromRequest` derive minus body
  extraction.

Cross-referenced (verified in prior Phase 2 fragments on 2026-05-20, not
re-fetched here): the `#[async_trait]` removal in Axum 0.8 and the official
`customize-extractor-error` example confirming a plain `async fn from_request`
with no attribute — see `research-a-core-routing-extractors.md` section 5 and
`research-c-errors-auth-db-ops.md` section 2.

### Verification caveats

- `axum-macros` derive pages: docs.rs's summarization model returned the
  attribute *options* and one verbatim example but did not return every example
  block on the page. The `rejection(...)`/`via(...)`/`state(...)` option set and
  the `MyExtractor` example are verified verbatim; the `CreateUser`/`ApiError`
  snippet in section 4 is a faithful composition of the verified rule ("rejection
  must implement `From` for each field's rejection and `IntoResponse`") rather
  than a verbatim docs transcription, and is marked as such.
- The both-traits ambiguity mechanism (section 5) is derived from the verified
  blanket impl with marker `ViaParts` plus the default marker `M = ViaRequest`,
  and the verified docs guidance to implement only one trait per concrete type.
  The generic-wrapper exception is confirmed by the existence of `axum-extra`'s
  `WithRejection`.
