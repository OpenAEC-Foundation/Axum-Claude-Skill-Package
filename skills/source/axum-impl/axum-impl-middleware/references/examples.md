# Examples : axum-impl-middleware

Complete, working middleware. Every snippet is built from signatures
WebFetch-verified against docs.rs on 2026-05-20. The `axum::middleware` helper
API is identical across Axum 0.7 and 0.8, so no snippet here diverges between
versions.

## Example 1 : from_fn logging and timing middleware

Runs code before and after the handler (the onion model).

```rust
// axum 0.7 / 0.8
use axum::{
    extract::Request,
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};
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

async fn handler() -> &'static str {
    "ok"
}

fn app() -> Router {
    Router::new()
        .route("/", get(handler))
        .layer(middleware::from_fn(log_requests))
}
```

## Example 2 : auth gate with a deliberate short-circuit

The middleware returns `401` WITHOUT calling `next.run()`, so the handler never
runs. Applied with `.route_layer()` so an unmatched path still returns `404`.

```rust
// axum 0.7 / 0.8
use axum::{
    extract::Request,
    http::{header, StatusCode},
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::get,
    Router,
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

async fn protected() -> &'static str {
    "secret data"
}

fn app() -> Router {
    Router::new()
        .route("/protected", get(protected))
        // route_layer: GET /not-found returns 404, not 401
        .route_layer(middleware::from_fn(require_auth))
}
```

## Example 3 : from_fn_with_state, a state-backed API-key gate

State is passed once into `from_fn_with_state` and once into `.with_state()`.
`State` is a `FromRequestParts` extractor, so it comes first in the argument
list.

```rust
// axum 0.7 / 0.8
use axum::{
    extract::{Request, State},
    http::StatusCode,
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::get,
    Router,
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

async fn handler() -> &'static str {
    "ok"
}

fn app() -> Router {
    let state = AppState {
        api_key: std::env::var("API_KEY").expect("API_KEY must be set"),
    };

    Router::new()
        .route("/", get(handler))
        // SAME state value passed to the layer AND to with_state
        .route_layer(middleware::from_fn_with_state(state.clone(), require_api_key))
        .with_state(state)
}
```

## Example 4 : a FromRequestParts extractor before Request

Any number of `FromRequestParts` extractors may precede the single `Request`
extractor. Here `HeaderMap` reads the parts without consuming the body.

```rust
// axum 0.7 / 0.8
use axum::{
    extract::Request,
    http::HeaderMap,
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};

async fn check_header(
    headers: HeaderMap,   // FromRequestParts extractor, allowed before Request
    request: Request,     // the single FromRequest extractor
    next: Next,
) -> Response {
    if headers.get("user-agent").is_none() {
        tracing::warn!("request without a user-agent header");
    }
    next.run(request).await
}

fn app() -> Router {
    Router::new()
        .route("/", get(|| async { "ok" }))
        .layer(middleware::from_fn(check_header))
}
```

## Example 5 : map_response adds a header to every response

`map_response` is a pure one-way transform: it sees only the response and has
no `Next`.

```rust
// axum 0.7 / 0.8
use axum::{
    http::{HeaderName, HeaderValue},
    middleware,
    response::Response,
    routing::get,
    Router,
};

async fn add_security_header(mut response: Response) -> Response {
    response.headers_mut().insert(
        HeaderName::from_static("x-content-type-options"),
        HeaderValue::from_static("nosniff"),
    );
    response
}

fn app() -> Router {
    Router::new()
        .route("/", get(|| async { "ok" }))
        .layer(middleware::map_response(add_security_header))
}
```

## Example 6 : map_request normalises the request before the handler

```rust
// axum 0.7 / 0.8
use axum::{
    extract::Request,
    http::{header::ACCEPT, HeaderValue},
    middleware,
    routing::get,
    Router,
};

async fn force_json_accept(mut request: Request) -> Request {
    request
        .headers_mut()
        .insert(ACCEPT, HeaderValue::from_static("application/json"));
    request
}

fn app() -> Router {
    Router::new()
        .route("/", get(|| async { "ok" }))
        .layer(middleware::map_request(force_json_accept))
}
```

## Example 7 : stacked layer ordering, flow traced

The LAST `.layer()` added is the OUTERMOST. Trace the flow before relying on
the order.

```rust
// axum 0.7 / 0.8
use axum::{
    extract::Request,
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};

async fn outer(request: Request, next: Next) -> Response {
    tracing::info!("outer: in");
    let response = next.run(request).await;
    tracing::info!("outer: out");
    response
}

async fn inner(request: Request, next: Next) -> Response {
    tracing::info!("inner: in");
    let response = next.run(request).await;
    tracing::info!("inner: out");
    response
}

fn app() -> Router {
    Router::new()
        .route("/", get(|| async { "ok" }))
        .layer(middleware::from_fn(inner))   // added first -> INNER
        .layer(middleware::from_fn(outer))   // added last  -> OUTER
}

// Log order for one request:
//   outer: in
//   inner: in
//   <handler runs>
//   inner: out
//   outer: out
```

## Example 8 : per-group middleware via a separate Router

To protect one group of routes only, build a separate `Router`, layer it, and
`.merge()` it into the main router. The public routes never touch the gate.

```rust
// axum 0.7 / 0.8
use axum::{middleware, routing::get, Router};

fn app() -> Router {
    let public = Router::new()
        .route("/", get(|| async { "public" }));

    let protected = Router::new()
        .route("/admin", get(|| async { "admin" }))
        .route_layer(middleware::from_fn(require_auth));

    public.merge(protected)
}
# async fn require_auth(
#     request: axum::extract::Request,
#     next: axum::middleware::Next,
# ) -> axum::response::Response {
#     next.run(request).await
# }
```
