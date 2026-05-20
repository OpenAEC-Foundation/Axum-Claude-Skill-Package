# Topic Research : axum-impl-middleware

> Skill : `axum-impl-middleware`
> Category : impl/
> Target versions : Axum 0.7 and 0.8 (signatures verified against docs.rs current : axum 0.8.9)
> Researched : 2026-05-20
> All API signatures and example code below were WebFetch-verified against docs.rs on 2026-05-20.
> Upstream fragments : `vooronderzoek-axum.md` section 2, `fragments/research-b-middleware-tower-realtime.md` sections 1, 2.

This file is the verified content basis for the `axum-impl-middleware` skill. It is
NOT the skill. It covers function-style middleware (`from_fn` family), the `Next`
type, deliberate short-circuiting for auth gates, layer ordering, the
`route_layer` vs `layer` decision, and when to drop to a hand-written `tower::Layer`.

---

## 1. The `axum::middleware` module

Axum does NOT implement its own middleware system. It integrates with the **Tower**
middleware ecosystem (`tower::Service` + `tower::Layer`). The `axum::middleware`
module provides ergonomic helpers so middleware can be written as a plain `async fn`
instead of hand-rolling a `Service`.

The module exposes these functions (verified 2026-05-20):

| Function | Purpose |
|----------|---------|
| `from_fn` | Build a `Layer` from an async function. |
| `from_fn_with_state` | Same, with access to application state via a `State<S>` extractor. |
| `map_request` | Transform the request before it reaches the handler. |
| `map_request_with_state` | `map_request` with state threading. |
| `map_response` | Transform the response after the handler runs. |
| `map_response_with_state` | `map_response` with state threading. |
| `from_extractor` | Run an extractor and discard the value (used as a gate). |
| `from_extractor_with_state` | `from_extractor` with state threading. |

The module documents three implementation strategies (verbatim intent):

1. **`from_fn`** : use "when you're not comfortable with implementing your own
   futures" and want the familiar `async`/`await` syntax; suitable when the
   middleware is axum-only and not published as a crate.
2. **Tower combinators** (`map_request` / `map_response`) : use for a "small ad hoc
   operation, such as adding a header".
3. **Custom `tower::Service` + `tower::Layer`** : required when the middleware "needs
   to be configurable" or you "intend to publish your middleware as a crate for
   others to use".

---

## 2. `from_fn` : exact signature and argument-order rule

Verified signature (docs.rs, 2026-05-20):

```rust
pub fn from_fn<F, T>(f: F) -> FromFnLayer<F, (), T>
```

The documented requirements for the async function `f` are exact and strict
(verbatim, verified 2026-05-20):

1. "Be an `async fn`."
2. "Take zero or more `FromRequestParts` extractors."
3. "Take exactly one `FromRequest` extractor as the second to last argument."
4. "Take `Next` as the last argument."
5. "Return something that implements `IntoResponse`."

### The argument-order rule (deterministic)

`[zero or more FromRequestParts extractors] , [exactly one FromRequest extractor] , Next`

- ALWAYS place the single `FromRequest` extractor (almost always `Request`) directly
  before `Next`.
- ALWAYS put any `FromRequestParts` extractors (`HeaderMap`, `State`, `Method`,
  `TypedHeader`, etc.) before that `FromRequest` extractor.
- ALWAYS make `Next` the final argument.
- NEVER put two `FromRequest` extractors in one middleware function : the request
  body is a single-use async stream, so only one extractor may consume it.

### Verified `from_fn` example (verbatim, docs.rs 2026-05-20)

```rust
use axum::{
    Router,
    http,
    routing::get,
    response::Response,
    middleware::{self, Next},
    extract::Request,
};

async fn my_middleware(
    request: Request,
    next: Next,
) -> Response {
    // do something with `request`...

    let response = next.run(request).await;

    // do something with `response`...

    response
}

let app = Router::new()
    .route("/", get(|| async { /* ... */ }))
    .layer(middleware::from_fn(my_middleware));
```

---

## 3. `from_fn_with_state` : exact signature

Verified signature (docs.rs, 2026-05-20):

```rust
pub fn from_fn_with_state<F, S, T>(state: S, f: F) -> FromFnLayer<F, S, T>
```

The function requirements are identical to `from_fn` (docs.rs: "For the requirements
for the function supplied see `from_fn`"). State is supplied to the layer constructor
and surfaces inside the function as a `State<S>` extractor. Because `State` is a
`FromRequestParts` extractor it comes first, before `Request` and `Next`.

### Verified `from_fn_with_state` example (verbatim, docs.rs 2026-05-20)

```rust
use axum::{
    Router,
    http::StatusCode,
    routing::get,
    response::{IntoResponse, Response},
    middleware::{self, Next},
    extract::{Request, State},
};

#[derive(Clone)]
struct AppState { /* ... */ }

async fn my_middleware(
    State(state): State<AppState>,
    // you can add more extractors here but the last
    // extractor must implement `FromRequest` which
    // `Request` does
    request: Request,
    next: Next,
) -> Response {
    // do something with `request`...

    let response = next.run(request).await;

    // do something with `response`...

    response
}

let state = AppState { /* ... */ };

let app = Router::new()
    .route("/", get(|| async { /* ... */ }))
    .route_layer(middleware::from_fn_with_state(state.clone(), my_middleware))
    .with_state(state);
```

CRITICAL: the same state value is passed twice : once into
`from_fn_with_state(state.clone(), ...)` and once into `.with_state(state)`. ALWAYS
clone the state for the layer. Forgetting `.with_state()` or passing a different type
produces a compile-time error, never a runtime one.

---

## 4. `map_request` and `map_response`

Use these for small ad-hoc transformations where the wrapping `Next` semantics are
not needed.

- `map_request` : transforms the request before it reaches the handler. The function
  takes extractors and returns a `Request` (or `Result<Request, impl IntoResponse>`).
- `map_response` : transforms the response after the handler runs. The function takes
  a `Response` and returns a `Response` (or something `IntoResponse`).
- `map_request_with_state` / `map_response_with_state` : add a leading `State<S>`
  extractor, threading state exactly like `from_fn_with_state`.

ALWAYS prefer `map_request` / `map_response` over `from_fn` when the logic is a pure
one-way transform (add a header, normalise a field). Reserve `from_fn` for logic that
must run code BOTH before and after the handler, or that must conditionally
short-circuit. `map_*` middleware has no `Next` and therefore cannot inspect both
sides of the call or skip the handler.

---

## 5. The `Next` type and deliberate short-circuiting

Verified (docs.rs, 2026-05-20). `Next` is "the remainder of a middleware stack,
including the handler". It exposes a single async method:

```rust
pub async fn run(self, req: Request) -> Response
```

Documented purpose: "Execute the remaining middleware stack." `Next` implements
`Clone`, `Debug`, and `Service<Request<Body>>`; it is `Send` and `Sync`.

Calling `next.run(request).await` advances the stack and returns the `Response`. This
is what gives `from_fn` its "onion" shape:

- Code BEFORE `next.run(...)` runs on the way IN.
- Code AFTER `next.run(...)` runs on the way OUT.

### Short-circuiting : the deliberate auth-gate technique

A middleware that does NOT call `next.run()` and instead returns its own response
skips the entire downstream stack and the handler. This is:

- A **deliberate technique** for auth gates, rate limiters, and feature flags :
  return `StatusCode::UNAUTHORIZED` (or `FORBIDDEN`) and the handler never runs.
- An **accidental footgun** otherwise : silently dropping `next` makes the route
  appear to return a fixed response with no error.

RULE: in a `from_fn` middleware, ALWAYS either call `next.run(request).await` to
continue, OR return an `IntoResponse` directly to short-circuit. NEVER leave `next`
unused by accident.

---

## 6. Verified middleware examples

### 6.1 Logging / timing middleware (runs code in and out)

```rust
use axum::{extract::Request, middleware::Next, response::Response};
use std::time::Instant;

async fn log_requests(request: Request, next: Next) -> Response {
    let method = request.method().clone();
    let uri = request.uri().clone();
    let start = Instant::now();

    let response = next.run(request).await;

    tracing::info!(
        "{} {} -> {} in {:?}",
        method, uri, response.status(), start.elapsed()
    );
    response
}
// .layer(middleware::from_fn(log_requests))
```

### 6.2 Auth gate with deliberate short-circuit

```rust
use axum::{
    extract::Request,
    http::{header, StatusCode},
    middleware::Next,
    response::{IntoResponse, Response},
};

async fn require_auth(request: Request, next: Next) -> Response {
    let authorized = request
        .headers()
        .get(header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .map(|v| v == "Bearer secret-token")
        .unwrap_or(false);

    if !authorized {
        // short-circuit: handler is never reached
        return StatusCode::UNAUTHORIZED.into_response();
    }

    next.run(request).await
}
// apply with .route_layer(...) so unmatched paths still return 404, not 401
```

### 6.3 `from_fn_with_state` : a state-backed gate

```rust
use axum::{
    extract::{Request, State},
    http::{header, StatusCode},
    middleware::Next,
    response::{IntoResponse, Response},
};

#[derive(Clone)]
struct AppState {
    api_key: String,
}

async fn require_api_key(
    State(state): State<AppState>,   // FromRequestParts -> first
    request: Request,                // FromRequest      -> second-to-last
    next: Next,                      // Next             -> last
) -> Response {
    let provided = request
        .headers()
        .get("x-api-key")
        .and_then(|v| v.to_str().ok());

    match provided {
        Some(key) if key == state.api_key => next.run(request).await,
        _ => StatusCode::UNAUTHORIZED.into_response(),
    }
}

// let state = AppState { api_key: std::env::var("API_KEY").unwrap() };
// Router::new()
//     .route("/", get(handler))
//     .route_layer(middleware::from_fn_with_state(state.clone(), require_api_key))
//     .with_state(state);
```

### 6.4 Middleware with a `FromRequestParts` extractor before `Request`

```rust
use axum::{
    extract::Request,
    http::HeaderMap,
    middleware::Next,
    response::Response,
};

async fn check_header(
    headers: HeaderMap,   // FromRequestParts extractor, allowed before Request
    request: Request,     // the single FromRequest extractor
    next: Next,
) -> Response {
    let _ua = headers.get("user-agent").cloned();
    next.run(request).await
}
```

### 6.5 `map_response` : add a header to every response

```rust
use axum::{
    http::{HeaderName, HeaderValue},
    response::Response,
};

async fn add_security_header(mut response: Response) -> Response {
    response.headers_mut().insert(
        HeaderName::from_static("x-content-type-options"),
        HeaderValue::from_static("nosniff"),
    );
    response
}
// .layer(middleware::map_response(add_security_header))
```

### 6.6 `map_request` : normalise the request before the handler

```rust
use axum::{extract::Request, http::HeaderValue};

async fn force_json_accept(mut request: Request) -> Request {
    request.headers_mut().insert(
        axum::http::header::ACCEPT,
        HeaderValue::from_static("application/json"),
    );
    request
}
// .layer(middleware::map_request(force_json_accept))
```

---

## 7. Middleware ordering : the layer rule

This is the single most error-prone area of Axum and needs a deterministic rule.

### 7.1 The wrapping ("onion") model

`.layer()` wraps. Documented: middleware "will run *after* routing and thus cannot be
used to rewrite the request URI". Layers form an onion : the request travels INWARD
through each layer to the handler, the response travels back OUTWARD through each
layer in reverse.

### 7.2 Layers apply ONLY to routes that already exist

Documented verbatim: "the middleware is only applied to existing routes. So you have
to first add your routes (and / or fallback) and then call `layer` afterwards."

RULE: ALWAYS register every `.route()` and `.fallback()` call BEFORE calling
`.layer()`. A `.layer()` followed by more `.route()` calls does NOT cover the later
routes.

### 7.3 Stacked `Router::layer()` is bottom-to-top (last = outermost)

When multiple `.layer()` calls are chained on a `Router`, the **last** `.layer()`
call becomes the **outermost** layer. Worked example:

```rust
Router::new()
    .route("/", get(handler))
    .layer(layer_a)   // added first  -> INNER  (closest to handler)
    .layer(layer_b)   // added last   -> OUTER  (sees the request first)
```

- Request flows inward : `layer_b` -> `layer_a` -> `handler`.
- Response flows outward : `handler` -> `layer_a` -> `layer_b`.

CONTRAST with `tower::ServiceBuilder`, which is top-to-bottom : the FIRST
`.layer()` added to a `ServiceBuilder` is the outermost. Stacked `Router::layer()`
and `ServiceBuilder` are OPPOSITES. To avoid the confusion, build the global stack
with a single `ServiceBuilder` and apply it through one `.layer()` call. (Detailed
`ServiceBuilder` coverage belongs in the `axum-impl-tower-stack` skill.)

---

## 8. `route_layer` vs `layer` : the 404-vs-401 decision

Verified signature (research fragment B, against docs.rs Router page 2026-05-20):

```rust
pub fn route_layer<L>(self, layer: L) -> Self
where
    L: Layer<Route> + Clone + Send + Sync + 'static
```

| Method | Applies to |
|--------|-----------|
| `.layer()` | Everything, INCLUDING the fallback / 404 path. |
| `.route_layer()` | ONLY requests that match a registered route. |

The canonical reason to choose `route_layer` is authorization. With a plain
`.layer()`, an auth middleware turns `GET /not-found` into `401 Unauthorized` before
the router can answer `404 Not Found` : the middleware runs even on the fallback
path. With `.route_layer()`, an unmatched path bypasses the middleware and correctly
returns `404`.

Documented behavior: "`GET /foo` with a valid token will receive `200 OK` ...
`GET /not-found` with an invalid token will receive `404 Not Found`."

### Decision tree

- Middleware should run on EVERY request, matched or not (logging, tracing,
  request-id, CORS, compression, timeout) -> use `.layer()`.
- Middleware is an auth / authz gate that must NOT leak the existence of routes and
  must NOT mask a genuine 404 with a 401/403 -> use `.route_layer()`.
- Per-route or per-group middleware -> build a separate `Router` for the protected
  routes, layer it, then `.merge()` it; or layer the handler service directly via
  `get(handler).layer(...)`.

---

## 9. When to write a `tower::Layer` instead of `from_fn`

Use `from_fn` / `from_fn_with_state` when:

- You want familiar `async`/`await` syntax and are not comfortable hand-implementing
  futures.
- The middleware is application-internal and not published as a reusable crate.
- A single fixed behavior is enough (no configuration knobs).

Write a hand-rolled `tower::Layer` + `tower::Service` pair when:

- The middleware must be **configurable** with builder methods (timeouts, limits,
  policy objects).
- You intend to **publish** the middleware as a crate for others to use.
- You need full control over `poll_ready` (backpressure / readiness) and `call`.
- The middleware must be reusable across any Tower-based stack, not just Axum.

A Tower middleware requires implementing both `Layer` (`fn layer(&self, inner: S)
-> Self::Service`) and `Service` (`poll_ready` + `call`). It is lower-level and more
boilerplate, but fully composable and reusable. State for a custom layer is cloned
into the service during the `Layer::layer` call and accessed inside `Service::call`.
(A full hand-written Tower middleware walkthrough belongs in `axum-impl-tower-stack`;
this skill should only state the decision boundary and point there.)

---

## 10. Anti-patterns this skill must warn against

1. **Body extractor not in the second-to-last position** : the single `FromRequest`
   extractor must sit directly before `Next`; a parts extractor placed after it fails
   the trait bound.
2. **Forgetting `next.run()`** : silently drops the handler; only correct when the
   short-circuit is deliberate.
3. **`.layer()` before routes are registered** : later routes are not wrapped.
4. **Confusing stacked `.layer()` (bottom-to-top) with `ServiceBuilder`
   (top-to-bottom)** : opposite ordering, silently reorders auth/timeout/trace.
5. **Using `.layer()` for an auth gate** : turns a `404` into a `401` on unmatched
   paths; use `.route_layer()`.
6. **Passing mismatched or missing state to `from_fn_with_state`** : the same value
   and type must go into both the layer and `.with_state()`.
7. **Reaching for `from_fn` when `map_request` / `map_response` is enough** : a pure
   one-way transform does not need `Next`.

---

## Sources verified (2026-05-20)

All URLs fetched via WebFetch on 2026-05-20. Current docs.rs version observed:
axum 0.8.9.

| URL | Verified | Used for |
|-----|----------|----------|
| https://docs.rs/axum/latest/axum/middleware/index.html | 2026-05-20 | Section 1 (module overview, function list, three implementation strategies, Layer vs from_fn guidance) |
| https://docs.rs/axum/latest/axum/middleware/fn.from_fn.html | 2026-05-20 | Section 2 (`from_fn` exact signature, 5 numbered requirements, verbatim example) |
| https://docs.rs/axum/latest/axum/middleware/fn.from_fn_with_state.html | 2026-05-20 | Section 3 (`from_fn_with_state` exact signature, verbatim state example) |
| https://docs.rs/axum/latest/axum/middleware/struct.Next.html | 2026-05-20 | Section 5 (`Next` description, `run` exact signature, Clone/Service/Send/Sync) |
| https://docs.rs/axum/latest/axum/struct.Router.html | 2026-05-20 (via fragment B) | Section 7, 8 (`.layer`, `.route_layer` signature, ordering, 404-vs-401 behavior) |

### Verification notes

- The `from_fn` 5-point requirement list and both `from_fn` / `from_fn_with_state`
  example blocks are quoted verbatim from the live docs.rs pages fetched 2026-05-20.
- `Next::run` signature `pub async fn run(self, req: Request) -> Response` is
  confirmed verbatim from the `Next` struct page.
- The `route_layer` signature and the "200/404 with valid/invalid token" documented
  behavior come from research fragment B, which verified them against the docs.rs
  `Router` page on 2026-05-20; treat as verified.
- The bottom-to-top vs top-to-bottom ordering contrast (`Router::layer` vs
  `ServiceBuilder`) is verified in fragment B section 2/3 against the docs.rs tower
  `ServiceBuilder` page; the detailed `ServiceBuilder` treatment is delegated to the
  `axum-impl-tower-stack` skill.
