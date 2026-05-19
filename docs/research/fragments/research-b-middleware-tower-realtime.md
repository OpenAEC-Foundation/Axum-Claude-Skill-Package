# Research Fragment B : Middleware, Tower, Realtime, Static

Research-agent B deep research for the Axum skill package, Phase 2. All API claims,
signatures and code examples below were verified via WebFetch against the official
sources listed in the final section. Target versions: Axum 0.7 and 0.8 (current
docs.rs version at research time: **axum 0.8.9**). Version-sensitive items are
annotated inline (`// axum 0.8`).

---

## 1. Middleware in Axum

Axum does not implement its own middleware system. It integrates with the **Tower**
middleware ecosystem (`tower::Service` + `tower::Layer`). The `axum::middleware`
module provides ergonomic helpers so you can write middleware as plain `async fn`
instead of hand-rolling a `Service`.

### `from_fn`

```rust
pub fn from_fn<F, T>(f: F) -> FromFnLayer<F, (), T>
```

`from_fn` builds a `Layer` from an async function. The documented requirements for
the function are exact and strict:

1. It must be an `async fn`.
2. It may take zero or more `FromRequestParts` extractors.
3. It must take exactly one `FromRequest` extractor as the **second-to-last** argument.
4. It must take `Next` as the **last** argument.
5. It must return something that implements `IntoResponse`.

Verified verbatim example:

```rust
use axum::{
    Router,
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

ALWAYS place `Request` (the single `FromRequest` extractor) directly before `Next`.
ALWAYS put any `FromRequestParts` extractors (headers, `State`, `Path`, etc.) before
`Request`. NEVER put two `FromRequest` extractors in one middleware function: only one
extractor may consume the body.

### `from_fn_with_state`

```rust
pub fn from_fn_with_state<F, S, T>(state: S, f: F) -> FromFnLayer<F, S, T>
```

Use this when the middleware needs access to application state. State is supplied to
the layer constructor and surfaces inside the function as a `State<S>` extractor.
Because `State` is a `FromRequestParts` extractor it comes first, before `Request`
and `Next`. Verified verbatim:

```rust
async fn my_middleware(
    State(state): State<AppState>,
    request: Request,
    next: Next,
) -> Response { /* ... */ }

let app = Router::new()
    .route("/", get(|| async { /* ... */ }))
    .route_layer(middleware::from_fn_with_state(state.clone(), my_middleware))
    .with_state(state);
```

Note the official example clones the state: the same value goes into the layer and
into `.with_state()`.

### `map_request` / `map_response`

These are for small ad-hoc transformations (for example adding a header) where you
do not need the wrapping `Next` semantics. `map_request` transforms the request
before it reaches the handler; `map_response` transforms the response after. Each has
a `_with_state` variant (`map_request_with_state`, `map_response_with_state`) that
threads state in the same way as `from_fn_with_state`.

### The `Next` type

`Next` represents "the remainder of a middleware stack, including the handler". You
advance the stack by calling `next.run(request).await`, which returns the
`Response`. This is what gives `from_fn` its onion shape: code before `next.run(...)`
runs on the way in, code after it runs on the way out. NEVER forget to actually call
`next.run()` — a middleware that drops `next` and returns its own response will
short-circuit the entire downstream stack and handler (this is a deliberate technique
for auth gates, but an accidental footgun otherwise).

### tower `Layer` vs `from_fn` (when to use which)

Use `from_fn` when you are not comfortable hand-implementing futures and want
familiar async/await syntax, and when you do not intend to publish the middleware as
a reusable crate. Write a real `tower::Layer` + `tower::Service` pair when you need
maximum control, configurable builder methods, or you want to publish the middleware
for others. A Tower middleware requires implementing both traits with explicit
`poll_ready` and `call` methods: lower-level, more boilerplate, but fully composable
and reusable across any Tower-based stack (not just Axum).

---

## 2. Middleware Ordering

This is the single most error-prone area of Axum and deserves a deterministic rule.

### The wrapping ("onion") model

`.layer()` wraps. The documentation states middleware "will run *after* routing and
thus cannot be used to rewrite the request URI". Layers form an onion: the request
travels inward through each layer to the handler, the response travels back outward
through each layer in reverse.

### Critical rule: layers apply only to routes that already exist

The docs are explicit: "the middleware is only applied to existing routes. So you
have to first add your routes (and / or fallback) and then call `layer` afterwards."

ALWAYS register all `.route()` / `.fallback()` calls BEFORE calling `.layer()`. A
`.layer()` followed by more `.route()` calls will NOT cover the later routes.

### Bottom-to-top stacking on a `Router`

When you chain multiple `.layer()` calls on a `Router`, the **last** `.layer()` call
becomes the **outermost** layer. Equivalently: layers stack bottom-to-top, the closest
`.layer()` to the handler runs last on the way in. Example:

```rust
// axum 0.7 / 0.8
Router::new()
    .route("/", get(handler))
    .layer(layer_a)   // inner   (added first)
    .layer(layer_b)   // outer   (added last -> sees request first)
```

Request flow inward: `layer_b` -> `layer_a` -> `handler`. Response flow outward:
`handler` -> `layer_a` -> `layer_b`.

### `.route_layer()` vs `.layer()`

```rust
pub fn route_layer<L>(self, layer: L) -> Self
where L: Layer<Route> + Clone + Send + Sync + 'static
```

`.route_layer()` applies middleware **only if the request matches a route**.
`.layer()` applies to everything including the fallback / 404 path. The canonical
reason to use `route_layer` is auth: with a plain `.layer()` an auth middleware would
turn a `GET /not-found` into a `401 Unauthorized` before the router can answer
`404 Not Found`. With `route_layer`, an unmatched path bypasses the middleware and
correctly returns `404`. Documented behavior: "GET /foo with a valid token will
receive 200 OK ... GET /not-found with an invalid token will receive 404 Not Found".

### Layering a `Router` vs a single route

You can apply middleware to one route by layering a small sub-router or by using
`route_layer`. Layering the whole `Router` covers all of its routes. For per-route
middleware, the common patterns are: (a) build a separate `Router` for the protected
routes, layer it, then `.merge()` it; or (b) layer the handler service directly via
`get(handler).layer(...)`.

---

## 3. Tower Fundamentals

### The `Service` trait

`Service` is the core Tower abstraction: conceptually
`async fn(Request) -> Result<Response, Error>`. Associated types:

- `Response` — the success type.
- `Error` — the error type.
- `Future` — the future returned by `call`.

Methods:

```rust
fn poll_ready(&mut self, cx: &mut Context<'_>)
    -> Poll<Result<(), Self::Error>>;

fn call(&mut self, req: Request) -> Self::Future;
```

`poll_ready` implements **backpressure / readiness**: a service signals when it is
ready to accept a request. The contract is that `poll_ready` must return
`Poll::Ready(Ok(()))` before `call` is invoked. This enables flow control (rate
limiting, concurrency limits, buffering) to be expressed uniformly. `call` consumes a
request and returns a future resolving to `Result<Response, Error>`.

### The `Layer` trait

`Layer` decorates a `Service`, transforming requests and/or responses:

```rust
fn layer(&self, inner: S) -> Self::Service;
```

It takes an inner service and returns a wrapped service with extra behavior. A
`Layer` is the reusable, composable unit; calling it produces the actual `Service`.

### `ServiceBuilder` and its ordering (verified carefully)

`ServiceBuilder` gives a declarative API for stacking layers onto a service. The
ordering semantics are the part everyone gets wrong, so this was verified verbatim.

Official wording: **"The order in which layers are added impacts how requests are
handled. Layers that are added first will be called with the request first. The
argument to `service` will be last to see the request."**

So in `ServiceBuilder`, the **first** `.layer()` added is the **outermost** layer.

```rust
// tower
ServiceBuilder::new()
    .buffer(100)              // added first  -> OUTERMOST, sees request first
    .concurrency_limit(10)    // added second -> inner
    .service(svc);            // the leaf, sees request last
```

Documented: "the buffer layer receives the request first followed by
`concurrency_limit`."

This is the **opposite** of manually nesting `Layer::layer()` calls and the opposite
of stacked `Router::layer()` calls. With manual nesting / stacked `.layer()`, the
LAST one written is the outermost. With `ServiceBuilder`, the FIRST one written is the
outermost. The deterministic rule:

- `ServiceBuilder::new().layer(A).layer(B)` -> A is outer, B is inner (top-to-bottom).
- `router.layer(A).layer(B)` -> B is outer, A is inner (bottom-to-top).

`ServiceBuilder`'s top-to-bottom reading order is generally considered easier to
reason about, which is why Axum users often build their global stack with a single
`ServiceBuilder` and pass it to one `.layer()` call.

---

## 4. Common tower-http Layers

`tower-http` is feature-gated; each layer needs its Cargo feature enabled.

### `TraceLayer` (feature `trace`)

```rust
use tower_http::trace::TraceLayer;
router.layer(TraceLayer::new_for_http())
```

`TraceLayer::new_for_http()` builds an HTTP-tuned tracing layer that emits spans and
events for each request/response. Pairs with the `tracing` + `tracing-subscriber`
crates.

### `CorsLayer` (feature `cors`)

```rust
use tower_http::cors::{CorsLayer, Any};
use axum::http::{Method, HeaderValue};

let cors = CorsLayer::new()
    .allow_origin("https://example.com".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET, Method::POST])
    .allow_headers(Any)
    .max_age(std::time::Duration::from_secs(3600));
```

Builder methods: `.allow_origin()`, `.allow_methods()`, `.allow_headers()`,
`.allow_credentials()`, `.expose_headers()`, `.max_age()`. Without `.max_age()`, the
browser issues a preflight `OPTIONS` for every request.

`CorsLayer::permissive()` allows: all origins, all methods, all request headers, and
exposes all headers. `CorsLayer::very_permissive()` additionally **allows
credentials**, and instead of a literal `*` it echoes back the request's origin,
`Access-Control-Request-Method`, and `Access-Control-Request-Headers` (because the
CORS spec forbids the `*` wildcard when credentials are allowed). The
credentials-vs-wildcard rule is a footgun: `allow_credentials(true)` combined with a
wildcard `Any` origin is invalid CORS and browsers reject it; `very_permissive()`
exists precisely to sidestep that by echoing the concrete origin.

### `CompressionLayer` (features `compression-gzip` / `-br` / `-deflate` / `-zstd`)

```rust
use tower_http::compression::CompressionLayer;
router.layer(CompressionLayer::new())
```

Compresses response bodies via content negotiation against the `Accept-Encoding`
request header.

### `TimeoutLayer` (feature `timeout`)

```rust
use tower_http::timeout::TimeoutLayer;
router.layer(TimeoutLayer::new(std::time::Duration::from_secs(30)))
```

Aborts requests that exceed the configured duration.

### `RequestBodyLimitLayer` (feature `limit`)

```rust
pub fn new(limit: usize) -> Self
```

```rust
use tower_http::limit::RequestBodyLimitLayer;
router.layer(RequestBodyLimitLayer::new(10 * 1024 * 1024)) // 10 MB
```

`limit` is the maximum body length in bytes. Bodies larger than the limit are
converted into `413 Payload Too Large` responses. This is the explicit limiter you
should set after disabling Axum's `DefaultBodyLimit` (see section 8).

### Rate limiting

There is no dedicated `tower-http` rate-limit layer. Options:

- `tower::limit::RateLimitLayer` / `ConcurrencyLimitLayer` from the core `tower`
  crate (`limit` feature) — coarse global limiting; `RateLimitLayer` requires a
  `Buffer` in front to be `Clone`.
- The `tower_governor` crate (`GovernorLayer`, built on the `governor` crate) — the
  community standard for per-IP token-bucket rate limiting with Axum. Recommend this
  for production rate limiting; document it as an ecosystem crate, not core.

---

## 5. State Management

### `Router::with_state` and the `State<S>` extractor

```rust
pub fn with_state<S2>(self, state: S) -> Router<S2>
```

In `Router<S>`, the type parameter `S` denotes a state the router is still **missing**.
Only `Router<()>` can be served. `with_state()` provides global state used for all
requests. The extractor is a newtype: `pub struct State<S>(pub S);`, destructured as:

```rust
async fn handler(State(state): State<AppState>) { /* ... */ }

let app = Router::new()
    .route("/", get(handler))
    .with_state(AppState { /* ... */ });
```

### Shared mutable state with `Arc`

State is `Clone`d per request, so wrap heavy or shared-mutable data in `Arc`:

```rust
#[derive(Clone)]
struct AppState {
    data: Arc<Mutex<String>>,
}

let state = AppState { data: Arc::new(Mutex::new("foo".to_owned())) };
```

CRITICAL: NEVER hold a `std::sync::Mutex` guard across an `.await` point — it makes
the future non-`Send` and Axum will reject the handler. Use `tokio::sync::Mutex` when
the lock must be held across `.await`, or scope the `std::sync::Mutex` guard so it is
dropped before any `.await`.

### `FromRef` and substates

`FromRef` lets a handler extract a smaller "substate" out of a larger app state, so
each handler depends only on what it needs. Verified verbatim:

```rust
#[derive(Clone)]
struct AppState {
    api_state: ApiState,
}

#[derive(Clone)]
struct ApiState {}

impl FromRef<AppState> for ApiState {
    fn from_ref(app_state: &AppState) -> ApiState {
        app_state.api_state.clone()
    }
}

async fn api_users(State(api_state): State<ApiState>) { /* ... */ }
```

`#[derive(FromRef)]` on the parent state generates these impls automatically for each
field, so any field type can be extracted as its own `State<FieldType>`.

### `Extension` vs `State`

- `State` — global, configuration-shaped data: DB pools, config, shared caches.
  Type-checked at compile time; the wrong type is a compile error.
- `Extension` — request-derived data inserted by middleware: auth claims, the
  authenticated user, per-request computed values. Resolved at runtime; a missing
  extension is a runtime rejection (`500`-class).

Documented guidance: "State is global and used in every request a router with state
receives. For accessing data derived from requests, such as authorization data, see
Extension." Prefer `State` whenever the data is known at startup; use `Extension`
only for data produced during request handling.

### The "missing state" compile error

A handler that extracts `State<AppState>` requires the router to eventually be given
that state. Two mistakes trigger compile errors:

1. Forgetting `.with_state(...)` entirely — the `Router` keeps a non-`()` `S` type
   parameter and cannot be served; the error mentions an unsatisfied trait bound
   tying the handler's `State<AppState>` to the router's state type.
2. Calling `.with_state(WrongType {})` — the handler expects `AppState`, the router
   supplies `OtherState`, producing a type-mismatch error.

Both fail at compile time, never at runtime — this is a deliberate Axum design
strength. When you see a confusing trait-bound error referencing `Handler` and
`State`, the fix is almost always a missing or wrong-typed `.with_state()`.

---

## 6. WebSockets

Requires the `ws` Cargo feature. Module: `axum::extract::ws`.

### `WebSocketUpgrade` and `on_upgrade`

`WebSocketUpgrade` is a `FromRequestParts` extractor that performs the HTTP->WebSocket
protocol upgrade.

```rust
pub fn on_upgrade<C, Fut>(self, f: C) -> Response
where
    C: FnOnce(WebSocket) -> Fut + Send + 'static,
    Fut: Future<Output = ()> + Send + 'static;
```

The callback receives the live `WebSocket` and owns the full connection lifecycle;
`on_upgrade` itself returns the `101 Switching Protocols` `Response` immediately. The
`OnFailedUpgrade` trait (default `DefaultOnFailedUpgrade`) controls behavior when the
upgrade itself fails; customize via `.on_failed_upgrade(...)`.

Verified verbatim echo handler:

```rust
use axum::{
    extract::ws::{WebSocketUpgrade, WebSocket},
    routing::any,
    response::Response,
    Router,
};

let app = Router::new().route("/ws", any(handler));

async fn handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(msg) = socket.recv().await {
        let msg = if let Ok(msg) = msg { msg } else { return; };
        if socket.send(msg).await.is_err() { return; }
    }
}
```

Combine with `State`: `WebSocketUpgrade` is a parts extractor, so
`async fn handler(ws: WebSocketUpgrade, State(s): State<AppState>) -> Response` works,
and you `move` the state into the `on_upgrade` closure.

### `WebSocket` methods

- `async fn recv(&mut self) -> Option<Result<Message, Error>>`
- `async fn send(&mut self, msg: Message) -> Result<(), Error>`
- `fn split(self) -> (SplitSink<WebSocket, Message>, SplitStream<WebSocket>)`
  (via `futures_util::StreamExt`)

### `Message` enum and the 0.7 -> 0.8 type change

Variants: `Text`, `Binary`, `Ping`, `Pong`, `Close` (with an optional `CloseFrame`
carrying a close code and reason).

VERSION-CRITICAL — Axum 0.8 breaking change. Verified against the CHANGELOG:
"`axum::extract::ws::Message` now uses `Bytes` in place of `Vec<u8>`, and a new
`Utf8Bytes` type in place of `String`, for its variants."

```rust
// axum 0.7
Message::Text(String)
Message::Binary(Vec<u8>)
Message::Ping(Vec<u8>)

// axum 0.8
Message::Text(Utf8Bytes)
Message::Binary(Bytes)
Message::Ping(Bytes)
```

`Utf8Bytes` and `Bytes` are cheap-to-clone reference-counted buffers. Code that did
`Message::Text("hi".to_string())` on 0.7 must become `Message::Text("hi".into())` (or
`Utf8Bytes::from_static("hi")`) on 0.8. This is the most common 0.7->0.8 migration
break in WebSocket code.

### Concurrent send/receive

A single `WebSocket` cannot be `send`-ed and `recv`-ed from two tasks at once. Split
it and drive each half in its own task; verified pattern from the official example:

```rust
let (mut sender, mut receiver) = socket.split();

let mut send_task = tokio::spawn(async move {
    // sender.send(Message::Text(...)).await ...
});
let mut recv_task = tokio::spawn(async move {
    while let Some(Ok(msg)) = receiver.next().await { /* process */ }
});

// abort the survivor when either side finishes
tokio::select! {
    _ = (&mut send_task) => recv_task.abort(),
    _ = (&mut recv_task) => send_task.abort(),
}
```

NEVER run blocking or CPU-heavy work directly inside the WebSocket handler task — it
stalls the connection and starves the Tokio worker. Offload to `tokio::spawn` /
`tokio::task::spawn_blocking`.

---

## 7. Server-Sent Events

Module: `axum::response::sse`. SSE is a one-way server->client text stream over a
normal HTTP response — simpler than WebSockets when you only push.

### `Sse`, `Event`, `KeepAlive`

`Sse::new(stream)` wraps a `Stream` whose item type is `Result<Event, E>`; `Sse<S>`
handles SSE wire encoding. The handler returns `Sse<impl Stream<Item = Result<Event, E>>>`.

`Event` builder methods: `.data(value)`, `.event(name)` (event type), `.id(value)`,
`.retry(duration)` (client reconnect hint), `.comment(text)` (ignored by clients;
also usable as a keep-alive). `EventDataWriter` implements `std::fmt::Write` for
formatted data without manual string building.

`KeepAlive` injects periodic keep-alive events on idle connections:
`KeepAlive::new()`, `.interval(duration)`, `.text(content)`. Attach via
`.keep_alive(...)`.

Verified verbatim official example:

```rust
async fn sse_handler(
    TypedHeader(user_agent): TypedHeader<headers::UserAgent>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok)
        .throttle(Duration::from_secs(1));

    Sse::new(stream).keep_alive(
        axum::response::sse::KeepAlive::new()
            .interval(Duration::from_secs(1))
            .text("keep-alive-text"),
    )
}
```

The stream item must be `Result<Event, E>`; use `.map(Ok)` to lift an infallible
`Stream<Item = Event>` into `Stream<Item = Result<Event, Infallible>>`. ALWAYS attach
a `KeepAlive` for long-lived SSE connections, otherwise idle connections may be
dropped by proxies and the client cannot tell a live-but-quiet stream from a dead one.

---

## 8. File Upload (Multipart)

Extractor: `axum::extract::Multipart` (`multipart/form-data`).

### Extracting fields

```rust
pub async fn next_field(&mut self) -> Result<Option<Field<'_>>, MultipartError>
```

Loop until `None`. Verified verbatim:

```rust
async fn upload(mut multipart: Multipart) {
    while let Some(mut field) = multipart.next_field().await.unwrap() {
        let name = field.name().unwrap().to_string();
        let data = field.bytes().await.unwrap();
        println!("Length of `{}` is {} bytes", name, data.len());
    }
}
```

CRITICAL ordering rule: `Multipart` consumes the request body, so it is a
`FromRequest` extractor and MUST be the **last** extractor argument in a handler.
Putting another body extractor after it is a compile error; putting it before a parts
extractor is fine but conventionally it goes last.

### `Field` methods

`.name()`, `.file_name()`, `.content_type()`, `.bytes()` (whole field into a `Bytes`),
`.text()` (whole field into a `String`), `.chunk()` (one streamed chunk).

### Streaming large files to disk

`.bytes()` buffers the entire field in memory — fine for small fields, dangerous for
large uploads. Stream chunk by chunk instead:

```rust
while let Some(chunk) = field.chunk().await.unwrap() {
    file.write_all(&chunk).await?;   // tokio::fs::File
}
```

### Body size limits — the multipart footgun

Axum's `DefaultBodyLimit` caps request bodies at **2 MB (2,097,152 bytes)** by
default ("For security reasons, `Bytes` will, by default, not accept bodies larger
than 2MB"). This silently blocks any multipart upload above 2 MB with a
`413`-class rejection. To raise or remove it:

```rust
use axum::extract::DefaultBodyLimit;

// raise to an explicit value
router.layer(DefaultBodyLimit::max(10 * 1024 * 1024)); // 10 MB

// or remove the default entirely (then set your own with tower-http)
router
    .layer(DefaultBodyLimit::disable())
    .layer(tower_http::limit::RequestBodyLimitLayer::new(100 * 1024 * 1024));
```

ALWAYS, when calling `DefaultBodyLimit::disable()`, follow it with a
`RequestBodyLimitLayer` so the server still has a hard ceiling — an unlimited body is
a memory-exhaustion DoS vector.

---

## 9. Static File Serving

Services: `tower_http::services::ServeDir` and `ServeFile` (feature `fs`). These are
`tower::Service`s, not layers, so they mount via `.nest_service()` or
`.fallback_service()`, NOT `.layer()`.

### `ServeDir` / `ServeFile`

`ServeDir::new("path")` serves a directory. `ServeFile::new("path")` serves a single
file. `ServeDir` builder: `.not_found_service(svc)` and `.fallback(svc)` define what
happens when a path does not resolve to a file.

```rust
// mount static assets under /assets
Router::new().nest_service("/assets", ServeDir::new("assets"))
```

### SPA fallback pattern

For single-page apps, any unmatched path should return `index.html` so the
client-side router can take over. Verified verbatim official pattern:

```rust
let index_html = SetStatus::new(
    ServeFile::new("assets/index.html"),
    StatusCode::NOT_FOUND,
);
let serve_dir = ServeDir::new("assets")
    .not_found_service(index_html.clone());

let app = Router::new()
    .route("/foo", get(|| async { "Hi from /foo" }))
    .fallback_service(serve_dir);
```

`.fallback_service()` (vs `.fallback()`) takes a `Service` — exactly what `ServeDir`
is. `.nest_service()` strips the matched prefix from the URI before passing the
request on, so `ServeDir::new("assets")` nested at `/assets` resolves `/assets/x.css`
to the file `assets/x.css`.

---

## Anti-Patterns (Cluster B)

**AP-1 — Calling `.layer()` before routes are registered.**
`Router::new().layer(TraceLayer::new_for_http()).route("/", get(h))` leaves `/`
without tracing. The documented rule: "the middleware is only applied to existing
routes." WHY it fails: `.layer()` wraps the `Router`'s current route set; routes added
afterwards are never wrapped. FIX: register all routes/fallbacks first, then `.layer()`.

**AP-2 — Confusing `ServiceBuilder` ordering with stacked `.layer()` ordering.**
Developers assume `router.layer(A).layer(B)` and
`ServiceBuilder::new().layer(A).layer(B)` produce the same nesting. They are
opposites. WHY it fails: stacked `Router::layer()` is bottom-to-top (last call =
outermost); `ServiceBuilder` is top-to-bottom (first call = outermost, "Layers that
are added first will be called with the request first"). Getting it wrong silently
reorders auth/timeout/trace and produces wrong behavior (for example timeout placed
inside compression instead of outside). FIX: pick one mental model — build the global
stack with a single `ServiceBuilder` and apply it through one `.layer()` call.

**AP-3 — `CorsLayer::permissive()` / `very_permissive()` in production.**
Shipping `CorsLayer::permissive()` to a public API allows every origin. WHY it fails:
it disables the cross-origin protection CORS exists to provide; combined with
credentialed endpoints it exposes authenticated data to any site. The deeper footgun:
`allow_credentials(true)` cannot legally be combined with a wildcard `Any` origin —
browsers reject it — which is exactly why `very_permissive()` echoes the concrete
origin instead of sending `*`. FIX: in production use `CorsLayer::new()` with an
explicit `.allow_origin(...)` allowlist; reserve `permissive()` for local dev.

**AP-4 — Blocking / CPU-bound work inside a WebSocket or SSE handler task.**
Doing `std::thread::sleep`, synchronous file IO, or a heavy CPU loop directly in the
`on_upgrade` task. WHY it fails: it blocks a Tokio worker thread, stalling that
connection and starving every other task scheduled on the same worker; the WebSocket
stops responding to pings and the peer eventually times out. FIX: `tokio::spawn` for
concurrent async work, `tokio::task::spawn_blocking` for synchronous/CPU work, and
split the socket so send and receive run in independent tasks.

**AP-5 — `DefaultBodyLimit` (2 MB) silently blocking multipart uploads.**
A file-upload endpoint works for tiny files and mysteriously `413`s on real uploads.
WHY it fails: Axum applies a 2 MB default body limit for security; any
`multipart/form-data` body above 2 MB is rejected before the handler runs. FIX: raise
it with `DefaultBodyLimit::max(n)` for the upload route, or `DefaultBodyLimit::disable()`
plus an explicit `RequestBodyLimitLayer::new(n)` ceiling.

**AP-6 — Holding a `std::sync::Mutex` guard across `.await`.**
`let g = state.data.lock().unwrap(); some_async_call().await; use(g);` WHY it fails:
the `MutexGuard` is not `Send`, so the future becomes non-`Send` and Axum's `Handler`
trait bound rejects it with a confusing compile error; even if it compiled it would
hold the lock across suspension and serialize all requests. FIX: scope the guard so it
drops before any `.await`, or use `tokio::sync::Mutex` whose guard is `Send`.

---

## Newly Discovered Sub-Topics (Cluster B)

The masterplan's planned Cluster B topics are: `axum-impl-middleware`,
`axum-impl-tower-stack`, `axum-impl-websockets`, `axum-impl-sse`,
`axum-impl-file-upload`, `axum-impl-static-files`, `axum-core-state`. Research
surfaced the following items that are NOT cleanly covered by those and should be
considered for the masterplan:

1. **0.7 -> 0.8 migration of WebSocket `Message` types** (`Vec<u8>`/`String` ->
   `Bytes`/`Utf8Bytes`). High-impact breaking change; warrants either a dedicated
   migration skill or a prominent version-difference section inside
   `axum-impl-websockets`.

2. **Routing path-syntax change** (`/:id` -> `/{id}`, `/*rest` -> `/{*rest}`,
   introduced in the 0.8 alpha line via the `matchit` 0.8 upgrade). Not strictly
   Cluster B but every code example in B uses routes, so it must be flagged so 0.7 vs
   0.8 examples are annotated consistently across the whole package.

3. **Native async-trait extractors** — `FromRequestParts` / `FromRequest` are now
   plain native async traits (`fn ... -> impl Future + Send`), so `#[async_trait]` is
   no longer needed for custom extractors. Belongs in a custom-extractor topic
   (likely a `axum-syntax-extractors` or `axum-core-extractors` skill, currently not
   in Cluster B's list).

4. **Rate limiting has no first-party tower-http layer.** Production rate limiting
   needs either `tower::limit` (`RateLimitLayer` + `Buffer`, `ConcurrencyLimitLayer`)
   or the ecosystem `tower_governor` crate. `axum-impl-tower-stack` should explicitly
   cover this gap and name `tower_governor` as the recommended approach.

5. **`route_layer` vs `layer` as a distinct decision point.** The 404-vs-401 auth
   interaction is subtle enough to deserve its own decision tree inside
   `axum-impl-middleware` (or a shared `axum-core-routing` skill).

6. **`SetStatus` from tower-http** is required for the canonical SPA fallback (return
   `index.html` with a `404` status). Worth an explicit pattern in
   `axum-impl-static-files`.

---

## Sources Verified (Cluster B)

All URLs below were fetched via WebFetch on **2026-05-20**. Current docs.rs version
observed: axum 0.8.9.

| URL | Verified | Used for |
|-----|----------|----------|
| https://docs.rs/axum/latest/axum/middleware/index.html | 2026-05-20 | Section 1 (middleware module overview, Layer vs from_fn) |
| https://docs.rs/axum/latest/axum/middleware/fn.from_fn.html | 2026-05-20 | Section 1 (`from_fn` exact signature + example) |
| https://docs.rs/axum/latest/axum/middleware/fn.from_fn_with_state.html | 2026-05-20 | Section 1 (`from_fn_with_state` signature + example) |
| https://docs.rs/axum/latest/axum/struct.Router.html | 2026-05-20 | Section 2 (`.layer`, `.route_layer`, `.nest`, `.fallback`, `.merge`) |
| https://docs.rs/axum/latest/axum/index.html | 2026-05-20 | Section 5 / sub-topics (routing syntax, version) |
| https://docs.rs/axum/latest/axum/extract/struct.State.html | 2026-05-20 | Section 5 (`State`, `FromRef`, Extension vs State, missing-state error) |
| https://docs.rs/axum/latest/axum/extract/trait.FromRequestParts.html | 2026-05-20 | Sub-topics (native async trait, no `#[async_trait]`) |
| https://docs.rs/axum/latest/axum/extract/ws/index.html | 2026-05-20 | Section 6 (WebSocketUpgrade, WebSocket, Message) |
| https://docs.rs/axum/latest/axum/response/sse/index.html | 2026-05-20 | Section 7 (Sse, Event, KeepAlive) |
| https://docs.rs/axum/latest/axum/extract/struct.Multipart.html | 2026-05-20 | Section 8 (`next_field`, `Field` methods) |
| https://docs.rs/axum/latest/axum/extract/struct.DefaultBodyLimit.html | 2026-05-20 | Section 8 (2 MB default, `max`, `disable`) |
| https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md | 2026-05-20 | Section 6 / sub-topics (0.7->0.8 breaking changes) |
| https://github.com/tokio-rs/axum/blob/main/examples/websockets/src/main.rs | 2026-05-20 | Section 6 (split, spawn, select, Message matching) |
| https://github.com/tokio-rs/axum/blob/main/examples/sse/src/main.rs | 2026-05-20 | Section 7 (verbatim SSE handler) |
| https://github.com/tokio-rs/axum/blob/main/examples/static-file-server/src/main.rs | 2026-05-20 | Section 9 (ServeDir, SPA fallback) |
| https://docs.rs/tower/latest/tower/index.html | 2026-05-20 | Section 3 (Service, Layer traits) |
| https://docs.rs/tower/latest/tower/builder/struct.ServiceBuilder.html | 2026-05-20 | Section 3 (ServiceBuilder ordering, verified verbatim) |
| https://docs.rs/tower-http/latest/tower_http/index.html | 2026-05-20 | Section 4 (layer inventory + feature flags) |
| https://docs.rs/tower-http/latest/tower_http/cors/struct.CorsLayer.html | 2026-05-20 | Section 4 (CORS builder, permissive vs very_permissive) |
| https://docs.rs/tower-http/latest/tower_http/limit/struct.RequestBodyLimitLayer.html | 2026-05-20 | Section 4 / 8 (`RequestBodyLimitLayer::new`) |

### Verification notes / caveats

- The tower-http `CorsLayer` page did not print an explicit verbatim "do not use in
  production" sentence; the production warning in AP-3 is derived from the documented
  behavior of `permissive`/`very_permissive` plus the well-established CORS spec rule
  that `*` origin is invalid with credentials (which `very_permissive()` works around
  by echoing the origin). This is flagged as inference, not a direct quote.
- The async-trait removal is confirmed indirectly: the live `FromRequestParts` trait
  definition uses native `-> impl Future + Send` (no `#[async_trait]`). The 0.8.0
  CHANGELOG section fetched did not surface an explicit "removed async_trait" line, so
  a future pass should confirm the exact CHANGELOG wording before a migration skill
  states it as a quoted breaking change.
- The path-syntax change (`/:id` -> `/{id}`) was confirmed via the CHANGELOG note
  about upgrading `matchit` to 0.8 in the 0.8 alpha line; the crate root example uses
  static paths so it neither confirms nor contradicts. Treat `/{id}` as the 0.8 syntax
  and `/:id` as the 0.7 syntax in all examples.
