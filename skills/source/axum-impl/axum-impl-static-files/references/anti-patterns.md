# axum-impl-static-files : Anti-Patterns

Each entry is a real mistake, the symptom, the root cause, and the fix. API
behavior verified against `tower-http` docs and the axum `static-file-server`
example on 2026-05-20.

## Anti-pattern: passing ServeDir to .layer()

```rust
// WRONG - axum 0.7 / 0.8. Does NOT compile.
use axum::Router;
use tower_http::services::ServeDir;

fn app() -> Router {
    Router::new().layer(ServeDir::new("assets"))
}
```

Symptom: a trait-bound compile error such as
`the trait bound ServeDir<_>: tower::Layer<_> is not satisfied`.

WHY it fails: `.layer()` expects a `tower::Layer`. `ServeDir` implements
`tower::Service`, not `tower::Layer`. A `Service` answers requests; a `Layer`
wraps a service. They are different traits. The mistake is common because other
`tower-http` items (`TraceLayer`, `CompressionLayer`) ARE layers.

Fix: mount `ServeDir` as a route service.

```rust
// CORRECT
fn app() -> Router {
    Router::new().nest_service("/assets", ServeDir::new("assets"))
}
```

## Anti-pattern: using .fallback() instead of .fallback_service()

```rust
// WRONG - axum 0.7 / 0.8. Does NOT compile.
use axum::Router;
use tower_http::services::ServeDir;

fn app() -> Router {
    Router::new().fallback(ServeDir::new("assets"))
}
```

Symptom: a trait-bound error stating `ServeDir` does not implement `Handler`.

WHY it fails: `.fallback()` expects a `Handler` (an async function or closure
that Axum can call). `ServeDir` is a `Service`, not a `Handler`. The two router
methods are deliberately separate: `.fallback()` for handlers,
`.fallback_service()` for services.

Fix: use `.fallback_service()` for any `tower-http` file service.

```rust
// CORRECT
fn app() -> Router {
    Router::new().fallback_service(ServeDir::new("assets"))
}
```

## Anti-pattern: repeating the URL prefix in the disk path

```rust
// WRONG INTENT - axum 0.7 / 0.8. Compiles, but every asset returns 404.
use axum::Router;
use tower_http::services::ServeDir;

fn app() -> Router {
    Router::new().nest_service("/assets", ServeDir::new("assets/assets"))
}
```

Symptom: no compile error, but `/assets/app.css` returns `404`. The server
looks for `assets/assets/app.css` on disk, which does not exist.

WHY it fails: `.nest_service("/assets", ...)` STRIPS the matched `/assets`
prefix before the inner service sees the URI. `ServeDir` then receives
`/app.css` and joins it onto its own base path. With base `assets/assets`, it
resolves `assets/assets/app.css`.

Fix: pass the bare directory. The prefix is handled by `.nest_service`, not by
`ServeDir`.

```rust
// CORRECT
fn app() -> Router {
    Router::new().nest_service("/assets", ServeDir::new("assets"))
}
```

## Anti-pattern: SPA fallback that returns 200 for unknown URLs

```rust
// WRONG - axum 0.7 / 0.8. Compiles, but every unknown URL returns 200 OK.
use axum::Router;
use tower_http::services::ServeFile;

fn app() -> Router {
    Router::new().fallback_service(ServeFile::new("assets/index.html"))
}
```

Symptom: a request for a genuinely non-existent path (`/does-not-exist`)
returns `200 OK` with the `index.html` body. Crawlers, monitoring, and API
clients cannot distinguish a real page from a missing one.

WHY it is wrong: a bare `ServeFile` always returns `200`. For a single-page
app, an unknown URL has no server-side resource; the correct HTTP semantics are
to return the `index.html` body WITH a `404` status so the client router still
renders, but the status is honest.

Fix: wrap the `ServeFile` in `SetStatus` to force the `404` status.

```rust
// CORRECT - tower-http features: ["fs", "set-status"]
use axum::{Router, http::StatusCode};
use tower_http::services::ServeFile;
use tower_http::set_status::SetStatus;

fn app() -> Router {
    let index_html = SetStatus::new(
        ServeFile::new("assets/index.html"),
        StatusCode::NOT_FOUND,
    );
    Router::new().fallback_service(index_html)
}
```

## Anti-pattern: confusing .fallback() and .not_found_service() on ServeDir

```rust
// MISLEADING - axum 0.7 / 0.8. Compiles, but the status is not 404.
use tower_http::services::{ServeDir, ServeFile};

// Developer expects a 404 status here.
let serve_dir = ServeDir::new("assets")
    .fallback(ServeFile::new("assets/index.html"));
```

Symptom: a missing file serves `index.html` with status `200`, not `404`.

WHY it is wrong: `.fallback(svc)` returns `ServeDir<F2>` and uses the fallback's
OWN status. `ServeFile` returns `200`, so the response is `200`.
`.not_found_service(svc)` returns `ServeDir<SetStatus<F2>>`: it wraps the
fallback in `SetStatus` and forces `404`.

Fix: use `.not_found_service()` when the fallback must carry a `404` status; use
`.fallback()` only when the fallback's own status is intentional.

```rust
// CORRECT - .not_found_service auto-wraps with SetStatus and forces 404
let serve_dir = ServeDir::new("assets")
    .not_found_service(ServeFile::new("assets/index.html"));
```

## Anti-pattern: forgetting the tower-http feature flag

```toml
# WRONG - ServeDir and ServeFile are not compiled in.
tower-http = "0.6"
```

Symptom: `unresolved import tower_http::services` (for `ServeDir` / `ServeFile`)
or `unresolved import tower_http::set_status` (for `SetStatus`).

WHY it fails: `ServeDir` and `ServeFile` are gated behind the `fs` feature;
`SetStatus` is gated behind the SEPARATE `set-status` feature. Without the
feature, the module is not compiled and the import does not resolve.

Fix: enable the features that the code uses.

```toml
# CORRECT - fs for file services, set-status for the SPA fallback
tower-http = { version = "0.6", features = ["fs", "set-status"] }
```

## Anti-pattern: treating a missing file as an error to handle

```rust
// WRONG MENTAL MODEL - axum 0.7 / 0.8.
// Expecting ServeDir to return an Err that needs a HandleError layer.
```

Symptom: a developer adds error-handling middleware around `ServeDir` expecting
to catch missing-file errors, and the middleware never fires.

WHY it is wrong: `ServeDir` has `type Error = Infallible`. It NEVER returns an
`Err`. A missing file produces a `404` RESPONSE (or runs the configured
`fallback` / `not_found_service`), not an error.

Fix: handle missing files with `.not_found_service()` or `.fallback()` on
`ServeDir`, not with error-handling middleware.
