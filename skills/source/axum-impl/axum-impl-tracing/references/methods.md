# axum-impl-tracing: API Signatures

All signatures verified 2026-05-20 against `https://docs.rs/tracing/latest/tracing/`,
`https://docs.rs/tracing-subscriber/latest/tracing_subscriber/`,
`https://docs.rs/tower-http/latest/tower_http/trace/index.html`, and the
`tokio-rs/axum` `examples/tracing-aka-logging` crate.

## tracing-subscriber: registry and composition

```rust
// Storage for span data shared by multiple Layers.
pub fn registry() -> Registry            // needs the `registry` feature
```

`.with()` and `.init()` are NOT inherent methods. They come from two extension
traits that MUST be imported:

```rust
use tracing_subscriber::layer::SubscriberExt;     // provides .with(layer)
use tracing_subscriber::util::SubscriberInitExt;  // provides .init()
// or: use tracing_subscriber::prelude::*;
```

- `.with(layer)` composes one `Layer` onto the subscriber; calls chain.
- `.init()` installs the composed subscriber as the global default. It can
  succeed only once per process; a second call fails.

## tracing-subscriber: EnvFilter

`EnvFilter` controls which spans and events are enabled. It lives in the
`tracing_subscriber::filter` module and needs the `env-filter` feature.

```rust
EnvFilter::try_from_default_env() -> Result<EnvFilter, FromEnvError>
EnvFilter::from_default_env()     -> EnvFilter   // panics on a bad directive
EnvFilter::new(directives: impl AsRef<str>) -> EnvFilter
```

`try_from_default_env` and `from_default_env` both read the `RUST_LOG`
environment variable. A directive string is a comma-separated list of
`target=level` pairs, for example
`my_crate=debug,tower_http=debug,axum::rejection=trace`.

## tracing-subscriber: fmt::layer

```rust
tracing_subscriber::fmt::layer()    // needs the `fmt` feature
```

Produces a `Layer` that formats and writes `tracing` data to the console in
human-readable form.

## tower-http: TraceLayer constructors

```rust
TraceLayer::new_for_http()           // classify responses by HTTP status code
TraceLayer::new_for_grpc()           // classify by the gRPC protocol
TraceLayer::new(classifier)          // a custom MakeClassifier
```

Default behavior: `DefaultMakeSpan` creates a span with the request method and
path; `DefaultOnRequest` logs `"started processing request"` at DEBUG;
`DefaultOnResponse` logs the response with latency at DEBUG.

## tower-http: TraceLayer hooks

Each hook returns a new `TraceLayer`. Set any hook to `()` to disable it.

| Hook | Fires when | Closure receives |
|------|-----------|------------------|
| `make_span_with` | a request arrives | `&Request<B>`, returns a `Span` |
| `on_request` | before the inner service | `&Request<B>, &Span` |
| `on_response` | the response completes | `&Response<B>, Duration, &Span` |
| `on_body_chunk` | a body chunk is produced | `&Bytes, Duration, &Span` |
| `on_eos` | a streaming response ends | `Option<&HeaderMap>, Duration, &Span` |
| `on_failure` | a classified failure occurs | failure class, `Duration`, `&Span` |

Imports for the customized form:

```rust
use axum::{body::Bytes, extract::MatchedPath,
           http::{HeaderMap, Request}, response::Response};
use std::time::Duration;
use tower_http::{classify::ServerErrorsFailureClass, trace::TraceLayer};
use tracing::{info_span, Span};
```

## tracing: event macros

Five macros, each `event!` with the `Level` pre-set:

```rust
tracing::trace!(/* fields, */ "message");
tracing::debug!(/* fields, */ "message");
tracing::info!(/* fields, */ "message");
tracing::warn!(/* fields, */ "message");
tracing::error!(/* fields, */ "message");
```

Fields use `name = value`, comma-separated, before the message. A bare local
variable name records itself (`user_id` means `user_id = user_id`).

Field sigils:

- `?value` records the value through its `fmt::Debug` implementation.
- `%value` records the value through its `fmt::Display` implementation.

## tracing: span macros

```rust
tracing::trace_span!("name", /* fields */) -> Span
tracing::debug_span!("name", /* fields */) -> Span
tracing::info_span!("name", /* fields */)  -> Span
tracing::warn_span!("name", /* fields */)  -> Span
tracing::error_span!("name", /* fields */) -> Span
tracing::span!(Level::INFO, "name", /* fields */) -> Span
```

`tracing::field::Empty` declares a field with no value yet; a later
`span.record("name", value)` fills it. A `record` for a field never declared at
creation is ignored.

```rust
// Synchronous use only: no .await inside the guard scope.
let span = tracing::info_span!("work");
let _guard = span.enter();   // -> drop guard, keeps span entered on this thread
```

## tracing: the instrument attribute

```rust
#[tracing::instrument]
#[tracing::instrument(skip(arg), fields(k = %v), ret, err, level = "debug")]
```

Creates a span named after the function, records arguments as fields through
`Debug`, and enters the span only while the future is polled (async-safe).
Options: `skip` omits an argument; `fields` adds or overrides fields; `ret`
records the return value; `err` records an `Err` return at error level; `level`
sets the span level.

## tracing: the Instrument trait

```rust
use tracing::Instrument;

future.instrument(span) -> Instrumented<F>   // attach a span to a future
```

The instrumented future enters the span only while it is polled. This is the
correct way to span across `.await` for an async block.

## tracing-opentelemetry: the bridge layer

```rust
tracing_opentelemetry::layer()                  -> OpenTelemetryLayer
tracing_opentelemetry::layer().with_tracer(t)    // attach an OpenTelemetry Tracer
```

The result is a `tracing-subscriber` `Layer`, composed onto the same `registry()`
with `.with(...)` alongside the `fmt` layer.
