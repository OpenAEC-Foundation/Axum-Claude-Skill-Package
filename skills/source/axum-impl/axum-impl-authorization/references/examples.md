# examples.md : axum-impl-authorization

Working, version-annotated examples for Axum authorization. Every API call was
verified against the official `docs.rs` pages on 2026-05-20 (axum 0.8.9). The
middleware and routing APIs are identical in Axum 0.7 and 0.8; divergent code
is annotated `// axum 0.8` and `// axum 0.7`.

Each example assumes a `Claims` type defined by `axum-impl-auth-jwt`:

```rust
// Defined and verified in axum-impl-auth-jwt. Shown here for context only.
#[derive(Clone)]
pub struct Claims {
    pub sub: String,                       // user id
    pub role: String,                      // coarse role label
    pub permissions: std::collections::HashSet<String>, // fine-grained capabilities
}
```

`Claims` is a `FromRequestParts` extractor: it verifies the JWT signature and
`exp` expiry, and rejects a missing or invalid token with `401` before any
guard body runs.

## Example 1: role-based guard protecting an /admin group

```rust
// Cargo.toml: axum = "0.8", tokio = { version = "1", features = ["full"] }

use axum::{
    extract::Request,
    http::StatusCode,
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};

async fn require_admin(
    claims: Claims,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    if claims.role != "admin" {
        return Err(StatusCode::FORBIDDEN); // 403, never 401
    }
    Ok(next.run(request).await)
}

async fn list_users() -> &'static str { "users" }
async fn settings() -> &'static str { "settings" }

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/admin/users", get(list_users))
        .route("/admin/settings", get(settings))
        // route_layer AFTER the routes: a missing /admin/* path stays 404.
        .route_layer(middleware::from_fn(require_admin));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## Example 2: permission guard with from_fn_with_state and a permission store

The decision queries application state, so the guard uses
`from_fn_with_state`. The state is passed twice.

```rust
use axum::{
    extract::{Request, State},
    http::StatusCode,
    middleware::{self, Next},
    response::Response,
    routing::{get, post},
    Router,
};
use std::collections::{HashMap, HashSet};
use std::sync::Arc;

// A trivial in-memory permission store. In production this is a sqlx pool,
// a policy cache, or a role-to-permission table.
#[derive(Clone)]
struct PermissionStore {
    by_user: Arc<HashMap<String, HashSet<String>>>,
}

impl PermissionStore {
    async fn has(&self, user: &str, perm: &str) -> Result<bool, std::io::Error> {
        Ok(self
            .by_user
            .get(user)
            .map(|set| set.contains(perm))
            .unwrap_or(false))
    }
}

#[derive(Clone)]
struct AppState {
    perms: PermissionStore,
}

async fn require_users_write(
    State(state): State<AppState>, // FromRequestParts: first
    claims: Claims,                // FromRequestParts: before Request
    request: Request,              // FromRequest: second to last
    next: Next,                    // Next: last
) -> Result<Response, StatusCode> {
    let allowed = state
        .perms
        .has(&claims.sub, "users:write")
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    if !allowed {
        return Err(StatusCode::FORBIDDEN);
    }
    Ok(next.run(request).await)
}

async fn create_user() -> &'static str { "created" }
async fn list_users() -> &'static str { "users" }

fn build_app(state: AppState) -> Router {
    Router::new()
        .route("/admin/users", get(list_users))
        .route("/admin/users", post(create_user))
        // state.clone() into the layer, state into with_state.
        .route_layer(middleware::from_fn_with_state(
            state.clone(),
            require_users_write,
        ))
        .with_state(state)
}
```

## Example 3: newtype guard extractor, axum 0.8 form

```rust
// axum 0.8: native async fn in trait, no attribute.
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
    routing::delete,
    Json, Router,
};

pub struct AdminClaims(pub Claims);

impl<S> FromRequestParts<S> for AdminClaims
where
    S: Send + Sync,
{
    type Rejection = StatusCode;

    async fn from_request_parts(parts: &mut Parts, state: &S)
        -> Result<Self, Self::Rejection>
    {
        let claims = Claims::from_request_parts(parts, state)
            .await
            .map_err(|_| StatusCode::UNAUTHORIZED)?; // bad/no token: 401
        if claims.role != "admin" {
            return Err(StatusCode::FORBIDDEN);        // known, not admin: 403
        }
        Ok(AdminClaims(claims))
    }
}

// The argument type proves this handler is admin-only.
async fn delete_user(AdminClaims(claims): AdminClaims)
    -> Result<Json<()>, StatusCode>
{
    tracing::info!(admin = %claims.sub, "admin deleting user");
    Ok(Json(()))
}

// axum 0.8 path syntax: {id}
let app = Router::new().route("/users/{id}", delete(delete_user));
```

## Example 4: the same newtype guard, axum 0.7 form

The `impl` block is identical except for the `#[async_trait]` attribute and
the route path syntax.

```rust
// axum 0.7: the impl block MUST be annotated with #[async_trait].
use axum::{
    async_trait,
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

pub struct AdminClaims(pub Claims);

#[async_trait]
impl<S> FromRequestParts<S> for AdminClaims
where
    S: Send + Sync,
{
    type Rejection = StatusCode;

    async fn from_request_parts(parts: &mut Parts, state: &S)
        -> Result<Self, Self::Rejection>
    {
        let claims = Claims::from_request_parts(parts, state)
            .await
            .map_err(|_| StatusCode::UNAUTHORIZED)?;
        if claims.role != "admin" {
            return Err(StatusCode::FORBIDDEN);
        }
        Ok(AdminClaims(claims))
    }
}

// axum 0.7 path syntax: :id
// let app = Router::new().route("/users/:id", delete(delete_user));
```

## Example 5: a generic permission guard extractor

The newtype idea generalizes to fine-grained permissions. A const-generic
`RequirePermission<PERM>` names the capability in the handler signature.

```rust
// axum 0.8 form.
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

// The handler signature names the exact capability it needs.
pub struct RequirePermission<const PERM: &'static str>(pub Claims);

impl<S, const PERM: &'static str> FromRequestParts<S> for RequirePermission<PERM>
where
    S: Send + Sync,
{
    type Rejection = StatusCode;

    async fn from_request_parts(parts: &mut Parts, state: &S)
        -> Result<Self, Self::Rejection>
    {
        let claims = Claims::from_request_parts(parts, state)
            .await
            .map_err(|_| StatusCode::UNAUTHORIZED)?;
        if !claims.permissions.contains(PERM) {
            return Err(StatusCode::FORBIDDEN);
        }
        Ok(RequirePermission(claims))
    }
}

// Usage: the type spells out the required capability.
async fn refund(_guard: RequirePermission<"billing:refund">)
    -> Result<&'static str, StatusCode>
{
    Ok("refunded")
}
```

If the target toolchain does not support `&'static str` const generics, use a
plain `RequirePermission` newtype that reads the permission from a route
constant, or a middleware guard with `from_fn`. The const-generic form is the
most self-documenting when the compiler supports it.

## Example 6: full protected-routes wiring

A public group, an authenticated group, and an admin group on one router.

```rust
use axum::{middleware, routing::get, Router};

fn app(state: AppState) -> Router {
    // Public routes: no guard.
    let public = Router::new().route("/health", get(|| async { "ok" }));

    // Admin routes: role guard applied with route_layer.
    let admin = Router::new()
        .route("/admin/users", get(|| async { "users" }))
        .route_layer(middleware::from_fn(require_admin));

    // Merge, then attach cross-cutting layers with plain .layer().
    Router::new()
        .merge(public)
        .merge(admin)
        .with_state(state)
    // tracing/CORS would be added here with .layer(), never route_layer.
}
# #[derive(Clone)] struct AppState;
# async fn require_admin(c: Claims, r: axum::extract::Request, n: axum::middleware::Next)
#     -> Result<axum::response::Response, axum::http::StatusCode> { Ok(n.run(r).await) }
```

Guards use `.route_layer()`. Cross-cutting middleware (tracing, CORS,
compression, timeout) uses `.layer()`.
