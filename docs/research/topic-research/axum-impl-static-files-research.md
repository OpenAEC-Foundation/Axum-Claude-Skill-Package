# Topic Research : axum-impl-static-files

Focused research for the `axum-impl-static-files` skill. All API signatures and code
below were verified via WebFetch against the official `tower-http` docs and the
`axum` static-file-server example on **2026-05-20**. Target versions: Axum 0.7 / 0.8,
current `tower-http` on docs.rs at research time. Static file serving is identical
between Axum 0.7 and 0.8 (it is `tower-http`, not Axum, that provides it), so there
are no version-specific differences in this topic.

---

## 1. The `fs` Feature Flag

`ServeDir` and `ServeFile` live in `tower_http::services` and are gated behind the
`fs` Cargo feature. The docs.rs page states verbatim for both: "Available on **crate
feature `fs`** only."

```toml
# Cargo.toml
[dependencies]
tower-http = { version = "0.6", features = ["fs"] }
```

`SetStatus` lives in `tower_http::set_status` and is gated behind a SEPARATE feature:
"Available on **crate feature `set-status`** only." The SPA fallback pattern below
needs `SetStatus`, so the SPA case requires BOTH features:

```toml
tower-http = { version = "0.6", features = ["fs", "set-status"] }
```

ALWAYS enable `fs` for any static file serving. ALWAYS add `set-status` as well when
implementing the SPA fallback. A missing feature surfaces as an unresolved-import
compile error (`unresolved import tower_http::services` / `tower_http::set_status`).

---

## 2. Service, Not Layer : the Mounting Distinction

`ServeDir`, `ServeFile`, and `SetStatus` are `tower::Service` implementations, NOT
`tower::Layer` implementations. This is the single most important rule in this topic.

Verified `Service` impl associated types:

- `ServeDir` : `type Response = Response<ResponseBody>`, `type Error = Infallible`,
  `type Future = InfallibleResponseFuture<ReqBody, F>`. `ServeDir` is infallible : it
  never returns an `Err`, it returns a `404` (or runs its fallback) for a missing
  file.
- `ServeFile` : delegates its `Response` / `Error` / `Future` to `ServeDir`.
- `SetStatus<S>` : `Response` / `Error` match the inner service; `Future` is a
  `ResponseFuture` wrapping the inner future.

Because they are Services, they are mounted as ROUTE TARGETS, never via `.layer()`:

| Method | Use for | Prefix handling |
|--------|---------|-----------------|
| `.nest_service(path, svc)` | mount a directory under a URL prefix | strips the matched prefix before passing the request on |
| `.fallback_service(svc)` | serve unmatched paths with a Service | no prefix stripping; sees the full path |
| `.route_service(path, svc)` | bind one exact route to a Service | exact-match route |

NEVER call `.layer(ServeDir::new(...))` : a `Service` is not a `Layer` and this is a
trait-bound compile error. The mistake comes from `tower-http`'s middleware (e.g.
`TraceLayer`) being layers while its file services are not.

`.nest_service()` strips the matched prefix from the URI before the inner service
sees it. With `ServeDir::new("assets")` nested at `/assets`, a request for
`/assets/app.css` has the `/assets` prefix removed, so `ServeDir` resolves it to the
file `assets/app.css` on disk. This is why the directory path passed to
`ServeDir::new` does NOT repeat the URL prefix.

`.fallback_service()` is the Service-typed sibling of `.fallback()` : `.fallback()`
takes a handler (`Handler`), `.fallback_service()` takes a `Service`. `ServeDir` is a
`Service`, so it goes through `.fallback_service()`.

---

## 3. `ServeDir` : Verified Signatures

```rust
pub struct ServeDir<F = DefaultServeDirFallback> { /* private fields */ }
```

Constructor:

```rust
pub fn new<P>(path: P) -> Self
where P: AsRef<Path>
```

Verified builder methods:

```rust
pub fn append_index_html_on_directories(self, append: bool) -> Self
pub fn with_buf_chunk_size(self, chunk_size: usize) -> Self
pub fn precompressed_gzip(self) -> Self
pub fn precompressed_br(self) -> Self
pub fn precompressed_deflate(self) -> Self
pub fn precompressed_zstd(self) -> Self
pub fn call_fallback_on_method_not_allowed(self, call_fallback: bool) -> Self
pub fn fallback<F2>(self, new_fallback: F2) -> ServeDir<F2>
pub fn not_found_service<F2>(self, new_fallback: F2) -> ServeDir<SetStatus<F2>>
```

CRITICAL nuance : `.fallback()` and `.not_found_service()` differ in status handling.

- `.fallback(svc)` returns `ServeDir<F2>` : the fallback service's response is used
  as-is, with WHATEVER status the fallback produced.
- `.not_found_service(svc)` returns `ServeDir<SetStatus<F2>>` : the fallback service
  is automatically wrapped in `SetStatus`, which forces the response status to `404
  Not Found` regardless of what the inner service returned.

So `.not_found_service(ServeFile::new("assets/index.html"))` already produces a
`404`-statused `index.html` on its own : the `ServeFile` would normally return `200`,
but the implicit `SetStatus` rewrites it to `404`. The explicit `SetStatus` wrapping
in section 5 is only needed when the SAME `index.html` service is also used for the
ROUTER-level `.fallback_service()`, which does not apply that automatic wrapping.

`.precompressed_*` methods make `ServeDir` look for a sibling file with the matching
extension (e.g. `app.css.br`) and serve it when the client's `Accept-Encoding`
allows. `.append_index_html_on_directories(true)` (the default) serves `index.html`
when a directory is requested.

---

## 4. `ServeFile` : Verified Signatures

```rust
pub struct ServeFile(/* private fields */);
```

Constructors:

```rust
pub fn new<P: AsRef<Path>>(path: P) -> Self
pub fn new_with_mime<P: AsRef<Path>>(path: P, mime: &Mime) -> Self
```

`new` guesses the `Content-Type` from the file extension. `new_with_mime` sets an
explicit MIME type and "will panic if the mime type isn't a valid header value".

Builder methods: `precompressed_gzip`, `precompressed_br`, `precompressed_deflate`,
`precompressed_zstd`, `with_buf_chunk_size(self, chunk_size: usize) -> Self` (default
read buffer capacity is 64 KB).

`ServeFile` serves exactly one file. Bind it to a route with `.route_service()`.

---

## 5. `SetStatus` and the SPA Fallback Pattern

```rust
pub struct SetStatus<S> { /* private fields */ }

pub fn new(inner: S, status: StatusCode) -> Self
pub fn layer(status: StatusCode) -> SetStatusLayer
```

`SetStatus::new(inner, status)` wraps a Service and overrides every response's status
code to `status`, regardless of what the inner service returned. It is `Clone` /
`Copy` (when the inner service is), so the same instance can be reused.

The canonical single-page-app fallback : any unmatched path should serve `index.html`
so the client-side router can take over. Verified verbatim from the official
`static-file-server` example (`using_serve_dir_with_assets_fallback`):

```rust
use axum::{routing::get, Router, http::StatusCode};
use tower_http::services::{ServeDir, ServeFile};
use tower_http::set_status::SetStatus;

fn using_serve_dir_with_assets_fallback() -> Router {
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

Why `SetStatus` here : a bare `ServeFile` serving `index.html` would return `200 OK`.
For an SPA, an unknown URL is conceptually "not found" on the server even though the
client router will render something, so the correct HTTP semantics are to return the
`index.html` body WITH a `404` status. `SetStatus::new(..., StatusCode::NOT_FOUND)`
rewrites the `200` into `404` while keeping the HTML body. The wrapped `index_html`
is used in two places : `.not_found_service()` on `ServeDir` (for missing files under
`/assets`) and `.fallback_service()` on the `Router` (for any other unmatched path).

---

## 6. Verified Snippets

Snippet A : mount a static directory under a URL prefix (verbatim,
`using_serve_dir`):

```rust
fn using_serve_dir() -> Router {
    Router::new().nest_service("/assets", ServeDir::new("assets"))
}
```

Snippet B : serve a directory from the site root via the router fallback (verbatim,
`using_serve_dir_only_from_root_via_fallback`):

```rust
fn using_serve_dir_only_from_root_via_fallback() -> Router {
    let serve_dir = ServeDir::new("assets")
        .not_found_service(ServeFile::new("assets/index.html"));

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}
```

Snippet C : a custom 404 handler as the `not_found_service` (verbatim,
`using_serve_dir_with_handler_as_service`):

```rust
fn using_serve_dir_with_handler_as_service() -> Router {
    async fn handle_404() -> (StatusCode, &'static str) {
        (StatusCode::NOT_FOUND, "Not found")
    }
    let service = handle_404.into_service();
    let serve_dir = ServeDir::new("assets").not_found_service(service);

    Router::new()
        .route("/foo", get(|| async { "Hi from /foo" }))
        .fallback_service(serve_dir)
}
```

Snippet D : multiple directories mounted under separate prefixes (verbatim,
`two_serve_dirs`):

```rust
fn two_serve_dirs() -> Router {
    Router::new()
        .nest_service("/assets", ServeDir::new("assets"))
        .nest_service("/dist", ServeDir::new("dist"))
}
```

Snippet E : serve a single file on one exact route (verbatim,
`using_serve_file_from_a_route`):

```rust
fn using_serve_file_from_a_route() -> Router {
    Router::new().route_service("/foo", ServeFile::new("assets/index.html"))
}
```

---

## 7. Anti-Patterns

- AP : `Router::new().layer(ServeDir::new("assets"))`. `ServeDir` is a `Service`, not
  a `Layer`; this is a trait-bound compile error. FIX : mount via `.nest_service()`,
  `.fallback_service()`, or `.route_service()`.
- AP : `.fallback(ServeDir::new(...))`. `.fallback()` expects a `Handler`, not a
  `Service`. FIX : use `.fallback_service()` for any `tower-http` file service.
- AP : repeating the URL prefix in the disk path, e.g.
  `.nest_service("/assets", ServeDir::new("assets/assets"))`. `.nest_service()` strips
  the `/assets` prefix before `ServeDir` sees the path. FIX : pass the bare directory
  (`ServeDir::new("assets")`).
- AP : serving an SPA `index.html` fallback with a bare `ServeFile` (returns `200`).
  Unknown SPA URLs should return the `index.html` body with a `404` status. FIX :
  wrap with `SetStatus::new(ServeFile::new("assets/index.html"),
  StatusCode::NOT_FOUND)`.

---

## Sources Verified

All URLs fetched via WebFetch on **2026-05-20**.

| URL | Verified | Used for |
|-----|----------|----------|
| https://docs.rs/tower-http/latest/tower_http/services/struct.ServeDir.html | 2026-05-20 | Section 1, 2, 3 (`ServeDir` signature, all builder methods, `fs` feature, Service associated types) |
| https://docs.rs/tower-http/latest/tower_http/services/struct.ServeFile.html | 2026-05-20 | Section 1, 4 (`ServeFile` signatures, `new_with_mime`, builder methods, `fs` feature) |
| https://docs.rs/tower-http/latest/tower_http/set_status/struct.SetStatus.html | 2026-05-20 | Section 1, 5 (`SetStatus::new` / `layer` signatures, `set-status` feature, Service associated types) |
| https://github.com/tokio-rs/axum/blob/main/examples/static-file-server/src/main.rs | 2026-05-20 | Section 2, 5, 6 (verbatim SPA fallback pattern and all verified snippets) |
| docs/research/fragments/research-b-middleware-tower-realtime.md (section 9) | 2026-05-20 | cross-check of the Service-not-Layer distinction and SPA pattern |
