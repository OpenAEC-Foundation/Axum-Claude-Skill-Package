# axum-impl-auth-session : Anti-Patterns

The seven session-auth mistakes that cause data loss, credential leaks, and
security holes. Each entry states WHAT goes wrong, WHY it fails, and the FIX.

## Anti-pattern 1: MemoryStore in production

```rust
// WRONG: MemoryStore is non-persistent and single-process
let session_layer = SessionManagerLayer::new(MemoryStore::default());
```

WHY it fails: `MemoryStore` keeps sessions in process memory. Every server
restart (a deploy, a crash, a host migration) wipes all sessions, logging out
every user. It is also single-process: with two or more app instances behind a
load balancer, a request routed to a different instance does not find the
session, so the user appears logged out at random.

FIX: use a persistent, shared `SessionStore` in production.

```rust
// CORRECT: a Postgres-backed store survives restarts and is shared
let store = tower_sessions_sqlx_store::PostgresStore::new(pg_pool);
store.migrate().await?;
let session_layer = SessionManagerLayer::new(store);
```

`MemoryStore` is acceptable ONLY for local development.

## Anti-pattern 2: with_secure(false) in production

```rust
// WRONG: cookie sent over plain HTTP
let session_layer = SessionManagerLayer::new(store).with_secure(false);
```

WHY it fails: without the `Secure` attribute the browser sends the session
cookie over plain HTTP as well as HTTPS. Any network observer (a shared Wi-Fi,
a compromised router) reads the session ID off an HTTP request and replays it to
impersonate the user. The session ID is a bearer credential.

FIX: set `with_secure(true)` in production. `false` is acceptable ONLY for local
HTTP development where there is no TLS to attach the cookie to.

```rust
// CORRECT
let session_layer = SessionManagerLayer::new(store).with_secure(true);
```

## Anti-pattern 3: not rotating the session id on login

```rust
// WRONG: same session id before and after login
async fn login(session: Session, Form(creds): Form<Credentials>) -> impl IntoResponse {
    let user_id = verify(&creds).await.unwrap();
    session.insert("user_id", user_id).await.unwrap(); // id never rotated
    Redirect::to("/dashboard")
}
```

WHY it fails: this is a session-fixation hole. An attacker visits the site
unauthenticated, obtains a session ID, and tricks the victim into using that
same ID (via a planted cookie or a URL). When the victim logs in, the ID does
not change, so the attacker now holds a session ID tied to the victim's
authenticated session.

FIX: rotate the session ID on every privilege change. Call `cycle_id()` right
after a successful authentication; the data is preserved, only the ID changes.

```rust
// CORRECT
session.cycle_id().await.unwrap();
session.insert("user_id", user_id).await.unwrap();
```

`axum-login`'s `login()` rotates the ID for you; calling it is enough.

## Anti-pattern 4: HTTP Basic auth over plain HTTP

```rust
// WRONG: a Basic-protected route reachable over http://
async fn admin(TypedHeader(auth): TypedHeader<Authorization<Basic>>) -> &'static str {
    /* ... */ "ok"
}
```

WHY it fails: HTTP Basic transmits `username:password` base64-encoded in the
`Authorization` header on EVERY request. Base64 is an encoding, not encryption:
it is trivially reversible. Over plain HTTP the password is effectively in
cleartext on the wire, and it is resent on every single request, so one captured
request leaks the password permanently.

FIX: serve Basic-protected endpoints over TLS only. Terminate HTTPS in front of
the app and reject plain HTTP. Use Basic only for machine-to-machine or internal
admin endpoints, NEVER as the primary login for a human browser app.

## Anti-pattern 5: skipping the OAuth2 state / CSRF check

```rust
// WRONG: callback exchanges the code without checking state
async fn callback(Query(params): Query<CallbackParams>) -> Redirect {
    let token = client
        .exchange_code(AuthorizationCode::new(params.code))
        .request_async(&http_client)
        .await
        .unwrap();
    Redirect::to("/dashboard")
}
```

WHY it fails: the OAuth2 `state` parameter is a CSRF defense. Without verifying
that the returned `state` equals the `CsrfToken` your app generated and stored,
an attacker can feed the victim a callback URL carrying the attacker's own
authorization code. The victim's browser exchanges it, and the victim ends up
logged into the ATTACKER's account, a login-confusion attack.

FIX: store the `CsrfToken` server-side when building the authorize URL, and in
the callback verify `params.state == stored_csrf` BEFORE calling
`exchange_code`. Reject the request with `400` on mismatch. ALWAYS pair this
with PKCE (`PkceCodeChallenge::new_random_sha256`).

## Anti-pattern 6: guard macro applied with .layer() instead of .route_layer()

```rust
// WRONG: login_required! on .layer() guards everything, including unmatched paths
let app = Router::new()
    .route("/protected", get(handler))
    .layer(login_required!(Backend, login_url = "/login"));
```

WHY it fails: `.layer()` wraps the ENTIRE router, including the fallback path
for URLs that match no route. An unauthenticated request to a non-existent URL
should get a `404`. With the guard on `.layer()`, it instead gets redirected to
`/login`, which leaks the existence of the auth system on every bad URL and
breaks `404` semantics for clients and crawlers.

FIX: apply guard macros with `.route_layer(...)`. `route_layer` runs only on
routes that actually matched, so an unmatched path stays a `404`.

```rust
// CORRECT
let app = Router::new()
    .route("/protected", get(handler))
    .route_layer(login_required!(Backend, login_url = "/login"));
```

Note also: `login_required!` guards only routes added BEFORE the `route_layer`
call. Place guarded routes above it and public routes below it (or on a
separately merged router).

## Anti-pattern 7: hardcoded client_secret or session signing key

```rust
// WRONG: a credential baked into the source
let client = BasicClient::new(ClientId::new("id".into()))
    .set_client_secret(ClientSecret::new("super-secret-value".into()));
```

WHY it fails: a hardcoded `client_secret` or session signing key is committed to
git history forever. Anyone with read access to the repository, any leaked
clone, any CI log, gets the production credential. Rotating it then means a code
change and redeploy, so it rarely happens.

FIX: read every secret from the environment at startup and fail fast if it is
absent. Keep secrets in a secret manager or a `.env` file that is gitignored.

```rust
// CORRECT
let secret = std::env::var("OAUTH_CLIENT_SECRET")
    .expect("OAUTH_CLIENT_SECRET must be set");
let client = BasicClient::new(ClientId::new("id".into()))
    .set_client_secret(ClientSecret::new(secret));
```
