# axum-impl-auth-session : Examples

Working session-auth code. The four crates here are third-party and
version-sensitive: every snippet is marked with the crate version it was
verified against on 2026-05-20. Confirm each line against the version pinned in
your `Cargo.toml`. The crates are independent of the Axum 0.7 vs 0.8 split, so
there are no Axum-version-divergent snippets here.

## Example 1: tower-sessions counter

A minimal session that counts visits. Demonstrates the `Session` extractor and
`get` / `insert`.

```rust
// version-sensitive: tower-sessions 0.15
use axum::{routing::get, Router};
use tower_sessions::{Expiry, MemoryStore, Session, SessionManagerLayer};
use time::Duration;

#[tokio::main]
async fn main() {
    let session_layer = SessionManagerLayer::new(MemoryStore::default())
        .with_secure(true)    // true behind TLS; false only for local HTTP dev
        .with_name("id")
        .with_expiry(Expiry::OnInactivity(Duration::seconds(600)));

    let app = Router::new()
        .route("/", get(counter))
        .layer(session_layer);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn counter(session: Session) -> String {
    let count: i32 = session.get("count").await.unwrap().unwrap_or(0);
    session.insert("count", count + 1).await.unwrap();
    format!("visit {}", count + 1)
}
```

## Example 2: login and logout with session-id rotation

```rust
// version-sensitive: tower-sessions 0.15
use axum::{extract::Form, response::{IntoResponse, Redirect}, http::StatusCode};
use serde::Deserialize;
use tower_sessions::Session;

#[derive(Deserialize)]
struct Credentials {
    username: String,
    password: String,
}

async fn login(session: Session, Form(creds): Form<Credentials>) -> impl IntoResponse {
    // verify_credentials does a constant-time hash comparison (argon2), never ==
    match verify_credentials(&creds).await {
        Some(user_id) => {
            // ALWAYS rotate the session id right after a successful login
            session.cycle_id().await.unwrap();
            session.insert("user_id", user_id).await.unwrap();
            Redirect::to("/dashboard").into_response()
        }
        None => StatusCode::UNAUTHORIZED.into_response(),
    }
}

async fn logout(session: Session) -> impl IntoResponse {
    session.delete().await.unwrap();   // destroy the server-side session
    Redirect::to("/").into_response()
}

// a guarded handler reads the user id back out of the session
async fn dashboard(session: Session) -> Result<String, StatusCode> {
    let user_id: i64 = session
        .get("user_id")
        .await
        .unwrap()
        .ok_or(StatusCode::UNAUTHORIZED)?;
    Ok(format!("user {user_id}"))
}
```

## Example 3: a complete axum-login AuthnBackend

The backend that `axum-login` drives. This is the part you implement.

```rust
// version-sensitive: axum-login 0.18
use axum_login::{AuthUser, AuthnBackend, UserId};
use serde::Deserialize;

#[derive(Debug, Clone)]
struct User {
    id: i64,
    pw_hash: Vec<u8>,   // an argon2 hash, never a plaintext password
}

impl AuthUser for User {
    type Id = i64;

    fn id(&self) -> Self::Id {
        self.id
    }

    // bytes that change when the credential changes; invalidates old sessions
    fn session_auth_hash(&self) -> &[u8] {
        &self.pw_hash
    }
}

#[derive(Debug, Clone, Deserialize)]
struct Credentials {
    username: String,
    password: String,
}

#[derive(Debug, Clone)]
struct Backend {
    db: sqlx::PgPool,
}

impl Backend {
    fn new(db: sqlx::PgPool) -> Self {
        Self { db }
    }
}

// axum-login 0.18 uses the native async-trait form; no #[async_trait] needed
impl AuthnBackend for Backend {
    type User = User;
    type Credentials = Credentials;
    type Error = sqlx::Error;

    async fn authenticate(
        &self,
        creds: Self::Credentials,
    ) -> Result<Option<Self::User>, Self::Error> {
        let user = load_user_by_name(&self.db, &creds.username).await?;
        Ok(user.filter(|u| verify_password(&creds.password, &u.pw_hash)))
    }

    async fn get_user(
        &self,
        user_id: &UserId<Self>,
    ) -> Result<Option<Self::User>, Self::Error> {
        load_user_by_id(&self.db, *user_id).await
    }
}
```

## Example 4: wiring axum-login and guarding routes

```rust
// version-sensitive: axum-login 0.18, tower-sessions 0.15
use axum::{routing::get, Router};
use axum_login::{login_required, AuthManagerLayerBuilder, AuthSession};
use tower_sessions::{MemoryStore, SessionManagerLayer};

fn build_app(db: sqlx::PgPool) -> Router {
    let session_layer = SessionManagerLayer::new(MemoryStore::default())
        .with_secure(true);
    let auth_layer =
        AuthManagerLayerBuilder::new(Backend::new(db), session_layer).build();

    Router::new()
        .route("/protected", get(protected))
        // login_required! guards routes ABOVE this line; route_layer, not layer
        .route_layer(login_required!(Backend, login_url = "/login"))
        .route("/login", get(login_form).post(login))
        .route("/logout", get(logout))
        .layer(auth_layer)
}

async fn protected(auth: AuthSession<Backend>) -> String {
    match auth.user {
        Some(user) => format!("hello user {}", user.id),
        None => "unreachable: route is guarded".to_string(),
    }
}

async fn login(
    mut auth: AuthSession<Backend>,
    axum::extract::Form(creds): axum::extract::Form<Credentials>,
) -> axum::response::Response {
    use axum::{http::StatusCode, response::{IntoResponse, Redirect}};

    let user = match auth.authenticate(creds).await {
        Ok(Some(user)) => user,
        Ok(None) => return StatusCode::UNAUTHORIZED.into_response(),
        Err(_) => return StatusCode::INTERNAL_SERVER_ERROR.into_response(),
    };
    // login() establishes the session and rotates the session id for you
    if auth.login(&user).await.is_err() {
        return StatusCode::INTERNAL_SERVER_ERROR.into_response();
    }
    Redirect::to("/protected").into_response()
}

async fn logout(mut auth: AuthSession<Backend>) -> axum::response::Redirect {
    let _ = auth.logout().await;
    axum::response::Redirect::to("/")
}
```

## Example 5: HTTP Basic auth handler

```rust
// version-sensitive on the headers re-export path
use axum::{
    http::{header, StatusCode},
    response::{IntoResponse, Response},
};
use axum_extra::TypedHeader;
use axum_extra::headers::{authorization::Basic, Authorization};

async fn admin(
    TypedHeader(auth): TypedHeader<Authorization<Basic>>,
) -> Result<&'static str, Response> {
    // verify_hash does a constant-time argon2 comparison, never == on plaintext
    if auth.username() == "admin" && verify_hash(auth.password()) {
        Ok("admin area")
    } else {
        Err((
            StatusCode::UNAUTHORIZED,
            // WWW-Authenticate makes the browser re-prompt for credentials
            [(header::WWW_AUTHENTICATE, "Basic realm=\"admin\"")],
        )
            .into_response())
    }
}
```

`Cargo.toml`: `axum-extra = { version = "...", features = ["typed-header"] }`.
Serve this endpoint over TLS only: Basic auth base64-encodes the password, it
does NOT encrypt it.

## Example 6: OAuth2 authorization-code flow, two routes

```rust
// version-sensitive: oauth2 5.0
use oauth2::{
    basic::BasicClient, AuthorizationCode, AuthUrl, ClientId, ClientSecret,
    CsrfToken, PkceCodeChallenge, RedirectUrl, Scope, TokenResponse, TokenUrl,
};

fn oauth_client() -> Result<BasicClient, Box<dyn std::error::Error>> {
    // NEVER hardcode the secret; read it from the environment, fail fast if absent
    let secret = std::env::var("OAUTH_CLIENT_SECRET")?;
    let client = BasicClient::new(ClientId::new("client_id".to_string()))
        .set_client_secret(ClientSecret::new(secret))
        .set_auth_uri(AuthUrl::new("https://provider/authorize".to_string())?)
        .set_token_uri(TokenUrl::new("https://provider/token".to_string())?)
        .set_redirect_uri(RedirectUrl::new(
            "https://myapp/auth/callback".to_string(),
        )?);
    Ok(client)
}

// --- redirect route: send the browser to the provider ---
async fn oauth_redirect(
    session: tower_sessions::Session,
) -> axum::response::Redirect {
    let client = oauth_client().unwrap();
    let (pkce_challenge, pkce_verifier) = PkceCodeChallenge::new_random_sha256();
    let (auth_url, csrf_token) = client
        .authorize_url(CsrfToken::new_random)
        .add_scope(Scope::new("read".to_string()))
        .set_pkce_challenge(pkce_challenge)
        .url();

    // store csrf_token and pkce_verifier server-side for the callback to check
    session.insert("csrf", csrf_token.secret()).await.unwrap();
    session
        .insert("pkce", pkce_verifier.secret())
        .await
        .unwrap();
    axum::response::Redirect::to(auth_url.as_str())
}

// --- callback route: verify state, then exchange the code ---
#[derive(serde::Deserialize)]
struct CallbackParams {
    code: String,
    state: String,
}

async fn oauth_callback(
    session: tower_sessions::Session,
    axum::extract::Query(params): axum::extract::Query<CallbackParams>,
) -> Result<axum::response::Redirect, axum::http::StatusCode> {
    use axum::http::StatusCode;
    use oauth2::{PkceCodeVerifier, reqwest};

    // ALWAYS verify state == stored CsrfToken BEFORE exchanging the code
    let stored_csrf: String = session
        .get("csrf")
        .await
        .unwrap()
        .ok_or(StatusCode::BAD_REQUEST)?;
    if params.state != stored_csrf {
        return Err(StatusCode::BAD_REQUEST);
    }

    let pkce: String = session
        .get("pkce")
        .await
        .unwrap()
        .ok_or(StatusCode::BAD_REQUEST)?;

    let http_client = reqwest::ClientBuilder::new()
        .redirect(reqwest::redirect::Policy::none())
        .build()
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let client = oauth_client().map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    let token = client
        .exchange_code(AuthorizationCode::new(params.code))
        .set_pkce_verifier(PkceCodeVerifier::new(pkce))
        .request_async(&http_client)
        .await
        .map_err(|_| StatusCode::UNAUTHORIZED)?;

    let _access_token = token.access_token().secret();
    // load or create the local user, then establish a session (Example 2 / 4)
    session.cycle_id().await.unwrap();
    Ok(axum::response::Redirect::to("/dashboard"))
}
```

## Cargo.toml dependencies

```toml
[dependencies]
axum = "0.8"                       # or "0.7"; session auth is version-independent
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }

# session auth (version-sensitive: pin these and confirm the API):
tower-sessions = "0.15"
axum-login = "0.18"
axum-extra = { version = "0.10", features = ["typed-header"] }
oauth2 = "5.0"
time = "0.3"

# a persistent session store for production (pick one):
# tower-sessions-sqlx-store = { version = "...", features = ["postgres"] }
# tower-sessions-redis-store = "..."
```
