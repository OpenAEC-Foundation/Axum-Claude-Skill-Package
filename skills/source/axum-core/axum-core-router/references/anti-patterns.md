# axum-core-router : Anti-Patterns

Real routing mistakes, each with the symptom, the root cause, and the fix. These
are observed in the Axum issue tracker, discussions, and the framework's own
documentation warnings. APIs verified against docs.rs on 2026-05-20.

## AP-1 : 0.7 path syntax on a 0.8 router

```rust
// axum 0.8 - WRONG: 0.7 colon and asterisk syntax
let app: Router = Router::new()
    .route("/users/:id", get(show))        // panics at startup
    .route("/files/*path", get(serve));    // panics at startup
```

Symptom: the server panics at startup, during router construction, before it ever
binds a port.

Root cause: axum 0.8 upgraded the `matchit` routing crate to 0.8, changing the
capture syntax from `/:single` and `/*many` to `/{single}` and `/{*many}`. The
maintainers deliberately made the old syntax panic rather than silently treating
`:id` as a literal segment, to prevent a silent routing regression after upgrade.

Fix: convert every path string to curly-brace syntax.

```rust
// axum 0.8 - CORRECT
let app: Router = Router::new()
    .route("/users/{id}", get(show))
    .route("/files/{*path}", get(serve));
```

NEVER reach for `Router::without_v07_checks()` as a migration shortcut. That
escape hatch exists only for paths that genuinely need a literal `:` or `*`
segment, and it makes `:id` a literal, not a capture.

## AP-2 : `.layer()` registered before the routes it should cover

```rust
// axum 0.8 - WRONG: /late is not wrapped by the trace layer
use tower_http::trace::TraceLayer;

let app: Router = Router::new()
    .route("/early", get(early))
    .layer(TraceLayer::new_for_http())   // wraps ONLY /early
    .route("/late", get(late));          // NOT traced
```

Symptom: middleware (tracing, CORS, compression, timeout) runs on some routes but
silently does nothing on others.

Root cause: `.layer()` wraps only the routes registered BEFORE the call. Routes
added after it are outside the layer.

Fix: ALWAYS register `.layer()` after every route it must cover.

```rust
// axum 0.8 - CORRECT: layer added last
let app: Router = Router::new()
    .route("/early", get(early))
    .route("/late", get(late))
    .layer(TraceLayer::new_for_http());  // wraps both routes
```

## AP-3 : merging two routers that both set a fallback

```rust
// axum 0.8 - WRONG: both routers have a fallback
let a = Router::new().route("/a", get(ha)).fallback(fallback_a);
let b = Router::new().route("/b", get(hb)).fallback(fallback_b);
let app: Router = a.merge(b);             // panics
```

Symptom: the server panics at startup when `.merge()` runs.

Root cause: a `Router` allows exactly one fallback. Merging two routers that each
define one would leave the merged router with an ambiguous pair.

Fix: call `.reset_fallback()` on one router before merging.

```rust
// axum 0.8 - CORRECT
let app: Router = a.merge(b.reset_fallback());
```

## AP-4 : expecting a wrong-method request to return 404

```rust
// axum 0.8 - WRONG mental model
let app: Router = Router::new()
    .route("/users", get(list_users))   // only GET registered
    .fallback(handle_404);              // POST /users does NOT reach this
```

Symptom: a `POST` to a `GET`-only path returns `405 Method Not Allowed`, and the
custom `fallback` handler is never invoked for it.

Root cause: there are two separate fallback mechanisms. `Router::fallback` handles
a path that matched NO route (a true `404`). A path that matched but with an
unregistered method is handled by the `MethodRouter`, which returns `405` by
default. The wrong-method request never reaches `Router::fallback`.

Fix: handle `405` explicitly with `method_not_allowed_fallback` (router-wide) or
`MethodRouter::fallback` (one path).

```rust
// axum 0.8 - CORRECT
let app: Router = Router::new()
    .route("/users", get(list_users))
    .fallback(handle_404)                      // unmatched path -> 404
    .method_not_allowed_fallback(handle_405);  // wrong method   -> 405
```

## AP-5 : passing a `Router` to `.route_service()`

```rust
// axum 0.8 - WRONG: route_service cannot take a Router
let sub = Router::new().route("/x", get(handler));
let app: Router = Router::new()
    .route_service("/sub", sub);     // panics
```

Symptom: the server panics at startup.

Root cause: `route_service` expects a `tower::Service`, and although a `Router`
is a `Service`, routing a path directly to a `Router` is explicitly rejected; the
docs direct you to `nest` instead.

Fix: mount a nested `Router` with `.nest()`.

```rust
// axum 0.8 - CORRECT
let app: Router = Router::new().nest("/sub", sub);
```

## AP-6 : nesting under an empty or wildcard prefix

```rust
// axum 0.8 - WRONG: nest prefix may not be empty or contain a wildcard
let app: Router = Router::new()
    .nest("", api_routes())               // panics: empty prefix
    .nest("/files/{*rest}", file_routes()); // panics: wildcard prefix
```

Symptom: the server panics at startup.

Root cause: `.nest()` strips a fixed prefix from the request URI. An empty prefix
or a wildcard prefix has no well-defined fixed length to strip.

Fix: nest under a plain static or single-capture prefix. To merge routes without
adding a prefix, use `.merge()` instead of `.nest("", ...)`.

## AP-7 : expecting a catch-all to match the empty segment

```rust
// axum 0.8 - WRONG assumption
let app: Router = Router::new()
    .route("/files/{*path}", get(serve));
// GET /files  -> does NOT match this route, falls through to 404
```

Symptom: a request to the bare prefix (`/files`) returns `404` even though a
catch-all route is registered.

Root cause: a catch-all `/{*key}` does not match the empty segment. It matches
`/files/a` and `/files/a/b`, never `/files`.

Fix: register the bare path separately when it must also be handled.

```rust
// axum 0.8 - CORRECT
let app: Router = Router::new()
    .route("/files", get(list_root))
    .route("/files/{*path}", get(serve));
```

## AP-8 : registering genuinely overlapping routes

```rust
// axum 0.8 - WRONG: two captures at the same position overlap
let app: Router = Router::new()
    .route("/items/{id}", get(by_id))
    .route("/items/{slug}", get(by_slug));  // panics: overlaps
```

Symptom: the server panics at startup with an overlap message.

Root cause: two dynamic captures at the same path position are genuinely
ambiguous; Axum cannot decide which to match. A static segment and a capture do
NOT overlap (the static route wins), but two captures do.

Fix: disambiguate with a distinct static prefix or a different path shape.

```rust
// axum 0.8 - CORRECT
let app: Router = Router::new()
    .route("/items/by-id/{id}", get(by_id))
    .route("/items/by-slug/{slug}", get(by_slug));
```

## AP-9 : `.route_layer()` applied before any route exists

```rust
// axum 0.8 - WRONG: no routes registered yet
let app: Router = Router::new()
    .route_layer(require_auth())          // panics: no routes
    .route("/private", get(private));
```

Symptom: the server panics at startup.

Root cause: `.route_layer()` wraps already-registered routes; it is meaningless
on an empty router.

Fix: register the routes first, then apply `.route_layer()` after them.

```rust
// axum 0.8 - CORRECT
let app: Router = Router::new()
    .route("/private", get(private))
    .route_layer(require_auth());
```

## AP-10 : using `.route_layer()` for middleware that must always run

```rust
// axum 0.8 - WRONG: tracing skipped for unmatched paths
let app: Router = Router::new()
    .route("/page", get(page))
    .route_layer(TraceLayer::new_for_http());  // no trace for 404s
```

Symptom: requests to unmatched paths produce no trace, no log, no CORS headers.

Root cause: `.route_layer()` runs ONLY when a request matches a route. That is
correct for an auth guard (a real `404` must stay `404`, never become `401`), but
wrong for observability and CORS, which must also cover unmatched requests.

Fix: use `.layer()` for always-run middleware; reserve `.route_layer()` for
early-returning guards such as authentication and authorization.

```rust
// axum 0.8 - CORRECT
let app: Router = Router::new()
    .route("/page", get(page))
    .route_layer(require_auth())               // guard: matched routes only
    .layer(TraceLayer::new_for_http());        // observability: every request
```
