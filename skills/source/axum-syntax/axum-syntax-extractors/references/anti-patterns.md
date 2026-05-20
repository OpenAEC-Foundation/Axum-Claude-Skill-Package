# axum-syntax-extractors: Anti-Patterns

Real, recurring mistakes with extractors in Axum, each with a root-cause
analysis. Verified against docs.rs and the Axum issue tracker on 2026-05-20.
The extractor surface is identical across Axum 0.7 and 0.8.

## AP-1: A body extractor that is not the last argument

```rust
// WRONG: String consumes the body but is not the last argument
async fn handler(body: String, headers: HeaderMap) {}
```

WHY IT FAILS: the request body is an asynchronous stream that can be consumed
exactly once. Axum enforces this by requiring the last handler argument to
implement `FromRequest` and every earlier argument to implement
`FromRequestParts`. A body extractor in a non-final position does not satisfy
the `Handler` trait bound, so the function is rejected with the opaque
`the trait bound ... Handler<_, _> is not satisfied` error, which never names
the real cause.

FIX: move the body extractor to the end. Place parts extractors
(`HeaderMap`, `Method`, `Path`, `Query`, `State`) before it. Apply
`#[debug_handler]` from `axum-macros` to get a precise diagnostic.

## AP-2: Two body extractors in one handler

```rust
// WRONG: both String and Json<T> consume the body
async fn handler(body: String, json: Json<Payload>) {}
```

WHY IT FAILS: the body can be read only once. Two `FromRequest` extractors both
try to consume it, which cannot satisfy the `Handler` trait bound. The compiler
reports the same opaque `Handler is not satisfied` error.

FIX: pick a single body extractor. If both the raw text and a parsed form are
needed, take the raw extractor (`Bytes` or `String`) and parse it inside the
handler, or take the whole `Request` and consume the body once explicitly.

## AP-3: A non-extractor handler argument

```rust
// WRONG: bool is not an extractor
async fn handler(flag: bool) {}
```

WHY IT FAILS: every handler argument must implement `FromRequestParts` or
`FromRequest`. A plain `bool`, `i32`, or domain struct implements neither, so
the function fails the `Handler` trait bound with the same cryptic error.

FIX: pass request data only through real extractors. Derive the value inside
the handler from extracted data, or wrap a custom type as a custom extractor
(see `axum-syntax-custom-extractors`).

## AP-4: Assuming Option<Path<T>> swallows all errors

```rust
// WRONG on Axum 0.8: expecting None on any failure
async fn handler(id: Option<Path<u64>>) {
    // Code written for 0.7 expects None when the segment is malformed.
}
```

WHY IT FAILS: on Axum 0.7, `Option<Path<T>>` swallowed every error condition
and produced `None`. A verified Axum 0.8 breaking change removed that: through
the `OptionalFromRequest` and `OptionalFromRequestParts` traits,
`Option<Extractor>` now distinguishes absent from present-but-invalid. A
present-but-malformed value REJECTS the request instead of yielding `None`, so
0.7 code silently changes behavior after upgrade.

FIX: use `Option<Extractor>` only when `None` should mean genuinely absent. To
handle every failure mode, use `Result<Extractor, Extractor::Rejection>` and
match on `Ok` / `Err` inside the handler.

## AP-5: Confusing State with Extension

```rust
// WRONG: application-wide state injected as a runtime extension
async fn handler(Extension(db): Extension<DbPool>) {}
```

WHY IT FAILS: `Extension<T>` is resolved at runtime from a type-keyed map. If
the matching `.layer(Extension(value))` is forgotten or mis-mounted, the
handler still compiles and every request returns `500 Internal Server Error`,
discovered only in production.

FIX: use `State<T>` for application-wide resources such as database pools and
configuration. `State` is checked at compile time: a router that is never given
its state does not type-check. Reserve `Extension` for per-request data that
middleware injects (auth claims, the authenticated user, a request id). See
`axum-core-state`.

## AP-6: Sending JSON without the Content-Type header

```rust
// Handler is correct, but the client omits Content-Type: application/json
async fn create(Json(payload): Json<CreateUser>) {}
```

WHY IT FAILS: `Json<T>` requires the request to carry a
`Content-Type: application/json` header. A request with a missing or wrong
content type is rejected with `JsonRejection::MissingJsonContentType` before
the handler body runs, which looks like a handler bug but is a client mistake.

FIX: ensure the client sends `Content-Type: application/json`. To surface a
clearer error, take `Result<Json<T>, JsonRejection>` and inspect the rejection
variant, or remap rejections globally with `axum-extra` `WithRejection` (see
`axum-errors-extractor-rejections`).
