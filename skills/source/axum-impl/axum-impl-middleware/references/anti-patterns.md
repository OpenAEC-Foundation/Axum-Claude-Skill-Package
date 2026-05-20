# Anti-Patterns : axum-impl-middleware

Real middleware mistakes, each with the root cause and the fix. Every
signature referenced was WebFetch-verified against docs.rs on 2026-05-20. None
of these patterns diverges between Axum 0.7 and 0.8.

## AP-1 : body extractor not in the second-to-last position

```rust
// axum 0.7 / 0.8 - WRONG: Request is not directly before Next
async fn bad(
    request: Request,   // FromRequest extractor
    headers: HeaderMap, // FromRequestParts extractor placed AFTER the body
    next: Next,
) -> Response {
    next.run(request).await
}
```

WHY THIS FAILS: `from_fn` requires the single `FromRequest` extractor to be
the second-to-last argument, directly before `Next`. Every argument before it
must be `FromRequestParts`. With `request` first and `headers` after it, the
function no longer matches the shape `from_fn` accepts. The layer fails to
build and the error surfaces as the opaque `the trait bound ... is not
satisfied`.

FIX: order the arguments as
`[FromRequestParts extractors] , Request , Next`.

```rust
// axum 0.7 / 0.8 - CORRECT
async fn good(
    headers: HeaderMap, // FromRequestParts -> first
    request: Request,   // FromRequest      -> second-to-last
    next: Next,         // Next             -> last
) -> Response {
    let _ = headers;
    next.run(request).await
}
```

## AP-2 : forgetting next.run, silently dropping the handler

```rust
// axum 0.7 / 0.8 - WRONG: next is never called
async fn bad(request: Request, next: Next) -> Response {
    let _ = request;
    // the route now always returns 200 OK with an empty body;
    // the handler never runs and there is no error
    StatusCode::OK.into_response()
}
```

WHY THIS FAILS: a `from_fn` middleware that returns without calling
`next.run()` short-circuits the entire downstream stack and the handler. When
this is unintentional, the route silently returns a fixed response. There is
no compile error and no runtime error; the handler is simply skipped.

FIX: in a `from_fn` middleware, ALWAYS call `next.run(request).await` to
continue. Returning early without `next.run()` is correct ONLY when the
short-circuit is deliberate (an auth gate, a rate limiter).

```rust
// axum 0.7 / 0.8 - CORRECT: the handler runs
async fn good(request: Request, next: Next) -> Response {
    next.run(request).await
}
```

## AP-3 : calling layer before the routes are registered

```rust
// axum 0.7 / 0.8 - WRONG: /late is added AFTER the layer
let app = Router::new()
    .route("/early", get(handler))
    .layer(middleware::from_fn(log_requests))
    .route("/late", get(handler));   // NOT wrapped by log_requests
```

WHY THIS FAILS: a layer wraps only the routes that already exist when
`.layer()` is called. Documented verbatim: "the middleware is only applied to
existing routes. So you have to first add your routes (and / or fallback) and
then call `layer` afterwards." `/late` is registered after the layer, so it
runs without the middleware. No error is raised.

FIX: register every `.route()` and `.fallback()` BEFORE the `.layer()` call.

```rust
// axum 0.7 / 0.8 - CORRECT: all routes first, then the layer
let app = Router::new()
    .route("/early", get(handler))
    .route("/late", get(handler))
    .layer(middleware::from_fn(log_requests));
```

## AP-4 : using layer for an auth gate, masking a real 404

```rust
// axum 0.7 / 0.8 - WRONG: an auth gate applied with .layer()
let app = Router::new()
    .route("/protected", get(handler))
    .layer(middleware::from_fn(require_auth));   // wraps the fallback too
```

WHY THIS FAILS: `.layer()` wraps every request, INCLUDING the fallback / 404
path. An unauthenticated `GET /does-not-exist` hits `require_auth` before the
router can answer `404 Not Found`, so the client receives `401 Unauthorized`.
The response leaks that authentication is the gate and hides whether the route
exists at all.

FIX: apply auth and authz gates with `.route_layer()`, which wraps only
matched routes. An unmatched path then bypasses the gate and returns `404`.

```rust
// axum 0.7 / 0.8 - CORRECT
let app = Router::new()
    .route("/protected", get(handler))
    .route_layer(middleware::from_fn(require_auth));
```

## AP-5 : confusing stacked layer order with ServiceBuilder

```rust
// axum 0.7 / 0.8 - WRONG ASSUMPTION about order
let app = Router::new()
    .route("/", get(handler))
    .layer(middleware::from_fn(auth))     // assumed to run first
    .layer(middleware::from_fn(logging)); // assumed to run second
// actual order: logging runs FIRST (outermost), auth runs SECOND
```

WHY THIS FAILS: stacked `Router::layer()` is bottom-to-top: the LAST `.layer()`
added is the OUTERMOST and sees the request first. This is the OPPOSITE of
`tower::ServiceBuilder`, which is top-to-bottom (first added = outermost).
Assuming `ServiceBuilder` order silently reverses the stack: auth may run
after logging, or a timeout may wrap the wrong layers.

FIX: build the global stack with a single `tower::ServiceBuilder` and apply it
in one `.layer()` call. Inside a `ServiceBuilder`, the first `.layer()` is the
outermost, which reads top-to-bottom.

```rust
// axum 0.7 / 0.8 - CORRECT: one ServiceBuilder, top-to-bottom order
use tower::ServiceBuilder;

let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(middleware::from_fn(logging)) // outermost: runs first
            .layer(middleware::from_fn(auth)),   // inner: runs second
    );
```

See `axum-impl-tower-stack` for the full `ServiceBuilder` treatment.

## AP-6 : mismatched or missing state for from_fn_with_state

```rust
// axum 0.7 / 0.8 - WRONG: the layer state is set, but with_state is missing
let state = AppState { api_key: "k".into() };
let app = Router::new()
    .route("/", get(handler))
    .route_layer(middleware::from_fn_with_state(state.clone(), require_api_key));
// missing: .with_state(state)
```

WHY THIS FAILS: `from_fn_with_state` makes the middleware function expect a
`State<S>` extractor. The router is still typed `Router<AppState>` because the
state was never provided with `.with_state()`. A `Router<AppState>` cannot be
served; `axum::serve` requires `Router<()>`. The mismatch is a compile-time
error, not a runtime one.

FIX: pass the SAME state value, of the SAME type, into both
`from_fn_with_state` and `.with_state()`.

```rust
// axum 0.7 / 0.8 - CORRECT
let state = AppState { api_key: "k".into() };
let app = Router::new()
    .route("/", get(handler))
    .route_layer(middleware::from_fn_with_state(state.clone(), require_api_key))
    .with_state(state);
```

## AP-7 : reaching for from_fn when map_request or map_response is enough

```rust
// axum 0.7 / 0.8 - WORKS, but heavier than needed
async fn add_header(request: Request, next: Next) -> Response {
    let mut response = next.run(request).await;
    response.headers_mut().insert(
        HeaderName::from_static("x-app"),
        HeaderValue::from_static("axum"),
    );
    response
}
```

WHY THIS IS WRONG: this middleware performs a pure one-way transform of the
response. It uses `Next` only to obtain the response, never to inspect the
request side or to short-circuit. `from_fn` carries the full onion machinery
that this logic does not need, and it makes the intent (a response-only
transform) less clear to a reader.

FIX: use `map_response` for a response-only transform and `map_request` for a
request-only transform. Reserve `from_fn` for logic that must run before AND
after the handler, or that conditionally short-circuits.

```rust
// axum 0.7 / 0.8 - CORRECT
async fn add_header(mut response: Response) -> Response {
    response.headers_mut().insert(
        HeaderName::from_static("x-app"),
        HeaderValue::from_static("axum"),
    );
    response
}
// .layer(middleware::map_response(add_header))
```

## AP-8 : two FromRequest extractors in one middleware

```rust
// axum 0.7 / 0.8 - WRONG: two body-consuming extractors
async fn bad(
    body: String,        // FromRequest: consumes the body
    request: Request,    // FromRequest: also consumes the body
    next: Next,
) -> Response {
    next.run(request).await
}
```

WHY THIS FAILS: `from_fn` allows exactly one `FromRequest` extractor. The
request body is a single-use async stream and can be consumed only once. Two
`FromRequest` extractors in one function fail the trait bound, and the layer
does not build.

FIX: take the single `Request` extractor and read whatever is needed from it
inside the function. To read the body bytes, consume the `Request` body
explicitly rather than adding a second extractor.

```rust
// axum 0.7 / 0.8 - CORRECT: one FromRequest extractor
async fn good(request: Request, next: Next) -> Response {
    // inspect request.headers(), request.method(), request.uri() here
    next.run(request).await
}
```
