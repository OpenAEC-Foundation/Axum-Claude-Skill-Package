# Topic Research : axum-syntax-responses

> Skill : `axum-syntax-responses`
> Phase : 4 (topic research)
> Target versions : Axum 0.7 and 0.8 (docs.rs current : 0.8.9), axum-extra 0.10.x
> Researched : 2026-05-20
> All API claims below are WebFetch-verified against official `docs.rs` pages on 2026-05-20.

This file scopes the verified research for the `axum-syntax-responses` skill: how a
handler's return value becomes an `http::Response`. It covers `IntoResponse`,
`IntoResponseParts`, status/header tuple combining, `Json`/`Html`/raw bodies,
`Redirect`, and cookie handling (raw `Set-Cookie` headers AND the `axum-extra`
`CookieJar` family).

---

## 1. The `IntoResponse` Built-In Implementation Table

`IntoResponse` is the conversion trait. Its single method, verified verbatim from
`docs.rs/axum/latest/axum/response/trait.IntoResponse.html`:

```rust
pub trait IntoResponse {
    fn into_response(self) -> Response<Body>;
}
```

Note the return type is `Response<Body>` where `Body` is `axum::body::Body`. In Axum
0.8 there is no longer a generic body type parameter; every `IntoResponse` MUST
produce a response whose body is `axum::body::Body` (see section 6).

Every type that implements `IntoResponse` out of the box, verified from the
"Implementations on Foreign Types" list on the trait page:

| Type | Status | Content-Type |
|------|--------|--------------|
| `()` | `200 OK` | none (empty body) |
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
| `StatusCode` | that status | none (empty body) |
| `HeaderMap` | `200 OK` | none (headers only, empty body) |
| `[(K, V); N]` (header array) | `200 OK` | none (headers only, empty body) |
| `Extensions` | `200 OK` | none (extensions only, empty body) |
| `Parts` (`http::response::Parts`) | from the parts | from the parts |
| `Response<B>` (any `Body`-compatible `B`) | unchanged | unchanged |
| `Infallible` | unreachable (cannot be constructed) | n/a |

Axum types that also implement `IntoResponse` (verified against the `response`
module overview):

- `Json<T>` where `T: Serialize` : JSON body, `content-type: application/json`,
  `200 OK`. A serialization failure produces `500 Internal Server Error`.
- `Html<T>` where `T: Into<Body>` : `content-type: text/html; charset=utf-8`,
  `200 OK`.
- `Form<T>` where `T: Serialize` : `application/x-www-form-urlencoded` body.
- `Redirect` : a `3xx` status plus a `Location` header (see section 3).
- `Extension<T>` : inserts a value into response extensions, empty body.
- `AppendHeaders<I>` : appends headers (does not replace existing ones).
- `NoContent` : `204 No Content`, empty body.
- `Sse<S>` : `text/event-stream`, server-sent-events streaming body.
- `Result<T, E>` where `T: IntoResponse` and `E: IntoResponse` : `Ok` and `Err`
  each convert to a response. This is the foundation of `?`-based error handling
  in handlers.
- Every extractor `Rejection` type (`JsonRejection`, `PathRejection`, etc.)
  implements `IntoResponse`, which is why a `Result<T, T::Rejection>` handler
  argument works.

DETERMINISTIC RULE : a handler may return any single type from this list directly.
To return a domain struct, you MUST wrap it (`Json(value)`) or implement
`IntoResponse` for it; a bare struct or a bare integer does NOT implement
`IntoResponse` and fails the `Handler` trait bound.

---

## 2. `IntoResponseParts` and Tuple Combining

`IntoResponseParts` modifies response *parts* (headers and extensions) WITHOUT
setting the status code or the body. Verified verbatim trait definition from
`docs.rs/axum/latest/axum/response/trait.IntoResponseParts.html`:

```rust
pub trait IntoResponseParts {
    type Error: IntoResponse;
    fn into_response_parts(
        self,
        res: ResponseParts,
    ) -> Result<ResponseParts, Self::Error>;
}
```

`ResponseParts` is an opaque struct holding the non-body response components
(status, headers, extensions). It exposes mutable accessors such as
`headers_mut()`. The `Error` associated type must itself implement `IntoResponse`,
so a failure while applying parts becomes a valid HTTP response rather than a
panic.

Types implementing `IntoResponseParts` (verified from the trait page):

- `()`
- `HeaderMap`
- `[(K, V); N]` (header arrays, `K: TryInto<HeaderName>`, `V: TryInto<HeaderValue>`)
- `Extensions`
- `Extension<T>`
- `AppendHeaders<I>`
- `Option<T>` where `T: IntoResponseParts`
- tuples `(T1,)` through `(T1, ..., T16)` where each `Ti: IntoResponseParts`

**Tuple-combining mechanism.** Axum implements `IntoResponse` for tuples of the
forms (verified from the `IntoResponse` trait page):

- `(R,)` where `R: IntoResponse`
- `(T1, R)` ... `(T1, ..., T16, R)` where every `Ti: IntoResponseParts` and
  `R: IntoResponse` (the LAST element is the body, all earlier elements are parts)
- `(StatusCode, R)` ... `(StatusCode, T1, ..., T16, R)` (a leading `StatusCode`
  overrides the status)
- `(Parts, R)` ... `(Parts, T1, ..., T16, R)`
- `(Response<()>, R)` ... `(Response<()>, T1, ..., T16, R)`

Evaluation order: the last element runs `into_response()` to produce the base
response; each preceding `IntoResponseParts` element runs `into_response_parts()`
in sequence, threading the `ResponseParts` through; a leading `StatusCode`
overwrites the status last. This is why `(StatusCode::CREATED, headers, Json(v))`
works in one return statement and never accidentally clobbers the body.

DETERMINISTIC RULE : in a parts/body tuple, EXACTLY ONE element (the final one) is
the body (`IntoResponse`); every element before it MUST be `IntoResponseParts`. A
`StatusCode`, if present, MUST be the first element.

---

## 3. `Redirect` Constructors and Exact Status Codes

Verified from `docs.rs/axum/latest/axum/response/struct.Redirect.html`. Every
constructor takes a `&str` URI and sets the `Location` response header.

| Constructor | Signature | Status code | Method-preservation behavior |
|-------------|-----------|-------------|------------------------------|
| `Redirect::to` | `to(uri: &str) -> Self` | `303 See Other` | instructs the client to change the request method to `GET` for the follow-up request |
| `Redirect::temporary` | `temporary(uri: &str) -> Self` | `307 Temporary Redirect` | preserves the original HTTP method and body |
| `Redirect::permanent` | `permanent(uri: &str) -> Self` | `308 Permanent Redirect` | preserves the original HTTP method and body |

Accessor methods : `location(&self) -> &str` returns the target URI;
`status_code(&self) -> StatusCode` returns the status. `Redirect` implements
`IntoResponse`.

DETERMINISTIC RULE :
- After a successful `POST`/`PUT` form submission, use `Redirect::to` (303) so the
  browser issues a `GET` and the user cannot resubmit the form on refresh.
- Use `Redirect::permanent` (308) for a permanent URL move where the method must
  be kept; use `301`-style behavior ONLY if you genuinely need method-rewrite,
  which Axum does not expose as a constructor.
- `Redirect::temporary` (307) for a temporary move that must keep `POST` as `POST`.

NOTE on the vooronderzoek : an earlier fragment stated `Redirect::permanent` issues
`301`. The live docs.rs page (verified 2026-05-20) states `permanent` issues
`308 Permanent Redirect` and `temporary` issues `307 Temporary Redirect`. The
skill MUST use the verified values: 303 / 307 / 308.

---

## 4. Cookie Handling

Axum core has NO cookie API. A cookie is a plain `Set-Cookie` response header on
the way out and a parsed `Cookie` request header on the way in.

### 4a. Raw `Set-Cookie` header

Because a header array is `IntoResponseParts`, a raw cookie is just a tuple
element:

```rust
use axum::http::header;
use axum::response::IntoResponse;

async fn login() -> impl IntoResponse {
    (
        [(header::SET_COOKIE, "session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/")],
        "logged in",
    )
}
```

This is dependency-free but unsigned, unencrypted, and requires manual attribute
formatting.

### 4b. The `axum-extra` `CookieJar` family

Verified from `docs.rs/axum-extra/latest/axum_extra/extract/cookie/index.html` and
`.../struct.CookieJar.html`. Three jar types:

| Type | Cargo feature | Purpose |
|------|---------------|---------|
| `CookieJar` | `cookie` | plain cookies, no crypto |
| `SignedCookieJar` | `cookie-signed` | tamper-evident (HMAC-signed) cookies |
| `PrivateCookieJar` | `cookie-private` | encrypted + authenticated cookies |

`Cookie` and `Key` are re-exported from the `cookie` crate (`cookie ^0.18`).

`CookieJar` verified methods:

- `CookieJar::new() -> Self`
- `CookieJar::from_headers(headers: &HeaderMap) -> Self`
- `get(&self, name: &str) -> Option<&Cookie<'static>>`
- `add<C: Into<Cookie<'static>>>(self, cookie: C) -> Self`
- `remove<C: Into<Cookie<'static>>>(self, cookie: C) -> Self`
- `iter(&self) -> impl Iterator<Item = &Cookie<'static>>`

`CookieJar` implements `FromRequestParts<S>` (so a handler extracts it as a parts
extractor) AND `IntoResponseParts` AND `IntoResponse`.

CRITICAL : `add` and `remove` CONSUME `self` and RETURN `Self`. The updated jar
MUST be returned from the handler for the `Set-Cookie` headers to reach the
response. A jar that is mutated but not returned silently loses the change.

The canonical pattern is a handler that takes a `CookieJar` and returns a
`CookieJar` (or a tuple with the jar first):

```rust
use axum_extra::extract::cookie::{Cookie, CookieJar};
use axum::response::Redirect;

async fn set_cookie(jar: CookieJar) -> CookieJar {
    jar.add(Cookie::new("session", "abc123"))
}

async fn login(jar: CookieJar) -> (CookieJar, Redirect) {
    let jar = jar.add(Cookie::new("session", "abc123"));
    (jar, Redirect::to("/"))   // jar first (IntoResponseParts), Redirect last (body)
}
```

`SignedCookieJar` and `PrivateCookieJar` require a `Key`. The `Key` is held in
application state and reached via `FromRef`, so the jar can be extracted:

```rust
use axum::extract::FromRef;
use axum_extra::extract::cookie::{Key, PrivateCookieJar};

#[derive(Clone)]
struct AppState { key: Key }

impl FromRef<AppState> for Key {
    fn from_ref(state: &AppState) -> Self { state.key.clone() }
}

async fn handler(jar: PrivateCookieJar) -> PrivateCookieJar {
    jar.add(Cookie::new("user_id", "42"))   // value is encrypted on the wire
}
```

DETERMINISTIC RULE : for session identifiers and any cookie a client must not read
or tamper with, use `PrivateCookieJar`; for integrity-only needs use
`SignedCookieJar`; use plain `CookieJar` only for non-sensitive values. ALWAYS
return the jar.

---

## 5. Lift-Ready Verified Snippets

```rust
// 1. Status + headers + JSON body in one tuple (parts-then-body combining).
use axum::{http::{StatusCode, header}, response::IntoResponse, Json};
async fn create() -> impl IntoResponse {
    (
        StatusCode::CREATED,
        [(header::CACHE_CONTROL, "no-store")],
        Json(serde_json::json!({ "id": 7 })),
    )
}
```

```rust
// 2. Raw bytes body with explicit content type.
use axum::{http::header, response::IntoResponse};
async fn download() -> impl IntoResponse {
    let bytes: Vec<u8> = std::fs::read("report.pdf").unwrap();
    ([(header::CONTENT_TYPE, "application/pdf")], bytes)
}
```

```rust
// 3. HTML response and a POST-redirect-GET redirect.
use axum::response::{Html, Redirect};
async fn page() -> Html<&'static str> { Html("<h1>hi</h1>") }
async fn submit() -> Redirect { Redirect::to("/thanks") }   // 303 See Other
```

```rust
// 4. Custom error type implementing IntoResponse for ?-based handlers.
use axum::{http::StatusCode, response::{IntoResponse, Response}};
struct AppError(anyhow::Error);
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (StatusCode::INTERNAL_SERVER_ERROR, self.0.to_string()).into_response()
    }
}
```

```rust
// 5. axum-extra CookieJar: set a cookie and redirect.
use axum_extra::extract::cookie::{Cookie, CookieJar};
use axum::response::Redirect;
async fn login(jar: CookieJar) -> (CookieJar, Redirect) {
    let jar = jar.add(
        Cookie::build(("session", "abc123"))
            .http_only(true)
            .secure(true)
            .path("/")
            .build(),
    );
    (jar, Redirect::to("/dashboard"))
}
```

```rust
// 6. AppendHeaders to add headers without replacing existing ones.
use axum::{http::header, response::{AppendHeaders, IntoResponse}};
async fn handler() -> impl IntoResponse {
    (
        AppendHeaders([
            (header::SET_COOKIE, "a=1"),
            (header::SET_COOKIE, "b=2"),
        ]),
        "two cookies appended",
    )
}
```

---

## 6. Axum 0.8 Response Body Note

In Axum 0.8 the generic body type parameter `B` was removed. Verified breaking
change: the response produced by `IntoResponse::into_response` MUST use
`axum::body::Body` as its body type, and `hyper::Body` is no longer re-exported.
The `IntoResponse` trait method signature on docs.rs is literally
`fn into_response(self) -> Response<Body>`. When porting 0.7 code that wrote
`Response<MyBody>` or used `hyper::Body`, change it to `axum::body::Body` (or use
`Body::from(...)` / `Body::empty()`).

---

## Sources Verified

All URLs fetched and verified via WebFetch on **2026-05-20**.

- https://docs.rs/axum/latest/axum/response/index.html : response module overview, Json/Html/Form/StatusCode/HeaderMap behavior, IntoResponseParts purpose.
- https://docs.rs/axum/latest/axum/response/trait.IntoResponse.html : verbatim `fn into_response(self) -> Response<Body>` signature, complete "Implementations on Foreign Types" list (unit, str, String, Box<str>, Cow, byte slices/arrays, Vec<u8>, Box<[u8]>, Bytes, BytesMut, StatusCode, HeaderMap, [(K,V);N], Extensions, Parts, Response<B>, Infallible), all tuple impl forms `(R,)` / `(T1..T16, R)` / `(StatusCode, ..)` / `(Parts, ..)` / `(Response<()>, ..)`.
- https://docs.rs/axum/latest/axum/response/trait.IntoResponseParts.html : verbatim trait definition (`type Error: IntoResponse`, `into_response_parts(self, res: ResponseParts) -> Result<ResponseParts, Self::Error>`), implementor list (HeaderMap, header arrays, Extensions, Extension, AppendHeaders, Option<T>, tuples up to 16), tuple-threading semantics.
- https://docs.rs/axum/latest/axum/response/struct.Redirect.html : verified constructors `to` (303 See Other, method->GET), `temporary` (307 Temporary Redirect, method preserved), `permanent` (308 Permanent Redirect, method preserved); `location()` and `status_code()` accessors; Location header.
- https://docs.rs/axum-extra/latest/axum_extra/extract/cookie/index.html : CookieJar / SignedCookieJar / PrivateCookieJar, Cargo features `cookie` / `cookie-signed` / `cookie-private`, Key via state + FromRef, handler-takes-and-returns-jar pattern.
- https://docs.rs/axum-extra/latest/axum_extra/extract/cookie/struct.CookieJar.html : `new`, `from_headers`, `get(&self, &str) -> Option<&Cookie<'static>>`, `add`/`remove` (consume self, return Self), `iter`; implements FromRequestParts + IntoResponseParts + IntoResponse; `Cookie`/`Key` re-exported from the `cookie` crate (`cookie ^0.18`).
