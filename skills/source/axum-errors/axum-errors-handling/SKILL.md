---
name: axum-errors-handling
description: >
  Use when a handler must return an error, when adding a custom error type to
  an Axum service, when the question mark operator will not compile inside a
  handler, when anyhow::Error does not implement IntoResponse, or when a
  fallible tower middleware Service is rejected by the Router.
  Prevents calling unwrap or panic in handlers, returning a 500 where a 404 or
  422 belongs, leaking internal error text to clients, and trying to impl
  IntoResponse for anyhow::Error directly against the orphan rule.
  Covers the infallible-handler model, the AppError enum with IntoResponse and
  From impls, thiserror versus anyhow, combining a thiserror enum with an
  anyhow catch-all variant, and HandleError / HandleErrorLayer for fallible
  tower::Service middleware.
  Keywords: IntoResponse, AppError, thiserror, anyhow, HandleError,
  HandleErrorLayer, Infallible, From impl, question mark operator, orphan rule,
  the trait IntoResponse is not implemented, anyhow does not implement
  IntoResponse, handler panics instead of returning a response, 500 instead of
  404, leaking error details to the client, how do I return an error from an
  axum handler, what is the error type in axum.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Axum Error Handling

## Overview

Axum makes errors a type-system property. At the `tower::Service` level every
routable Axum service has `Infallible` as its error type: a service cannot
"fail", it can only produce a `Response`. A handler that returns
`Result<T, E>` does NOT fail in the `Service` sense. `Err(E)` is still turned
into a valid HTTP `Response`, because `E` implements `IntoResponse`.

This single fact drives every rule in this skill:

1. The handler return type IS the error channel. ALWAYS return
   `Result<T, E>` where `E: IntoResponse`. NEVER use a panic, `unwrap`, or
   `expect` to send an error to the client.
2. `IntoResponse` is the linchpin. A custom error type becomes usable as a
   handler return value, and the `?` operator becomes usable in the handler
   body, exactly when that type implements `IntoResponse`.
3. The `?` operator converts source errors through `From` impls. Every foreign
   error a handler can hit needs a `From<ThatError> for AppError`.
4. `HandleError` / `HandleErrorLayer` are a separate tool, ONLY for a fallible
   `tower::Service` used as middleware. A normal handler NEVER needs them.

Version note: the error-handling APIs in this skill are unchanged between Axum
0.7 and 0.8. The `AppError` enum, `IntoResponse`, `From` impls, and
`HandleError` / `HandleErrorLayer` behave identically. No code below is
version-divergent.

## Quick Reference

| Need | Approach | Notes |
|------|----------|-------|
| Handler can fail recoverably | return `Result<T, AppError>` | `AppError: IntoResponse` |
| Trivial inline error | return `Result<T, StatusCode>` | `StatusCode` is `IntoResponse` |
| Status code plus a message | return `Result<T, (StatusCode, String)>` | tuple is `IntoResponse` |
| Make `?` convert a source error | `impl From<SourceError> for AppError` | one impl per source type |
| Per-error status codes | `thiserror` enum + `IntoResponse` | each variant maps to a status |
| One opaque catch-all error | `anyhow` newtype wrapper | always 500 |
| Known errors plus a catch-all | `thiserror` enum with an `anyhow` variant | the production shape |
| A fallible tower middleware | `HandleError` / `HandleErrorLayer` | adapts `Error` to `Infallible` |
| Log the cause, hide it from the client | stash error in `response.extensions_mut()` | generic body, full error to logs |

Required crates:

```toml
[dependencies]
axum = "0.8"                                  # axum 0.8
# axum = "0.7"                                # axum 0.7
serde = { version = "1", features = ["derive"] }
thiserror = "2"        # for structured, status-mapped error enums
anyhow = "1"           # for an opaque catch-all error
tower = "0.5"          # for HandleErrorLayer with a ServiceBuilder
```

`thiserror` and `anyhow` are independent: add only the one a service needs, or
both when combining the patterns.

## Decision Trees

### Where does my error become a response?

```
Where did the error originate?
|
+- Inside a handler (a query failed, validation failed, a parse failed)
|  `- Return Result<T, E> from the handler. E: IntoResponse turns Err into
|     a Response. This covers almost every error. Patterns 1 to 5.
|
`- Inside a tower::Service used as middleware (a TimeoutLayer, a load-shed
   layer, a hand-written fallible Service) that has a non-Infallible Error
   `- Wrap it with HandleError / HandleErrorLayer so the Router accepts it.
      Pattern 6.
```

### thiserror, anyhow, or both?

```
Does the caller or the client need to tell error cases apart?
|
+- YES: a 404 must differ from a 422 must differ from a 500
|  `- thiserror enum. Each variant maps to its own status code. Pattern 3.
|
+- NO: every failure collapses to one generic 500, no branching on the kind
|  `- anyhow newtype wrapper. One opaque error, always 500. Pattern 4.
|
`- SOME errors are known and status-mapped, the rest are unanticipated glue
   `- thiserror enum with an #[error(transparent)] #[from] anyhow::Error
      variant. Known variants keep their status; everything else is a 500.
      Pattern 5. This is the common production shape.
```

### Tempted to unwrap, expect, or panic in a handler?

```
About to write .unwrap() / .expect() / panic! inside an async handler?
|
+- The value comes from request input or a runtime operation (DB, parse, IO)
|  `- NEVER. A panic aborts the request task and drops the connection with no
|     clean status. Convert to a response: return Result and use ?.
|
`- The value is a startup invariant (binding the TcpListener, reading a
   mandatory env var in main, before serving begins)
   `- expect is acceptable in main only. The process SHOULD refuse to start.
      It is NOT acceptable inside a handler.
```

## Patterns

### Pattern 1: Return Result, never panic

A handler signals failure through its return type. Any `E: IntoResponse` is a
valid error type. The simplest options need no custom type at all.

```rust
use axum::http::StatusCode;

// StatusCode alone implements IntoResponse: Err(...) becomes that status.
async fn find_widget(id: u64) -> Result<String, StatusCode> {
    if id == 0 {
        return Err(StatusCode::NOT_FOUND);
    }
    Ok(format!("widget {id}"))
}

// A (StatusCode, String) tuple implements IntoResponse: status plus a body.
async fn parse_amount(raw: &str) -> Result<i64, (StatusCode, String)> {
    raw.parse::<i64>()
        .map_err(|e| (StatusCode::BAD_REQUEST, format!("bad amount: {e}")))
}
```

NEVER reach for `unwrap`, `expect`, or `panic!` to report an error to a
client. A panic aborts the request task: the client receives a dropped
connection or an opaque 500 with no useful body, and the failure is invisible
in normal logs. `expect` is acceptable ONLY for startup invariants in `main`
(binding the listener, a mandatory env var), where refusing to start is the
correct behavior.

### Pattern 2: The AppError enum (IntoResponse plus From plus ?)

For anything beyond a one-line handler, define one application error type.
The recipe has three fixed parts. This is the official `examples/error-handling`
shape, with no derive macros.

Part 1: the enum, one variant per error category.

```rust
use axum::extract::rejection::JsonRejection;

#[derive(Debug)]
enum AppError {
    // The request body contained invalid JSON.
    JsonRejection(JsonRejection),
    // Some error from a third-party library.
    TimeError(time_library::Error),
}
```

Part 2: `IntoResponse`, mapping each variant to a status and a body. The
catch-all variant stashes the original error into the response extensions so a
logging layer can see it WITHOUT leaking internals to the client.

```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use serde::Serialize;
use std::sync::Arc;

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
            (status, axum::Json(ErrorResponse { message })).into_response();
        if let Some(err) = err {
            // Logged by a middleware layer; never serialized to the client.
            response.extensions_mut().insert(Arc::new(err));
        }
        response
    }
}
```

Part 3: one `From` impl per source error. This is what makes `?` convert
automatically.

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

Inside a handler the conversion is implicit. `?` converts the source error to
`AppError` via `From`, and `AppError` becomes a `Response` via `IntoResponse`:

```rust
async fn create() -> Result<String, AppError> {
    let created_at = Timestamp::now()?; // time_library::Error -> AppError
    Ok(format!("created at {created_at:?}"))
}
```

ALWAYS keep all three parts in sync: a new source error needs a new variant, a
new `IntoResponse` match arm, and a new `From` impl. A missing `From` impl is
the cause of "the trait `From<...>` is not implemented" on a `?`. The official
example also derives an `AppJson<T>` wrapper so its own JSON extraction
rejection routes through `AppError`; that full wrapper is in
`references/examples.md`.

### Pattern 3: thiserror for structured, status-mapped errors

`thiserror` generates the boilerplate of Pattern 2. `#[from]` writes the
`From` impl; `#[error("...")]` writes `Display`. The author still controls the
status mapping in `IntoResponse`.

```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};

#[derive(thiserror::Error, Debug)]
enum AppError {
    #[error("entity not found")]
    NotFound,

    #[error("invalid input: {0}")]
    Validation(String),

    #[error("database error")]
    Database(#[from] sqlx::Error), // #[from] generates From<sqlx::Error>
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let status = match &self {
            AppError::NotFound => StatusCode::NOT_FOUND,
            AppError::Validation(_) => StatusCode::UNPROCESSABLE_ENTITY,
            AppError::Database(_) => StatusCode::INTERNAL_SERVER_ERROR,
        };
        // Use the per-variant message ONLY for client-safe variants.
        let body = match &self {
            AppError::Database(_) => "internal server error".to_owned(),
            other => other.to_string(),
        };
        (status, body).into_response()
    }
}
```

ALWAYS use `thiserror` when the client must tell error cases apart: a 404 must
differ from a 422 must differ from a 500. NEVER serialize the `Display` text of
an internal variant (a wrapped `sqlx::Error`) to the client; map it to a
generic body and log the detail. `thiserror` is a compile-time helper only: it
adds no runtime cost and no dependency on `anyhow`.

### Pattern 4: The anyhow newtype wrapper

`anyhow::Error` is one opaque "something went wrong" type with a context chain.
It CANNOT implement `IntoResponse` directly: `anyhow::Error` is a foreign type
and `IntoResponse` is a foreign trait, so the orphan rule forbids the impl. The
fix, from the official `examples/anyhow-error-response`, is a local newtype.

```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};

// A local newtype around anyhow::Error: the orphan-rule workaround.
struct AppError(anyhow::Error);

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        )
            .into_response()
    }
}

// The blanket From: enables ? on any function whose error converts into
// anyhow::Error, turning Result<_, E> into Result<_, AppError>.
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
```

```rust
async fn handler() -> Result<(), AppError> {
    try_thing()?; // any anyhow-compatible error converts into AppError
    Ok(())
}
```

NEVER write `impl IntoResponse for anyhow::Error`: it does not compile (orphan
rule). ALWAYS wrap it in a local newtype. The blanket `From` is greedy: it
covers EVERY error type. That is why a plain `anyhow` newtype yields exactly
one status code, 500. Use it ONLY when no error needs its own status. The
example formats the error into the body for clarity; in production return a
generic body and log the detail (see `references/anti-patterns.md`).

### Pattern 5: Combining thiserror with an anyhow catch-all

The production shape: a `thiserror` enum for known, status-mapped errors plus
one catch-all variant wrapping `anyhow::Error` for everything unanticipated.

```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};

#[derive(thiserror::Error, Debug)]
enum AppError {
    #[error("entity not found")]
    NotFound,

    #[error("invalid input: {0}")]
    Validation(String),

    // transparent: delegate Display/source to the inner anyhow::Error.
    // from: generate From<anyhow::Error> for the Unexpected variant.
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

The `#[error(transparent)]` plus `#[from] anyhow::Error` combination lets `?`
funnel any unclassified error into `Unexpected`, while the explicit variants
keep their dedicated status codes. ALWAYS add a known variant when an error
needs its own status; let it fall through to `Unexpected` only when a generic
500 is correct. NEVER expose the inner `anyhow` message of `Unexpected` to the
client in production: match on it, log the detail, return a generic 500 body.

### Pattern 6: Fallible tower::Service middleware (HandleError)

A `tower::Service` used as middleware CAN legitimately fail with a
non-`Infallible` `Error` type: a `TimeoutLayer`, a load-shed layer, a
hand-written fallible service. `Router::layer` will NOT accept such a service,
because routable services must be infallible. `HandleError` is the adapter: it
wraps a fallible service and runs an error handler that converts the error into
a `Response`, so the wrapper itself is infallible.

The error handler is a `FnOnce`. It MAY accept extractors (they implement
`FromRequestParts<()>`); the service error is ALWAYS the final argument. Its
future must output something implementing `IntoResponse`.

```rust
use axum::http::{Method, StatusCode, Uri};
use axum::{routing::get, Router};
use axum::error_handling::HandleErrorLayer;
use tower::{ServiceBuilder, BoxError};
use tower::timeout::TimeoutLayer;
use std::time::Duration;

// Error handler: extractors first, the service error (BoxError) last.
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

fn app() -> Router {
    Router::new().route("/", get(handler)).layer(
        ServiceBuilder::new()
            // HandleErrorLayer is ABOVE the fallible layer it protects.
            .layer(HandleErrorLayer::new(handle_timeout_error))
            .layer(TimeoutLayer::new(Duration::from_secs(10))),
    )
}
```

ALWAYS place `HandleErrorLayer` ABOVE the fallible layer in a `ServiceBuilder`:
`ServiceBuilder` applies top-to-bottom, so the error handler must be the outer
layer to catch the inner layer's error. NEVER hand a fallible service to
`Router::layer` without a `HandleError` adapter; the router requires
`Error = Infallible` and the code will not compile.

`HandleError` / `HandleErrorLayer` are ONLY for fallible `tower::Service`
middleware. A normal handler NEVER needs them: its error path is the
`Result<T, E>` return type plus `IntoResponse`, as in Patterns 1 to 5.

## Reference Links

- `references/methods.md` complete API signatures: the `tower::Service` trait
  and its `Infallible` constraint, `IntoResponse`, `HandleError<S, F, T>` and
  `HandleErrorLayer<F, T>` with their constructors and error-handler arity
  rules, the `thiserror` derive surface, the `anyhow` types.
- `references/examples.md` working, version-annotated code: a full
  `error-handling` service with the `AppJson<T>` derived wrapper, the complete
  `anyhow` newtype service, the combined `thiserror` plus `anyhow` service, and
  a `HandleErrorLayer` timeout stack.
- `references/anti-patterns.md` real mistakes and why each fails: `unwrap` in a
  handler, `impl IntoResponse for anyhow::Error`, leaking internal error text,
  a missing `From` impl breaking `?`, a fallible service handed to
  `Router::layer`, `HandleErrorLayer` placed below the layer it protects.

Related skills: `axum-syntax-responses` (the `IntoResponse` trait and its
built-in impls), `axum-errors-extractor-rejections` (the `Rejection` types and
`WithRejection`), `axum-impl-database` (mapping `sqlx::Error` to `AppError`).

Sources: `https://docs.rs/axum/latest/axum/error_handling/index.html`,
`https://docs.rs/axum/latest/axum/error_handling/struct.HandleError.html`,
`https://docs.rs/axum/latest/axum/error_handling/struct.HandleErrorLayer.html`,
`https://docs.rs/tower/latest/tower/trait.Service.html`,
`https://github.com/tokio-rs/axum/blob/main/examples/error-handling/src/main.rs`,
`https://github.com/tokio-rs/axum/blob/main/examples/anyhow-error-response/src/main.rs`.
