---
name: axum-impl-authorization
description: >
  Use when an Axum app must decide whether an already-authenticated user is
  allowed to reach a route: protecting an /admin section, gating an endpoint
  behind a role, or enforcing a fine-grained permission like users:write.
  Prevents applying the guard with .layer() instead of .route_layer() (which
  turns a genuine 404 into a 401 and leaks which routes exist), prevents
  returning 401 on a failed authorization check when the answer is 403,
  prevents hardcoding a role hierarchy into every endpoint, prevents
  re-verifying the token inside a guard extractor, and prevents the wrong
  middleware argument order.
  Covers role-based and permission-based access control as from_fn and
  from_fn_with_state middleware applied with route_layer, the 401-versus-403
  status rule, the layer-versus-route_layer decision, and the newtype guard
  extractor pattern (AdminClaims) with its FromRequestParts impl in both Axum
  0.7 and 0.8 form.
  Keywords: axum authorization, RBAC, role-based access control, permission
  check, from_fn, from_fn_with_state, route_layer, layer, Next, FromRequestParts,
  AdminClaims, guard extractor, middleware guard, 403 Forbidden, 401
  Unauthorized, 404 leak, my 404 returns 401, auth guard returns 401 not 404,
  forbidden vs unauthorized, route exists but forbidden, how do I protect a
  route, how do I require admin, how do I check permissions, who can access
  this endpoint, protect /admin routes.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-impl-authorization

## Overview

Authorization answers "is this known user allowed to do this". It is a
separate concern from authentication, which answers "who are you".
Authentication is handled by `axum-impl-auth-jwt` and produces a verified
`Claims` value. Authorization always runs AFTER authentication and reads that
`Claims` value.

Axum has no built-in authorization. It provides the building blocks:
`FromRequestParts` extractors, `from_fn` and `from_fn_with_state` middleware,
and the `route_layer` application method. There are two mechanisms, and they
are NOT interchangeable:

1. Middleware guard: `from_fn` or `from_fn_with_state` applied with
   `route_layer`. Protects a GROUP of routes. The requirement lives on the
   router.
2. Newtype guard extractor: a custom `FromRequestParts` type such as
   `AdminClaims`. Protects a SINGLE handler. The requirement lives in the
   handler signature.

Two rules dominate this skill:

- A failed authorization check ALWAYS returns `403 Forbidden`, NEVER `401`.
- A guard is ALWAYS applied with `.route_layer()`, NEVER `.layer()`.

The authorization mechanisms are version-stable: `from_fn`,
`from_fn_with_state`, `Next`, `route_layer`, and `layer` have the same shape
in Axum 0.7 and 0.8. Only the newtype guard's `FromRequestParts` impl differs
between versions (see the version note in the patterns below).

## Quick Reference

| Mechanism | Protects | Requirement lives on | Built from |
|-----------|----------|----------------------|------------|
| Middleware guard | a group of routes | the router | `from_fn` / `from_fn_with_state` + `route_layer` |
| Newtype guard extractor | a single handler | the handler signature | a `FromRequestParts` impl |

Status code rule:

| Situation | Status | Meaning |
|-----------|--------|---------|
| Missing or invalid credentials | `401 Unauthorized` | "I do not know who you are" (authentication layer) |
| Known user lacks a role or permission | `403 Forbidden` | "I know who you are, you may not do this" (authorization layer) |

`.layer()` versus `.route_layer()`:

| Method | Runs on | Use for |
|--------|---------|---------|
| `.layer()` | EVERY request, including the unmatched fallback (404) path | logging, tracing, request-id, CORS, compression, timeout |
| `.route_layer()` | ONLY requests that match a registered route | authorization and authentication GUARDS |

Middleware guard argument order (`from_fn` / `from_fn_with_state`):

```
[zero or more FromRequestParts extractors], [exactly one FromRequest extractor], Next
```

`State` and `Claims` are `FromRequestParts` and come first. `Request` is the
single `FromRequest` extractor and is second to last. `Next` is always last.

Core rules:

- ALWAYS apply an authorization guard with `.route_layer()`. NEVER `.layer()`.
- ALWAYS return `403 Forbidden` on a failed role or permission check.
- ALWAYS register routes BEFORE attaching `.route_layer()`; it only wraps
  routes that already exist.
- ALWAYS reuse the `Claims` extractor for identity. NEVER re-verify the token
  inside a guard.
- ALWAYS check a permission (`articles:write`) rather than a coarse role when
  the role set may grow.
- NEVER implement both `FromRequest` and `FromRequestParts` for a guard
  newtype; the official docs warn this makes the extractor unusable.

## Decision Trees

### Middleware guard or newtype extractor

```
How many routes does this requirement protect?
  a whole group (every route under /admin)
    -> middleware guard: from_fn(_) or from_fn_with_state(_) + .route_layer()
  exactly one handler
    -> newtype guard extractor: a FromRequestParts type in the handler signature
  one handler, and the requirement must be visible in the signature
    -> newtype guard extractor (self-documenting, impossible to forget)
```

### from_fn or from_fn_with_state

```
Does the authorization decision need application state?
  no, it only reads fields already inside Claims (claims.role, claims.permissions)
    -> from_fn(guard)
  yes, it queries a permissions database, a policy cache, or a role-to-permission table
    -> from_fn_with_state(state.clone(), guard), and call .with_state(state) on the router
```

### Which status code

```
Why is the request being rejected?
  no credentials, or the token is invalid or expired
    -> 401 Unauthorized (this is the Claims extractor's job, not the guard's)
  the user is known but the role or permission does not match
    -> 403 Forbidden (the guard returns this)
NEVER return 401 from an authorization guard.
```

## Patterns

### Pattern: role-based middleware guard with route_layer

`from_fn` turns an `async fn` into a `Layer`. The guard reads the verified
`Claims`, checks the role, and either returns `403` or calls `next.run`.

```rust
use axum::{
    extract::Request,
    http::StatusCode,
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};

// `Claims` is the verified JWT extractor from axum-impl-auth-jwt.
// It is FromRequestParts, so it runs first and a missing or invalid
// token already rejects with 401 before this guard body runs.
async fn require_admin(
    claims: Claims,    // FromRequestParts: runs first
    request: Request,  // FromRequest: the single body extractor
    next: Next,        // Next: always last
) -> Result<Response, StatusCode> {
    if claims.role != "admin" {
        // authenticated but not authorized: 403, NEVER 401
        return Err(StatusCode::FORBIDDEN);
    }
    Ok(next.run(request).await)
}

let admin_routes = Router::new()
    .route("/admin/users", get(list_users))
    .route("/admin/settings", get(settings))
    // route_layer, NOT layer: a non-existent /admin/* path stays 404
    .route_layer(middleware::from_fn(require_admin));
```

### Pattern: from_fn_with_state for a state-dependent permission check

When the decision needs application state (a permission store, a policy
cache), use `from_fn_with_state`. Verified signature:

```rust
pub fn from_fn_with_state<F, S, T>(state: S, f: F) -> FromFnLayer<F, S, T>
```

State surfaces inside the guard as a `State<S>` extractor.

```rust
use axum::{
    extract::{Request, State},
    http::StatusCode,
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};

#[derive(Clone)]
struct AppState {
    perms: PermissionStore,
}

async fn require_users_write(
    State(state): State<AppState>, // FromRequestParts: first
    claims: Claims,                // FromRequestParts: still before Request
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

let state = AppState { perms: PermissionStore::new() };

let app = Router::new()
    .route("/admin/users", get(list_users))
    // the same state value is passed twice: clone for the layer
    .route_layer(middleware::from_fn_with_state(state.clone(), require_users_write))
    .with_state(state);
```

ALWAYS pass the state twice: `state.clone()` into `from_fn_with_state` and the
original `state` into `.with_state()`. The official docs example uses exactly
this `route_layer` plus double-pass shape.

### Pattern: permission-based check is the preferred default

A role check hardcodes a fixed hierarchy into every endpoint. A permission
check names the CAPABILITY the endpoint needs, independent of which roles hold
it. The mechanism is identical; only the predicate changes.

```rust
// BRITTLE: every editor-or-above guard must repeat the role list.
// Adding a "moderator" role means editing every one of these.
async fn require_editor(claims: Claims, request: Request, next: Next)
    -> Result<Response, StatusCode>
{
    if claims.role != "editor" && claims.role != "admin" {
        return Err(StatusCode::FORBIDDEN);
    }
    Ok(next.run(request).await)
}

// STABLE: the endpoint names the capability, not the roles that have it.
// Adding a "moderator" role only changes the role-to-permission mapping.
async fn require_articles_write(claims: Claims, request: Request, next: Next)
    -> Result<Response, StatusCode>
{
    if !claims.permissions.contains("articles:write") {
        return Err(StatusCode::FORBIDDEN);
    }
    Ok(next.run(request).await)
}
```

The `Claims` struct SHOULD carry a permission set (`HashSet<String>` or
`Vec<String>`) using a `resource:action` convention (`users:read`,
`users:write`, `billing:refund`). The authentication step expands a role into
its permission set when minting the token, so guards only ever check
permissions.

### Pattern: newtype guard extractor (AdminClaims)

For a SINGLE protected handler, a newtype `FromRequestParts` extractor makes
the requirement visible in the signature. It composes WITH the `Claims`
extractor by calling `Claims::from_request_parts`, never re-verifying the
token.

```rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

// A handler that takes `AdminClaims` is provably admin-only.
pub struct AdminClaims(pub Claims);

// axum 0.8: native async fn in trait, no attribute needed.
impl<S> FromRequestParts<S> for AdminClaims
where
    S: Send + Sync,
{
    type Rejection = StatusCode;

    async fn from_request_parts(parts: &mut Parts, state: &S)
        -> Result<Self, Self::Rejection>
    {
        // Reuse the existing Claims extractor for identity.
        let claims = Claims::from_request_parts(parts, state)
            .await
            .map_err(|_| StatusCode::UNAUTHORIZED)?; // bad/no token: 401
        if claims.role != "admin" {
            return Err(StatusCode::FORBIDDEN);        // known, not admin: 403
        }
        Ok(AdminClaims(claims))
    }
}
```

```rust
// axum 0.7: the SAME impl block must be annotated with #[async_trait].
#[async_trait::async_trait]
impl<S> FromRequestParts<S> for AdminClaims
where
    S: Send + Sync,
{
    type Rejection = StatusCode;

    async fn from_request_parts(parts: &mut Parts, state: &S)
        -> Result<Self, Self::Rejection>
    { /* identical body as the axum 0.8 version above */ }
}
```

Used in a handler, the requirement is self-documenting:

```rust
use axum::Json;

// The argument type PROVES this handler is admin-only.
async fn delete_user(AdminClaims(claims): AdminClaims)
    -> Result<Json<()>, StatusCode>
{
    tracing::info!(admin = %claims.sub, "admin deleting user");
    Ok(Json(()))
}
```

NEVER implement both `FromRequest` and `FromRequestParts` for the newtype; the
official docs warn this makes the extractor unusable. A guard inspects request
parts (headers, claims) only, so `FromRequestParts` alone is correct.

## Reference Links

- `references/methods.md`: verified signatures for `from_fn`,
  `from_fn_with_state`, `Next`, `route_layer`, `layer`, `with_state`, and
  `FromRequestParts`, plus the middleware argument-order rule.
- `references/examples.md`: full working programs, a permission store wired
  through `from_fn_with_state`, a generic `RequirePermission` extractor, and
  the 0.7 versus 0.8 newtype-guard forms.
- `references/anti-patterns.md`: the seven recurring authorization mistakes
  with the root cause of each and the fix.

Related skills:

- `axum-impl-auth-jwt`: defines the `Claims` extractor that every guard here
  reuses for identity.
- `axum-impl-middleware`: `from_fn` argument-order rules and layer ordering.
- `axum-syntax-custom-extractors`: the `FromRequestParts` trait the newtype
  guard implements.
