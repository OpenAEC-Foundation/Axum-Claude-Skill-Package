# Anti-patterns: axum-errors-extractor-rejections

Each entry is a real mistake, the exact symptom it produces, the root cause,
and the fix. Verified against docs.rs and the official
`customize-extractor-error` example (2026-05-20).

## Anti-pattern 1: matching a rejection enum without a catch-all arm

```rust
// WRONG - does not compile
match rejection {
    JsonRejection::JsonSyntaxError(_) => /* ... */,
    JsonRejection::JsonDataError(_)   => /* ... */,
    JsonRejection::MissingJsonContentType(_) => /* ... */,
    JsonRejection::BytesRejection(_)  => /* ... */,
}
```

Symptom: `error[E0004]: non-exhaustive patterns` even though every visible
variant is listed.

Root cause: every rejection enum in `axum::extract::rejection` is
`#[non_exhaustive]`. From outside the `axum` crate the compiler treats the enum
as if more variants could exist, because Axum is allowed to add failure
variants in a minor release without it being a breaking change.

Fix: ALWAYS add a `_ =>` arm. Make it useful by delegating to the rejection's
own `status()` and `body_text()`:

```rust
// CORRECT
match rejection {
    JsonRejection::JsonSyntaxError(_) => /* ... */,
    // ... other arms ...
    rejection => (rejection.status(), rejection.body_text()).into_response(),
}
```

## Anti-pattern 2: inventing a status code instead of reusing the rejection's

```rust
// WRONG - hard-codes 400 for every JSON failure
Err(_rejection) => (StatusCode::BAD_REQUEST, "bad request").into_response(),
```

Symptom: a missing `Content-Type` header returns `400` instead of the correct
`415 Unsupported Media Type`; a type mismatch returns `400` instead of `422
Unprocessable Entity`. Clients cannot distinguish a malformed body from a
schema mismatch.

Root cause: each rejection variant carries the semantically correct status.
Discarding the rejection with `_` throws that information away.

Fix: ALWAYS build the response from `rejection.status()` and
`rejection.body_text()`. They return exactly what Axum would have used:

```rust
// CORRECT
Err(rejection) => (rejection.status(), rejection.body_text()).into_response(),
```

## Anti-pattern 3: assuming Option<T> swallows all errors on Axum 0.8

```rust
// WRONG ASSUMPTION on Axum 0.8
async fn handler(id: Option<Path<u32>>) -> String {
    match id {
        Some(Path(id)) => format!("id {id}"),
        None => "no id".to_owned(), // expected to also catch /abc
    }
}
```

Symptom: after upgrading 0.7 to 0.8, a request like `/abc` (non-numeric where
`u32` is expected) returns a `400` rejection instead of running the `None`
branch. Handlers that used `Option` as a soft fallback start returning 4xx.

Root cause: Axum 0.8 backs `Option<T>` with `OptionalFromRequest` /
`OptionalFromRequestParts`. The 0.8.0 release note states verbatim:
"`Option<Path<T>>` no longer swallows all error conditions, instead rejecting
the request in many cases." Genuinely absent yields `None`; present but
malformed yields a rejection. Axum 0.7 swallowed everything into `None`.

Fix: after a 0.7 to 0.8 upgrade, audit every `Option<Extractor>` handler
argument. To keep the old swallow-all behavior, use `Result<T, T::Rejection>`
and map `Err(_)` to your default:

```rust
// CORRECT on 0.8 - explicit "any failure is absent"
async fn handler(id: Result<Path<u32>, PathRejection>) -> String {
    match id {
        Ok(Path(id)) => format!("id {id}"),
        Err(_)       => "no id".to_owned(),
    }
}
```

## Anti-pattern 4: scattering Result<T, Rejection> across every handler

```rust
// WRONG at scale - the same match copied into 40 handlers
async fn h1(p: Result<Json<A>, JsonRejection>) -> Response { /* match ... */ }
async fn h2(p: Result<Json<B>, JsonRejection>) -> Response { /* match ... */ }
// ... 38 more ...
```

Symptom: the rejection-to-response mapping is duplicated in every handler.
Changing the error shape means editing every handler; handlers drift apart.

Root cause: `Result<T, T::Rejection>` is a per-handler tool. It is correct for
one or two handlers, wrong as a codebase-wide strategy.

Fix: for a uniform error shape across many handlers, define the mapping once.
Use `WithRejection<E, R>` from `axum-extra`, or a `#[derive(FromRequest)]`
wrapper with `rejection(...)`. The mapping then lives in one `IntoResponse` and
one `From` impl.

## Anti-pattern 5: implementing both FromRequest and FromRequestParts

```rust
// WRONG - the extractor becomes unusable
struct MyExtractor(String);
impl<S> FromRequest<S> for MyExtractor { /* ... */ }
impl<S> FromRequestParts<S> for MyExtractor { /* ... */ }
```

Symptom: handlers that take `MyExtractor` fail the `Handler` trait bound;
errors point at extractor ordering or at `Handler is not satisfied`.

Root cause: the official docs state that implementing both `FromRequest` and
`FromRequestParts` for the same concrete type makes the extractor unusable,
because Axum cannot decide which impl applies.

Fix: implement exactly one. A body-consuming extractor implements
`FromRequest` only. A parts-only extractor implements `FromRequestParts` only.
The single exception is a generic wrapper around another extractor (for
example a `Wrap<E>`), where implementing both is correct and intended.

## Anti-pattern 6: a custom Rejection type that does not implement IntoResponse

```rust
// WRONG - MyRejection does not implement IntoResponse
struct MyRejection { code: u16 }

impl<S> FromRequest<S> for MyExtractor {
    type Rejection = MyRejection; // compile error: bound not satisfied
    // ...
}
```

Symptom: `error[E0277]: the trait bound MyRejection: IntoResponse is not
satisfied`.

Root cause: both extractor traits bound the associated type with
`type Rejection: IntoResponse`. Axum needs to turn a failed extraction
directly into an HTTP response, so the rejection MUST be a response.

Fix: implement `IntoResponse` for the rejection type, or use a type that
already does. A `(StatusCode, String)` or `(StatusCode, Json<Value>)` tuple
implements `IntoResponse` out of the box and is a common rejection type.

## Anti-pattern 7: forgetting the feature flags

```rust
// WRONG - import fails, the API is feature-gated
use axum_extra::extract::WithRejection;       // needs "with-rejection"
use axum::extract::FromRequest;               // derive needs "macros"
```

Symptom: `unresolved import` for `WithRejection`, or `cannot find derive macro
FromRequest` even though the path looks right.

Root cause: `WithRejection` is behind the `axum-extra` `with-rejection`
feature; the `#[derive(FromRequest)]` and `#[derive(FromRequestParts)]` macros
are behind the `axum` `macros` feature. Neither is enabled by default.

Fix: enable the features in `Cargo.toml`:

```toml
axum = { version = "0.8", features = ["macros"] }
axum-extra = { version = "0.10", features = ["with-rejection"] }
```

## Sources verified (2026-05-20)

- https://docs.rs/axum/latest/axum/extract/rejection/index.html
- https://docs.rs/axum/latest/axum/extract/rejection/enum.JsonRejection.html
- https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html
- https://docs.rs/axum/latest/axum/extract/trait.FromRequestParts.html
- https://docs.rs/axum-extra/latest/axum_extra/extract/struct.WithRejection.html
- https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0
