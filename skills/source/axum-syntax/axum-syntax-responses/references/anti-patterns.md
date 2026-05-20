# axum-syntax-responses : Anti-Patterns

Each entry is a real mistake, the exact symptom, the root cause, and the fix.
API behavior verified against docs.rs on 2026-05-20.

## Anti-pattern: returning a bare domain struct

```rust
// WRONG - axum 0.7 / 0.8. Does NOT compile.
struct User { id: u64 }

async fn get_user() -> User {
    User { id: 7 }
}
```

Symptom: `the trait bound fn(...) -> ... {get_user}: Handler<_, _> is not
satisfied`, or `User: IntoResponse is not satisfied`.

WHY it fails: a handler's return type MUST implement `IntoResponse`. A plain
struct does not. The `Handler` blanket impl is gated on the return type
implementing `IntoResponse`, so the failure surfaces as an opaque `Handler`
trait-bound error rather than pointing at the struct.

Fix: wrap the value in a type that implements `IntoResponse`, or implement
`IntoResponse` for the struct.

```rust
// CORRECT - wrap in Json (struct must derive Serialize).
use axum::Json;
use serde::Serialize;

#[derive(Serialize)]
struct User { id: u64 }

async fn get_user() -> Json<User> {
    Json(User { id: 7 })
}
```

## Anti-pattern: returning a bare integer or other unsupported scalar

```rust
// WRONG - axum 0.7 / 0.8. Does NOT compile.
async fn count() -> u64 {
    42
}
```

Symptom: the same `Handler is not satisfied` trait-bound error.

WHY it fails: `u64` does not implement `IntoResponse`. Only the types in the
built-in table (see `methods.md`) do; numbers are not among them.

Fix: return a supported type. Use `Json(42)`, or `StatusCode`, or a string.

```rust
// CORRECT
use axum::Json;

async fn count() -> Json<u64> {
    Json(42)
}
```

## Anti-pattern: body element is not last in a tuple

```rust
// WRONG - axum 0.7 / 0.8. Does NOT compile.
use axum::{http::StatusCode, Json};

async fn create() -> (Json<serde_json::Value>, StatusCode) {
    (Json(serde_json::json!({ "id": 7 })), StatusCode::CREATED)
}
```

Symptom: `StatusCode: IntoResponseParts is not satisfied`, or an
`IntoResponse` trait-bound error on the tuple.

WHY it fails: in a response tuple the LAST element is the body and must
implement `IntoResponse`; every earlier element must implement
`IntoResponseParts`. `StatusCode` implements `IntoResponse` but as a leading
element, not as a trailing one in this order. The body (`Json`) must be last.

Fix: put `StatusCode` first and the body last.

```rust
// CORRECT
async fn create() -> (StatusCode, Json<serde_json::Value>) {
    (StatusCode::CREATED, Json(serde_json::json!({ "id": 7 })))
}
```

## Anti-pattern: two body-typed elements in one tuple

```rust
// WRONG - axum 0.7 / 0.8. Does NOT compile.
async fn two_bodies() -> (String, String) {
    ("first".into(), "second".into())
}
```

Symptom: `String: IntoResponseParts is not satisfied`.

WHY it fails: only the final element may be the body. The first `String` would
have to implement `IntoResponseParts`, which it does not. A response has exactly
one body.

Fix: produce a single body. Concatenate, or pick one value.

## Anti-pattern: assuming Redirect::permanent emits 301

```rust
// MISLEADING ASSUMPTION - axum 0.7 / 0.8.
async fn moved() -> axum::response::Redirect {
    // Developer expects 301 Moved Permanently here.
    axum::response::Redirect::permanent("/new")
}
```

Symptom: an integration test asserting `301` fails; the response status is
`308`. A client that downgrades the method on `301` keeps the method instead.

WHY the assumption is wrong: `Redirect::permanent` emits `308 Permanent
Redirect`, NOT `301`. `Redirect::to` emits `303 See Other` and `Redirect::
temporary` emits `307 Temporary Redirect`. Axum exposes no `301` or `302`
constructor on purpose: `303`, `307`, and `308` are method-explicit, while
`301` and `302` let clients silently rewrite the method.

Fix: assert against the real codes. Use `303` after a form POST,
`308` for a permanent move that keeps the method, `307` for a temporary move
that keeps the method.

## Anti-pattern: mutating a CookieJar without returning it

```rust
// WRONG - axum 0.7 / 0.8. Compiles, but the cookie is never sent.
use axum_extra::extract::cookie::{Cookie, CookieJar};

async fn login(jar: CookieJar) -> &'static str {
    jar.add(Cookie::new("session", "abc123"));   // return value discarded
    "logged in"
}
```

Symptom: the handler succeeds, no error appears, but the browser never
receives a `Set-Cookie` header and the session is not stored.

WHY it fails: `CookieJar::add` and `CookieJar::remove` CONSUME `self` and
RETURN a new jar. The `Set-Cookie` headers are emitted only when the jar is
part of the handler's response, because the jar implements `IntoResponseParts`.
Discarding the returned jar discards the change.

Fix: bind the returned jar and return it from the handler.

```rust
// CORRECT
async fn login(jar: CookieJar) -> (CookieJar, &'static str) {
    let jar = jar.add(Cookie::new("session", "abc123"));
    (jar, "logged in")
}
```

## Anti-pattern: header array used to set multiple Set-Cookie headers

```rust
// WRONG INTENT - axum 0.7 / 0.8. Compiles, but only one cookie survives.
use axum::http::header;

async fn cookies() -> impl axum::response::IntoResponse {
    (
        [
            (header::SET_COOKIE, "a=1"),
            (header::SET_COOKIE, "b=2"),
        ],
        "done",
    )
}
```

Symptom: only one `Set-Cookie` header reaches the client.

WHY it fails: a `[(HeaderName, value); N]` array applied as `IntoResponseParts`
behaves like inserting into a `HeaderMap`. A second insert of the same header
name REPLACES the first rather than appending. Cookies need multiple headers
with the same name.

Fix: use `AppendHeaders`, which appends instead of replacing.

```rust
// CORRECT
use axum::{http::header, response::AppendHeaders};

async fn cookies() -> impl axum::response::IntoResponse {
    (
        AppendHeaders([
            (header::SET_COOKIE, "a=1"),
            (header::SET_COOKIE, "b=2"),
        ]),
        "done",
    )
}
```

## Anti-pattern: using hyper::Body in a 0.8 response

```rust
// WRONG - axum 0.8. Does NOT compile.
use axum::http::Response;

async fn manual() -> Response<hyper::Body> {
    Response::new(hyper::Body::from("hi"))
}
```

Symptom: `cannot find type Body in crate hyper` after a re-export change, or
`expected Response<Body>, found Response<hyper::Body>`.

WHY it fails: Axum 0.8 removed the generic body type parameter and no longer
re-exports `hyper::Body`. `IntoResponse::into_response` returns
`Response<axum::body::Body>`, so a manually built response must use that body
type.

Fix: use `axum::body::Body`.

```rust
// CORRECT - axum 0.8
use axum::{body::Body, http::Response};

async fn manual() -> Response<Body> {
    Response::new(Body::from("hi"))
}
```

## Anti-pattern: plain CookieJar for a session identifier

```rust
// WRONG CHOICE - axum 0.7 / 0.8. Compiles and works, but is insecure.
use axum_extra::extract::cookie::{Cookie, CookieJar};

async fn login(jar: CookieJar) -> CookieJar {
    jar.add(Cookie::new("session_id", "user-42-secret"))
}
```

Symptom: no compile error and no runtime error, but the session identifier is
sent in cleartext and a client can read or forge it.

WHY it is wrong: `CookieJar` applies no cryptography. The value is visible and
editable by anyone with the cookie. A session identifier is exactly the kind of
value a client must not read or tamper with.

Fix: use `PrivateCookieJar` (encrypted and authenticated) for session
identifiers and any secret value, or `SignedCookieJar` (tamper-evident) when
the value may be read but must not be forged. Both need a `Key` in application
state reached through `FromRef`.
