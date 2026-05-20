# Topic Research : axum-impl-authorization

> Skill : `axum-impl-authorization`
> Category : impl/
> Target versions : Axum 0.7 and 0.8 (signatures verified against docs.rs current : axum 0.8.9)
> Researched : 2026-05-20
> All API signatures and example code below were WebFetch-verified against docs.rs on 2026-05-20.
> Upstream fragments : `vooronderzoek-axum.md` section 5, `fragments/research-c-errors-auth-db-ops.md` section 5,
> `topic-research/axum-impl-middleware-research.md` (reused for `from_fn`/`from_fn_with_state`/`route_layer`).

This file is the verified content basis for the `axum-impl-authorization` skill. It is
NOT the skill. Authorization is a SEPARATE concern from authentication: authentication
answers "who are you" (covered by `axum-impl-auth`), authorization answers "are you
allowed to do this". This research covers role-based and permission-based access
control as middleware, the `route_layer`-vs-`layer` decision for guards, the
permission-vs-role tradeoff, and the dedicated newtype-guard-extractor pattern.

---

## 1. The mental model : authorization runs AFTER authentication

Axum has no built-in authorization. It provides the composable building blocks
(`FromRequestParts` extractors, `from_fn` / `from_fn_with_state` middleware) and the
`route_layer` application method. Authorization always assumes authentication already
ran and produced a verified `Claims` value carrying identity plus either a role or a
permission set.

Two distinct mechanisms exist, and they are NOT interchangeable:

1. **Middleware guard** (`from_fn` / `from_fn_with_state` applied with `route_layer`):
   protects a GROUP of routes at once. The requirement is declared on the router, not
   on each handler. Best for "every route under `/admin` needs the admin role".
2. **Newtype guard extractor** (a custom `FromRequestParts` type such as
   `AdminClaims`): protects a SINGLE handler and makes the requirement visible in the
   handler signature. Best for one-off protected endpoints and for self-documenting
   handlers.

Both are valid; the skill must present both and the decision boundary between them.

---

## 2. Role-based access control as middleware

The `axum::middleware::from_fn` function turns an `async fn` into a `Layer`. A
role-based guard extracts the already-verified `Claims` (the JWT extractor from
`axum-impl-auth` runs first because `Claims` is a `FromRequestParts` extractor), checks
the role field, and either short-circuits with `403 Forbidden` or calls
`next.run(request).await` to continue.

```rust
use axum::{
    extract::Request,
    http::StatusCode,
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};

// `Claims` is the verified JWT extractor from `axum-impl-auth`.
// Because it is a `FromRequestParts` extractor it runs BEFORE `Request`,
// and a failed extraction (no/invalid token) already rejects with 401.
async fn require_admin(
    claims: Claims,    // FromRequestParts -> runs first
    request: Request,  // FromRequest     -> second-to-last
    next: Next,        // Next            -> last
) -> Result<Response, StatusCode> {
    if claims.role != "admin" {
        // authenticated but not authorized -> 403, NOT 401
        return Err(StatusCode::FORBIDDEN);
    }
    Ok(next.run(request).await)
}

let admin_routes = Router::new()
    .route("/admin/users", get(list_users))
    .route("/admin/settings", get(settings))
    .route_layer(middleware::from_fn(require_admin));
```

The middleware argument order is the same strict rule as any `from_fn` middleware:
`[zero or more FromRequestParts extractors] , [exactly one FromRequest extractor] , Next`.
`Claims` is `FromRequestParts`, `Request` is the single `FromRequest` extractor, `Next`
is last.

**Status-code rule (deterministic):**
- `401 Unauthorized` means "I do not know who you are" : missing or invalid
  credentials. This is the authentication layer's job and is already produced when the
  `Claims` extractor fails.
- `403 Forbidden` means "I know who you are, and you may not do this" : a failed
  authorization check. The authorization guard ALWAYS returns `403`, NEVER `401`, on a
  role/permission mismatch.

---

## 3. `from_fn_with_state` : authorization that needs application state

When the authorization decision depends on application state (a permissions database,
a policy cache, a role-to-permission mapping table), use `from_fn_with_state`. Verified
signature (docs.rs, 2026-05-20):

```rust
pub fn from_fn_with_state<F, S, T>(state: S, f: F) -> FromFnLayer<F, S, T>
```

The docs state verbatim: "For the requirements for the function supplied see
`from_fn`." State surfaces inside the function as a `State<S>` extractor; because
`State` is `FromRequestParts` it comes first, before `Request` and `Next`.

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
    perms: PermissionStore, // e.g. an Arc-wrapped lookup, a sqlx pool, a cache
}

async fn require_permission(
    State(state): State<AppState>, // FromRequestParts -> first
    claims: Claims,                // FromRequestParts -> still before Request
    request: Request,              // FromRequest      -> second-to-last
    next: Next,                    // Next             -> last
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
    .route_layer(middleware::from_fn_with_state(state.clone(), require_permission))
    .with_state(state);
```

CRITICAL: the same state value is passed twice : once into
`from_fn_with_state(state.clone(), ...)` and once into `.with_state(state)`. ALWAYS
clone the state for the layer. The official docs.rs example for `from_fn_with_state`
itself uses `.route_layer(...)` (not `.layer(...)`) and the `state.clone()` / `.with_state(state)`
double-pass, confirmed verbatim 2026-05-20.

---

## 4. Why `route_layer` and NOT `layer` for guards

This is the single most important rule for an authorization guard. Verified method
signatures (docs.rs `Router` page, 2026-05-20):

```rust
pub fn layer<L>(self, layer: L) -> Router<S>
where
    L: Layer<Route> + Clone + Send + Sync + 'static,
    /* ... identical service bounds ... */

pub fn route_layer<L>(self, layer: L) -> Self
where
    L: Layer<Route> + Clone + Send + Sync + 'static,
    /* ... identical service bounds ... */
```

The bounds are identical; the BEHAVIOR is not. Verbatim documentation
(docs.rs `Router::route_layer`, fetched 2026-05-20):

> "This works similarly to `Router::layer` except the middleware will only run if the
> request matches a route. This is useful for middleware that return early (such as
> authorization) which might otherwise convert a `404 Not Found` into a
> `401 Unauthorized`."

Documented behavior of the canonical example:
- `GET /foo` with a valid token   -> `200 OK`
- `GET /foo` with an invalid token -> `401 Unauthorized`
- `GET /not-found` with an invalid token -> `404 Not Found`

| Method | Applies to | Use for |
|--------|-----------|---------|
| `.layer()` | EVERY request, including the unmatched / fallback (404) path | logging, tracing, request-id, CORS, compression, timeout |
| `.route_layer()` | ONLY requests that match a registered route | authorization / authentication GUARDS |

WHY this matters: with a plain `.layer()` an auth guard runs on the fallback path too,
so a request to a path that does not exist hits the guard FIRST and returns `401`/`403`
before the router can answer `404`. That both leaks the existence of routes (an
attacker probing paths cannot tell "exists but forbidden" from "does not exist") and
masks a genuine `404` with a misleading auth error. `route_layer` skips the unmatched
path entirely, so a non-existent route correctly returns `404`.

RULE: an authorization guard is ALWAYS applied with `.route_layer()`, NEVER `.layer()`.

---

## 5. Permission-based vs role-based : decouple endpoints from the role hierarchy

Role-based access control (RBAC) checks a coarse label (`claims.role == "admin"`).
Permission-based access control checks a fine-grained capability
(`claims.permissions.contains("users:write")`). The mechanism is identical (a
middleware guard or a newtype extractor); only the predicate differs.

**Permission-based is the preferred default.** WHY: a role check hardcodes a fixed role
hierarchy into every endpoint. The moment a fourth role appears, or a single role needs
one extra capability, every guard that named a role must be edited. A permission check
names the CAPABILITY the endpoint needs ("this route writes users"), independent of
which roles happen to hold that capability. Roles then become a presentation-layer
grouping that maps to permission sets, and endpoints never need to know the role
hierarchy at all.

Worked example. Role-based, brittle:

```rust
// Every editor-or-above endpoint must repeat the role list.
// Adding a "moderator" role means editing every one of these guards.
async fn require_editor(claims: Claims, request: Request, next: Next)
    -> Result<Response, StatusCode>
{
    if claims.role != "editor" && claims.role != "admin" {
        return Err(StatusCode::FORBIDDEN);
    }
    Ok(next.run(request).await)
}
```

Permission-based, stable:

```rust
// The endpoint names the capability it needs, not the roles that have it.
// Adding a "moderator" role only changes the role->permission mapping,
// never this guard.
async fn require_articles_write(claims: Claims, request: Request, next: Next)
    -> Result<Response, StatusCode>
{
    if !claims.permissions.contains("articles:write") {
        return Err(StatusCode::FORBIDDEN);
    }
    Ok(next.run(request).await)
}
```

The `Claims` struct in `axum-impl-auth` therefore SHOULD carry a permission set
(e.g. `permissions: Vec<String>` or a `HashSet<String>`) in addition to or instead of
a single `role: String`. A permission convention such as `resource:action`
(`users:read`, `users:write`, `billing:refund`) keeps checks readable. Roles can still
exist as a convenience: the authentication step expands a role into its permission set
when minting the token, so the token carries permissions and the guards only ever
check permissions.

---

## 6. The newtype-guard-extractor pattern : `AdminClaims`

A middleware guard protects a GROUP of routes. For a SINGLE protected handler, a
dedicated newtype `FromRequestParts` extractor is cleaner: the authorization
requirement appears directly in the handler signature, so the protection is
self-documenting and impossible to forget when adding the route.

`AdminClaims` is a newtype wrapper around `Claims` whose `FromRequestParts`
implementation runs the `Claims` extraction first, then rejects non-admins with `403`:

```rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

// Newtype guard: a handler that takes `AdminClaims` is provably admin-only.
pub struct AdminClaims(pub Claims);

impl<S> FromRequestParts<S> for AdminClaims
where
    S: Send + Sync,
{
    type Rejection = StatusCode;

    // Axum 0.8: native `async fn` in trait (no `#[async_trait]`).
    // Axum 0.7: this `impl` block needs `#[async_trait]`.
    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        // Compose with the existing `Claims` extractor instead of
        // re-implementing token verification.
        let claims = Claims::from_request_parts(parts, state)
            .await
            .map_err(|_| StatusCode::UNAUTHORIZED)?; // bad/no token -> 401

        if claims.role != "admin" {
            return Err(StatusCode::FORBIDDEN); // authenticated, not admin -> 403
        }
        Ok(AdminClaims(claims))
    }
}
```

Used in a handler, the requirement is visible in the signature:

```rust
use axum::Json;

// The type of the argument PROVES this handler is admin-only.
async fn delete_user(
    AdminClaims(claims): AdminClaims,
    /* Path, Json, etc. */
) -> Result<Json<()>, StatusCode> {
    tracing::info!(admin = %claims.sub, "admin deleting user");
    // ... privileged work ...
    Ok(Json(()))
}

let app = Router::new().route("/users/{id}", axum::routing::delete(delete_user));
```

Notes for the skill:
- The rejection type `StatusCode` is the simplest choice; a richer `AuthError` enum
  (with an `IntoResponse` impl that returns a JSON body) is the production-grade choice
  and matches the error pattern from `axum-errors-handling`.
- The newtype guard composes WITH the JWT `Claims` extractor by CALLING
  `Claims::from_request_parts` rather than duplicating token verification. Identity
  verification lives in exactly one place (the `Claims` extractor); `AdminClaims` only
  adds the authorization predicate on top.
- This pattern generalizes to permissions. A `RequirePermission` newtype, or a
  const-generic `RequirePermission<const PERM: &'static str>` extractor, applies the
  same idea to fine-grained capabilities: the handler signature names the capability it
  needs.
- Do NOT implement BOTH `FromRequest` and `FromRequestParts` for the same concrete
  newtype; the official docs warn this makes the extractor unusable. A guard only needs
  `FromRequestParts` because it inspects request parts (headers/claims), never the body.

---

## 7. How the two mechanisms compose

| Concern | Mechanism | Where the requirement lives |
|---------|-----------|-----------------------------|
| Authentication ("who are you") | `Claims` extractor (`axum-impl-auth`) | the handler/middleware argument list |
| Authorize a GROUP of routes | `from_fn` / `from_fn_with_state` + `route_layer` | the router |
| Authorize a SINGLE handler | `AdminClaims` newtype extractor | the handler signature |

All three layer on top of the same verified `Claims` value. The middleware guard reads
`Claims` as a first argument; the newtype guard wraps `Claims` and calls its
`FromRequestParts` impl. Token verification (signature, `exp` expiry) is implemented
ONCE in the `Claims` extractor; every authorization construct reuses it. This is the
"protected routes" workflow from `vooronderzoek-axum.md` section 6: a `Claims`
extractor for authentication plus a `route_layer` guard (or a newtype extractor) for
authorization.

---

## 8. Anti-patterns this skill must warn against

1. **Applying an auth guard with `.layer()` instead of `.route_layer()`** : the
   guard runs on the fallback path and turns a genuine `404 Not Found` into a
   `401`/`403`, leaking route existence. ALWAYS use `.route_layer()` for guards.
2. **Returning `401` on a failed authorization check** : a known user who lacks a
   role/permission is `403 Forbidden`, not `401 Unauthorized`. `401` is reserved for
   missing/invalid identity.
3. **Hardcoding a role hierarchy into every endpoint** : `claims.role == "admin"`
   repeated across the codebase breaks the moment a role is added. Check permissions
   (`articles:write`), not coarse roles.
4. **Duplicating token verification inside `AdminClaims`** : the newtype guard must
   CALL `Claims::from_request_parts`, not re-decode the JWT. One place verifies
   identity.
5. **Putting the body extractor in the wrong middleware position** : in a `from_fn`
   guard the single `FromRequest` extractor (`Request`) sits directly before `Next`;
   `Claims` and `State` are `FromRequestParts` and must come before it.
6. **Implementing both `FromRequest` and `FromRequestParts` for the newtype guard** :
   the official docs warn this makes the extractor unusable. A guard only needs
   `FromRequestParts`.
7. **Registering routes after `.route_layer()`** : like `.layer()`, `route_layer`
   only wraps routes that already exist; add routes first, then the guard.

---

## 9. Version notes (0.7 vs 0.8)

- The newtype guard's `impl FromRequestParts for AdminClaims` block needs
  `#[async_trait]` on Axum **0.7** and must NOT have it on Axum **0.8** (0.8 removed
  `#[async_trait]` in favor of native `async fn` in traits, MSRV Rust 1.75). The skill
  MUST show both forms.
- `from_fn`, `from_fn_with_state`, `Next`, `route_layer`, and `layer` are unchanged in
  shape between 0.7 and 0.8; the authorization patterns themselves are version-stable.
- Route-path syntax in the surrounding `.route(...)` calls differs: `/users/:id` on
  0.7, `/users/{id}` on 0.8. This is incidental to authorization but appears in any
  full example.

---

## Sources verified (2026-05-20)

All URLs fetched via WebFetch on 2026-05-20. Current docs.rs version observed:
axum 0.8.9.

| URL | Verified | Used for |
|-----|----------|----------|
| https://docs.rs/axum/latest/axum/middleware/fn.from_fn_with_state.html | 2026-05-20 | Section 3 (`from_fn_with_state` exact signature, verbatim state example, "same requirements as `from_fn`", example uses `route_layer` + `state.clone()`/`with_state`) |
| https://docs.rs/axum/latest/axum/struct.Router.html | 2026-05-20 | Section 4 (`layer` and `route_layer` exact signatures + bounds, `with_state` signature, verbatim `route_layer` doc text on 404-vs-401, documented 200/401/404 behavior) |
| (reused) topic-research/axum-impl-middleware-research.md | 2026-05-20 | `from_fn` 5-point requirement list, argument-order rule, `Next::run` signature, layer-ordering rule |
| (reused) fragments/research-c-errors-auth-db-ops.md section 5 | 2026-05-20 | role-based / permission-based RBAC pattern, `require_admin` example, the `AdminClaims` / `RequirePermission` newtype-extractor idea |
| (reused) fragments/research-c-errors-auth-db-ops.md section 4 | 2026-05-20 | `Claims` `FromRequestParts` extractor that authorization composes on top of |
| (reused) vooronderzoek-axum.md section 5 + section 6 | 2026-05-20 | "Axum has no built-in auth", authorization-via-`route_layer` model, the protected-routes workflow |

### Verification notes

- The `from_fn_with_state` signature and full example block are quoted verbatim from
  the live docs.rs page fetched 2026-05-20; the page explicitly uses `.route_layer(...)`
  for the middleware, which directly supports this skill's core rule.
- The `route_layer` documentation quote ("...might otherwise convert a `404 Not Found`
  into a `401 Unauthorized`") and the 200/401/404 behavior matrix are verbatim from the
  docs.rs `Router` page fetched 2026-05-20.
- The `#[async_trait]` 0.7-vs-0.8 split for the custom extractor is verified via the
  upstream cluster-C fragment, which confirmed it against the live native-async trait
  definitions and the Axum 0.8 announcement on 2026-05-20.
- The permission-vs-role guidance is a design recommendation derived from the documented
  mechanism (both checks use the identical guard machinery), not a verbatim docs quote.
