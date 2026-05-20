# Axum Core Architecture: Anti-Patterns

Real architecture-level mistakes, each with the symptom, why it fails at the root, and
the deterministic fix. These are observed in the Axum issue tracker, discussions, and
the framework's own documentation warnings.

## AP-1: treating Axum as a monolith and reimplementing tower

Symptom: hand-writing a request-timeout wrapper, a logging middleware, or a CORS check
from scratch inside the application, or asking "how does Axum do X" for X that is a
generic HTTP-middleware concern.

Why it fails: Axum implements almost nothing itself. It is a composition over tokio,
hyper, and tower. Middleware in Axum is the `tower::Service` and `tower::Layer`
abstraction, and the `tower-http` crate already ships `TraceLayer`, `CorsLayer`,
`CompressionLayer`, `TimeoutLayer`, and `RequestBodyLimitLayer`. A hand-written
equivalent is more code, is untested against the spec, and does not compose with the
rest of the stack.

Fix: ALWAYS map a feature to the layer that owns it first. Sockets, timers, and
blocking offload belong to tokio. HTTP wire behavior belongs to hyper. Cross-cutting
middleware belongs to tower and tower-http. Reach for a tower-http layer before
writing one. See the `axum-impl-tower-stack` skill.

## AP-2: adding a needless 0.7-vs-0.8 version split to the entry point

Symptom: wrapping `axum::serve` in `#[cfg]` branches, or writing two `main` functions,
"to support both Axum versions".

Why it fails: `axum::serve` was introduced in 0.7 as the replacement for the removed
`axum::Server`, and its signature did not change across the 0.7 to 0.8 boundary. The
minimal `main`, the `Serve` struct, `with_graceful_shutdown`, and `into_make_service`
are all identical on 0.7 and 0.8. A version split here is dead complexity that implies
a difference that does not exist.

Fix: NEVER `cfg`-split the entry point. Write `axum::serve(listener, app).await` once.
The genuine 0.7 to 0.8 breaks are path syntax, the WebSocket `Message` type, the
`#[async_trait]` removal, and a set of removed APIs. None of them touch the server
entry point. See the `axum-core-version-migration` skill for the real matrix.

## AP-3: expecting axum::serve to return, so main exits immediately

Symptom: the process starts and exits within milliseconds, no requests are served, or
code is structured as if `axum::serve(...).await` returns a value to act on.

Why it fails: the `Serve` future, per the official docs, "will never actually complete
or return an error". A socket error is handled internally by sleeping briefly and
continuing. If `main` reaches its end, it is because the runtime was never actually
blocked on `serve`: the `serve` call was not `.await`ed, or it was spawned and
forgotten, or an earlier line returned an error through `?`.

Fix: ALWAYS `.await` the `serve` future as the last thing `main` does, and ALWAYS
handle the bind `Result` explicitly. A clean exit happens only through a
`with_graceful_shutdown` signal, never on its own.

## AP-4: using axum::Server on Axum 0.8

Symptom: `error[E0433]: failed to resolve: could not find Server in axum`, or migration
code that still calls `axum::Server::bind(...).serve(...)`.

Why it fails: `axum::Server` was the pre-0.7 entry point. It was deprecated in 0.7 and
REMOVED entirely in 0.8. It does not exist in the 0.8 crate.

Fix: bind a `tokio::net::TcpListener` first, then call `axum::serve(listener, app)`. A
`Router<()>` no longer needs `.into_make_service()` for plain serving. See Example 7
in `references/examples.md`.

## AP-5: guessing at the "Handler is not satisfied" error

Symptom: a long, opaque compiler error of the form `the trait bound 'fn(..) {handler}:
Handler<_, _>' is not satisfied`, followed by trial-and-error edits to the handler.

Why it fails: the no-macros design enforces handler eligibility purely through the
`Handler` trait bounds. When any requirement fails, the compiler reports only that the
bound is unsatisfied; it does not name which of the five requirements broke. Guessing
wastes time and often "fixes" the wrong thing.

Fix: ALWAYS add `#[axum::debug_handler]` above the handler (requires the `macros`
feature), recompile, and read the precise plain-language error it generates. The five
root causes are: an argument that is not an extractor, a body extractor that is not
last, a return type that is not `IntoResponse`, a function that is not `async`, and a
`!Send` future. Remove `#[debug_handler]` once the handler compiles. Full diagnosis is
in the `axum-errors-handler-trait` skill.

## AP-6: a custom-extractor library depending on the full axum crate

Symptom: a reusable library crate whose only Axum surface is a `FromRequestParts` or
`IntoResponse` impl, yet it declares `axum = "0.8"` as a dependency.

Why it fails: depending on the full `axum` crate ties the library to the router and
the concrete extractors, surfaces that change more often than the foundational traits.
Every `axum` minor release that touches the router can force a library bump even
though the library never uses the router.

Fix: ALWAYS depend on `axum-core` for a library that only implements `FromRequest`,
`FromRequestParts`, or `IntoResponse`. The official guidance is explicit: such library
authors "should depend on the `axum-core` crate, instead of `axum` if possible". An
application binary still depends on `axum` directly.

## AP-7: holding a !Send value across .await in a handler

Symptom: the cryptic `Handler is not satisfied` error appears after adding an `.await`
to a handler that locks a `std::sync::Mutex`, or uses an `Rc` or a `RefCell`.

Why it fails: tokio's multi-threaded scheduler can migrate a task between worker
threads, so `axum::serve` requires every handler future to be `Send`. An `Rc`, a
`RefCell` borrow, or a `std::sync::MutexGuard` kept alive across an `.await` point
makes the whole future `!Send`, and the `Handler` blanket impl stops applying. This is
an architectural consequence of running on a multi-threaded runtime, not an Axum quirk.

Fix: ALWAYS use `tokio::sync::Mutex` for state guarded across `.await`, or scope a
`std::sync::Mutex` lock so the guard drops before any `.await`. Replace `Rc` with
`Arc`. The async-runtime detail is covered by the `axum-core-async-performance` skill.

## AP-8: shipping #[debug_handler] to production

Symptom: `#[debug_handler]` attributes left on handlers after the compile error is
resolved.

Why it fails: `#[debug_handler]` exists only to improve compiler diagnostics. It adds
no runtime value. Leaving it in place is dead annotation and signals to a reader that
the handler is still being debugged.

Fix: ALWAYS remove `#[debug_handler]` once the handler compiles cleanly. Treat it the
same way as a temporary `dbg!` call.

## Sources verified

All fetched via WebFetch on 2026-05-20:

- https://docs.rs/axum/latest/axum/ : the composition model, the `axum-core`
  library-author rule, the `#[debug_handler]` description.
- https://docs.rs/axum/latest/axum/fn.serve.html : the "never completes" guarantee.
- https://docs.rs/axum/latest/axum/handler/index.html : the `Handler` trait criteria
  and the invalid-handler causes.
- https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md : the removal of
  `axum::Server` in 0.8.
