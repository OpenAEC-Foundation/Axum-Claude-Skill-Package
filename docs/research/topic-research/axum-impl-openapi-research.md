# Topic Research : axum-impl-openapi

Verified research file for the Axum skill package skill `axum-impl-openapi`.
All API claims, macro arguments, and code examples below were verified via
WebFetch against the official `docs.rs` documentation on **2026-05-20**.
Target Axum versions: **0.7** and **0.8**.

## Crate versions (verified 2026-05-20)

| Crate | Version on docs.rs `latest` | Role |
|-------|------------------------------|------|
| `utoipa` | **5.5.0** | OpenAPI 3.x spec generation from annotations |
| `utoipa-swagger-ui` | **9.0.2** | Serves the interactive Swagger UI |
| `utoipa-axum` | **0.2.0** | `OpenApiRouter` : unifies routing + spec |

These three crates version independently. `utoipa-axum 0.2.0` and
`utoipa-swagger-ui 9.0.2` both depend on the `utoipa` 5.x line. A skill MUST
pin all three explicitly in `Cargo.toml` because their version numbers are NOT
synchronised, and a `utoipa` major bump (4.x to 5.x) changed macro behaviour.
Treat the entire `utoipa` API surface as version-sensitive : the examples here
are valid for the 5.x line.

## Why utoipa exists

The core problem `utoipa` solves is documentation drift : a hand-written
OpenAPI spec inevitably diverges from the actual handler code. `utoipa`
generates the OpenAPI 3 document from Rust annotations at compile time, so the
spec is always derived from the same types and handlers the server runs.

## The utoipa macro surface

`utoipa` (5.5.0) provides five derive macros and one attribute macro:

- Derive macros : `ToSchema`, `OpenApi`, `IntoParams`, `IntoResponses`,
  `ToResponse`.
- Attribute macro : `#[utoipa::path(...)]`.

The three load-bearing pieces for a typical skill are `ToSchema`,
`#[utoipa::path]`, and `OpenApi`.

### `#[derive(ToSchema)]`

`#[derive(ToSchema)]` generates a reusable OpenAPI schema component for a Rust
type and implements the `ToSchema` trait. It requires the `macros` feature
(enabled by default). Doc comments on the struct and its fields become the
OpenAPI `description` automatically.

Container-level `#[schema(...)]` attributes verified from the docs include
`description`, `example` / `examples`, `title`, `as` (alternative schema path,
e.g. `as = path::to::Pet`), `rename_all`, `default`, `deprecated`,
`no_recursion` (breaks recursive type trees), and `bound` (override generic
trait bounds).

Field-level `#[schema(...)]` attributes include `value_type` (override the
inferred OpenAPI type for third-party types), `format`, `inline`,
`read_only` / `write_only`, `required`, `nullable`, `pattern` (ECMA-262 regex),
`max_length` / `min_length`, `maximum` / `minimum` / `exclusive_*`,
`max_items` / `min_items`, and `schema_with` (point to a custom schema
function).

`ToSchema` respects a subset of serde attributes : `rename_all`, `rename`,
`skip`, `skip_serializing`, `tag`, `untagged`, `default`, `flatten`,
`deny_unknown_fields`. When both a serde attribute and a `#[schema]` equivalent
are present, the **serde attribute takes precedence**. Generics are fully
supported : every type parameter must implement `ToSchema` (or use `bound` to
override), and a generic type is named for OpenAPI with `as`.

```rust
use utoipa::ToSchema;

#[derive(ToSchema, serde::Serialize)]
struct Pet {
    /// Unique database id of the pet.
    id: u64,
    name: String,
    #[schema(maximum = 30, minimum = 0)]
    age: Option<i32>,
}

#[derive(ToSchema)]
#[schema(as = path::to::PageList)]
struct Page<T> {
    items: Vec<T>,
}
```

Enums are handled in three shapes : unit-variant enums, mixed enums (unit +
named + unnamed variants, with `discriminator` support), and `#[repr(u*/i*)]`
field-less enums.

### `#[utoipa::path(...)]`

`#[utoipa::path(...)]` documents one endpoint. The HTTP operation MUST be the
first argument : one of `get, post, put, delete, head, options, patch, trace`.

Verified arguments :

- `path = "..."` : an OpenAPI-format path string. **Path arguments use the
  curly-brace `{param}` syntax** (e.g. `path = "/pets/{id}"`). This matches
  Axum 0.8's native route syntax exactly, so a 0.8 route string and its
  `utoipa::path` annotation are character-for-character identical. On Axum 0.7
  the route uses `:id` while `utoipa` still wants `{id}`, so on 0.7 the two
  strings deliberately differ.
- `responses(...)` : each entry is `(status = ..., description = "...",
  body = ..., content_type = "...")`. `status` accepts an integer
  (`200`), a status-class range string (`"4XX"`), or `"default"`. `body` is
  optional; `content_type` defaults to `text/plain` for primitives and
  `application/json` for structs.
- `params(...)` : tuple form `("name" = ParameterType, ParameterIn, ...)`
  where `ParameterIn` is `Path` or `Query`; or `IntoParams` form
  `params(MyParameters)` where `MyParameters` derives `IntoParams`. Both forms
  can be combined.
- `request_body` : simple `request_body = Type`, or advanced
  `request_body(content = ..., description = "...", example = ...)` with
  multi-content-type support.
- `operation_id`, `tag` / `tags`, `context_path`, `security`, `description`.

```rust
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

### `#[derive(OpenApi)]`

`#[derive(OpenApi)]` assembles the root OpenAPI document on a marker struct.
The `#[openapi(...)]` attribute registers `paths(...)` (the annotated
handlers) and `components(schemas(...))` (the `ToSchema` types). It also
accepts `info`, `tags`, `servers`, `security`, and `modifiers`. The derive
implements the `OpenApi` trait, whose associated function `openapi()` returns
the constructed `utoipa::openapi::OpenApi` value. That value can be serialised
with `to_json()` / `to_pretty_json()` (or `to_yaml()` with the `yaml` feature).

```rust
use utoipa::OpenApi;

#[derive(OpenApi)]
#[openapi(
    paths(get_pet_by_id),
    components(schemas(Pet))
)]
struct ApiDoc;

// ApiDoc::openapi() yields the spec value
println!("{}", ApiDoc::openapi().to_pretty_json().unwrap());
```

Relevant `utoipa` feature flags : `macros` (default), `yaml`, `axum_extras`,
`actix_extras`, `rocket_extras`, plus type-support flags `chrono`, `time`,
`uuid`, `url`. `axum_extras` improves type inference for Axum extractors in
`#[utoipa::path]` annotations.

## utoipa-swagger-ui : serving the interactive UI

`utoipa-swagger-ui` (9.0.2) bundles the Swagger UI static assets and exposes
the `SwaggerUi` struct, a builder-style entry point for serving the UI plus the
api-doc JSON. `SwaggerUi::new(path)` takes the mount path; `.url(json_path,
spec)` registers an OpenAPI document at a JSON endpoint.

The crate has framework feature flags : `axum`, `actix-web`, `rocket`. With the
`axum` feature, `SwaggerUi` implements conversion into an Axum router, so it is
attached with the standard `Router::merge` method :

```rust
use utoipa_swagger_ui::SwaggerUi;

let app = Router::new()
    .route("/pets/{id}", get(get_pet_by_id))
    .merge(
        SwaggerUi::new("/swagger-ui")
            .url("/api-docs/openapi.json", ApiDoc::openapi()),
    );
```

This serves the UI at `/swagger-ui` and the raw spec at
`/api-docs/openapi.json`. The `axum` feature flag is mandatory for the `merge`
integration; without it, `SwaggerUi` cannot be turned into an Axum router.

## utoipa-axum : OpenApiRouter unifies routing and spec

`utoipa-axum` (0.2.0) solves the remaining drift risk in the plain `utoipa` +
`utoipa-swagger-ui` setup : even with annotations, you must still register each
handler twice, once on the Axum `Router` and once in `#[openapi(paths(...))]`.
Forgetting one half means the route exists but is undocumented, or vice versa.

`OpenApiRouter` registers a handler **once** and derives both the route and the
spec entry from it.

- `OpenApiRouter::new()` : a fresh router with an empty spec.
- `OpenApiRouter::with_openapi(openapi)` : a router seeded with an existing
  `OpenApi` value (use this to attach top-level `info`, `tags`, `servers` from
  a `#[derive(OpenApi)]` marker struct).
- `.routes(routes!(handler_a, handler_b))` : the `routes!()` macro collects
  Axum handlers annotated with `#[utoipa::path]`, deriving HTTP method and path
  directly from each `#[utoipa::path]` annotation, so no separate `get(...)` /
  route-string is needed.
- `.nest(path, other_router)` and `.merge(other_router)` : compose
  `OpenApiRouter`s, mirroring the Axum `Router` methods, while keeping the spec
  consistent.
- `.split_for_parts()` : consumes the `OpenApiRouter` and returns the tuple
  `(axum::Router, utoipa::openapi::OpenApi)`, i.e. the runnable router and the
  finished spec.

```rust
use utoipa_axum::router::OpenApiRouter;
use utoipa_axum::routes;

let (router, api): (axum::Router, utoipa::openapi::OpenApi) =
    OpenApiRouter::new()
        .routes(routes!(get_pet_by_id))
        .split_for_parts();
```

The idiomatic full integration combines all three crates : build the
`OpenApiRouter`, split it, then feed the produced `OpenApi` value into
`SwaggerUi`.

```rust
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

`utoipa-axum` has a `debug` feature flag that enables `Debug` impls on its
types. Because `utoipa-axum` is still on a `0.x` version (0.2.0), its API is
the most version-sensitive of the three : a future `0.x` bump may rename
`routes!`, `split_for_parts`, or the `OpenApiRouter` constructors. A skill MUST
state the pinned `utoipa-axum = "0.2"` version and warn that the
`OpenApiRouter` surface can change before 1.0.

## Sources verified

All URLs fetched and verified on **2026-05-20** :

- https://docs.rs/utoipa/latest/utoipa/ — utoipa 5.5.0; derive macros
  `ToSchema`, `OpenApi`, `IntoParams`, `IntoResponses`, `ToResponse`; attribute
  macro `#[utoipa::path]`; `#[openapi(paths(...))]`; `OpenApi::openapi()`;
  `to_pretty_json()`; feature flags (`macros`, `yaml`, `axum_extras`, etc.).
- https://docs.rs/utoipa/latest/utoipa/derive.ToSchema.html — utoipa 5.5.0;
  container and field `#[schema(...)]` attributes; serde-attribute precedence;
  enum shapes; generics with `as` and `bound`.
- https://docs.rs/utoipa/latest/utoipa/attr.path.html — utoipa 5.5.0;
  operation-first rule; `{param}` brace path syntax; `responses(...)` tuple
  shape; `params(...)` tuple and `IntoParams` forms; `request_body`,
  `operation_id`, `tag`, `context_path`.
- https://docs.rs/utoipa-swagger-ui/latest/utoipa_swagger_ui/ — utoipa-swagger-ui
  9.0.2; `SwaggerUi::new()`, `.url()`; `Router::merge` axum integration;
  `axum` / `actix-web` / `rocket` feature flags.
- https://docs.rs/utoipa-axum/latest/utoipa_axum/ — utoipa-axum 0.2.0;
  `OpenApiRouter::new()` / `with_openapi()`; `.routes()` with `routes!()`
  macro; `.nest()` / `.merge()`; `.split_for_parts()` returning
  `(axum::Router, OpenApi)`; `debug` feature flag.
