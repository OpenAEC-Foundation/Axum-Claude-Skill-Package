# Research Fragment C : Errors, Auth, Database, Ops

Verified deep-research fragment for the Axum skill package, covering cluster C
(error handling, extractor rejections, validation, authentication,
authorization, database integration, testing, tracing, graceful shutdown,
OpenAPI, performance/deployment). All API claims, signatures, and code
examples below were verified via WebFetch against official documentation and
the canonical `tokio-rs/axum` example crates on 2026-05-20. Target versions:
Axum **0.7** and **0.8**.

Version note that affects almost every section: Axum **0.8** removed the
`#[async_trait]` macro from `FromRequest` and `FromRequestParts`. Native
RPITIT (return-position `impl Trait` in traits) replaced it. Skills covering
custom extractors MUST show both forms: with `#[async_trait]` for 0.7, without
it for 0.8. Axum 0.8 also changed route-parameter syntax from `:param` to
`{param}` and `*wildcard` to `{*wildcard}`.

---

## 1. Error Handling

Axum's central design rule is that **handlers must be infallible at the
service level**. The official `error_handling` module documentation states
that "axum makes sure you always produce a response by relying on the type
system" and that all services must have `Infallible` as their error type. This
is the single most important mental model: there is no such thing as a handler
that "fails" in the `tower::Service` sense. A handler that returns
`Result<T, E>` does not actually fail. It returns `Err(E)` where `E`
implements `IntoResponse`, and that `Err` is still turned into a valid HTTP
response. For example `async fn handler() -> Result<String, StatusCode>` is a
perfectly infallible service: `Err(StatusCode::NOT_FOUND)` produces a 404
response via `StatusCode`'s `IntoResponse` implementation.

The `IntoResponse` trait is the linchpin. Any type that implements
`IntoResponse` can be returned from a handler. This lets you define a
domain-specific error enum and use the `?` operator freely inside handlers.
The canonical pattern, verified against the official `examples/error-handling`
crate, defines an application error enum and implements `IntoResponse` for it:

```rust
#[derive(Debug)]
enum AppError {
    JsonRejection(JsonRejection),
    TimeError(time_library::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        #[derive(Serialize)]
        struct ErrorResponse { message: String }

        let (status, message, err) = match &self {
            AppError::JsonRejection(rejection) => {
                (rejection.status(), rejection.body_text(), None)
            }
            AppError::TimeError(_err) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "Something went wrong".to_owned(),
                Some(self),
            ),
        };

        let mut response =
            (status, AppJson(ErrorResponse { message })).into_response();
        if let Some(err) = err {
            response.extensions_mut().insert(Arc::new(err));
        }
        response
    }
}
```

The `?` operator works because of `From` implementations that convert
foreign error types into `AppError`:

```rust
impl From<JsonRejection> for AppError {
    fn from(rejection: JsonRejection) -> Self { Self::JsonRejection(rejection) }
}
impl From<time_library::Error> for AppError {
    fn from(error: time_library::Error) -> Self { Self::TimeError(error) }
}
```

Inside a handler you then simply write `let created_at = Timestamp::now()?;`
and the `time_library::Error` is converted to `AppError` automatically, which
in turn becomes an HTTP response. Note the example also stashes the original
error into `response.extensions_mut()` wrapped in `Arc`, so a middleware
layer can log it without exposing internals to the client.

**`anyhow` vs `thiserror`.** Use `thiserror` when you want a structured,
exhaustive error enum that maps each variant to a specific HTTP status code,
which is the right choice for library-quality APIs and for any error type that
implements `IntoResponse`. Use `anyhow` when you want a single opaque
"something went wrong" error type for application glue code where you do not
care about distinguishing variants. The standard `anyhow` pattern in Axum is a
newtype wrapper, because you cannot implement `IntoResponse` for `anyhow::Error`
directly (orphan rule): wrap it as `struct AppError(anyhow::Error)`, implement
`IntoResponse` on the wrapper (returning a 500), and add a blanket
`impl<E: Into<anyhow::Error>> From<E> for AppError`. In practice many
production Axum services combine both: `thiserror` for known, status-mapped
errors and an `anyhow`-backed catch-all variant for everything else.

**Fallible services and `HandleError`.** A `tower::Service` used as middleware
*can* legitimately fail with a non-`Infallible` error type, and Axum's router
will not accept it directly. The `HandleError` struct
(`pub struct HandleError<S, F>`) adapts a fallible service by running an error
handler function that converts the error into a response, transforming the
service error type to `Infallible`. The handler signature looks like
`async fn handle_anyhow_error(err: anyhow::Error) -> (StatusCode, String)`. To
apply this to a `tower::Layer` stack, use `HandleErrorLayer`
(`pub struct HandleErrorLayer<F>`). The error handler given to
`HandleErrorLayer` can also accept extractors such as `Method` and `Uri` in
addition to the error, with the error always passed as the final argument.

---

## 2. Extractor Rejections

Every extractor that can fail has an associated **rejection type** that itself
implements `IntoResponse`. The `FromRequest` and `FromRequestParts` traits
both have a `type Rejection` associated type. Verified signatures:

```rust
// consumes the body, must be the LAST handler argument
trait FromRequest<S> {
    type Rejection;
    async fn from_request(req: Request, state: &S)
        -> Result<Self, Self::Rejection>;
}

// does not touch the body, can appear anywhere
trait FromRequestParts<S> {
    type Rejection;
    async fn from_request_parts(parts: &mut Parts, state: &S)
        -> Result<Self, Self::Rejection>;
}
```

The body is an asynchronous stream that can only be consumed once, so Axum
enforces that the last extractor implements `FromRequest` and all preceding
ones implement `FromRequestParts`. Common rejection types live in the
`axum::extract::rejection` module: `JsonRejection`, `PathRejection`,
`QueryRejection`, `FormRejection`, `BytesRejection`, `StringRejection`,
`ExtensionRejection`, and so on. These types are marked `#[non_exhaustive]`,
so any `match` on their variants needs a catch-all arm. `JsonRejection`, for
example, has the variants `MissingJsonContentType`, `JsonDataError`,
`JsonSyntaxError`, and `BytesRejection`.

To customize a rejection response without writing a whole custom extractor,
wrap the extractor in `Result<T, T::Rejection>`. A handler argument typed
`Result<Json<Value>, JsonRejection>` lets you `match` on the rejection inside
the handler and return whatever response you want. You can also walk the
`std::error::Error` source chain and `downcast` to reach the underlying
`serde_json::Error` for fine-grained diagnostics.

**The `WithRejection` pattern.** The `axum-extra` crate provides
`WithRejection<E, R>`, an extractor wrapper that runs extractor `E` but, on
failure, converts its rejection into your own type `R` (any type that
implements `IntoResponse` and `From<E::Rejection>`). This is the cleanest way
to globally remap, for instance, every `JsonRejection` to a unified
`ApiError` JSON shape without touching each handler. Axum 0.8 additionally
made `Option<T>` an extractor backed by the new `OptionalFromRequestParts` /
`OptionalFromRequest` traits, which lets a missing-but-otherwise-valid
extraction yield `None` while a genuine extraction error still becomes a
rejection.

**Deriving custom rejections.** `axum-macros` provides
`#[derive(FromRequest)]` and `#[derive(FromRequestParts)]` with a
`#[from_request(rejection(MyRejection))]` attribute, so a struct that bundles
several extractors can declare one unified rejection type. When implementing
an extractor by hand, do NOT implement both `FromRequest` and
`FromRequestParts` directly for the same concrete type: the official docs warn
this makes the extractor unusable, unless the type is a generic wrapper around
another extractor (in which case implementing both is correct and intended).

---

## 3. Validation

Axum has no built-in request-body validation. The idiomatic solution is the
`validator` crate (current release 0.20.x) plus a small custom extractor. The
`validator` crate exposes a `#[derive(Validate)]` macro and a `.validate()`
method that returns `Result<(), ValidationErrors>`. Verified attribute syntax:

```rust
use validator::Validate;

#[derive(serde::Deserialize, Validate)]
struct SignupData {
    #[validate(email)]
    mail: String,
    #[validate(url)]
    site: String,
    #[validate(length(min = 1, max = 100))]
    first_name: String,
    #[validate(range(min = 18, max = 65))]
    age: u32,
    #[validate(custom(function = "validate_unique_username"))]
    username: String,
}
```

Other attributes verified from the docs: `contains`, `does_not_contain`,
`regex`, `must_match`, `credit_card` (behind the `card` feature), and `nested`
for recursive validation of sub-structs. A custom validator is a free function
returning `Result<(), ValidationError>`.

The recommended skill pattern is a `ValidatedJson<T>` custom extractor that
deserializes the JSON body and then runs validation, returning a clean 422 on
failure:

```rust
struct ValidatedJson<T>(T);

impl<T, S> FromRequest<S> for ValidatedJson<T>
where
    T: serde::de::DeserializeOwned + validator::Validate,
    S: Send + Sync,
    Json<T>: FromRequest<S, Rejection = JsonRejection>,
{
    type Rejection = (StatusCode, String);

    async fn from_request(req: Request, state: &S)
        -> Result<Self, Self::Rejection>
    {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|r| (r.status(), r.body_text()))?;
        value.validate()
            .map_err(|e| (StatusCode::UNPROCESSABLE_ENTITY, e.to_string()))?;
        Ok(ValidatedJson(value))
    }
}
```

In Axum 0.7 this `impl` block needs `#[async_trait]`; in 0.8 it does not. The
`garde` crate is a modern alternative to `validator` with a similar derive but
a stricter, context-aware API (`#[garde(...)]` attributes, `validate(&ctx)`);
it is worth mentioning as an option but `validator` is the more widely adopted
default for the skill package's examples.

---

## 4. Authentication

**JWT.** The canonical JWT flow uses the `jsonwebtoken` crate (the official
`examples/jwt` Cargo.toml pins `jsonwebtoken = "10"` with the `aws_lc_rs`
feature, `axum-extra` for the typed `Authorization` header, and `tokio = "1.0"`
with `full`). A `Claims` struct derives `Serialize`/`Deserialize` and an
`exp` field is mandatory because `Validation::default()` checks expiry:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims { sub: String, company: String, exp: usize }
```

Authentication is implemented by extracting and verifying the token in a
`FromRequestParts` extractor, so any handler that takes `Claims` as an argument
is automatically protected:

```rust
impl<S> FromRequestParts<S> for Claims
where S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &S)
        -> Result<Self, Self::Rejection>
    {
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;

        let token_data = decode::<Claims>(
            bearer.token(), &KEYS.decoding, &Validation::default(),
        ).map_err(|_| AuthError::InvalidToken)?;

        Ok(token_data.claims)
    }
}
```

`KEYS` wraps an `EncodingKey` and a `DecodingKey`, both built from a secret via
`EncodingKey::from_secret` / `DecodingKey::from_secret`. The secret MUST come
from an environment variable, never a literal. Token creation on the login
route uses `encode(&Header::default(), &claims, &KEYS.encoding)`. `AuthError`
is an enum (`WrongCredentials`, `MissingCredentials`, `TokenCreation`,
`InvalidToken`) with an `IntoResponse` impl mapping each variant to a status
code, exactly the error-handling pattern from section 1. The protected handler
is simply `async fn protected(claims: Claims) -> Result<String, AuthError>`.

**Session-based auth.** Instead of a self-contained token, store a session ID
in a cookie and keep server-side session state. The `tower-sessions` crate
provides a `SessionManagerLayer` plus pluggable session stores (in-memory,
Postgres, Redis); a handler extracts the `Session` and reads/writes typed
values. `axum-login` builds on `tower-sessions` to provide a full
`AuthSession` extractor, login/logout helpers, and a `login_required!` route
guard macro. Both are worth mentioning in the auth skill as the ecosystem
standard for stateful auth.

**HTTP Basic auth.** `axum-extra` exposes `TypedHeader<Authorization<Basic>>`,
giving `.username()` and `.password()`. Verify credentials in a
`FromRequestParts` extractor and return a `401` with a
`WWW-Authenticate: Basic` header on failure. Use Basic only over TLS and only
for machine-to-machine or admin endpoints.

**OAuth2.** The `oauth2` crate models the authorization-code flow: build a
`BasicClient` with client ID/secret and the authorization + token endpoint
URLs, call `.authorize_url()` (with a CSRF token and PKCE challenge) to
redirect the user to the provider, then in the callback route exchange the
returned `code` via `.exchange_code(...).request_async(...)` for an access
token. Axum's role is two routes: one that redirects, one callback that
completes the exchange and establishes a session.

---

## 5. Authorization

Authentication answers "who are you"; authorization answers "are you allowed".
Once `Claims` (carrying a role or a permission set) are available, enforce
access with middleware. The `axum::middleware::from_fn_with_state` function
turns an async function into a layer that can read application state and the
request. Role-based access control checks the role inside that middleware:

```rust
async fn require_admin(
    claims: Claims,         // runs the JWT extractor first
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    if claims.role != "admin" {
        return Err(StatusCode::FORBIDDEN);
    }
    Ok(next.run(request).await)
}

let admin_routes = Router::new()
    .route("/admin/users", get(list_users))
    .route_layer(middleware::from_fn(require_admin));
```

Use `.route_layer(...)` rather than `.layer(...)` for guards so the middleware
applies only to matched routes and a 404 (unmatched path) is not intercepted
by an auth check. Permission-based access control is the same pattern but
checks a fine-grained capability (for example `claims.permissions.contains
("users:write")`) instead of a coarse role; this is preferable because it
decouples endpoints from a fixed role hierarchy. Per-route guards can also be
done with a dedicated extractor type, for example a `RequirePermission<const
P: &str>` extractor or a `AdminClaims` newtype whose `FromRequestParts`
rejects non-admins, so the requirement is visible in the handler signature.

---

## 6. Database Integration

Use `sqlx` with a connection pool created **once** at startup and shared as
Axum state. The official `examples/sqlx-postgres` crate builds the pool with
`PgPoolOptions`:

```rust
let pool = PgPoolOptions::new()
    .max_connections(5)
    .acquire_timeout(Duration::from_secs(3))
    .connect(&db_connection_str)
    .await
    .expect("can't connect to database");

let app = Router::new()
    .route("/", get(handler))
    .with_state(pool);
```

`Pool` implements `Clone`, and the official docs are explicit that "cloning
`Pool` is cheap as it is simply a reference-counted handle to the inner pool
state" and a clone is "tied to the same shared connection pool". This is why
you do NOT need to wrap a `PgPool` in `Arc` for Axum state. `PgPool` is the
type alias for `Pool<Postgres>`; `MySqlPool` and `SqlitePool` are the analogues.

Two ways to run queries. Pass the pool reference directly:
`sqlx::query_scalar("select 1").fetch_one(&pool).await`. Or write a custom
`FromRequestParts` extractor that checks out a dedicated connection per
request:

```rust
struct DatabaseConnection(sqlx::pool::PoolConnection<sqlx::Postgres>);

impl<S> FromRequestParts<S> for DatabaseConnection
where PgPool: FromRef<S>, S: Send + Sync,
{
    type Rejection = (StatusCode, String);
    async fn from_request_parts(_parts: &mut Parts, state: &S)
        -> Result<Self, Self::Rejection>
    {
        let pool = PgPool::from_ref(state);
        let conn = pool.acquire().await.map_err(internal_error)?;
        Ok(Self(conn))
    }
}
```

A checked-out connection is used as `&mut *conn`. `acquire()`'s wait time is
bounded by `acquire_timeout`; if it elapses you get a pool error. When the pool
is at capacity, additional `acquire()` callers wait fairly in a queue.

**Query macros vs functions.** `query!`, `query_as!`, and `query_scalar!` are
compile-time-checked: they connect to a real database (via `DATABASE_URL` or an
offline `.sqlx` cache) at build time and verify SQL syntax and input/output
types. `query()`, `query_as()`, and `query_scalar()` are the runtime functions
with no compile-time checking, used for dynamic SQL. `query_as!` takes a path
to an explicitly defined output struct. Result methods: `fetch_one` (errors if
no row), `fetch_optional` (`Option`), `fetch_all` (`Vec`), `execute` (returns
affected-row count).

**Transactions.** `pool.begin().await?` returns a `Transaction<'_, DB>` (alias
`PgTransaction`). Run queries against `&mut *tx`, then `tx.commit().await?` to
persist or `tx.rollback().await?` to discard. If a `Transaction` is dropped
without `commit`, it rolls back automatically, which is a safe default.

**Pool sizing.** `max_connections` is the hard upper bound. Size it so that
`(number of app instances) x max_connections` stays comfortably below the
database server's `max_connections` limit, leaving headroom for migrations and
admin tooling. A common starting point is the number of CPU cores on the
database server; oversizing increases lock contention rather than throughput.

---

## 7. Testing

There are two complementary approaches. The `axum-test` crate (MIT) wraps a
`Router` in a `TestServer`:

```rust
use axum_test::TestServer;

let app = Router::new().route("/users", put(handler));
let server = TestServer::new(app).unwrap();

let response = server.post("/users")
    .json(&serde_json::json!({ "username": "John" }))
    .await;
response.assert_status_ok();
response.assert_json(&serde_json::json!({ "id": 1 }));
```

Assertion methods include `assert_status_ok()` (200), `assert_json()`, and
`assert_text()`. `TestServerConfig` / `TestServerBuilder` customize transport
mode (mock vs a real HTTP socket) and other behavior, and `TestServerConfig`
implements `Default`.

The lower-level `tower` oneshot pattern needs no extra crate, because a
`Router` already implements `tower::Service<Request<Body>>`. The official
`examples/testing` crate uses it:

```rust
use tower::ServiceExt;        // for `oneshot`
use http_body_util::BodyExt; // for `collect`

let response = app
    .oneshot(Request::get("/").body(Body::empty()).unwrap())
    .await
    .unwrap();
assert_eq!(response.status(), StatusCode::OK);
let body = response.into_body().collect().await.unwrap().to_bytes();
assert_eq!(&body[..], b"Hello, World!");
```

`oneshot` consumes the router and serves exactly one request, ideal for fast
unit-style tests. For JSON, set `CONTENT_TYPE` to `application/json` and build
the body with `serde_json::to_vec`. Integration tests live in a `tests/`
directory; a shared `app()` constructor function returns the configured
`Router` so both unit tests and integration tests build the same app.

---

## 8. Tracing and Logging

`tracing` is the structured-logging and instrumentation framework; `tracing-
subscriber` collects and formats the data. Verified setup from the official
`examples/tracing-aka-logging` crate:

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
use tower_http::trace::TraceLayer;

tracing_subscriber::registry()
    .with(tracing_subscriber::EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| format!(
            "{}=debug,tower_http=debug,axum::rejection=trace",
            env!("CARGO_CRATE_NAME")).into()))
    .with(tracing_subscriber::fmt::layer())
    .init();
```

`EnvFilter` reads the `RUST_LOG` environment variable; the fallback string sets
sensible defaults. `tower-http`'s `TraceLayer::new_for_http()` instruments
every request automatically and is added with `.layer(TraceLayer::new_for_http
())`. It can be customized with `.make_span_with(...)`, `.on_request(...)`,
`.on_response(...)`, `.on_body_chunk(...)`, `.on_eos(...)`, and
`.on_failure(...)`. A custom span typically reads `MatchedPath` from request
extensions so logs group by route template rather than concrete URL.

For structured logging in handler code, use the event macros `error!`,
`warn!`, `info!`, `debug!`, `trace!` with `key = value` fields. The `?` sigil
formats a field with `Debug`, the `%` sigil with `Display`. The `#[instrument]`
attribute macro wraps a function in a span named after the function and records
its arguments as fields. Critical async warning from the docs: never hold the
guard from `Span::enter()` across an `.await` point because it produces
incorrect traces; use `#[instrument]` or `Future::instrument` instead.

**OpenTelemetry export.** Add the `opentelemetry`, `opentelemetry-otlp`, and
`tracing-opentelemetry` crates. Build an OTLP tracer pipeline, wrap it in
`tracing_opentelemetry::layer()`, and add that layer to the same
`tracing_subscriber::registry()` shown above alongside the `fmt` layer. Spans
created by `TraceLayer` and `#[instrument]` are then exported to any
OTLP-compatible backend (Jaeger, Tempo, Honeycomb). Propagating the incoming
`traceparent` header into the request span gives end-to-end distributed
tracing.

---

## 9. Graceful Shutdown

`axum::serve(listener, app).with_graceful_shutdown(future)` stops accepting new
connections when `future` resolves and waits for in-flight requests to finish
before returning. Verified from the official `examples/graceful-shutdown`
crate, the shutdown signal future handles both SIGINT and SIGTERM:

```rust
async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv().await;
    };
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

```rust
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await
    .unwrap();
```

SIGTERM matters because that is the signal Docker, Kubernetes, and systemd send
on stop, and the default is to kill the process immediately and drop every
in-flight request. The graceful-shutdown example also pairs the server with a
`TimeoutLayer` so a request that never finishes (`std::future::pending`) cannot
block shutdown forever. In a real service, on shutdown you also close the
database pool (`pool.close().await`) and flush the tracing/OTLP exporter.

---

## 10. OpenAPI

`utoipa` generates an OpenAPI 3 spec from annotations, so the documentation
stays in sync with the code. Verified macro surface:

- `#[derive(ToSchema)]` on a struct generates a reusable schema component.
- `#[utoipa::path(...)]` on a handler documents one endpoint: HTTP method,
  `path = "..."`, a `responses(...)` list with `(status, description, body)`
  tuples, and `params(...)` for path/query parameters.
- `#[derive(OpenApi)]` with `#[openapi(paths(...), components(schemas(...)))]`
  assembles the root document.

```rust
#[derive(ToSchema, Serialize)]
struct Pet { id: u64, name: String, age: Option<i32> }

#[utoipa::path(
    get,
    path = "/pets/{id}",
    responses(
        (status = 200, description = "Pet found", body = Pet),
        (status = NOT_FOUND, description = "Pet not found")
    ),
    params(("id" = u64, Path, description = "Pet database id"))
)]
async fn get_pet_by_id(/* ... */) { /* ... */ }

#[derive(OpenApi)]
#[openapi(paths(get_pet_by_id), components(schemas(Pet)))]
struct ApiDoc;
```

The `utoipa-swagger-ui` crate serves the interactive UI. With the `axum`
feature it provides `SwaggerUi`, merged into the router with something like
`.merge(SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json",
ApiDoc::openapi()))`. The companion crate `utoipa-axum` provides
`OpenApiRouter`, which collects `#[utoipa::path]` handlers and the spec
together so routing and documentation cannot drift apart. Note `utoipa::path`'s
`path` argument uses the `{param}` brace syntax, which is also Axum 0.8's
native route syntax, so 0.8 routes and utoipa annotations match exactly.

---

## 11. Performance and Deployment

The cardinal async rule: **never block the Tokio runtime**. Each runtime
worker thread runs many futures cooperatively; a synchronous blocking call (a
slow `std::fs` read, a CPU-bound loop, a blocking `rusqlite`/Mutex call, a
`std::thread::sleep`) inside an `async` handler stalls that entire worker
thread and starves every other future scheduled on it. The fixes: use async
equivalents (`tokio::fs`, `sqlx`, `tokio::time::sleep`) for I/O, and wrap
genuine CPU-bound work in `tokio::task::spawn_blocking(...)`, which runs the
closure on a separate, larger blocking thread pool and returns a `JoinHandle`
you `.await`. For long-running CPU work prefer `rayon` or a dedicated thread
pool, again awaited asynchronously.

Connection pool tuning ties back to section 6: size `max_connections` against
the database's global limit divided by instance count, and set
`acquire_timeout` so a saturated pool fails fast with a 503 instead of hanging.

Build for production with `cargo build --release`; the dev profile is far
slower. For a fully static, dependency-free binary, build against the
`x86_64-unknown-linux-musl` target (`rustup target add
x86_64-unknown-linux-musl` then `cargo build --release --target
x86_64-unknown-linux-musl`). A musl-static binary needs no glibc and can run in
a `scratch` or `distroless` container.

A multi-stage Docker build keeps the image small: a `rust` builder stage
compiles the release binary (ideally with `cargo-chef` to cache the dependency
layer so source-only changes do not recompile every crate), then a minimal
runtime stage (`debian:bookworm-slim`, `distroless`, or `scratch` for a musl
binary) copies only the compiled binary. Final images shrink from ~1.5 GB to a
few tens of MB. Bind the listener to `0.0.0.0` inside containers, drive
configuration from environment variables, and pair the container with the
SIGTERM-aware graceful shutdown from section 9 so orchestrator-initiated
restarts and rolling deploys never drop requests.

---

## Anti-Patterns (Cluster C)

**AP-1 : Blocking call inside an async handler starves the runtime.**
Calling a synchronous blocking operation directly in an `async fn` handler (a
blocking file write, a `std::thread::sleep`, a CPU-heavy loop, or a sync
`Mutex` guarding a blocking `rusqlite` query) freezes the Tokio worker thread
that future is running on. Multiple `tokio-rs/axum` discussions document this:
a synchronous `save_to_file()` called under a `Mutex` stalls the handler future
under concurrent traffic, and handlers that acquire a sync `Mutex` then run
blocking SQLite queries block the whole Tokio worker. WHY it fails: Tokio
schedules many futures per worker thread cooperatively; a blocking call never
yields, so every other future on that thread is frozen until it returns,
collapsing throughput and tail latency. FIX: use async I/O, or move CPU work to
`tokio::task::spawn_blocking` and `.await` the `JoinHandle`.

**AP-2 : Creating a new database pool per request.**
Calling `PgPoolOptions::new()...connect()` inside a handler builds a brand-new
connection pool on every request. WHY it fails: a database has a finite
`max_connections` limit; a per-request pool opens fresh TCP+TLS connections
(50-100 ms each for Postgres over TLS) and quickly exhausts the server's
connection slots, after which every request fails with a connection error. It
also throws away the entire benefit of pooling. FIX: build one pool in `main()`
and share it via `.with_state(pool)`. `PgPool` is a cheap, reference-counted
`Clone`, so no per-request allocation and no `Arc` wrapper are needed.

**AP-3 : `.unwrap()` / `.expect()` in a handler instead of an error response.**
Writing `let user = db_query().await.unwrap();` in a handler turns any runtime
error (a missing row, a dropped connection) into a `panic!`. WHY it fails: a
panic in a handler aborts that request's task; the client receives an abruptly
closed connection or an opaque 500 with no useful body, and the panic noise
pollutes logs. It conflates "this row does not exist" (a normal 404) with
"the process is broken". FIX: return `Result<T, AppError>`, convert errors with
`?` and `From` impls, and let the `IntoResponse` impl from section 1 produce a
proper status code and JSON body. Reserve `unwrap`/`expect` for genuinely
unrecoverable startup-time invariants such as binding the listener.

**AP-4 : Missing graceful shutdown drops in-flight requests.**
Serving with plain `axum::serve(listener, app).await` and no
`.with_graceful_shutdown(...)` means the process dies the instant it receives
SIGTERM. WHY it fails: SIGTERM is exactly the signal Docker, Kubernetes, and
systemd send on stop, restart, and rolling deploy; without a handler the
runtime is killed mid-request, so every in-flight response is lost, database
transactions may be left half-applied, and clients see connection resets on
every single deployment. FIX: pass a `shutdown_signal()` future that selects
over `ctrl_c()` and the Unix `SignalKind::terminate()` stream to
`.with_graceful_shutdown(...)`, so the server stops accepting new connections
and drains existing ones first.

**AP-5 : Hardcoding the JWT (or database) secret in source.**
Writing `Keys::new(b"super-secret-key")` or an inline `DATABASE_URL` literal
bakes a credential into the binary and the git history. WHY it fails: anyone
with read access to the repository or the compiled binary can forge valid JWTs
or read the database; rotating the secret then requires a recompile and
redeploy; and the secret leaks into every CI artifact and Docker layer. FIX:
read secrets from environment variables at startup (`std::env::var
("JWT_SECRET")`), fail fast if absent, and supply them via the orchestrator's
secret store. Never commit a real secret, and add `.env` to `.gitignore`.

**AP-6 : Holding a `tracing` span guard across an `.await`.**
Calling `let _g = span.enter();` and then `.await`-ing inside that scope. WHY
it fails: the official `tracing` docs explicitly warn this "may produce
incorrect traces" because the span is exited and re-entered by other tasks each
time the future yields, so spans interleave and the recorded timing and
parent/child structure become wrong. FIX: annotate the async function with
`#[instrument]`, or attach the span with `Future::instrument`, which correctly
enters the span only while the future is polled.

---

## Newly Discovered Sub-Topics (Cluster C)

Topics surfaced during this research that are NOT yet explicitly captured by
the existing masterplan topic list (`axum-errors-handling`,
`axum-errors-extractor-rejections`, `axum-errors-trait-bounds`,
`axum-impl-auth`, `axum-impl-authorization`, `axum-impl-database`,
`axum-impl-validation`, `axum-impl-testing`, `axum-impl-tracing`,
`axum-impl-graceful-shutdown`, `axum-impl-openapi`, `axum-impl-deployment`,
`axum-core-performance`):

1. **`#[async_trait]` removal in Axum 0.8 (custom extractor migration).** Every
   custom-extractor example differs between 0.7 and 0.8. This is a cross-cutting
   version concern that belongs either in `axum-errors-trait-bounds` or in a
   dedicated `axum-core-versions` / migration skill, and it must be flagged in
   `axum-impl-auth`, `axum-impl-database`, and `axum-impl-validation` because
   each of those builds a custom extractor.

2. **Route-parameter syntax change `:param` -> `{param}` (0.7 -> 0.8).** A
   migration-relevant breaking change. Pairs naturally with item 1 in a
   version/migration skill, and also touches OpenAPI because `utoipa`'s
   `{param}` syntax now matches Axum 0.8 exactly.

3. **`Option<T>` as an extractor via `OptionalFromRequestParts` /
   `OptionalFromRequest` (new in 0.8).** A genuinely new extractor capability
   relevant to `axum-errors-extractor-rejections`; consider an explicit bullet
   there.

4. **`anyhow`-wrapper newtype pattern for `IntoResponse`.** The orphan-rule
   workaround (`struct AppError(anyhow::Error)` + blanket `From`) is a distinct
   sub-pattern that `axum-errors-handling` should cover alongside the
   `thiserror` enum pattern, since they are chosen for different situations.

5. **`tower-sessions` + `axum-login` ecosystem for stateful auth.** Beyond
   stateless JWT, the masterplan's `axum-impl-auth` should explicitly carve out
   session-based auth (cookie + server-side store) and the `axum-login` route
   guard, as these are the ecosystem standard and structurally different from
   JWT.

6. **`utoipa-axum` `OpenApiRouter`.** A newer integration crate that unifies
   routing and spec generation, distinct from the plain `utoipa` +
   `utoipa-swagger-ui` combination; worth an explicit mention in
   `axum-impl-openapi`.

7. **`cargo-chef` Docker layer caching.** A deployment-specific optimization
   for fast Rust container builds; belongs as a concrete sub-section under
   `axum-impl-deployment`.

8. **`garde` as a validation alternative to `validator`.** Mentioned only,
   per the cluster brief, but worth a one-line "alternatives" note in
   `axum-impl-validation`.

9. **OpenTelemetry export (`opentelemetry` + `tracing-opentelemetry`).**
   Distributed tracing export is a clear sub-topic of `axum-impl-tracing` and
   may warrant its own dedicated section given the extra crate stack and
   trace-context propagation concerns.

10. **`WithRejection` (axum-extra) and derive-based custom rejections.** A
    concrete, reusable mechanism that `axum-errors-extractor-rejections` should
    name explicitly rather than only describing the manual `Result<T,
    Rejection>` approach.

---

## Sources Verified (Cluster C)

All URLs below were fetched and verified on **2026-05-20**:

- https://docs.rs/axum/latest/axum/error_handling/index.html — infallible
  service model, `IntoResponse`, `HandleError`, `HandleErrorLayer`.
- https://docs.rs/axum/latest/axum/extract/index.html — `FromRequest` /
  `FromRequestParts` traits, rejection types, custom extractor rules,
  `JsonRejection` variants.
- https://github.com/tokio-rs/axum/blob/main/examples/error-handling/src/main.rs
  — `AppError` enum, `IntoResponse` impl, `From` impls, `?` operator usage.
- https://github.com/tokio-rs/axum/blob/main/examples/jwt/src/main.rs —
  `Claims` struct, `FromRequestParts` for `Claims`, `Keys`, `AuthError` +
  `IntoResponse`, protected handler.
- https://github.com/tokio-rs/axum/blob/main/examples/jwt/Cargo.toml —
  `jsonwebtoken = "10"` (aws_lc_rs), `serde = "1.0"`, `tokio = "1.0"`.
- https://github.com/tokio-rs/axum/blob/main/examples/sqlx-postgres/src/main.rs
  — `PgPoolOptions`, `max_connections`, `acquire_timeout`, `with_state(pool)`,
  `DatabaseConnection` custom extractor.
- https://github.com/tokio-rs/axum/blob/main/examples/graceful-shutdown/src/main.rs
  — `shutdown_signal()` (SIGINT + SIGTERM), `with_graceful_shutdown`,
  `TimeoutLayer`.
- https://github.com/tokio-rs/axum/blob/main/examples/testing/src/main.rs —
  `tower::ServiceExt::oneshot`, `http_body_util::BodyExt::collect`, status/body
  assertions.
- https://github.com/tokio-rs/axum/blob/main/examples/tracing-aka-logging/src/main.rs
  — `tracing_subscriber::registry()`, `EnvFilter`, `fmt::layer()`,
  `TraceLayer::new_for_http()`.
- https://docs.rs/sqlx/latest/sqlx/ — `query!` / `query_as!` / `query_scalar!`
  vs runtime functions, transactions, `fetch_one`/`fetch_optional`/`fetch_all`/
  `execute`.
- https://docs.rs/sqlx/latest/sqlx/struct.Pool.html — `Pool` is `Clone`,
  reference-counted, `acquire()`, `begin()`, sizing methods.
- https://docs.rs/axum-test/latest/axum_test/ — `TestServer`,
  `assert_status_ok`, `assert_json`, `assert_text`, `TestServerConfig`.
- https://docs.rs/utoipa/latest/utoipa/ — `#[derive(ToSchema)]`,
  `#[derive(OpenApi)]`, `#[openapi(...)]`, `#[utoipa::path(...)]`,
  `utoipa-swagger-ui` integration.
- https://docs.rs/validator/latest/validator/ — `#[derive(Validate)]`,
  validation attributes, custom validators, `ValidationErrors`.
- https://docs.rs/tracing/latest/tracing/ — `#[instrument]`, `span!`/`event!`
  macros, structured field syntax, async span-guard warning.
- https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0 — Axum 0.8 breaking
  changes: `async_trait` removal, route syntax `:param` -> `{param}`,
  `Option<T>` extractor traits.
- https://github.com/tokio-rs/axum/discussions/2045,
  https://github.com/tokio-rs/axum/discussions/2321,
  https://github.com/tokio-rs/axum/discussions/2508 — real-world anti-patterns:
  blocking calls starving the runtime, connection pooling done correctly.
