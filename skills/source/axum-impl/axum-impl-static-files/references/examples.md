# axum-impl-static-files : Working Examples

Every example is verified against the official `tower-http` docs and the axum
`static-file-server` example (verified 2026-05-20). Static file serving is
identical on Axum 0.7 and 0.8, so every snippet carries the
`// axum 0.7 / 0.8 - identical` annotation.

## Cargo.toml

```toml
[dependencies]
axum = "0.8"                 # or "0.7"; the static-file code is the same
tokio = { version = "1", features = ["full"] }
# fs covers ServeDir and ServeFile; set-status covers SetStatus (SPA fallback)
tower-http = { version = "0.6", features = ["fs", "set-status"] }
```

## Serve a directory under a URL prefix

```rust
// axum 0.7 / 0.8 - identical
use axum::Router;
use tower_http::services::ServeDir;

fn using_serve_dir() -> Router {
    // GET /assets/app.css  ->  disk file  assets/app.css
    Router::new().nest_service("/assets", ServeDir::new("assets"))
}
```

## Serve multiple directories under separate prefixes

```rust
// axum 0.7 / 0.8 - identical
use axum::Router;
use tower_http::services::ServeDir;

fn two_serve_dirs() -> Router {
    Router::new()
        .nest_service("/assets", ServeDir::new("assets"))
        .nest_service("/dist", ServeDir::new("dist"))
}
```

## Serve a directory from the site root via the router fallback

```rust
// axum 0.7 / 0.8 - identical
use axum::{routing::get, Router};
use tower_http::services::{ServeDir, ServeFile};

fn using_serve_dir_only_from_root_via_fallback() -> Router {
    let serve_dir = ServeDir::new("assets")
        .not_found_service(ServeFile::new("assets/index.html"));

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}
```

## Serve a single file on one exact route

```rust
// axum 0.7 / 0.8 - identical
use axum::Router;
use tower_http::services::ServeFile;

fn using_serve_file_from_a_route() -> Router {
    Router::new().route_service("/foo", ServeFile::new("assets/index.html"))
}
```

## Serve a single file with an explicit MIME type

```rust
// axum 0.7 / 0.8 - identical
use axum::Router;
use tower_http::services::ServeFile;

fn serve_with_mime() -> Router {
    // new_with_mime panics if the MIME is not a valid header value.
    let svc = ServeFile::new_with_mime("data/feed.xml", &mime::TEXT_XML);
    Router::new().route_service("/feed.xml", svc)
}
```

## A custom 404 handler as the not_found_service

```rust
// axum 0.7 / 0.8 - identical
use axum::{
    handler::HandlerWithoutStateExt, http::StatusCode, routing::get, Router,
};
use tower_http::services::ServeDir;

fn using_serve_dir_with_handler_as_service() -> Router {
    async fn handle_404() -> (StatusCode, &'static str) {
        (StatusCode::NOT_FOUND, "Not found")
    }
    // into_service() converts a stateless handler into a tower Service.
    let service = handle_404.into_service();
    let serve_dir = ServeDir::new("assets").not_found_service(service);

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}
```

## The SPA fallback pattern (index.html with a 404 status)

```rust
// axum 0.7 / 0.8 - identical. tower-http features: ["fs", "set-status"]
use axum::{routing::get, Router, http::StatusCode};
use tower_http::services::{ServeDir, ServeFile};
use tower_http::set_status::SetStatus;

fn using_serve_dir_with_assets_fallback() -> Router {
    // A bare ServeFile returns 200. SetStatus rewrites it to 404 while
    // keeping the index.html body, which is the correct SPA semantics.
    let index_html = SetStatus::new(
        ServeFile::new("assets/index.html"),
        StatusCode::NOT_FOUND,
    );

    let serve_dir = ServeDir::new("assets")
        .not_found_service(index_html.clone());

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .nest_service("/assets", serve_dir)
        .fallback_service(index_html)
}
```

`index_html` is `Clone`, so the same service instance is used in two places:
`ServeDir::not_found_service` (a missing file under `/assets`) and the router's
`fallback_service` (any other unmatched path).

## Precompressed assets

```rust
// axum 0.7 / 0.8 - identical
use axum::Router;
use tower_http::services::ServeDir;

fn precompressed() -> Router {
    // Serves assets/app.js.br when the client sends Accept-Encoding: br,
    // assets/app.js.gz when it sends gzip, otherwise the plain file.
    let serve_dir = ServeDir::new("assets")
        .precompressed_br()
        .precompressed_gzip();

    Router::new().nest_service("/assets", serve_dir)
}
```

## Full runnable program with the SPA fallback

```rust
// axum 0.8 - axum::serve is the 0.8 entry point
use axum::{routing::get, Router, http::StatusCode};
use tower_http::services::{ServeDir, ServeFile};
use tower_http::set_status::SetStatus;

#[tokio::main]
async fn main() {
    let index_html = SetStatus::new(
        ServeFile::new("assets/index.html"),
        StatusCode::NOT_FOUND,
    );

    let serve_dir = ServeDir::new("assets")
        .not_found_service(index_html.clone());

    let app: Router = Router::new()
        .route("/api/health", get(|| async { "ok" }))
        .nest_service("/assets", serve_dir)
        .fallback_service(index_html);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .expect("port 3000 is free");
    axum::serve(listener, app).await.expect("server runs");
}
```

```rust
// axum 0.7 - the only difference is the serve entry point:
//   axum::Server::bind(&addr).serve(app.into_make_service()).await
// The ServeDir / ServeFile / SetStatus code above is unchanged.
```
