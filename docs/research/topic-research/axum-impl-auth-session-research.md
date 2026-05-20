# Topic Research : axum-impl-auth-session

> Skill : `axum-impl-auth-session`
> Phase : 4 (topic research)
> Researched : 2026-05-20
> Target versions : Axum 0.7 and 0.8
> Scope : session-based auth (cookie session id + server-side store), `tower-sessions`,
> `axum-login`, HTTP Basic auth via `axum-extra`, OAuth2 authorization-code flow,
> and the four-way auth decision tree (JWT vs session vs basic vs OAuth2).

This file supports a single skill. It complements the JWT-focused material in
`fragments/research-c-errors-auth-db-ops.md` section 4 and `vooronderzoek-axum.md`
section 5; this skill owns stateful and delegated auth, JWT is owned elsewhere.

Third-party crate versions evolve faster than Axum. Where the exact method signature
could not be fully pinned from docs.rs, the snippet is explicitly marked
**version-sensitive**: skill authors MUST tell users to confirm the line against the
crate version they actually depend on.

---

## 1. Session-based auth : the model

A session-based scheme stores **no user data in the cookie**. The cookie carries only
an opaque, unguessable session ID; all real state (user id, roles, CSRF token, flash
messages) lives server-side in a session store. This is the structural opposite of JWT,
where the token IS the state and the server holds nothing.

Consequences that drive the decision tree:

- Sessions are **revocable instantly** : delete the store row and the cookie is dead.
  JWTs cannot be revoked before `exp` without an extra denylist.
- Sessions need a **store** (memory, Postgres, Redis) and one store round-trip per
  request. JWT verification is pure CPU, no I/O.
- Sessions are the natural fit for **server-rendered web apps** with a browser; the
  browser carries the cookie automatically.
- Mutable session data (cart, wizard step) is trivial server-side, awkward in a JWT.

Axum ships no session machinery. The ecosystem standard is `tower-sessions` for the raw
session layer and `axum-login` layered on top for full authentication.

---

## 2. tower-sessions : SessionManagerLayer and the Session API

`tower-sessions` (version documented on docs.rs : **0.15.0**) provides a `tower::Layer`
that loads a session for every request and writes the `Set-Cookie` header back.

**SessionManagerLayer** : constructed with `SessionManagerLayer::new(store)`, then
configured with builder methods. Verified builder methods:

- `with_secure(bool)` : sets the cookie `Secure` attribute. ALWAYS `true` in production
  (cookie only sent over HTTPS); `false` only acceptable for local HTTP dev.
- `with_expiry(Expiry)` : expiration strategy, e.g. `Expiry::OnInactivity(...)` or
  `Expiry::OnSessionEnd`.
- `with_name(String)` : the cookie name (default `id`).
- `with_signed(key)` / `with_private(key)` : feature-gated cookie signing / encryption.

**Session stores** : `MemoryStore` (built-in, non-persistent, single-process only,
NEVER for production), `CachingSessionStore` (cache layer over a backend), and any type
implementing the `SessionStore` trait. Production stores are separate crates :
`tower-sessions-sqlx-store` (Postgres/MySQL/SQLite), `tower-sessions-redis-store`, etc.
ALWAYS use a persistent store in production; `MemoryStore` loses every session on
restart and does not work across multiple instances.

**Session API** (the `Session` extractor, a `FromRequestParts` extractor, so it can
appear as any handler argument) : `insert(key, value)`, `get(key)` -> `Option<T>`,
`remove(key)`, `clear()`, `cycle_id()`, `flush()`, `delete()`. Values are serialized
with serde.

Verified setup snippet (from the tower-sessions docs, **version-sensitive** :
confirm `Expiry` / `Duration` imports against 0.15.x) :

```rust
use tower_sessions::{Expiry, MemoryStore, Session, SessionManagerLayer};
use time::Duration;

let session_store = MemoryStore::default();
let session_layer = SessionManagerLayer::new(session_store)
    .with_secure(false)                                    // true in production
    .with_expiry(Expiry::OnInactivity(Duration::seconds(600)));

let app = Router::new()
    .route("/", get(handler))
    .layer(session_layer);
```

Reading and writing session state inside a handler :

```rust
async fn handler(session: Session) {
    let count: i32 = session.get("count").await.unwrap().unwrap_or(0);
    session.insert("count", count + 1).await.unwrap();
}
```

ALWAYS call `cycle_id()` right after a successful login. Rotating the session ID on
privilege change defeats session-fixation attacks. The login route writes the user id
into the session AND rotates the ID; the logout route calls `delete()` (or `flush()`)
to destroy server-side state.

---

## 3. axum-login : AuthSession, AuthnBackend, login_required!

`axum-login` (version documented on docs.rs : **0.18.0**, MIT) builds a full
authentication system on top of `tower-sessions`. It supplies a typed user, login /
logout helpers, and route-guard macros, so you do not hand-roll session-key handling.

**Three traits you implement :**

- `AuthUser` : minimal user interface. Required methods : `id(&self)` returning the
  user's ID type, and `session_auth_hash(&self)` returning bytes used to invalidate the
  session when the user's credential (e.g. password hash) changes.
- `AuthnBackend` : authentication backend. Required methods : `authenticate(credentials)`
  returning `Result<Option<Self::User>>`, and `get_user(user_id)` loading the user.
- `AuthzBackend` (optional) : adds user and group permission queries for authorization.

**AuthSession** : an extractor (usable directly as a handler argument). Methods :
`authenticate(credentials)` (runs the backend), `login(&user)` (establishes the
session), `logout()` (clears it), and the `user` field holding `Option<User>`.

**AuthManagerLayer** : the middleware that injects `AuthSession`. Built with
`AuthManagerLayerBuilder::new(backend, session_layer).build()` where `session_layer` is
the `tower-sessions` `SessionManagerLayer` from section 2. The auth layer requires the
session layer; they are stacked together.

**login_required! macro** : protects routes that need an authenticated user. Applied
with `.route_layer(...)` so it does NOT intercept unmatched paths (a 404 must stay a 404,
not become a redirect). Unauthenticated requests are redirected to `login_url`.
`permission_required!` is the analogous macro for fine-grained permission checks.

Verified setup snippet (from the axum-login docs, **version-sensitive** : the macro
argument shape and builder method names can change between 0.x releases) :

```rust
use axum_login::{login_required, AuthManagerLayerBuilder};
use tower_sessions::{MemoryStore, SessionManagerLayer};

let session_store = MemoryStore::default();
let session_layer = SessionManagerLayer::new(session_store);

let backend = Backend::default();                          // your AuthnBackend impl
let auth_layer = AuthManagerLayerBuilder::new(backend, session_layer).build();

let app = Router::new()
    .route("/protected", get(protected_handler))
    .route_layer(login_required!(Backend, login_url = "/login"))
    .merge(public_routes)
    .layer(auth_layer);
```

A login handler uses the `AuthSession` extractor directly :

```rust
async fn login(mut auth_session: AuthSession<Backend>, Form(creds): Form<Credentials>)
    -> impl IntoResponse
{
    let user = match auth_session.authenticate(creds.clone()).await {
        Ok(Some(user)) => user,
        Ok(None)  => return StatusCode::UNAUTHORIZED.into_response(),
        Err(_)    => return StatusCode::INTERNAL_SERVER_ERROR.into_response(),
    };
    if auth_session.login(&user).await.is_err() {
        return StatusCode::INTERNAL_SERVER_ERROR.into_response();
    }
    Redirect::to("/protected").into_response()
}
```

Place `login_required!` ABOVE the routes it must guard and BELOW the auth layer in the
stack; routes added after the `route_layer` call are not protected by it.

---

## 4. HTTP Basic auth : axum-extra TypedHeader<Authorization<Basic>>

`axum-extra` exposes `TypedHeader`, an extractor for strongly typed headers from the
`headers` crate. It requires the **`typed-header`** Cargo feature on `axum-extra`
(`axum-extra = { version = "...", features = ["typed-header"] }`).

`TypedHeader<Authorization<Basic>>` parses an `Authorization: Basic ...` header and
gives `.username()` and `.password()`. The companion `Authorization<Bearer>` parses a
bearer token (that path is JWT territory, owned by the JWT skill).

Basic auth sends the password (base64-encoded, NOT encrypted) on every request. ALWAYS
serve Basic-protected endpoints over TLS only; over plain HTTP the credentials are
effectively in cleartext. Use Basic only for machine-to-machine or internal admin
endpoints, NEVER as the primary login for a human-facing browser app.

Verified handler shape (the `TypedHeader` extractor pattern is stable;
**version-sensitive** on which `headers` re-export path `Authorization` / `Basic` come
from) :

```rust
use axum_extra::TypedHeader;
use axum_extra::headers::{authorization::Basic, Authorization};

async fn admin(
    TypedHeader(auth): TypedHeader<Authorization<Basic>>,
) -> Result<&'static str, Response> {
    if auth.username() == "admin" && verify(auth.password()) {
        Ok("ok")
    } else {
        // 401 + WWW-Authenticate so the browser re-prompts
        Err((
            StatusCode::UNAUTHORIZED,
            [(header::WWW_AUTHENTICATE, "Basic realm=\"admin\"")],
        ).into_response())
    }
}
```

ALWAYS compare the password with a constant-time check against a hash (e.g. `argon2`),
NEVER with `==` against a plaintext literal. On failure return `401` WITH a
`WWW-Authenticate: Basic` header, otherwise the client cannot know to send credentials.

---

## 5. OAuth2 authorization-code flow with the oauth2 crate

The `oauth2` crate (version documented on docs.rs : **5.0.0**) models the OAuth2
authorization-code flow. Auth is delegated to an external provider (Google, GitHub,
your own IdP); your app never sees the user's password. Axum's job is exactly **two
routes** : one that redirects the browser to the provider, one callback that exchanges
the returned `code` for a token.

**Client construction** : `BasicClient::new(ClientId)` then chained setters
`set_client_secret`, `set_auth_uri` (`AuthUrl`), `set_token_uri` (`TokenUrl`),
`set_redirect_uri` (`RedirectUrl`). The 5.x builder takes the `ClientId` in `new(...)`
and applies the rest with typed setter methods.

**Redirect route** : `client.authorize_url(CsrfToken::new_random)` starts the builder,
`.add_scope(Scope::new(...))` adds scopes, `.set_pkce_challenge(...)` attaches a PKCE
challenge, `.url()` returns `(Url, CsrfToken)`. Store the `CsrfToken` and the PKCE
verifier server-side (in the tower-session) and redirect the browser to the URL.

**Callback route** : verify the returned `state` equals the stored `CsrfToken` (CSRF
defense), then `client.exchange_code(AuthorizationCode::new(code))`,
`.set_pkce_verifier(pkce_verifier)`, `.request_async(&http_client).await?` to obtain the
token. Then load/create the local user and establish a session (section 2 / 3).

Verified flow snippet (from the oauth2 5.0.0 docs, **version-sensitive** : the 4.x API
differed substantially, builder-style `new()` is a 5.x shape) :

```rust
use oauth2::{AuthorizationCode, AuthUrl, ClientId, ClientSecret, CsrfToken,
    PkceCodeChallenge, RedirectUrl, Scope, TokenResponse, TokenUrl};
use oauth2::basic::BasicClient;

let client = BasicClient::new(ClientId::new("client_id".to_string()))
    .set_client_secret(ClientSecret::new("client_secret".to_string()))
    .set_auth_uri(AuthUrl::new("https://provider/authorize".to_string())?)
    .set_token_uri(TokenUrl::new("https://provider/token".to_string())?)
    .set_redirect_uri(RedirectUrl::new("https://myapp/auth/callback".to_string())?);

// --- redirect route ---
let (pkce_challenge, pkce_verifier) = PkceCodeChallenge::new_random_sha256();
let (auth_url, csrf_token) = client
    .authorize_url(CsrfToken::new_random)
    .add_scope(Scope::new("read".to_string()))
    .set_pkce_challenge(pkce_challenge)
    .url();
// store csrf_token + pkce_verifier in the session, then Redirect::to(auth_url)

// --- callback route (verify state == csrf_token first) ---
let http_client = oauth2::reqwest::ClientBuilder::new()
    .redirect(oauth2::reqwest::redirect::Policy::none())
    .build()?;

let token = client
    .exchange_code(AuthorizationCode::new(returned_code))
    .set_pkce_verifier(pkce_verifier)
    .request_async(&http_client)
    .await?;
let access_token = token.access_token().secret();
```

ALWAYS verify the `state` parameter against the stored `CsrfToken` before exchanging
the code; skipping it opens a CSRF / login-confusion hole. ALWAYS use PKCE
(`PkceCodeChallenge::new_random_sha256`) even for confidential clients. NEVER hardcode
`client_secret`; read it from the environment.

---

## 6. The four-way auth decision tree : JWT vs session vs Basic vs OAuth2

```
Need to authenticate a request. Pick ONE primary mechanism.

1. Is auth delegated to an external identity provider
   (Google / GitHub / corporate SSO)?
      YES -> OAuth2 (oauth2 crate, authorization-code + PKCE).
             After callback, establish a LOCAL session for the browser.
      NO  -> continue.

2. Is the caller a browser using a server-rendered web app,
   or do you need instant revocation / mutable server-side state?
      YES -> Session-based auth.
             - raw layer only        -> tower-sessions
             - full login system     -> axum-login on top of tower-sessions
      NO  -> continue.

3. Is the caller a machine / internal admin tool over TLS,
   and is a simple username+password per request acceptable?
      YES -> HTTP Basic (axum-extra TypedHeader<Authorization<Basic>>).
             TLS ONLY. Not for human browser login.
      NO  -> continue.

4. Is the caller a stateless API client (SPA, mobile, service-to-service)
   where every request carries its own credential and you can accept
   that tokens are valid until exp?
      YES -> JWT (jsonwebtoken crate, Authorization<Bearer>).
             See the axum-impl-auth-jwt skill.
```

Tie-breakers and combinations :

- OAuth2 is an **identity** mechanism, not a session mechanism. After the callback you
  still pick session OR JWT to carry the logged-in state. OAuth2 + session is the
  normal browser combination.
- Session vs JWT, the deciding question is **revocation and state** : need to kill a
  login immediately, or store mutable per-user data -> session. Need zero server state
  and horizontal scale with no shared store -> JWT.
- Basic is the narrowest tool : machine-to-machine over TLS, never the primary login
  for end users.
- NEVER mix two primary mechanisms on the same route. One route, one auth extractor.

---

## 7. Anti-patterns specific to this skill

1. **`MemoryStore` in production** : sessions vanish on restart and are not shared
   across instances. Use a persistent `SessionStore` (sqlx / Redis store crate).
2. **`with_secure(false)` in production** : the session cookie then travels over plain
   HTTP and can be sniffed. ALWAYS `true` behind TLS.
3. **Not rotating the session ID on login** : leaves a session-fixation hole. Call
   `cycle_id()` (or rely on `axum-login`'s `login`, which rotates) right after a
   successful authentication.
4. **Basic auth over plain HTTP** : the password is base64, not encrypted, so it is
   effectively cleartext. TLS only.
5. **Skipping the OAuth2 `state` / CSRF check** : exchanging the code without verifying
   `state == CsrfToken` opens a login-CSRF hole.
6. **Guard macros applied with `.layer()` instead of `.route_layer()`** : `login_required!`
   on `.layer()` can turn an unmatched-path 404 into an auth redirect. Use `route_layer`.
7. **Hardcoded OAuth2 `client_secret` / session signing key** : bakes a credential into
   the binary and git history. Read from the environment, fail fast if absent.

---

## Sources verified

All URLs below were fetched and verified on **2026-05-20** :

- https://docs.rs/tower-sessions/latest/tower_sessions/ — `SessionManagerLayer::new`,
  builder methods (`with_secure`, `with_expiry`, `with_name`, `with_signed`,
  `with_private`), the `Session` API (`insert`, `get`, `remove`, `clear`, `cycle_id`,
  `flush`, `delete`), `MemoryStore` / `CachingSessionStore` / `SessionStore` trait.
  Documented version : tower-sessions 0.15.0.
- https://docs.rs/axum-login/latest/axum_login/ — `AuthSession` extractor
  (`authenticate`, `login`, `logout`, `user`), `AuthManagerLayerBuilder::new(...).build()`,
  the `AuthUser` / `AuthnBackend` / `AuthzBackend` traits, the `login_required!` and
  `permission_required!` macros. Documented version : axum-login 0.18.0.
- https://docs.rs/axum-extra/latest/axum_extra/typed_header/index.html — `TypedHeader`
  extractor, `TypedHeaderRejection` / `TypedHeaderRejectionReason`, the required
  `typed-header` feature flag.
- https://docs.rs/oauth2/latest/oauth2/ — `BasicClient::new` + typed setters
  (`set_client_secret`, `set_auth_uri`, `set_token_uri`, `set_redirect_uri`),
  `authorize_url` with `CsrfToken::new_random` / `add_scope` / `set_pkce_challenge`,
  `PkceCodeChallenge::new_random_sha256`, `exchange_code` / `set_pkce_verifier` /
  `request_async`. Documented version : oauth2 5.0.0.
- Cross-reference (not re-fetched, used for the JWT branch of the decision tree) :
  `docs/research/fragments/research-c-errors-auth-db-ops.md` section 4 and
  `docs/research/vooronderzoek-axum.md` section 5.

### Verification caveats

- The exact builder-method names of `tower-sessions` and `axum-login` and the precise
  argument shape of `login_required!` and `BasicClient::new` evolve across 0.x / 5.x
  releases. Every third-party snippet above is marked **version-sensitive**; the skill
  MUST instruct users to confirm each line against the crate version pinned in their
  `Cargo.toml` (tower-sessions 0.15.x, axum-login 0.18.x, oauth2 5.0.x as documented).
- All four crates are independent of Axum's own 0.7 vs 0.8 split. The Axum-side concern
  is only that custom extractors built around these crates follow the 0.8 native-async
  trait form (no `#[async_trait]`); the crates' own APIs do not differ by Axum version.
