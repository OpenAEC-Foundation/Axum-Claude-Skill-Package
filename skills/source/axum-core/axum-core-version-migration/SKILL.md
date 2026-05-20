---
name: axum-core-version-migration
description: >
  Use when upgrading an Axum project from 0.7 to 0.8, when an app that compiled
  on 0.7 now panics at startup or fails to compile after the version bump, or
  when deciding which path and extractor syntax a piece of Axum code targets.
  Prevents leaving 0.7 path syntax in place (a startup panic), keeping
  #[async_trait] on custom extractors (a compile error), relying on the old
  Option<Path<T>> swallow-all behavior, and calling APIs removed in 0.8.
  Covers the complete 0.7 to 0.8 breaking-change matrix, the path-syntax change,
  #[async_trait] removal and RPITIT, the WebSocket Message type change, the
  Option extractor change, the Sync bound, tuple Path arity, the removed-API
  checklist, and an ordered step-by-step migration checklist.
  Keywords: axum 0.7 to 0.8, axum upgrade, axum migration, breaking changes,
  matchit 0.8, path syntax panic, async_trait removed, RPITIT, OptionalFromRequest,
  Utf8Bytes, axum::Server removed, RawBody removed, TypedHeader moved, MSRV 1.75,
  app panics on startup after upgrade, code stopped compiling after cargo update,
  how do I upgrade axum, why does my router panic, what changed in axum 0.8.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-core-version-migration

This skill is the migration reference for moving an Axum project from version
0.7 to version 0.8. Axum 0.7.0 released 2023-11-27; Axum 0.8.0 released
2024-12-02. The upgrade has one panic-at-startup change (path syntax) and
several compile-time changes. Every breaking change is mechanical and
individually verifiable.

## When To Use This Skill

- Bumping `axum = "0.7"` to `axum = "0.8"` in `Cargo.toml`
- An Axum app that ran on 0.7 panics at startup after the version bump
- Code stopped compiling after `cargo update -p axum`
- A custom extractor no longer compiles (the `#[async_trait]` change)
- WebSocket code fails to compile (the `Message` payload type change)
- Deciding whether a path string or example targets 0.7 or 0.8

## Quick Reference

### Breaking-change matrix (0.7 to 0.8)

| Area | 0.7 form | 0.8 form |
|------|----------|----------|
| Path single capture | `/:id` | `/{id}` |
| Path catch-all | `/*rest` | `/{*rest}` |
| Old path syntax on 0.8 | accepted | PANICS at router construction |
| Literal `{` or `}` in a path | literal | escape by doubling: `{{`, `}}` |
| Custom extractor async | `#[async_trait]` on the `impl` | native `async fn` in trait (RPITIT) |
| Minimum Rust version | lower | Rust 1.75 |
| WebSocket `Message::Text` | `String` | `Utf8Bytes` |
| WebSocket `Binary` / `Ping` / `Pong` | `Vec<u8>` | `Bytes` |
| `Option<Path<T>>` and other `Option<Extractor>` | swallows all errors to `None` | rejects in many cases (`OptionalFromRequest`) |
| Handler / service `Sync` bound | not required | `Sync` required on `Router` and `MethodRouter` |
| Tuple `Path<T>` arity | lenient | parameter count must equal tuple length exactly |
| `WebSocket::close` | available | removed (send `Message::Close` explicitly) |
| `serve` listener / IO types | concrete | generic over listener and IO types |
| `Serve::tcp_nodelay` | available | removed |
| `RawBody` extractor | available | removed |
| `extract::BodyStream` | available | removed |
| `axum::Server` | available | removed (use `axum::serve`) |
| `B` body type parameter | generic on `Router` / handlers | removed; body is fixed `axum::body::Body` |
| `hyper::Body` re-export | re-exported by `axum` | removed (use `axum::body::Body`) |
| `headers` feature, `TypedHeader` | in the `axum` crate | moved to `axum-extra` |
| `Host` extractor | in the `axum` crate | moved to `axum-extra` |

Removed-API replacements and full signatures: `references/methods.md`.

### Severity classification

| Severity | Change | Failure mode |
|----------|--------|--------------|
| Panic at startup | 0.7 path syntax on 0.8 | server boots, then panics on router construction |
| Compile error | `#[async_trait]`, removed APIs, `B` param, `Sync` bound, `hyper::Body` | `cargo build` fails |
| Silent behavior change | `Option<Path<T>>` rejection, tuple `Path` arity | compiles, but runtime responses change |

ALWAYS treat the silent behavior changes as the highest audit priority: they do
NOT announce themselves with a panic or a compile error.

## Decision Trees

### Identifying which version a piece of code targets

```
Inspecting an Axum code sample or path string?
├── Path contains "/:name" or "/*name"           -> targets axum 0.7
├── Path contains "/{name}" or "/{*name}"        -> targets axum 0.8
├── A custom extractor impl has #[async_trait]   -> targets axum 0.7
├── Message::Text holds a String                 -> targets axum 0.7
├── Message::Text holds a Utf8Bytes / .into()    -> targets axum 0.8
├── axum::Server::bind(...)                      -> targets axum 0.7
└── axum::serve(listener, app)                   -> targets axum 0.7 or 0.8
```

`axum::serve` exists in both 0.7 and 0.8, so it alone does NOT identify the
version. ALWAYS confirm with the path syntax or an extractor impl.

### Converting a route path

```
Migrating a .route(), .nest(), or .route_service() path string?
├── Path has a ":param" single capture?
│   └── Rewrite ":param" as "{param}"
├── Path has a "*rest" catch-all?
│   └── Rewrite "*rest" as "{*rest}"
├── Path contains a literal "{" or "}"?
│   └── Double it: "{" becomes "{{", "}" becomes "}}"
└── Path needs a LITERAL ":" or "*" segment (not a capture)?
    └── Add Router::without_v07_checks() so the 0.7-syntax panic is disabled
```

### Migrating an `Option<Extractor>`

```
A handler argument is Option<Path<T>> (or another Option<Extractor>)?
├── The 0.7 code relied on a malformed value becoming None?
│   └── Switch to Result<Extractor, Rejection> to inspect the failure
└── You only need "absent -> None, present-and-valid -> Some"?
    └── Keep Option<Extractor>; accept the 0.8 reject-on-malformed semantics
```

## Migration Checklist

Follow top to bottom. Each step is mechanical and individually verifiable.

1. Bump dependencies: set `axum = "0.8"` in `Cargo.toml`, bump `axum-extra` to
   its 0.8-compatible release (0.10.x), run `cargo update -p axum`.
2. Set the toolchain to Rust 1.75 or newer (the 0.8 MSRV). Update
   `rust-toolchain.toml` and CI images that pin an older version.
3. Convert every route path: `:param` becomes `{param}`, `*rest` becomes
   `{*rest}`, in every `.route()`, `.nest()`, and `.route_service()` call.
   Escape literal braces as `{{` / `}}`. For a genuinely literal `:` or `*`
   segment, add `Router::without_v07_checks()`.
4. Remove `#[async_trait]` from every `impl FromRequest` and
   `impl FromRequestParts` block. Drop the `async-trait` dependency if nothing
   else uses it. The `async fn` bodies stay unchanged.
5. Fix WebSocket `Message` construction: `Message::Text(s.to_string())` becomes
   `Message::Text(s.into())`; `Message::Binary(v)` becomes
   `Message::Binary(v.into())`. Replace `String` / `Vec<u8>` pattern bindings on
   `Message` variants with `Utf8Bytes` / `Bytes`. Replace `socket.close()` with
   `socket.send(Message::Close(None))`.
6. Audit every `Option<Path<T>>` and other `Option<Extractor>`: switch to
   `Result<Extractor, Rejection>` where the 0.7 swallow-all behavior was relied
   on, otherwise accept the new reject-on-malformed semantics.
7. Check tuple `Path` arity: every `Path<(A, B, ...)>` tuple length must equal
   the number of `{captures}` in its route exactly.
8. Replace removed APIs: `axum::Server` becomes `axum::serve` plus a
   `TcpListener`; `RawBody` / `BodyStream` become `Bytes` / `String` /
   `Request::into_body()`; `hyper::Body` becomes `axum::body::Body`; remove the
   `B` body generic from `Router` and handler signatures.
9. Move `TypedHeader`, the `headers` feature, and the `Host` extractor to
   `axum-extra`: drop the `headers` feature from `axum`, add `axum-extra` with
   the `typed-header` feature, update imports to `axum_extra::TypedHeader` and
   `axum_extra::extract::Host`.
10. Satisfy the `Sync` bound: if the compiler reports a handler or service is
    not `Sync`, make the offending captured type `Sync`.
11. Compile and run: `cargo build`, then `cargo test`. A 0.7-syntax path missed
    in step 3 panics at startup, so also smoke-test the server boot.

## Patterns

### Pattern: path syntax conversion

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

The 0.7 syntax on a 0.8 router panics at router construction (startup), never at
request time. This is deliberate, so a routing regression cannot pass unnoticed.
NEVER use `without_v07_checks()` as a migration shortcut; it makes `:id` a
LITERAL segment, not a capture.

### Pattern: removing #[async_trait] from a custom extractor

```rust
// axum 0.7 - the macro is required
#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S)
        -> Result<Self, Self::Rejection> { /* ... */ }
}

// axum 0.8 - identical body, NO #[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for ExtractUserAgent {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S)
        -> Result<Self, Self::Rejection> { /* ... */ }
}
```

Axum 0.8 uses native return-position `impl Trait` in traits (RPITIT, stabilized
in Rust 1.75), so `FromRequest` and `FromRequestParts` no longer need
`#[async_trait]`. ALWAYS delete the attribute and remove the `async-trait`
dependency; the `async fn` body is unchanged. See `axum-syntax-custom-extractors`.

### Pattern: WebSocket Message type change

```rust
// axum 0.7
socket.send(Message::Text("hello".to_string())).await?;
socket.send(Message::Binary(vec![1, 2, 3])).await?;

// axum 0.8 - Utf8Bytes and Bytes, both have From impls from the old types
socket.send(Message::Text("hello".into())).await?;
socket.send(Message::Binary(vec![1, 2, 3].into())).await?;
```

`Message::Text` holds `Utf8Bytes` in 0.8 (was `String`); `Binary`, `Ping`, and
`Pong` hold `Bytes` (was `Vec<u8>`). `WebSocket::close` was removed in 0.8;
ALWAYS send `Message::Close(None)` explicitly instead.

### Pattern: Option extractor behavior change

```rust
// axum 0.7 - a malformed path id produced None
async fn handler(maybe: Option<Path<u64>>) { /* ... */ }

// axum 0.8 - a malformed path id REJECTS with 400; use Result to inspect it
async fn handler(res: Result<Path<u64>, PathRejection>) {
    match res {
        Ok(Path(id)) => { /* present and valid */ }
        Err(rej)     => { /* present but malformed, or absent */ }
    }
}
```

In 0.8, `Option<T>` requires `T` to implement `OptionalFromRequest` or
`OptionalFromRequestParts`. The new semantics distinguish "absent" from "present
but invalid": an absent optional value can still yield `None`, but a malformed
one rejects. This change is SILENT (it compiles), so audit every
`Option<Extractor>` explicitly.

### Pattern: replacing the removed server type

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

Full removed-API replacement table: `references/methods.md`.

## Common Mistakes

- Leaving a single `:param` path unconverted: the whole app panics at startup,
  not just that route. See `references/anti-patterns.md`.
- Keeping `#[async_trait]` on a custom extractor: a compile error on 0.8.
- Assuming `Option<Path<T>>` still swallows errors: it now rejects.
- Using `without_v07_checks()` to skip path migration: it makes captures literal.
- Forgetting the `axum-extra` move for `TypedHeader` and `Host`: import errors.

## Reference Links

- `references/methods.md` : the complete removed-and-moved API table with exact
  0.8 replacements, plus the `without_v07_checks` and `axum::serve` signatures.
- `references/examples.md` : full before-and-after migration examples, including
  `Cargo.toml` diffs and the `axum-extra` feature setup.
- `references/anti-patterns.md` : the migration mistakes that cause a startup
  panic, a compile error, or a silent behavior change, each with the fix.

## Related Skills

- `axum-core-router` : `.route()`, `.nest()`, and `without_v07_checks` in depth.
- `axum-syntax-custom-extractors` : the 0.7 and 0.8 forms of a custom extractor.
- `axum-impl-websockets` : the `WebSocket` and `Message` API in full.
