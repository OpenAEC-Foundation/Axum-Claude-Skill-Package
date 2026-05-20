# axum-impl-auth-session : Methods Reference

API signatures for session-based, Basic, and OAuth2 auth in Axum. All URLs
WebFetch-verified on 2026-05-20.

These crates are third-party and version-sensitive. Confirm every signature
against the version pinned in your `Cargo.toml`. Verified versions:

| Crate | Verified version | Source |
|-------|------------------|--------|
| `tower-sessions` | 0.15.0 | https://docs.rs/tower-sessions/latest/tower_sessions/ |
| `axum-login` | 0.18.0 | https://docs.rs/axum-login/latest/axum_login/ |
| `oauth2` | 5.0.0 | https://docs.rs/oauth2/latest/oauth2/ |
| `axum-extra` | current | https://docs.rs/axum-extra/latest/axum_extra/typed_header/index.html |

## tower-sessions 0.15

### SessionManagerLayer

A `tower::Layer` that loads a session for every request and writes the
`Set-Cookie` header back on the response.

```rust
SessionManagerLayer::new(store)   // store: any type implementing SessionStore
```

Builder methods (each consumes self, returns Self):

| Method | Effect |
|--------|--------|
| `with_secure(bool)` | sets the cookie `Secure` attribute; `true` in production |
| `with_expiry(Expiry)` | expiration strategy |
| `with_name(String)` | the cookie name, default `id` |
| `with_signed(key)` | feature-gated cookie signing |
| `with_private(key)` | feature-gated cookie encryption |

`Expiry` variants include `Expiry::OnInactivity(Duration)` and
`Expiry::OnSessionEnd`. `Duration` is `time::Duration`.

### Session

`Session` is a `FromRequestParts` extractor: it appears as any handler argument.
Values are serialized with serde. All methods are async.

| Method | Signature shape | Effect |
|--------|-----------------|--------|
| `insert` | `insert(key: &str, value: impl Serialize)` | store a value |
| `get` | `get::<T>(key: &str) -> Result<Option<T>>` | read a value |
| `remove` | `remove::<T>(key: &str) -> Result<Option<T>>` | drop one key |
| `clear` | `clear()` | drop all keys, keep the session row |
| `cycle_id` | `cycle_id()` | issue a new session id, keep the data |
| `flush` | `flush()` | destroy the server-side session |
| `delete` | `delete()` | mark the session for deletion |

### Session stores

| Store | Crate | Production safe |
|-------|-------|-----------------|
| `MemoryStore` | `tower-sessions` (feature `memory-store`) | NO, dev only |
| `CachingSessionStore` | `tower-sessions` | yes, with a persistent backend |
| sqlx store | `tower-sessions-sqlx-store` | yes (Postgres/MySQL/SQLite) |
| redis store | `tower-sessions-redis-store` | yes |

Any type implementing the `SessionStore` trait works as a store. `MemoryStore`
is non-persistent and single-process: it loses every session on restart and is
not shared across instances. NEVER use it in production.

## axum-login 0.18

Builds a full authentication system on top of `tower-sessions`.

### Traits to implement

| Trait | Required methods | Purpose |
|-------|------------------|---------|
| `AuthUser` | `id(&self)`, `session_auth_hash(&self)` | minimal user interface |
| `AuthnBackend` | `authenticate(creds)`, `get_user(user_id)` | authentication backend |
| `AuthzBackend` | permission queries (optional) | fine-grained authorization |

- `AuthUser::id` returns the user's ID type.
- `AuthUser::session_auth_hash` returns bytes used to invalidate the session
  when the user's credential (for example a password hash) changes.
- `AuthnBackend::authenticate` returns `Result<Option<Self::User>>`.
- `AuthnBackend::get_user` loads a user by ID for session restoration.

### AuthSession

An extractor, usable directly as a handler argument.

| Member | Shape | Effect |
|--------|-------|--------|
| `authenticate` | `authenticate(creds) -> Result<Option<User>>` | runs the backend |
| `login` | `login(&user) -> Result<()>` | establishes the session, rotates the id |
| `logout` | `logout() -> Result<Option<User>>` | clears the session |
| `user` | field, `Option<User>` | the currently authenticated user |

### AuthManagerLayer

```rust
AuthManagerLayerBuilder::new(backend, session_layer).build()
```

`backend` is your `AuthnBackend` implementation. `session_layer` is the
`tower-sessions` `SessionManagerLayer`. The auth layer requires the session
layer; they stack together.

### Guard macros

| Macro | Use |
|-------|-----|
| `login_required!(Backend, login_url = "/login")` | redirect unauthenticated requests |
| `permission_required!(Backend, "perm")` | require a specific permission |

Apply guards with `.route_layer(...)`, NOT `.layer(...)`. `route_layer` runs
only on matched routes, so an unmatched-path 404 stays a 404. Routes added after
the `route_layer` call are not protected by it.

## axum-extra TypedHeader (HTTP Basic)

`TypedHeader` is an extractor for strongly typed headers. It requires the
`typed-header` Cargo feature on `axum-extra`:

```toml
axum-extra = { version = "...", features = ["typed-header"] }
```

`TypedHeader<Authorization<Basic>>` parses an `Authorization: Basic ...` header.

| Method | Returns |
|--------|---------|
| `.username()` | the decoded username |
| `.password()` | the decoded password |

The companion `Authorization<Bearer>` parses a bearer token; that path is JWT
territory, owned by the `axum-impl-auth-jwt` skill. On a failed Basic check,
return `401` WITH a `WWW-Authenticate: Basic realm="..."` header so the client
knows to send credentials.

## oauth2 5.0

The `oauth2` crate models the OAuth2 authorization-code flow.

### Client construction

```rust
BasicClient::new(ClientId::new(id))
    .set_client_secret(ClientSecret::new(secret))
    .set_auth_uri(AuthUrl::new(auth_url)?)
    .set_token_uri(TokenUrl::new(token_url)?)
    .set_redirect_uri(RedirectUrl::new(redirect_url)?)
```

The 5.x builder takes the `ClientId` in `new(...)` and applies the rest with
typed setter methods. The 4.x API differed substantially; builder-style `new()`
is a 5.x shape.

### Authorization URL (redirect route)

| Call | Effect |
|------|--------|
| `client.authorize_url(CsrfToken::new_random)` | start the builder with a random CSRF token |
| `.add_scope(Scope::new(...))` | add a scope |
| `.set_pkce_challenge(challenge)` | attach a PKCE challenge |
| `.url()` | returns `(Url, CsrfToken)` |

`PkceCodeChallenge::new_random_sha256()` returns `(PkceCodeChallenge,
PkceCodeVerifier)`. Store the `CsrfToken` and the `PkceCodeVerifier` server-side
in the session, then redirect the browser to the URL.

### Token exchange (callback route)

| Call | Effect |
|------|--------|
| `client.exchange_code(AuthorizationCode::new(code))` | start the exchange |
| `.set_pkce_verifier(verifier)` | attach the stored PKCE verifier |
| `.request_async(&http_client).await?` | perform the HTTP exchange |

The result implements `TokenResponse`; `.access_token().secret()` reads the
access token. Verify the returned `state` equals the stored `CsrfToken` BEFORE
calling `exchange_code`.

## Feature flags summary

| Crate | Feature | Gates |
|-------|---------|-------|
| `tower-sessions` | `memory-store` | `MemoryStore` |
| `tower-sessions` | `signed` / `private` | `with_signed` / `with_private` |
| `axum-extra` | `typed-header` | `TypedHeader` extractor |
| `axum` | `form` | the `Form` extractor used by login handlers |
