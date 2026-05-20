# Topic Research : axum-syntax-extractors

> Skill : `axum-syntax-extractors`
> Phase : 4 (topic research)
> Target versions : Axum 0.7 and 0.8 (docs.rs current : 0.8.x)
> Researched : 2026-05-20
> All API-shape claims WebFetch-verified against docs.rs on 2026-05-20.

This file scopes the built-in extractor surface of Axum: which extractors exist,
which trait each implements, the body-extractor ordering rule, `Path<T>`
deserialization patterns, `Json<T>` / `Query<T>` exact behavior, `Option<Extractor>`
0.8 semantics, and a decision tree for picking an extractor.

---

## 1. What an extractor is

An extractor is a type that knows how to construct itself from an incoming request.
Extractors are the ONLY way handler arguments receive request data. Two traits govern
them, and the difference is whether the body is touched:

- `FromRequestParts<S>` : extracts from request *parts only* (method, URI, headers,
  extensions, captured path params). Never reads the body.
- `FromRequest<S>` : extracts by *consuming the request body*.

A blanket impl makes every `FromRequestParts` type also usable as a `FromRequest`
type. Therefore an all-parts handler always compiles; the trait constraint only bites
when a body extractor is in the wrong place (see section 3).

---

## 2. Classification table of built-in extractors

Verified against docs.rs `extract` index, `Path`, `Query`, `Json`, and the rejection
module on 2026-05-20.

| Extractor | Trait | What it extracts | Rejection type |
|-----------|-------|------------------|----------------|
| `Path<T>` | `FromRequestParts` | captured path segments, serde-deserialized | `PathRejection` |
| `Query<T>` | `FromRequestParts` | URL query string, serde-deserialized | `QueryRejection` |
| `State<T>` | `FromRequestParts` | shared application state (compile-time checked) | `Infallible` |
| `Extension<T>` | `FromRequestParts` | per-request typed value from request extensions | `ExtensionRejection` |
| `HeaderMap` | `FromRequestParts` | all request headers | `Infallible` |
| `Method` | `FromRequestParts` | the HTTP method | `Infallible` |
| `Uri` | `FromRequestParts` | the full request URI | `Infallible` |
| `Bytes` | `FromRequest` | raw request body bytes | `BytesRejection` |
| `String` | `FromRequest` | request body as UTF-8 string | `StringRejection` |
| `Json<T>` | `FromRequest` | body parsed as JSON (`T: DeserializeOwned`) | `JsonRejection` |
| `Form<T>` | `FromRequest` | body as `application/x-www-form-urlencoded` | `FormRejection` |
| `Request` | `FromRequest` | the entire request, full ownership | `Infallible` |

Key reading of the table: the top seven extractors are `FromRequestParts` (body-safe,
can appear in any position and any number of times). The bottom five are `FromRequest`
(body-consuming). `State`, `HeaderMap`, `Method`, `Uri`, and `Request` never fail
during extraction (`Infallible` rejection).

---

## 3. The extractor ordering rule

Verified verbatim from the `extract` module docs (2026-05-20):

> "The request body is an asynchronous stream that can only be consumed once.
> Therefore you can only have one extractor that consumes the request body."

And:

> "axum enforces this by requiring the last extractor implements `FromRequest` and
> all others implement `FromRequestParts`."

The rule, stated deterministically:

- A handler may have AT MOST ONE `FromRequest` (body) extractor.
- That body extractor MUST be the LAST argument.
- Every argument before the last MUST be a `FromRequestParts` extractor.

Violations do not produce a clear error. They fail the `Handler` trait bound and the
compiler emits the opaque `the trait bound ... Handler<_, _> is not satisfied`. Use
`#[debug_handler]` from `axum-macros` to get a precise diagnostic.

### Verified compile-fail examples

```rust
// axum 0.8 / 0.7 - WRONG: body extractor String is not the last argument.
// FAILS the Handler trait bound (does NOT compile).
async fn bad_position(body: String, method: Method) {}

// axum 0.8 / 0.7 - WRONG: two body extractors. The body cannot be read twice.
// FAILS the Handler trait bound (does NOT compile).
async fn bad_two_bodies(body: String, json: Json<Payload>) {}
```

### Verified correct example

```rust
// axum 0.8 / 0.7 - CORRECT: all FromRequestParts extractors first,
// exactly one FromRequest (body) extractor, and it is last.
use axum::{extract::{Path, State, Json}, http::{Method, HeaderMap}};

async fn ok(
    method: Method,            // FromRequestParts
    headers: HeaderMap,        // FromRequestParts
    State(state): State<AppState>, // FromRequestParts
    Path(id): Path<u64>,       // FromRequestParts
    Json(payload): Json<Payload>,  // FromRequest - LAST
) {}
```

---

## 4. Path<T> deserialization patterns

Struct definition, verified verbatim: `pub struct Path<T>(pub T);`

`Path<T>` deserializes captured path segments through serde. The capture syntax in the
route string itself is version-specific: `axum 0.7` uses `/:id` and `/*rest`;
`axum 0.8` uses `/{id}` and `/{*rest}` (matchit 0.8). The extractor side below is
identical across versions; only the route string differs.

All four patterns are verified verbatim against the `Path` docs (2026-05-20):

```rust
// axum 0.8 - single primitive. Route: .route("/users/{user_id}", get(user_info))
async fn user_info(Path(user_id): Path<Uuid>) {}

// axum 0.8 - multiple params via tuple. Tuple order = order of captures in path.
// Route: .route("/users/{user_id}/team/{team_id}", get(users_teams_show))
async fn users_teams_show(Path((user_id, team_id)): Path<(Uuid, Uuid)>) {}

// axum 0.8 - multiple params via named struct. Field names = capture names.
#[derive(serde::Deserialize)]
struct Params { user_id: Uuid, team_id: Uuid }
async fn show_struct(Path(Params { user_id, team_id }): Path<Params>) {}

// axum 0.8 - capture everything as a map.
async fn params_map(Path(params): Path<HashMap<String, String>>) {}

// axum 0.8 - capture everything as ordered key/value pairs.
async fn params_vec(Path(params): Path<Vec<(String, String)>>) {}
```

Percent-encoding and UTF-8, verified verbatim from the `Path` docs:

> "Any percent encoded parameters will be automatically decoded. The decoded
> parameters must be valid UTF-8, otherwise `Path` will fail and return a
> `400 Bad Request` response."

Any deserialization failure produces a `PathRejection` and a `400 Bad Request`
response. To customize that response, take `Result<Path<T>, PathRejection>` in the
handler signature, or follow the `customize-path-rejection` example.

---

## 5. Json<T> and Query<T> exact behavior

### Json<T>

Struct definition, verified verbatim: `pub struct Json<T>(pub T);`

As an extractor, `Json<T>` implements `FromRequest<S>` where `T: DeserializeOwned`
and `S: Send + Sync`. It consumes the body, so it MUST be the last handler argument.

The request is rejected (`JsonRejection`) in these documented failure modes:

- Missing or wrong `Content-Type: application/json` header. The request must carry a
  `Content-Type: application/json` header (or a compatible one).
- Body contains syntactically invalid JSON.
- Valid JSON that does not deserialize into the target type `T`.
- Request body buffering fails.

`JsonRejection` has four verified variants (verified against the rejection docs,
2026-05-20): `JsonDataError`, `JsonSyntaxError`, `MissingJsonContentType`,
`BytesRejection`. Each variant maps to a status code via the `status()` method; skill
authors should present the variant names as the authoritative surface and direct
readers to `JsonRejection::status()` for the exact code per variant rather than
hardcoding numbers.

As a response, `Json<T>` (for `T: Serialize`) serializes the value to JSON and sets
the `Content-Type: application/json` header automatically. If serialization fails, a
`500` response is produced with the error message as the body.

```rust
// axum 0.8 - Json as a body extractor. Last argument.
async fn create_user(extract::Json(payload): extract::Json<CreateUser>) {
    // payload is a CreateUser
}
```

### Query<T>

Struct definition, verified verbatim: `pub struct Query<T>(pub T);`

`Query<T>` implements `FromRequestParts<S>` where `T: DeserializeOwned` and
`S: Send + Sync` (verified verbatim from the impl block on docs.rs). It is a
PARTS extractor: it never touches the body, so it can appear in any position and
alongside a body extractor. Deserialization uses serde over the URL query string.

Rejection, verified verbatim: "If the query string cannot be parsed it will reject
the request with a `400 Bad Request` response." The rejection type is `QueryRejection`.

```rust
// axum 0.8 / 0.7 - identical. URL ?page=2&per_page=30 deserializes into Pagination.
#[derive(serde::Deserialize)]
struct Pagination { page: usize, per_page: usize }

async fn list_things(pagination: Query<Pagination>) {
    let pagination: Pagination = pagination.0;
}
```

Tip for optional query fields: make struct fields `Option<u32>` so absent query
parameters deserialize to `None` rather than rejecting.

---

## 6. Option<Extractor> behavior (0.8)

Verified from the `extract` index docs (2026-05-20): some extractors that implement
`OptionalFromRequestParts` or `OptionalFromRequest` can be wrapped in `Option`, with
the behavior depending on the specific implementation.

This is a deliberate 0.8 change. In `axum 0.7`, `Option<Path<T>>` swallowed ALL
error conditions and produced `None` on any failure. In `axum 0.8` that behavior was
removed: the verified 0.8.0 breaking change states `Option<Path<T>>` "no longer
swallows all error conditions, instead rejecting the request in many cases."

Deterministic 0.8 rule:

- `Option<Extractor>` distinguishes ABSENT from PRESENT-BUT-INVALID.
- A genuinely absent value yields `None`.
- A present-but-malformed value REJECTS the request rather than yielding `None`.

Do NOT use `Option<Extractor>` to mean "ignore all errors". To handle every failure
mode explicitly, use `Result<Extractor, Extractor::Rejection>` instead. Verified from
the docs: an extractor can be wrapped in `Result` "with the error being the rejection
type of the extractor", letting the handler inspect the failure.

```rust
// axum 0.8 - Option: None only when truly absent; rejects when present-but-invalid.
async fn maybe(opt: Option<Query<Filters>>) {}

// axum 0.8 - Result: handle every rejection explicitly inside the handler.
async fn explicit(res: Result<Json<Payload>, JsonRejection>) {
    match res {
        Ok(Json(payload)) => { /* ... */ }
        Err(rejection)    => { /* inspect rejection.status() etc. */ }
    }
}
```

---

## 7. Decision tree : which extractor for which input

```
Need data from the request?
├─ From the URL PATH (captured segments)?      -> Path<T>
├─ From the URL QUERY STRING (?a=1&b=2)?       -> Query<T>
├─ From a request HEADER?
│   ├─ all headers at once                     -> HeaderMap
│   └─ a single typed header                   -> axum-extra TypedHeader
├─ The HTTP METHOD or full URI?                -> Method / Uri
├─ Application-wide shared resource (DB pool)? -> State<T>   (compile-time checked)
├─ Per-request value injected by middleware?   -> Extension<T>
└─ From the request BODY?  (pick ONE, place LAST)
    ├─ JSON payload                            -> Json<T>
    ├─ HTML form (urlencoded)                  -> Form<T>
    ├─ raw bytes                               -> Bytes
    ├─ UTF-8 text                              -> String
    └─ need total control over the request     -> Request
```

Two rules that override every branch above:
1. At most one body (`FromRequest`) extractor per handler, and it must be last.
2. `State` vs `Extension` : prefer `State` for app resources (compile-time checked);
   `Extension` is runtime-resolved and a missing extension is a `500`.

---

## 8. Lift-ready verified code snippets

```rust
// SNIPPET 1 - axum 0.8 - single path param. Route: "/users/{id}"
async fn show(Path(id): Path<u64>) -> String { format!("user {id}") }

// SNIPPET 2 - axum 0.7 - SAME handler, 0.7 route syntax differs: "/users/:id"
// The extractor code is identical; only the route string changes between versions.
async fn show_v07(Path(id): Path<u64>) -> String { format!("user {id}") }

// SNIPPET 3 - axum 0.8 - two path params via tuple. Route: "/u/{uid}/t/{tid}"
async fn team(Path((uid, tid)): Path<(u64, u64)>) {}

// SNIPPET 4 - axum 0.8 - typed query string with optional fields
#[derive(serde::Deserialize)]
struct Page { page: Option<u32>, per_page: Option<u32> }
async fn list(Query(p): Query<Page>) {}

// SNIPPET 5 - axum 0.8 - correct ordering: parts first, body (Json) last
async fn create(
    method: Method,
    Path(id): Path<u64>,
    Json(body): Json<CreatePayload>,
) {}

// SNIPPET 6 - axum 0.8 - Result wrapper to inspect a body-extractor rejection
async fn safe_create(payload: Result<Json<CreatePayload>, JsonRejection>) {
    match payload {
        Ok(Json(p))  => { /* use p */ }
        Err(rej)     => { /* rej.status(), rej.body_text() */ }
    }
}
```

---

## 9. Sources verified

All URLs fetched and verified via WebFetch on **2026-05-20**.

- https://docs.rs/axum/latest/axum/extract/index.html - common-extractor table,
  body-consumption rule (verbatim), `FromRequest`/`FromRequestParts` enforcement
  (verbatim), `Option<T>` and `Result<T, T::Rejection>` wrapper behavior.
- https://docs.rs/axum/latest/axum/extract/struct.Path.html - `Path<T>` struct
  definition, all four deserialization patterns (single, tuple, struct, HashMap,
  Vec of tuples), percent-encoding and UTF-8 rejection wording (verbatim),
  `PathRejection`.
- https://docs.rs/axum/latest/axum/extract/struct.Query.html - `Query<T>` struct
  definition, `FromRequestParts` impl block bounds, serde deserialization,
  `QueryRejection` and 400 wording (verbatim).
- https://docs.rs/axum/latest/axum/struct.Json.html - `Json<T>` struct definition,
  `FromRequest` impl, content-type requirement, failure modes, `JsonRejection`,
  response-side serialization behavior.
- https://docs.rs/axum/latest/axum/extract/rejection/enum.JsonRejection.html -
  the four `JsonRejection` variants (`JsonDataError`, `JsonSyntaxError`,
  `MissingJsonContentType`, `BytesRejection`) and the `status()` method.

Verification caveat: the `JsonRejection` page documents the variant names and the
`status()` accessor but does not print per-variant status-code numbers. Skill content
should cite variant names plus `status()` as the authoritative surface and avoid
hardcoding per-variant numeric codes that were not verbatim on the fetched page.
