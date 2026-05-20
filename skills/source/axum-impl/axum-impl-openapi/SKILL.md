---
name: axum-impl-openapi
description: >
  Use when generating an OpenAPI 3 specification and a Swagger UI for an Axum
  service: documenting handlers, describing request and response schemas, or
  serving an interactive API explorer.
  Prevents the hand-written spec that silently drifts from the real handlers,
  the route registered on the Router but missing from the spec, the wrong
  {param} versus :id path syntax, and pinning three independently-versioned
  crates to mismatched releases.
  Covers utoipa derive macros ToSchema, OpenApi, IntoParams, IntoResponses,
  ToResponse, the #[utoipa::path] attribute, utoipa-swagger-ui SwaggerUi, and
  utoipa-axum OpenApiRouter with the routes! macro that registers a handler
  once for both routing and the spec.
  Keywords: axum openapi, utoipa, utoipa-axum, utoipa-swagger-ui, ToSchema,
  utoipa::path macro, derive OpenApi, OpenApiRouter, routes macro,
  split_for_parts, SwaggerUi, swagger ui, api documentation drift,
  spec out of date, route documented but returns 404, schema missing from spec,
  swagger ui blank page, how do I add swagger to axum, generate an openapi spec.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-impl-openapi

## Overview

Axum produces no API documentation of its own. The idiomatic solution is the
`utoipa` family of crates, which generates an OpenAPI 3 specification from Rust
annotations at compile time. Because the spec is derived from the same types
and handlers the server runs, it cannot silently drift from the code.

Three crates, each versioned independently:

- `utoipa` (5.5.0): the derive and attribute macros that annotate types and
  handlers.
- `utoipa-swagger-ui` (9.0.2): bundles and serves the interactive Swagger UI.
- `utoipa-axum` (0.2.0): supplies `OpenApiRouter`, which registers a handler
  ONCE for both routing and the spec, removing the last drift risk.

These three version numbers are NOT synchronised. ALWAYS pin all three
explicitly. The `utoipa` 4.x to 5.x bump changed macro behaviour; all code
here is valid for the 5.x line. Because `utoipa-axum` is still pre-1.0, treat
its `OpenApiRouter` surface as version-sensitive: a future `0.x` release may
rename `routes!`, `split_for_parts`, or the constructors.

## Quick Reference

Cargo dependencies (pin all three; versions verified 2026-05-20):

```toml
# version-sensitive: these three crates version independently
utoipa = { version = "5", features = ["axum_extras"] }
utoipa-axum = "0.2"
utoipa-swagger-ui = { version = "9", features = ["axum"] }
```

The `utoipa` macro surface:

| Macro | Kind | Applies to | Generates |
|-------|------|------------|-----------|
| `ToSchema` | derive | a type | a reusable OpenAPI schema component |
| `#[utoipa::path(...)]` | attribute | a handler `fn` | one OpenAPI operation |
| `OpenApi` | derive | a marker struct | the root OpenAPI document |
| `IntoParams` | derive | a query/path struct | grouped parameter definitions |
| `IntoResponses` | derive | a response enum/struct | grouped response definitions |
| `ToResponse` | derive | a type | a reusable response component |

Core rules:

- ALWAYS pin `utoipa`, `utoipa-axum`, and `utoipa-swagger-ui` to explicit,
  independent versions. NEVER assume their version numbers match.
- ALWAYS write `#[utoipa::path]` `path` arguments with the curly-brace
  `{param}` syntax. The macro requires it on BOTH Axum 0.7 and 0.8.
- ALWAYS put the HTTP operation (`get`, `post`, ...) as the FIRST argument of
  `#[utoipa::path]`.
- ALWAYS prefer `utoipa-axum`'s `OpenApiRouter` for a new service: it registers
  each handler once and cannot drift.
- ALWAYS enable the `axum` feature on `utoipa-swagger-ui`; without it
  `SwaggerUi` cannot become an Axum router.
- NEVER hand-write or hand-edit the OpenAPI JSON. The whole point is that the
  spec is generated.

## Decision Trees

### Which integration approach to use

```
Building the OpenAPI integration for an Axum service:
  new service, want routing and spec from one source of truth
      -> utoipa-axum OpenApiRouter + routes!()  (register a handler once)
  existing Router, adding docs incrementally, cannot restructure routing
      -> plain utoipa: #[derive(OpenApi)] with #[openapi(paths(...))]
         (register each handler twice, once routed, once in paths(...))
  need only the spec file, no interactive UI
      -> utoipa alone; serialise with ApiDoc::openapi().to_pretty_json()
```

Plain `utoipa` requires registering every handler twice: once on the Axum
`Router` and once inside `#[openapi(paths(...))]`. Forgetting one half means a
route exists but is undocumented, or a documented operation has no route.
`OpenApiRouter` derives both from a single `.routes(routes!(...))` call, so the
two cannot disagree. ALWAYS pick `OpenApiRouter` for a new service.

### The {param} versus :id path-syntax decision

```
Writing the path string for a documented route:
  the #[utoipa::path] annotation        -> ALWAYS {param}  (e.g. "/pets/{id}")
  the Axum route string on 0.8          -> {param}  -> identical to utoipa
  the Axum route string on 0.7          -> :param   -> deliberately differs
```

`#[utoipa::path]` requires the OpenAPI brace form `{id}` on every Axum version.
Axum 0.8's native route syntax is also `{id}`, so on 0.8 the route string and
the `utoipa::path` annotation are character-for-character identical. On Axum
0.7 the route uses `:id` while the annotation still uses `{id}`; the two
strings differ on purpose. `OpenApiRouter` with `routes!()` removes the issue
entirely: it derives the route from the annotation, so there is one string.

## Patterns

### Pattern: a schema with #[derive(ToSchema)]

`#[derive(ToSchema)]` turns a type into a reusable OpenAPI schema component.
Doc comments become the OpenAPI `description`. Field-level `#[schema(...)]`
attributes refine the generated schema.

```rust
// utoipa 5.x
use utoipa::ToSchema;

#[derive(ToSchema, serde::Serialize)]
struct Pet {
    /// Unique database id of the pet.
    id: u64,
    name: String,
    #[schema(maximum = 30, minimum = 0)]
    age: Option<i32>,
}
```

When both a serde attribute and a `#[schema]` equivalent are present on a
field, the serde attribute takes precedence. Every generic type parameter must
itself implement `ToSchema`.

### Pattern: documenting a handler with #[utoipa::path]

`#[utoipa::path(...)]` documents one endpoint. The HTTP operation MUST be the
first argument. The `path` uses the `{param}` brace syntax.

```rust
// utoipa 5.x - the {param} path syntax is required on Axum 0.7 AND 0.8
#[utoipa::path(
    get,
    path = "/pets/{id}",
    responses(
        (status = 200, description = "Pet found successfully", body = Pet),
        (status = NOT_FOUND, description = "Pet was not found")
    ),
    params(
        ("id" = u64, Path, description = "Pet database id")
    )
)]
async fn get_pet_by_id(/* extractors */) { /* ... */ }
```

`responses(...)` entries are `(status = ..., description = "...", body = ...)`
tuples; `body` is optional. `params(...)` accepts the tuple form shown or an
`IntoParams` struct.

### Pattern: assembling the root document with #[derive(OpenApi)]

`#[derive(OpenApi)]` builds the root OpenAPI document on a marker struct.
`paths(...)` registers annotated handlers; `components(schemas(...))` registers
`ToSchema` types. The derive implements the `OpenApi` trait, whose `openapi()`
function returns the `utoipa::openapi::OpenApi` value.

```rust
// utoipa 5.x
use utoipa::OpenApi;

#[derive(OpenApi)]
#[openapi(
    paths(get_pet_by_id),
    components(schemas(Pet))
)]
struct ApiDoc;

// ApiDoc::openapi() yields the spec; .to_pretty_json() serialises it
println!("{}", ApiDoc::openapi().to_pretty_json().unwrap());
```

This is the plain-`utoipa` approach: `get_pet_by_id` must ALSO be registered on
the Axum `Router` separately. The next pattern removes that duplication.

### Pattern: OpenApiRouter registers a handler once

`utoipa-axum`'s `OpenApiRouter` collects handlers through the `routes!()`
macro, deriving the HTTP method and path from each `#[utoipa::path]`
annotation. `.split_for_parts()` consumes the router and returns the runnable
`axum::Router` plus the finished `OpenApi` value.

```rust
// utoipa-axum 0.2.x - version-sensitive API; pin utoipa-axum = "0.2"
use utoipa_axum::router::OpenApiRouter;
use utoipa_axum::routes;

let (router, api): (axum::Router, utoipa::openapi::OpenApi) =
    OpenApiRouter::new()
        .routes(routes!(get_pet_by_id))
        .split_for_parts();
```

`OpenApiRouter::with_openapi(openapi)` seeds the router with an existing
`OpenApi` value, so top-level `info`, `tags`, and `servers` from a
`#[derive(OpenApi)]` marker struct carry through. `.nest(path, router)` and
`.merge(router)` compose `OpenApiRouter`s exactly like the Axum `Router`
methods while keeping the spec consistent.

### Pattern: serving the Swagger UI

`utoipa-swagger-ui`'s `SwaggerUi` serves the interactive UI and the raw spec
JSON. With the `axum` feature it converts into an Axum router and attaches via
`Router::merge`.

```rust
// utoipa-swagger-ui 9.x - the `axum` feature is mandatory for merge
use utoipa_swagger_ui::SwaggerUi;

let app = Router::new()
    .route("/pets/{id}", get(get_pet_by_id))   // axum 0.8 route string
    .merge(
        SwaggerUi::new("/swagger-ui")
            .url("/api-docs/openapi.json", ApiDoc::openapi()),
    );
```

`SwaggerUi::new("/swagger-ui")` mounts the UI; `.url(json_path, spec)`
registers the spec at a JSON endpoint. On Axum 0.7 the route string is
`"/pets/:id"` while the `utoipa::path` annotation stays `"/pets/{id}"`.

### Pattern: the full three-crate integration

The idiomatic setup builds an `OpenApiRouter`, splits it, and feeds the
produced spec into `SwaggerUi`. Routing, schemas, and the UI all derive from
one registration.

```rust
// utoipa 5.x + utoipa-axum 0.2.x + utoipa-swagger-ui 9.x
use utoipa::OpenApi;
use utoipa_axum::router::OpenApiRouter;
use utoipa_axum::routes;
use utoipa_swagger_ui::SwaggerUi;

#[derive(OpenApi)]
#[openapi(components(schemas(Pet)))]
struct ApiDoc;

let (router, api) = OpenApiRouter::with_openapi(ApiDoc::openapi())
    .routes(routes!(get_pet_by_id))
    .split_for_parts();

let app = router.merge(
    SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json", api),
);
```

`get_pet_by_id` is registered exactly once, in `routes!()`. Its route and its
spec operation are both derived from the single `#[utoipa::path]` annotation,
so they cannot drift.

## Reference Links

- `references/methods.md`: the full `utoipa` macro surface, every verified
  `#[utoipa::path]` and `#[schema]` argument, the `OpenApi` trait, the
  `OpenApiRouter` method list with signatures, `SwaggerUi`, and the feature
  flags of all three crates.
- `references/examples.md`: a complete `Cargo.toml`, schema and handler
  annotations, the plain-`utoipa` assembly, the `OpenApiRouter` full
  integration, `IntoParams` for query structs, and serving the raw spec.
- `references/anti-patterns.md`: real mistakes with root-cause analysis,
  including hand-editing the spec, the route missing from `paths(...)`, the
  `:id` versus `{id}` mismatch, mismatched crate versions, the missing `axum`
  feature, and the operation argument placed after `path`.

Related skills: `axum-core-router` (`{id}` versus `:id` route syntax across
versions, `.nest` and `.merge`), `axum-syntax-handlers` (the handler signatures
that `#[utoipa::path]` annotates), `axum-core-version-migration` (the 0.7-to-0.8
route-syntax change).
