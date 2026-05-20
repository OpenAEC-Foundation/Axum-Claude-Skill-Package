# Axum Deployment : Working Examples

Verified on 2026-05-20 against the official `tokio-rs/axum`
`examples/graceful-shutdown` crate and the cited docs.rs pages. The graceful
shutdown code is identical on Axum 0.7 and 0.8.

## Example 1: Complete main.rs with graceful shutdown

A production `main.rs`: env-var config, a `SIGINT`/`SIGTERM` signal future,
`TimeoutLayer`-bounded drain, and resource release after the drain.

```rust
use std::time::Duration;

use axum::{http::StatusCode, routing::get, Router};
use tokio::net::TcpListener;
use tokio::signal;
use tokio::time::sleep;
use tower_http::timeout::TimeoutLayer;
use tower_http::trace::TraceLayer;

#[tokio::main]
async fn main() {
    // Configuration from environment variables. NEVER hardcode these.
    let port = std::env::var("PORT").unwrap_or_else(|_| "3000".to_string());
    let addr = format!("0.0.0.0:{port}");

    // ... build the DB pool here (see axum-impl-database) ...

    let app = Router::new()
        .route("/slow", get(|| sleep(Duration::from_secs(5))))
        .route("/forever", get(std::future::pending::<()>))
        .layer((
            TraceLayer::new_for_http(),
            // Graceful shutdown waits for outstanding requests. The timeout
            // stops a never-completing request from blocking the drain.
            TimeoutLayer::with_status_code(
                StatusCode::REQUEST_TIMEOUT,
                Duration::from_secs(10),
            ),
        ));

    // Bind 0.0.0.0 so the container's published port is reachable.
    let listener = TcpListener::bind(&addr).await.unwrap();

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();

    // The drain has completed here. Release process-wide resources.
    // pool.close().await;                  // sqlx: return + close connections
    // Flush buffered tracing / OpenTelemetry spans before exit.
}

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

The `shutdown_signal()` function is verified verbatim from the official
`examples/graceful-shutdown/src/main.rs`. `ctrl_c` covers `SIGINT`; `terminate`
covers `SIGTERM`. The `#[cfg(not(unix))]` branch is `std::future::pending`
because `SIGTERM` does not exist on Windows.

## Example 2: Resource release on shutdown (with a DB pool)

```rust
#[tokio::main]
async fn main() {
    // tracing setup ...
    let pool = sqlx::PgPool::connect(&std::env::var("DATABASE_URL").unwrap())
        .await
        .unwrap();

    let app = build_app(pool.clone());   // your Router constructor
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();

    // After drain: close the pool so connections are returned and closed
    // cleanly instead of being reset abruptly during a rolling deploy.
    pool.close().await;

    // Flush a tracing / OpenTelemetry exporter, for example an OTLP pipeline,
    // so no spans are lost on exit.
}
```

## Example 3: Cargo.toml release profile and dependencies

```toml
[profile.release]
opt-level = 3
lto = "thin"
codegen-units = 1
strip = true

[dependencies]
axum         = "0.8"    # axum 0.8 ; for axum 0.7 use "0.7"
tokio        = { version = "1", features = ["full"] }   # full includes `signal`
tower-http   = { version = "0.6", features = ["timeout", "trace"] }
```

`tokio`'s `signal` feature is required for `tokio::signal`; the `full` umbrella
includes it. `tower-http`'s `timeout` feature is required for `TimeoutLayer`.

## Example 4: Multi-stage Dockerfile, debian-slim runtime

```dockerfile
# ---- Builder stage ----
FROM rust:1-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

# ---- Runtime stage ----
FROM debian:bookworm-slim AS runtime
WORKDIR /app
# ca-certificates is needed for outbound TLS (calling other HTTPS APIs).
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
# Run as a non-root user.
RUN useradd --uid 10001 --no-create-home appuser
USER appuser
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
EXPOSE 3000
ENV RUST_LOG=info
CMD ["myapp"]
```

## Example 5: Static musl binary into a scratch image

```dockerfile
FROM rust:1-bookworm AS builder
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends musl-tools \
    && rustup target add x86_64-unknown-linux-musl
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch AS runtime
# scratch has no ca-certificates; copy the CA bundle for outbound HTTPS.
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder \
    /app/target/x86_64-unknown-linux-musl/release/myapp /myapp
USER 10001:10001
EXPOSE 3000
ENV RUST_LOG=info
CMD ["/myapp"]
```

`scratch` is the smallest possible image. It has no shell and no libc, so it
REQUIRES a fully static musl binary.

## Example 6: cargo-chef Dockerfile (3-stage)

`cargo-chef` caches the dependency build so a source-only change recompiles
only the application crate.

```dockerfile
# ---- Stage 1: cargo-chef base + planner ----
FROM lukemathwalker/cargo-chef:latest-rust-1 AS chef
WORKDIR /app

FROM chef AS planner
COPY . .
# recipe.json describes ONLY the dependency graph.
RUN cargo chef prepare --recipe-path recipe.json

# ---- Stage 2: build dependencies (cached), then the app ----
FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
# This layer is cached; it re-runs ONLY when recipe.json changes.
RUN cargo chef cook --release --recipe-path recipe.json
# Now copy real source and compile only the application crate.
COPY . .
RUN cargo build --release --bin myapp

# ---- Stage 3: minimal runtime ----
FROM debian:bookworm-slim AS runtime
WORKDIR /app
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
RUN useradd --uid 10001 --no-create-home appuser
USER appuser
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
EXPOSE 3000
ENV RUST_LOG=info
CMD ["myapp"]
```

For a musl static build with `cargo-chef`, add `--target
x86_64-unknown-linux-musl` to BOTH `cargo chef cook` and `cargo build` so the
cooked dependency artifacts match the final build target, then copy from
`target/x86_64-unknown-linux-musl/release/`.

## Example 7: .dockerignore

Keeps the build context small and prevents secret leaks into Docker layers.

```
target/
.git/
.env
.env.*
*.md
```

`target/` is the largest exclusion (gigabytes of build output). `.env` files
hold local secrets and MUST NOT enter the build context.
