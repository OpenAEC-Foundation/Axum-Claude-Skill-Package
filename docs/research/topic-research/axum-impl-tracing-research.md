# Topic Research : axum-impl-tracing

Focused, verified research for the single skill `axum-impl-tracing`. All API
names, signatures, and code below were confirmed via WebFetch against official
documentation and the canonical `tokio-rs/axum` example crate on **2026-05-20**.
Target versions: Axum **0.7** and **0.8**.

This skill covers structured logging and request instrumentation for an Axum
service: the `tracing-subscriber` registry setup, `EnvFilter`, `fmt::layer()`,
`tower_http::trace::TraceLayer`, the `tracing` event macros and `#[instrument]`,
the span-guard-across-await rule, and OpenTelemetry export.

---

## 1. The Two-Crate Model

`tracing` is the instrumentation facade: handler code and middleware emit
**events** and create **spans** through it, with no knowledge of where the data
goes. `tracing-subscriber` is the **collector**: it receives that data and
decides what to filter, format, and export. The application wires them together
exactly once, in `main()`, before the server starts.

A `Subscriber` in `tracing-subscriber` is built compositionally from a
**`Registry`** plus one or more **`Layer`s**. The `Registry` is "storage for
span data shared by multiple `Layer`s" : it holds span state, and each `Layer`
adds one modular behavior (filtering, console formatting, OTLP export) on top.

---

## 2. Registry Setup (verified from the official example)

The `examples/tracing-aka-logging` crate establishes the global subscriber with
this exact sequence:

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

tracing_subscriber::registry()
    .with(
        tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
            format!(
                "{}=debug,tower_http=debug,axum::rejection=trace",
                env!("CARGO_CRATE_NAME")
            )
            .into()
        }),
    )
    .with(tracing_subscriber::fmt::layer())
    .init();
```

Verified facts:

- `tracing_subscriber::registry()` has signature `pub fn registry() -> Registry`
  and requires the `registry` and `std` feature flags.
- `.with(...)` and `.init()` come from the `SubscriberExt` and
  `SubscriberInitExt` extension traits. Both imports are **mandatory** : without
  them `.with()` and `.init()` do not resolve. They are also re-exported from
  `tracing_subscriber::prelude`.
- `.with()` composes one `Layer` onto the registry; calls chain.
- `.init()` installs the composed subscriber as the **global default**. Call it
  exactly once for the process lifetime.

`fmt::layer()` lives in the `fmt` module (requires the `fmt` and `std` features)
and produces a layer "for formatting and logging `tracing` data" to the console.
It is the human-readable output layer.

ALWAYS install the subscriber before `axum::serve(...)`. NEVER call `.init()`
more than once : a second call returns an error / panics because the global
default is already set.

---

## 3. EnvFilter

`EnvFilter` lives in the `tracing_subscriber::filter` module and "controls which
spans and events are enabled by the wrapped subscriber." Verified constructors:

- `EnvFilter::try_from_default_env()` : reads the `RUST_LOG` environment
  variable, returns `Result`. Use this so a missing or malformed `RUST_LOG`
  falls back instead of crashing.
- `EnvFilter::from_default_env()` : same source, but **panics** on error.
- `EnvFilter::new(directives)` : builds a filter from an explicit directive
  string.

The fallback string in the official example,
`"{crate}=debug,tower_http=debug,axum::rejection=trace"`, is a directive list:
`target=level` pairs separated by commas. `axum::rejection=trace` makes Axum log
extractor-rejection details, which is invaluable when debugging why a request
returned a 4xx. ALWAYS pass `EnvFilter` as a layer via `.with(...)`, never as a
raw `set_max_level` call.

---

## 4. TraceLayer (tower-http)

`tower_http::trace::TraceLayer` is middleware that wraps every request in a
tracing span and fires lifecycle callbacks. Construction:

- `TraceLayer::new_for_http()` : classifies responses by HTTP status code; the
  default for an Axum service.
- `TraceLayer::new_for_grpc()` : classifies by the gRPC protocol; supports
  streaming responses.
- `TraceLayer::new(classifier)` : a custom `MakeClassifier`.

Default behavior: `DefaultMakeSpan` creates a span with the request method and
path; `DefaultOnRequest` logs `"started processing request"` at **DEBUG**;
`DefaultOnResponse` logs at **DEBUG** with latency. A bare
`.layer(TraceLayer::new_for_http())` is enough to instrument every request.

Six chainable customization hooks (set any to `()` to disable it):

| Hook | Fires when | Callback receives |
|------|-----------|-------------------|
| `make_span_with` | request arrives | `&Request<B>` -> returns `Span` |
| `on_request` | before inner service | `&Request<B>, &Span` |
| `on_response` | response completes | `&Response<B>, Duration, &Span` |
| `on_body_chunk` | a body chunk is produced | `&Bytes, Duration, &Span` |
| `on_eos` | a streaming response ends | `Option<&HeaderMap>, Duration, &Span` |
| `on_failure` | service / classified error | failure class, `Duration`, `&Span` |

The canonical customized configuration from the official example, verbatim:

```rust
.layer(
    TraceLayer::new_for_http()
        .make_span_with(|request: &Request<_>| {
            let matched_path = request
                .extensions()
                .get::<MatchedPath>()
                .map(MatchedPath::as_str);

            info_span!(
                "http_request",
                method = ?request.method(),
                matched_path,
                some_other_field = tracing::field::Empty,
            )
        })
        .on_request(|_request: &Request<_>, _span: &Span| {
            tracing::debug!("started processing request")
        })
        .on_response(|_response: &Response, _latency: Duration, _span: &Span| {
            tracing::debug!("finished processing request")
        })
        .on_body_chunk(|_chunk: &Bytes, _latency: Duration, _span: &Span| {
            tracing::debug!("sending body chunk")
        })
        .on_eos(
            |_trailers: Option<&HeaderMap>, _stream_duration: Duration, _span: &Span| {
                tracing::debug!("stream closed")
            },
        )
        .on_failure(
            |_error: ServerErrorsFailureClass, _latency: Duration, _span: &Span| {
                tracing::error!("something went wrong")
            },
        ),
)
```

Two facts worth highlighting. First, `make_span_with` reads `MatchedPath` from
the request extensions, so the span groups by the **route template**
(`/users/{id}`) rather than the concrete URL (`/users/42`); this keeps span
cardinality bounded. ALWAYS span on `MatchedPath`, not the raw URI. Second,
`some_other_field = tracing::field::Empty` declares a field with no value yet :
a later callback can fill it with `span.record("some_other_field", value)`,
because a span field that was never declared at creation cannot be recorded
afterward.

Required imports for the customized version:

```rust
use axum::{body::Bytes, extract::MatchedPath,
           http::{HeaderMap, Request}, response::Response};
use std::time::Duration;
use tower_http::{classify::ServerErrorsFailureClass, trace::TraceLayer};
use tracing::{info_span, Span};
```

---

## 5. Event Macros and Field Sigils

`tracing` provides five event macros : `trace!`, `debug!`, `info!`, `warn!`,
`error!`. Each is `event!` with the `Level` pre-set. Fields use
`field_name = value` syntax, comma-separated. Shorthand applies : a bare local
variable name records itself, mirroring struct-initializer shorthand.

```rust
let user_id = 42;
tracing::info!(user_id, "user fetched");          // user_id = user_id
tracing::error!(error = %err, "query failed");    // Display sigil
tracing::debug!(payload = ?body, "received");     // Debug sigil
```

Field sigils, verified:

- `?value` records the value via its `fmt::Debug` implementation.
- `%value` records the value via its `fmt::Display` implementation.

Use `%` for error types and IDs that have a clean `Display`, `?` for structs
and enums you only have `Debug` for. NEVER interpolate values into the message
string (`format!`) : that destroys the structured key/value separation that
makes logs queryable.

---

## 6. `#[instrument]`

The `#[instrument]` attribute macro automatically creates a span named after the
function and **enters it** for the duration of the call, recording the
function's arguments as fields via `fmt::Debug`. Documented options include
`skip` (omit an argument, e.g. a large body or a secret), `fields` (add or
override fields), `ret` (record the return value), `err` (record an `Err`
return at error level), and `level` (set the span level).

```rust
#[tracing::instrument(skip(pool), fields(user_id = %id), err)]
async fn get_user(pool: &PgPool, id: i64) -> Result<User, AppError> {
    let user = sqlx::query_as!(User, "select * from users where id = $1", id)
        .fetch_one(pool)
        .await?;
    Ok(user)
}
```

`#[instrument]` is the correct way to instrument an `async fn` : it attaches the
span to the future and enters it **only while the future is polled**, so it is
async-safe (see section 7). ALWAYS use `skip` for connection pools, request
bodies, and any field carrying secrets.

---

## 7. The Span-Guard-Across-Await Rule

This is the single most important async correctness rule for this skill. The
official `tracing` documentation states verbatim:

> In asynchronous code that uses async/await syntax, `Span::enter` may produce
> incorrect traces if the returned drop guard is held across an await point.

NEVER write this:

```rust
// WRONG : guard held across .await
let span = tracing::info_span!("work");
let _guard = span.enter();
do_async_thing().await;   // span is wrongly entered for other tasks too
```

WHY it fails : `Span::enter()` returns a drop guard that keeps the span entered
on the **current thread** until the guard drops. When the future yields at an
`.await`, the thread runs other tasks while the guard is still alive, so the
span is recorded as active for unrelated work. Timing and parent/child
structure become wrong.

CORRECT options:

```rust
// Option A : #[instrument] on the async fn
#[tracing::instrument]
async fn work() { do_async_thing().await; }

// Option B : Future::instrument from the tracing crate
use tracing::Instrument;
async { do_async_thing().await }
    .instrument(tracing::info_span!("work"))
    .await;
```

`Span::enter()` is fine only in **synchronous** code with no `.await` inside the
guard's scope.

---

## 8. OpenTelemetry Export

To export spans to a distributed-tracing backend (Jaeger, Tempo, Honeycomb,
Grafana), add three crates: `opentelemetry` (the API and SDK), `opentelemetry-otlp`
(the OTLP gRPC/HTTP exporter), and `tracing-opentelemetry` (the bridge layer).

The bridge is `tracing_opentelemetry::layer()`, which produces a
`tracing-subscriber` `Layer` backed by an OpenTelemetry `Tracer`. It composes
onto the **same registry** built in section 2:

```rust
let tracer = /* OTLP tracer pipeline from opentelemetry-otlp */;

tracing_subscriber::registry()
    .with(tracing_subscriber::EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| "info".into()))
    .with(tracing_subscriber::fmt::layer())                 // console
    .with(tracing_opentelemetry::layer().with_tracer(tracer)) // OTLP export
    .init();
```

Every span produced by `TraceLayer` and by `#[instrument]` is then exported
without any further per-handler change : the export is purely a subscriber-layer
concern. For genuine **distributed** tracing, propagate the inbound
`traceparent` header into the request span so a downstream service continues the
same trace. On shutdown, flush the exporter (the graceful-shutdown path) so
buffered spans are not lost.

---

## 9. Anti-Patterns

- **Holding a `Span::enter()` guard across `.await`** : produces interleaved,
  incorrectly-timed traces. Fix: `#[instrument]` or `Future::instrument`.
- **Calling `.init()` more than once** : the global default subscriber can only
  be set once. Fix: initialize once in `main()`.
- **Forgetting the `SubscriberExt` / `SubscriberInitExt` imports** : `.with()`
  and `.init()` fail to resolve. Fix: import both (or the `prelude`).
- **Spanning on the raw URI instead of `MatchedPath`** : unbounded span
  cardinality, one span name per concrete path. Fix: read `MatchedPath` in
  `make_span_with`.
- **Recording a span field that was never declared at span creation** : the
  `record` call is silently ignored. Fix: declare it as `tracing::field::Empty`
  in the `info_span!`.
- **Interpolating values into the log message via `format!`** : loses
  structured key/value data. Fix: use `key = value` fields with `?` / `%`.

---

## 10. Sources Verified

All URLs fetched and verified on **2026-05-20**:

- https://docs.rs/tracing/latest/tracing/ : event macros (`trace!`/`debug!`/
  `info!`/`warn!`/`error!`), `field = value` syntax and shorthand, `?` (Debug)
  and `%` (Display) sigils, `#[instrument]` options (`skip`, `fields`, `ret`,
  `err`, `level`), the `span!` macro, and the verbatim async warning about
  `Span::enter` and drop guards held across await points.
- https://docs.rs/tracing-subscriber/latest/tracing_subscriber/ :
  `registry() -> Registry`, the `Registry` type, `SubscriberExt` /
  `SubscriberInitExt` extension traits (`.with()`, `.init()`), `EnvFilter`
  (`try_from_default_env`, `from_default_env`, `new`, `RUST_LOG`), the `fmt`
  module and `fmt::layer()`.
- https://docs.rs/tower-http/latest/tower_http/trace/index.html : `TraceLayer`,
  `new_for_http()`, `new_for_grpc()`, `new()`; the six hooks `make_span_with`,
  `on_request`, `on_response`, `on_body_chunk`, `on_eos`, `on_failure`; the
  `DefaultMakeSpan` / `DefaultOnRequest` / `DefaultOnResponse` defaults and
  their DEBUG log levels; `MakeSpan` / `OnRequest` / `OnResponse` callback
  signatures.
- https://github.com/tokio-rs/axum/blob/main/examples/tracing-aka-logging/src/main.rs
  : the exact `tracing_subscriber::registry()` setup, the `EnvFilter` fallback
  directive string, `fmt::layer()`, the fully customized `TraceLayer::new_for_http()`
  with all six closures, the `info_span!("http_request", ...)` reading
  `MatchedPath`, and the required imports.
