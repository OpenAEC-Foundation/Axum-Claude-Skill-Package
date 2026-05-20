# methods.md : axum-impl-authorization

Complete, verified API signatures for Axum authorization. Every signature
below was verified via WebFetch against the official `docs.rs` pages on
2026-05-20 (axum 0.8.9). The middleware and routing APIs are identical in
Axum 0.7 and 0.8; only the `FromRequestParts` impl form differs (see the
version note at the end).

## axum::middleware::from_fn

```rust
pub fn from_fn<F, T>(f: F) -> FromFnLayer<F, (), T>
```

Turns an `async fn` into a `tower::Layer`. The function `f` is the guard. Use
`from_fn` when the authorization decision needs no application state.

## axum::middleware::from_fn_with_state

```rust
pub fn from_fn_with_state<F, S, T>(state: S, f: F) -> FromFnLayer<F, S, T>
```

Like `from_fn`, but threads an application state value `S` into the guard.
The official docs state verbatim: "For the requirements for the function
supplied see `from_fn`." State surfaces inside the guard as a `State<S>`
extractor.

The official documentation example applies the layer with `route_layer` and
passes the state twice:

```rust
let state = AppState { /* ... */ };

let app = Router::new()
    .route("/", get(|| async { /* ... */ }))
    .route_layer(middleware::from_fn_with_state(state.clone(), my_middleware))
    .with_state(state);
```

## The guard function signature

A `from_fn` / `from_fn_with_state` guard is an `async fn` whose arguments obey
a strict order:

```
[zero or more FromRequestParts extractors], [exactly one FromRequest extractor], Next
```

- `FromRequestParts` extractors (`State<S>`, `Claims`, `HeaderMap`,
  `TypedHeader`) inspect request parts only and come first.
- Exactly one `FromRequest` extractor is allowed; for a guard it is
  `axum::extract::Request`, placed second to last.
- `axum::middleware::Next` is always the final argument.

The guard returns a type that implements `IntoResponse`, commonly
`Result<Response, StatusCode>`.

## axum::middleware::Next

```rust
pub struct Next { /* private fields */ }

impl Next {
    pub async fn run(self, req: Request) -> Response
}
```

`next.run(request).await` continues to the inner service (the next layer or
the handler). A guard that returns `Err(...)` before calling `run`
short-circuits, and the inner handler never executes.

## axum::Router::layer

```rust
pub fn layer<L>(self, layer: L) -> Router<S>
where
    L: Layer<Route> + Clone + Send + Sync + 'static,
    L::Service: Service<Request> + Clone + Send + Sync + 'static,
    <L::Service as Service<Request>>::Response: IntoResponse + 'static,
    <L::Service as Service<Request>>::Error: Into<Infallible> + 'static,
    <L::Service as Service<Request>>::Future: Send + 'static,
```

Applies a layer to EVERY request, including requests that match no route and
fall through to the fallback (404) path. Correct for cross-cutting concerns
(tracing, CORS, compression, timeout). WRONG for authorization guards.

## axum::Router::route_layer

```rust
pub fn route_layer<L>(self, layer: L) -> Self
where
    L: Layer<Route> + Clone + Send + Sync + 'static,
    L::Service: Service<Request> + Clone + Send + Sync + 'static,
    <L::Service as Service<Request>>::Response: IntoResponse + 'static,
    <L::Service as Service<Request>>::Error: Into<Infallible> + 'static,
    <L::Service as Service<Request>>::Future: Send + 'static,
```

The trait bounds are identical to `layer`; the behaviour is not. Verbatim
documentation:

> "This works similarly to `Router::layer` except the middleware will only run
> if the request matches a route. This is useful for middleware that return
> early (such as authorization) which might otherwise convert a `404 Not
> Found` into a `401 Unauthorized`."

Documented example behaviour for a bearer-token guard applied with
`route_layer`:

| Request | Token | Result |
|---------|-------|--------|
| `GET /foo` | valid | `200 OK` |
| `GET /foo` | invalid | `401 Unauthorized` |
| `GET /not-found` | invalid | `404 Not Found` |

`route_layer` only wraps routes that already exist on the router, so it MUST
be called AFTER the `.route(...)` calls it should protect.

## axum::Router::with_state

```rust
pub fn with_state<S2>(self, state: S) -> Router<S2>
```

Provides the state value that `State<S>` extractors (in handlers and in
`from_fn_with_state` guards) read. When a guard uses `from_fn_with_state`, the
same state value is passed both into `from_fn_with_state(state.clone(), ...)`
and into `.with_state(state)`.

## axum::extract::FromRequestParts

```rust
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;

    // axum 0.8: native async fn in trait.
    async fn from_request_parts(parts: &mut Parts, state: &S)
        -> Result<Self, Self::Rejection>;
}
```

A newtype guard extractor (`AdminClaims`) implements `FromRequestParts`. The
`Rejection` type must implement `IntoResponse`; `StatusCode` is the simplest
choice, a custom `AuthError` enum is the production choice.

A guard newtype implements `FromRequestParts` ONLY. The official docs warn
that implementing both `FromRequest` and `FromRequestParts` for the same type
makes the extractor unusable.

## Version note: 0.7 versus 0.8

- Axum 0.8 uses native `async fn` in traits (MSRV Rust 1.75). An
  `impl FromRequestParts for AdminClaims` block needs NO attribute.
- Axum 0.7 predates native async traits. The same `impl` block MUST be
  annotated with `#[async_trait::async_trait]`.
- `from_fn`, `from_fn_with_state`, `Next`, `route_layer`, `layer`, and
  `with_state` are unchanged in shape between 0.7 and 0.8.
- Route path syntax in surrounding `.route(...)` calls differs: `/users/:id`
  on 0.7, `/users/{id}` on 0.8. This is incidental to authorization.

## Sources verified (2026-05-20)

| URL | Used for |
|-----|----------|
| https://docs.rs/axum/latest/axum/middleware/fn.from_fn_with_state.html | `from_fn_with_state` signature, verbatim `route_layer` plus state-double-pass example, "same requirements as `from_fn`" |
| https://docs.rs/axum/latest/axum/struct.Router.html | `layer`, `route_layer`, `with_state` signatures and bounds, verbatim `route_layer` 404-versus-401 text, documented 200/401/404 behaviour |
| https://docs.rs/axum/latest/axum/middleware/fn.from_fn.html | `from_fn` signature, guard function argument-order requirements |
