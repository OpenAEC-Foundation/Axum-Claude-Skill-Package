# anti-patterns.md : axum-impl-file-upload

The six recurring file-upload mistakes in Axum, each with the root cause and
the fix. Verified against the official `docs.rs` pages on 2026-05-20. The
`Multipart` API is identical in Axum 0.7 and 0.8, so every anti-pattern below
applies to both lines.

## AP1: the 2 MB default silently blocks real uploads

```rust
// WRONG: no body limit configured.
let app = Router::new().route("/upload", post(upload));
```

WHY THIS FAILS: Axum applies `DefaultBodyLimit` with a 2 MB ceiling
(2,097,152 bytes) to every body-consuming extractor, `Multipart` included. The
endpoint passes every test with a tiny file and then rejects the first real
upload above 2 MB with a `413`-class response, before the handler body runs.
This is the single most common Axum file-upload bug because the failure does
not appear during development.

FIX: size a `DefaultBodyLimit::max(n)` to the endpoint.

```rust
// CORRECT: explicit 25 MB ceiling.
let app = Router::new()
    .route("/upload", post(upload))
    .layer(DefaultBodyLimit::max(25 * 1024 * 1024));
```

## AP2: a second body extractor in the upload handler

```rust
// WRONG: Json after Multipart, or Multipart not last.
async fn upload(mut multipart: Multipart, Json(meta): Json<Meta>) { /* ... */ }
```

WHY THIS FAILS: `Multipart` implements `FromRequest` because parsing it
consumes the request body. A request body can be consumed exactly once, so
only one body-consuming extractor (`Multipart`, `Json`, `Form`, `String`,
`Bytes`) is allowed per handler, and it must be the last argument. A second
body extractor does not satisfy the handler trait bounds and fails to compile.

FIX: keep one body extractor, place it last, and put all `FromRequestParts`
extractors (`State`, `Path`, `HeaderMap`) before it. Carry metadata as extra
multipart fields or as path and query parameters.

```rust
// CORRECT: parts extractor first, single body extractor last.
async fn upload(State(state): State<AppState>, mut multipart: Multipart) { /* ... */ }
```

## AP3: disable() with no replacement ceiling

```rust
// WRONG: the body limit is removed and never replaced.
let app = Router::new()
    .route("/upload", post(upload))
    .layer(DefaultBodyLimit::disable());
```

WHY THIS FAILS: `DefaultBodyLimit::disable()` removes the body limit entirely.
The server now accepts an arbitrarily large request body. A single malicious
or accidental upload can exhaust memory or fill the disk, a denial-of-service
vector. The endpoint looks healthy until it is attacked.

FIX: pair `disable()` with `RequestBodyLimitLayer::new(n)`, which converts an
oversized body into `413 Payload Too Large`. For most cases, prefer
`DefaultBodyLimit::max(n)` and avoid `disable()` altogether.

```rust
// CORRECT: disable() is always paired with a hard ceiling.
let app = Router::new()
    .route("/upload", post(upload))
    .layer(DefaultBodyLimit::disable())
    .layer(RequestBodyLimitLayer::new(100 * 1024 * 1024));
```

## AP4: buffering a large file with bytes()

```rust
// WRONG: the whole file is pulled into memory.
let data = field.bytes().await.unwrap();
tokio::fs::write("/var/uploads/file.bin", &data).await.unwrap();
```

WHY THIS FAILS: `field.bytes()` reads the entire field body into a `Bytes`
buffer in RAM. For a 1 GB file that is 1 GB resident, and with concurrent
uploads the cost multiplies by the number of in-flight requests. Memory use
becomes proportional to upload size times concurrency, which crashes the
process under load even when each individual upload is within the body limit.

FIX: stream the field with `chunk()` and write each chunk straight to a
`tokio::fs::File`, so peak memory stays at one chunk.

```rust
// CORRECT: one chunk in memory at a time.
let mut file = tokio::fs::File::create(dest).await?;
while let Some(chunk) = field.chunk().await? {
    file.write_all(&chunk).await?;
}
file.flush().await?;
```

`bytes()` and `text()` remain correct for small bounded fields such as text
inputs and small thumbnails.

## AP5: trusting file_name() as a destination path

```rust
// WRONG: the client controls the path.
let name = field.file_name().unwrap();
let mut file = File::create(format!("/var/uploads/{name}")).await?;
```

WHY THIS FAILS: `file_name()` returns the client-supplied `filename=`
attribute. A hostile client sends `../../etc/cron.d/payload` or an absolute
path, and the server writes outside the upload directory. This is a path
traversal vulnerability that can overwrite arbitrary files. `content_type()`
is equally untrusted and must not gate security decisions either.

FIX: the server generates the on-disk name. Use a UUID or timestamp stem, and
take at most a sanitized extension from the client name. Reject or strip path
separators and `..`.

```rust
// CORRECT: server-controlled name, sanitized extension only.
fn safe_dest(client_name: Option<&str>) -> PathBuf {
    let ext = client_name
        .and_then(|n| std::path::Path::new(n).extension())
        .and_then(|e| e.to_str())
        .filter(|e| e.chars().all(|c| c.is_ascii_alphanumeric()))
        .unwrap_or("bin");
    let id = uuid::Uuid::new_v4();
    std::path::Path::new("/var/uploads").join(format!("{id}.{ext}"))
}
```

## AP6: unwrap() on next_field() and chunk() in production

```rust
// WRONG: a malformed body panics the handler.
while let Some(mut field) = multipart.next_field().await.unwrap() {
    let data = field.bytes().await.unwrap();
}
```

WHY THIS FAILS: `next_field()`, `bytes()`, `text()`, and `chunk()` return
`Result`. A truncated or malformed `multipart/form-data` body, or a client
that disconnects mid-upload, surfaces as `Err(MultipartError)`. `.unwrap()`
panics, and Axum turns the panic into a `500 Internal Server Error`. The real
fault is a bad client request, which is a `400`-class condition, and the panic
also pollutes logs and can abort the connection abruptly.

FIX: map `MultipartError` to a `400` response. `MultipartError` implements
`IntoResponse`, so it can also be returned directly through the `?` operator.

```rust
// CORRECT: client error becomes a 400, not a 500.
while let Some(field) = multipart
    .next_field()
    .await
    .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?
{
    let data = field
        .bytes()
        .await
        .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?;
}
```

## Summary table

| Anti-pattern | Symptom | Fix |
|--------------|---------|-----|
| AP1: no body limit set | works for small files, `413` on real files | `DefaultBodyLimit::max(n)` |
| AP2: second body extractor | does not compile | one body extractor, placed last |
| AP3: `disable()` alone | unbounded body, memory or disk DoS | pair with `RequestBodyLimitLayer::new(n)` |
| AP4: `bytes()` on large files | out of memory under load | stream with `chunk()` |
| AP5: `file_name()` as path | path traversal, arbitrary file write | server-generated name, sanitized extension |
| AP6: `unwrap()` on body methods | bad request becomes `500`, handler panics | map `MultipartError` to `400` |
