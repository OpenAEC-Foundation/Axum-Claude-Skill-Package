# axum-impl-openapi : Anti-Patterns

Real mistakes in Axum OpenAPI integration, each with a root-cause analysis and
the correct form. Crate versions: `utoipa` 5.5.0, `utoipa-swagger-ui` 9.0.2,
`utoipa-axum` 0.2.0.

## AP-1 : hand-writing or hand-editing the OpenAPI spec

### Wrong

```rust
// WRONG - a hand-maintained spec string.
const OPENAPI_JSON: &str = r#"{
  "openapi": "3.0.0",
  "paths": { "/pets/{id}": { "get": { "responses": { "200": { ... } } } } }
}"#;
```

### Why it fails

A hand-written spec is a second source of truth that drifts the moment a
handler changes. A field renamed on `Pet`, a new `404` branch, a route moved:
none of it propagates to the JSON. Clients then generate stubs from a spec that
no longer matches the server. Documentation drift is the exact problem
`utoipa` exists to eliminate.

### Correct

Annotate the types and handlers and let `utoipa` generate the spec at compile
time. The spec is then derived from the same code the server runs and cannot
disagree with it. NEVER edit the generated JSON.

## AP-2 : a route registered but missing from paths(...)

### Wrong

```rust
// WRONG - create_pet is routed but not listed in #[openapi(paths(...))].
#[derive(OpenApi)]
#[openapi(paths(get_pet_by_id), components(schemas(Pet)))]
struct ApiDoc;

let app = Router::new()
    .route("/pets/{id}", get(get_pet_by_id))
    .route("/pets", post(create_pet));        // routed, but undocumented
```

### Why it fails

Plain `utoipa` requires registering each handler twice: once on the `Router`
and once in `#[openapi(paths(...))]`. The two lists are maintained by hand and
nothing checks they agree. Here `create_pet` works as an endpoint but never
appears in the spec, so the Swagger UI does not show it and generated clients
cannot call it. The reverse mistake, a handler in `paths(...)` with no route,
documents an endpoint that returns `404`.

### Correct

Use `utoipa-axum`'s `OpenApiRouter` with `routes!()`. It registers a handler
once and derives both the route and the spec entry from the single
`#[utoipa::path]` annotation, so the two cannot diverge.

```rust
// CORRECT - one registration per handler
let (router, api) = OpenApiRouter::new()
    .routes(routes!(get_pet_by_id))
    .routes(routes!(create_pet))
    .split_for_parts();
```

## AP-3 : using :id syntax in #[utoipa::path]

### Wrong

```rust
// WRONG - #[utoipa::path] does not accept the colon syntax.
#[utoipa::path(get, path = "/pets/:id", responses((status = 200)))]
async fn get_pet_by_id(/* ... */) { /* ... */ }
```

### Why it fails

`#[utoipa::path]` produces an OpenAPI document, and OpenAPI path templating
uses the curly-brace `{id}` form on every framework. The colon form `:id` is
Axum 0.7's route syntax, not an OpenAPI form. A `:id` annotation yields a spec
with a literal `/pets/:id` path, so the documented path does not match the
actual parameterised route and tooling treats `:id` as a static segment.

### Correct

ALWAYS write `path = "/pets/{id}"` in `#[utoipa::path]`, on Axum 0.7 AND 0.8.
On Axum 0.8 the route string matches it exactly. On Axum 0.7 the route string
is `"/pets/:id"` while the annotation stays `"/pets/{id}"`; the two differ on
purpose.

## AP-4 : assuming the three crate versions match

### Wrong

```toml
# WRONG - utoipa-axum and utoipa-swagger-ui have no version 5.
utoipa = "5"
utoipa-axum = "5"
utoipa-swagger-ui = "5"
```

### Why it fails

`utoipa`, `utoipa-axum`, and `utoipa-swagger-ui` are separate crates that
release independently. At the 2026-05-20 verification they were `utoipa` 5.5.0,
`utoipa-axum` 0.2.0, and `utoipa-swagger-ui` 9.0.2. Pinning all three to one
number either fails to resolve or pulls an unintended release with a different
API.

### Correct

```toml
# CORRECT - each crate pinned to its own line
utoipa = { version = "5", features = ["axum_extras"] }
utoipa-axum = "0.2"
utoipa-swagger-ui = { version = "9", features = ["axum"] }
```

## AP-5 : omitting the axum feature on utoipa-swagger-ui

### Wrong

```toml
# WRONG - no `axum` feature.
utoipa-swagger-ui = "9"
```

```rust
let app = Router::new().merge(SwaggerUi::new("/swagger-ui").url(/* ... */));
// error: SwaggerUi does not convert into an Axum router
```

### Why it fails

`utoipa-swagger-ui` supports several web frameworks behind feature flags
(`axum`, `actix-web`, `rocket`). Without the `axum` feature, `SwaggerUi` does
not implement the conversion that `Router::merge` needs, so the merge fails to
compile. The Swagger UI assets are present but unreachable.

### Correct

```toml
# CORRECT - the axum feature enables the Router::merge integration
utoipa-swagger-ui = { version = "9", features = ["axum"] }
```

## AP-6 : the HTTP operation placed after path

### Wrong

```rust
// WRONG - the operation is not the first argument.
#[utoipa::path(
    path = "/pets/{id}",
    get,
    responses((status = 200, body = Pet))
)]
async fn get_pet_by_id(/* ... */) { /* ... */ }
```

### Why it fails

`#[utoipa::path]` requires the HTTP operation (`get`, `post`, `put`, `delete`,
`head`, `options`, `patch`, `trace`) as the FIRST argument. The macro parses it
positionally; placing `path` first is a macro-expansion error, not a runtime
one.

### Correct

```rust
// CORRECT - operation first, then path, then the rest
#[utoipa::path(
    get,
    path = "/pets/{id}",
    responses((status = 200, body = Pet))
)]
async fn get_pet_by_id(/* ... */) { /* ... */ }
```

## AP-7 : a body type that does not derive ToSchema

### Wrong

```rust
// WRONG - Pet has no #[derive(ToSchema)].
#[derive(serde::Serialize)]
struct Pet { id: u64, name: String }

#[utoipa::path(get, path = "/pets/{id}", responses((status = 200, body = Pet)))]
async fn get_pet_by_id(/* ... */) { /* ... */ }
```

### Why it fails

`body = Pet` and `components(schemas(Pet))` both require `Pet` to implement
`ToSchema`. A type that only derives `Serialize` has no schema component, so
the macro cannot describe the response body and the build fails. `Serialize`
controls JSON output; `ToSchema` controls the OpenAPI schema. They are separate
and both are needed.

### Correct

```rust
// CORRECT - derive ToSchema alongside Serialize
#[derive(utoipa::ToSchema, serde::Serialize)]
struct Pet { id: u64, name: String }
```

Then register it: `#[openapi(components(schemas(Pet)))]`, or with
`OpenApiRouter`, seed it through `with_openapi(ApiDoc::openapi())`.

## AP-8 : forgetting split_for_parts before serving

### Wrong

```rust
// WRONG - an OpenApiRouter is not an axum::Router.
let router = OpenApiRouter::new().routes(routes!(get_pet_by_id));
axum::serve(listener, router).await.unwrap();   // type error
```

### Why it fails

`OpenApiRouter` is a `utoipa-axum` type, not an `axum::Router`. `axum::serve`
needs an `axum::Router`. `OpenApiRouter` must be consumed by
`.split_for_parts()`, which returns the `(axum::Router, OpenApi)` tuple: the
runnable router and the spec.

### Correct

```rust
// CORRECT - split, then serve the axum::Router half
let (router, api) = OpenApiRouter::new()
    .routes(routes!(get_pet_by_id))
    .split_for_parts();

let app = router.merge(
    SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json", api),
);
axum::serve(listener, app).await.unwrap();
```

## Sources

All verified via WebFetch on 2026-05-20.

- https://docs.rs/utoipa/latest/utoipa/ : `utoipa` 5.5.0, the operation-first
  rule, `ToSchema` requirement for `body` and `schemas(...)`.
- https://docs.rs/utoipa/latest/utoipa/attr.path.html : `{param}` brace syntax,
  argument order.
- https://docs.rs/utoipa-swagger-ui/latest/utoipa_swagger_ui/ : the `axum`
  feature requirement for `Router::merge`.
- https://docs.rs/utoipa-axum/0.2.0/utoipa_axum/router/struct.OpenApiRouter.html :
  `OpenApiRouter`, `routes!()`, `split_for_parts()` returning
  `(axum::Router, OpenApi)`.
