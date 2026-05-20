# methods.md : axum-impl-file-upload

Complete, verified API signatures for Axum file uploads. Every signature below
was verified via WebFetch against the official `docs.rs` pages on 2026-05-20
(axum 0.8.9, tower-http 0.6.11). The `Multipart` and `Field` APIs are identical
in Axum 0.7 and 0.8.

## The multipart Cargo feature

`axum::extract::Multipart` is feature-gated. It does NOT exist unless the
`multipart` feature is enabled on the `axum` dependency.

```toml
[dependencies]
axum = { version = "0.8", features = ["multipart"] }
```

Without the feature, `use axum::extract::Multipart;` fails to resolve. The
compiler reports an unresolved import, not a "enable this feature" hint on
older toolchains.

## axum::extract::Multipart

```rust
pub struct Multipart { /* private fields */ }
```

Extractor that parses `multipart/form-data` request bodies, commonly used for
file uploads.

### Trait implementations

- `Multipart` implements `FromRequest`. Parsing multipart data consumes the
  request body, so `Multipart` MUST be the last argument in a handler when
  multiple extractors are present. Exactly one body-consuming extractor is
  allowed per handler.
- `Multipart` also implements `OptionalFromRequest`, so `Option<Multipart>` is
  a valid handler argument.

### next_field

```rust
pub async fn next_field(&mut self) -> Result<Option<Field<'_>>, MultipartError>
```

Yields the next field of the body. Returns `Ok(None)` when the body is
exhausted. Returns `Err(MultipartError)` for a malformed body. Drive it in a
`while let Some(field) = multipart.next_field().await? { ... }` loop.

The returned `Field<'_>` borrows from the `Multipart` value. A field must be
fully processed (its data read) before `next_field()` is called again. Two
`Field` values cannot be held at once; the borrow checker enforces this.

## axum::extract::multipart::Field

```rust
pub struct Field<'a> { /* private fields */ }
```

One field of a `multipart/form-data` body. A field has metadata accessors and
body accessors.

### Metadata accessors

```rust
pub fn name(&self) -> Option<&str>
pub fn file_name(&self) -> Option<&str>
pub fn content_type(&self) -> Option<&str>
pub fn headers(&self) -> &HeaderMap
```

| Method | Returns | Meaning |
|--------|---------|---------|
| `name` | `Option<&str>` | the form field name (the `name=` attribute); `None` if absent |
| `file_name` | `Option<&str>` | the client-supplied file name (the `filename=` attribute); `None` for non-file fields |
| `content_type` | `Option<&str>` | the field MIME type, for example `image/png`; `None` if absent |
| `headers` | `&HeaderMap` | all raw headers of this field part |

All four take `&self` and may be called before reading the body. `file_name`
and `content_type` are client-controlled and MUST NOT be trusted for security
decisions.

### Body accessors

```rust
pub async fn bytes(self)       -> Result<Bytes, MultipartError>
pub async fn text(self)        -> Result<String, MultipartError>
pub async fn chunk(&mut self)  -> Result<Option<Bytes>, MultipartError>
```

| Method | Receiver | Behaviour |
|--------|----------|-----------|
| `bytes` | `self` (consumes the `Field`) | reads the entire field body into a `Bytes` buffer in memory |
| `text` | `self` (consumes the `Field`) | reads the entire field body into a UTF-8 `String` |
| `chunk` | `&mut self` | reads one streamed chunk; returns `Ok(None)` when the field is fully drained |

`bytes()` and `text()` take `self` by value, so calling either consumes the
`Field` and cannot be combined with `chunk()` on the same field. Read all
needed metadata before calling them.

`chunk()` takes `&mut self`, so the field binding must be `mut field` and
`chunk()` can be called repeatedly in a loop.

### Stream implementation

`Field<'a>` implements `futures_core::Stream` with:

```rust
type Item = Result<Bytes, MultipartError>
```

`chunk()` does the same thing as the `Stream` implementation. A field can
therefore be driven with `StreamExt::next()` instead of `chunk()`.

## axum::extract::multipart::MultipartError

The error type returned by `next_field()`, `bytes()`, `text()`, and
`chunk()`. It implements `std::error::Error` and `IntoResponse`, so returning
it from a handler produces a `400`-class response. Production handlers either
return `MultipartError` directly or map it with `.map_err(...)` to a custom
`(StatusCode, String)` response.

## axum::extract::DefaultBodyLimit

```rust
pub const fn max(limit: usize) -> DefaultBodyLimit
pub const fn disable() -> DefaultBodyLimit
```

`DefaultBodyLimit` is applied to a `Router` or `MethodRouter` with `.layer()`.

| Method | Effect |
|--------|--------|
| `DefaultBodyLimit::max(n)` | replace the 2 MB default with `n` bytes |
| `DefaultBodyLimit::disable()` | remove the body limit entirely |

The default limit is 2 MB (2,097,152 bytes). The official documentation states
that `Bytes`, by default, does not accept bodies larger than 2 MB, and the
limit applies to every extractor built on `Bytes`, including `String`, `Json`,
`Form`, and `Multipart`. `disable()` is the documented way to receive bodies
larger than the default using `Bytes` or an extractor built on it.

## tower_http::limit::RequestBodyLimitLayer

```rust
pub fn new(limit: usize) -> Self
```

`limit` is the maximum body length in bytes. The layer intercepts requests
with a body longer than `limit` and converts them into `413 Payload Too Large`
responses. It requires the `limit` feature on `tower-http`:

```toml
tower-http = { version = "0.6", features = ["limit"] }
```

`RequestBodyLimitLayer` is the explicit hard ceiling that MUST follow
`DefaultBodyLimit::disable()`.

## tokio::fs::File and AsyncWriteExt

Streaming a field to disk uses the standard `tokio` async file API:

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

let mut file = File::create(path).await?; // -> std::io::Result<File>
file.write_all(&chunk).await?;            // chunk: Bytes, derefs to &[u8]
file.flush().await?;
```

`tokio::fs::File` requires the `fs` feature on `tokio`; the `full` feature set
includes it.

## Sources verified (2026-05-20)

| URL | Used for |
|-----|----------|
| https://docs.rs/axum/latest/axum/extract/struct.Multipart.html | `Multipart` struct, `next_field` signature, "must be last" rule, 2 MB note |
| https://docs.rs/axum/latest/axum/extract/multipart/struct.Field.html | `Field` method signatures and receivers, `Stream` item type |
| https://docs.rs/axum/latest/axum/extract/struct.DefaultBodyLimit.html | `max` and `disable` signatures, 2 MB default, `disable()` purpose |
| https://docs.rs/tower-http/latest/tower_http/limit/struct.RequestBodyLimitLayer.html | `RequestBodyLimitLayer::new` signature, `413 Payload Too Large` behaviour |
