---
name: axum-syntax-responses
description: >
  Use when a handler must return a status code, headers, JSON, HTML, raw bytes,
  a redirect, or a cookie in Axum, or when a handler return value fails the
  Handler trait bound because the returned type does not implement IntoResponse.
  Prevents returning a bare domain struct or a bare integer that does not
  implement IntoResponse, prevents the wrong tuple order where the body is not
  the final element, prevents using 301 for a permanent redirect when Axum
  emits 308, and prevents a CookieJar silently dropping changes when the jar is
  not returned from the handler.
  Covers IntoResponse and IntoResponseParts, status and header and body tuple
  combining, Json and Html and raw byte bodies, Redirect (303, 307, 308),
  AppendHeaders, raw Set-Cookie headers, the axum-extra CookieJar family, and
  the Axum 0.8 axum::body::Body requirement.
  Keywords: axum IntoResponse, IntoResponseParts, ResponseParts, Json, Html,
  Form, Redirect, Redirect::to, Redirect::permanent, Redirect::temporary,
  AppendHeaders, NoContent, StatusCode, HeaderMap, Set-Cookie, CookieJar,
  SignedCookieJar, PrivateCookieJar, the trait bound Handler is not satisfied,
  handler return type does not implement IntoResponse, my cookie is not set,
  redirect returns the wrong status, 301 vs 308, how do I return JSON, how do I
  set a response header, how do I redirect, what status does Redirect use.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-syntax-responses

## Overview

A handler's return value becomes an `http::Response` through one trait:
`IntoResponse`. Its single method is `fn into_response(self) -> Response<Body>`,
where `Body` is `axum::body::Body`. Axum calls this method automatically on
whatever a handler returns, so the return type MUST implement `IntoResponse` or
the handler fails the `Handler` trait bound.

A companion trait, `IntoResponseParts`, modifies response parts (headers and
extensions) WITHOUT setting the status code or body. Tuples combine one
`IntoResponse` body with one or more `IntoResponseParts` elements and an
optional leading `StatusCode`, which is how a single return statement sets
status, headers, and body together.

Axum 0.8 removed the generic body type parameter. Every response body MUST be
`axum::body::Body`; `hyper::Body` is no longer re-exported.

## Quick Reference

| Return type | Status | Content-Type |
|-------------|--------|--------------|
| `()` | `200 OK` | none, empty body |
| `&'static str`, `String`, `Cow<str>` | `200 OK` | `text/plain; charset=utf-8` |
| `Vec<u8>`, `Bytes`, `&'static [u8]` | `200 OK` | `application/octet-stream` |
| `StatusCode` | that status | none, empty body |
| `HeaderMap`, `[(K, V); N]` | `200 OK` | none, headers only |
| `Json<T>` (`T: Serialize`) | `200 OK` | `application/json` |
| `Html<T>` (`T: Into<Body>`) | `200 OK` | `text/html; charset=utf-8` |
| `Form<T>` (`T: Serialize`) | `200 OK` | `application/x-www-form-urlencoded` |
| `NoContent` | `204 No Content` | none, empty body |
| `Redirect` | `303` / `307` / `308` | none, `Location` header |
| `Response` | unchanged | unchanged |
| `(StatusCode, R)` | leading status | from `R` |
| `(StatusCode, parts..., R)` | leading status | from `R`, plus parts |
| `Result<T, E>` (both `IntoResponse`) | from the variant | from the variant |

Core rules:

- ALWAYS return a type that implements `IntoResponse`. A bare domain struct or
  a bare integer does NOT implement it; wrap it in `Json(...)` or implement
  `IntoResponse` for it.
- ALWAYS make the body the LAST element of a response tuple. Every element
  before it MUST implement `IntoResponseParts`.
- ALWAYS place `StatusCode` first when a tuple sets the status.
- NEVER assume `Redirect::permanent` emits `301`. It emits `308`. `Redirect::to`
  emits `303`, `Redirect::temporary` emits `307`.
- ALWAYS return the `CookieJar` from the handler. `add` and `remove` consume the
  jar and return a new one; an unreturned jar silently loses every change.
- ALWAYS use `axum::body::Body` for a manually built `Response` on Axum 0.8.

## Decision Trees

### What return type for which response

```
What does the handler produce?
  a serializable struct as JSON          -> Json<T>
  an HTML page                           -> Html<T>
  only a status, no body                 -> StatusCode  (204 -> NoContent)
  a file or binary payload               -> (headers, Vec<u8>)  or  (headers, Bytes)
  plain text                             -> String  or  &'static str
  a redirect to another URL              -> Redirect  (see next tree)
  status + headers + body together       -> (StatusCode, [(HeaderName, _); N], body)
  a fallible operation with ? operator   -> Result<T, E>  where E: IntoResponse
  total control over the Response        -> Response  (build with Response::builder)
```

### Which Redirect constructor

```
Why are you redirecting?
  after a successful POST or PUT form submission
    -> Redirect::to        (303 See Other, client switches to GET, no resubmit)
  the resource moved permanently, keep the method and body
    -> Redirect::permanent (308 Permanent Redirect)
  the resource moved temporarily, keep the method and body
    -> Redirect::temporary (307 Temporary Redirect)
```

Axum exposes NO constructor for `301` or `302`. `301` and `302` rewrite a
non-GET method to GET in many clients; Axum deliberately offers only the
method-explicit codes `303`, `307`, and `308`.

### Which cookie mechanism

```
What kind of cookie value?
  non-sensitive, readable by the client (theme, locale)
    -> CookieJar         (axum-extra feature "cookie")
  client must not tamper with it, but may read it
    -> SignedCookieJar   (axum-extra feature "cookie-signed", needs Key)
  client must neither read nor tamper (session id, user id)
    -> PrivateCookieJar  (axum-extra feature "cookie-private", needs Key)
  no axum-extra dependency wanted, one simple cookie
    -> raw [(header::SET_COOKIE, "name=value; HttpOnly; Path=/")]
```

## Patterns

### Pattern: return JSON

`Json<T>` wraps any `T: Serialize`. A serialization failure produces
`500 Internal Server Error` automatically.

```rust
// axum 0.7 / 0.8 - identical
use axum::Json;
use serde::Serialize;

#[derive(Serialize)]
struct User { id: u64, name: String }

async fn get_user() -> Json<User> {
    Json(User { id: 7, name: "Ada".into() })
}
```

A bare `User` does NOT implement `IntoResponse`. Returning it fails the
`Handler` trait bound. The wrap in `Json(...)` is mandatory.

### Pattern: status, headers, and body in one tuple

Axum implements `IntoResponse` for `(StatusCode, T1, ..., Tn, R)` where every
`Ti` implements `IntoResponseParts` and the final `R` implements
`IntoResponse`. The leading `StatusCode` overrides the status; the final
element is the body.

```rust
// axum 0.7 / 0.8 - identical
use axum::{http::{StatusCode, header}, response::IntoResponse, Json};

async fn create() -> impl IntoResponse {
    (
        StatusCode::CREATED,                          // status, must be first
        [(header::CACHE_CONTROL, "no-store")],        // IntoResponseParts
        Json(serde_json::json!({ "id": 7 })),         // body, must be last
    )
}
```

The body is the LAST element. Putting it anywhere else, or putting two
body-typed elements in one tuple, fails to compile.

### Pattern: raw bytes with an explicit content type

A `Vec<u8>` body defaults to `application/octet-stream`. Override it by
prepending a header array.

```rust
// axum 0.7 / 0.8 - identical
use axum::{http::header, response::IntoResponse};

async fn download() -> impl IntoResponse {
    let bytes: Vec<u8> = std::fs::read("report.pdf").expect("file present");
    ([(header::CONTENT_TYPE, "application/pdf")], bytes)
}
```

### Pattern: HTML page and a redirect

```rust
// axum 0.7 / 0.8 - identical
use axum::response::{Html, Redirect};

async fn page() -> Html<&'static str> {
    Html("<h1>hi</h1>")
}

async fn submit_form() -> Redirect {
    Redirect::to("/thanks")          // 303 See Other, browser issues a GET
}

async fn moved() -> Redirect {
    Redirect::permanent("/new-path") // 308 Permanent Redirect, method preserved
}
```

### Pattern: custom error type for the ? operator

`Result<T, E>` implements `IntoResponse` when both `T` and `E` implement it.
Implement `IntoResponse` for a domain error so handlers can use `?`.

```rust
// axum 0.7 / 0.8 - identical
use axum::{http::StatusCode, response::{IntoResponse, Response}};

struct AppError(anyhow::Error);

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (StatusCode::INTERNAL_SERVER_ERROR, self.0.to_string()).into_response()
    }
}

async fn handler() -> Result<Json<User>, AppError> {
    let user = load_user().map_err(AppError)?;
    Ok(Json(user))
}
```

### Pattern: AppendHeaders to add without replacing

A `HeaderMap` or header array in a tuple REPLACES matching headers.
`AppendHeaders` APPENDS, which is required for emitting multiple `Set-Cookie`
headers in one response.

```rust
// axum 0.7 / 0.8 - identical
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

### Pattern: cookies with the axum-extra CookieJar

Axum core has no cookie API. `axum-extra` provides `CookieJar`,
`SignedCookieJar`, and `PrivateCookieJar`. A jar is extracted as a handler
argument and the MODIFIED jar MUST be returned, because `add` and `remove`
consume the jar and return a new one.

```rust
// axum 0.7 / 0.8 - identical. Cargo.toml: axum-extra with feature "cookie".
use axum::response::Redirect;
use axum_extra::extract::cookie::{Cookie, CookieJar};

async fn login(jar: CookieJar) -> (CookieJar, Redirect) {
    let jar = jar.add(
        Cookie::build(("session", "abc123"))
            .http_only(true)
            .secure(true)
            .path("/")
            .build(),
    );
    (jar, Redirect::to("/dashboard"))  // jar first (parts), Redirect last (body)
}
```

The jar implements `IntoResponseParts`, so it goes BEFORE the body in a tuple.
`SignedCookieJar` and `PrivateCookieJar` need a `Key` in application state,
reached via `FromRef`; see `references/examples.md`.

## Reference Links

- `references/methods.md`: the `IntoResponse` and `IntoResponseParts` trait
  definitions, the complete built-in implementation table, every tuple impl
  form, `Redirect` constructors and accessors, `Json` / `Html` / `Form` /
  `NoContent` / `AppendHeaders` signatures, and the `CookieJar` method set.
- `references/examples.md`: full version-annotated working code for every
  response type, the tuple combining rule, `SignedCookieJar` and
  `PrivateCookieJar` with a `Key`, and the Axum 0.8 `axum::body::Body` change.
- `references/anti-patterns.md`: real mistakes with root-cause analysis,
  including the bare-struct return, the wrong tuple order, the `301` redirect
  myth, and the unreturned cookie jar.

Related skills: `axum-syntax-handlers` (the `Handler` trait and what makes a
return type valid), `axum-errors-handling` (mapping domain errors to responses
through `IntoResponse`), `axum-syntax-extractors` (reading data from a request).
