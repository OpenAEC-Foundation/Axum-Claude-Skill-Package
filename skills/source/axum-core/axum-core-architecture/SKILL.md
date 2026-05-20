---
name: axum-core-architecture
description: >
  Use when starting an Axum project, choosing Axum over another Rust web framework,
  explaining how Axum fits together, or debugging why a handler will not compile or a
  server process exits immediately.
  Prevents treating Axum as a monolith, adding a needless 0.7-vs-0.8 version split to
  the entry point, and chasing the cryptic "Handler is not satisfied" error without
  the #[debug_handler] macro.
  Covers the tokio plus hyper plus tower composition, the full request lifecycle, the
  no-macros design and its trade-off, axum::serve as the glue, the axum-core crate
  boundary, and the decision context for choosing Axum.
  Keywords: axum architecture, axum::serve, tokio hyper tower composition, request
  lifecycle, macro-free API, no macros, Handler is not satisfied, debug_handler,
  axum-core crate boundary, server exits immediately, main returns too soon,
  what is axum, how do I start an axum app, why will not my handler compile,
  #[tokio::main], how does axum work.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Axum Core Architecture

## Overview

Axum is an async web framework for Rust, built and maintained by the Tokio team.
ALWAYS start from the founding fact: Axum implements almost nothing itself. It is a
thin composition over three lower layers, tokio, hyper, and tower. Every architectural
question about Axum resolves to "which layer owns this".

Core principle: routing is data and handlers are ordinary `async fn`s. There are no
routing macros. The single macro a minimal Axum app uses is `#[tokio::main]`, and that
macro belongs to tokio, not to Axum.

This skill covers the composition model, the request lifecycle, the no-macros design,
`axum::serve`, the `axum-core` crate boundary, and when to choose Axum. For the router
API see `axum-core-router`. For handler eligibility see `axum-syntax-handlers`. For
diagnosing handler errors see `axum-errors-handler-trait`.

## Quick Reference

### The three layers Axum composes

| Layer | Crate | Owns |
|-------|-------|------|
| Runtime | tokio | executor, `TcpListener`, timers, `tokio::sync`, task scheduling |
| HTTP | hyper 1.x | HTTP/1 and HTTP/2, parsing socket bytes into `http::Request<Body>` |
| Service abstraction | tower | the `Service` trait, the universal `async fn(Request) -> Result<Response, Error>`, and the tower / tower-http middleware ecosystem |

Axum itself adds the `Router`, the concrete extractors, the `Handler` trait,
`IntoResponse`, and `axum::serve`. Because everything routable in Axum is a
`tower::Service`, Axum inherits timeouts, tracing, compression, CORS, and rate limiting
for free. NEVER reimplement middleware that tower or tower-http already provides.

Axum 0.7 and later require `hyper ^1.1.0` and `http ^1.0.0`. This is true for both 0.7
and 0.8.

### Minimal app (axum 0.7 and 0.8, identical)

```rust
// axum 0.7 and 0.8: this entry point is byte-for-byte identical.
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "Hello, World!" }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

NEVER add a 0.7-vs-0.8 version split to the entry point. `axum::serve` was introduced
in 0.7 as the replacement for the removed `axum::Server`, and its shape did not change
across the 0.7 to 0.8 boundary.

### The `axum::serve` signature (verified, identical 0.7 and 0.8)

```rust
pub fn serve<L, M, S>(listener: L, make_service: M) -> Serve<L, M, S>
where
    L: Listener,
    M: for<'a> Service<IncomingStream<'a, L>, Error = Infallible, Response = S>,
    S: Service<Request, Response = Response, Error = Infallible> + Clone + Send + 'static,
    S::Future: Send;
```

ALWAYS remember the `Serve` future "will never actually complete or return an error".
`main` runs an infinite server until the process is killed or a graceful-shutdown
signal future resolves. A `main` that returns immediately means the runtime was never
blocked on `serve`. See `references/methods.md` for the full `Serve` API.

### The request lifecycle

1. `tokio::net::TcpListener` accepts an incoming TCP connection.
2. hyper reads bytes off the socket and parses them into `http::Request<Body>`.
3. `axum::serve` drives the `Router` as a `tower::Service` by calling `Service::call`.
4. The `Router` matches the request path and selects a `MethodRouter`.
5. The `MethodRouter` selects the handler for the request's HTTP method, or its 405
   fallback when no method matches.
6. Each handler argument runs its extractor: `FromRequestParts` for parts-only
   extractors, `FromRequest` for the single body extractor, which MUST be last.
7. The handler `async fn` body runs and returns a value.
8. `IntoResponse::into_response` turns that value into `http::Response<Body>`.
9. hyper serializes the response back to socket bytes.

Middleware (`tower::Layer` and `tower::Service`) wraps steps 3 through 8: an outer
layer sees the request before the router and the response after the handler.

## Decision Trees

### When to choose Axum

```
Need a Rust web framework?
|- Want a framework maintained by the Tokio team with tight hyper
|  integration and the full tower / tower-http middleware ecosystem?  -> Axum
|- Want routing as composable data, unit-testable with no macro expansion? -> Axum
|- Specifically want attribute-macro routing such as #[get("/path")]? -> Rocket or
|                                                                       Actix Web
|- Need a non-tokio async runtime?                                    -> not Axum
```

ALWAYS expect that Axum gives building blocks, not a full stack. There is no
built-in auth, ORM, or templating engine. ALWAYS expect handler futures to be `Send`,
because tokio's multi-threaded scheduler can migrate tasks between worker threads, so
`!Send` types such as `Rc`, `RefCell`, and `std::sync::MutexGuard` cannot be held
across `.await`.

### Dependency choice: `axum` vs `axum-core`

```
What are you building?
|- An application binary, a runnable server?              -> depend on `axum`
|- A reusable library whose only Axum surface is impls of
|  FromRequest / FromRequestParts / IntoResponse?         -> depend on `axum-core`
```

The official rule, verbatim: "Libraries authors that want to provide `FromRequest`,
`FromRequestParts`, or `IntoResponse` implementations should depend on the `axum-core`
crate, instead of `axum` if possible." Depending on `axum-core` insulates a library
from breaking changes in the router and extractor surface.

### Diagnosing the cryptic `Handler` error

```
Compiler says: the trait bound `fn(..) {handler}: Handler<_, _>` is not satisfied
|- STEP 1: add #[axum::debug_handler] above the handler (needs the `macros` feature)
|- STEP 2: recompile and read the precise plain-language error it emits
|- STEP 3: fix the root cause it names. The 5 causes are:
   |- an argument is not an extractor, for example a bare `bool`
   |- a body extractor is not the last handler argument
   |- the return type does not implement IntoResponse
   |- the function is not declared `async`
   |- the future is !Send, an Rc / RefCell / std::sync::MutexGuard held across .await
```

NEVER guess the cause from the raw error. The no-macros design concentrates every
handler-shape mistake into this one message, and `#[debug_handler]` is the
deterministic tool to decode it. Full diagnosis lives in `axum-errors-handler-trait`.

## Patterns

### Pattern: Axum is a composition, not a monolith

ALWAYS map a feature to the layer that owns it before reaching for an Axum-specific
answer. Sockets, timers, and `spawn_blocking` belong to tokio. HTTP/1 and HTTP/2 wire
behavior belongs to hyper. Middleware, timeouts, tracing, compression, and CORS belong
to tower and tower-http. Axum owns only routing, extraction, handler dispatch, and
response conversion. This map prevents reimplementing what a lower layer already
provides and explains why an Axum app pulls in tokio, hyper, and tower transitively.

### Pattern: the no-macros design and its one trade-off

Routes are ordinary values: `Router::new().route("/", get(handler))` is plain function
calls producing data. Handlers are ordinary `async fn`s whose eligibility is enforced
purely by the `Handler` trait bounds, not by an attribute macro. The benefit is that
routing composes, returns from functions, merges, nests, and unit-tests cleanly, and
no macro expansion obscures compiler errors.

The trade-off: when a function fails the `Handler` trait bounds, the compiler emits the
cryptic `Handler is not satisfied` error that does not name the failing requirement.
The mitigation is the one macro Axum ships for this, `#[debug_handler]` from the
`axum-macros` crate (also re-exported as `axum::debug_handler`, requires the `macros`
feature). It generates a precise diagnostic and is a development-only aid.

### Pattern: `axum::serve` is the glue

`axum::serve` binds a `tokio::net::TcpListener` to a make-service. A `Router<()>` is
accepted directly; a `MethodRouter` or `Handler` becomes a make-service via
`.into_make_service()`. ALWAYS supply all required state with `.with_state(...)` before
serving, because only a `Router<()>` is serveable. `axum::serve` requires the `tokio`
feature plus one of `http1` or `http2`; these are on by default and matter only when
`default-features = false` is set.

For deployments add `.with_graceful_shutdown(signal)`, where `signal` is a future that
resolves on `ctrl_c()` and on the Unix SIGTERM signal. See `references/examples.md`
and the `axum-impl-deployment` skill.

### Pattern: the `axum-core` crate boundary

`axum-core` is a separate crate holding the foundational traits `FromRequest`,
`FromRequestParts`, `IntoResponse`, `IntoResponseParts`, and the `Error` type. The
`axum` crate depends on `axum-core` and adds the router, the concrete extractors, and
`serve`. An application binary depends on `axum`. A reusable library that only
implements core traits depends on `axum-core`.

### Pattern: a `Router` IS a `tower::Service`

A fully configured `Router<()>` implements `tower::Service`. This is why a router can
be passed to `axum::serve`, nested inside another router, merged, and tested via
`tower::ServiceExt::oneshot` with no extra crate. ALWAYS treat the router as a service
value, not as a special framework object. See `axum-impl-testing` for the oneshot
test pattern.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Adding a 0.7-vs-0.8 `cfg` split around `axum::serve` | The entry point is identical on both versions, write it once |
| Expecting `axum::serve(...).await` to return | It never completes, `main` blocks on it until the process is killed |
| Using `axum::Server` on 0.8 | `axum::Server` was removed in 0.8, use `axum::serve` |
| Reimplementing a timeout or trace layer by hand | Use the tower-http layer, Axum inherits the tower ecosystem |
| Guessing at `Handler is not satisfied` | Add `#[debug_handler]` and read the precise error |
| A custom-extractor library depending on full `axum` | Depend on `axum-core` instead |

Full anti-pattern analysis with root causes is in `references/anti-patterns.md`.

## Reference Links

- `references/methods.md`: complete API signatures for `axum::serve`, the `Serve`
  struct, `into_make_service`, `#[tokio::main]`, `#[debug_handler]`, and the
  `axum-core` exports, with feature gates.
- `references/examples.md`: working, version-annotated code for the minimal app,
  graceful shutdown, a router used as a `tower::Service`, `#[debug_handler]`, an
  `axum-core` library extractor, and the `axum::Server` to `axum::serve` migration.
- `references/anti-patterns.md`: real architecture mistakes with "why this fails"
  explanations and fixes.

Related skills:

- `axum-core-router`: the `Router` API, route matching, nesting, merging, fallbacks.
- `axum-core-state`: `State<S>`, `with_state`, `FromRef` substate composition.
- `axum-syntax-handlers`: the `Handler` trait and what makes a valid handler.
- `axum-errors-handler-trait`: full diagnosis of `Handler is not satisfied`.
- `axum-core-version-migration`: the complete 0.7 to 0.8 breaking-change matrix.
- `axum-impl-deployment`: graceful shutdown and production entry points.
