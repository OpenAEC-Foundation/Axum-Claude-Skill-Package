# axum-impl-tracing: Working Examples

Every example is verified against the `tokio-rs/axum` `examples/tracing-aka-logging`
crate and the official `tracing`, `tracing-subscriber`, and `tower-http` docs
(2026-05-20). The setup is identical for Axum 0.7 and 0.8.

## Example 1: Full main.rs with subscriber and TraceLayer

```rust
use axum::{routing::get, Router};
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // Install the global subscriber exactly once, before the server starts.
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
        .init();

    let app = Router::new()
        .route("/", get(handler))
        .layer(TraceLayer::new_for_http()); // instruments every request

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    tracing::info!("listening on 0.0.0.0:3000");
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> &'static str {
    "ok"
}
```

## Example 2: The fully customized TraceLayer

All six hooks, verbatim from the official example. Each closure declares its
parameter types so the compiler can infer the hook signature.

```rust
use axum::{
    body::Bytes,
    extract::MatchedPath,
    http::{HeaderMap, Request},
    response::Response,
};
use std::time::Duration;
use tower_http::{classify::ServerErrorsFailureClass, trace::TraceLayer};
use tracing::{info_span, Span};

let trace_layer = TraceLayer::new_for_http()
    .make_span_with(|request: &Request<_>| {
        // Group by the route template, not the concrete URL.
        let matched_path = request
            .extensions()
            .get::<MatchedPath>()
            .map(MatchedPath::as_str);

        info_span!(
            "http_request",
            method = ?request.method(),
            matched_path,
            // Declared now, filled later with span.record("some_other_field", v).
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
    );
```

## Example 3: Structured events in a handler

```rust
use axum::extract::Path;
use axum::http::StatusCode;

async fn get_user(
    Path(id): Path<i64>,
) -> Result<String, StatusCode> {
    tracing::info!(user_id = id, "looking up user");

    match load_user(id).await {
        Ok(name) => {
            tracing::info!(user_id = id, "user found");
            Ok(name)
        }
        Err(err) => {
            // % uses Display: clean, single-line error rendering.
            tracing::error!(user_id = id, error = %err, "user lookup failed");
            Err(StatusCode::INTERNAL_SERVER_ERROR)
        }
    }
}
```

NEVER write `tracing::info!("user {id} found")`. The interpolated form loses the
`user_id` field, so the log line can no longer be filtered or aggregated by user.

## Example 4: An instrumented async handler

```rust
use axum::extract::{Path, State};

// skip(pool): the pool would otherwise be recorded through Debug.
// fields(user_id = %id): a clean field name on the span.
// err: an Err return is recorded at error level automatically.
#[tracing::instrument(skip(pool), fields(user_id = %id), err)]
async fn get_user(
    State(pool): State<PgPool>,
    Path(id): Path<i64>,
) -> Result<Json<User>, AppError> {
    let user = sqlx::query_as!(User, "select * from users where id = $1", id)
        .fetch_one(&pool)
        .await?; // an Err here is recorded by the `err` option
    Ok(Json(user))
}
```

`#[instrument]` enters the span only while the future is polled, so it stays
correct across every `.await` inside the function.

## Example 5: Spanning across an await, wrong versus right

```rust
// WRONG: the drop guard is held across .await.
async fn process_wrong() {
    let span = tracing::info_span!("process");
    let _guard = span.enter();
    fetch_data().await;   // while suspended here, the runtime runs other tasks
    transform().await;    // on this thread; they are wrongly attributed to "process"
}
```

```rust
// RIGHT, option A: #[instrument] on the async fn.
#[tracing::instrument]
async fn process_a() {
    fetch_data().await;
    transform().await;
}
```

```rust
// RIGHT, option B: Future::instrument on an async block.
use tracing::Instrument;

async fn process_b() {
    async {
        fetch_data().await;
        transform().await;
    }
    .instrument(tracing::info_span!("process"))
    .await;
}
```

`span.enter()` is acceptable ONLY in synchronous code:

```rust
fn compute_checksum(data: &[u8]) -> u32 {
    let span = tracing::info_span!("checksum");
    let _guard = span.enter(); // no .await in this scope: safe
    data.iter().fold(0u32, |acc, b| acc.wrapping_add(*b as u32))
}
```

## Example 6: OpenTelemetry export composition

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

fn init_telemetry(tracer: impl tracing_opentelemetry::PreSampledTracer + Send + Sync + 'static) {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "info".into()),
        )
        .with(tracing_subscriber::fmt::layer())                   // console
        .with(tracing_opentelemetry::layer().with_tracer(tracer)) // OTLP export
        .init();
}
```

The `tracer` is built from an `opentelemetry-otlp` pipeline. Once this registry
is installed, every span from `TraceLayer` and `#[instrument]` is exported with
no further per-handler change. Flush the exporter on graceful shutdown so
buffered spans are not lost.
