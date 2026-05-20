---
name: axum-impl-static-files
description: >
  Use when an Axum app must serve static files: a CSS/JS/image assets
  directory, a single file on one route, or a single-page-app build where any
  unmatched path falls back to index.html.
  Prevents the trait-bound compile error from passing ServeDir to .layer()
  instead of mounting it as a route service, prevents the .fallback() versus
  .fallback_service() mix-up, prevents repeating the URL prefix inside the disk
  path, and prevents an SPA fallback that returns 200 for unknown URLs.
  Covers the tower-http fs and set-status feature flags, ServeDir and ServeFile
  as tower Services, .nest_service / .fallback_service / .route_service mounts,
  .not_found_service versus .fallback, SetStatus, and the canonical SPA
  fallback pattern.
  Keywords: axum static files, ServeDir, ServeFile, tower-http, nest_service,
  fallback_service, route_service, not_found_service, SetStatus, fs feature,
  set-status feature, serve a directory, serve index.html, SPA fallback,
  single page app routing, ServeDir is not a Layer, the trait bound Layer is
  not satisfied, unresolved import tower_http services, my assets return 404,
  static files not found, 404 instead of index.html, how do I serve CSS, how
  do I host a React build, how do I serve a folder.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-impl-static-files

## Overview

Axum itself has no static file API. Static file serving is provided by the
`tower-http` crate through two types: `ServeDir` (serves a directory) and
`ServeFile` (serves one file). Both are `tower::Service` implementations, NOT
`tower::Layer` implementations.

This single fact governs the whole topic: a `Service` is a route TARGET and is
mounted with `.nest_service()`, `.fallback_service()`, or `.route_service()`. It
is NEVER passed to `.layer()`. The confusion is common because other `tower-http`
items (`TraceLayer`, `CompressionLayer`) ARE layers.

Static file serving is identical between Axum 0.7 and 0.8, because the code lives
in `tower-http`, not in Axum. There are no version-divergent snippets in this
skill.

## Quick Reference

| Type | Purpose | Mount with |
|------|---------|------------|
| `ServeDir::new(dir)` | serve a whole directory | `.nest_service()` or `.fallback_service()` |
| `ServeFile::new(file)` | serve one file | `.route_service()` |
| `SetStatus::new(svc, code)` | force a fixed status on a Service | wraps a `ServeFile` for SPA |

| Mount method | Takes | Prefix handling |
|--------------|-------|-----------------|
| `.nest_service(path, svc)` | a `Service` | strips the matched prefix before the service sees the URI |
| `.fallback_service(svc)` | a `Service` | no stripping; service sees the full path |
| `.route_service(path, svc)` | a `Service` | exact-match route |
| `.fallback(handler)` | a `Handler` | NOT for `ServeDir`; takes a handler function |
| `.layer(layer)` | a `Layer` | NEVER for `ServeDir`; a compile error |

Cargo features (in `tower-http`):

```toml
# directory or single-file serving
tower-http = { version = "0.6", features = ["fs"] }
# add set-status when you build the SPA fallback
tower-http = { version = "0.6", features = ["fs", "set-status"] }
```

Core rules:

- ALWAYS mount `ServeDir` and `ServeFile` as route services
  (`.nest_service` / `.fallback_service` / `.route_service`).
- NEVER pass `ServeDir` or `ServeFile` to `.layer()`. They are Services.
- ALWAYS use `.fallback_service()` for a Service; `.fallback()` takes a Handler.
- NEVER repeat the URL prefix in the disk path. `.nest_service("/assets", ...)`
  strips `/assets`, so the disk path is the bare directory.
- ALWAYS enable feature `fs`; add `set-status` for the SPA fallback.
- `ServeDir` is infallible: a missing file yields `404`, never an `Err`.

## Decision Trees

### Which type and mount for the job

```
What are you serving?
  a whole directory of assets under a URL prefix (/assets/...)
    -> ServeDir::new("assets")  mounted with  .nest_service("/assets", _)
  a whole directory served from the site root
    -> ServeDir::new("assets")  mounted with  .fallback_service(_)
  exactly one file on one exact route
    -> ServeFile::new("path/to/file")  mounted with  .route_service("/path", _)
  a single-page-app build (unknown paths must load index.html)
    -> see the SPA fallback pattern below (ServeDir + SetStatus)
```

### What happens when a file is missing under ServeDir

```
A request under ServeDir does not resolve to a file. What should happen?
  return a plain 404
    -> default behavior, no extra configuration
  serve a custom page or file, KEEP its own status code
    -> .fallback(service)            returns ServeDir<F2>
  serve a custom file but FORCE the status to 404
    -> .not_found_service(service)   returns ServeDir<SetStatus<F2>>
```

`.not_found_service()` automatically wraps the given service in `SetStatus` and
forces `404`. `.fallback()` does NOT wrap: the fallback's own status is used.

## Patterns

### Pattern: serve a directory under a URL prefix

```rust
// axum 0.7 / 0.8 - identical
use axum::Router;
use tower_http::services::ServeDir;

fn app() -> Router {
    // A request for /assets/app.css resolves to the disk file assets/app.css.
    Router::new().nest_service("/assets", ServeDir::new("assets"))
}
```

`.nest_service("/assets", ...)` strips the `/assets` prefix before `ServeDir`
sees the URI. The path passed to `ServeDir::new` is therefore the bare directory
`assets`, NOT `assets/assets`.

### Pattern: serve a directory from the site root

```rust
// axum 0.7 / 0.8 - identical
use axum::{routing::get, Router};
use tower_http::services::{ServeDir, ServeFile};

fn app() -> Router {
    let serve_dir = ServeDir::new("assets")
        .not_found_service(ServeFile::new("assets/index.html"));

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}
```

`.fallback_service()` receives the full path (no prefix stripping), so every
path not matched by an explicit route is handed to `ServeDir`.

### Pattern: serve a single file on one route

```rust
// axum 0.7 / 0.8 - identical
use axum::Router;
use tower_http::services::ServeFile;

fn app() -> Router {
    Router::new().route_service("/favicon.ico", ServeFile::new("assets/favicon.ico"))
}
```

`ServeFile::new` guesses the `Content-Type` from the file extension. Use
`ServeFile::new_with_mime(path, &mime)` to set it explicitly; that constructor
panics if the MIME string is not a valid header value.

### Pattern: the SPA fallback (return index.html with a 404 status)

A single-page app needs every unmatched URL to load `index.html` so the
client-side router can render the right view. The correct HTTP semantics are to
return the `index.html` body WITH a `404` status, because the server genuinely
has no resource at that URL.

```rust
// axum 0.7 / 0.8 - identical. tower-http features: ["fs", "set-status"]
use axum::{routing::get, Router, http::StatusCode};
use tower_http::services::{ServeDir, ServeFile};
use tower_http::set_status::SetStatus;

fn app() -> Router {
    // SetStatus rewrites the 200 from ServeFile into a 404, keeping the body.
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

The same `index_html` service is used twice: as `ServeDir`'s
`not_found_service` (a missing file under `/assets`) and as the router's
`fallback_service` (any other unmatched path). `SetStatus::new(..., NOT_FOUND)`
is required because a bare `ServeFile` returns `200`; the router-level
`.fallback_service()` does NOT auto-wrap with `SetStatus` the way
`.not_found_service()` does.

### Pattern: a custom 404 handler as the not_found_service

```rust
// axum 0.7 / 0.8 - identical
use axum::{routing::get, Router, http::StatusCode, handler::HandlerWithoutStateExt};
use tower_http::services::ServeDir;

fn app() -> Router {
    async fn handle_404() -> (StatusCode, &'static str) {
        (StatusCode::NOT_FOUND, "Not found")
    }
    // .into_service() turns a handler into a Service that ServeDir can use.
    let serve_dir = ServeDir::new("assets")
        .not_found_service(handle_404.into_service());

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}
```

### Pattern: precompressed assets

```rust
// axum 0.7 / 0.8 - identical
use tower_http::services::ServeDir;

// Serves app.css.br when the client sends Accept-Encoding: br.
let serve_dir = ServeDir::new("assets")
    .precompressed_br()
    .precompressed_gzip();
```

`ServeDir` looks for a sibling file with the encoding extension
(`.br`, `.gz`, `.zst`, `.zz`) and serves it when `Accept-Encoding` permits.

## Reference Links

- `references/methods.md`: the `fs` and `set-status` feature flags, the
  `Service`-not-`Layer` distinction, full `ServeDir` / `ServeFile` / `SetStatus`
  signatures, every builder method, and the mount-method comparison table.
- `references/examples.md`: complete working routers for every pattern, the
  multi-directory case, and the full SPA fallback wired into `axum::serve`.
- `references/anti-patterns.md`: real mistakes with root-cause analysis,
  including `.layer(ServeDir)`, `.fallback` vs `.fallback_service`, the repeated
  prefix, and the `200`-status SPA fallback bug.

Related skills: `axum-core-router` (`.nest_service`, `.fallback`,
`.route_service`, prefix stripping), `axum-impl-tower-stack` (the `Service` and
`Layer` traits and why file services are Services).
