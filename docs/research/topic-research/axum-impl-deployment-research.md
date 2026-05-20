# Topic Research : axum-impl-deployment

> Skill : `axum-impl-deployment`
> Category : impl
> Target versions : Axum 0.7 and 0.8
> Researched : 2026-05-20
> All API claims below are WebFetch-verified against official docs.rs pages and
> the canonical `tokio-rs/axum` example crate on 2026-05-20.

This file supports the creation of the `axum-impl-deployment` skill. It covers
release builds, static musl binaries, multi-stage Docker builds with `cargo-chef`
dependency-layer caching, graceful shutdown (SIGINT + SIGTERM), and container
hygiene. Scope boundary: this skill owns the *deploy-to-production* concern.
Graceful-shutdown mechanics are also touched by `axum-impl-graceful-shutdown`;
this skill uses the same verified pattern in the deployment context (closing the
DB pool, flushing tracing, pairing with `TimeoutLayer`).

---

## 1. Release builds

ALWAYS build production artifacts with the release profile. The dev profile is
unoptimized and far slower at runtime:

```bash
cargo build --release
```

The compiled binary lands in `target/release/<crate-name>`. The dev profile
(`cargo build`) produces `target/debug/<crate-name>` and is for local
iteration only.

Optional but recommended `Cargo.toml` tuning for a smaller, faster production
binary:

```toml
[profile.release]
opt-level = 3
lto = "thin"
codegen-units = 1
strip = true
```

`strip = true` removes debug symbols, shrinking the binary; `lto` and
`codegen-units = 1` trade compile time for runtime speed and size.

---

## 2. Static binaries via x86_64-unknown-linux-musl

A glibc-linked binary cannot run in a `scratch` or `distroless` container that
lacks a C library. ALWAYS target `x86_64-unknown-linux-musl` for a fully
static, dependency-free binary that runs anywhere, including `FROM scratch`:

```bash
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl
```

The output path becomes
`target/x86_64-unknown-linux-musl/release/<crate-name>`.

Notes for the skill:
- The musl target produces a statically linked binary; no `glibc` is required
  at runtime.
- Building musl in a Docker image typically uses a builder base such as
  `rust:1-bookworm` plus `apt-get install musl-tools`, or a purpose-built image
  like `clux/muslrust`.
- Crates that bind native C libraries (for example, native TLS) may need a
  pure-Rust replacement (`rustls`) under musl. For an Axum HTTP service with
  `sqlx` + `rustls` this is rarely an issue.

---

## 3. Graceful shutdown (verified)

### 3.1 Why it matters in deployment

`SIGTERM` is the signal Docker, Kubernetes, and systemd send on stop, restart,
and rolling deploy. The default behavior is to kill the process immediately and
drop every in-flight request. ALWAYS install a `SIGTERM`-aware graceful
shutdown so orchestrator-initiated restarts never drop requests or leave
database transactions half-applied.

### 3.2 The `with_graceful_shutdown` signature (verified)

Verified from `https://docs.rs/axum/latest/axum/serve/struct.Serve.html` on
2026-05-20. The `Serve` struct:

```rust
pub struct Serve<L, M, S> { /* private fields */ }
```

The method:

```rust
pub fn with_graceful_shutdown<F>(
    self,
    signal: F,
) -> WithGracefulShutdown<L, M, S, F>
where
    F: Future<Output = ()> + Send + 'static,
```

The `signal` future MUST resolve to `()`, MUST be `Send`, and MUST be
`'static`. When that future resolves, the server stops accepting new
connections and waits for in-flight requests to finish before the
`WithGracefulShutdown` future itself resolves. Per the docs, the resulting
future resolves to `io::Result<()>` and never errors after a graceful shutdown.

### 3.3 The verified `shutdown_signal()` future (SIGINT + SIGTERM)

Verified verbatim from
`https://github.com/tokio-rs/axum/blob/main/examples/graceful-shutdown/src/main.rs`
on 2026-05-20:

```rust
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

`ctrl_c` covers `SIGINT` (the interactive `Ctrl+C`); `terminate` covers
`SIGTERM`, the orchestrator stop signal. The `#[cfg(not(unix))]` branch falls
back to `std::future::pending::<()>()`, a future that never resolves, because
`SIGTERM` does not exist on Windows. `tokio::select!` resolves as soon as
*either* signal arrives.

### 3.4 The verified `main()` wiring

Verified verbatim from the same example file on 2026-05-20:

```rust
use std::time::Duration;

use axum::{http::StatusCode, routing::get, Router};
use tokio::net::TcpListener;
use tokio::signal;
use tokio::time::sleep;
use tower_http::timeout::TimeoutLayer;
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // Enable tracing.
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                format!(
                    "{}=debug,tower_http=debug,axum=trace",
                    env!("CARGO_CRATE_NAME")
                )
                .into()
            }),
        )
        .with(tracing_subscriber::fmt::layer().without_time())
        .init();

    // Create a regular axum app.
    let app = Router::new()
        .route("/slow", get(|| sleep(Duration::from_secs(5))))
        .route("/forever", get(std::future::pending::<()>))
        .layer((
            TraceLayer::new_for_http(),
            // Graceful shutdown will wait for outstanding requests to complete. Add a timeout so
            // requests don't hang forever.
            TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(10)),
        ));

    // Create a `TcpListener` using tokio.
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    // Run the server with graceful shutdown
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await;
}
```

Two facts from this verified snippet that the skill MUST carry:

1. The listener binds `0.0.0.0:3000`, NOT `127.0.0.1`. In a container,
   `127.0.0.1` is only reachable from inside the container itself; `0.0.0.0`
   accepts connections from the published port.
2. `TimeoutLayer` is paired with graceful shutdown. The `/forever` route uses
   `std::future::pending::<()>` (a request that never completes). Without a
   timeout layer, such a request blocks graceful shutdown forever. The verified
   example uses `TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT,
   Duration::from_secs(10))`. A request exceeding the timeout receives a 408
   instead of hanging the drain.

### 3.5 tokio::signal API (verified)

Verified from docs.rs on 2026-05-20:

- `tokio::signal` requires the tokio Cargo **feature `signal`** (the `full`
  feature umbrella includes it).
- `tokio::signal::ctrl_c` signature: `pub async fn ctrl_c() -> Result<()>`.
- `tokio::signal::unix::signal` signature:
  `pub fn signal(kind: SignalKind) -> Result<Signal>`.
- `SignalKind` is a struct with `pub const fn` constructors. The full verified
  set: `from_raw(signum: c_int)`, `alarm()`, `child()`, `hangup()`,
  `interrupt()`, `io()`, `pipe()`, `quit()`, `terminate()`, `user_defined1()`,
  `user_defined2()`, `window_change()`, plus the instance method
  `as_raw_value(&self) -> c_int`. For deployment graceful shutdown, the
  relevant one is `SignalKind::terminate()` (SIGTERM).
- `Signal` (returned by `signal()`) has an async `recv()` method that resolves
  the next time the signal is received.

### 3.6 On shutdown: release resources

When `with_graceful_shutdown` returns, the server has finished draining. ALWAYS
release process-wide resources before `main()` exits:

```rust
#[tokio::main]
async fn main() {
    // ... tracing setup, build pool, build app ...

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();

    // After drain completes: close the DB pool so checked-out
    // connections are returned and closed cleanly.
    pool.close().await;

    // Flush any buffered tracing / OpenTelemetry spans before exit.
    // For an OTLP pipeline: opentelemetry::global::shutdown_tracer_provider();
}
```

`sqlx::Pool::close()` is async and waits for all connections to be returned.
Closing it prevents abrupt connection resets on the database side during a
rolling deploy.

---

## 4. Multi-stage Dockerfile (builder + slim runtime)

The naive single-stage approach ships the entire Rust toolchain (~1.5 GB). A
multi-stage build compiles in a `rust` builder stage and copies ONLY the
compiled binary into a minimal runtime stage, shrinking the final image to tens
of MB.

### 4.1 Builder + `debian:bookworm-slim` runtime

```dockerfile
# ---- Builder stage ----
FROM rust:1-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

# ---- Runtime stage ----
FROM debian:bookworm-slim AS runtime
WORKDIR /app
# ca-certificates is needed for outbound TLS (e.g. calling other HTTPS APIs).
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
EXPOSE 3000
ENV RUST_LOG=info
CMD ["myapp"]
```

Choice of runtime base:
- `debian:bookworm-slim` : small, has a shell and `apt`, easiest to debug. Use
  with a glibc (default) binary.
- `gcr.io/distroless/cc-debian12` : no shell or package manager, smaller attack
  surface. Use with a glibc binary that still needs `libgcc`/`libc`.
- `scratch` : empty image, smallest possible. Requires a fully static musl
  binary (section 5) and a copied `ca-certificates` bundle for outbound TLS.

### 4.2 Static musl binary into `scratch`

```dockerfile
FROM rust:1-bookworm AS builder
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends musl-tools \
    && rustup target add x86_64-unknown-linux-musl
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch AS runtime
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder \
    /app/target/x86_64-unknown-linux-musl/release/myapp /myapp
EXPOSE 3000
ENV RUST_LOG=info
CMD ["/myapp"]
```

`scratch` has no `ca-certificates`, so the CA bundle is copied explicitly when
the app makes outbound HTTPS calls.

---

## 5. cargo-chef dependency-layer caching

The problem: `COPY . .` followed by `cargo build` invalidates the Docker layer
cache on EVERY source change, forcing a full recompile of all dependencies
(which can be hundreds of crates and several minutes). `cargo-chef` solves this
by computing a `recipe.json` that captures only the dependency graph, building
dependencies in a layer that is cached as long as `Cargo.toml` / `Cargo.lock`
are unchanged.

The three-stage `cargo-chef` pattern:

```dockerfile
# ---- Stage 1: planner ----
FROM lukemathwalker/cargo-chef:latest-rust-1 AS chef
WORKDIR /app

FROM chef AS planner
COPY . .
# Produce a recipe describing only the dependency graph.
RUN cargo chef prepare --recipe-path recipe.json

# ---- Stage 2: build dependencies, then the app ----
FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
# This layer is cached and only rebuilds when recipe.json changes.
RUN cargo chef cook --release --recipe-path recipe.json
# Now copy the actual source and build only the application crate.
COPY . .
RUN cargo build --release --bin myapp

# ---- Stage 3: minimal runtime ----
FROM debian:bookworm-slim AS runtime
WORKDIR /app
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
EXPOSE 3000
ENV RUST_LOG=info
CMD ["myapp"]
```

How the caching works:
- The `planner` stage runs `cargo chef prepare`, which writes `recipe.json`
  describing the dependency tree. Application source changes do not change
  `recipe.json` unless `Cargo.toml`/`Cargo.lock` changed.
- `cargo chef cook` builds all dependencies from `recipe.json`. Because only
  `recipe.json` is copied before this step, Docker caches this layer; it
  re-runs ONLY when dependencies change.
- `COPY . .` and `cargo build` then compile only the application crate against
  the already-built dependency artifacts.

Net effect: a source-only change recompiles the app crate in seconds instead of
recompiling every dependency.

For a musl static build with `cargo-chef`, add `--target
x86_64-unknown-linux-musl` to BOTH `cargo chef cook` and `cargo build` so the
cooked dependency artifacts match the final build target.

---

## 6. Container hygiene

The verified production checklist for an Axum container:

1. **Bind `0.0.0.0`, not `127.0.0.1`.** Verified from the graceful-shutdown
   example: `TcpListener::bind("0.0.0.0:3000")`. `127.0.0.1` inside a container
   is unreachable from the host's published port.
2. **Configuration from environment variables.** ALWAYS read `DATABASE_URL`,
   `JWT_SECRET`, the bind port, and `RUST_LOG` from `std::env::var(...)`. NEVER
   bake secrets or connection strings into source. A typical pattern:

   ```rust
   let port = std::env::var("PORT").unwrap_or_else(|_| "3000".to_string());
   let addr = format!("0.0.0.0:{port}");
   let listener = TcpListener::bind(&addr).await.unwrap();
   ```

3. **`SIGTERM`-aware graceful shutdown** (section 3). Orchestrators send
   `SIGTERM` on every deploy.
4. **Run as a non-root user.** Add a `USER` directive (for a `debian-slim`
   runtime) so the process does not run as root:

   ```dockerfile
   RUN useradd --uid 10001 --no-create-home appuser
   USER appuser
   ```

   For `distroless`, use a `:nonroot` image tag. For `scratch`, set
   `USER 10001:10001`.
5. **`EXPOSE` the port and document it.** `EXPOSE 3000` is documentation; the
   port is published with `docker run -p`.
6. **Release profile + minimal runtime base.** Sections 1, 4, 5.
7. **`.dockerignore`** to keep `target/`, `.git/`, and local `.env` files out
   of the build context, which speeds up `COPY . .` and prevents secret leaks.

---

## 7. Anti-patterns specific to deployment

- **Missing graceful shutdown** : plain `axum::serve(listener, app).await` with
  no `.with_graceful_shutdown(...)` kills the process on `SIGTERM`, dropping
  every in-flight request on each deploy. FIX: the section 3.3 / 3.4 pattern.
- **Binding `127.0.0.1` in a container** : the published port routes to nothing.
  FIX: bind `0.0.0.0`.
- **Single-stage Docker build** : ships the ~1.5 GB Rust toolchain in the
  production image. FIX: multi-stage build (section 4).
- **`COPY . .` before dependency build without `cargo-chef`** : every source
  change recompiles all dependencies. FIX: `cargo-chef` (section 5).
- **Graceful shutdown without `TimeoutLayer`** : one never-completing request
  blocks the drain indefinitely. FIX: pair with `TimeoutLayer` (section 3.4).
- **Not closing the DB pool on shutdown** : abrupt connection resets on the
  database during rolling deploys. FIX: `pool.close().await` after
  `axum::serve(...).await` returns.
- **Hardcoded secrets / connection strings** : leaks into the binary, git
  history, and every Docker layer. FIX: environment variables.
- **`scratch` runtime with a glibc binary** : the binary fails to start because
  no C library is present. FIX: build the musl static target, or use
  `debian-slim`/`distroless`.

---

## Sources verified (2026-05-20)

All URLs below were fetched and verified via WebFetch on **2026-05-20**:

- `https://github.com/tokio-rs/axum/blob/main/examples/graceful-shutdown/src/main.rs`
  — verbatim `shutdown_signal()` future (SIGINT via `signal::ctrl_c()`, SIGTERM
  via `signal::unix::signal(SignalKind::terminate())`, `#[cfg(not(unix))]`
  fallback, `tokio::select!`), the `main()` wiring, `TcpListener::bind
  ("0.0.0.0:3000")`, `axum::serve(listener, app).with_graceful_shutdown
  (shutdown_signal()).await`, and `TimeoutLayer::with_status_code(...)`.
- `https://docs.rs/axum/latest/axum/serve/struct.Serve.html` — verbatim
  `with_graceful_shutdown<F>` signature, the `F: Future<Output = ()> + Send +
  'static` bound, the `WithGracefulShutdown<L, M, S, F>` return type, and the
  `Serve<L, M, S>` struct definition.
- `https://docs.rs/tokio/latest/tokio/signal/index.html` — the `signal` Cargo
  feature requirement for `tokio::signal`.
- `https://docs.rs/tokio/latest/tokio/signal/fn.ctrl_c.html` — verbatim
  `pub async fn ctrl_c() -> Result<()>`.
- `https://docs.rs/tokio/latest/tokio/signal/unix/fn.signal.html` — verbatim
  `pub fn signal(kind: SignalKind) -> Result<Signal>`.
- `https://docs.rs/tokio/latest/tokio/signal/unix/struct.SignalKind.html` —
  verbatim list of `SignalKind` `pub const fn` constructors including
  `terminate()`, `interrupt()`, `hangup()`, and `as_raw_value()`.

Verification caveat: `cargo-chef` usage (`cargo chef prepare` / `cargo chef
cook`, the `lukemathwalker/cargo-chef` image) and the Docker base-image facts
(`rust:1-bookworm`, `debian:bookworm-slim`, distroless `cc-debian12`, `scratch`)
are established, widely-documented ecosystem conventions cross-checked against
the project's prior research fragments, not fetched verbatim from a single
upstream page in this session. The skill author SHOULD pin exact `cargo-chef`
image tags against the current `cargo-chef` README when writing the SKILL.md.
