# axum-syntax-responses : Methods and API Signatures

All signatures verified against `docs.rs/axum` and `docs.rs/axum-extra` on
2026-05-20. Versions: Axum 0.7 and 0.8, axum-extra 0.10.x.

## The IntoResponse trait

```rust
// axum 0.8 - verbatim from docs.rs/axum/latest/axum/response/trait.IntoResponse.html
pub trait IntoResponse {
    fn into_response(self) -> Response<Body>;
}
```

`Body` is `axum::body::Body`. A handler's return type MUST implement this trait
or the `Handler` trait bound fails to compile.

Axum 0.7 had a generic body type parameter. Axum 0.8 removed it: every
`into_response` result uses `axum::body::Body`, and `hyper::Body` is no longer
re-exported.

## Built-in IntoResponse implementations on foreign types

Verified from the "Implementations on Foreign Types" list on the trait page.

| Type | Status | Content-Type |
|------|--------|--------------|
| `()` | `200 OK` | none, empty body |
| `&'static str` | `200 OK` | `text/plain; charset=utf-8` |
| `String` | `200 OK` | `text/plain; charset=utf-8` |
| `Box<str>` | `200 OK` | `text/plain; charset=utf-8` |
| `Cow<'static, str>` | `200 OK` | `text/plain; charset=utf-8` |
| `&'static [u8]` | `200 OK` | `application/octet-stream` |
| `&'static [u8; N]` | `200 OK` | `application/octet-stream` |
| `Vec<u8>` | `200 OK` | `application/octet-stream` |
| `Box<[u8]>` | `200 OK` | `application/octet-stream` |
| `Cow<'static, [u8]>` | `200 OK` | `application/octet-stream` |
| `Bytes` | `200 OK` | `application/octet-stream` |
| `BytesMut` | `200 OK` | `application/octet-stream` |
| `StatusCode` | that status | none, empty body |
| `HeaderMap` | `200 OK` | none, headers only |
| `[(K, V); N]` | `200 OK` | none, headers only |
| `Extensions` | `200 OK` | none, extensions only |
| `Parts` (`http::response::Parts`) | from the parts | from the parts |
| `Response<B>` | unchanged | unchanged |
| `Infallible` | unreachable, cannot be constructed | n/a |

## Built-in IntoResponse implementations from axum

Verified against `docs.rs/axum/latest/axum/response/index.html`.

- `Json<T>` where `T: Serialize` : `200 OK`, `content-type: application/json`.
  A serialization failure produces `500 Internal Server Error`.
- `Html<T>` where `T: Into<Body>` : `200 OK`,
  `content-type: text/html; charset=utf-8`.
- `Form<T>` where `T: Serialize` : `application/x-www-form-urlencoded` body.
- `NoContent` : `204 No Content`, empty body.
- `Redirect` : a `3xx` status plus a `Location` header.
- `Extension<T>` : inserts a value into the response extensions, empty body.
- `AppendHeaders<I>` : appends headers without replacing existing ones.
- `Sse<S>` : `content-type: text/event-stream`, server-sent-events body.
- `Result<T, E>` where `T: IntoResponse` and `E: IntoResponse` : each variant
  converts to a response. This is the foundation of `?`-based error handling.
- Every extractor `Rejection` type (`JsonRejection`, `PathRejection`, ...)
  implements `IntoResponse`.

## The IntoResponseParts trait

```rust
// axum 0.8 - verbatim from docs.rs/axum/latest/axum/response/trait.IntoResponseParts.html
pub trait IntoResponseParts {
    type Error: IntoResponse;
    fn into_response_parts(
        self,
        res: ResponseParts,
    ) -> Result<ResponseParts, Self::Error>;
}
```

`IntoResponseParts` modifies headers and extensions only. It NEVER sets the
status code or the body. The `Error` associated type must itself implement
`IntoResponse`, so a failure while applying parts becomes a valid HTTP response.

`ResponseParts` is an opaque struct holding the non-body response components
(status, headers, extensions). It exposes mutable accessors such as
`headers_mut()`.

Types implementing `IntoResponseParts`:

- `()`
- `HeaderMap`
- `[(K, V); N]` where `K: TryInto<HeaderName>`, `V: TryInto<HeaderValue>`
- `Extensions`
- `Extension<T>`
- `AppendHeaders<I>`
- `Option<T>` where `T: IntoResponseParts`
- tuples `(T1,)` through `(T1, ..., T16)` where each `Ti: IntoResponseParts`

## Tuple implementations of IntoResponse

Verified from the `IntoResponse` trait page. Axum implements `IntoResponse`
for these tuple shapes:

- `(R,)` where `R: IntoResponse`
- `(T1, R)` through `(T1, ..., T16, R)` where every `Ti: IntoResponseParts`
  and `R: IntoResponse`
- `(StatusCode, R)` through `(StatusCode, T1, ..., T16, R)`
- `(Parts, R)` through `(Parts, T1, ..., T16, R)`
- `(Response<()>, R)` through `(Response<()>, T1, ..., T16, R)`

Evaluation order: the LAST element runs `into_response()` to produce the base
response; each preceding `IntoResponseParts` element runs `into_response_parts()`
in sequence, threading the `ResponseParts`; a leading `StatusCode` overwrites
the status last.

DETERMINISTIC RULE: exactly one element (the final one) is the body; every
element before it MUST implement `IntoResponseParts`; a `StatusCode`, if
present, MUST be first.

## Redirect

Verified from `docs.rs/axum/latest/axum/response/struct.Redirect.html`.

| Constructor | Signature | Status code | Method behavior |
|-------------|-----------|-------------|-----------------|
| `Redirect::to` | `to(uri: &str) -> Self` | `303 See Other` | client switches to `GET` |
| `Redirect::temporary` | `temporary(uri: &str) -> Self` | `307 Temporary Redirect` | original method and body preserved |
| `Redirect::permanent` | `permanent(uri: &str) -> Self` | `308 Permanent Redirect` | original method and body preserved |

Every constructor takes a `&str` URI and sets the `Location` response header.

Accessors:

```rust
pub fn location(&self) -> &str        // the target URI
pub fn status_code(&self) -> StatusCode  // the redirect status
```

`Redirect` implements `IntoResponse`. Axum exposes NO constructor for `301` or
`302`.

## axum-extra cookie API

Verified from `docs.rs/axum-extra/latest/axum_extra/extract/cookie/`.

| Type | Cargo feature | Purpose |
|------|---------------|---------|
| `CookieJar` | `cookie` | plain cookies, no crypto |
| `SignedCookieJar` | `cookie-signed` | tamper-evident (HMAC-signed) cookies |
| `PrivateCookieJar` | `cookie-private` | encrypted and authenticated cookies |

`Cookie` and `Key` are re-exported from the `cookie` crate (`cookie ^0.18`).

`CookieJar` methods:

```rust
CookieJar::new() -> Self
CookieJar::from_headers(headers: &HeaderMap) -> Self
fn get(&self, name: &str) -> Option<&Cookie<'static>>
fn add<C: Into<Cookie<'static>>>(self, cookie: C) -> Self
fn remove<C: Into<Cookie<'static>>>(self, cookie: C) -> Self
fn iter(&self) -> impl Iterator<Item = &Cookie<'static>>
```

`CookieJar` implements `FromRequestParts<S>` (so a handler extracts it),
`IntoResponseParts`, and `IntoResponse`.

CRITICAL: `add` and `remove` CONSUME `self` and RETURN a new `Self`. The updated
jar MUST be returned from the handler for the `Set-Cookie` headers to reach the
response. A jar that is mutated but not returned silently loses the change.

`SignedCookieJar` and `PrivateCookieJar` need a `Key`. The `Key` lives in
application state and is reached through an `impl FromRef<AppState> for Key`,
which lets the jar be extracted as a handler argument.
