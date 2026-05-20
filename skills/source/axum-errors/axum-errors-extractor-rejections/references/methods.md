# Methods and API signatures: axum-errors-extractor-rejections

All signatures verified via WebFetch on 2026-05-20 against the URLs listed at
the bottom. Signatures are identical across Axum 0.7 and 0.8 unless a version
is annotated.

## 1. The Rejection associated type

Both extractor traits declare an associated `Rejection` type bound by
`IntoResponse`. This bound is the mechanism behind Axum's infallible handlers:
a failed extraction becomes an HTTP response, never a `Service` error.

```rust
// axum 0.8 - verbatim from docs.rs
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

Axum 0.8 uses native `impl Future` returns and does NOT use `#[async_trait]`.
Axum 0.7 declared the same methods under `#[async_trait]` with `async fn`. The
associated `Rejection: IntoResponse` bound is unchanged between versions.

`FromRequest` is generic over a marker `M` (default `ViaRequest`) so the body
can be consumed at most once. `FromRequestParts` never touches the body.

## 2. The axum::extract::rejection module inventory

The module holds every built-in rejection type. Enums are the top-level
rejection an extractor returns; structs are the granular failure variants.

`#[non_exhaustive]` enums (11):

| Enum | Feature gate |
|------|--------------|
| `BytesRejection` | none |
| `ExtensionRejection` | none |
| `FailedToBufferBody` | none |
| `FormRejection` | none |
| `JsonRejection` | `json` |
| `MatchedPathRejection` | `matched-path` |
| `PathRejection` | none |
| `QueryRejection` | none |
| `RawFormRejection` | none |
| `RawPathParamsRejection` | none |
| `StringRejection` | none |

Granular failure structs (14): `FailedToDeserializeForm`,
`FailedToDeserializeFormBody`, `FailedToDeserializeQueryString`,
`InvalidFormContentType`, `InvalidUtf8`, `JsonDataError`, `JsonSyntaxError`,
`LengthLimitError`, `MatchedPathMissing`, `MissingExtension`,
`MissingJsonContentType`, `MissingPathParams`, `NestedPathRejection`,
`UnknownBodyError`.

Per-extractor mapping:

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
| `NestedPath` | `NestedPathRejection` | used directly as a struct |

Every rejection enum implements `Debug`, `Display`, `std::error::Error`,
`IntoResponse`, and a per-variant `From<InnerType>`. Each exposes two inherent
methods:

```rust
pub fn status(&self) -> StatusCode;   // the HTTP status Axum would have used
pub fn body_text(&self) -> String;    // the plain-text error message
```

## 3. JsonRejection in detail

```rust
#[non_exhaustive]
pub enum JsonRejection {
    JsonDataError(JsonDataError),
    JsonSyntaxError(JsonSyntaxError),
    MissingJsonContentType(MissingJsonContentType),
    BytesRejection(BytesRejection),
}
```

Inherent methods: `body_text(&self) -> String`, `status(&self) -> StatusCode`.

Trait impls: `Debug`, `Display`, `Error`, `IntoResponse`,
`From<JsonDataError>`, `From<JsonSyntaxError>`,
`From<MissingJsonContentType>`, `From<BytesRejection>`.

Variant meanings:

| Variant | Cause | Default status |
|---------|-------|----------------|
| `MissingJsonContentType` | `Content-Type` is not `application/json` | 415 |
| `JsonSyntaxError` | body is not syntactically valid JSON | 400 |
| `JsonDataError` | valid JSON, does not match the target type | 422 |
| `BytesRejection` | the body could not be buffered | 400 |

The `Error::source()` chain on `JsonSyntaxError` and `JsonDataError` leads to
the underlying `serde_json::Error`. The chain can be walked and downcast for
fine-grained diagnostics. The official `customize-extractor-error` example
does NOT do this; it relies on `body_text()`.

## 4. WithRejection<E, R> (axum-extra)

```rust
// available on axum-extra crate feature "with-rejection" only
pub struct WithRejection<E, R>(pub E, pub PhantomData<R>);
```

Generic parameters:

- `E`: an extractor implementing `FromRequest<S>` (or `FromRequestParts<S>`).
- `R`: the custom rejection. `R` MUST implement `IntoResponse` AND
  `From<E::Rejection>`.

On extraction failure, `WithRejection` converts `E`'s rejection into `R` via
`From`, then `R` becomes the response via `IntoResponse`. Used as a handler
argument, both tuple fields are destructured; the second is `PhantomData`,
discarded with `_`:

```rust
async fn handler(
    WithRejection(Json(value), _): WithRejection<Json<Value>, ApiError>,
) { /* value is the extracted payload */ }
```

## 5. The #[from_request] derive attribute

`#[derive(FromRequest)]` and `#[derive(FromRequestParts)]` come from
`axum-macros`, re-exported as `axum::extract::FromRequest` /
`axum::extract::FromRequestParts` behind the `axum` `macros` feature.

Attribute keys on `#[from_request(...)]`:

| Key | Meaning |
|-----|---------|
| `via(Extractor)` | The inner extractor the derive delegates to |
| `rejection(R)` | The type the inner rejection is converted into via `From` |

The type named in `rejection(...)` MUST implement `IntoResponse` and
`From<InnerExtractor::Rejection>`. When a struct bundles several extractors,
`rejection(...)` declares one unified rejection type for all of them.

Rule: a body-consuming derived extractor derives `FromRequest`; a parts-only
one derives `FromRequestParts`. When implementing an extractor by hand, do NOT
implement both `FromRequest` and `FromRequestParts` for the same concrete
type; the official docs state this makes the extractor unusable. The sole
exception is a generic wrapper around another extractor.

## 6. OptionalFromRequest and OptionalFromRequestParts (0.8)

In Axum 0.8 the `Option<T>` extractor is backed by two dedicated traits:

```rust
// axum 0.8 - the traits that back Option<T> as an extractor
pub trait OptionalFromRequestParts<S>: Sized {
    type Rejection: IntoResponse;
    fn from_request_parts(parts: &mut Parts, state: &S)
        -> impl Future<Output = Result<Option<Self>, Self::Rejection>> + Send;
}

pub trait OptionalFromRequest<S, M = ViaRequest>: Sized {
    type Rejection: IntoResponse;
    fn from_request(req: Request, state: &S)
        -> impl Future<Output = Result<Option<Self>, Self::Rejection>> + Send;
}
```

The return type is `Result<Option<Self>, Self::Rejection>`: a genuinely absent
value yields `Ok(None)`, a present but malformed value yields `Err(rejection)`.

In Axum 0.7 there were no such traits. `Option<T>` swallowed all failures and
yielded `None`. The 0.8.0 release note states verbatim: "`Option<Path<T>>` no
longer swallows all error conditions, instead rejecting the request in many
cases." `OptionalFromRequest` impls for `Json` and `Extension` were added in a
0.8.x patch release.

## Sources verified (2026-05-20)

- https://docs.rs/axum/latest/axum/extract/rejection/index.html
- https://docs.rs/axum/latest/axum/extract/rejection/enum.JsonRejection.html
- https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html
- https://docs.rs/axum/latest/axum/extract/trait.FromRequestParts.html
- https://docs.rs/axum-extra/latest/axum_extra/extract/struct.WithRejection.html
- https://docs.rs/axum-macros/latest/axum_macros/
- https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0
