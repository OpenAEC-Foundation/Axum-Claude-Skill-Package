# axum-impl-openapi : Methods and API Signatures

Complete API reference for generating an OpenAPI 3 spec and Swagger UI for an
Axum service. All signatures verified via WebFetch against `docs.rs` on
2026-05-20. Crate versions: `utoipa` 5.5.0, `utoipa-swagger-ui` 9.0.2,
`utoipa-axum` 0.2.0. The `utoipa-axum` surface is version-sensitive (pre-1.0).

## Crate setup

```toml
# All three version independently; pin each explicitly.
utoipa = { version = "5", features = ["axum_extras"] }
utoipa-axum = "0.2"
utoipa-swagger-ui = { version = "9", features = ["axum"] }
```

### Feature flags

| Crate | Flag | Effect |
|-------|------|--------|
| `utoipa` | `macros` (default) | the derive and attribute macros |
| `utoipa` | `axum_extras` | better type inference for Axum extractors |
| `utoipa` | `yaml` | `to_yaml()` serialisation |
| `utoipa` | `chrono` / `time` / `uuid` / `url` | schema support for those types |
| `utoipa-swagger-ui` | `axum` | `SwaggerUi` becomes an Axum router |
| `utoipa-swagger-ui` | `actix-web` / `rocket` | other framework integrations |
| `utoipa-axum` | `debug` | `Debug` impls on `utoipa-axum` types |

## The utoipa macro surface

`utoipa` 5.5.0 provides five derive macros and one attribute macro:

- Derive: `ToSchema`, `OpenApi`, `IntoParams`, `IntoResponses`, `ToResponse`.
- Attribute: `#[utoipa::path(...)]`.

### #[derive(ToSchema)]

Generates a reusable OpenAPI schema component and implements the `ToSchema`
trait. Requires the `macros` feature (default). Doc comments become the
OpenAPI `description`.

Container-level `#[schema(...)]` attributes (verified):

| Attribute | Effect |
|-----------|--------|
| `description` | override the description |
| `example` / `examples` | attach example values |
| `title` | schema title |
| `as` | alternative schema path, e.g. `as = path::to::Pet` |
| `rename_all` | rename-all casing |
| `default` | default value |
| `deprecated` | mark the schema deprecated |
| `no_recursion` | break a recursive type tree |
| `bound` | override generic trait bounds |

Field-level `#[schema(...)]` attributes (verified):

| Attribute | Effect |
|-----------|--------|
| `value_type` | override the inferred OpenAPI type |
| `format` | OpenAPI format string |
| `inline` | inline the schema instead of referencing it |
| `read_only` / `write_only` | direction-restricted field |
| `required` | force the field required |
| `nullable` | allow null |
| `pattern` | ECMA-262 regex constraint |
| `max_length` / `min_length` | string length bounds |
| `maximum` / `minimum` / `exclusive_*` | numeric bounds |
| `max_items` / `min_items` | collection size bounds |
| `schema_with` | point to a custom schema function |

`ToSchema` respects a subset of serde attributes: `rename_all`, `rename`,
`skip`, `skip_serializing`, `tag`, `untagged`, `default`, `flatten`,
`deny_unknown_fields`. When a serde attribute and a `#[schema]` equivalent
both appear, the serde attribute takes precedence.

### #[utoipa::path(...)]

Documents one endpoint. The HTTP operation MUST be the first argument: one of
`get, post, put, delete, head, options, patch, trace`.

Verified arguments:

| Argument | Form |
|----------|------|
| `path` | `path = "/pets/{id}"` ; OpenAPI brace `{param}` syntax, required |
| `responses` | `responses((status = ..., description = "...", body = ..., content_type = "..."))` |
| `params` | tuple `("name" = Type, Path|Query, ...)` OR `params(MyIntoParamsStruct)` |
| `request_body` | `request_body = Type` OR `request_body(content = ..., description = ..., example = ...)` |
| `operation_id` | a custom operation id |
| `tag` / `tags` | group the operation |
| `context_path` | a path prefix prepended to `path` |
| `security` | security requirements |
| `description` | operation description |

`status` accepts an integer (`200`), a named constant (`NOT_FOUND`), a
status-class range string (`"4XX"`), or `"default"`. `body` is optional;
`content_type` defaults to `text/plain` for primitives and `application/json`
for structs.

### #[derive(OpenApi)]

Assembles the root OpenAPI document on a marker struct. The `#[openapi(...)]`
attribute accepts `paths(...)`, `components(schemas(...))`, `info`, `tags`,
`servers`, `security`, and `modifiers`. The derive implements the `OpenApi`
trait.

```rust
// utoipa 5.x
pub trait OpenApi {
    fn openapi() -> utoipa::openapi::OpenApi;
}
```

The returned `utoipa::openapi::OpenApi` value serialises with:

- `to_json()` / `to_pretty_json()` : `Result<String, _>` JSON output.
- `to_yaml()` : YAML output (requires the `yaml` feature).

### #[derive(IntoParams)]

Generates grouped path/query parameter definitions from a struct. Used as
`params(MyParameters)` inside `#[utoipa::path]`.

### #[derive(IntoResponses)] and #[derive(ToResponse)]

`IntoResponses` derives grouped response definitions from an enum or struct.
`ToResponse` derives a reusable response component, parallel to how `ToSchema`
derives a reusable schema component.

## utoipa-swagger-ui : SwaggerUi

`utoipa-swagger-ui` 9.0.2 bundles the Swagger UI static assets.

| Method | Signature | Effect |
|--------|-----------|--------|
| `SwaggerUi::new` | `SwaggerUi::new(path)` | mount the UI at `path` |
| `.url` | `.url(json_path, openapi)` | register a spec at a JSON endpoint |

With the `axum` feature, `SwaggerUi` converts into an Axum router and attaches
through the standard `Router::merge`. Without the `axum` feature it cannot
become an Axum router.

## utoipa-axum : OpenApiRouter

`utoipa-axum` 0.2.0. All methods are on `OpenApiRouter<S>` where
`S: Send + Sync + Clone + 'static`. Verified signatures:

| Method | Signature |
|--------|-----------|
| `new` | `OpenApiRouter::new() -> Self` (info from cargo env vars) |
| `with_openapi` | `with_openapi(openapi: OpenApi) -> Self` |
| `routes` | `routes(self, UtoipaMethodRouter<S>) -> Self` (fed by `routes!()`) |
| `route` | `route(self, path: &str, method_router: MethodRouter<S>) -> Self` |
| `nest` | `nest(self, path: &str, router: OpenApiRouter<S>) -> Self` |
| `merge` | `merge(self, router: OpenApiRouter<S>) -> Self` |
| `layer` | `layer<L>(self, layer: L) -> Self` |
| `split_for_parts` | `split_for_parts(self) -> (Router<S>, OpenApi)` |
| `to_openapi` | `to_openapi(&mut self) -> OpenApi` |
| `into_openapi` | `into_openapi(self) -> OpenApi` |

### The routes!() macro

`routes!(handler_a, handler_b, ...)` collects Axum handlers annotated with
`#[utoipa::path]` and produces the `UtoipaMethodRouter` value that `.routes()`
consumes. The HTTP method and path are derived from each handler's
`#[utoipa::path]` annotation, so no separate `get(...)` wrapper or route
string is needed.

### split_for_parts

`.split_for_parts()` consumes the `OpenApiRouter` and returns the tuple
`(axum::Router, utoipa::openapi::OpenApi)`: the runnable router and the
finished spec. Feed the `OpenApi` half into `SwaggerUi::url(...)`.

## Version-sensitivity warning

`utoipa-axum` is pre-1.0 (0.2.0). Its `OpenApiRouter` surface, the `routes!`
macro, and `split_for_parts` may change names or signatures in a future `0.x`
release. ALWAYS pin `utoipa-axum = "0.2"` exactly and re-verify against
`docs.rs` after a version bump. `utoipa` 5.x and `utoipa-swagger-ui` 9.x are
post-1.0 and more stable, but still version independently.

## Sources

All verified via WebFetch on 2026-05-20.

- https://docs.rs/utoipa/latest/utoipa/ : `utoipa` 5.5.0, the macro surface,
  the `OpenApi` trait, `to_pretty_json()`, feature flags.
- https://docs.rs/utoipa/latest/utoipa/derive.ToSchema.html : container and
  field `#[schema(...)]` attributes, serde-attribute precedence.
- https://docs.rs/utoipa/latest/utoipa/attr.path.html : operation-first rule,
  `{param}` brace syntax, `responses`/`params`/`request_body` forms.
- https://docs.rs/utoipa-swagger-ui/latest/utoipa_swagger_ui/ :
  `utoipa-swagger-ui` 9.0.2, `SwaggerUi::new()`, `.url()`, the `axum` feature.
- https://docs.rs/utoipa-axum/0.2.0/utoipa_axum/router/struct.OpenApiRouter.html :
  `utoipa-axum` 0.2.0, every `OpenApiRouter` method signature, `routes!()`,
  `split_for_parts()` return tuple, the `debug` feature flag.
