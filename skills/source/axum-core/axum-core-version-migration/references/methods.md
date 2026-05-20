# axum-core-version-migration : Methods Reference

The complete removed-and-moved API inventory for the Axum 0.7 to 0.8 upgrade,
with the exact 0.8 replacement for each item. All breaking-change wording and API
shapes are verified against the official `tokio-rs/axum` CHANGELOG and the 0.8.0
announcement blog post (verified 2026-05-20). Source URLs are listed at the end.

## Removed and moved APIs with replacements

| Removed or moved in 0.8 | 0.8 replacement |
|-------------------------|-----------------|
| `RawBody` extractor | `Bytes`, `String`, or take the whole `Request` and call `.into_body()` |
| `extract::BodyStream` | take `Request`, call `.into_body()`, consume via `http_body_util` or `Body::into_data_stream()` |
| `axum::Server` and `Server::bind` | `axum::serve(listener, app)` with a `tokio::net::TcpListener` |
| `B` body type parameter on `Router` and handlers | removed; the body is fixed `axum::body::Body`. `IntoResponse` output must use `axum::body::Body` |
| `hyper::Body` re-export from `axum` | `axum::body::Body` |
| `headers` Cargo feature and `TypedHeader` | the `axum-extra` crate: `axum_extra::TypedHeader`, `axum_extra::headers` |
| `Host` extractor | the `axum-extra` crate: `axum_extra::extract::Host` |
| `WebSocket::close` method | send `Message::Close(None)` explicitly over the socket |
| `Serve::tcp_nodelay`, `WithGracefulShutdown::tcp_nodelay` | configure nodelay on the `TcpListener` or accepted streams before serving |

## Signatures relevant to migration

### `axum::serve` (0.7 and 0.8)

```rust
pub fn serve<M, S>(listener: TcpListener, make_service: M) -> Serve<TcpListener, M, S>
```

`axum::serve` exists in both versions. In 0.8 it is generic over the listener and
IO types; in 0.7 the listener and IO types are concrete. The common call shape is
identical:

```rust
// axum 0.7 and 0.8
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, app).await.unwrap();
```

### `Router::without_v07_checks` (0.8 only)

```rust
pub fn without_v07_checks(self) -> Self
```

Disables the 0.8 panic that rejects 0.7-style path strings (paths beginning with
`:` or `*`). With it enabled, such segments are treated as LITERAL, not as
captures. Use it ONLY when a literal `:` or `*` path segment is genuinely
required, NEVER as a shortcut to avoid migrating path syntax.

```rust
// axum 0.8 - opt out of the 0.7-syntax panic for a literal colon path
Router::new()
    .without_v07_checks()
    .route("/ratio/:value", get(handler)); // ":value" is a literal segment here
```

## The custom-extractor traits (no `#[async_trait]` in 0.8)

In 0.8 the extractor traits use native return-position `impl Trait` in traits
(RPITIT), so no `#[async_trait]` macro is needed. Verified trait shapes:

```rust
// axum 0.8
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;
    fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}

pub trait FromRequest<S, M = ViaRequest>: Sized {
    type Rejection: IntoResponse;
    fn from_request(
        req: Request,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

In 0.7 the same traits are defined with `#[async_trait]`, and every `impl` block
carries the `#[async_trait]` attribute. Migration: delete the attribute from each
`impl`, then remove the `async-trait` dependency from `Cargo.toml`.

## WebSocket `Message` variant payloads

| Variant | 0.7 payload | 0.8 payload |
|---------|-------------|-------------|
| `Message::Text` | `String` | `Utf8Bytes` |
| `Message::Binary` | `Vec<u8>` | `Bytes` |
| `Message::Ping` | `Vec<u8>` | `Bytes` |
| `Message::Pong` | `Vec<u8>` | `Bytes` |
| `Message::Close` | `Option<CloseFrame>` | `Option<CloseFrame>` (unchanged) |

`Utf8Bytes` and `Bytes` are cheap-to-clone reference-counted buffers. Both have
`From` impls from the old types, so `"text".into()` and `vec.into()` construct
them. `WebSocket::close` is removed in 0.8; send `Message::Close(None)` instead.

## `Option<Extractor>` and the new optional traits

0.8 adds two traits that govern `Option<Extractor>`:

```rust
// axum 0.8 - the traits that Option<T> now requires of T
pub trait OptionalFromRequestParts<S>: Sized { /* ... */ }
pub trait OptionalFromRequest<S, M = ViaRequest>: Sized { /* ... */ }
```

In 0.7, `Option<Path<T>>` swallowed every rejection and produced `None`. In 0.8,
`Option<T>` requires `T` to implement `OptionalFromRequestParts` or
`OptionalFromRequest`; an absent value can yield `None`, but a present-yet-invalid
value rejects the request. To inspect a failure, use `Result<Extractor, Rejection>`
instead of `Option<Extractor>`.

## Other 0.8 breaking constraints

- `Sync` bound: 0.8 requires `Sync` for all handlers and services added to a
  `Router` or `MethodRouter`. A `Send`-but-not-`Sync` handler no longer compiles.
- Tuple `Path` arity: 0.8 checks that the number of route `{captures}` matches the
  tuple length of `Path<(A, B, ...)>` exactly. A mismatch rejects the request.
- Minimum Rust version: 0.8 requires Rust 1.75 or newer (the RPITIT stabilization
  release). Update `rust-toolchain.toml` and CI images.

## Sources verified (fetched 2026-05-20)

- https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md : the verbatim
  0.8.0 breaking-change bullets (matchit 0.8 path syntax, `Sync` requirement,
  tuple `Path` arity check, `Host` extractor move, `WebSocket::close` removal,
  `serve` generic over listener and IO, `tcp_nodelay` removal, `Option<Path<T>>`
  rejection change, `Message` `Bytes` and `Utf8Bytes` change, MSRV 1.75).
- https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0 : the verbatim
  path-syntax change, brace escaping, `Option<T>` requiring `OptionalFromRequest`
  or `OptionalFromRequestParts`, and the instruction to remove `#[async_trait]`
  from custom extractors.
- https://docs.rs/axum/latest/axum/ : `axum::serve`, `Router::without_v07_checks`,
  and the native `FromRequest` and `FromRequestParts` trait definitions.

Verification caveat: WebFetch's summarizer cannot reliably transcribe the exact
parenthesised CHANGELOG release dates. Release dates (0.7.0 = 2023-11-27,
0.8.0 = 2024-12-02) are cross-checked knowledge; every API-shape and
breaking-change-wording claim above is verified against the fetched official
content.
