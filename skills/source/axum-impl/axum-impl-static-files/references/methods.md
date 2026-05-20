# axum-impl-static-files : Methods and API Signatures

All signatures verified via WebFetch against `docs.rs/tower-http` and the
official `axum` `static-file-server` example on 2026-05-20. Static file serving
is identical on Axum 0.7 and 0.8; the code lives in `tower-http`.

## Cargo features

`ServeDir` and `ServeFile` live in `tower_http::services` and are gated behind
the `fs` feature. The docs.rs pages state verbatim: "Available on crate feature
`fs` only."

`SetStatus` lives in `tower_http::set_status` and is gated behind a SEPARATE
feature: "Available on crate feature `set-status` only."

```toml
# directory or single-file serving
tower-http = { version = "0.6", features = ["fs"] }
# the SPA fallback also needs SetStatus
tower-http = { version = "0.6", features = ["fs", "set-status"] }
```

A missing feature surfaces as `unresolved import tower_http::services` or
`unresolved import tower_http::set_status` at compile time.

## Service, not Layer

`ServeDir`, `ServeFile`, and `SetStatus` implement `tower::Service`, NOT
`tower::Layer`. Verified `Service` associated types:

- `ServeDir`: `type Response = Response<ResponseBody>`,
  `type Error = Infallible`, `type Future = InfallibleResponseFuture<...>`.
  `ServeDir` is infallible: it never returns `Err`; a missing file produces a
  `404` response (or runs the configured fallback).
- `ServeFile`: delegates `Response` / `Error` / `Future` to `ServeDir`.
- `SetStatus<S>`: `Response` / `Error` match the inner service; `Future` is a
  `ResponseFuture` wrapping the inner future.

Because they are Services, they are mounted as route targets.

## Mount methods (on Router)

| Method | Argument type | Prefix handling |
|--------|---------------|-----------------|
| `.nest_service(path, svc)` | a `Service` | strips the matched prefix before the service sees the URI |
| `.fallback_service(svc)` | a `Service` | no stripping; the service sees the full path |
| `.route_service(path, svc)` | a `Service` | exact-match route bound to one Service |
| `.fallback(handler)` | a `Handler` | takes a handler function, NOT a Service |
| `.layer(layer)` | a `Layer` | NEVER valid for a file Service |

`.nest_service("/assets", ServeDir::new("assets"))`: a request for
`/assets/app.css` has `/assets` removed, so `ServeDir` resolves it to the disk
file `assets/app.css`. The disk path passed to `ServeDir::new` does NOT repeat
the URL prefix.

## ServeDir

```rust
pub struct ServeDir<F = DefaultServeDirFallback> { /* private fields */ }

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

Fallback handling differs between the two methods:

- `.fallback(svc)` returns `ServeDir<F2>`. The fallback's response is used
  as-is, with WHATEVER status the fallback produced.
- `.not_found_service(svc)` returns `ServeDir<SetStatus<F2>>`. The fallback is
  automatically wrapped in `SetStatus`, which forces the response status to
  `404 Not Found` regardless of the inner service's status.

So `.not_found_service(ServeFile::new("assets/index.html"))` already produces a
`404`-statused `index.html`: the implicit `SetStatus` rewrites the `ServeFile`
`200` into `404`.

`.append_index_html_on_directories(true)` (the default) serves `index.html` when
a directory path is requested. `.precompressed_*` makes `ServeDir` look for a
sibling file with the matching extension (`app.css.br`) and serve it when the
client's `Accept-Encoding` allows.

## ServeFile

```rust
pub struct ServeFile(/* private fields */);

pub fn new<P: AsRef<Path>>(path: P) -> Self
pub fn new_with_mime<P: AsRef<Path>>(path: P, mime: &Mime) -> Self
```

`new` guesses the `Content-Type` from the file extension. `new_with_mime` sets
an explicit MIME type and panics if the MIME is not a valid header value.

Builder methods: `precompressed_gzip`, `precompressed_br`,
`precompressed_deflate`, `precompressed_zstd`,
`with_buf_chunk_size(self, chunk_size: usize) -> Self` (the default read buffer
capacity is 64 KB).

`ServeFile` serves exactly one file. Bind it to a route with `.route_service()`.

## SetStatus

```rust
pub struct SetStatus<S> { /* private fields */ }

pub fn new(inner: S, status: StatusCode) -> Self
pub fn layer(status: StatusCode) -> SetStatusLayer
```

`SetStatus::new(inner, status)` wraps a Service and overrides EVERY response's
status code to `status`, regardless of the inner service's status. It is
`Clone` (when the inner service is), so one instance can be reused in two mount
points.

`SetStatus::layer(status)` returns a `SetStatusLayer`, which IS a `tower::Layer`
and applies the status override as middleware. The SPA fallback pattern uses
`SetStatus::new` (the Service form), not the layer form.

## Turning a handler into a Service

`ServeDir::not_found_service` and `ServeDir::fallback` accept a `Service`. To
pass a plain handler function, convert it with `.into_service()`, provided by
`axum::handler::HandlerWithoutStateExt`:

```rust
use axum::handler::HandlerWithoutStateExt;

async fn handle_404() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "Not found")
}
let svc = handle_404.into_service();
```
