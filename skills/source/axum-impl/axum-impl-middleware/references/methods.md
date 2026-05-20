# Methods : axum-impl-middleware

Exact API signatures for Axum middleware. Every signature below was
WebFetch-verified against docs.rs on 2026-05-20 (current version observed:
axum 0.8.9). The `axum::middleware` helper API is identical across Axum 0.7 and
0.8; signatures verified against axum 0.7.9 match those for 0.8.9.

## The axum::middleware module

Axum has no middleware system of its own. It integrates Tower
(`tower::Service` plus `tower::Layer`). The `axum::middleware` module provides
helpers so middleware is a plain `async fn`.

| Function | Purpose |
|----------|---------|
| `from_fn` | build a `Layer` from an async function |
| `from_fn_with_state` | same, with application state via `State<S>` |
| `map_request` | transform the request before the handler |
| `map_request_with_state` | `map_request` with state threading |
| `map_response` | transform the response after the handler |
| `map_response_with_state` | `map_response` with state threading |
| `from_extractor` | run an extractor and discard the value (a gate) |
| `from_extractor_with_state` | `from_extractor` with state threading |

## from_fn

```rust
// axum 0.7 / 0.8 - verified verbatim from docs.rs
pub fn from_fn<F, T>(f: F) -> FromFnLayer<F, (), T>
```

The async function `f` must satisfy these requirements (verbatim from
docs.rs):

1. "Be an `async fn`."
2. "Take zero or more `FromRequestParts` extractors."
3. "Take exactly one `FromRequest` extractor as the second to last argument."
4. "Take `Next` as the last argument."
5. "Return something that implements `IntoResponse`."

The deterministic argument order is therefore:

```
[zero or more FromRequestParts extractors] , [one FromRequest extractor] , Next
```

The single `FromRequest` extractor is almost always `Request`. A handler that
violates this order fails the `Handler` trait bound and surfaces the opaque
`the trait bound ... Handler<_, _> is not satisfied` error.

## from_fn_with_state

```rust
// axum 0.7 / 0.8 - verified verbatim from docs.rs
pub fn from_fn_with_state<F, S, T>(state: S, f: F) -> FromFnLayer<F, S, T>
```

The function requirements are identical to `from_fn` (docs.rs: "For the
requirements for the function supplied see `from_fn`"). State is supplied to
the layer constructor and surfaces inside the function as a `State<S>`
extractor. `State` is a `FromRequestParts` extractor, so it appears first,
before `Request` and `Next`.

The same state value MUST be passed twice: into `from_fn_with_state(state
.clone(), f)` and into `Router::with_state(state)`. A missing or
type-mismatched `.with_state()` is a compile-time error.

## map_request and map_response

- `map_request(f)`: `f` takes extractors and returns a `Request`, or a
  `Result<Request, impl IntoResponse>`. It runs before the handler.
- `map_response(f)`: `f` takes a `Response` and returns a `Response`, or
  something `IntoResponse`. It runs after the handler.
- `map_request_with_state` / `map_response_with_state`: add a leading
  `State<S>` extractor, threading state exactly like `from_fn_with_state`.

`map_*` middleware has no `Next`. It cannot inspect both sides of the call and
cannot short-circuit the handler. Use `map_*` for a pure one-way transform;
use `from_fn` when `Next` is needed.

## The Next type

```rust
// axum 0.7 / 0.8 - verified from docs.rs
pub struct Next { /* private fields */ }

impl Next {
    pub async fn run(self, req: Request) -> Response
}
```

- `Next` is "the remainder of a middleware stack, including the handler".
- `Next::run` advances the stack and returns the `Response`. Verified
  verbatim: `pub async fn run(self, req: Request) -> Response`.
- `Next` implements `Clone`, `Debug`, `Service<Request<Body>>`, `Send`,
  `Sync`, and `Unpin`. It has NO generic type parameter in Axum 0.7 or 0.8;
  neither `Next` nor `Request` carries a generic body parameter.
- Code before `next.run(...)` runs on the way in. Code after runs on the way
  out. This is the onion model.
- A middleware that returns a response WITHOUT calling `next.run()`
  short-circuits: the downstream stack and the handler never run.

## Router::layer and Router::route_layer

```rust
// axum 0.7 / 0.8 - route_layer signature, verified against the Router page
pub fn route_layer<L>(self, layer: L) -> Self
where
    L: Layer<Route> + Clone + Send + Sync + 'static
```

| Method | Applies to |
|--------|-----------|
| `.layer()` | every request, INCLUDING the fallback / 404 path |
| `.route_layer()` | ONLY requests that match a registered route |

Verified docs behavior:

- A layer "is only applied to existing routes. So you have to first add your
  routes (and / or fallback) and then call `layer` afterwards." Routes added
  AFTER a `.layer()` call are not wrapped by it.
- With `.route_layer()` and an auth middleware: "`GET /foo` with a valid token
  will receive `200 OK` ... `GET /not-found` with an invalid token will
  receive `404 Not Found`." The gate does not run on the fallback path.

## Layer ordering

- Stacked `Router::layer()` is bottom-to-top: the LAST `.layer()` call is the
  OUTERMOST layer (it sees the request first); the FIRST is the INNERMOST
  (closest to the handler).
- `tower::ServiceBuilder` is top-to-bottom: the FIRST `.layer()` added is the
  OUTERMOST. Stacked `Router::layer()` and `ServiceBuilder` are OPPOSITES.
- Middleware runs AFTER routing, so it cannot rewrite the request URI to
  change which route matched.

## Choosing from_fn versus a hand-written tower::Layer

Use `from_fn` / `from_fn_with_state` when:

- familiar `async`/`await` syntax is wanted and hand-implementing futures is
  not.
- the middleware is application-internal, not published as a crate.
- a single fixed behavior is enough (no configuration knobs).

Write a hand-rolled `tower::Layer` plus `tower::Service` pair when:

- the middleware must be configurable with builder methods.
- it will be published as a reusable crate.
- it needs control over `poll_ready` (backpressure / readiness).
- it must be reusable across any Tower stack, not just Axum.

The full hand-written Tower middleware walkthrough belongs in
`axum-impl-tower-stack`.

## Sources verified

All URLs fetched via WebFetch on 2026-05-20.

- https://docs.rs/axum/latest/axum/middleware/index.html
- https://docs.rs/axum/latest/axum/middleware/fn.from_fn.html
- https://docs.rs/axum/0.7.9/axum/middleware/fn.from_fn.html
- https://docs.rs/axum/latest/axum/middleware/fn.from_fn_with_state.html
- https://docs.rs/axum/latest/axum/middleware/struct.Next.html
- https://docs.rs/axum/latest/axum/struct.Router.html
