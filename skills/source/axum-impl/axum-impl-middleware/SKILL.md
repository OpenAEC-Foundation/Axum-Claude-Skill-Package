---
name: axum-impl-middleware
description: >
  Use when adding middleware to an Axum service: a logging or timing layer, an
  auth gate, a header rewriter, or any cross-cutting code that must run before
  or after a handler with axum::middleware::from_fn, from_fn_with_state,
  map_request, or map_response.
  Prevents the wrong middleware argument order that triggers a "Handler is not
  satisfied" trait-bound error, the auth gate that turns a 404 into a 401
  because it used layer instead of route_layer, the silently dropped handler
  from a from_fn that never calls next.run, and the reversed execution order
  from confusing stacked Router::layer with ServiceBuilder.
  Covers the axum::middleware helper family, the parts-then-Request-then-Next
  argument rule, the Next type and deliberate short-circuiting, layer ordering,
  the route_layer versus layer decision, and when to drop to a tower::Layer.
  Keywords: axum middleware, from_fn, from_fn_with_state, map_request,
  map_response, Next, next.run, route_layer, layer, tower Layer, middleware
  order, onion model, Handler is not satisfied, 404 became 401, middleware not
  running, handler never runs, missing with_state, auth gate, how do I add
  middleware, run code before the handler, protect a route.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-impl-middleware

## Overview

Axum has no middleware system of its own. It integrates the Tower ecosystem
(`tower::Service` plus `tower::Layer`). The `axum::middleware` module supplies
ergonomic helpers so middleware can be a plain `async fn` instead of a
hand-written `Service`.

Three implementation strategies, narrowest first:

- `map_request` / `map_response`: a pure one-way transform (add a header,
  normalise a field). No `Next`, cannot short-circuit, cannot see both sides.
- `from_fn` / `from_fn_with_state`: an `async fn` that receives `Next` and runs
  code before AND after the handler, and may short-circuit it.
- A hand-written `tower::Layer` plus `tower::Service`: required only when the
  middleware must be configurable or published as a crate.

The `axum::middleware` helper API (`from_fn`, `from_fn_with_state`, `Next`,
`map_request`, `map_response`) is identical across Axum 0.7 and 0.8. `Next` and
`Request` carry no generic body parameter in either version. No code in this
skill diverges between versions.

## Quick Reference

| Helper | Use for | Has `Next` |
|--------|---------|-----------|
| `from_fn(f)` | code before and after the handler, or short-circuit | YES |
| `from_fn_with_state(state, f)` | same, with `State<S>` access | YES |
| `map_request(f)` | transform the request only | NO |
| `map_response(f)` | transform the response only | NO |
| `map_request_with_state` / `map_response_with_state` | `map_*` with state | NO |

The `from_fn` argument order rule, exact and enforced by the compiler:

```
[zero or more FromRequestParts extractors] , [one FromRequest extractor] , Next
```

`from_fn` requirements (verbatim from docs.rs): the function must "Be an
`async fn`", "Take zero or more `FromRequestParts` extractors", "Take exactly
one `FromRequest` extractor as the second to last argument", "Take `Next` as
the last argument", and "Return something that implements `IntoResponse`".

Core rules:

- ALWAYS make `Next` the final argument of a `from_fn` middleware.
- ALWAYS make `Request` (the single `FromRequest` extractor) the second-to-last
  argument, directly before `Next`.
- ALWAYS place `FromRequestParts` extractors (`State`, `HeaderMap`, `Method`,
  `TypedHeader`) before `Request`.
- NEVER use two `FromRequest` extractors in one middleware. The body is a
  single-use stream.
- ALWAYS register every `.route()` and `.fallback()` BEFORE calling `.layer()`.
  A layer wraps only routes that already exist.
- ALWAYS use `.route_layer()` for an auth or authz gate, NEVER `.layer()`.
- ALWAYS pass the same state value to both `from_fn_with_state(state.clone(),
  ...)` and `.with_state(state)`.
- NEVER leave `next` unused unless the short-circuit is deliberate.

## Decision Trees

### Which middleware helper to use

```
What does the middleware need to do?
  transform ONLY the request, no response access  -> map_request
  transform ONLY the response, no request access  -> map_response
  run code BEFORE and AFTER the handler            -> from_fn
  conditionally SKIP the handler (auth, rate limit)-> from_fn
  any of the above, needs application state        -> *_with_state variant
  must be CONFIGURABLE or PUBLISHED as a crate      -> hand-written tower::Layer
```

`map_*` middleware has no `Next`, so it can neither inspect both sides of the
call nor skip the handler. Reach for `from_fn` only when `Next` is genuinely
needed. For a configurable or publishable layer, see `axum-impl-tower-stack`.

### layer versus route_layer : the 404-vs-401 decision

```
Should the middleware run on requests that match NO route?
  YES, every request matched or not (logging, tracing, CORS, compression,
       timeout, request-id)                       -> .layer()
  NO, it is an auth or authz gate and must not
      mask a real 404 with a 401 or 403            -> .route_layer()
  it is per-route or per-group only                -> separate Router +
                                                      .merge(), or layer the
                                                      handler service directly
```

`.layer()` wraps everything, INCLUDING the fallback path. An auth middleware
applied with `.layer()` turns `GET /not-found` into `401 Unauthorized` before
the router can answer `404 Not Found`. `.route_layer()` wraps only matched
routes, so an unmatched path bypasses the gate and returns `404` correctly.

### Stacked Router::layer ordering

```
Multiple .layer() calls chained on one Router?
  the LAST .layer() added is the OUTERMOST (sees the request first)
  the FIRST .layer() added is the INNERMOST (closest to the handler)
```

Stacked `Router::layer()` is bottom-to-top. This is the OPPOSITE of
`tower::ServiceBuilder`, which is top-to-bottom (first added = outermost). To
avoid the confusion, build one global stack with a single `ServiceBuilder` and
apply it through one `.layer()` call. See `axum-impl-tower-stack`.

## Patterns

### Pattern: from_fn middleware that runs in and out

A `from_fn` middleware forms an onion. Code before `next.run(...)` runs on the
way in; code after runs on the way out.

```rust
// axum 0.7 / 0.8 - identical. Logging and timing middleware.
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
```

Apply it (a layer wraps only routes registered before it):

```rust
// axum 0.7 / 0.8
use axum::{middleware, routing::get, Router};

let app = Router::new()
    .route("/", get(handler))
    .layer(middleware::from_fn(log_requests));
```

### Pattern: deliberate short-circuit for an auth gate

A `from_fn` middleware that returns its own response WITHOUT calling
`next.run()` skips the entire downstream stack and the handler. This is the
correct technique for auth gates, rate limiters, and feature flags.

```rust
// axum 0.7 / 0.8 - identical. Auth gate: handler never runs when unauthorized.
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
        // short-circuit: the handler is never reached
        return StatusCode::UNAUTHORIZED.into_response();
    }

    next.run(request).await
}
```

ALWAYS apply an auth gate with `.route_layer()` so an unmatched path still
returns `404`, not `401`:

```rust
// axum 0.7 / 0.8
let app = Router::new()
    .route("/protected", get(handler))
    .route_layer(middleware::from_fn(require_auth));
```

In a `from_fn` middleware, ALWAYS either call `next.run(request).await` to
continue OR return an `IntoResponse` to short-circuit. NEVER leave `next`
unused by accident: that silently drops the handler and the route appears to
return a fixed response.

### Pattern: from_fn_with_state

`from_fn_with_state` threads application state into the middleware. State is
supplied to the layer constructor and surfaces inside the function as a
`State<S>` extractor. Because `State` is a `FromRequestParts` extractor it
comes first, before `Request` and `Next`.

```rust
// axum 0.7 / 0.8 - identical. State-backed API-key gate.
use axum::{
    extract::{Request, State},
    http::StatusCode,
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
```

ALWAYS pass the same state value twice: once into `from_fn_with_state` and once
into `.with_state()`. Forgetting `.with_state()` or passing a different type is
a compile-time error, never a runtime one.

```rust
// axum 0.7 / 0.8
let state = AppState { api_key: std::env::var("API_KEY").unwrap() };

let app = Router::new()
    .route("/", get(handler))
    .route_layer(middleware::from_fn_with_state(state.clone(), require_api_key))
    .with_state(state);
```

### Pattern: a FromRequestParts extractor before Request

Any number of `FromRequestParts` extractors may precede the single `Request`
extractor. They run on the request parts only and never consume the body.

```rust
// axum 0.7 / 0.8 - identical.
use axum::{extract::Request, http::HeaderMap, middleware::Next, response::Response};

async fn check_header(
    headers: HeaderMap,   // FromRequestParts extractor, allowed before Request
    request: Request,     // the single FromRequest extractor
    next: Next,
) -> Response {
    let _ua = headers.get("user-agent").cloned();
    next.run(request).await
}
```

### Pattern: map_response and map_request

Use `map_response` and `map_request` for a pure one-way transform. They have no
`Next` and cannot short-circuit.

```rust
// axum 0.7 / 0.8 - identical. Add a security header to every response.
use axum::http::{HeaderName, HeaderValue};
use axum::response::Response;

async fn add_security_header(mut response: Response) -> Response {
    response.headers_mut().insert(
        HeaderName::from_static("x-content-type-options"),
        HeaderValue::from_static("nosniff"),
    );
    response
}
// .layer(middleware::map_response(add_security_header))
```

```rust
// axum 0.7 / 0.8 - identical. Normalise the request before the handler.
use axum::{extract::Request, http::{header::ACCEPT, HeaderValue}};

async fn force_json_accept(mut request: Request) -> Request {
    request.headers_mut().insert(ACCEPT, HeaderValue::from_static("application/json"));
    request
}
// .layer(middleware::map_request(force_json_accept))
```

### Pattern: stacked layer ordering

```rust
// axum 0.7 / 0.8 - the LAST .layer() is the OUTERMOST.
let app = Router::new()
    .route("/", get(handler))
    .layer(middleware::from_fn(inner_layer))   // added first -> INNER
    .layer(middleware::from_fn(outer_layer));  // added last  -> OUTER
// request flow:  outer_layer -> inner_layer -> handler
// response flow: handler -> inner_layer -> outer_layer
```

For more than two layers, ALWAYS build one `ServiceBuilder` and apply it in a
single `.layer()` call instead of chaining `.layer()` repeatedly. The two
ordering models are opposites and stacking them invites silent reordering of
auth, timeout, and tracing. See `axum-impl-tower-stack`.

### Pattern: when to write a tower::Layer instead

Use `from_fn` / `from_fn_with_state` when the middleware is
application-internal, needs no configuration, and a single fixed behavior is
enough. Write a hand-rolled `tower::Layer` plus `tower::Service` pair when the
middleware must be configurable with builder methods, must be published as a
reusable crate, or needs control over `poll_ready` backpressure. The full
hand-written Tower walkthrough lives in `axum-impl-tower-stack`; this skill
states only the boundary.

## Reference Links

- `references/methods.md`: exact signatures for `from_fn`, `from_fn_with_state`,
  `map_request`, `map_response`, the `Next` type and `Next::run`, and
  `Router::layer` versus `Router::route_layer`.
- `references/examples.md`: complete working middleware, the argument-order
  rule applied, the auth-gate short-circuit, state threading, and layer
  ordering with the request and response flow traced.
- `references/anti-patterns.md`: real mistakes with root-cause analysis,
  including wrong argument order, the dropped `next.run()`, `.layer()` before
  routes, the `.layer()` auth gate that masks a 404, and the stacked-layer
  versus `ServiceBuilder` ordering reversal.

Related skills: `axum-impl-tower-stack` (`ServiceBuilder`, tower-http layers,
hand-written `tower::Layer`), `axum-impl-authorization` (role and permission
gates built on `route_layer`), `axum-core-router` (`.layer` versus
`.route_layer` placement, the 405-vs-404 model), `axum-syntax-extractors`
(the `FromRequestParts` versus `FromRequest` classification).
