---
name: axum-impl-deployment
description: >
  Use when deploying an Axum service to production, containerizing it with
  Docker, or when a SIGTERM from Docker, Kubernetes, or systemd kills in-flight
  requests on every restart or rolling deploy.
  Prevents dropped requests on shutdown, 1.5 GB images from single-stage builds,
  a full dependency recompile on every source change, and binaries that fail to
  start in a scratch container.
  Covers release builds, static musl binaries, multi-stage Dockerfiles,
  cargo-chef dependency-layer caching, with_graceful_shutdown plus a
  SIGINT/SIGTERM signal future, closing the DB pool on shutdown, and container
  hygiene (bind 0.0.0.0, env-var config, non-root user).
  Keywords: axum deployment, Docker, with_graceful_shutdown, shutdown_signal,
  SIGTERM, SIGINT, graceful shutdown, cargo-chef, multi-stage build, musl,
  x86_64-unknown-linux-musl, scratch image, distroless, TimeoutLayer, release
  build, requests dropped on deploy, container exits immediately, image too big,
  how do I dockerize axum, why does my binary not run in scratch.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Axum Deployment

## Overview

Deploying an Axum service to production has four concerns: a release build, a
small container image, a graceful shutdown that survives orchestrator restarts,
and container hygiene. All deployment patterns in this skill are identical on
Axum 0.7 and 0.8; `axum::serve` and `Serve::with_graceful_shutdown` have the
same signature in both versions.

The single most important runtime concern is graceful shutdown. Docker,
Kubernetes, and systemd send `SIGTERM` on every stop, restart, and rolling
deploy. A plain `axum::serve(listener, app).await` ignores `SIGTERM`, so the
process is killed immediately and every in-flight request is dropped. ALWAYS
install a `SIGTERM`-aware graceful shutdown.

## Quick Reference

| Task | Command |
|------|---------|
| Release build | `cargo build --release` |
| Static musl binary | `cargo build --release --target x86_64-unknown-linux-musl` |
| Add the musl target | `rustup target add x86_64-unknown-linux-musl` |

| Need | API or artifact |
|------|-----------------|
| Run server with shutdown hook | `axum::serve(listener, app).with_graceful_shutdown(signal).await` |
| Catch SIGINT (Ctrl+C) | `tokio::signal::ctrl_c().await` |
| Catch SIGTERM (Unix) | `tokio::signal::unix::signal(SignalKind::terminate())?.recv().await` |
| Cap drain time per request | `tower_http::timeout::TimeoutLayer` |
| Close the DB pool on exit | `pool.close().await` (`sqlx`) |
| Container bind address | `0.0.0.0:<port>`, NEVER `127.0.0.1` |
| Cache the Docker dependency layer | `cargo-chef` (`cargo chef prepare` / `cargo chef cook`) |

`with_graceful_shutdown` signature (verified, Axum 0.7 and 0.8):

```rust
pub fn with_graceful_shutdown<F>(self, signal: F) -> WithGracefulShutdown<L, M, S, F>
where
    F: Future<Output = ()> + Send + 'static,
```

The `signal` future MUST resolve to `()`, MUST be `Send`, and MUST be
`'static`. When it resolves, the server stops accepting new connections and
drains in-flight requests. The resulting future resolves to `io::Result<()>`.

`tokio::signal` requires the tokio `signal` Cargo feature (the `full` umbrella
includes it).

## Decision Trees

### Runtime base image

```
Choosing the container runtime stage?
├─ Built a static musl binary (x86_64-unknown-linux-musl)?
│  └─ FROM scratch : smallest image, no shell, no libc. Copy ca-certificates
│     explicitly for outbound TLS. Set USER 10001:10001.
├─ Built a default glibc binary, want a shell to debug?
│  └─ FROM debian:bookworm-slim : small, has apt and a shell. Add a non-root
│     USER. Easiest to troubleshoot.
└─ Built a glibc binary, want a minimal attack surface, no debug shell?
   └─ FROM gcr.io/distroless/cc-debian12 : no shell or package manager. Use a
      :nonroot tag.
```

### When to use cargo-chef

```
Building the app in Docker?
├─ Iterating, image rebuilt often, dependency tree is large?
│  └─ Use the 3-stage cargo-chef pattern. A source-only change then recompiles
│     only the app crate, not every dependency.
└─ One-off build, tiny dependency tree?
   └─ A plain multi-stage build (builder + runtime) is enough.
```

NEVER ship a single-stage build to production; it embeds the ~1.5 GB Rust
toolchain in the final image.

## Patterns

### Pattern 1: Release build and profile tuning

ALWAYS build production artifacts with `--release`. The dev profile is
unoptimized and far slower at runtime. The release binary lands in
`target/release/<crate-name>`.

```toml
# Cargo.toml : optional production tuning for a smaller, faster binary
[profile.release]
opt-level = 3
lto = "thin"
codegen-units = 1
strip = true          # remove debug symbols, shrink the binary
```

### Pattern 2: Static musl binary

A glibc-linked binary cannot run in a `scratch` or `distroless` container that
lacks a C library. For a fully static, dependency-free binary, build the musl
target:

```bash
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl
# output: target/x86_64-unknown-linux-musl/release/<crate-name>
```

Use `rustls` instead of native-TLS crates under musl; for an Axum service with
`sqlx` + `rustls` this is rarely an issue.

### Pattern 3: Graceful shutdown signal future

The `signal` future MUST handle both `SIGINT` (interactive Ctrl+C) and
`SIGTERM` (the orchestrator stop signal). This future is verified verbatim from
the official `examples/graceful-shutdown` crate:

```rust
use tokio::signal;

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

The `#[cfg(not(unix))]` branch is `std::future::pending::<()>()`, a future that
never resolves, because `SIGTERM` does not exist on Windows. `tokio::select!`
resolves as soon as either signal arrives.

### Pattern 4: Wiring serve, TimeoutLayer, and resource release

```rust
#[tokio::main]
async fn main() {
    // ... tracing setup, build the DB pool, build the Router ...

    let app = Router::new()
        .route("/slow", get(|| sleep(Duration::from_secs(5))))
        .layer((
            TraceLayer::new_for_http(),
            // Cap the drain: a request exceeding the timeout gets 408
            // instead of blocking graceful shutdown forever.
            TimeoutLayer::with_status_code(
                StatusCode::REQUEST_TIMEOUT,
                Duration::from_secs(10),
            ),
        ));

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();

    // Drain has completed here. Release process-wide resources:
    pool.close().await;   // sqlx: returns and closes all connections
    // Flush buffered tracing / OpenTelemetry spans before exit.
}
```

Two rules from this verified pattern:

- ALWAYS pair graceful shutdown with a `TimeoutLayer`. A request that never
  completes (for example a hung upstream call) otherwise blocks the drain
  forever.
- ALWAYS release process-wide resources AFTER `axum::serve(...).await` returns:
  `pool.close().await` so connections are returned cleanly, and flush the
  tracing exporter so no spans are lost.

### Pattern 5: Multi-stage Dockerfile

Compile in a `rust` builder stage, copy ONLY the binary into a minimal runtime
stage. This shrinks the image from ~1.5 GB to tens of MB.

```dockerfile
# ---- Builder stage ----
FROM rust:1-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

# ---- Runtime stage ----
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

`ca-certificates` is needed when the app makes outbound HTTPS calls. See
`references/examples.md` for the `scratch` + musl variant.

### Pattern 6: cargo-chef dependency-layer caching

`COPY . .` then `cargo build` invalidates the Docker layer cache on EVERY
source change, forcing a full recompile of all dependencies. `cargo-chef`
caches dependencies in a layer keyed only on `Cargo.toml` / `Cargo.lock`.

```dockerfile
FROM lukemathwalker/cargo-chef:latest-rust-1 AS chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
# Cached layer: rebuilds ONLY when recipe.json (dependencies) changes.
RUN cargo chef cook --release --recipe-path recipe.json
COPY . .
RUN cargo build --release --bin myapp
# ... minimal runtime stage as in Pattern 5 ...
```

For a musl static build, add `--target x86_64-unknown-linux-musl` to BOTH
`cargo chef cook` and `cargo build` so the cooked artifacts match the final
target.

### Pattern 7: Container hygiene checklist

- ALWAYS bind `0.0.0.0`, NEVER `127.0.0.1`. Inside a container `127.0.0.1` is
  unreachable from the host's published port.
- ALWAYS read `DATABASE_URL`, secrets, the port, and `RUST_LOG` from
  `std::env::var(...)`. NEVER bake secrets or connection strings into source.
- ALWAYS install the `SIGTERM`-aware graceful shutdown (Patterns 3 and 4).
- ALWAYS run as a non-root user (`USER` directive, or a `:nonroot` distroless
  tag, or `USER 10001:10001` for `scratch`).
- ALWAYS add a `.dockerignore` for `target/`, `.git/`, and local `.env` files.
- `EXPOSE 3000` documents the port; publish it with `docker run -p`.

## ALWAYS / NEVER

- ALWAYS call `.with_graceful_shutdown(signal)` on `axum::serve`. NEVER ship a
  plain `axum::serve(listener, app).await`; `SIGTERM` then drops every in-flight
  request on each deploy.
- ALWAYS handle both `SIGINT` and `SIGTERM` in the signal future. NEVER handle
  only `ctrl_c()`; orchestrators send `SIGTERM`, not `SIGINT`.
- ALWAYS pair graceful shutdown with a `TimeoutLayer`. NEVER drain unbounded; a
  hung request blocks shutdown forever.
- ALWAYS use a multi-stage Docker build. NEVER ship a single-stage image; it
  embeds the ~1.5 GB toolchain.
- ALWAYS build a musl static binary for a `scratch` runtime. NEVER put a glibc
  binary in `scratch`; it fails to start with no C library present.
- ALWAYS bind `0.0.0.0` in a container. NEVER bind `127.0.0.1` there.
- ALWAYS close the DB pool and flush tracing after `serve` returns. NEVER let
  `main()` exit while connections are still checked out.

## Reference Links

- `references/methods.md` : verified signatures for `axum::serve`,
  `Serve::with_graceful_shutdown`, `tokio::signal::ctrl_c`,
  `tokio::signal::unix::signal`, `SignalKind`, `TimeoutLayer`, `Pool::close`.
- `references/examples.md` : full `main.rs` with graceful shutdown and resource
  release, env-var config, the `scratch` + musl Dockerfile, the full cargo-chef
  Dockerfile, and a `.dockerignore`.
- `references/anti-patterns.md` : real deployment mistakes, the failure each
  produces, and the fix.

Cross-references:

- `axum-impl-tracing` : configuring and flushing the tracing subscriber.
- `axum-impl-database` : building the `sqlx` pool that Pattern 4 closes.
- `axum-core-async-performance` : pool tuning and never blocking the runtime.
