# Topic Research : axum-core-version-migration

Focused research file for ONE skill: `axum-core-version-migration`. Scope is the
complete Axum 0.7 -> 0.8 migration. Every API-shape claim below is verified against
the official `tokio-rs/axum` `CHANGELOG.md` and the official 0.8.0 announcement blog
post, cross-referenced with the Phase 2 research fragments. Verified 2026-05-20.

Axum 0.7.0 released 2023-11-27; axum 0.8.0 released 2024-12-02 (release dates are
cross-checked knowledge, not transcribed from WebFetch; see L-001 and the verification
note at the end). All breaking-change WORDING and API SHAPES are verified.

---

## 1. Complete breaking-change matrix

| Change | 0.7 form | 0.8 form |
|--------|----------|----------|
| Path single-segment capture | `/:id` | `/{id}` |
| Path catch-all wildcard | `/*rest` | `/{*rest}` |
| Old path syntax on 0.8 | accepted | PANICS at router construction (deliberate, no silent change) |
| Literal `{`/`}` in a path | literal | must be escaped by doubling: `{{`, `}}` |
| Custom extractor async | `#[async_trait]` on the `impl` block | native `async fn` in trait (RPITIT), no macro |
| Minimum Rust version (MSRV) | lower | Rust 1.75 (RPITIT stabilization) |
| WebSocket `Message::Text` payload | `String` | `Utf8Bytes` |
| WebSocket `Message::Binary`/`Ping`/`Pong` payload | `Vec<u8>` | `Bytes` |
| `Option<Path<T>>` (and other `Option<Extractor>`) | swallows all errors -> `None` | rejects the request in many cases via `OptionalFromRequest` / `OptionalFromRequestParts` |
| Handler / service `Sync` bound | not required | `Sync` required for all handlers and services on `Router`/`MethodRouter` |
| Tuple / tuple-struct `Path<T>` | lenient | parameter count must match the tuple length exactly |
| `WebSocket::close` | available | removed (send a `Message::Close` explicitly) |
| `serve` listener/IO types | concrete | `serve` is generic over the listener and IO types |
| `Serve::tcp_nodelay` / `WithGracefulShutdown::tcp_nodelay` | available | removed |
| `RawBody` extractor | available | removed |
| `extract::BodyStream` extractor | available | removed |
| `axum::Server` | available | removed (use `axum::serve`) |
| `B` body type parameter | generic `B` on `Router`/handlers | removed; response body is fixed `axum::body::Body` |
| `hyper::Body` re-export | re-exported by `axum` | removed (use `axum::body::Body`) |
| `headers` feature + `TypedHeader` | in the `axum` crate | moved to `axum-extra` |
| `Host` extractor | in the `axum` crate | moved to `axum-extra` |

---

## 2. Each breaking change with before/after code

### 2.1 Path parameter syntax (highest-impact change)

CHANGELOG, verbatim: "Upgrade matchit to 0.8, changing the path parameter syntax from
`/:single` and `/*many` to `/{single}` and `/{*many}`." The old syntax does not
silently degrade; it PANICS at router construction so a routing regression cannot pass
unnoticed.

```rust
// axum 0.7
Router::new()
    .route("/users/:id", get(show))
    .route("/assets/*path", get(serve_file));

// axum 0.8
Router::new()
    .route("/users/{id}", get(show))
    .route("/assets/{*path}", get(serve_file));
```

Escaping a literal brace (blog, verbatim): "Escaping is done with double braces, so if
you want to match a literal `{` or `}` character, you can do so by writing `{{` or
`}}`."

`Router::without_v07_checks()` is the deliberate escape hatch for a router that must
keep literal `:` or `*` path strings; it disables the 0.7-syntax panic check. ALWAYS
prefer converting paths to `{...}` syntax; use `without_v07_checks` only for genuinely
literal `:`/`*` segments.

```rust
// axum 0.8 - opt out of the 0.7-syntax panic for literal colon/asterisk paths
Router::new().without_v07_checks()
    .route("/ratio/:value", get(handler)); // ":value" treated literally, no panic
```

### 2.2 `#[async_trait]` removal from custom extractors

Blog, verbatim: "This feature is called return-position `impl Trait` in traits and
means that we no longer need the `#[async_trait]` macro to define async methods in
traits." And: "If you have custom extractors that implement these traits, you will
need to remove the `#[async_trait]` annotation from them." This affects every custom
`FromRequest` / `FromRequestParts` impl. MSRV is now Rust 1.75 (the RPITIT
stabilization release).

```rust
// axum 0.7 - custom extractor with the macro
#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S) -> Result<Self, Self::Rejection> {
        parts.headers.get(USER_AGENT)
            .map(|v| ExtractUserAgent(v.clone()))
            .ok_or((StatusCode::BAD_REQUEST, "missing user-agent"))
    }
}

// axum 0.8 - identical body, NO #[async_trait], drop the `async-trait` dependency
impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S) -> Result<Self, Self::Rejection> {
        parts.headers.get(USER_AGENT)
            .map(|v| ExtractUserAgent(v.clone()))
            .ok_or((StatusCode::BAD_REQUEST, "missing user-agent"))
    }
}
```

### 2.3 WebSocket `Message` payload types

CHANGELOG, verbatim: "`axum::extract::ws::Message` now uses `Bytes` in place of
`Vec<u8>`, and a new `Utf8Bytes` type in place of `String`."

```rust
// axum 0.7
Message::Text(String)
Message::Binary(Vec<u8>)
Message::Ping(Vec<u8>)
Message::Pong(Vec<u8>)

// axum 0.8
Message::Text(Utf8Bytes)
Message::Binary(Bytes)
Message::Ping(Bytes)
Message::Pong(Bytes)
```

Constructing a text message changes accordingly. `Utf8Bytes` and `Bytes` are
cheap-to-clone reference-counted buffers; both have `From` impls from the old types.

```rust
// axum 0.7
socket.send(Message::Text("hello".to_string())).await?;
// axum 0.8
socket.send(Message::Text("hello".into())).await?;        // &str -> Utf8Bytes
socket.send(Message::Binary(vec![1, 2, 3].into())).await?; // Vec<u8> -> Bytes
```

CHANGELOG also: `WebSocket::close` was removed. Send `Message::Close(None)` explicitly
instead of calling `socket.close()`.

### 2.4 `Option<Path<T>>` rejection behavior

CHANGELOG, verbatim: "`Option<Path<T>>` no longer swallows all error conditions,
instead rejecting the request in many cases." Blog, verbatim: "Previously, any
rejections from the `T` extractor were simply ignored and turned into `None`. Now,
`Option<T>` as an extractor requires `T` to implement the new trait
`OptionalFromRequestParts` (or `OptionalFromRequest`)."

```rust
// axum 0.7 mental model - WRONG on 0.8
async fn handler(maybe: Option<Path<u64>>) {
    // 0.7: a malformed path id produced None
    // 0.8: a malformed path id now REJECTS with 400, the handler never runs
}

// axum 0.8 - to actually inspect the failure, use Result
async fn handler(res: Result<Path<u64>, PathRejection>) {
    match res {
        Ok(Path(id)) => { /* present and valid */ }
        Err(rej)     => { /* present but malformed, or absent */ }
    }
}
```

The 0.8 semantics distinguish "absent" from "present but invalid": a genuinely missing
optional value can still yield `None`, but a present-yet-malformed value rejects.

### 2.5 Tuple `Path` parameter-count check

CHANGELOG, verbatim: "The tuple and tuple_struct `Path` extractor deserializers now
check that the number of parameters matches the tuple length exactly." A `Path<(A, B)>`
against a route with one or three captures now rejects instead of silently coercing.

```rust
// axum 0.8 - the tuple arity MUST equal the number of {captures} in the route
// route "/users/{uid}/teams/{tid}"  ->  Path<(Uuid, Uuid)>   OK
// route "/users/{uid}"              ->  Path<(Uuid, Uuid)>   rejects on 0.8
```

### 2.6 `Sync` bound on handlers and services

CHANGELOG, verbatim: "Require `Sync` for all handlers and services added to `Router`
and `MethodRouter`." A handler or `tower::Service` that is `Send` but not `Sync` no
longer compiles on a 0.8 `Router`; make the captured types `Sync` (most are already).

---

## 3. Removed-API checklist with 0.8 replacements

| Removed / moved in 0.8 | 0.8 replacement |
|------------------------|-----------------|
| `RawBody` extractor | `Bytes`, `String`, or take the whole `Request` and call `.into_body()` |
| `extract::BodyStream` | take `Request`, call `.into_body()`, consume via `http_body_util` / `Body::into_data_stream()` |
| `axum::Server` (and `Server::bind`) | `axum::serve(listener, app)` with a `tokio::net::TcpListener` |
| `B` body type parameter on `Router`/handlers | removed; body is fixed `axum::body::Body`. `IntoResponse` output must use `axum::body::Body` |
| `hyper::Body` re-export from `axum` | `axum::body::Body` |
| `headers` Cargo feature + `TypedHeader` | the `axum-extra` crate (`axum_extra::TypedHeader`, `axum_extra::headers`) |
| `Host` extractor | the `axum-extra` crate (`axum_extra::extract::Host`) |
| `WebSocket::close` method | send `Message::Close(None)` explicitly over the socket |
| `Serve::tcp_nodelay`, `WithGracefulShutdown::tcp_nodelay` | configure nodelay on the `TcpListener` / accepted streams before serving |

```rust
// axum 0.7 - the old server type
axum::Server::bind(&"0.0.0.0:3000".parse().unwrap())
    .serve(app.into_make_service())
    .await
    .unwrap();

// axum 0.8 - serve over an explicit tokio listener
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, app).await.unwrap();
```

```toml
# axum 0.7 - TypedHeader behind a feature
axum = { version = "0.7", features = ["headers"] }

# axum 0.8 - TypedHeader now lives in axum-extra
axum = "0.8"
axum-extra = { version = "0.10", features = ["typed-header"] }
```

---

## 4. Ordered migration checklist

Follow top to bottom. Each step is mechanical and individually verifiable.

1. **Bump dependencies.** Set `axum = "0.8"` in `Cargo.toml`. Bump `axum-extra` to its
   0.8-compatible release (0.10.x). Run `cargo update -p axum`.
2. **Set the toolchain.** Ensure the Rust toolchain is >= 1.75 (0.8 MSRV). Update
   `rust-toolchain.toml` / CI images if they pin an older version.
3. **Convert every route path.** Replace `:param` -> `{param}` and `*rest` -> `{*rest}`
   in every `.route(...)`, `.nest(...)`, `.route_service(...)` call. Escape any literal
   brace as `{{` / `}}`. If a path must keep a literal `:`/`*`, add
   `Router::without_v07_checks()`.
4. **Remove `#[async_trait]`.** Delete the `#[async_trait]` attribute from every
   `impl FromRequest`/`impl FromRequestParts` block. Remove the `async-trait`
   dependency from `Cargo.toml` if nothing else uses it. The `async fn` bodies stay
   unchanged.
5. **Fix WebSocket `Message` construction.** Change `Message::Text(s.to_string())` ->
   `Message::Text(s.into())` and `Message::Binary(v)` -> `Message::Binary(v.into())`.
   Replace `Vec<u8>` / `String` pattern bindings on `Message` variants with `Bytes` /
   `Utf8Bytes`. Replace any `socket.close()` call with `socket.send(Message::Close(None))`.
6. **Audit `Option<Extractor>` usage.** For every `Option<Path<T>>` (and other
   `Option<...>` extractors) that relied on the 0.7 swallow-all behavior, switch to
   `Result<Extractor, Rejection>` if you need to inspect or recover from a malformed
   value; otherwise accept the new reject-on-error semantics.
7. **Check tuple `Path` arity.** Confirm every `Path<(A, B, ...)>` tuple length exactly
   matches the number of `{captures}` in its route.
8. **Replace removed APIs.** Swap `axum::Server` -> `axum::serve` + `TcpListener`.
   Replace `RawBody` / `BodyStream` with `Bytes` / `String` / `Request::into_body()`.
   Replace `hyper::Body` with `axum::body::Body`. Remove the `B` body generic from
   `Router`/handler signatures.
9. **Move `TypedHeader` / `headers` / `Host` to `axum-extra`.** Drop the `headers`
   feature from the `axum` dependency, add `axum-extra` with the `typed-header`
   feature, and update imports to `axum_extra::TypedHeader` / `axum_extra::extract::Host`.
10. **Satisfy the `Sync` bound.** If the compiler reports a handler or service is not
    `Sync`, make the offending captured type `Sync`.
11. **Compile and run the test suite.** `cargo build` then `cargo test`. A 0.7-syntax
    path missed in step 3 panics at startup, so also smoke-test the server boot.

---

## 5. Sources verified

All URLs fetched and verified via WebFetch on 2026-05-20.

- https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md - verbatim 0.8.0
  breaking-change bullets: matchit 0.8 path syntax, `Sync` requirement on handlers and
  services, tuple/tuple_struct `Path` parameter-count check, `Host` extractor move,
  `WebSocket::close` removal, `serve` generic over listener/IO, `tcp_nodelay` removal,
  `Option<Path<T>>` rejection change, `Message` `Bytes`/`Utf8Bytes` change, MSRV 1.75.
- https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0 - verbatim path-syntax change
  (`/:single`,`/*many` -> `/{single}`,`/{*many}`), brace escaping (`{{`,`}}`),
  `Option<T>` requiring `OptionalFromRequest`/`OptionalFromRequestParts`, RPITIT
  replacing `#[async_trait]` and the instruction to remove `#[async_trait]` from custom
  extractors.
- docs/research/fragments/research-a-core-routing-extractors.md (Phase 2, sections 3
  and 5) - `Router::without_v07_checks`, `RawBody`/`BodyStream` removal, `B` param /
  `hyper::Body` re-export removal, native trait definitions verified on docs.rs.
- docs/research/fragments/research-b-middleware-tower-realtime.md (Phase 2, section 6)
  - WebSocket `Message` 0.7->0.8 type table verified against the CHANGELOG.
- docs/research/fragments/research-c-errors-auth-db-ops.md (Phase 2, version note) -
  confirms `#[async_trait]` removal and path-syntax change affect cross-cutting code.

Verification caveat (per L-001): WebFetch's summarizer cannot reliably transcribe the
exact parenthesised CHANGELOG release-DATES (it returns chronologically impossible
values). Release dates above (0.7.0 = 2023-11-27, 0.8.0 = 2024-12-02) are
cross-checked knowledge, not transcribed. Every API-SHAPE and breaking-change-WORDING
claim above IS verified against the fetched official content. The 0.8.0 blog post
fetch surfaced the path-syntax, `Option<T>`, and `async_trait` sections but not the
WebSocket `Message` or MSRV sections; those are verified instead via the CHANGELOG
fetch, which states them explicitly.
