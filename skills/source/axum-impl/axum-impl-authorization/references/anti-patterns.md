# anti-patterns.md : axum-impl-authorization

The seven recurring authorization mistakes in Axum, each with the root cause
and the fix. Verified against the official `docs.rs` pages on 2026-05-20
(axum 0.8.9). The mechanisms are version-stable, so every anti-pattern below
applies to both Axum 0.7 and 0.8.

## AP1: applying a guard with .layer() instead of .route_layer()

```rust
// WRONG: a guard on .layer() runs on the unmatched fallback path too.
let app = Router::new()
    .route("/admin/users", get(list_users))
    .layer(middleware::from_fn(require_admin));
```

WHY THIS FAILS: `.layer()` runs on EVERY request, including a request whose
path matches no route. A request to `/does-not-exist` hits the guard before
the router can answer `404`, so the guard returns `401` or `403` for a path
that does not exist. Verbatim official documentation: `route_layer` exists for
"middleware that return early (such as authorization) which might otherwise
convert a `404 Not Found` into a `401 Unauthorized`". This both masks a
genuine `404` and leaks route existence: an attacker probing paths cannot tell
"exists but forbidden" from "does not exist".

FIX: apply every authorization guard with `.route_layer()`, which runs only on
requests that match a registered route.

```rust
// CORRECT: a non-existent path correctly returns 404.
let app = Router::new()
    .route("/admin/users", get(list_users))
    .route_layer(middleware::from_fn(require_admin));
```

## AP2: returning 401 on a failed authorization check

```rust
// WRONG: a known user who lacks the role is not "unauthenticated".
async fn require_admin(claims: Claims, request: Request, next: Next)
    -> Result<Response, StatusCode>
{
    if claims.role != "admin" {
        return Err(StatusCode::UNAUTHORIZED); // 401 is wrong here
    }
    Ok(next.run(request).await)
}
```

WHY THIS FAILS: `401 Unauthorized` means "I do not know who you are": missing
or invalid credentials. By the time a guard body runs, the `Claims` extractor
has already verified identity; a missing or invalid token was already rejected
with `401`. A user who is known but lacks the role is a different condition:
`403 Forbidden`, "I know who you are and you may not do this". Returning `401`
here tells a correctly-authenticated client to re-authenticate, which cannot
fix the problem and confuses clients and logs.

FIX: return `403 Forbidden` from the guard on a role or permission mismatch.

```rust
// CORRECT.
if claims.role != "admin" {
    return Err(StatusCode::FORBIDDEN);
}
```

## AP3: hardcoding a role hierarchy into every endpoint

```rust
// WRONG: every editor-or-above guard repeats the role list.
if claims.role != "editor" && claims.role != "admin" {
    return Err(StatusCode::FORBIDDEN);
}
```

WHY THIS FAILS: naming roles in every guard hardcodes a fixed hierarchy across
the codebase. Adding a "moderator" role, or giving one role a single extra
capability, forces an edit to every guard that named a role. The endpoints
know about the role hierarchy when they should only know what capability they
need.

FIX: check a fine-grained permission. The endpoint names the CAPABILITY; roles
become a mapping that the authentication step expands into permissions when
minting the token.

```rust
// CORRECT: the endpoint names the capability, not the roles that hold it.
if !claims.permissions.contains("articles:write") {
    return Err(StatusCode::FORBIDDEN);
}
```

## AP4: re-verifying the token inside a guard extractor

```rust
// WRONG: AdminClaims decodes the JWT again itself.
async fn from_request_parts(parts: &mut Parts, _state: &S)
    -> Result<Self, Self::Rejection>
{
    let header = parts.headers.get("authorization") /* ... */;
    let token = decode_jwt(header)?;          // duplicated verification
    if token.role != "admin" { /* ... */ }
}
```

WHY THIS FAILS: token verification (signature check, `exp` expiry, audience)
is security-critical and must exist in exactly one place. Duplicating it inside
`AdminClaims` means two code paths can drift: a fix to one is missed in the
other, and the duplicate is an untested second attack surface.

FIX: the newtype guard composes WITH the `Claims` extractor by calling
`Claims::from_request_parts`. Identity verification stays in the `Claims`
extractor; the guard only adds the authorization predicate.

```rust
// CORRECT: reuse the one verified Claims extractor.
let claims = Claims::from_request_parts(parts, state)
    .await
    .map_err(|_| StatusCode::UNAUTHORIZED)?;
if claims.role != "admin" {
    return Err(StatusCode::FORBIDDEN);
}
```

## AP5: wrong middleware argument order in a guard

```rust
// WRONG: Request placed before a FromRequestParts extractor.
async fn require_admin(request: Request, claims: Claims, next: Next)
    -> Result<Response, StatusCode>
{ /* ... */ }
```

WHY THIS FAILS: a `from_fn` guard allows exactly one `FromRequest` extractor
(`Request`), and it must sit directly before `Next`. `Claims` and `State` are
`FromRequestParts` and must come before the `FromRequest` extractor. Placing
`Request` first, or adding a second `FromRequest` extractor, does not satisfy
the middleware trait bounds and fails to compile.

FIX: order the arguments as `[FromRequestParts extractors], Request, Next`.

```rust
// CORRECT.
async fn require_admin(claims: Claims, request: Request, next: Next)
    -> Result<Response, StatusCode>
{ /* ... */ }
```

## AP6: implementing both FromRequest and FromRequestParts for the guard newtype

```rust
// WRONG: both trait impls on the same concrete type.
impl<S> FromRequestParts<S> for AdminClaims { /* ... */ }
impl<S> FromRequest<S>      for AdminClaims { /* ... */ }
```

WHY THIS FAILS: the official Axum documentation warns that implementing both
`FromRequest` and `FromRequestParts` for the same concrete type makes the
extractor unusable, because the compiler cannot pick which impl applies in a
handler argument position.

FIX: a guard inspects request parts (headers, claims) only and never the body,
so it needs `FromRequestParts` ONLY. Implement that one trait.

## AP7: registering routes after .route_layer()

```rust
// WRONG: the guard is attached before the route it should protect.
let app = Router::new()
    .route_layer(middleware::from_fn(require_admin))
    .route("/admin/users", get(list_users)); // NOT protected
```

WHY THIS FAILS: `route_layer`, like `layer`, only wraps routes that already
exist on the router at the moment it is called. A route registered AFTER
`.route_layer()` is not wrapped by the guard and is silently left
unprotected. There is no compile error; the hole is invisible until tested.

FIX: register every route the guard must protect FIRST, then call
`.route_layer()`.

```rust
// CORRECT.
let app = Router::new()
    .route("/admin/users", get(list_users))
    .route_layer(middleware::from_fn(require_admin));
```

## Summary table

| Anti-pattern | Symptom | Fix |
|--------------|---------|-----|
| AP1: `.layer()` for a guard | a missing path returns `401`/`403`, not `404` | apply guards with `.route_layer()` |
| AP2: `401` on a failed check | client told to re-authenticate but cannot fix it | return `403 Forbidden` |
| AP3: hardcoded role hierarchy | adding a role forces edits everywhere | check permissions, not roles |
| AP4: re-verifying the token | two drifting verification paths | call `Claims::from_request_parts` |
| AP5: wrong argument order | guard does not compile | `[FromRequestParts], Request, Next` |
| AP6: both extractor traits | extractor unusable in handlers | implement `FromRequestParts` only |
| AP7: routes after `route_layer` | a route is silently unprotected | register routes first, then guard |
