---
name: axum-core-router
description: >
  Use when defining routes, mounting sub-routers, or wiring middleware onto an
  Axum Router, especially when the server panics on startup with a routing
  message, returns 404 where 405 is expected, or runs middleware on only some
  routes.
  Prevents the 0.7 path-syntax startup panic, the double-fallback merge panic,
  the layer-registered-before-routes bug, and 404-versus-405 confusion.
  Covers Router<S>, .route, method-routers, .nest, .nest_service, .merge,
  .fallback, .method_not_allowed_fallback, .reset_fallback, .layer versus
  .route_layer, with_state, NestedPath, route precedence and overlap panics,
  and the 0.7 ":id" versus 0.8 "{id}" path-syntax change.
  Keywords: axum router, .route, .nest, .merge, method router, MethodRouter,
  405 Method Not Allowed, 404 Not Found, route_layer, with_state, NestedPath,
  catch-all wildcard, overlaps with another route, server panics on startup,
  routes not matching, middleware not running on some routes, how do I add a
  route, what is nest versus merge, why does my axum app crash at startup.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-core-router

`Router<S>` is the central routing type in Axum. It maps request paths to
handlers through method-routers, composes sub-routers, and is itself a
`tower::Service`. The generic `S` is the state type the router still requires;
a serveable router is `Router<()>`.

## When To Use This Skill

- Defining routes with `.route()` and method-routers
- Mounting sub-routers with `.nest()` or flattening them with `.merge()`
- Wiring middleware with `.layer()` or `.route_layer()`
- The server panics at startup with a routing or path message
- A request returns `404` where `405` is expected, or the reverse
- Middleware runs on some routes but not others

## Quick Reference

### Router methods (axum 0.7 and 0.8 share these signatures)

| Method | Purpose |
|--------|---------|
| `Router::new()` | Empty router, answers `404` to every request |
| `.route(path, mr)` | Register one path to a `MethodRouter` |
| `.route_service(path, svc)` | Register a path to a raw `tower::Service` |
| `.nest(path, router)` | Mount a sub-router under a prefix (prefix stripped) |
| `.nest_service(path, svc)` | Mount a `tower::Service` under a prefix |
| `.merge(other)` | Flatten another router's routes into this one |
| `.fallback(handler)` | Handler for unmatched paths (true `404`) |
| `.fallback_service(svc)` | Service variant of `.fallback` |
| `.method_not_allowed_fallback(h)` | Router-wide `405` handler |
| `.reset_fallback()` | Clear the fallback, required before some `.merge()` calls |
| `.with_state(state)` | Provide state, turning `Router<S>` into `Router<()>` |
| `.layer(L)` | Wrap all routes registered SO FAR in middleware |
| `.route_layer(L)` | Wrap routes, but run only when a route matches |
| `.without_v07_checks()` | Disable the 0.7-path-syntax startup panic |

Full signatures, where-clauses, and documented panics: `references/methods.md`.

### Method-router functions (`axum::routing`)

`get`, `post`, `put`, `delete`, `patch`, `head`, `options`, `connect`, `trace`,
`any` (matches every method), and `on(filter, handler)` (takes a `MethodFilter`).
Each function has a `*_service` sibling (`get_service`, `post_service`, ...) that
accepts a `tower::Service` instead of a handler. Each returns a `MethodRouter`;
the same names chain on a `MethodRouter` to register several methods on one path.

### Path parameter syntax

| Capture | axum 0.7 | axum 0.8 |
|---------|----------|----------|
| Single segment | `/users/:id` | `/users/{id}` |
| Catch-all wildcard | `/assets/*path` | `/assets/{*path}` |

ALWAYS match the path syntax to the target Axum version. The 0.7 syntax on a 0.8
router panics at router construction (startup), never at request time. This is
deliberate, to prevent a silent routing regression. NEVER expect `/:id` to be
treated as a literal segment on 0.8.

### 404 versus 405

| Situation | Response | Customize with |
|-----------|----------|----------------|
| No route path matches at all | `404 Not Found` | `Router::fallback` |
| Path matched, method not registered | `405 Method Not Allowed` | `MethodRouter::fallback` (one path) or `Router::method_not_allowed_fallback` (router-wide) |

A path that matched with an unsupported method NEVER reaches `Router::fallback`.

## Decision Trees

### Choosing the composition method

```
Need to add request handling to a Router?
├── A single path to a handler?
│   └── .route(path, get(handler)...)
├── A single path to a raw tower::Service (not a handler, not a Router)?
│   └── .route_service(path, service)
├── A whole sub-Router under a URL prefix (prefix stripped from child)?
│   └── .nest(prefix, sub_router)
├── A tower::Service mounted under a URL prefix?
│   └── .nest_service(prefix, service)   // e.g. ServeDir
└── Another Router's routes flattened in at the SAME level (no prefix)?
    └── .merge(other_router)
```

NEVER pass a `Router` to `.route_service()`; it panics. Use `.nest()` for a
nested `Router`. `.nest()` adds a prefix; `.merge()` does not.

### `.layer()` versus `.route_layer()`

```
Applying a tower::Layer to a Router?
├── Middleware must run on EVERY request, including unmatched paths?
│   │   (logging, tracing, CORS, compression, timeout)
│   └── .layer(L)        // place AFTER all routes it must cover
└── Middleware must run ONLY when a route matched, and a genuine 404
    must stay 404 (not become 401/403)?
    │   (authentication and authorization guards)
    └── .route_layer(L)  // place AFTER the routes it guards
```

`.layer()` wraps only the routes registered BEFORE the call. ALWAYS register
`.layer()` after every route it must cover. `.route_layer()` runs only on a
matched route, so an auth layer added with it can NEVER turn a real `404` into a
`401`. `.route_layer()` panics if no routes have been added yet.

### Path syntax for the target version

```
Writing a route path string?
├── Targeting axum 0.8?
│   ├── Single capture  -> "/users/{id}"
│   └── Catch-all       -> "/assets/{*path}"
├── Targeting axum 0.7?
│   ├── Single capture  -> "/users/:id"
│   └── Catch-all       -> "/assets/*path"
└── Need a LITERAL ':' or '*' segment on axum 0.8?
    └── .without_v07_checks()   // escape hatch ONLY, never a migration shortcut
```

## Patterns

### Pattern: basic router and method chaining

```rust
// axum 0.8 - method routers chained on one path
use axum::{Router, routing::get};

let app: Router = Router::new()
    .route("/", get(root))
    .route("/users", get(list_users).post(create_user))
    .route("/users/{id}", get(show_user).put(update_user).delete(delete_user));

// axum 0.7 - identical, except the path-capture syntax
let app: Router = Router::new()
    .route("/users/:id", get(show_user).put(update_user));
```

Chaining several method-routers on one path registers multiple HTTP methods.
A `get` route also answers `HEAD` requests, with the response body removed.

### Pattern: nesting and recovering the mount prefix

```rust
// axum 0.8 - nest strips the matched prefix from the child's view
use axum::{Router, routing::get, extract::NestedPath};

async fn show(nested: NestedPath) -> String {
    // NestedPath recovers the prefix this router was mounted under
    format!("mounted under {}", nested.as_str())  // "/api/v1"
}

let api = Router::new().route("/show", get(show));   // child path is relative
let app: Router = Router::new().nest("/api/v1", api); // serves GET /api/v1/show
```

`.nest()` strips the matched prefix before the child router sees the request, so
child routes are written relative to the mount point. ALWAYS use the `NestedPath`
extractor (axum 0.8) when a nested handler must build an absolute URL, because the
child does not otherwise know its own prefix. `.nest()` panics if the prefix is
empty or contains a wildcard.

### Pattern: merging routers and the double-fallback fix

```rust
// axum 0.8 - merge flattens routes at the same level (no prefix added)
let users  = Router::new().route("/users", get(list_users));
let posts  = Router::new().route("/posts", get(list_posts));
let app: Router = Router::new().merge(users).merge(posts);

// axum 0.8 - two routers that BOTH set a fallback cannot merge directly
let a = Router::new().route("/a", get(ha)).fallback(fallback_a);
let b = Router::new().route("/b", get(hb)).fallback(fallback_b);
let app: Router = a.merge(b.reset_fallback()); // clear one fallback first
```

`.merge()` requires both routers to share the state type `S`. Merging two routers
that each define a fallback panics, because a `Router` allows only one fallback.
ALWAYS call `.reset_fallback()` on one router before merging when both have one.

### Pattern: distinct 404 and 405 handling

```rust
// axum 0.8 - separate handlers for unmatched path and wrong method
use axum::{Router, routing::get, http::StatusCode};

async fn not_found()    -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "no such resource")
}
async fn wrong_method() -> (StatusCode, &'static str) {
    (StatusCode::METHOD_NOT_ALLOWED, "method not allowed")
}

let app: Router = Router::new()
    .route("/users", get(list_users))         // GET ok; POST /users -> 405
    .fallback(not_found)                      // unknown path -> 404
    .method_not_allowed_fallback(wrong_method); // matched path, wrong method -> 405
```

A freshly built `MethodRouter` responds `405 Method Not Allowed` by default and
sets the `Allow` header. `Router::method_not_allowed_fallback` applies one `405`
handler across every method-router registered so far. NEVER expect a wrong-method
request to reach `Router::fallback`; it produces a `405`, not a `404`.

### Pattern: layer placement

```rust
// axum 0.8 - route_layer guards matched routes; layer wraps everything above
use axum::{Router, routing::get};
use tower_http::trace::TraceLayer;

let app: Router = Router::new()
    .route("/public", get(public))
    .route("/private", get(private))
    .route_layer(require_auth())          // runs only on a matched route
    .layer(TraceLayer::new_for_http());   // wraps both routes, runs after routing
```

Stacked `Router::layer()` calls nest bottom-to-top: the LAST `.layer()` is the
outermost. A `tower::ServiceBuilder` stack is the opposite (first = outermost).
NEVER mix the two mental models; see `axum-impl-middleware` for the full ordering
rule.

### Pattern: serving the router with state

```rust
// axum 0.8 and 0.7 - identical
#[derive(Clone)]
struct AppState { /* db pool, config */ }

let app: Router = Router::new()
    .route("/", get(handler))
    .with_state(AppState { /* ... */ }); // Router<AppState> -> Router<()>

let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, app).await.unwrap();
```

`axum::serve` accepts only a `Router<()>`. A `Router<S>` with `S != ()` does not
type-check against `serve`, so a forgotten `.with_state()` is a compile error,
never a runtime surprise. State details: see `axum-core-state`.

### Pattern: route precedence and catch-all

```rust
// axum 0.8 - a static route always beats a dynamic capture
let app: Router = Router::new()
    .route("/users/me", get(current_user))   // GET /users/me hits THIS
    .route("/users/{id}", get(show_user))    // GET /users/42 hits this
    .route("/assets/{*path}", get(serve));   // catch-all: /assets/a, /assets/a/b
```

Static segments take precedence over dynamic captures; `/users/me` and
`/users/{id}` do not overlap. A catch-all `/{*path}` does NOT match the empty
segment, so `/assets/{*path}` matches `/assets/a` but not `/assets`. Two routes
that genuinely overlap (two captures at the same position) panic at router
construction. An empty path string panics.

## Common Mistakes

- 0.7 path syntax (`/:id`, `/*path`) on a 0.8 router: panics at startup. Convert
  every path to `{id}` / `{*path}`. See `references/anti-patterns.md`.
- `.layer()` placed before some routes: those later routes are not wrapped.
- Merging two routers that both set a fallback: panics. Call `.reset_fallback()`.
- Expecting a wrong-method request to return `404`: it returns `405`.
- Passing a `Router` to `.route_service()`: panics. Use `.nest()`.

## Reference Links

- `references/methods.md` : complete `Router` and `MethodRouter` API signatures,
  where-clauses, and every documented panic.
- `references/examples.md` : full working, version-annotated routing examples.
- `references/anti-patterns.md` : real routing mistakes with the WHY behind each
  failure and the fix.

## Related Skills

- `axum-core-version-migration` : the full 0.7 to 0.8 breaking-change matrix,
  including the path-syntax change.
- `axum-impl-middleware` : `.layer()` versus `.route_layer()` ordering in depth.
- `axum-core-state` : `with_state`, the `State<S>` extractor, and `FromRef`.
