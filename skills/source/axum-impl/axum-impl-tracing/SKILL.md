---
name: axum-impl-tracing
description: >
  Use when adding structured logging or request tracing to an Axum service,
  when wiring a tracing-subscriber, when instrumenting handlers, when log
  timings look interleaved or spans nest under the wrong parent, or when
  exporting spans to an OpenTelemetry backend.
  Prevents holding a Span::enter guard across an await point, calling init
  twice, missing the SubscriberExt and SubscriberInitExt imports, spanning on
  the raw URI instead of MatchedPath, recording a span field that was never
  declared, and interpolating values into the log message with format.
  Covers tracing-subscriber registry and EnvFilter and fmt::layer, the
  tower-http TraceLayer::new_for_http middleware and its six hooks, the tracing
  event macros, the instrument attribute, the Debug and Display field sigils,
  the span-guard-across-await rule, and OpenTelemetry export.
  Keywords: tracing, tracing-subscriber, registry, EnvFilter, RUST_LOG,
  fmt layer, TraceLayer, new_for_http, make_span_with, on_response, on_failure,
  info macro, error macro, instrument, Span enter, Instrument, MatchedPath,
  opentelemetry, otlp, traceparent, logs are interleaved, traces look wrong,
  timings are wrong, with does not resolve, no logs appear, panic on second
  init, how do I add logging to axum, what is structured logging.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Axum Tracing and Structured Logging

## Overview

Observability in an Axum service rests on two crates. `tracing` is the
instrumentation facade: handler code and middleware emit events and create spans
through it, with no knowledge of where the data goes. `tracing-subscriber` is the
collector: it receives that data and decides what to filter, format, and export.
The application wires them together exactly once, in `main`, before the server
starts.

A subscriber is built compositionally: a `Registry` for span storage plus one or
more `Layer`s, each adding one behavior (filtering, console formatting, OTLP
export). The `tracing` and `tracing-subscriber` APIs in this skill are identical
for Axum 0.7 and 0.8; the version difference in Axum does not reach this layer.

Three rules dominate correct tracing code:

1. Install the subscriber exactly once in `main`, before `axum::serve`. NEVER
   call `.init()` a second time.
2. NEVER hold the drop guard from `Span::enter()` across an `.await`. Use
   `#[instrument]` or `Future::instrument` for async code.
3. Emit structured `key = value` fields. NEVER interpolate values into the
   message string with `format!`.

## Quick Reference

| Need | API | Notes |
|------|-----|-------|
| Build the subscriber | `tracing_subscriber::registry()` | needs `registry` feature |
| Compose a layer | `.with(layer)` | from `SubscriberExt`, import required |
| Install globally | `.init()` | from `SubscriberInitExt`, call once |
| Filter by `RUST_LOG` | `EnvFilter::try_from_default_env()` | returns `Result`, falls back |
| Console output | `tracing_subscriber::fmt::layer()` | human-readable formatting |
| Instrument every request | `.layer(TraceLayer::new_for_http())` | one line, full coverage |
| Customize the request span | `.make_span_with(closure)` | read `MatchedPath` here |
| Emit an event | `tracing::info!(field = value, "msg")` | also `error!`, `warn!`, `debug!`, `trace!` |
| Record via Debug | `field = ?value` | the `?` sigil |
| Record via Display | `field = %value` | the `%` sigil |
| Instrument an async fn | `#[tracing::instrument]` | async-safe span |
| Instrument a future | `.instrument(span)` | from the `Instrument` trait |
| Export to a backend | `tracing_opentelemetry::layer()` | extra layer on the same registry |

Required crates:

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "fmt", "registry"] }
tower-http = { version = "0.6", features = ["trace"] }
# OpenTelemetry export (optional): opentelemetry, opentelemetry-otlp,
# tracing-opentelemetry. Pick mutually compatible versions from docs.rs.
```

## Decision Trees

### Where should this log line come from?

```
What do you want to observe?
|
+- The lifecycle of every HTTP request (method, path, status, latency)
|  `- Add TraceLayer::new_for_http() as a layer. No per-handler code.
|
+- Something that happens inside one handler or function
|  `- Emit an event: tracing::info!(...) / warn! / error! / debug! / trace!.
|
`- The full duration and arguments of one function
   `- Annotate it with #[tracing::instrument].
```

### Synchronous or asynchronous span entering?

```
Do you need a span active around some work?
|
+- The work is synchronous, no .await inside the span scope
|  `- let _g = span.enter(); is safe here.
|
+- The work is an async fn
|  `- #[tracing::instrument] on the fn. NEVER span.enter() + .await.
|
`- The work is an async block or future
   `- future.instrument(span).await, from the tracing::Instrument trait.
      NEVER hold a span.enter() guard across the .await.
```

### Which EnvFilter constructor?

```
Where does the filter directive come from?
|
+- The RUST_LOG environment variable, with a safe fallback
|  `- EnvFilter::try_from_default_env().unwrap_or_else(|_| "info".into())
|     try_* returns Result, so a missing or malformed RUST_LOG falls back.
|
+- The RUST_LOG environment variable, crash if invalid
|  `- EnvFilter::from_default_env()   (panics on a bad directive)
|
`- A fixed directive string in code
   `- EnvFilter::new("my_crate=debug,tower_http=debug")
```

### Console only, or also a distributed-tracing backend?

```
Where do spans need to go?
|
+- Console only (local development, simple deployments)
|  `- registry().with(EnvFilter).with(fmt::layer()).init()
|
`- Console plus Jaeger / Tempo / Honeycomb / Grafana
   `- Add .with(tracing_opentelemetry::layer().with_tracer(tracer))
      to the SAME registry. Flush the exporter on shutdown.
```

## Patterns

### Pattern 1: Install the subscriber once in main

Build the subscriber from `registry()`, an `EnvFilter`, and `fmt::layer()`, then
`.init()` it before the server starts. The `SubscriberExt` and
`SubscriberInitExt` imports are mandatory: without them `.with()` and `.init()`
do not resolve.

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| {
                    format!(
                        "{}=debug,tower_http=debug,axum::rejection=trace",
                        env!("CARGO_CRATE_NAME")
                    )
                    .into()
                }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init(); // installs the global default: call exactly once

    // build the router, then axum::serve(...) below
}
```

`.init()` installs the composed subscriber as the process-wide global default.
A second `.init()` fails because the global default is already set. The fallback
directive includes `axum::rejection=trace`, which logs why an extractor rejected
a request: invaluable when a route returns an unexpected 4xx.

### Pattern 2: Instrument every request with TraceLayer

`tower_http::trace::TraceLayer` wraps every request in a span and logs its
lifecycle. A bare `new_for_http()` is enough for full coverage.

```rust
use tower_http::trace::TraceLayer;

let app = Router::new()
    .route("/", get(handler))
    .layer(TraceLayer::new_for_http());
```

`new_for_http()` classifies responses by HTTP status; `new_for_grpc()`
classifies by the gRPC protocol. The defaults log `"started processing request"`
and the response with latency at DEBUG level, so `RUST_LOG` must allow
`tower_http=debug` for those lines to appear. See `axum-impl-tower-stack` for
layer ordering.

### Pattern 3: Customize the request span

To control span name and fields, pass a closure to `make_span_with`. ALWAYS read
`MatchedPath` from the request extensions so the span groups by the route
template (`/users/{id}`), not the concrete URL (`/users/42`). Spanning on the
raw URI creates one span name per distinct path and makes span cardinality
unbounded.

```rust
use axum::{extract::MatchedPath, http::Request};
use tracing::info_span;

TraceLayer::new_for_http().make_span_with(|request: &Request<_>| {
    let matched_path = request
        .extensions()
        .get::<MatchedPath>()
        .map(MatchedPath::as_str);

    info_span!(
        "http_request",
        method = ?request.method(),
        matched_path,
        some_other_field = tracing::field::Empty, // declared, value set later
    )
})
```

A field that should be filled by a later callback MUST be declared at span
creation as `tracing::field::Empty`. A `span.record("name", value)` call for a
field that was never declared is silently ignored. The full six-hook
configuration is in `references/examples.md`.

### Pattern 4: Emit structured events in handlers

Inside handler code, use the event macros with `key = value` fields. NEVER build
the message with `format!`: structured fields stay queryable, an interpolated
string does not.

```rust
async fn get_user(/* ... */) {
    let user_id = 42;
    tracing::info!(user_id, "user fetched");        // shorthand: user_id = user_id
    tracing::error!(error = %err, "query failed");  // % uses Display
    tracing::debug!(payload = ?body, "received");   // ? uses Debug
}
```

The `%` sigil records a value through its `Display` impl, the `?` sigil through
its `Debug` impl. Use `%` for error types and IDs with a clean `Display`, `?`
for structs and enums.

### Pattern 5: Instrument async functions with #[instrument]

`#[tracing::instrument]` creates a span named after the function, records its
arguments as fields, and enters the span only while the future is polled, so it
is async-safe.

```rust
#[tracing::instrument(skip(pool), fields(user_id = %id), err)]
async fn get_user(pool: &PgPool, id: i64) -> Result<User, AppError> {
    let user = fetch(pool, id).await?;
    Ok(user)
}
```

ALWAYS `skip` connection pools, request bodies, and any argument carrying a
secret: an unskipped argument is recorded through `Debug`. `err` records an
`Err` return at error level; `ret` records the return value; `fields(...)` adds
or overrides fields; `level` sets the span level.

### Pattern 6: NEVER hold a Span::enter guard across an await

The official `tracing` documentation states that `Span::enter` may produce
incorrect traces if its drop guard is held across an await point.

```rust
// WRONG: the guard stays alive across .await
let span = tracing::info_span!("work");
let _guard = span.enter();
do_async_thing().await; // the span is wrongly active for other tasks too
```

`Span::enter()` returns a drop guard that keeps the span entered on the current
thread until the guard drops. When the future yields at an `.await`, the runtime
runs other tasks on that thread while the guard is still alive, so the span is
recorded as active for unrelated work and timings become wrong. Use
`#[instrument]` (Pattern 5) or `Future::instrument`:

```rust
use tracing::Instrument;

async { do_async_thing().await }
    .instrument(tracing::info_span!("work"))
    .await;
```

`Span::enter()` is correct ONLY in synchronous code with no `.await` in the
guard's scope. See `axum-core-async-performance` for the async runtime model.

### Pattern 7: Export to OpenTelemetry

To export spans to a distributed-tracing backend, add the `opentelemetry`,
`opentelemetry-otlp`, and `tracing-opentelemetry` crates. The bridge layer
`tracing_opentelemetry::layer()` composes onto the same registry from Pattern 1.

```rust
let tracer = /* OTLP tracer pipeline from opentelemetry-otlp */;

tracing_subscriber::registry()
    .with(tracing_subscriber::EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| "info".into()))
    .with(tracing_subscriber::fmt::layer())                   // console
    .with(tracing_opentelemetry::layer().with_tracer(tracer)) // OTLP export
    .init();
```

Every span from `TraceLayer` and `#[instrument]` is then exported with no
per-handler change. For genuine distributed tracing, propagate the inbound
`traceparent` header into the request span. On graceful shutdown, flush the
exporter so buffered spans are not lost.

## Reference Links

- `references/methods.md` complete API signatures: `registry`, `SubscriberExt`,
  `SubscriberInitExt`, `EnvFilter` constructors, `fmt::layer`, `TraceLayer`
  constructors and the six hooks, the event macros, `#[instrument]` options.
- `references/examples.md` working code: full `main.rs` setup, the fully
  customized `TraceLayer` with all six closures, handler events, an
  `#[instrument]` handler, the wrong-versus-right span-across-await comparison,
  the OpenTelemetry registry composition.
- `references/anti-patterns.md` real mistakes and why each one fails: guard
  across `.await`, double `.init()`, missing extension-trait imports, spanning
  on the raw URI, recording an undeclared field, `format!` in the message.

Related skills: `axum-impl-tower-stack` (layer ordering and the `tower` stack),
`axum-core-async-performance` (the async runtime and blocking work).

Sources: `https://docs.rs/tracing/latest/tracing/`,
`https://docs.rs/tracing-subscriber/latest/tracing_subscriber/`,
`https://docs.rs/tower-http/latest/tower_http/trace/index.html`,
`https://github.com/tokio-rs/axum/tree/main/examples/tracing-aka-logging`.
