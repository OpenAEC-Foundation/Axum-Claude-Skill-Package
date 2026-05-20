# axum-syntax-responses : Working Examples

Every example is verified against the official docs.rs API (verified 2026-05-20).
Version-divergent lines are annotated `// axum 0.8` or `// axum 0.7`. Where no
annotation appears, the code is identical on both versions.

## Returning plain text and status only

```rust
// axum 0.7 / 0.8 - identical
use axum::http::StatusCode;

async fn text() -> &'static str {
    "plain text body, 200 OK, text/plain"
}

async fn owned_text() -> String {
    format!("computed at runtime")
}

async fn no_body() -> StatusCode {
    StatusCode::ACCEPTED            // 202, empty body
}
```

## Returning JSON

```rust
// axum 0.7 / 0.8 - identical
use axum::Json;
use serde::Serialize;

#[derive(Serialize)]
struct User { id: u64, name: String }

async fn get_user() -> Json<User> {
    Json(User { id: 7, name: "Ada".into() })
}

// Ad-hoc JSON with serde_json::json!
async fn get_status() -> Json<serde_json::Value> {
    Json(serde_json::json!({ "ok": true, "count": 3 }))
}
```

## Returning HTML

```rust
// axum 0.7 / 0.8 - identical
use axum::response::Html;

async fn page() -> Html<&'static str> {
    Html("<!doctype html><h1>hello</h1>")
}

async fn rendered() -> Html<String> {
    Html(format!("<h1>{}</h1>", "dynamic"))
}
```

## Status plus headers plus body in one tuple

```rust
// axum 0.7 / 0.8 - identical
use axum::{http::{StatusCode, header}, response::IntoResponse, Json};

async fn create() -> impl IntoResponse {
    (
        StatusCode::CREATED,                       // status first
        [
            (header::CACHE_CONTROL, "no-store"),
            (header::LOCATION, "/users/7"),
        ],                                          // IntoResponseParts
        Json(serde_json::json!({ "id": 7 })),       // body last
    )
}

// Status with body, no extra headers.
async fn not_found() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, "no such resource")
}
```

## Raw bytes with an explicit content type

```rust
// axum 0.7 / 0.8 - identical
use axum::{body::Bytes, http::header, response::IntoResponse};

async fn pdf() -> impl IntoResponse {
    let data: Vec<u8> = std::fs::read("report.pdf").expect("file present");
    ([(header::CONTENT_TYPE, "application/pdf")], data)
}

async fn image(bytes: Bytes) -> impl IntoResponse {
    ([(header::CONTENT_TYPE, "image/png")], bytes)
}
```

## Redirects with the exact status codes

```rust
// axum 0.7 / 0.8 - identical
use axum::response::Redirect;

// 303 See Other. After a POST or PUT, the browser issues a GET and cannot
// resubmit the form on refresh.
async fn after_form_post() -> Redirect {
    Redirect::to("/thanks")
}

// 308 Permanent Redirect. The URL moved for good; method and body preserved.
async fn moved_for_good() -> Redirect {
    Redirect::permanent("/new-home")
}

// 307 Temporary Redirect. Temporary move; method and body preserved.
async fn temporarily_elsewhere() -> Redirect {
    Redirect::temporary("/maintenance")
}
```

## Custom error type for the ? operator

```rust
// axum 0.7 / 0.8 - identical
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct User { id: u64 }

struct AppError(anyhow::Error);

// Any error type converts into AppError so handlers can use `?`.
impl<E: Into<anyhow::Error>> From<E> for AppError {
    fn from(err: E) -> Self { Self(err.into()) }
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("internal error: {}", self.0),
        )
            .into_response()
    }
}

async fn load() -> Result<Json<User>, AppError> {
    let raw = std::fs::read_to_string("user.json")?;   // io error -> AppError
    let user: User = serde_json::from_str(&raw)?;       // parse error -> AppError
    Ok(Json(user))
}
```

## AppendHeaders for multiple Set-Cookie headers

```rust
// axum 0.7 / 0.8 - identical
use axum::{http::header, response::{AppendHeaders, IntoResponse}};

async fn set_two_cookies() -> impl IntoResponse {
    (
        AppendHeaders([
            (header::SET_COOKIE, "a=1; Path=/"),
            (header::SET_COOKIE, "b=2; Path=/"),
        ]),
        "two cookies appended",
    )
}
```

A header array `[(header::SET_COOKIE, ...); 2]` would REPLACE rather than
append; `AppendHeaders` is required to emit more than one cookie.

## Raw Set-Cookie without axum-extra

```rust
// axum 0.7 / 0.8 - identical
use axum::{http::header, response::IntoResponse};

async fn login_raw() -> impl IntoResponse {
    (
        [(
            header::SET_COOKIE,
            "session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/",
        )],
        "logged in",
    )
}
```

This is dependency-free but unsigned, unencrypted, and the attribute string is
formatted by hand.

## axum-extra CookieJar : set and read

```rust
// axum 0.7 / 0.8 - identical. Cargo.toml: axum-extra = { features = ["cookie"] }
use axum::response::Redirect;
use axum_extra::extract::cookie::{Cookie, CookieJar};

// The handler takes a CookieJar and MUST return the modified jar.
async fn login(jar: CookieJar) -> (CookieJar, Redirect) {
    let jar = jar.add(
        Cookie::build(("session", "abc123"))
            .http_only(true)
            .secure(true)
            .path("/")
            .build(),
    );
    (jar, Redirect::to("/dashboard"))   // jar (parts) first, Redirect (body) last
}

// Reading a cookie. get returns Option<&Cookie>.
async fn whoami(jar: CookieJar) -> String {
    match jar.get("session") {
        Some(c) => format!("session = {}", c.value()),
        None => "no session".into(),
    }
}

// Removing a cookie. remove also consumes and returns the jar.
async fn logout(jar: CookieJar) -> (CookieJar, Redirect) {
    let jar = jar.remove(Cookie::from("session"));
    (jar, Redirect::to("/"))
}
```

## axum-extra PrivateCookieJar with a Key in state

```rust
// axum 0.7 / 0.8 - identical. Cargo.toml: axum-extra = { features = ["cookie-private"] }
use axum::extract::FromRef;
use axum_extra::extract::cookie::{Cookie, Key, PrivateCookieJar};

#[derive(Clone)]
struct AppState {
    key: Key,
}

// FromRef lets the jar pull the Key out of AppState during extraction.
impl FromRef<AppState> for Key {
    fn from_ref(state: &AppState) -> Self {
        state.key.clone()
    }
}

// The cookie value is encrypted and authenticated on the wire.
async fn store_user_id(jar: PrivateCookieJar) -> PrivateCookieJar {
    jar.add(Cookie::new("user_id", "42"))
}

async fn read_user_id(jar: PrivateCookieJar) -> String {
    match jar.get("user_id") {
        Some(c) => c.value().to_owned(),
        None => "anonymous".into(),
    }
}
```

`SignedCookieJar` uses the same pattern with feature `cookie-signed`; the value
is readable by the client but tamper-evident.

## Axum 0.8 : build a Response manually with axum::body::Body

```rust
// axum 0.8 - the body type MUST be axum::body::Body
use axum::{body::Body, http::{Response, StatusCode}};

async fn manual() -> Response<Body> {
    Response::builder()
        .status(StatusCode::OK)
        .header("content-type", "text/plain")
        .body(Body::from("manually built"))
        .expect("valid response")
}
```

```rust
// axum 0.7 - the same handler used hyper::Body, which 0.8 no longer re-exports.
// Porting to 0.8: replace hyper::Body with axum::body::Body.
```

Empty body on 0.8: `Body::empty()`. From bytes: `Body::from(vec_u8)`.
