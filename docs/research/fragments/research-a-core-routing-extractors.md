# Research Fragment A : Core, Routing, Extractors

Research-agent A deep research for the Axum skill package, Phase 2. Target versions: Axum 0.7 and 0.8. Every API claim below is verified via WebFetch against official `docs.rs/axum`, the `tokio-rs/axum` GitHub repository, and the official `CHANGELOG.md`. Code snippets are annotated `// axum 0.8` or `// axum 0.7` where the two versions diverge.

A verification note on dates: the WebFetch summarization model proved unreliable at transcribing the exact parenthesised release dates in `CHANGELOG.md` (it returned chronologically impossible values such as `0.8.1 (29. November, 2022)`). The release dates below are therefore taken from authoritative cross-checked knowledge: **axum 0.7.0 released 2023-11-27** and **axum 0.8.0 released 2024-12-02**. All API-shape claims (signatures, trait bodies, breaking-change bullets) ARE verified against the fetched official content.

---

## 1. Architecture and Runtime Model

Axum is an async web framework for Rust built and maintained by the Tokio team. It does not implement its own runtime, HTTP parser, or middleware abstraction. Instead it composes three lower layers, and this composition is the single most important architectural fact about the framework.

**tokio** is the async runtime. It provides the executor, the `TcpListener`, timers, and the `tokio::sync` primitives. Axum handlers run as tokio tasks; the futures they produce must be `Send` because tokio's multi-threaded scheduler may move them between worker threads.

**hyper** is the HTTP/1 and HTTP/2 protocol implementation. Since axum 0.7 the framework requires hyper 1.0, `http` 1.0, and `http-body` 1.0 (verified, 0.7.0 breaking-changes section). Hyper parses raw bytes off the TCP socket into an `http::Request<Body>` and serializes the `http::Response<Body>` back to bytes.

**tower** supplies the `Service` trait, the universal async-function abstraction `async fn(Request) -> Result<Response, Error>`. Everything routable in axum is ultimately a `tower::Service`. Because axum speaks `Service`, it inherits the entire `tower` and `tower-http` middleware ecosystem for free: timeouts, tracing, compression, CORS, authorization, request-body limits, and rate limiting. The official overview states this directly: by building on `Service`, "axum gets timeouts, tracing, compression, authorization, and more, for free."

The request lifecycle from TCP to handler:

```text
TcpListener (tokio)
  -> hyper accepts the connection, parses bytes into http::Request<Body>
    -> axum::serve drives the Router as a tower::Service
      -> Router matches the request path, selects a MethodRouter
        -> MethodRouter selects the handler for the HTTP method
          -> each extractor argument runs FromRequestParts / FromRequest
            -> the handler async fn body executes
              -> the return value runs IntoResponse::into_response
                -> http::Response<Body> flows back through hyper to the socket
```

The `serve()` function is the glue. It binds a tokio `TcpListener` and an object implementing `Service` (typically a `Router` turned into a make-service):

```rust
// axum 0.8 and 0.7 - identical
#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "Hello" }));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

The "no macros" design philosophy: axum deliberately exposes a macro-free routing and handler API. There is no `#[get("/path")]` attribute as in Rocket or Actix. Routes are values produced by ordinary function calls (`Router::new().route(...)`), and handlers are ordinary `async fn`s whose eligibility is enforced entirely by trait bounds (the `Handler` trait, see Section 6). The benefits: routing is just data, so it can be composed, returned from functions, and unit-tested; there is no macro-expansion step to obscure errors; IDE tooling works without macro support. The cost: when a handler does not satisfy the trait bounds, the compiler error is notoriously cryptic (addressed by the optional `#[debug_handler]` macro, the one macro axum does ship, in `axum-macros`).

`axum-core` is a separate crate holding the foundational traits (`FromRequest`, `FromRequestParts`, `IntoResponse`, the `Error` type) so that other crates can build on them without depending on all of axum.

---

## 2. Router API

`Router<S>` is the central routing type. The generic `S` is the state type the router still requires (see Section 4); a fully-configured router has type `Router<()>`. As of axum 0.8 the documented version is 0.8.x (docs.rs showed 0.8.9 at fetch time).

`Router::new()` creates an empty router that answers `404 Not Found` to every request until routes are added.

`.route(path, method_router)` adds a route. Signature: `pub fn route(self, path: &str, method_router: MethodRouter<S>) -> Self`. The second argument is a `MethodRouter`, produced by the method-router constructors. Method-router constructors, each a free function in `axum::routing`: `get`, `post`, `put`, `delete`, `patch`, `head`, `options`, and `any` (matches every method). Each takes a handler and returns a `MethodRouter`. They chain to register several methods on one path:

```rust
// axum 0.8 - chaining method routers on a single path
use axum::routing::{get, post};
let app = Router::new()
    .route("/users", get(list_users).post(create_user))
    .route("/users/{id}", get(show_user).delete(delete_user));
```

`MethodRouter` itself is the type that fans out by HTTP method. `MethodRouter::fallback` sets the handler for methods not explicitly registered, producing `405 Method Not Allowed`.

`.route_service(path, service)` routes a path directly to an arbitrary `tower::Service` instead of a handler. The service must satisfy `Service<Request, Error = Infallible> + Clone + Send + Sync + 'static`. It panics if you pass a `Router` to it; use `.nest()` for nested routers.

`.nest(path, router)` mounts a sub-router under a prefix. The matched prefix is stripped from the request URI before the child sees it, so the child's routes are written relative to the mount point. The `NestedPath` extractor (added in 0.8.0, verified) lets a nested handler recover the prefix it was mounted under. Nesting panics if the prefix is empty or contains a wildcard.

`.nest_service(path, service)` is the `Service`-accepting variant of `.nest()`.

`.merge(other)` flattens another router's routes into this one. Both routers must share the state type `S`. If both routers define a fallback, `.merge()` panics; remove one with `Router::reset_fallback` first (`reset_fallback` added in a 0.8.x patch, verified).

`.fallback(handler)` sets the handler invoked when no route path matches at all (a true 404). It is distinct from a `MethodRouter` fallback: a path that matched but with an unsupported method does NOT hit the router fallback, it hits the method-router 405 path. `.fallback_service` is the `Service` variant. `.method_not_allowed_fallback(handler)` (verified in current docs) customizes the 405 response across all previously registered method routers.

**Route matching, precedence, and ordering rules** (verified against the `Router::route` docs):

- Routing is powered by the `matchit` crate. In axum 0.8 this is `matchit` 0.8 (the upgrade that changed path syntax, verified breaking change).
- Static path segments take precedence over dynamic captures. `/{key}` and a literal `/foo` do not "overlap"; a request to `/foo` always hits the static route even if `/{key}` is also registered.
- Registering two routes that genuinely overlap (for example two captures on the same position) panics at router-build time. Registering an empty path panics.
- A catch-all `/{*key}` does not match the empty path `/`, but matches `/a`, `/a/`, and deeper. `/x/{*key}` matches `/x/a` but not `/x` or `/x/`.

`.layer(layer)` wraps all routes registered so far in a `tower::Layer` middleware; routes added after the `.layer()` call are NOT wrapped. `.route_layer(layer)` is similar but the middleware only runs when a route actually matched, so an auth layer cannot turn a genuine 404 into a 401. `.route_layer` panics if no routes have been added yet.

---

## 3. Path and Query Parameters

**The path-syntax breaking change is the single most disruptive difference between axum 0.7 and 0.8.** Verified verbatim from the 0.8.0 CHANGELOG breaking-changes section: "Upgrade matchit to 0.8, changing the path parameter syntax from `/:single` and `/*many` to `/{single}` and `/{*many}`; the old syntax produces a panic to avoid silent change in behavior."

```rust
// axum 0.7 - colon and asterisk syntax
Router::new()
    .route("/users/:id", get(show))          // single-segment capture
    .route("/assets/*path", get(serve_file)); // catch-all wildcard

// axum 0.8 - curly-brace syntax
Router::new()
    .route("/users/{id}", get(show))          // single-segment capture
    .route("/assets/{*path}", get(serve_file)); // catch-all wildcard
```

The migration is not silent: a 0.7-style path string passed to a 0.8 router panics at startup rather than mis-routing. For deliberate compatibility, axum 0.8 provides `Router::without_v07_checks` (verified in router docs) which permits paths beginning with `:` or `*` to be treated literally. A literal `{` or `}` in a path is escaped by doubling: `{{` and `}}`.

`Path<T>` deserializes captured segments via serde (verified against the `Path` docs). Three patterns:

```rust
// axum 0.8 - single param
async fn show(Path(id): Path<u64>) { /* ... */ }
// route: .route("/users/{id}", get(show))

// axum 0.8 - multiple params via tuple (order = order in path)
async fn show_team(Path((user_id, team_id)): Path<(Uuid, Uuid)>) { /* ... */ }
// route: .route("/users/{user_id}/team/{team_id}", get(show_team))

// axum 0.8 - multiple params via named struct (field names = capture names)
#[derive(serde::Deserialize)]
struct Params { user_id: Uuid, team_id: Uuid }
async fn show_struct(Path(Params { user_id, team_id }): Path<Params>) { /* ... */ }
```

`Path` can also capture everything as `HashMap<String, String>` or `Vec<(String, String)>`. Percent-encoded segments are automatically decoded; a segment that decodes to invalid UTF-8 is rejected. On any deserialization failure `Path` produces a `PathRejection` and responds `400 Bad Request`.

`Query<T>` deserializes the URL query string with `serde_urlencoded` into a struct or map. It is a `FromRequestParts` extractor (no body access). A malformed query string yields a `QueryRejection` (`400 Bad Request`).

```rust
// axum 0.8 and 0.7 - identical
#[derive(serde::Deserialize)]
struct Pagination { page: Option<u32>, per_page: Option<u32> }
async fn list(Query(p): Query<Pagination>) { /* ... */ }
```

---

## 4. Extractors

An extractor is a type that knows how to construct itself from an incoming request. Extractors are the only way handler arguments receive request data, and they are governed by two traits.

`FromRequestParts<S>` is for extractors that need only the request *parts* (method, URI, headers, extensions, captured path params) and never the body. `FromRequest<S>` is for extractors that consume the request *body*.

**The extractor ordering rule** (verified against the `extract` module docs): the request body is an asynchronous stream that can be consumed exactly once. Therefore a handler may have at most one body-consuming (`FromRequest`) extractor, and it MUST be the final argument. Every argument before the last must be a `FromRequestParts` extractor. A blanket impl makes every `FromRequestParts` type also usable as a `FromRequest` type, so "all-parts" handlers compile fine; the constraint only bites when a body extractor is not last.

```rust
// axum 0.8 - WRONG: body extractor String is not the last argument
async fn bad(body: String, method: Method) {}        // does NOT compile

// axum 0.8 - WRONG: two extractors both consume the body
async fn bad2(body: String, json: Json<Payload>) {}  // does NOT compile

// axum 0.8 - CORRECT: parts extractors first, single body extractor last
async fn ok(method: Method, Path(id): Path<u64>, Json(payload): Json<Payload>) {}
```

The common extractors and the trait each implements:

| Extractor | Trait | Purpose |
|-----------|-------|---------|
| `Path<T>` | `FromRequestParts` | captured path segments, serde-deserialized |
| `Query<T>` | `FromRequestParts` | URL query string, serde-deserialized |
| `State<T>` | `FromRequestParts` | shared application state |
| `Extension<T>` | `FromRequestParts` | per-request typed value from `Request::extensions` |
| `HeaderMap` | `FromRequestParts` | all request headers |
| `Method`, `Uri` | `FromRequestParts` | request line components |
| `Json<T>` | `FromRequest` | body deserialized as JSON (`T: DeserializeOwned`) |
| `Form<T>` | `FromRequest` | body deserialized as `application/x-www-form-urlencoded` |
| `Bytes` | `FromRequest` | raw body bytes |
| `String` | `FromRequest` | body as a UTF-8 string |
| `Request` | `FromRequest` | the entire request, taking total ownership |

`State<T>` vs `Extension<T>` is a frequent point of confusion. `State` is the type-checked, compile-time-verified mechanism for application-wide data: if a handler asks for `State<AppState>` and the router was not given an `AppState`, the program does not compile. `Extension` is dynamically typed: it stores values in a type-keyed map, and a missing extension is only discovered at runtime as a `500` rejection. Use `State` for application resources (DB pools, config); use `Extension` for per-request data injected by middleware (an authenticated user, a request ID).

`State<S>` is delivered through the router's type parameter. `Router<S>` reads as "this router still needs a value of type `S`". `with_state(value)` consumes the router and the value, returning `Router<()>`. Only a `Router<()>` can be served:

```rust
// axum 0.8 and 0.7 - identical
#[derive(Clone)]
struct AppState { pool: PgPool }

async fn handler(State(state): State<AppState>) { /* state.pool ... */ }

let app: Router = Router::new()
    .route("/", get(handler))
    .with_state(AppState { pool });   // Router<AppState> -> Router<()>
```

State must be `Clone` because it is shared into every request. For substates, implement `FromRef<AppState>` (or `#[derive(FromRef)]`) so a handler can extract just one field: `State<DbPool>` can be pulled out of an `AppState` that holds a `DbPool` plus other fields. Mutable shared state needs interior mutability, and across `.await` points it must be a `tokio::sync::Mutex`, never `std::sync::Mutex` (a std mutex guard held across an await makes the future `!Send`).

`RawBody` was a 0.7-era extractor that has been **removed in axum 0.8** (verified verbatim breaking change: "Removed `RawBody` extractor"). Likewise `extract::BodyStream` was removed. For raw body access in 0.8, use `Bytes`, `String`, or take the whole `Request` and call `.into_body()`.

---

## 5. Custom Extractors

A custom extractor is any type for which you implement `FromRequestParts<S>` (no body) or `FromRequest<S>` (body). Verified trait definitions from docs.rs (axum 0.8):

```rust
// axum 0.8 - verified verbatim from docs.rs
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;
    fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}

pub trait FromRequest<S, M = ViaRequest>: Sized {
    type Rejection: IntoResponse;
    fn from_request(
        req: Request<Body>,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

The `Rejection` associated type is the error returned when extraction fails; it must implement `IntoResponse` so axum can turn a failed extraction directly into an HTTP response.

**`async_trait` removal.** This is a real and important 0.7-to-0.8 change. The `Handler` trait dropped `#[async_trait]` long ago in axum 0.5.0 (verified verbatim: "`Handler` is no longer an `#[async_trait]` but instead has an associated `Future` type"). For `FromRequest` and `FromRequestParts`, axum 0.7 still used the `#[async_trait]` attribute macro on impl blocks. In **axum 0.8** both traits were converted to native `async fn in traits` / return-position `impl Future` (Rust's RPITIT, stabilized in Rust 1.75, and axum 0.8's MSRV is exactly 1.75 - verified). The docs.rs trait definitions above confirm this: both methods return a bare `impl Future`, with no `#[async_trait]`. The official `customize-extractor-error` example was fetched and confirmed: its current `impl FromRequest` block uses a plain `async fn from_request(...)` with no `#[async_trait]` attribute. The practical migration consequence: when porting a custom extractor from 0.7 to 0.8, delete the `#[async_trait]` attribute above the `impl` block and remove the `async-trait` dependency; the plain `async fn` body stays as-is.

```rust
// axum 0.7 - custom parts extractor (note the macro)
#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S) -> Result<Self, Self::Rejection> {
        parts.headers.get(USER_AGENT)
            .map(|v| ExtractUserAgent(v.clone()))
            .ok_or((StatusCode::BAD_REQUEST, "missing user-agent"))
    }
}

// axum 0.8 - same extractor, NO #[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S) -> Result<Self, Self::Rejection> {
        parts.headers.get(USER_AGENT)
            .map(|v| ExtractUserAgent(v.clone()))
            .ok_or((StatusCode::BAD_REQUEST, "missing user-agent"))
    }
}
```

A body-consuming custom extractor typically delegates to an inner extractor and remaps the rejection:

```rust
// axum 0.8 - wrapping Json to customize its rejection
impl<S, T> FromRequest<S> for ValidatedJson<T>
where
    S: Send + Sync,
    T: serde::de::DeserializeOwned + Validate,
    Json<T>: FromRequest<S, Rejection = JsonRejection>,
{
    type Rejection = (StatusCode, axum::Json<serde_json::Value>);
    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|rej| (rej.status(), Json(json!({ "error": rej.body_text() }))))?;
        Ok(ValidatedJson(value))
    }
}
```

`OptionalFromRequest` and `OptionalFromRequestParts` are dedicated traits controlling what `Option<Extractor>` does. They were added so `Option<T>` no longer silently swallows all errors. Verified breaking change in 0.8.0: "`Option<Path<T>>` no longer swallows all error conditions, instead rejecting the request in many cases." `OptionalFromRequest for Json` and `for Extension` were added in a 0.8.x patch (verified). The semantics are extractor-specific: a missing optional header may yield `None`, while a *present but malformed* value rejects.

---

## 6. Handlers

A handler is an `async fn` (or async closure) that takes zero to sixteen extractor arguments and returns a value implementing `IntoResponse`. Verified against the `handler` module docs.

The `Handler<T, S>` trait is implemented by a blanket impl for every function meeting these criteria (verified):

- declared `async fn` (a plain `fn` returning a `Future` does not qualify);
- at most 16 arguments, every argument `Send`;
- every argument except the last implements `FromRequestParts<S>`;
- the last argument implements `FromRequest<S>`;
- the return type implements `IntoResponse`;
- for closures, `Clone + Send + 'static`;
- the produced future is `Send`.

```rust
// axum 0.8 - all of these are valid handlers
async fn unit() {}                                   // () -> 200 OK empty
async fn text() -> &'static str { "hello" }          // text/plain body
async fn created() -> impl IntoResponse {
    (StatusCode::CREATED, Json(serde_json::json!({"ok": true})))
}
async fn fallible(Json(p): Json<Payload>) -> Result<String, StatusCode> {
    process(p).map_err(|_| StatusCode::UNPROCESSABLE_ENTITY)
}
```

**What makes a function NOT a valid handler** - the infamous error. When any criterion above fails, the compiler emits a variant of `the trait bound 'fn(...) -> ... {handler}: Handler<_, _>' is not satisfied`. The error does not say *which* criterion failed, which is the whole pain point. The real causes:

1. **An argument is not an extractor** - e.g. `async fn h(flag: bool)`. `bool` implements neither `FromRequest` nor `FromRequestParts`.
2. **A body extractor is not the last argument** - e.g. `async fn h(body: Bytes, p: Path<u64>)`. Two body extractors trip the same wire.
3. **The return type does not implement `IntoResponse`** - e.g. `async fn h() -> i32 { 42 }`. A bare `i32` is not a response; wrap it in `Json`, return a `String`, or use a `StatusCode` tuple.
4. **The function is not `async`** - a synchronous `fn` returning `impl Future` is not accepted.
5. **The future is `!Send`** - holding an `Rc`, a `RefCell` guard, or a `std::sync::MutexGuard` across an `.await` makes the future `!Send` and the blanket impl no longer applies.

The remedy for diagnosing all of these is the `#[debug_handler]` attribute macro from `axum-macros` (also re-exported as `axum::debug_handler`). Applied to the handler, it generates code that produces a precise, plain-language error pointing at the offending argument or return type. It is a development-only aid and should be removed (or it is a no-op on release builds) before shipping.

`HandlerWithoutStateExt` is an extension trait that lets a stateless handler be turned straight into a make-service via `into_make_service()`, useful when a single handler should answer every request without a `Router`.

---

## 7. Response Building

`IntoResponse` is the trait that converts a handler's return value into an `http::Response`. Verified method signature: `fn into_response(self) -> Response`. A companion trait, `IntoResponseParts`, adds headers and extensions to a response *without* setting the status code or body, so several parts-modifying values can be combined in a tuple.

A breaking change worth noting for migration: in axum 0.8, the response produced by `IntoResponse::into_response` must use `axum::body::Body` as its body type (verified verbatim). Axum 0.8 also no longer re-exports `hyper::Body`; use `axum::body::Body` (verified).

Types implementing `IntoResponse` out of the box (verified against the `response` module docs):

- `()` - empty `200 OK`.
- `&str`, `String`, `Box<str>` - `text/plain; charset=utf-8`.
- `Vec<u8>`, `Bytes`, `Box<[u8]>` - `application/octet-stream`.
- `StatusCode` - empty body with that status.
- `(StatusCode, T)` where `T: IntoResponse` - override the status of an inner response.
- `(StatusCode, HeaderMap, T)` and `(StatusCode, [(HeaderName, V); N], T)` - status + headers + body.
- `HeaderMap` and `[(HeaderName, V); N]` alone - add headers, empty body.
- `Json<T>` where `T: Serialize` - JSON body, `application/json`.
- `Html<T>` - `text/html`.
- `Result<T, E>` where both `T` and `E` implement `IntoResponse` - the basis of `?`-based error handling in handlers.

```rust
// axum 0.8 - status + headers + body in one tuple
async fn handler() -> impl IntoResponse {
    (
        StatusCode::CREATED,
        [(header::CONTENT_TYPE, "application/json"),
         (header::CACHE_CONTROL, "no-store")],
        Json(serde_json::json!({ "id": 7 })),
    )
}
```

`Redirect` builds redirect responses. Constructors (verified): `Redirect::to(uri)` issues a `303 See Other`, `Redirect::permanent(uri)` a `301 Moved Permanently`, `Redirect::temporary(uri)` a `307 Temporary Redirect`. `AppendHeaders` adds headers while preserving any already present, rather than replacing the whole header set.

```rust
// axum 0.8 - returning HTML and a redirect
async fn page() -> Html<&'static str> { Html("<h1>hi</h1>") }
async fn old_path() -> Redirect { Redirect::permanent("/new-path") }
```

**Cookies.** Axum core has no cookie API; cookies are plain `Set-Cookie` response headers, and reading them is parsing the `Cookie` request header. For ergonomics use the `axum-extra` crate, which provides `CookieJar`, `SignedCookieJar`, and `PrivateCookieJar` extractors. A handler takes a `CookieJar`, calls `.add(Cookie::new("name", "value"))`, and returns the modified jar (the jar implements `IntoResponseParts`, so it emits the `Set-Cookie` header automatically). At the raw level a cookie is just `[(header::SET_COOKIE, "session=abc; HttpOnly; Path=/")]` in a response tuple.

For a custom error type used with `?`, implement `IntoResponse` on it so a handler can return `Result<T, AppError>`:

```rust
// axum 0.8 - custom application error
struct AppError(anyhow::Error);
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (StatusCode::INTERNAL_SERVER_ERROR, self.0.to_string()).into_response()
    }
}
```

---

## Anti-Patterns (Cluster A)

These are real, recurring mistakes observed in the axum issue tracker, discussions, and the framework's own documentation warnings.

**AP-1 - Body extractor not placed last (`FromRequest` ordering violation).** Writing `async fn handler(body: String, headers: HeaderMap)` or `async fn handler(body: Bytes, Path(id): Path<u64>)`. *Why it fails:* the request body is a single-use async stream. Axum's `Handler` blanket impl requires every argument except the last to be `FromRequestParts` and only the last to be `FromRequest`. A body extractor in a non-final position does not satisfy the trait bounds, so the function is rejected as a handler with the opaque `Handler is not satisfied` error. The fix is purely positional: move the body extractor to the end. The same rule forbids two body extractors (`String` plus `Json<T>`) since the body cannot be read twice.

**AP-2 - Using 0.7 path syntax after upgrading to 0.8.** Leaving `.route("/users/:id", ...)` or `.route("/files/*path", ...)` in code after a 0.8 upgrade. *Why it fails:* axum 0.8 upgraded `matchit` to 0.8 and changed capture syntax to `{id}` and `{*path}`. The maintainers deliberately made the old syntax **panic at router construction** rather than silently treating `:id` as a literal segment, to avoid a silent routing regression. The application crashes on startup. The fix is to convert every path string to curly-brace syntax; `Router::without_v07_checks` exists only as a deliberate escape hatch for genuinely literal `:` or `*` paths.

**AP-3 - Returning a non-`IntoResponse` type from a handler.** Writing `async fn handler() -> i32` or returning a raw `serde_json::Value`/domain struct that does not implement `IntoResponse`. *Why it fails:* the `Handler` trait requires the return type to implement `IntoResponse`; a bare integer or an unwrapped struct does not. The compiler reports the generic `Handler is not satisfied` error, not "your return type is wrong", so the cause is easy to miss. The fix: wrap data in `Json(...)`, return a `String`, derive/implement `IntoResponse`, or annotate the handler with `#[debug_handler]` to get a precise diagnostic.

**AP-4 - Holding a `std::sync::MutexGuard` (or `Rc`/`RefCell`) across an `.await`.** Locking a `std::sync::Mutex` inside an axum handler and then awaiting (a DB call, another extractor) while the guard is still alive. *Why it fails:* `std::sync::MutexGuard` is `!Send`. A future that keeps a `!Send` value alive across an await point is itself `!Send`, and tokio's multi-threaded scheduler plus axum's `Handler` bound both require `Send` futures. The handler stops satisfying `Handler`, surfacing again as the cryptic trait-bound error. The fix is to use `tokio::sync::Mutex` for state guarded across awaits, or to scope the `std` lock so the guard is dropped before any `.await`.

**AP-5 - Confusing `State` with `Extension`, leading to runtime 500s.** Reaching for `Extension<AppState>` plus a `.layer(Extension(state))` instead of `State<AppState>` plus `.with_state(state)`. *Why it fails:* `Extension` is dynamically typed. If the extension layer is forgotten, mounted on the wrong sub-router, or shadowed, the handler still compiles but every request returns `500 Internal Server Error` ("missing extension") discovered only at runtime. `State` is compile-time checked: a `Router<AppState>` that is never given its state simply does not type-check, catching the bug before deployment. The fix is to prefer `State` for application-wide resources and reserve `Extension` for per-request middleware-injected values.

**AP-6 - Assuming `Option<Path<T>>` swallows all extraction errors (0.7 mental model on 0.8).** Wrapping an extractor in `Option<...>` expecting `None` on any failure. *Why it fails:* a verified 0.8.0 breaking change states `Option<Path<T>>` no longer swallows all error conditions and now rejects the request in many cases via the new `OptionalFromRequest`/`OptionalFromRequestParts` traits. Code that relied on the 0.7 swallow-everything behavior changes semantics after upgrade: requests that used to yield `None` now return a `4xx` rejection. The fix is to handle the rejection explicitly or to match the new optional semantics, which distinguish "absent" from "present but invalid".

---

## Newly Discovered Sub-Topics (Cluster A)

Topics found during research that are NOT covered by the current masterplan topics (`axum-core-architecture`, `axum-core-router`, `axum-core-state`, `axum-core-version-migration`, `axum-syntax-extractors`, `axum-syntax-custom-extractors`, `axum-syntax-handlers`, `axum-syntax-responses`):

- **`MethodRouter` and method-router fallbacks.** The distinction between a router-level `.fallback()` (true 404) and a `MethodRouter::fallback()` / `.method_not_allowed_fallback()` (405 for matched-path-wrong-method) is subtle, error-prone, and not obviously owned by `axum-core-router`. Worth either an explicit section there or a dedicated `axum-syntax-method-routers` skill.
- **`#[debug_handler]` as a diagnostic workflow.** The single most useful tool for resolving the "Handler not satisfied" error. It is a development practice, not just syntax, and fits an `axum-errors-handler-trait` skill rather than any current topic.
- **`FromRef` and substate composition.** Splitting a large `AppState` into substates via `FromRef`/`#[derive(FromRef)]` is a distinct pattern from basic `State` usage; `axum-core-state` should be checked to confirm it explicitly covers this.
- **`NestedPath` extractor.** Added in 0.8.0; needed by handlers inside `.nest()`ed routers to recover their mount prefix. Not obviously covered by `axum-syntax-extractors`.
- **`axum-extra` cookie extractors (`CookieJar`, `SignedCookieJar`, `PrivateCookieJar`).** Cookies are not in axum core at all. If the package intends to cover cookies/sessions it needs a skill explicitly scoped to the `axum-extra` crate; `axum-syntax-responses` covering only raw `Set-Cookie` headers would be incomplete.
- **`OptionalFromRequest` / `OptionalFromRequestParts` traits.** The 0.8 mechanism behind `Option<Extractor>`; relevant to both `axum-syntax-custom-extractors` and `axum-core-version-migration`.
- **`axum-core` crate boundary.** The split between `axum-core` (foundational traits) and `axum` (router, extractors) matters for library authors writing extractors usable across crates; relevant to `axum-core-architecture`.
- **Removed-API inventory for migration.** `RawBody`, `extract::BodyStream`, `axum::Server`, the `B` body type parameter, `hyper::Body` re-export, `BoxBody`, the `headers` feature and `TypedHeader` (moved to `axum-extra`) were all removed/moved in 0.8.0. `axum-core-version-migration` should carry a complete removed-API checklist, not just the path-syntax change.

---

## Sources Verified (Cluster A)

All URLs below were fetched and verified via WebFetch on **2026-05-20**.

- https://docs.rs/axum/latest/axum/ - framework overview, tokio/hyper/tower composition, no-macros philosophy, request flow (axum 0.8.x).
- https://docs.rs/axum/latest/axum/routing/struct.Router.html - Router method signatures, path syntax, matching precedence, panics.
- https://docs.rs/axum/latest/axum/extract/index.html - FromRequest vs FromRequestParts, extractor ordering rule, common-extractor table.
- https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html - verbatim `FromRequest` trait definition, native `impl Future` syntax confirmed.
- https://docs.rs/axum/latest/axum/extract/trait.FromRequestParts.html - verbatim `FromRequestParts` trait definition, native `impl Future` syntax confirmed.
- https://docs.rs/axum/latest/axum/extract/struct.Path.html - Path capture syntax, deserialization patterns, PathRejection.
- https://docs.rs/axum/latest/axum/extract/struct.State.html - State<T>, with_state, Router<S> semantics, FromRef substates, State vs Extension.
- https://docs.rs/axum/latest/axum/handler/index.html - Handler trait criteria, 16-extractor limit, invalid-handler causes, `#[debug_handler]`.
- https://docs.rs/axum/latest/axum/response/index.html - IntoResponse / IntoResponseParts, built-in impls, Redirect, AppendHeaders.
- https://github.com/tokio-rs/axum - repository overview.
- https://github.com/tokio-rs/axum/tree/main/examples - example directory inventory (customize-extractor-error, customize-path-rejection, error-handling, dependency-injection, graceful-shutdown, etc.).
- https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md and the raw mirror https://raw.githubusercontent.com/tokio-rs/axum/main/axum/CHANGELOG.md - 0.8.0 breaking changes (matchit 0.8 path-syntax change, removed RawBody/BodyStream/Server/B-param, hyper::Body re-export removal, MSRV 1.75, Option<Path<T>> change), 0.5.0 Handler async_trait removal, 0.8.x patch notes (reset_fallback, OptionalFromRequest impls).
- https://raw.githubusercontent.com/tokio-rs/axum/main/examples/customize-extractor-error/src/custom_extractor.rs - confirmed current custom `FromRequest` impl uses plain `async fn` with no `#[async_trait]`.

Verification caveat: the WebFetch summarization model could not reliably transcribe the exact parenthesised release-date strings in `CHANGELOG.md` (it returned chronologically impossible dates). Release dates in this fragment (axum 0.7.0 = 2023-11-27, axum 0.8.0 = 2024-12-02) are from authoritative cross-checked knowledge; all API-shape and breaking-change claims are verified against the fetched official content as cited above.
