---
name: axum-impl-file-upload
description: >
  Use when an Axum handler must accept a file upload or any
  multipart/form-data request: a browser form with an input type=file, an
  image or document upload, or a mixed form that carries text fields and a
  file together.
  Prevents the upload that works for tiny test files and then returns 413 on
  a real file, prevents a second body extractor placed after Multipart,
  prevents buffering a huge file into RAM, prevents a path-traversal hole
  from trusting the client-supplied file name, and prevents an unbounded
  request body DoS after calling disable().
  Covers the multipart Cargo feature, the Multipart extractor and its
  FromRequest "must be last" rule, the next_field loop, Field metadata and
  body methods, buffering with bytes or text versus streaming with chunk to
  a tokio::fs::File, and the DefaultBodyLimit and RequestBodyLimitLayer body
  size ceiling. The Multipart API is identical in Axum 0.7 and 0.8.
  Keywords: axum file upload, Multipart, multipart form-data, next_field,
  Field, bytes, text, chunk, DefaultBodyLimit, RequestBodyLimitLayer,
  multipart feature, tower-http limit feature, 413 Payload Too Large, body
  too large, upload fails on large file, upload works for small files only,
  413 instead of 200, unresolved import Multipart, cannot find Multipart in
  axum, out of memory on upload, second body extractor compile error, path
  traversal upload, how do I accept a file upload, how do I handle a form
  with a file, how do I stream a file to disk, why is my upload rejected.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-impl-file-upload

## Overview

Axum receives file uploads through `axum::extract::Multipart`, the extractor
for `multipart/form-data` request bodies. It is feature-gated behind the
`multipart` Cargo feature and does not exist without it.

Two facts govern every upload handler:

1. `Multipart` consumes the request body, so it is a `FromRequest` extractor
   and MUST be the last argument of the handler. Only one body-consuming
   extractor is allowed per handler.
2. Axum caps request bodies at 2 MB by default through `DefaultBodyLimit`. An
   upload endpoint therefore works for small test files and then rejects real
   files with a `413`-class response until that ceiling is raised.

The `Multipart` API is identical in Axum 0.7 and 0.8. The struct,
`next_field()`, and every `Field` method are the same in both lines, so this
skill contains no version-divergent snippets.

## Quick Reference

Cargo features:

```toml
[dependencies]
axum = { version = "0.8", features = ["multipart"] }
tokio = { version = "1", features = ["full"] }       # streaming to disk
tower-http = { version = "0.6", features = ["limit"] } # only for RequestBodyLimitLayer
```

`Multipart` and `Field`:

| Item | Signature | Notes |
|------|-----------|-------|
| `Multipart` | `pub struct Multipart` | `FromRequest` extractor, must be last |
| `next_field` | `async fn next_field(&mut self) -> Result<Option<Field<'_>>, MultipartError>` | loop until `Ok(None)` |
| `Field::name` | `fn name(&self) -> Option<&str>` | the form `name=` attribute |
| `Field::file_name` | `fn file_name(&self) -> Option<&str>` | client-supplied `filename=`, untrusted |
| `Field::content_type` | `fn content_type(&self) -> Option<&str>` | client-supplied MIME, untrusted |
| `Field::headers` | `fn headers(&self) -> &HeaderMap` | raw part headers |
| `Field::bytes` | `async fn bytes(self) -> Result<Bytes, MultipartError>` | whole field into RAM, consumes `Field` |
| `Field::text` | `async fn text(self) -> Result<String, MultipartError>` | whole field as UTF-8, consumes `Field` |
| `Field::chunk` | `async fn chunk(&mut self) -> Result<Option<Bytes>, MultipartError>` | one streamed chunk |

Body size ceiling:

| Approach | API | When |
|----------|-----|------|
| Raise the limit | `DefaultBodyLimit::max(n)` | a fixed known maximum, no extra crate |
| Remove the default | `DefaultBodyLimit::disable()` | only as a pair with the next row |
| Hard ceiling | `RequestBodyLimitLayer::new(n)` | always follows `disable()` |

Core rules:

- ALWAYS add `features = ["multipart"]` to the `axum` dependency before
  importing `Multipart`. Without it the import fails to resolve.
- ALWAYS place `Multipart` as the last argument of the handler.
- NEVER put a second body-consuming extractor (`Json`, `Form`, `String`,
  `Bytes`, another `Multipart`) in the same handler.
- ALWAYS drive `next_field()` in a `while let Some(field) = ... ` loop until
  it returns `Ok(None)`.
- ALWAYS read `Field` metadata (`name`, `file_name`, `content_type`) BEFORE
  calling `bytes()`, `text()`, or `chunk()`, because `bytes()` and `text()`
  consume the `Field`.
- ALWAYS stream large or unbounded files with `chunk()`; NEVER buffer them
  with `bytes()`.
- ALWAYS keep a hard body ceiling. NEVER call `DefaultBodyLimit::disable()`
  without a following `RequestBodyLimitLayer::new(n)`.
- NEVER trust `file_name()` or `content_type()` for security decisions. Both
  are client-controlled.

## Decision Trees

### Buffer the field or stream it

```
How large is the field being read?
  small and bounded (text input, caption, thumbnail well under ~1 MB)
    -> buffer it: field.text() for text, field.bytes() for binary
  large or unbounded (a user-supplied file of unknown size)
    -> stream it: while let Some(chunk) = field.chunk().await? { write to
       a tokio::fs::File }, and enforce a hard body ceiling
```

### Which body-size ceiling to apply

```
What ceiling does the upload endpoint need?
  a fixed, known maximum (for example 10 MB)
    -> .layer(DefaultBodyLimit::max(10 * 1024 * 1024))   one layer, no extra crate
  the default must be removed for a separate limiter
    -> .layer(DefaultBodyLimit::disable())
       .layer(RequestBodyLimitLayer::new(n))             NEVER disable alone
  uploads always stay under 2 MB
    -> do nothing: the 2 MB DefaultBodyLimit already applies
```

### Handler argument order

```
Does the handler also need State, Path, or headers?
  yes -> write those FromRequestParts extractors first, Multipart LAST
  no  -> Multipart is the only argument
NEVER add Json, Form, String, Bytes, or a second Multipart to the handler.
```

## Patterns

### Pattern: minimal upload handler

Read each field fully before advancing the loop. This is the verbatim
official example.

```rust
use axum::{extract::Multipart, routing::post, Router};

async fn upload(mut multipart: Multipart) {
    while let Some(mut field) = multipart.next_field().await.unwrap() {
        let name = field.name().unwrap().to_string();
        let data = field.bytes().await.unwrap();

        println!("Length of `{}` is {} bytes", name, data.len());
    }
}

let app = Router::new().route("/upload", post(upload));
```

`.unwrap()` here is fine only for a throwaway example. Production handlers
map `MultipartError` to a `400` response (see the error pattern below).

### Pattern: Multipart last with other extractors

`FromRequestParts` extractors (`State`, `Path`, `HeaderMap`) do not touch the
body and SHOULD be written before `Multipart`.

```rust
use axum::extract::{Multipart, State};

// CORRECT: parts extractor first, the body extractor last.
async fn upload(
    State(state): State<AppState>,
    mut multipart: Multipart,
) {
    while let Some(field) = multipart.next_field().await.unwrap() {
        // ... use state and field ...
    }
}
```

Placing `State` after `Multipart` would not compile cleanly, and adding a
second body extractor such as `Json` is always a compile error.

### Pattern: stream a large file to disk

`bytes()` holds the whole field in memory. For files, stream chunk by chunk
so peak memory stays at one chunk. `tokio::fs::File` with `AsyncWriteExt`
provides the async writer.

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

async fn save_to_disk(
    mut field: axum::extract::multipart::Field<'_>,
    dest: &std::path::Path,
) -> std::io::Result<()> {
    let mut file = File::create(dest).await?;
    while let Some(chunk) = field.chunk().await.unwrap() {
        file.write_all(&chunk).await?;
    }
    file.flush().await?;
    Ok(())
}
```

ALWAYS derive `dest` yourself. NEVER concatenate `field.file_name()` straight
into a path. See `references/examples.md` for a sanitized destination helper.

### Pattern: raise the body limit

`DefaultBodyLimit::max(n)` replaces the 2 MB default with `n` bytes. One
layer, no extra crate. The official example caps at 1024 bytes:

```rust
use axum::{body::Bytes, extract::DefaultBodyLimit, routing::get, Router};

let app: Router<()> = Router::new()
    .route("/", get(|body: Bytes| async {}))
    // Replace the default of 2MB with 1024 bytes.
    .layer(DefaultBodyLimit::max(1024));
```

For an upload endpoint, size the ceiling to the endpoint:

```rust
use axum::{extract::DefaultBodyLimit, routing::post, Router};

let app = Router::new()
    .route("/upload", post(upload))
    .layer(DefaultBodyLimit::max(10 * 1024 * 1024)); // 10 MB
```

### Pattern: disable the default and set a hard ceiling

`DefaultBodyLimit::disable()` removes the limit entirely. An unlimited body is
a memory-exhaustion and disk-exhaustion DoS vector, so it MUST be paired with
`RequestBodyLimitLayer`, which converts oversized bodies into
`413 Payload Too Large`.

```rust
use axum::{extract::DefaultBodyLimit, routing::post, Router};
use tower_http::limit::RequestBodyLimitLayer;

let app = Router::new()
    .route("/upload", post(upload))
    .layer(DefaultBodyLimit::disable())
    .layer(RequestBodyLimitLayer::new(100 * 1024 * 1024)); // 100 MB ceiling
```

Prefer `DefaultBodyLimit::max(n)` for most cases. Reach for `disable()` plus
`RequestBodyLimitLayer` only when the `tower-http` limiter is specifically
needed.

### Pattern: handle MultipartError instead of unwrap

`next_field()` and `chunk()` return `Result`. A malformed body surfaces as
`Err(MultipartError)`. `.unwrap()` panics the handler into a `500`; mapping
the error yields a correct `400`-class response.

```rust
use axum::{extract::Multipart, http::StatusCode, response::IntoResponse};

async fn upload(mut multipart: Multipart) -> Result<impl IntoResponse, (StatusCode, String)> {
    while let Some(field) = multipart
        .next_field()
        .await
        .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?
    {
        let data = field
            .bytes()
            .await
            .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?;
        // ... process data ...
        let _ = data;
    }
    Ok(StatusCode::OK)
}
```

`MultipartError` also implements `IntoResponse` directly; see
`references/methods.md`.

## Reference Links

- `references/methods.md`: complete signatures for `Multipart`, `Field`,
  `MultipartError`, `DefaultBodyLimit`, and `RequestBodyLimitLayer`, plus the
  `FromRequest` and `Stream` trait notes.
- `references/examples.md`: full working programs, a sanitized destination
  helper, a mixed text-and-file form handler, and body-limit configurations.
- `references/anti-patterns.md`: the six recurring file-upload mistakes with
  the root cause of each and the fix.

Related skills:

- `axum-impl-tower-stack`: layer ordering, where `DefaultBodyLimit` and
  `RequestBodyLimitLayer` sit in the middleware stack.
- `axum-core-async-performance`: why streaming to disk matters under
  concurrent load.
- `axum-syntax-extractors`: extractor ordering and `FromRequest` versus
  `FromRequestParts`.
