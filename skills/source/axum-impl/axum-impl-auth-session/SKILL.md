---
name: axum-impl-auth-session
description: >
  Use when an Axum app must authenticate users with server-side sessions
  instead of self-contained tokens: a server-rendered web app with a login
  form, a flow that needs instant logout or revocation, mutable per-user state
  like a cart, HTTP Basic auth for an internal admin endpoint, or an OAuth2
  social-login (Google, GitHub, corporate SSO) callback.
  Prevents shipping MemoryStore to production where sessions vanish on restart,
  prevents the with_secure(false) cookie leak over plain HTTP, prevents the
  session-fixation hole from not rotating the session id on login, prevents
  Basic auth credentials traveling in cleartext, prevents the OAuth2 login-CSRF
  hole from skipping the state check, and prevents login_required! applied with
  .layer() turning a 404 into an auth redirect.
  Covers the four-way auth decision tree (JWT vs session vs Basic vs OAuth2),
  tower-sessions SessionManagerLayer and the Session API, axum-login AuthSession
  with the AuthUser and AuthnBackend traits and the login_required! guard,
  axum-extra TypedHeader Authorization Basic, and the oauth2 crate
  authorization-code flow with PKCE.
  Keywords: axum session auth, tower-sessions, SessionManagerLayer, axum-login,
  AuthSession, login_required, AuthnBackend, AuthUser, cycle_id, session store,
  MemoryStore, HTTP Basic auth, TypedHeader Authorization Basic, oauth2 crate,
  authorization code flow, PKCE, CsrfToken, OAuth2 callback, Set-Cookie,
  session cookie, my sessions disappear after restart, logged out on deploy,
  cannot revoke a login, 404 redirects to login, login keeps failing,
  how do I add a login form, how do I do social login, what is session auth,
  session vs jwt, server-side session.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-impl-auth-session

## Overview

Session-based authentication stores NO user data in the cookie. The cookie
carries only an opaque, unguessable session ID; all real state (user id, roles,
CSRF token) lives server-side in a session store. This is the structural
opposite of JWT, where the token IS the state and the server holds nothing.

Axum ships no session machinery. This skill covers the four stateful and
delegated auth approaches and the crates that provide them:

| Approach | Crate | Use for |
|----------|-------|---------|
| Raw sessions | `tower-sessions` | session layer, no opinion on login |
| Full login system | `axum-login` (on `tower-sessions`) | typed user, login/logout, route guards |
| HTTP Basic | `axum-extra` `TypedHeader` | machine-to-machine over TLS |
| OAuth2 | `oauth2` | delegated identity (Google, GitHub, SSO) |

JWT (stateless bearer tokens) is a separate mechanism owned by the
`axum-impl-auth-jwt` skill.

## Version Sensitivity

The four crates here are third-party and evolve faster than Axum. Builder method
names and macro argument shapes change across releases. The verified versions
for this skill are:

| Crate | Verified version |
|-------|------------------|
| `tower-sessions` | 0.15.0 |
| `axum-login` | 0.18.0 |
| `oauth2` | 5.0.0 |
| `axum-extra` | current (`typed-header` feature) |

ALWAYS confirm every snippet below against the exact version pinned in your
`Cargo.toml`. Snippets are marked `// version-sensitive` where the API shape is
known to drift. The crates themselves are independent of the Axum 0.7 vs 0.8
split; the only Axum-version concern is that a custom extractor built around
them uses the 0.8 native-async trait form, no `#[async_trait]`.

## Quick Reference

| Need | Do this |
|------|---------|
| Add a session layer | `.layer(SessionManagerLayer::new(store))` |
| Read session value | `session.get::<T>("key").await` returns `Option<T>` |
| Write session value | `session.insert("key", value).await` |
| Destroy a session | `session.delete().await` or `session.flush().await` |
| Rotate session id on login | `session.cycle_id().await` |
| Full login system | `axum-login` `AuthSession` + `AuthnBackend` impl |
| Guard routes | `.route_layer(login_required!(Backend, login_url = "/login"))` |
| HTTP Basic header | `TypedHeader<Authorization<Basic>>` extractor |
| Delegate to Google/GitHub | `oauth2` crate authorization-code flow |

| `tower-sessions` `Session` method | Effect |
|-----------------------------------|--------|
| `insert(key, value)` | serde-serialize a value into the session |
| `get::<T>(key)` | read a value, `Option<T>` |
| `remove::<T>(key)` | drop one key |
| `clear()` | drop all keys, keep the session row |
| `cycle_id()` | issue a new session id, keep the data |
| `flush()` / `delete()` | destroy the server-side session |

Full API signatures: `references/methods.md`.

## Decision Trees

### Which auth mechanism

```
Authenticate a request. Pick ONE primary mechanism per route.

1. Is auth delegated to an external identity provider
   (Google / GitHub / corporate SSO)?
      YES -> OAuth2 (oauth2 crate, authorization-code + PKCE).
             After the callback, establish a LOCAL session.
      NO  -> continue.

2. Is the caller a browser using a server-rendered app, OR do you
   need instant revocation OR mutable server-side per-user state?
      YES -> Session-based auth.
             raw layer only    -> tower-sessions
             full login system -> axum-login on tower-sessions
      NO  -> continue.

3. Is the caller a machine / internal admin tool over TLS, and is a
   username+password sent on every request acceptable?
      YES -> HTTP Basic (axum-extra TypedHeader<Authorization<Basic>>).
             TLS ONLY. Never the primary login for human browsers.
      NO  -> continue.

4. Is the caller a stateless API client (SPA, mobile, service) where
   every request carries its own credential and tokens valid-until-exp
   is acceptable?
      YES -> JWT. See the axum-impl-auth-jwt skill.
```

Tie-breakers:

- OAuth2 is an IDENTITY mechanism, not a session mechanism. After the callback
  you still pick session OR JWT to carry the logged-in state. OAuth2 + session
  is the normal browser combination.
- Session vs JWT, the deciding question is revocation and state: need to kill a
  login immediately or store mutable per-user data, choose session. Need zero
  server state and horizontal scale with no shared store, choose JWT.
- NEVER mix two primary mechanisms on one route. One route, one auth extractor.

### Raw tower-sessions or full axum-login

```
Do you need a typed User, login/logout helpers, and route guards?
├── YES -> axum-login. It layers on tower-sessions and handles
│          session-key bookkeeping for you.
└── NO  -> tower-sessions alone. Use it for CSRF tokens, flash
           messages, wizard state, a cart: any server-side state
           that is not a full authentication system.
```

## Patterns

### Pattern: tower-sessions setup and session state

```rust
// version-sensitive: tower-sessions 0.15
use tower_sessions::{Expiry, MemoryStore, Session, SessionManagerLayer};
use time::Duration;

let store = MemoryStore::default();              // dev only, NEVER production
let session_layer = SessionManagerLayer::new(store)
    .with_secure(true)                           // true behind TLS; false only for local HTTP
    .with_expiry(Expiry::OnInactivity(Duration::seconds(600)));

let app = Router::new().route("/", get(handler)).layer(session_layer);

// the Session extractor is FromRequestParts, so it is just a handler argument
async fn handler(session: Session) {
    let count: i32 = session.get("count").await.unwrap().unwrap_or(0);
    session.insert("count", count + 1).await.unwrap();
}
```

ALWAYS use a persistent `SessionStore` in production (`tower-sessions-sqlx-store`
or `tower-sessions-redis-store`). `MemoryStore` loses every session on restart
and is not shared across instances.

### Pattern: rotate the session id on login

```rust
// version-sensitive: tower-sessions 0.15
async fn login(session: Session, Form(creds): Form<Credentials>) -> impl IntoResponse {
    let user = verify_credentials(&creds).await; // your check, constant-time hash compare
    match user {
        Some(user) => {
            session.cycle_id().await.unwrap();   // ALWAYS rotate id on privilege change
            session.insert("user_id", user.id).await.unwrap();
            Redirect::to("/dashboard").into_response()
        }
        None => StatusCode::UNAUTHORIZED.into_response(),
    }
}

async fn logout(session: Session) -> impl IntoResponse {
    session.delete().await.unwrap();             // destroy server-side state
    Redirect::to("/").into_response()
}
```

ALWAYS call `cycle_id()` right after a successful login. Rotating the session ID
on a privilege change defeats session-fixation attacks. `axum-login`'s `login()`
rotates the ID for you.

### Pattern: axum-login full login system

`axum-login` needs an `AuthnBackend` implementation and the `AuthUser` trait on
your user type. The full backend implementation is in `references/examples.md`.
Wiring:

```rust
// version-sensitive: axum-login 0.18, tower-sessions 0.15
use axum_login::{login_required, AuthManagerLayerBuilder};
use tower_sessions::{MemoryStore, SessionManagerLayer};

let session_layer = SessionManagerLayer::new(MemoryStore::default());
let backend = Backend::new(db_pool);                       // your AuthnBackend impl
let auth_layer = AuthManagerLayerBuilder::new(backend, session_layer).build();

let app = Router::new()
    .route("/protected", get(protected_handler))
    .route_layer(login_required!(Backend, login_url = "/login"))  // guards routes ABOVE
    .merge(public_routes)
    .layer(auth_layer);
```

The login handler uses the `AuthSession` extractor directly:

```rust
// version-sensitive: axum-login 0.18
async fn login(mut auth: AuthSession<Backend>, Form(creds): Form<Credentials>)
    -> impl IntoResponse
{
    let user = match auth.authenticate(creds).await {
        Ok(Some(user)) => user,
        Ok(None)       => return StatusCode::UNAUTHORIZED.into_response(),
        Err(_)         => return StatusCode::INTERNAL_SERVER_ERROR.into_response(),
    };
    if auth.login(&user).await.is_err() {
        return StatusCode::INTERNAL_SERVER_ERROR.into_response();
    }
    Redirect::to("/protected").into_response()
}
```

ALWAYS apply `login_required!` with `.route_layer(...)`, NEVER `.layer(...)`.
`route_layer` runs only on matched routes, so an unmatched-path 404 stays a 404
instead of becoming a login redirect. Routes added AFTER the `route_layer` call
are not protected by it: place guarded routes above it.

### Pattern: HTTP Basic auth (TLS only)

```rust
// version-sensitive on the headers re-export path
use axum_extra::TypedHeader;
use axum_extra::headers::{authorization::Basic, Authorization};

async fn admin(
    TypedHeader(auth): TypedHeader<Authorization<Basic>>,
) -> Result<&'static str, Response> {
    if auth.username() == "admin" && verify_hash(auth.password()) {
        Ok("ok")
    } else {
        // 401 + WWW-Authenticate so the browser re-prompts for credentials
        Err((
            StatusCode::UNAUTHORIZED,
            [(header::WWW_AUTHENTICATE, "Basic realm=\"admin\"")],
        ).into_response())
    }
}
```

`TypedHeader` requires the `typed-header` feature on `axum-extra`. Basic auth
sends the password base64-encoded, NOT encrypted. ALWAYS serve Basic-protected
endpoints over TLS only. ALWAYS compare the password with a constant-time check
against a hash (`argon2`), NEVER with `==` against a plaintext literal. Use
Basic only for machine-to-machine or internal admin endpoints, NEVER as the
primary login for a human-facing browser app.

### Pattern: OAuth2 authorization-code flow

OAuth2 delegates auth to an external provider; your app needs exactly two
routes. The full two-route code is in `references/examples.md`. The shape:

```rust
// version-sensitive: oauth2 5.0
// redirect route: build the provider URL, store csrf + pkce verifier in the session
let (pkce_challenge, pkce_verifier) = PkceCodeChallenge::new_random_sha256();
let (auth_url, csrf_token) = client
    .authorize_url(CsrfToken::new_random)
    .add_scope(Scope::new("read".to_string()))
    .set_pkce_challenge(pkce_challenge)
    .url();
// store csrf_token + pkce_verifier in the tower-session, then Redirect::to(auth_url)

// callback route: verify state == stored csrf_token, THEN exchange the code
let token = client
    .exchange_code(AuthorizationCode::new(returned_code))
    .set_pkce_verifier(pkce_verifier)
    .request_async(&http_client)
    .await?;
// then load/create the local user and establish a session
```

ALWAYS verify the returned `state` equals the stored `CsrfToken` BEFORE
exchanging the code; skipping it opens a login-CSRF hole. ALWAYS use PKCE, even
for confidential clients. NEVER hardcode `client_secret`; read it from the
environment and fail fast if absent.

## Common Mistakes

| Mistake | Result | Fix |
|---------|--------|-----|
| `MemoryStore` in production | Sessions vanish on restart, not shared across instances | Persistent `SessionStore` (sqlx / Redis crate) |
| `with_secure(false)` in production | Session cookie sent over plain HTTP, sniffable | `with_secure(true)` behind TLS |
| No `cycle_id()` on login | Session-fixation hole | Rotate id after authentication |
| Basic auth over plain HTTP | Password effectively cleartext | TLS only |
| Skip OAuth2 `state` check | Login-CSRF hole | Verify `state == CsrfToken` before code exchange |
| `login_required!` on `.layer()` | A 404 becomes an auth redirect | Use `.route_layer(...)` |
| Hardcoded `client_secret` / signing key | Credential baked into binary and git history | Read from environment |

Full anti-pattern analysis with WHY each fails: `references/anti-patterns.md`.

## Reference Links

- `references/methods.md` : complete API signatures for `tower-sessions`,
  `axum-login`, `axum-extra` `TypedHeader`, and the `oauth2` crate, with feature
  flags and verified versions.
- `references/examples.md` : full working code, including a complete
  `AuthnBackend` implementation, the OAuth2 two-route flow, and `Cargo.toml`.
- `references/anti-patterns.md` : the seven session-auth anti-patterns with
  root-cause explanations and fixes.

Related skills:

- `axum-impl-auth-jwt` : stateless bearer-token auth, the JWT branch of the
  decision tree.
- `axum-impl-authorization` : role and permission checks after authentication.
- `axum-syntax-responses` : `Redirect`, status codes, and `IntoResponse`.

Verified sources (2026-05-20):

- https://docs.rs/tower-sessions/latest/tower_sessions/
- https://docs.rs/axum-login/latest/axum_login/
- https://docs.rs/axum-extra/latest/axum_extra/typed_header/index.html
- https://docs.rs/oauth2/latest/oauth2/
