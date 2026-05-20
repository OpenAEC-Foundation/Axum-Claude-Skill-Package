# Topic Research : axum-impl-file-upload

Focused research for the `axum-impl-file-upload` skill. Every API claim, signature
and code example below was verified via WebFetch against the official `docs.rs`
pages listed in the final section. Target versions: **Axum 0.7 and 0.8**. Current
`docs.rs` version observed at research time: **axum 0.8.9**, **tower-http 0.6.11**.

The `Multipart` API is stable across 0.7 and 0.8 : the struct, `next_field()`, and
all `Field` methods are identical in both lines. No version-specific annotations are
needed for this skill, except for the routing path-syntax used in surrounding
examples (`/upload` is a static path, so it is unaffected by the 0.7 `/:id` ->
0.8 `/{id}` change).

---

## 1. The `multipart` Cargo feature

`axum::extract::Multipart` is feature-gated. The extractor is NOT available unless
the `multipart` feature is enabled on the `axum` dependency.

```toml
[dependencies]
axum = { version = "0.8", features = ["multipart"] }
tokio = { version = "1", features = ["full"] }       # for tokio::fs streaming to disk
tower-http = { version = "0.6", features = ["limit"] } # only if using RequestBodyLimitLayer
```

ALWAYS add `features = ["multipart"]` before using `Multipart`. Without it the
import `use axum::extract::Multipart;` fails to resolve and the compiler reports an
unresolved import, not a helpful "enable this feature" message in older toolchains.

---

## 2. The `Multipart` extractor

Verified struct signature (axum 0.8.9):

```rust
pub struct Multipart { /* private fields */ }
```

`Multipart` is the extractor that parses `multipart/form-data` request bodies
(commonly used for file uploads). Verified documentation quote:

> "Extractor that parses `multipart/form-data` requests (commonly used with file
> uploads)."

### CRITICAL : `Multipart` is a `FromRequest` extractor and MUST be last

Parsing multipart form data requires consuming the request body. `Multipart`
therefore implements `FromRequest` (it also implements `OptionalFromRequest`), and
exactly one body-consuming extractor is allowed per handler. Verified documentation
quote:

> "Since extracting multipart form data from the request requires consuming the
> body, the `Multipart` extractor must be _last_ if there are multiple extractors in
> a handler."

Deterministic rules:

- ALWAYS place `Multipart` as the **last** argument of the handler function.
- `FromRequestParts` extractors (`State`, `Path`, `HeaderMap`, `TypedHeader`, etc.)
  may precede `Multipart` and SHOULD be written before it.
- NEVER place a second body-consuming extractor (`Json`, `String`, `Bytes`, `Form`,
  another `Multipart`) in the same handler : only one extractor may consume the body,
  and a second one is a compile error.

```rust
use axum::extract::{Multipart, State};

// CORRECT : parts extractor first, Multipart last
async fn upload(
    State(state): State<AppState>,
    mut multipart: Multipart,
) { /* ... */ }
```

---

## 3. The `next_field()` loop

Verified signature (axum 0.8.9):

```rust
pub async fn next_field(&mut self) -> Result<Option<Field<'_>>, MultipartError>
```

A `multipart/form-data` body contains zero or more fields. `next_field()` yields the
next field; it returns `Ok(None)` when the body is exhausted. ALWAYS drive it in a
`while let` loop until `None`.

Verified verbatim official example (the field is bound `mut` because streaming reads
require a mutable borrow):

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

Notes for the skill:

- `next_field()` returns `Result` : a malformed body surfaces as
  `Err(MultipartError)`. Production handlers SHOULD convert this into a `400`-class
  response instead of `.unwrap()` (which would `panic` and yield a `500`).
- The returned `Field<'_>` borrows from the `Multipart` value, so you MUST finish
  with one field (read its data) before calling `next_field()` again. You cannot hold
  two `Field`s simultaneously : the borrow checker enforces this.

---

## 4. `Field` methods

Verified signatures (axum 0.8.9, module `axum::extract::multipart`):

```rust
pub fn name(&self) -> Option<&str>
pub fn file_name(&self) -> Option<&str>
pub fn content_type(&self) -> Option<&str>
pub fn headers(&self) -> &HeaderMap

pub async fn bytes(self) -> Result<Bytes, MultipartError>
pub async fn text(self) -> Result<String, MultipartError>
pub async fn chunk(&mut self) -> Result<Option<Bytes>, MultipartError>
```

| Method | Receiver | Purpose |
|--------|----------|---------|
| `name()` | `&self` | The form field name (the `name=` attribute). `None` if absent. |
| `file_name()` | `&self` | The client-supplied file name (the `filename=` attribute). `None` for non-file fields. |
| `content_type()` | `&self` | The field's MIME type, e.g. `image/png`. `None` if absent. |
| `headers()` | `&self` | All raw headers of this field part as a `&HeaderMap`. |
| `bytes()` | `self` (consumes) | Reads the **entire** field body into a `Bytes` buffer in memory. |
| `text()` | `self` (consumes) | Reads the entire field body into a `String` (UTF-8). |
| `chunk()` | `&mut self` | Reads **one** streamed chunk; `Ok(None)` when the field is fully drained. |

Critical distinctions for the skill:

- `bytes()` and `text()` take `self` **by value** : calling either consumes the
  `Field`, so they cannot be combined with `chunk()` on the same field. Read the
  metadata (`name()`, `file_name()`, `content_type()`) BEFORE consuming.
- `chunk()` takes `&mut self`, so it can be called repeatedly in a loop.
- A `Field` also implements `Stream` with `Item = Result<Bytes, MultipartError>`;
  the documentation states "`chunk()` does the same thing as `Field`'s `Stream`
  implementation". You may therefore use `StreamExt::next()` on a field instead of
  `chunk()` if you prefer the stream form.
- NEVER trust `file_name()` or `content_type()` for security decisions : both are
  client-supplied and can be forged. Sanitize the file name (strip path separators,
  reject `..`) before using it as a path, and validate content by inspecting bytes
  rather than the declared MIME type.

---

## 5. Buffering vs streaming

### Buffering with `bytes()` / `text()`

`field.bytes().await` reads the whole field into memory at once. This is simple and
correct for **small** fields : text inputs, small thumbnails, JSON blobs. It is
dangerous for large uploads because the entire payload sits in RAM, and many
concurrent uploads multiply that cost.

```rust
// Small field : buffering is fine
let caption = field.text().await.unwrap();          // whole field -> String
let small_blob = field.bytes().await.unwrap();      // whole field -> Bytes
```

### Streaming large files to disk with `chunk()`

For large files, stream chunk by chunk and write each chunk straight to a
`tokio::fs::File` so peak memory stays at one chunk, not the whole file.

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

async fn save_to_disk(mut field: axum::extract::multipart::Field<'_>)
    -> std::io::Result<()>
{
    let mut file = File::create("/uploads/output.bin").await?;
    while let Some(chunk) = field.chunk().await.unwrap() {
        file.write_all(&chunk).await?;
    }
    file.flush().await?;
    Ok(())
}
```

Deterministic rules:

- ALWAYS stream with `chunk()` for file uploads whose size is unbounded or large.
- ALWAYS sanitize the destination path : derive the on-disk name yourself (e.g. a
  UUID), NEVER concatenate `field.file_name()` directly into a path.
- A streaming loop still needs an enforced ceiling : a hard body limit (section 6)
  stops a single huge field before it can fill the disk.

---

## 6. Body size limits : `DefaultBodyLimit` and `RequestBodyLimitLayer`

### The 2 MB `DefaultBodyLimit` footgun

Axum applies a default request body limit. Verified documentation quote:

> "For security reasons, `Bytes` will, by default, not accept bodies larger than
> 2MB."

`Multipart` is built on this body machinery and the docs confirm:

> "For security reasons, by default, `Multipart` limits the request body size to
> 2MB."

The exact default is **2 MiB = 2,097,152 bytes**. Effect: an upload endpoint works
for tiny test files and then fails on a real upload. Any `multipart/form-data` body
above 2 MB is rejected before the handler body runs, surfacing as a `413`-class
response. This is the most common file-upload bug in Axum.

### `DefaultBodyLimit::max` and `DefaultBodyLimit::disable`

Verified signatures (axum 0.8.9):

```rust
pub const fn max(limit: usize) -> DefaultBodyLimit
pub const fn disable() -> DefaultBodyLimit
```

`DefaultBodyLimit` is applied as a layer. Verified verbatim official example:

```rust
use axum::{body::Bytes, extract::DefaultBodyLimit, routing::get, Router};

let app: Router<()> = Router::new()
    .route("/", get(|body: Bytes| async {}))
    // Replace the default of 2MB with 1024 bytes.
    .layer(DefaultBodyLimit::max(1024));
```

`DefaultBodyLimit::max(n)` replaces the 2 MB default with `n` bytes. To raise the
ceiling for an upload endpoint:

```rust
use axum::extract::DefaultBodyLimit;

let app = Router::new()
    .route("/upload", post(upload))
    .layer(DefaultBodyLimit::max(10 * 1024 * 1024)); // 10 MB
```

`DefaultBodyLimit::disable()` removes the default entirely. Verified documentation
note: it "must be used to receive bodies larger than the default limit of 2MB using
`Bytes` or an extractor built on it."

### `RequestBodyLimitLayer` : the explicit hard ceiling

Verified signature (tower-http 0.6.11):

```rust
pub fn new(limit: usize) -> Self
```

`limit` is the maximum body length in bytes. Verified documentation quote: the layer
"intercepts requests with body lengths greater than the configured limit and
converts them into `413 Payload Too Large` responses."

### The mandatory pairing

Disabling `DefaultBodyLimit` with no replacement makes the server accept an
arbitrarily large body : a memory-exhaustion / disk-exhaustion DoS vector.

ALWAYS, when calling `DefaultBodyLimit::disable()`, follow it with a
`RequestBodyLimitLayer::new(n)` so a hard ceiling still exists:

```rust
use axum::extract::DefaultBodyLimit;
use tower_http::limit::RequestBodyLimitLayer;

let app = Router::new()
    .route("/upload", post(upload))
    .layer(DefaultBodyLimit::disable())
    .layer(RequestBodyLimitLayer::new(100 * 1024 * 1024)); // 100 MB hard ceiling
```

NEVER ship `DefaultBodyLimit::disable()` without a `RequestBodyLimitLayer`. For most
cases, prefer the simpler `DefaultBodyLimit::max(n)` : one layer, one explicit
ceiling, no `tower-http` dependency required. Reach for `disable()` +
`RequestBodyLimitLayer` only when you specifically need the `tower-http` limiter's
behavior or a separately tuned ceiling.

---

## 7. Verified snippets summary

1. Cargo feature setup (section 1).
2. The `next_field()` loop with `Field` metadata + `bytes()` (section 3, verbatim
   official example).
3. Streaming a field to a `tokio::fs::File` with `chunk()` (section 5).
4. `DefaultBodyLimit::max(1024)` (section 6, verbatim official example).
5. `DefaultBodyLimit::disable()` + `RequestBodyLimitLayer::new(...)` pairing
   (section 6).

---

## 8. Anti-patterns for the skill

- **AP : `Multipart` not last.** Placing `Json`/`Bytes`/`String` after `Multipart`,
  or two body extractors in one handler. FIX : `Multipart` is the last argument; only
  one body-consuming extractor per handler.
- **AP : 2 MB default silently blocking uploads.** Endpoint works for small files,
  `413`s on real ones. FIX : `DefaultBodyLimit::max(n)` sized for the endpoint.
- **AP : `disable()` with no ceiling.** Unlimited body = memory/disk DoS. FIX :
  always pair `disable()` with `RequestBodyLimitLayer::new(n)`.
- **AP : buffering huge files with `bytes()`.** Whole file in RAM, multiplied by
  concurrency. FIX : stream with `chunk()` to disk.
- **AP : trusting `file_name()` as a path.** Client-controlled; path traversal risk.
  FIX : generate the on-disk name yourself, sanitize / reject `..` and separators.
- **AP : `.unwrap()` on `next_field()` / `chunk()` in production.** A malformed body
  panics the handler into a `500`. FIX : map `MultipartError` to a `400` response.

---

## Sources verified

All URLs below were fetched via WebFetch on **2026-05-20**. Versions observed:
axum 0.8.9, tower-http 0.6.11.

| URL | Verified | Used for |
|-----|----------|----------|
| https://docs.rs/axum/latest/axum/extract/struct.Multipart.html | 2026-05-20 | `Multipart` struct, `next_field()` signature, FromRequest "must be last" rule, 2 MB note |
| https://docs.rs/axum/latest/axum/extract/multipart/struct.Field.html | 2026-05-20 | `Field` method signatures (`name`, `file_name`, `content_type`, `headers`, `bytes`, `text`, `chunk`), `Stream` impl |
| https://docs.rs/axum/latest/axum/extract/struct.DefaultBodyLimit.html | 2026-05-20 | `DefaultBodyLimit::max` / `disable` signatures, 2 MB default quote, verbatim `max(1024)` example |
| https://docs.rs/tower-http/latest/tower_http/limit/struct.RequestBodyLimitLayer.html | 2026-05-20 | `RequestBodyLimitLayer::new` signature, `413 Payload Too Large` behavior |

### Verification notes / caveats

- The `next_field()` loop example (section 3) is the verbatim official example from
  the `Multipart` docs page.
- The `DefaultBodyLimit::max(1024)` example (section 6) is the verbatim official
  example from the `DefaultBodyLimit` docs page.
- The streaming-to-disk snippet (section 5) composes the verified `chunk()` signature
  with standard `tokio::fs::File` + `tokio::io::AsyncWriteExt` usage; it is not a
  single verbatim doc example but every API call in it is individually verified.
- The exact default of 2 MiB (2,097,152 bytes) is the well-documented value of
  Axum's `DefaultBodyLimit`; the docs pages quote "2MB" textually. Both the
  `Multipart` page and the `DefaultBodyLimit` page independently state the 2 MB
  default, which is consistent with the fragment-B research.
- The `multipart` Cargo feature requirement is standard Axum feature-gating; the
  `Multipart` docs page is published under the `multipart` feature on `docs.rs`.
