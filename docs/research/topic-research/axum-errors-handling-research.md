# Topic Research : axum-errors-handling

> Skill : `axum-errors-handling`
> Category : errors/
> Target versions : Axum 0.7 and 0.8
> Researched : 2026-05-20
> All API claims WebFetch-verified against docs.rs and the `tokio-rs/axum`
> example crates on 2026-05-20 (see "Sources verified").

This research file supports a single skill: how to handle errors in Axum. It
covers the infallible-handler model, the canonical `AppError` enum pattern, the
`anyhow` newtype wrapper, `thiserror`-versus-`anyhow` guidance, and fallible
`tower::Service` middleware via `HandleError` / `HandleErrorLayer`.

---

## 1. The infallible-handler model (verified)

The single most important mental model: **at the `tower::Service` level, every
Axum service has `Infallible` as its error type**. The official
`error_handling` module documentation states verbatim: "axum makes sure you
always produce a response by relying on the type system" and "axum does this by
requiring all services have `Infallible` as their error type."

The `tower::Service` trait (verified signature) is:

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;
    fn poll_ready(&mut self, cx: &mut Context<'_>)
        -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}
```

A `tower::Service` is fallible in general: `type Error` can be anything. Axum
forbids this for anything routable. A handler that returns `Result<T, E>` does
NOT fail in the `Service` sense. `Err(E)` is still turned into a valid HTTP
`Response`, because `E` implements `IntoResponse`. The docs put it directly:
"If this handler returns `Err(some_status_code)` that will still be converted
into a Response." Therefore `async fn handler() -> Result<String, StatusCode>`
is a fully infallible service: `Err(StatusCode::NOT_FOUND)` produces a 404 via
the built-in `IntoResponse` impl for `StatusCode`.

ALWAYS treat the handler return type as the error channel. NEVER expect a
panic, `unwrap`, or `expect` to be the way an error reaches the client. A panic
aborts the request task and yields an opaque dropped connection, not a clean
status code.

`IntoResponse` is the linchpin. Any type that implements `IntoResponse` can be
returned from a handler, which is what makes a custom error enum usable as a
handler return type and the `?` operator usable inside the handler body.

---

## 2. The canonical AppError pattern (verified, examples/error-handling)

The official `examples/error-handling/src/main.rs` defines an application error
enum and implements `IntoResponse` for it. Verified enum:

```rust
#[derive(Debug)]
enum AppError {
    // The request body contained invalid JSON
    JsonRejection(JsonRejection),
    // Some error from a third-party library
    TimeError(time_library::Error),
}
```

Verified `IntoResponse` impl. Each variant maps to a status code and a JSON
body; the catch-all variant stashes the original error into the response
extensions so a logging layer can see it without leaking internals to the
client:

```rust
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        #[derive(Serialize)]
        struct ErrorResponse {
            message: String,
        }

        let (status, message, err) = match &self {
            AppError::JsonRejection(rejection) => {
                (rejection.status(), rejection.body_text(), None)
            }
            AppError::TimeError(_err) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "Something went wrong".to_owned(),
                Some(self),
            ),
        };

        let mut response =
            (status, AppJson(ErrorResponse { message })).into_response();
        if let Some(err) = err {
            response.extensions_mut().insert(Arc::new(err));
        }
        response
    }
}
```

Verified `From` impls. These are what make the `?` operator work: each foreign
error type gets a `From` conversion into `AppError`:

```rust
impl From<JsonRejection> for AppError {
    fn from(rejection: JsonRejection) -> Self {
        Self::JsonRejection(rejection)
    }
}

impl From<time_library::Error> for AppError {
    fn from(error: time_library::Error) -> Self {
        Self::TimeError(error)
    }
}
```

Inside a handler the conversion is implicit. Verified usage: `let created_at =
Timestamp::now()?;` where `Timestamp::now()` returns
`Result<_, time_library::Error>`. The `?` operator converts the error to
`AppError` via the `From` impl, and `AppError` then becomes an HTTP response via
its `IntoResponse` impl.

The example also derives a custom `AppJson<T>` wrapper so the rejection from its
own JSON extraction routes through `AppError` instead of the default
`JsonRejection` response. Verified:

```rust
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);

impl<T> IntoResponse for AppJson<T>
where
    axum::Json<T>: IntoResponse,
{
    fn into_response(self) -> Response {
        axum::Json(self.0).into_response()
    }
}
```

The three-part recipe to remember: (1) define an `AppError` enum, (2) implement
`IntoResponse` mapping each variant to a status + body, (3) implement `From` for
every source error so `?` converts automatically.

---

## 3. The anyhow newtype wrapper (verified, examples/anyhow-error-response)

`anyhow::Error` cannot directly implement `IntoResponse` because of Rust's
orphan rule: `anyhow::Error` is a foreign type and `IntoResponse` is a foreign
trait, so neither is local to your crate. The fix, verified in the official
`examples/anyhow-error-response/src/main.rs`, is a local newtype wrapper.

Verified newtype:

```rust
// Make our own error that wraps `anyhow::Error`.
struct AppError(anyhow::Error);
```

Verified `IntoResponse` impl (always returns 500, since `anyhow::Error` is
opaque):

```rust
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        )
            .into_response()
    }
}
```

Verified blanket `From` impl. This is what enables `?` on any function whose
error converts into `anyhow::Error`:

```rust
// This enables using `?` on functions that return `Result<_, anyhow::Error>`
// to turn them into `Result<_, AppError>`.
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
```

Verified handler usage:

```rust
async fn handler() -> Result<(), AppError> {
    try_thing()?;
    Ok(())
}
```

NEVER try `impl IntoResponse for anyhow::Error` directly; it does not compile
(orphan rule). ALWAYS wrap it in a local newtype.

---

## 4. thiserror vs anyhow guidance

Both crates model errors; they solve different problems.

- **`thiserror`** generates a structured, exhaustive error enum. Each variant
  carries its own data and `#[from]` attribute, and you control exactly how each
  variant maps to an HTTP status code in the `IntoResponse` impl. ALWAYS use
  `thiserror` when the caller (or the client) needs to distinguish error cases:
  library-quality APIs, 404-versus-422-versus-500 distinctions, any error type
  that must implement `IntoResponse` with per-variant status mapping.

- **`anyhow`** provides one opaque "something went wrong" error type with a
  context chain and a backtrace. It erases the concrete variant. ALWAYS use
  `anyhow` for application glue code where every failure collapses to a single
  500 and you do not branch on the error kind.

The two are not mutually exclusive. **Combining both** is the common
production shape: a `thiserror` enum for known, status-mapped errors plus one
catch-all variant that wraps `anyhow::Error` for everything unanticipated:

```rust
#[derive(thiserror::Error, Debug)]
enum AppError {
    #[error("entity not found")]
    NotFound,
    #[error("invalid input: {0}")]
    Validation(String),
    #[error(transparent)]
    Unexpected(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let status = match self {
            AppError::NotFound => StatusCode::NOT_FOUND,
            AppError::Validation(_) => StatusCode::UNPROCESSABLE_ENTITY,
            AppError::Unexpected(_) => StatusCode::INTERNAL_SERVER_ERROR,
        };
        (status, self.to_string()).into_response()
    }
}
```

The `#[error(transparent)]` + `#[from] anyhow::Error` combination lets `?`
funnel any unclassified error into the `Unexpected` variant while the explicit
variants keep their dedicated status codes. NEVER expose the inner `anyhow`
message to the client for the catch-all variant in production; log it, return a
generic 500 body.

---

## 5. Fallible tower::Service middleware : HandleError / HandleErrorLayer

A `tower::Service` used as middleware CAN legitimately fail with a
non-`Infallible` error type (a timeout layer, a load-shed layer, a hand-written
fallible service). Axum's `Router` will NOT accept such a service directly,
because routable services must be infallible. The adapter that bridges the gap
is `HandleError`.

Verified `HandleError` definition:

```rust
pub struct HandleError<S, F, T> { /* private fields */ }
```

Three type parameters: `S` is the inner (fallible) service, `F` is the error
handler function, `T` is the tuple of extractors the handler accepts. The docs
describe it verbatim as "A `Service` adapter that handles errors by converting
them into responses." It wraps a fallible service and transforms its error into
an HTTP response, so the wrapper itself is infallible. Constructor:
`pub fn new(inner: S, f: F) -> Self`.

Verified `HandleErrorLayer` definition (the `tower::Layer` form, for applying
the adapter to a layer stack):

```rust
pub struct HandleErrorLayer<F, T> { /* private fields */ }
```

Constructor: `pub fn new(f: F) -> Self`. The docs describe it as a "Layer that
applies `HandleError` which is a `Service` adapter that handles errors by
converting them into responses."

Verified error handler shape. The handler is a `FnOnce`. It CAN accept
extractors, and they implement `FromRequestParts<()>`. The service error is
ALWAYS the final argument. Verified function-arity patterns:

```rust
// no extractors
F: FnOnce(S::Error) -> Fut
// one extractor, error last
F: FnOnce(T1, S::Error) -> Fut
// up to three extractors, error always last
F: FnOnce(T1, T2, T3, S::Error) -> Fut
```

The handler's future must output something implementing `IntoResponse`.

Verified minimal error handler (from the `error_handling` module docs):

```rust
async fn handle_anyhow_error(err: anyhow::Error) -> (StatusCode, String) {
    (
        StatusCode::INTERNAL_SERVER_ERROR,
        format!("Something went wrong: {err}"),
    )
}
```

An error handler that also reads request parts puts the error last:

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

Typical application : wrap a fallible tower layer stack so the `Router` accepts
it.

```rust
let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(HandleErrorLayer::new(handle_timeout_error))
            .layer(TimeoutLayer::new(Duration::from_secs(10))),
    );
```

ALWAYS place `HandleErrorLayer` ABOVE the fallible layer it protects in a
`ServiceBuilder` (`ServiceBuilder` applies top-to-bottom, so the error handler
must be outer). NEVER hand a fallible service to `Router::layer` without a
`HandleError` adapter; the router requires `Error = Infallible` and the code
will not compile otherwise.

Note the distinction from handler-level errors. `HandleError` /
`HandleErrorLayer` are ONLY for fallible `tower::Service` middleware. A normal
handler never needs them: its error path is the `Result<T, E>` return type plus
`IntoResponse`, as in sections 2 to 4.

---

## 6. Quick decision guide

- Handler can fail in a recoverable way -> return `Result<T, AppError>`, use
  `?`, give `AppError` an `IntoResponse` impl. (Sections 2 to 4.)
- Need per-error status codes -> `thiserror` enum. (Section 4.)
- Need a single opaque catch-all -> `anyhow` newtype wrapper. (Section 3.)
- Want both -> `thiserror` enum with an `#[error(transparent)] #[from]
  anyhow::Error` variant. (Section 4.)
- A tower middleware `Service` itself fails (timeout, load-shed) -> wrap with
  `HandleError` / `HandleErrorLayer`. (Section 5.)
- Tempted to `unwrap`/`expect`/`panic` in a handler -> do not; convert to a
  response instead. Reserve `expect` for startup invariants (binding the
  listener).

---

## Sources verified

All URLs below were fetched and verified on **2026-05-20**:

- https://docs.rs/axum/latest/axum/error_handling/index.html — infallible
  service model, "all services have `Infallible` as their error type", the
  `Err(...)` is still converted to a `Response` statement, `HandleError` /
  `HandleErrorLayer` descriptions, `handle_anyhow_error` example.
- https://docs.rs/axum/latest/axum/error_handling/struct.HandleErrorLayer.html
  — `pub struct HandleErrorLayer<F, T>`, `pub fn new(f: F) -> Self`, "Layer
  that applies `HandleError`" description.
- https://docs.rs/axum/latest/axum/error_handling/struct.HandleError.html —
  `pub struct HandleError<S, F, T>`, `pub fn new(inner: S, f: F) -> Self`,
  error handler accepts extractors (`FromRequestParts<()>`) with the service
  error as the final argument, function-arity patterns.
- https://docs.rs/tower/latest/tower/trait.Service.html — `Service` trait
  definition: `type Response`, `type Error`, `type Future`, `poll_ready`,
  `call` signatures.
- https://github.com/tokio-rs/axum/blob/main/examples/error-handling/src/main.rs
  — `AppError` enum (`JsonRejection`, `TimeError` variants), `IntoResponse`
  impl, `From<JsonRejection>` / `From<time_library::Error>` impls, `?` operator
  usage (`Timestamp::now()?`), `AppJson<T>` derived `FromRequest` wrapper,
  error stashed into `response.extensions_mut()` wrapped in `Arc`.
- https://github.com/tokio-rs/axum/blob/main/examples/anyhow-error-response/src/main.rs
  — `struct AppError(anyhow::Error)` newtype, `IntoResponse` impl (500 +
  formatted message), blanket `impl<E: Into<anyhow::Error>> From<E> for
  AppError`, handler usage, orphan-rule comment.

### Verification caveats

- The `thiserror` combined-pattern snippet in section 4 is composed from the
  documented `thiserror` derive surface (`#[error(...)]`, `#[from]`,
  `#[error(transparent)]`) plus the verified Axum `IntoResponse` mapping
  pattern; it is idiomatic and compiles, but it is not lifted verbatim from a
  single official Axum example (the official examples show `thiserror` and
  `anyhow` patterns separately, not merged).
- The `handle_timeout_error` extractor example in section 5 follows the
  verified `HandleError` rule that extractors precede the error argument; the
  exact `Method`/`Uri`/`BoxError` argument list is a representative
  application of that rule, not a verbatim doc quote.
