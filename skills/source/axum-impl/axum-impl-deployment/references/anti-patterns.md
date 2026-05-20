# Axum Deployment : Anti-Patterns

Each entry is a real deployment mistake, the exact failure it produces, and the
fix. Verified on 2026-05-20 against the official `examples/graceful-shutdown`
crate and the cited docs.rs pages.

## 1. Missing graceful shutdown

WRONG:

```rust
let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, app).await.unwrap();   // no shutdown hook
```

WHY THIS FAILS: A plain `axum::serve(...).await` does not react to `SIGTERM`.
Docker, Kubernetes, and systemd send `SIGTERM` on every stop, restart, and
rolling deploy. The default action for `SIGTERM` is to kill the process
immediately, so every in-flight request is dropped and any half-applied work is
abandoned. Each deploy then produces a burst of client-visible errors.

FIX: Install a `SIGTERM`-aware graceful shutdown.

```rust
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await
    .unwrap();
```

## 2. Handling only Ctrl+C, not SIGTERM

WRONG:

```rust
async fn shutdown_signal() {
    signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
}
```

WHY THIS FAILS: `ctrl_c()` resolves on `SIGINT`, the interactive Ctrl+C. An
orchestrator never sends `SIGINT`; it sends `SIGTERM`. A signal future that
waits only on `ctrl_c()` therefore never resolves during a real deploy, so the
graceful drain never starts and the process is hard-killed anyway.

FIX: Select over BOTH signals. The verified future waits on `ctrl_c()` and on a
Unix `SignalKind::terminate()` stream:

```rust
#[cfg(unix)]
let terminate = async {
    signal::unix::signal(signal::unix::SignalKind::terminate())
        .expect("failed to install signal handler")
        .recv()
        .await;
};
tokio::select! {
    _ = ctrl_c => {},
    _ = terminate => {},
}
```

## 3. Graceful shutdown without a TimeoutLayer

WRONG: `with_graceful_shutdown(shutdown_signal())` on a server with no request
timeout layer.

WHY THIS FAILS: Graceful shutdown waits for every in-flight request to finish
before the future resolves. A request that never completes (a hung upstream
call, a `std::future::pending` handler, a slow streaming client) blocks the
drain indefinitely. The orchestrator then escalates to `SIGKILL` after its
grace period, defeating the point of graceful shutdown.

FIX: Pair the server with a `TimeoutLayer` so a stuck request is cut off:

```rust
TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(10))
```

A request exceeding the timeout receives `408` and the drain proceeds.

## 4. Binding 127.0.0.1 inside a container

WRONG:

```rust
let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
```

WHY THIS FAILS: `127.0.0.1` inside a container is the container's own loopback
interface. It is reachable only from inside that container. The host's
published port (`docker run -p 8080:3000`) routes to the container's external
interface, which `127.0.0.1` does not listen on. Connections to the published
port are refused even though the process is running.

FIX: Bind `0.0.0.0`, which accepts connections on all interfaces:

```rust
let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
```

## 5. Single-stage Docker build

WRONG:

```dockerfile
FROM rust:1-bookworm
WORKDIR /app
COPY . .
RUN cargo build --release
CMD ["./target/release/myapp"]
```

WHY THIS FAILS: The final image is the `rust` toolchain image plus the build
output: roughly 1.5 GB. It ships the compiler, cargo, the entire crate
registry cache, and `target/` to production. That is slow to pull, wastes
registry storage, and enlarges the attack surface.

FIX: Use a multi-stage build. Compile in a `rust` builder stage, then copy ONLY
the compiled binary into a minimal runtime stage (`debian:bookworm-slim`,
`distroless`, or `scratch`). The final image drops to tens of MB.

## 6. COPY . . before the dependency build without cargo-chef

WRONG:

```dockerfile
FROM rust:1-bookworm AS builder
COPY . .
RUN cargo build --release
```

WHY THIS FAILS: `COPY . .` copies the source before the build, so the Docker
layer that runs `cargo build` is invalidated by ANY source change. Every
one-line edit then triggers a full recompile of all dependencies, which for a
real service is hundreds of crates and several minutes per build.

FIX: Use the 3-stage `cargo-chef` pattern. `cargo chef cook` builds
dependencies from a `recipe.json` that changes only when `Cargo.toml` /
`Cargo.lock` change, so the dependency layer stays cached across source edits.

## 7. Not closing the DB pool on shutdown

WRONG: `main()` exits immediately after `axum::serve(...).await` returns, with
the `sqlx` pool still holding open connections.

WHY THIS FAILS: When the process exits, open database connections are torn down
at the TCP level without a clean close. The database server logs each as an
abrupt reset, and during a rolling deploy this happens for every instance on
every release, producing connection-error noise and, on some servers, slower
connection-slot reclamation.

FIX: Close the pool after the drain completes:

```rust
axum::serve(listener, app).with_graceful_shutdown(shutdown_signal()).await.unwrap();
pool.close().await;   // async: waits for connections to return, then closes them
```

## 8. Hardcoded secrets and connection strings

WRONG:

```rust
let pool = PgPool::connect("postgres://user:hunter2@db/prod").await.unwrap();
let jwt_secret = "s3cr3t-signing-key";
```

WHY THIS FAILS: The secret is compiled into the binary, committed to git
history, and embedded in every Docker image layer. Anyone with the binary, the
repo, or the image can extract it. Rotating the secret then requires a rebuild
and redeploy.

FIX: Read all secrets and connection strings from environment variables:

```rust
let pool = PgPool::connect(&std::env::var("DATABASE_URL").unwrap()).await.unwrap();
let jwt_secret = std::env::var("JWT_SECRET").unwrap();
```

Provide them at runtime (orchestrator secrets, `docker run -e`, a `.env` kept
out of the build context via `.dockerignore`).

## 9. A glibc binary in a scratch runtime

WRONG:

```dockerfile
FROM rust:1-bookworm AS builder
RUN cargo build --release                 # default target, glibc-linked

FROM scratch AS runtime
COPY --from=builder /app/target/release/myapp /myapp
CMD ["/myapp"]
```

WHY THIS FAILS: The default `cargo build` target produces a binary dynamically
linked against `glibc`. The `scratch` image is empty: no C library, no dynamic
loader. The binary cannot start; the container exits immediately with an error
such as `no such file or directory` (the loader, not the binary, is missing).

FIX: Build a fully static binary for the musl target, then copy that into
`scratch`:

```dockerfile
RUN rustup target add x86_64-unknown-linux-musl
RUN cargo build --release --target x86_64-unknown-linux-musl
# COPY from target/x86_64-unknown-linux-musl/release/
```

Alternatively keep the glibc binary and use `debian:bookworm-slim` or a
`distroless/cc` image, which do provide a C library.

## 10. Running the container as root

WRONG: A Dockerfile with no `USER` directive. The process runs as `root` (UID
0) inside the container.

WHY THIS FAILS: A compromised process running as `root` has far more leverage:
it can write anywhere in the container filesystem and, combined with a
container-escape vulnerability, attacks the host with root privileges. It also
violates the least-privilege baseline most security policies require.

FIX: Run as a non-root user.

- `debian-slim`: `RUN useradd --uid 10001 --no-create-home appuser` then
  `USER appuser`.
- `distroless`: use a `:nonroot` image tag.
- `scratch`: `USER 10001:10001` (no user database exists, so use the numeric
  UID:GID form).
