# axum-syntax-extractors: Methods and Signatures

All signatures verified via WebFetch against docs.rs on 2026-05-20 (docs.rs
current at research time: axum 0.8.9). The built-in extractor surface is
identical across Axum 0.7 and 0.8.

Sources:

- https://docs.rs/axum/latest/axum/extract/index.html
- https://docs.rs/axum/latest/axum/extract/struct.Path.html
- https://docs.rs/axum/latest/axum/extract/struct.Query.html
- https://docs.rs/axum/latest/axum/struct.Json.html
- https://docs.rs/axum/latest/axum/extract/rejection/enum.JsonRejection.html

## The two extractor traits

```rust
// axum 0.8 - verbatim from docs.rs (native async fn in trait, RPITIT)
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;
    fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}

pub trait FromRequest<S, M = ViaRequest>: Sized {
    type Rejection: IntoResponse;
    fn from_request(
        req: Request<Body>,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

`FromRequestParts` reads request parts only and never consumes the body.
`FromRequest` consumes the body. A blanket impl makes every `FromRequestParts`
type also usable as a `FromRequest` type, so an all-parts handler always
compiles; the constraint only bites a body extractor in the wrong position.

On Axum 0.7 these traits used the `#[async_trait]` attribute macro. On Axum 0.8
they are native async traits. This matters only when implementing a custom
extractor (see `axum-syntax-custom-extractors`); built-in extractors are
unaffected.

## Extractor struct definitions

```rust
// axum 0.7 / 0.8 - verbatim tuple-struct definitions
pub struct Path<T>(pub T);
pub struct Query<T>(pub T);
pub struct Json<T>(pub T);
```

Each is a newtype wrapping the deserialized value. Destructure it in the
handler signature, for example `Path(id): Path<u64>`, or access the field
through `.0`.

## Built-in extractor reference

| Extractor | Trait | Bounds | Notes |
|-----------|-------|--------|-------|
| `Path<T>` | `FromRequestParts` | `T: DeserializeOwned` | captured route segments |
| `Query<T>` | `FromRequestParts` | `T: DeserializeOwned`, `S: Send + Sync` | URL query string via serde |
| `State<T>` | `FromRequestParts` | `T: FromRef<S>` | compile-time-checked app state |
| `Extension<T>` | `FromRequestParts` | `T: Clone + Send + Sync + 'static` | runtime-resolved request extension |
| `HeaderMap` | `FromRequestParts` | none | all request headers, `Infallible` |
| `Method` | `FromRequestParts` | none | the HTTP method, `Infallible` |
| `Uri` | `FromRequestParts` | none | the full request URI, `Infallible` |
| `Json<T>` | `FromRequest` | `T: DeserializeOwned`, `S: Send + Sync` | requires `Content-Type: application/json` |
| `Form<T>` | `FromRequest` | `T: DeserializeOwned` | `application/x-www-form-urlencoded` body |
| `Bytes` | `FromRequest` | none | raw body bytes |
| `String` | `FromRequest` | none | UTF-8 body text |
| `Request` | `FromRequest` | none | the whole request, `Infallible` |

## Path deserialization shapes

`Path<T>` accepts four shapes of `T`:

| Shape | Example `T` | Meaning |
|-------|-------------|---------|
| single primitive | `Path<u64>` | one captured segment |
| tuple | `Path<(Uuid, Uuid)>` | several segments, by position in the route |
| named struct | `Path<Params>` | several segments, by capture name |
| map | `Path<HashMap<String, String>>` | every segment as key/value pairs |

Documented, verbatim: "Any percent encoded parameters will be automatically
decoded. The decoded parameters must be valid UTF-8, otherwise `Path` will fail
and return a `400 Bad Request` response."

## JsonRejection variants

`JsonRejection` is `#[non_exhaustive]` with four verified variants:

| Variant | Cause |
|---------|-------|
| `JsonDataError` | valid JSON that does not deserialize into the target type |
| `JsonSyntaxError` | the body is syntactically invalid JSON |
| `MissingJsonContentType` | the `Content-Type: application/json` header is missing or wrong |
| `BytesRejection` | buffering the request body failed |

Call `JsonRejection::status()` for the exact status code of a variant and
`body_text()` for a human-readable message. Treat the variant names plus
`status()` as the authoritative surface rather than hardcoding numeric codes.

## Wrapper behavior

| Wrapper | Behavior |
|---------|----------|
| `Option<Extractor>` | Axum 0.8: yields `None` when absent, REJECTS when present-but-invalid |
| `Result<Extractor, Extractor::Rejection>` | never rejects; the handler inspects `Ok` / `Err` itself |

On Axum 0.7, `Option<Path<T>>` swallowed every error and yielded `None`. Axum
0.8 removed that behavior through the `OptionalFromRequest` and
`OptionalFromRequestParts` traits.
