# Topic Research : axum-core-architecture

> Skill : `axum-core-architecture`
> Phase : 4 (topic research)
> Target versions : Axum 0.7 and 0.8 (docs.rs current : 0.8.9)
> Researched / verified : 2026-05-20
> Sources : `docs.rs/axum` (crate overview, `fn.serve`, `serve/struct.Serve`), vooronderzoek-axum.md, fragment A.

This file supports the `axum-core-architecture` skill only. It covers Axum as a
composition over tokio + hyper + tower, the request lifecycle, the no-macros design,
`axum::serve` as the glue, the `axum-core` crate boundary, and when to choose Axum.

---

## 1. What Axum Is : A Composition, Not a Monolith

Axum is an async web framework for Rust, built and maintained by the Tokio team. Its
single most important architectural fact: **Axum implements almost nothing itself.** It
composes three lower layers, and the skill must lead with this.

- **tokio** : the async runtime. Provides the executor, `tokio::net::TcpListener`,
  timers, and `tokio::sync` primitives. Handler futures must be `Send` because tokio's
  multi-threaded scheduler can move tasks between worker threads.
- **hyper** : the HTTP/1 and HTTP/2 protocol implementation. Hyper parses raw socket
  bytes into `http::Request<Body>` and serializes `http::Response<Body>` back.
- **tower** : supplies the `Service` trait, the universal abstraction
  `async fn(Request) -> Result<Response, Error>`. Everything routable in Axum is
  ultimately a `tower::Service`. The crate overview states it directly: "axum doesn't
  have its own middleware system but instead uses `tower::Service`." Speaking `Service`
  means Axum inherits the entire tower / tower-http middleware ecosystem (timeouts,
  tracing, compression, CORS, auth, rate limiting) for free.

The crate overview lists Axum's five high-level features verbatim: "Route requests to
handlers with a macro-free API", declarative request parsing with extractors, a simple
error handling model, minimal-boilerplate responses, and full leverage of the tower
ecosystem.

---

## 2. The `axum::serve` Signature (verified 2026-05-20)

`axum::serve` is the glue that binds a listener to a service. The exact signature on
`docs.rs/axum/latest/axum/fn.serve.html`:

```rust
pub fn serve<L, M, S>(listener: L, make_service: M) -> Serve<L, M, S>
where
    L: Listener,
    M: for<'a> Service<IncomingStream<'a, L>, Error = Infallible, Response = S>,
    S: Service<Request, Response = Response, Error = Infallible> + Clone + Send + 'static,
    S::Future: Send,
```

Notes the skill should carry:

- Available only when crate features `tokio` and (`http1` or `http2`) are enabled.
- It takes a `Listener` (in practice `tokio::net::TcpListener`) and a make-service.
  A `Router`, `MethodRouter`, or `Handler` becomes a make-service via
  `.into_make_service()` (or is accepted directly for a serveable `Router<()>`).
- The returned `Serve<L, M, S>` future resolves to `io::Result<()>` but
  **"will never actually complete or return an error."** This is why `main` is an
  infinite-loop server until killed.
- `Serve` has two methods: `local_addr()` (returns `Result<L::Addr>`) and
  `with_graceful_shutdown(signal)` where
  `signal: F where F: Future<Output = ()> + Send + 'static`. The shutdown variant
  returns `Ok(())` only after the `signal` future completes.

**Version note for the skill: the `axum::serve` signature and the minimal `main()`
below are IDENTICAL on Axum 0.7 and 0.8.** `serve` was introduced in 0.7 as the
replacement for the removed `axum::Server`, and its shape has not changed across the
0.7 -> 0.8 boundary. The skill must state this explicitly so authors do not add a
needless version split here.

---

## 3. Minimal Verified `main()` (0.7 and 0.8, identical)

```rust
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "Hello, World!" }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

The entry point is `#[tokio::main] async fn main()` : the macro expands to a `fn main()`
that builds a tokio runtime and blocks on the async body. This is the only macro most
Axum apps ever use, and it comes from tokio, not Axum.

---

## 4. The Request Lifecycle (ordered, verified)

The skill must present the lifecycle as an ordered list. Verified against the crate
overview's request-flow description:

1. `tokio::net::TcpListener` accepts an incoming TCP connection.
2. **hyper** reads bytes off the socket and parses them into `http::Request<Body>`.
3. `axum::serve` drives the `Router` as a `tower::Service` (calls `Service::call`).
4. The `Router` matches the request path and selects a `MethodRouter`.
5. The `MethodRouter` selects the handler registered for the request's HTTP method
   (or its 405 fallback if no method matches).
6. Each handler argument runs its extractor : `FromRequestParts` for parts-only
   extractors, `FromRequest` for the single body-consuming extractor (which must be
   the last argument).
7. The handler `async fn` body executes and returns a value.
8. The return value runs `IntoResponse::into_response`, producing `http::Response<Body>`.
9. The response flows back out through **hyper**, which serializes it to socket bytes.

Middleware (`tower::Layer` / `tower::Service`) wraps steps 3-8 : an outer layer sees
the request before the router and the response after the handler.

---

## 5. The "No Macros" Design Philosophy and Its Trade-Off

Axum deliberately exposes a macro-free routing and handler API. There is no
`#[get("/path")]` attribute as in Rocket or Actix.

- **Routes are ordinary values.** `Router::new().route("/", get(handler))` is plain
  function calls producing data. Routing composes, can be returned from functions,
  merged, nested, and unit-tested without a macro-expansion step.
- **Handlers are ordinary `async fn`s.** Eligibility is enforced purely by trait bounds
  (the `Handler` trait), not by an attribute macro.
- **Benefit** : no macro expansion to obscure compiler errors; IDE tooling, go-to-
  definition, and rustc spans all work normally.
- **The trade-off (the skill's key gotcha)** : when a function fails the `Handler`
  trait bounds, the compiler emits a notoriously cryptic error of the form
  `the trait bound 'fn(...) {handler}: Handler<_, _>' is not satisfied`. The error does
  not say *which* requirement failed.
- **The mitigation** : Axum ships exactly one macro for this, `#[debug_handler]` from
  the `axum-macros` crate (requires the `macros` feature; also `axum::debug_handler`).
  The crate overview confirms it "generates better error messages when applied to
  handler functions." It is a development-only diagnostic aid.

---

## 6. The `axum-core` Crate Boundary

`axum-core` is a separate crate holding the foundational traits : `FromRequest`,
`FromRequestParts`, `IntoResponse`, `IntoResponseParts`, and the `Error` type. The
`axum` crate depends on `axum-core` and adds the router, the concrete extractors, and
`serve`.

The crate overview gives a verified, actionable rule: **library authors should "depend
on the `axum-core` crate, instead of `axum` if possible"** when they only implement core
traits (for example a reusable custom extractor). Depending on `axum-core` rather than
the full `axum` crate insulates the library from breaking changes in the router or
extractor surface. An application binary depends on `axum` directly.

---

## 7. Verified Code Snippets a Skill Author Can Lift

```rust
// Snippet 1 : graceful shutdown (0.7 and 0.8 identical) -- Serve::with_graceful_shutdown
async fn shutdown_signal() {
    tokio::signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "ok" }));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}
```

```rust
// Snippet 2 : a Router is itself a tower::Service -- composition made concrete
use axum::{routing::get, Router};
let app: Router = Router::new().route("/health", get(|| async { "alive" }));
// `app` can be passed to axum::serve, nested, merged, or tested via
// tower::ServiceExt::oneshot -- because it implements tower::Service.
```

```rust
// Snippet 3 : a handler is just an async fn returning impl IntoResponse
async fn hello() -> &'static str { "hello" }   // valid Handler: async + IntoResponse return
// Registered as data, no macro: Router::new().route("/", get(hello))
```

```rust
// Snippet 4 : #[debug_handler] -- the diagnostic for the cryptic Handler error
use axum::debug_handler;          // requires the `macros` feature

#[debug_handler]
async fn create_user(/* extractors */) -> impl axum::response::IntoResponse {
    axum::http::StatusCode::CREATED
}
```

---

## 8. When to Choose Axum

Decision content for the skill's "when to choose Axum" section:

- **Choose Axum when** : the project already uses or wants the tokio async ecosystem;
  you want first-class access to the tower / tower-http middleware ecosystem; you value
  a macro-free, data-as-routing API that composes and unit-tests cleanly; you want a
  framework maintained by the Tokio team with tight hyper integration.
- **Be aware** : the no-macros design produces cryptic `Handler` trait-bound errors
  (mitigated by `#[debug_handler]`); there is no batteries-included auth, ORM, or
  templating (Axum gives building blocks, not a full stack); a multi-threaded runtime
  forces `Send` futures, so `!Send` types (`Rc`, `RefCell`, `std::sync::MutexGuard`)
  cannot be held across `.await`.
- **Consider an alternative** when you specifically want attribute-macro routing
  (Rocket, Actix Web) or a non-tokio runtime.

---

## 9. Key Facts and Gotchas (skill-specific)

- Axum requires **hyper ^1.1.0** and **http ^1.0.0** (verified on the crate overview,
  0.8.9). The jump to hyper 1.0 / http 1.0 / http-body 1.0 happened in **Axum 0.7**;
  Axum 0.6 and earlier used hyper 0.14. A skill claiming "hyper 1.0" is correct for
  both 0.7 and 0.8.
- `axum::Server` was **removed in 0.8** (it was already deprecated in 0.7). The only
  correct entry point for 0.7 and 0.8 is `axum::serve`.
- The `serve` future never completes on its own : it runs until the process is killed,
  or until the `with_graceful_shutdown` signal future resolves.
- A serveable router has type `Router<()>` : all required state must be supplied via
  `.with_state(...)` before passing it to `axum::serve`.
- The `Send` requirement on handler futures is a tokio consequence, not an Axum choice :
  the multi-threaded scheduler can migrate tasks across worker threads.
- `serve` needs the `tokio` feature plus one of `http1` / `http2`; these are on by
  default but matter when `default-features = false` is used.
- `#[tokio::main]` comes from tokio, not Axum : it is the runtime entry point, the only
  macro a minimal Axum app uses.

---

## 10. Sources Verified

All URLs fetched and verified via WebFetch on **2026-05-20**.

- https://docs.rs/axum/latest/axum/ - crate overview (0.8.9) : tokio/hyper/tower
  composition, the five high-level features, "macro-free API" wording, hyper ^1.1.0 and
  http ^1.0.0 requirement, the `axum-core` library-author recommendation,
  `#[debug_handler]` description.
- https://docs.rs/axum/latest/axum/fn.serve.html - verbatim `serve<L, M, S>` signature,
  trait bounds, `Serve<L, M, S>` return type, feature-gating, "never completes" note.
- https://docs.rs/axum/latest/axum/serve/struct.Serve.html - `Serve` methods
  `local_addr()` and `with_graceful_shutdown(signal)` with verbatim signature.
- /home/freek/GitHub/Axum-Claude-Skill-Package/docs/research/vooronderzoek-axum.md -
  sections 1 and 2 (architecture, runtime model, request lifecycle).
- /home/freek/GitHub/Axum-Claude-Skill-Package/docs/research/fragments/research-a-core-routing-extractors.md -
  section 1 (architecture and runtime model, minimal `main` example).

Verification caveat (inherited from fragment A) : exact CHANGELOG release-date strings
could not be reliably transcribed by WebFetch. Release dates used here (Axum 0.7.0 =
2023-11-27, 0.8.0 = 2024-12-02) are cross-checked knowledge; all API-shape claims
above ARE verified against the fetched official docs.rs content.
