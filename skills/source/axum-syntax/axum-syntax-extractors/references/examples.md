# axum-syntax-extractors: Examples

Every example is assembled from signatures and code verified via WebFetch
against docs.rs on 2026-05-20. The extractor surface is identical across Axum
0.7 and 0.8; the only version-divergent detail is the route-string capture
syntax read by `Path<T>`, annotated where it appears.

## 1. Single path parameter

```rust
// axum 0.8 - route: .route("/users/{id}", get(show))
async fn show(Path(id): Path<u64>) -> String {
    format!("user {id}")
}

// axum 0.7 - SAME handler, only the route string differs:
//            .route("/users/:id", get(show))
```

## 2. Multiple path parameters via tuple

```rust
// axum 0.8 - route: .route("/u/{uid}/t/{tid}", get(team))
// Tuple order matches the order of captures in the route.
async fn team(Path((uid, tid)): Path<(u64, u64)>) -> String {
    format!("user {uid}, team {tid}")
}
```

## 3. Path parameters via a named struct

```rust
// axum 0.8 - route: .route("/users/{user_id}/team/{team_id}", get(show))
// Struct field names must match the capture names in the route.
#[derive(serde::Deserialize)]
struct Params {
    user_id: uuid::Uuid,
    team_id: uuid::Uuid,
}

async fn show(Path(Params { user_id, team_id }): Path<Params>) {}
```

## 4. Capture every segment as a map

```rust
// axum 0.8 - collects all captures into a map
use std::collections::HashMap;

async fn params_map(Path(params): Path<HashMap<String, String>>) {
    for (key, value) in params {
        // ...
    }
}
```

## 5. Typed query string with optional fields

```rust
// axum 0.7 / 0.8 - identical. URL ?page=2&per_page=30 deserializes into Page.
#[derive(serde::Deserialize)]
struct Page {
    page: Option<u32>,
    per_page: Option<u32>,
}

async fn list(Query(p): Query<Page>) {
    let page = p.page.unwrap_or(1);
    let per_page = p.per_page.unwrap_or(20);
    // ...
}
```

Optional fields use `Option<T>` so an absent query parameter deserializes to
`None` rather than rejecting the request.

## 6. Json body extractor

```rust
// axum 0.7 / 0.8 - Json implements FromRequest, so it is the LAST argument.
#[derive(serde::Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

async fn create_user(Json(payload): Json<CreateUser>) -> String {
    format!("created {}", payload.name)
}
```

The request MUST carry a `Content-Type: application/json` header, otherwise
`Json` rejects with `JsonRejection::MissingJsonContentType`.

## 7. Correct extractor ordering

```rust
// axum 0.8 - parts extractors first, exactly one body extractor, and it is last.
use axum::{extract::{Path, State, Json}, http::{Method, HeaderMap}};

async fn create(
    method: Method,                     // FromRequestParts
    headers: HeaderMap,                 // FromRequestParts
    State(state): State<AppState>,      // FromRequestParts
    Path(id): Path<u64>,                // FromRequestParts
    Json(body): Json<CreatePayload>,    // FromRequest - LAST
) {}
```

## 8. Wrong extractor ordering (does not compile)

```rust
// axum 0.7 / 0.8 - WRONG: body extractor String is not the last argument.
// Fails the Handler trait bound.
async fn bad_position(body: String, method: Method) {}

// axum 0.7 / 0.8 - WRONG: two body extractors. The body cannot be read twice.
// Fails the Handler trait bound.
async fn bad_two_bodies(body: String, json: Json<Payload>) {}
```

## 9. Option wrapper (Axum 0.8 absent-vs-invalid semantics)

```rust
// axum 0.8 - Option yields None only when the value is genuinely absent.
// A present-but-malformed value REJECTS the request instead of yielding None.
async fn maybe(opt: Option<Query<Filters>>) {
    match opt {
        Some(Query(filters)) => { /* present and valid */ }
        None                 => { /* genuinely absent */ }
    }
}
```

NEVER use `Option<Extractor>` on Axum 0.8 to mean "ignore all errors". On Axum
0.7 `Option<Path<T>>` swallowed every error; Axum 0.8 removed that behavior.

## 10. Result wrapper to inspect a rejection

```rust
// axum 0.8 - Result lets the handler inspect every failure mode itself.
use axum::extract::rejection::JsonRejection;

async fn safe_create(payload: Result<Json<CreatePayload>, JsonRejection>) {
    match payload {
        Ok(Json(p)) => { /* use p */ }
        Err(rej)    => {
            // rej.status() gives the status code,
            // rej.body_text() gives a human-readable message
        }
    }
}
```
