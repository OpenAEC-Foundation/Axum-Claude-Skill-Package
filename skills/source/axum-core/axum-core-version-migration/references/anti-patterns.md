# axum-core-version-migration : Anti-Patterns

Migration mistakes that turn a 0.7-to-0.8 upgrade into a startup panic, a compile
error, or a silent behavior change. Each entry gives the symptom, the root cause,
and the fix. APIs verified against the official CHANGELOG and the 0.8.0
announcement blog on 2026-05-20.

## AP-1 : leaving 0.7 path syntax in place after the bump

```rust
// axum 0.8 - WRONG: 0.7 colon syntax still in the code
let app = Router::new()
    .route("/users/{id}", get(show_user))   // converted
    .route("/posts/:slug", get(show_post)); // MISSED, still 0.7 syntax
```

Symptom: the binary builds successfully, then the whole server panics at startup
on router construction. The panic is not limited to the one route; the app never
serves a request.

Root cause: Axum 0.8 upgraded `matchit` to 0.8. The old `:param` and `*rest`
syntax produces a deliberate panic so a routing regression cannot pass silently.

Fix: convert EVERY path string, then smoke-test the server boot. A single missed
path takes the whole app down at startup.

```rust
// axum 0.8 - CORRECT
let app = Router::new()
    .route("/users/{id}", get(show_user))
    .route("/posts/{slug}", get(show_post));
```

## AP-2 : using without_v07_checks to skip path migration

```rust
// axum 0.8 - WRONG: silencing the panic instead of migrating
let app = Router::new()
    .without_v07_checks()
    .route("/users/:id", get(show_user)); // ":id" is now a LITERAL segment
```

Symptom: the server boots without panicking, but the route never matches a real
request. `GET /users/42` returns `404`, because the router is waiting for the
literal path `/users/:id`.

Root cause: `without_v07_checks()` disables the 0.7-syntax panic by treating `:`
and `*` segments as LITERAL text, not as captures. It does not restore 0.7
capture behavior.

Fix: convert the path to 0.8 capture syntax. Reserve `without_v07_checks()` for a
path that genuinely needs a literal `:` or `*` character.

```rust
// axum 0.8 - CORRECT
let app = Router::new()
    .route("/users/{id}", get(show_user));
```

## AP-3 : keeping #[async_trait] on a custom extractor

```rust
// axum 0.8 - WRONG: the macro is still on the impl
#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for AuthToken {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S)
        -> Result<Self, Self::Rejection> { /* ... */ }
}
```

Symptom: `cargo build` fails on the `impl` block with a trait-signature mismatch.

Root cause: Axum 0.8 defines `FromRequest` and `FromRequestParts` with native
return-position `impl Trait` in traits (RPITIT). The `#[async_trait]` macro
rewrites the method signature into a boxed future, which no longer matches the
native trait definition.

Fix: delete the `#[async_trait]` attribute and its import. The `async fn` body is
unchanged. Remove the `async-trait` dependency from `Cargo.toml` if nothing else
uses it.

## AP-4 : assuming Option<Path<T>> still swallows errors

```rust
// axum 0.8 - WRONG mental model: this still compiles, but behaves differently
async fn handler(maybe_id: Option<Path<u64>>) -> String {
    match maybe_id {
        Some(Path(id)) => format!("id {id}"),
        None => "no id".to_string(),  // 0.7: also hit on a malformed id
    }
}
```

Symptom: no panic, no compile error. After the upgrade, a request with a
malformed path id (for example `/users/abc` where `u64` is expected) returns a
`400` rejection instead of reaching the `None` branch.

Root cause: in 0.8, `Option<T>` requires `T` to implement `OptionalFromRequest`
or `OptionalFromRequestParts`. A present-but-malformed value now rejects the
request; only a genuinely absent value yields `None`. This is a SILENT behavior
change, the most dangerous kind in a migration.

Fix: audit every `Option<Extractor>`. Where the 0.7 swallow-all behavior was
relied on, switch to `Result<Extractor, Rejection>` to inspect the failure.

```rust
// axum 0.8 - CORRECT
async fn handler(id: Result<Path<u64>, PathRejection>) -> String {
    match id {
        Ok(Path(id)) => format!("id {id}"),
        Err(_) => "id missing or malformed".to_string(),
    }
}
```

## AP-5 : mismatched tuple Path arity

```rust
// axum 0.8 - WRONG: tuple length does not match the route's capture count
let app = Router::new()
    .route("/users/{id}", get(handler));  // 1 capture

async fn handler(Path((id, extra)): Path<(u64, String)>) { /* ... */ }
// Path tuple has arity 2, the route has 1 capture -> 0.8 rejects the request
```

Symptom: no compile error. Every request to the route fails at runtime with a
path rejection.

Root cause: 0.8 checks that a tuple or tuple-struct `Path` deserializer has
exactly as many elements as the route has `{captures}`. 0.7 was lenient and
coerced silently.

Fix: make every `Path<(A, B, ...)>` tuple length equal the route's capture count.

## AP-6 : calling removed APIs

```rust
// axum 0.8 - WRONG: axum::Server and hyper::Body were removed
axum::Server::bind(&addr).serve(app.into_make_service()).await.unwrap();
let body: hyper::Body = hyper::Body::empty();
```

Symptom: `cargo build` fails with unresolved-path and unresolved-import errors.

Root cause: 0.8 removed `axum::Server`, the `hyper::Body` re-export, `RawBody`,
`extract::BodyStream`, the `B` body type parameter, and `Serve::tcp_nodelay`.

Fix: apply the replacement table in `references/methods.md`. `axum::Server`
becomes `axum::serve` plus a `TcpListener`; `hyper::Body` becomes
`axum::body::Body`; `RawBody` becomes `Bytes` / `String` / `Request::into_body()`.

```rust
// axum 0.8 - CORRECT
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, app).await.unwrap();
let body = axum::body::Body::empty();
```

## AP-7 : forgetting the axum-extra move for TypedHeader and Host

```rust
// axum 0.8 - WRONG: these no longer live in the axum crate
use axum::extract::Host;
use axum::TypedHeader;
```

Symptom: `cargo build` fails with unresolved-import errors for `Host` and
`TypedHeader`.

Root cause: 0.8 moved the `headers` feature, `TypedHeader`, and the `Host`
extractor out of `axum` into the `axum-extra` crate.

Fix: drop the `headers` feature from the `axum` dependency, add `axum-extra` with
the `typed-header` feature, and update the imports.

```rust
// axum 0.8 - CORRECT
use axum_extra::extract::Host;
use axum_extra::TypedHeader;
```

## AP-8 : a partial upgrade left on an old toolchain

```toml
# axum 0.8 - WRONG: dependency bumped, toolchain still pinned old
[dependencies]
axum = "0.8"
```
```toml
# rust-toolchain.toml still pins an older release
[toolchain]
channel = "1.70"
```

Symptom: `cargo build` fails with errors about `impl Trait` in trait position,
or the build succeeds locally but fails in CI on a pinned older image.

Root cause: Axum 0.8 requires Rust 1.75 or newer, the release that stabilized
return-position `impl Trait` in traits (RPITIT). An older toolchain cannot
compile the native async trait definitions.

Fix: set the toolchain to 1.75 or newer in `rust-toolchain.toml` AND in every CI
image, then rebuild.

## AP-9 : keeping socket.close() on a WebSocket

```rust
// axum 0.8 - WRONG: WebSocket::close was removed
socket.close().await?;
```

Symptom: `cargo build` fails; `WebSocket` has no `close` method.

Root cause: 0.8 removed `WebSocket::close`. Closing is now an explicit message.

Fix: send a close frame over the socket.

```rust
// axum 0.8 - CORRECT
socket.send(Message::Close(None)).await?;
```
