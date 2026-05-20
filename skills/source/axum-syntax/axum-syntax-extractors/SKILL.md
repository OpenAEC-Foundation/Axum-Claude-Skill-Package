---
name: axum-syntax-extractors
description: >
  Use when a handler must read data from a request in Axum: path parameters,
  query strings, JSON or form bodies, headers, the method, or shared state.
  Prevents the "Handler is not satisfied" trait-bound error caused by a body
  extractor placed before another argument or by two body extractors in one
  handler, and prevents the 0.7-era assumption that Option<Path<T>> swallows
  every error.
  Covers the built-in extractor set, the FromRequestParts vs FromRequest
  classification, the body-extractor ordering rule, Path deserialization
  (single, tuple, struct, map), Json and Query behavior, and the 0.8 Option
  and Result wrapper semantics.
  Keywords: axum extractor, FromRequest, FromRequestParts, Path, Query, Json,
  Form, State, Extension, HeaderMap, Multipart, body extractor must be last,
  Handler is not satisfied, cannot consume body twice, 400 Bad Request path
  param, missing content-type json, Option Path swallows errors, how do I
  read a path parameter, parse the query string, get the request body, which
  extractor do I use.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-syntax-extractors

## Overview

An extractor is a type that knows how to construct itself from an incoming
request. Extractors are the ONLY way a handler argument receives request data.
Two traits govern every extractor, and the difference is whether the request
body is touched:

- `FromRequestParts<S>`: extracts from request parts only (method, URI,
  headers, extensions, captured path params). It NEVER reads the body.
- `FromRequest<S>`: extracts by consuming the request body.

The extractor surface is identical across Axum 0.7 and 0.8. The only
version-divergent detail is the route-string capture syntax that `Path<T>`
reads from: 0.7 uses `/:id` and `/*rest`, 0.8 uses `/{id}` and `/{*rest}`.
The extractor code itself does not change.

## Quick Reference

| Extractor | Trait | Extracts | Rejection |
|-----------|-------|----------|-----------|
| `Path<T>` | `FromRequestParts` | captured path segments, serde-deserialized | `PathRejection` |
| `Query<T>` | `FromRequestParts` | URL query string, serde-deserialized | `QueryRejection` |
| `State<T>` | `FromRequestParts` | shared application state (compile-time checked) | `Infallible` |
| `Extension<T>` | `FromRequestParts` | per-request typed value from request extensions | `ExtensionRejection` |
| `HeaderMap` | `FromRequestParts` | all request headers | `Infallible` |
| `Method` | `FromRequestParts` | the HTTP method | `Infallible` |
| `Uri` | `FromRequestParts` | the full request URI | `Infallible` |
| `Json<T>` | `FromRequest` | body parsed as JSON (`T: DeserializeOwned`) | `JsonRejection` |
| `Form<T>` | `FromRequest` | body as `application/x-www-form-urlencoded` | `FormRejection` |
| `Bytes` | `FromRequest` | raw request body bytes | `BytesRejection` |
| `String` | `FromRequest` | request body as a UTF-8 string | `StringRejection` |
| `Request` | `FromRequest` | the entire request, full ownership | `Infallible` |

Core rules:

- ALWAYS place every `FromRequestParts` extractor before the body extractor.
- ALWAYS make the single `FromRequest` (body) extractor the LAST argument.
- NEVER put two body extractors in one handler. The body is a single-use
  async stream and can be consumed only once.
- ALWAYS use `State<T>` for application resources. It is compile-time checked.
- NEVER use `Option<Extractor>` to mean "ignore all errors" on Axum 0.8.

## Decision Trees

### Which extractor for which input

```
Need data from the request?
  URL PATH (captured segments)?         -> Path<T>
  URL QUERY STRING (?a=1&b=2)?          -> Query<T>
  a request HEADER?
    all headers at once                 -> HeaderMap
    one typed header                    -> axum-extra TypedHeader
  the HTTP METHOD or full URI?          -> Method / Uri
  application-wide resource (DB pool)?  -> State<T>   (compile-time checked)
  per-request value from middleware?    -> Extension<T>
  the request BODY?  (pick ONE, place it LAST)
    JSON payload                        -> Json<T>
    HTML form (urlencoded)              -> Form<T>
    raw bytes                           -> Bytes
    UTF-8 text                          -> String
    total control over the request      -> Request
```

Two rules override every branch above:

1. At most one body (`FromRequest`) extractor per handler, and it MUST be last.
2. Prefer `State` over `Extension` for app resources. `State` is checked at
   compile time; a missing `Extension` is a runtime 500.

### Option vs Result wrapper (Axum 0.8)

```
Want to handle a possibly-missing or possibly-invalid extractor?
  value can be genuinely ABSENT, and a malformed value SHOULD reject
                                        -> Option<Extractor>
  want to inspect EVERY failure mode inside the handler
                                        -> Result<Extractor, Extractor::Rejection>
```

On Axum 0.8, `Option<Extractor>` distinguishes absent from present-but-invalid:
an absent value yields `None`, a present-but-malformed value REJECTS the
request. To catch every failure explicitly, use the `Result` wrapper instead.

## Patterns

### Pattern: extractor ordering

The body is an asynchronous stream consumed exactly once. Axum enforces this by
requiring the last extractor to implement `FromRequest` and all others to
implement `FromRequestParts`.

```rust
// axum 0.7 / 0.8 - CORRECT: parts extractors first, one body extractor last
use axum::{extract::{Path, State, Json}, http::{Method, HeaderMap}};

async fn ok(
    method: Method,                 // FromRequestParts
    headers: HeaderMap,             // FromRequestParts
    State(state): State<AppState>,  // FromRequestParts
    Path(id): Path<u64>,            // FromRequestParts
    Json(payload): Json<Payload>,   // FromRequest - LAST
) {}
```

```rust
// axum 0.7 / 0.8 - WRONG: body extractor String is not last. Does NOT compile.
async fn bad_position(body: String, method: Method) {}

// axum 0.7 / 0.8 - WRONG: two body extractors. Does NOT compile.
async fn bad_two_bodies(body: String, json: Json<Payload>) {}
```

A violation fails the `Handler` trait bound and surfaces as the opaque
`the trait bound ... Handler<_, _> is not satisfied` error. Apply
`#[debug_handler]` from `axum-macros` for a precise diagnostic.

### Pattern: Path deserialization

`Path<T>` deserializes captured route segments through serde. The route-string
syntax is version-specific; the extractor code is identical.

```rust
// axum 0.8 - single param.   route: .route("/users/{id}", get(show))
// axum 0.7 - same handler,   route: .route("/users/:id", get(show))
async fn show(Path(id): Path<u64>) {}

// axum 0.8 - tuple, order = order of captures.
// route: .route("/users/{user_id}/team/{team_id}", get(team))
async fn team(Path((user_id, team_id)): Path<(Uuid, Uuid)>) {}

// axum 0.8 - named struct, field names = capture names.
#[derive(serde::Deserialize)]
struct Params { user_id: Uuid, team_id: Uuid }
async fn show_struct(Path(Params { user_id, team_id }): Path<Params>) {}

// axum 0.8 - capture every segment as a map.
async fn params_map(Path(params): Path<HashMap<String, String>>) {}
```

Percent-encoded segments are decoded automatically. A segment that decodes to
invalid UTF-8 is rejected. Any deserialization failure produces a
`PathRejection` and a `400 Bad Request` response.

### Pattern: Query string

`Query<T>` is a `FromRequestParts` extractor, so it never touches the body and
may appear in any position alongside a body extractor.

```rust
// axum 0.7 / 0.8 - identical. URL ?page=2&per_page=30 deserializes into Page.
#[derive(serde::Deserialize)]
struct Page { page: Option<u32>, per_page: Option<u32> }

async fn list(Query(p): Query<Page>) {}
```

ALWAYS make optional query fields `Option<T>` so an absent parameter
deserializes to `None` instead of rejecting the request. A query string that
cannot be parsed rejects with a `400 Bad Request` (`QueryRejection`).

### Pattern: Json body extractor

`Json<T>` implements `FromRequest`, so it MUST be the last argument. It
requires a `Content-Type: application/json` request header.

```rust
// axum 0.7 / 0.8 - Json as the last (body) argument.
async fn create_user(Json(payload): Json<CreateUser>) {}
```

`Json<T>` rejects (`JsonRejection`) when the `Content-Type` header is missing
or wrong, when the body is syntactically invalid JSON, when valid JSON does not
deserialize into `T`, or when body buffering fails.

### Pattern: handling rejections explicitly

Wrap a body extractor in `Result` to inspect its rejection inside the handler
instead of returning Axum's default response.

```rust
// axum 0.8 - Result wrapper: handle every Json failure mode explicitly.
use axum::extract::rejection::JsonRejection;

async fn safe_create(payload: Result<Json<CreatePayload>, JsonRejection>) {
    match payload {
        Ok(Json(p)) => { /* use p */ }
        Err(rej)    => { /* inspect rej.status(), rej.body_text() */ }
    }
}
```

See `references/examples.md` for the `Option<Extractor>` variant and its 0.8
absent-vs-invalid semantics.

## Reference Links

- `references/methods.md`: the `FromRequestParts` and `FromRequest` trait
  definitions, every built-in extractor signature, `Path` / `Query` / `Json`
  struct definitions, and the `JsonRejection` variants.
- `references/examples.md`: full version-annotated working code for every
  extractor, the ordering rule, and the `Option` and `Result` wrappers.
- `references/anti-patterns.md`: real mistakes with root-cause analysis,
  including misplaced body extractors and the `Option<Path<T>>` 0.8 change.

Related skills: `axum-syntax-custom-extractors` (implementing the two traits),
`axum-errors-extractor-rejections` (the rejection module in depth),
`axum-core-state` (the `State` extractor and `Router<S>` typing model).
