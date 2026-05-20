# axum-core-router : Examples

Working, version-annotated routing examples. Every code block is annotated
`// axum 0.8` or `// axum 0.7` where the two versions diverge. APIs verified
against docs.rs on 2026-05-20 (see `methods.md` for source URLs).

## Example: a complete minimal server

```rust
// axum 0.8 - full runnable entry point
use axum::{Router, routing::get};

async fn root() -> &'static str {
    "hello"
}

#[tokio::main]
async fn main() {
    let app: Router = Router::new().route("/", get(root));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

`axum::serve` accepts only a `Router<()>`. This router never calls `.with_state()`,
so its type is already `Router<()>`.

## Example: method-router chaining for a REST resource

```rust
// axum 0.8 - several HTTP methods per path
use axum::{Router, routing::get};

let app: Router = Router::new()
    .route("/users", get(list_users).post(create_user))
    .route("/users/{id}", get(show_user)
        .put(replace_user)
        .patch(update_user)
        .delete(delete_user));
```

```rust
// axum 0.7 - identical, only the capture syntax differs
let app: Router = Router::new()
    .route("/users", get(list_users).post(create_user))
    .route("/users/:id", get(show_user)
        .put(replace_user)
        .delete(delete_user));
```

## Example: matching every method, and a method filter

```rust
// axum 0.8 - `any` matches every method; `on` matches a chosen set
use axum::{Router, routing::{any, on, MethodFilter}};

async fn catch_all_methods() -> &'static str { "any method" }
async fn get_or_post() -> &'static str { "GET or POST" }

let app: Router = Router::new()
    .route("/health", any(catch_all_methods))
    .route("/submit", on(MethodFilter::GET.or(MethodFilter::POST), get_or_post));
```

## Example: nesting sub-routers

```rust
// axum 0.8 - the matched prefix is stripped from the child's view
use axum::{Router, routing::get};

fn user_routes() -> Router {
    // child paths are written relative to the mount point
    Router::new()
        .route("/", get(list_users))         // becomes /api/users
        .route("/{id}", get(show_user))      // becomes /api/users/{id}
}

let app: Router = Router::new()
    .nest("/api/users", user_routes());
```

## Example: NestedPath to recover the mount prefix

```rust
// axum 0.8 - a nested handler that does not hardcode its own prefix
use axum::{Router, routing::get, extract::NestedPath};

async fn build_link(nested: NestedPath) -> String {
    // nested.as_str() yields the prefix this router was mounted under
    format!("{}/next-page", nested.as_str())
}

let pages = Router::new().route("/current", get(build_link));
let app: Router = Router::new().nest("/docs/v2", pages);
// GET /docs/v2/current responds with "/docs/v2/next-page"
```

`NestedPath` was added in axum 0.8. Use it whenever a nested handler must produce
an absolute URL, instead of duplicating the prefix string.

## Example: merging routers

```rust
// axum 0.8 - merge flattens routes at the same level, no prefix added
use axum::{Router, routing::get};

fn health_routes() -> Router {
    Router::new()
        .route("/healthz", get(|| async { "ok" }))
        .route("/readyz", get(|| async { "ready" }))
}

fn api_routes() -> Router {
    Router::new().route("/api/version", get(|| async { "1.0" }))
}

let app: Router = Router::new()
    .merge(health_routes())
    .merge(api_routes());
```

## Example: merging when both routers have a fallback

```rust
// axum 0.8 - reset_fallback prevents the double-fallback panic
use axum::{Router, routing::get, http::StatusCode};

async fn site_404() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "page not found")
}
async fn api_404() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "unknown endpoint")
}

let site = Router::new()
    .route("/", get(home))
    .fallback(site_404);

let api = Router::new()
    .route("/api/data", get(data))
    .fallback(api_404);

// clear one fallback so the merged router keeps exactly one
let app: Router = site.merge(api.reset_fallback());
```

## Example: distinct 404 and 405 handlers

```rust
// axum 0.8 - separate handling for unmatched path and wrong method
use axum::{Router, routing::get, http::StatusCode};

async fn handle_404() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "no such path")
}
async fn handle_405() -> (StatusCode, &'static str) {
    (StatusCode::METHOD_NOT_ALLOWED, "method not allowed here")
}

let app: Router = Router::new()
    .route("/articles", get(list_articles))   // POST /articles -> 405
    .fallback(handle_404)                      // GET /missing  -> 404
    .method_not_allowed_fallback(handle_405);  // wrong method  -> 405
```

To customize the `405` for a single path only, set it on the `MethodRouter`:

```rust
// axum 0.8 - per-path 405 customization
use axum::{Router, routing::get, http::StatusCode};

let one_path = get(list_articles)
    .fallback(|| async { (StatusCode::METHOD_NOT_ALLOWED, "only GET here") });

let app: Router = Router::new().route("/articles", one_path);
```

## Example: layer placement and route_layer

```rust
// axum 0.8 - route_layer guards matched routes; layer wraps all routes above it
use axum::{Router, routing::get};
use tower_http::trace::TraceLayer;
use tower_http::compression::CompressionLayer;

let app: Router = Router::new()
    .route("/public", get(public_page))
    .route("/private", get(private_page))
    .route_layer(require_auth())            // runs ONLY on a matched route
    .layer(CompressionLayer::new())         // wraps the two routes above
    .layer(TraceLayer::new_for_http());     // outermost: last layer wins
```

`TraceLayer` is the outermost layer here because stacked `.layer()` calls nest
bottom-to-top. A route added AFTER these `.layer()` calls would not be wrapped by
them.

## Example: route precedence and a catch-all wildcard

```rust
// axum 0.8 - static beats dynamic; catch-all does not match the empty segment
use axum::{Router, routing::get};

let app: Router = Router::new()
    .route("/users/me", get(current_user))    // GET /users/me  -> current_user
    .route("/users/{id}", get(show_user))     // GET /users/42  -> show_user
    .route("/assets/{*path}", get(serve_asset)); // /assets/a, /assets/a/b
// GET /assets (no trailing segment) does NOT match the catch-all route
```

```rust
// axum 0.7 - same precedence rules, colon and asterisk syntax
let app: Router = Router::new()
    .route("/users/me", get(current_user))
    .route("/users/:id", get(show_user))
    .route("/assets/*path", get(serve_asset));
```

## Example: serving a router that needs state

```rust
// axum 0.8 and 0.7 - identical
use axum::{Router, routing::get, extract::State};

#[derive(Clone)]
struct AppState {
    greeting: String,
}

async fn greet(State(state): State<AppState>) -> String {
    state.greeting.clone()
}

#[tokio::main]
async fn main() {
    let app: Router = Router::new()
        .route("/greet", get(greet))
        .with_state(AppState { greeting: "hi".to_string() }); // -> Router<()>

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## Example: mounting a tower::Service with nest_service

```rust
// axum 0.8 - serve static files under a prefix with ServeDir
use axum::{Router, routing::get};
use tower_http::services::ServeDir;

let app: Router = Router::new()
    .route("/", get(home))
    .nest_service("/static", ServeDir::new("public"));
// GET /static/style.css serves public/style.css
```

`ServeDir` is a `tower::Service`, not a handler, so it is mounted with
`nest_service` (or `route_service`), never with `nest`.
