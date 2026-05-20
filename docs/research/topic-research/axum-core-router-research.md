# Topic Research : axum-core-router

> Skill : `axum-core-router`
> Target versions : Axum 0.7 and 0.8 (docs.rs current at fetch : 0.8.x)
> Researched : 2026-05-20
> All API signatures WebFetch-verified against docs.rs on 2026-05-20.

This file supports the `axum-core-router` skill. Scope: `Router<S>`, `.route()`,
method-routers, `MethodRouter`, the 405-vs-404 distinction, `.nest()` /
`.nest_service()` / `.merge()`, route precedence and overlap panics, `.layer()` vs
`.route_layer()`, the `NestedPath` extractor, and the 0.7 -> 0.8 path-syntax change.

---

## 1. Exact Router signatures (verified)

`Router<S>` is the central routing type. `S` is the still-required state type; a
serveable router is `Router<()>`. Signatures below are verbatim from
docs.rs `axum::routing::Router` (fetched 2026-05-20):

```rust
pub fn new() -> Self

pub fn route(self, path: &str, method_router: MethodRouter<S>) -> Self

pub fn route_service<T>(self, path: &str, service: T) -> Self
where T: Service<Request, Error = Infallible> + Clone + Send + Sync + 'static

pub fn nest(self, path: &str, router: Router<S>) -> Self

pub fn nest_service<T>(self, path: &str, service: T) -> Self
where T: Service<Request, Error = Infallible> + Clone + Send + Sync + 'static

pub fn merge<R>(self, other: R) -> Self
where R: Into<Router<S>>

pub fn fallback<H, T>(self, handler: H) -> Self
where H: Handler<T, S>, T: 'static

pub fn fallback_service<T>(self, service: T) -> Self
where T: Service<Request, Error = Infallible> + Clone + Send + Sync + 'static

pub fn method_not_allowed_fallback<H, T>(self, handler: H) -> Self
where H: Handler<T, S>, T: 'static

pub fn reset_fallback(self) -> Self

pub fn with_state<S2>(self, state: S) -> Router<S2>

pub fn layer<L>(self, layer: L) -> Router<S>
where L: Layer<Route> + Clone + Send + Sync + 'static

pub fn route_layer<L>(self, layer: L) -> Self
where L: Layer<Route> + Clone + Send + Sync + 'static

pub fn without_v07_checks(self) -> Self
```

`route_service` panics for the same reasons as `route` "or if you attempt to route
to a `Router`: Use `Router::nest` instead." `with_state` consumes the router and the
state value, turning a `Router<S>` into a `Router<S2>` (in practice `Router<()>`).

---

## 2. Method-router free functions and chaining

`axum::routing` re-exports these free functions (verified from the routing module
index). Each handler-taking function has a `*_service` sibling that accepts a
`tower::Service` instead of a handler:

`get` / `get_service`, `post` / `post_service`, `put` / `put_service`,
`delete` / `delete_service`, `patch` / `patch_service`, `head` / `head_service`,
`options` / `options_service`, `connect` / `connect_service`,
`trace` / `trace_service`, `any` / `any_service`, `on` / `on_service`.

`any` matches every HTTP method. `on(filter, handler)` takes a `MethodFilter`
(a bitflag-style filter that can match one or more methods). Each function returns a
`MethodRouter`, which itself exposes the same names as chaining methods so multiple
methods register on one path:

```rust
// axum 0.8 - chaining method routers on a single path
use axum::{Router, routing::get};

let app: Router = Router::new()
    .route("/users", get(list_users).post(create_user))
    .route("/users/{id}", get(show_user).put(update_user).delete(delete_user));
```

`MethodRouter` chaining method signatures (verified): `get<H,T>(self, handler: H) ->
Self`, identically for `post`/`put`/`delete`/`patch`/`head`/`options`, plus
`on<H,T>(self, filter: MethodFilter, handler: H) -> Self` and the `*_service`
variants. Documented behavior: "Note that `get` routes will also be called for `HEAD`
requests but will have the response body removed."

---

## 3. The 405-vs-404 distinction

This is the subtlest part of the skill. There are two completely separate fallback
mechanisms:

- **Router `.fallback()`** : invoked when **no route path matches at all** (true
  `404 Not Found`).
- **`MethodRouter` default / `MethodRouter::fallback`** : invoked when the **path
  matched but the HTTP method is not registered** (`405 Method Not Allowed`).

A freshly-constructed `MethodRouter` "will respond with `405 Method Not Allowed` to
all requests" by default (verified verbatim). "By default `MethodRouter` will set the
`Allow` header when returning `405 Method Not Allowed`" (verified verbatim).

Three ways to customize:

| Method | Scope | Result |
|--------|-------|--------|
| `Router::fallback(h)` | path did not match any route | 404 |
| `MethodRouter::fallback(h)` | one specific path, wrong method | 405, that path only |
| `Router::method_not_allowed_fallback(h)` | wrong method, all method-routers | 405, router-wide |

`Router::method_not_allowed_fallback` applies the 405 handler to every method-router
registered so far, so you do not have to repeat `.fallback()` per route.

```rust
// axum 0.8 - distinct 404 and 405 handling
use axum::{Router, routing::get, http::StatusCode};

async fn not_found()        -> (StatusCode, &'static str) { (StatusCode::NOT_FOUND, "404") }
async fn wrong_method()     -> (StatusCode, &'static str) { (StatusCode::METHOD_NOT_ALLOWED, "405") }

let app: Router = Router::new()
    .route("/users", get(list_users))            // GET ok; POST -> 405
    .fallback(not_found)                          // unknown path -> 404
    .method_not_allowed_fallback(wrong_method);   // matched path, wrong method -> 405
```

---

## 4. nest, nest_service, merge

`.nest(path, router)` mounts a sub-router under a prefix. The matched prefix is
stripped from the request URI before the child router sees it, so child routes are
written relative to the mount point. `.nest_service` is the `Service` variant.
Panics: "If the route contains a wildcard (`*`). If `path` is empty."

`.merge(other)` flattens another router's routes into this one; both must share the
state type `S`. Double-fallback panic, verified verbatim: "Two routers that both have
a fallback cannot be merged. Doing so results in a panic." The fix is
`Router::reset_fallback` on one of them before merging (`reset_fallback` clears the
fallback so the merge has a single fallback to keep).

`NestedPath` is an extractor added in axum 0.8 that lets a handler inside a
`.nest()`ed router recover the prefix it was mounted under. It implements
`FromRequestParts`. Use it to build correct absolute links from inside a sub-router
that does not know its own mount point.

```rust
// axum 0.8 - nesting plus NestedPath
use axum::{Router, routing::get, extract::NestedPath};

async fn show(nested: NestedPath) -> String {
    format!("mounted under {}", nested.as_str())   // e.g. "/api/v1"
}

let api = Router::new().route("/show", get(show));
let app: Router = Router::new().nest("/api/v1", api);  // GET /api/v1/show
```

---

## 5. Path syntax: 0.7 vs 0.8

The path-syntax change is the single most disruptive 0.7 -> 0.8 break. Axum 0.8
upgraded the `matchit` routing crate to 0.8, changing capture syntax. Verified
verbatim from the 0.8.0 CHANGELOG: "Upgrade `matchit` to 0.8, changing the path
parameter syntax from `/:single` and `/*many` to `/{single}` and `/{*many}`; the old
syntax produces a panic to avoid silent change in behavior."

```rust
// axum 0.7 - colon and asterisk syntax
Router::new()
    .route("/users/:id", get(show))            // single-segment capture
    .route("/assets/*path", get(serve_file));  // catch-all wildcard

// axum 0.8 - curly-brace syntax
Router::new()
    .route("/users/{id}", get(show))           // single-segment capture
    .route("/assets/{*path}", get(serve_file));// catch-all wildcard
```

Key facts:

- A 0.7-style path string on a 0.8 router **panics at router construction**, not at
  request time. The crash is deliberate, to prevent a silent routing regression.
- `Router::without_v07_checks()` is the deliberate escape hatch: it disables the
  panic so paths beginning with `:` or `*` are treated as **literal** segments. Use
  it only when you genuinely need a literal `:` or `*` in a path, not as a migration
  shortcut.
- A literal `{` or `}` in a 0.8 path is escaped by doubling it: `{{` and `}}`.
- `/{*key}` (catch-all) does **not** match empty segments, verified verbatim: "Note
  that `/{*key}` doesn't match empty segments." It matches `/a` and `/a/b`, not `/`.

---

## 6. Route precedence and overlap panics

Verified against the `Router::route` documentation:

- **Static beats dynamic.** Verified verbatim: "The static route `/foo` and the
  dynamic route `/{key}` are not considered to overlap and `/foo` will take
  precedence." A request to `/foo` always hits the static route even when `/{key}` is
  also registered.
- **Genuine overlap panics at build time.** Verified verbatim: "Panics if the route
  overlaps with another route." Two captures at the same position genuinely overlap.
- **Empty path panics.** Verified verbatim: "Also panics if `path` is empty."
- These panics happen during router construction (startup), so an overlap or empty
  path is caught immediately, never mis-routed at request time.

---

## 7. layer vs route_layer placement rule

Both apply a `tower::Layer`, but they differ in when the middleware runs and which
routes it wraps.

- `.layer(L)` : "Apply a `tower::Layer` to **all routes** in the router." Middleware
  "added with this method will run **after** routing." Critically, it wraps only the
  routes registered **before** the `.layer()` call; routes added after are NOT
  wrapped. **Placement rule: register `.layer()` AFTER all routes it must cover.**
- `.route_layer(L)` : "Apply a `tower::Layer` to the router that will only run **if
  the request matches a route**." Verified intent: "This is useful for middleware
  that return early (such as authorization)." Because it runs only on a matched
  route, an auth layer added with `route_layer` cannot turn a genuine `404` into a
  `401`, the desired behavior for authorization guards.

```rust
// axum 0.8 - layer wraps routes registered BEFORE it; route_layer guards matches
use axum::{Router, routing::get};
use tower_http::trace::TraceLayer;

let app: Router = Router::new()
    .route("/public", get(public))
    .route("/private", get(private))
    .route_layer(auth_layer())     // runs only on matched routes (404 stays 404)
    .layer(TraceLayer::new_for_http()); // wraps everything above, runs after routing
```

Stacked `Router::layer()` calls nest bottom-to-top: the **last** `.layer()` is the
outermost. A `ServiceBuilder` stack is the opposite (top-to-bottom, first =
outermost). Mixing the two mental models is a common ordering bug.

---

## 8. Key gotchas

1. **0.7 path syntax panics on 0.8.** `/:id` or `/*path` on a 0.8 router crashes at
   construction. Convert to `{id}` / `{*path}`; reach for `without_v07_checks()` only
   for genuinely literal `:`/`*` paths.
2. **`.layer()` placed before routes.** Routes registered after the `.layer()` call
   are not wrapped. Always register `.layer()` last.
3. **Merging two routers that both have a fallback panics.** Call `reset_fallback()`
   on one before `.merge()`.
4. **404 vs 405 confusion.** A matched path with a wrong method is `405`, not `404`;
   it never reaches `Router::fallback`. Use `method_not_allowed_fallback` or
   `MethodRouter::fallback` for the 405 path.
5. **`route_service` with a `Router` panics.** Use `.nest()` for nested routers; the
   docs say so verbatim.
6. **`nest` with a wildcard or empty prefix panics.** The mount prefix must be a
   plain static or capture path, never `*`, never empty.
7. **Catch-all `/{*key}` does not match the empty segment.** It will not catch `/`.
8. **`route_layer` on an empty router.** `route_layer` is meaningful only after
   routes exist; apply it after the routes it should guard.
9. **Forgetting `with_state`.** A `Router<S>` with `S != ()` cannot be served;
   `axum::serve` needs `Router<()>`. The compiler enforces this.

---

## Sources verified (fetched 2026-05-20)

- https://docs.rs/axum/latest/axum/routing/struct.Router.html - Router method
  signatures (`new`, `route`, `route_service`, `nest`, `nest_service`, `merge`,
  `fallback`, `fallback_service`, `method_not_allowed_fallback`, `reset_fallback`,
  `with_state`, `layer`, `route_layer`, `without_v07_checks`), path syntax, route
  precedence, overlap and empty-path panics, merge double-fallback panic, nest
  wildcard panic, `layer` vs `route_layer` semantics.
- https://docs.rs/axum/latest/axum/routing/method_routing/struct.MethodRouter.html -
  MethodRouter chaining methods (`get`/`post`/`put`/`delete`/`patch`/`head`/
  `options`/`on`/`on_service`), `fallback`, `layer`, `route_layer`, `merge`,
  `with_state`; default 405 behavior, `Allow` header on 405, merge double-fallback
  panic, HEAD-via-GET note.
- https://docs.rs/axum/latest/axum/routing/index.html - free-function inventory
  (`get`/`post`/`put`/`delete`/`patch`/`head`/`options`/`connect`/`trace`/`any`/`on`
  plus `*_service` variants), exported types (`Router`, `MethodRouter`, `Route`,
  `MethodFilter`, `IntoMakeService`, `RouterAsService`, `RouterIntoService`).
- 0.7 -> 0.8 path-syntax change : verified verbatim against the axum 0.8.0 CHANGELOG
  breaking-changes section (cited and confirmed in
  `docs/research/fragments/research-a-core-routing-extractors.md`, fetched 2026-05-20).

Verification note: the docs.rs URL `axum/routing/struct.MethodRouter.html` returns
404; the correct path is `axum/routing/method_routing/struct.MethodRouter.html`.
