# axum-impl-openapi : Examples

Working, version-annotated code for OpenAPI generation in Axum. Crate versions:
`utoipa` 5.5.0, `utoipa-swagger-ui` 9.0.2, `utoipa-axum` 0.2.0 (verified
2026-05-20). The `utoipa-axum` API is version-sensitive (pre-1.0).

## Example 1 : Cargo.toml

```toml
[dependencies]
axum = "0.8"                 # or "0.7"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }

# OpenAPI stack - three independently-versioned crates, all pinned.
utoipa = { version = "5", features = ["axum_extras"] }
utoipa-axum = "0.2"
utoipa-swagger-ui = { version = "9", features = ["axum"] }
```

The three version numbers (`5`, `0.2`, `9`) are NOT a typo and NOT
synchronised. Each crate releases on its own schedule.

## Example 2 : a schema with #[derive(ToSchema)]

```rust
// utoipa 5.x
use utoipa::ToSchema;

#[derive(ToSchema, serde::Serialize, serde::Deserialize)]
struct Pet {
    /// Unique database id of the pet.
    id: u64,
    /// Display name of the pet.
    name: String,
    #[schema(maximum = 30, minimum = 0)]
    age: Option<i32>,
}

// A generic type: every type parameter must implement ToSchema.
#[derive(ToSchema, serde::Serialize)]
#[schema(as = page::PetPage)]
struct Page<T> {
    items: Vec<T>,
    total: u64,
}
```

## Example 3 : documenting handlers with #[utoipa::path]

```rust
// utoipa 5.x - the {param} brace syntax is required on Axum 0.7 AND 0.8.
use axum::extract::Path;
use axum::http::StatusCode;
use axum::Json;

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
async fn get_pet_by_id(Path(id): Path<u64>) -> Result<Json<Pet>, StatusCode> {
    if id == 1 {
        Ok(Json(Pet { id, name: "Rex".into(), age: Some(3) }))
    } else {
        Err(StatusCode::NOT_FOUND)
    }
}

#[utoipa::path(
    post,
    path = "/pets",
    request_body = Pet,
    responses(
        (status = 201, description = "Pet created", body = Pet)
    )
)]
async fn create_pet(Json(pet): Json<Pet>) -> (StatusCode, Json<Pet>) {
    (StatusCode::CREATED, Json(pet))
}
```

## Example 4 : the plain-utoipa assembly (register twice)

Without `utoipa-axum`, each handler is registered twice: once on the Axum
`Router`, once in `#[openapi(paths(...))]`. Keep the two lists in sync by hand.

```rust
// utoipa 5.x + axum 0.8
use axum::{routing::{get, post}, Router};
use utoipa::OpenApi;

#[derive(OpenApi)]
#[openapi(
    paths(get_pet_by_id, create_pet),       // must list every documented handler
    components(schemas(Pet))
)]
struct ApiDoc;

fn app() -> Router {
    Router::new()
        .route("/pets/{id}", get(get_pet_by_id))   // 0.8 route string
        .route("/pets", post(create_pet))
    // ApiDoc::openapi() now produces the spec for the routes above.
}
```

On Axum 0.7 the route strings are `"/pets/:id"` and `"/pets"`; the
`#[utoipa::path]` annotations stay `"/pets/{id}"` and `"/pets"`.

## Example 5 : serving the raw spec JSON

```rust
// utoipa 5.x - serialise the spec to a string or an endpoint
use axum::{routing::get, Json, Router};
use utoipa::OpenApi;

// As a string (build script, file output, test assertion):
let spec: String = ApiDoc::openapi().to_pretty_json().unwrap();

// As an Axum endpoint returning the spec as JSON:
async fn openapi_json() -> Json<utoipa::openapi::OpenApi> {
    Json(ApiDoc::openapi())
}

let app = Router::new().route("/api-docs/openapi.json", get(openapi_json));
```

## Example 6 : the OpenApiRouter full integration (register once)

This is the recommended setup. `routes!()` derives the route from each
`#[utoipa::path]` annotation, so a handler is registered exactly once.

```rust
// utoipa 5.x + utoipa-axum 0.2.x + utoipa-swagger-ui 9.x
// version-sensitive: utoipa-axum API may change before 1.0
use axum::Router;
use utoipa::OpenApi;
use utoipa_axum::router::OpenApiRouter;
use utoipa_axum::routes;
use utoipa_swagger_ui::SwaggerUi;

#[derive(OpenApi)]
#[openapi(
    info(title = "Pet Store API", version = "1.0.0"),
    components(schemas(Pet))
)]
struct ApiDoc;

fn build_app() -> Router {
    // 1. Build the OpenApiRouter, seeded with top-level info from ApiDoc.
    let (router, api) = OpenApiRouter::with_openapi(ApiDoc::openapi())
        .routes(routes!(get_pet_by_id))     // route derived from the annotation
        .routes(routes!(create_pet))
        .split_for_parts();                 // -> (axum::Router, OpenApi)

    // 2. Merge the Swagger UI, fed with the spec produced by split_for_parts.
    router.merge(
        SwaggerUi::new("/swagger-ui")
            .url("/api-docs/openapi.json", api),
    )
}

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    // axum 0.8: axum::serve. UI at /swagger-ui, spec at /api-docs/openapi.json
    axum::serve(listener, build_app()).await.unwrap();
}
```

`get_pet_by_id` and `create_pet` each appear once, inside `routes!()`. Their
routes and their spec operations are both derived from the single
`#[utoipa::path]` annotation, so the spec cannot drift from the routing.

## Example 7 : composing OpenApiRouters with nest

`OpenApiRouter` mirrors the Axum `Router` `.nest()` and `.merge()` methods
while keeping the spec consistent across the composition.

```rust
// utoipa-axum 0.2.x - version-sensitive
use utoipa_axum::router::OpenApiRouter;
use utoipa_axum::routes;

fn pets_router() -> OpenApiRouter {
    OpenApiRouter::new()
        .routes(routes!(get_pet_by_id))
        .routes(routes!(create_pet))
}

fn api() -> OpenApiRouter {
    OpenApiRouter::new()
        .nest("/v1", pets_router())     // routes become /v1/pets/{id}, /v1/pets
}
```

## Example 8 : IntoParams for a query struct

`#[derive(IntoParams)]` groups query parameters into one struct, referenced in
`#[utoipa::path]` via `params(StructName)`.

```rust
// utoipa 5.x
use utoipa::IntoParams;

#[derive(IntoParams, serde::Deserialize)]
struct ListQuery {
    /// Maximum number of pets to return.
    limit: Option<u32>,
    /// Number of pets to skip.
    offset: Option<u32>,
}

#[utoipa::path(
    get,
    path = "/pets",
    params(ListQuery),
    responses((status = 200, description = "Pet list", body = [Pet]))
)]
async fn list_pets(/* axum::extract::Query<ListQuery> */) { /* ... */ }
```

## Migration note : 0.7 to 0.8

The `utoipa` annotations do NOT change between Axum 0.7 and 0.8: `path` is
always `{param}`. Only the Axum route strings change (`:id` to `{id}`). The
`OpenApiRouter` approach in Example 6 sidesteps the change entirely, because
the route string is derived from the annotation and never written separately.
