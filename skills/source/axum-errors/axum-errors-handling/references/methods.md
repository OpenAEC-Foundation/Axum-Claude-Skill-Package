# axum-errors-handling: API Reference

Complete signatures for the types this skill uses. All signatures verified
2026-05-20 against the URLs in the "Sources" section at the bottom.

Version note: every signature below is identical between Axum 0.7 and Axum 0.8.
Error handling is not part of the 0.7 to 0.8 breaking-change surface.

## The tower::Service trait

Every routable Axum service is a `tower::Service`. The trait is generally
fallible, `type Error` can be anything:

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(
        &mut self,
        cx: &mut Context<'_>,
    ) -> Poll<Result<(), Self::Error>>;

    fn call(&mut self, req: Request) -> Self::Future;
}
```

Axum constrains anything routable to `type Error = Infallible`. The official
`error_handling` module documentation states it verbatim: "axum makes sure you
always produce a response by relying on the type system" and "axum does this by
requiring all services have `Infallible` as their error type".

Consequence: a handler returning `Result<T, E>` is still an infallible service.
`Err(E)` is converted into a `Response` because `E: IntoResponse`. The docs:
"If this handler returns `Err(some_status_code)` that will still be converted
into a Response."

## IntoResponse

The trait that makes a type returnable from a handler and usable as the `Err`
side of a handler `Result`:

```rust
pub trait IntoResponse {
    fn into_response(self) -> Response;
}
```

`Response` is `axum::response::Response`, an alias for
`http::Response<axum::body::Body>`. A custom error type becomes a valid handler
error type exactly when it implements `IntoResponse`.

Built-in `IntoResponse` impls relevant to error handling:

| Type | Resulting response |
|------|--------------------|
| `StatusCode` | that status, empty body |
| `(StatusCode, T)` where `T: IntoResponse` | that status plus `T`'s body |
| `(StatusCode, HeaderMap, T)` | status, headers, body |
| `String`, `&'static str` | 200, `text/plain` body |
| `axum::Json<T>` where `T: Serialize` | 200, `application/json` body |
| `Result<T, E>` where `T, E: IntoResponse` | `T` or `E` response |

The `Result` impl is what lets a handler return `Result<T, E>` directly.

## The ? operator and From

`?` inside a handler that returns `Result<T, AppError>` requires
`AppError: From<SourceError>` for the source error type. There is no Axum API
here, it is the standard library `?` desugaring:

```rust
// `expr?` where expr: Result<T, SourceError> desugars to:
match expr {
    Ok(v) => v,
    Err(e) => return Err(AppError::from(e)),
}
```

A missing `From<SourceError> for AppError` impl is the cause of the compile
error "the trait bound `AppError: From<SourceError>` is not satisfied" on a
`?`.

## HandleError

Adapter that turns a fallible `tower::Service` into an infallible one by
running an error handler. Verified definition:

```rust
pub struct HandleError<S, F, T> { /* private fields */ }

impl<S, F, T> HandleError<S, F, T> {
    pub fn new(inner: S, f: F) -> Self;
}
```

Type parameters:

- `S`: the inner, fallible service.
- `F`: the error handler function.
- `T`: the tuple of extractors the error handler accepts.

The docs describe it verbatim as "A `Service` adapter that handles errors by
converting them into responses."

## HandleErrorLayer

The `tower::Layer` form of `HandleError`, for applying the adapter to a layer
stack. Verified definition:

```rust
pub struct HandleErrorLayer<F, T> { /* private fields */ }

impl<F, T> HandleErrorLayer<F, T> {
    pub fn new(f: F) -> Self;
}
```

The docs describe it as a "Layer that applies `HandleError` which is a
`Service` adapter that handles errors by converting them into responses."

Both `HandleError` and `HandleErrorLayer` live in the `axum::error_handling`
module.

## The error handler function

The function passed to `HandleError::new` or `HandleErrorLayer::new` is a
`FnOnce`. It MAY accept extractors that implement `FromRequestParts<()>`. The
service error is ALWAYS the final argument. Verified arity patterns:

```rust
// no extractors
F: FnOnce(S::Error) -> Fut
// one extractor, error last
F: FnOnce(T1, S::Error) -> Fut
// two extractors, error last
F: FnOnce(T1, T2, S::Error) -> Fut
// up to three extractors, error always last
F: FnOnce(T1, T2, T3, S::Error) -> Fut
```

`Fut::Output` must implement `IntoResponse`.

Verified minimal error handler (no extractors), from the `error_handling`
module docs:

```rust
async fn handle_anyhow_error(err: anyhow::Error) -> (StatusCode, String) {
    (
        StatusCode::INTERNAL_SERVER_ERROR,
        format!("Something went wrong: {err}"),
    )
}
```

Error handler that also reads request parts, error argument last:

```rust
async fn handle_timeout_error(
    method: Method,
    uri: Uri,
    err: BoxError,
) -> (StatusCode, String) {
    (
        StatusCode::REQUEST_TIMEOUT,
        format!("`{method} {uri}` failed: {err}"),
    )
}
```

`BoxError` is `tower::BoxError`, an alias for
`Box<dyn std::error::Error + Send + Sync>`, the common boxed error type for
tower layer stacks.

## thiserror derive surface

`thiserror` (crate `thiserror = "2"`) generates `std::error::Error` and
`Display` impls for an error enum or struct. The attributes this skill uses:

| Attribute | Effect |
|-----------|--------|
| `#[derive(thiserror::Error, Debug)]` | derive the `Error` impl; `Debug` is required by `Error` |
| `#[error("text {0}")]` on a variant | generate `Display`; `{0}` interpolates a tuple field |
| `#[error(transparent)]` on a variant | delegate `Display` and `source` to the single wrapped error |
| `#[from]` on a variant field | generate `From<FieldType>` for the enum |

`#[from]` and `#[error(transparent)]` are typically combined on a catch-all
variant: `#[error(transparent)] Unexpected(#[from] anyhow::Error)`.

`thiserror` does NOT generate `IntoResponse`. The author writes the
`IntoResponse` impl by hand and chooses the per-variant status mapping.

## anyhow types

`anyhow` (crate `anyhow = "1"`) provides:

| Item | Role |
|------|------|
| `anyhow::Error` | one opaque error type with a context chain and backtrace |
| `anyhow::Result<T>` | alias for `Result<T, anyhow::Error>` |
| `anyhow::Context` | `.context(...)` / `.with_context(...)` on a `Result` |
| `anyhow::anyhow!("...")` | construct an ad-hoc `anyhow::Error` |

`anyhow::Error` implements `From<E>` for any `E: std::error::Error + Send +
Sync + 'static`. It CANNOT implement `IntoResponse` directly: `anyhow::Error`
is a foreign type and `IntoResponse` is a foreign trait, so the orphan rule
forbids it. A local newtype wrapper is required (see SKILL.md Pattern 4).

## response.extensions_mut

Used to carry the original error alongside a response so a logging layer can
read it without the client ever seeing it:

```rust
fn extensions_mut(&mut self) -> &mut http::Extensions;
```

`Extensions` is a typemap keyed by type. The official `error-handling` example
inserts `Arc<AppError>`:

```rust
response.extensions_mut().insert(Arc::new(err));
```

A downstream `tower_http::trace::TraceLayer` (or a custom layer) reads the
extension and logs the full error; the serialized response body stays generic.

## Sources

Verified 2026-05-20:

- `https://docs.rs/axum/latest/axum/error_handling/index.html`
- `https://docs.rs/axum/latest/axum/error_handling/struct.HandleError.html`
- `https://docs.rs/axum/latest/axum/error_handling/struct.HandleErrorLayer.html`
- `https://docs.rs/axum/latest/axum/response/trait.IntoResponse.html`
- `https://docs.rs/tower/latest/tower/trait.Service.html`
- `https://docs.rs/thiserror/latest/thiserror/`
- `https://docs.rs/anyhow/latest/anyhow/`
- `https://github.com/tokio-rs/axum/blob/main/examples/error-handling/src/main.rs`
- `https://github.com/tokio-rs/axum/blob/main/examples/anyhow-error-response/src/main.rs`
