# Axum Deployment : API Reference

All signatures verified on 2026-05-20 via WebFetch against the official docs.rs
pages and the canonical `tokio-rs/axum` example crate. The deployment APIs are
identical on Axum 0.7 and 0.8.

## 1. axum::serve and Serve

`axum::serve(listener, app)` runs an app on a `tokio::net::TcpListener`. It
returns a `Serve<L, M, S>`:

```rust
pub struct Serve<L, M, S> { /* private fields */ }
```

### Serve::with_graceful_shutdown

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
`WithGracefulShutdown` future itself resolves.

The resulting future resolves to `io::Result<()>`. Per the documentation it
never errors after a graceful shutdown; it returns `Ok(())` once the `signal`
future has completed and the drain has finished.

Both `Serve` and `WithGracefulShutdown` are futures, so both are `.await`ed.
`axum::serve(listener, app).await` (no shutdown hook) is valid but ignores
`SIGTERM`.

## 2. tokio::signal

`tokio::signal` requires the tokio Cargo feature `signal`. The `full` feature
umbrella includes it.

### tokio::signal::ctrl_c

```rust
pub async fn ctrl_c() -> Result<()>
```

Resolves the first time the process receives `SIGINT` (the interactive
Ctrl+C). On Unix and Windows alike. `Result` here is `std::io::Result`.

### tokio::signal::unix::signal

```rust
pub fn signal(kind: SignalKind) -> Result<Signal>
```

Unix only. Returns a `Signal` stream for the given kind. `Result` is
`std::io::Result`.

### Signal

The `Signal` value returned by `signal()` has an async `recv()` method:

```rust
pub async fn recv(&mut self) -> Option<()>
```

`recv().await` resolves the next time that signal is delivered.

### SignalKind

`SignalKind` is a struct with `pub const fn` constructors. Verified full set:

```rust
SignalKind::from_raw(signum: c_int)
SignalKind::alarm()
SignalKind::child()
SignalKind::hangup()
SignalKind::interrupt()
SignalKind::io()
SignalKind::pipe()
SignalKind::quit()
SignalKind::terminate()        // SIGTERM : the deployment-relevant kind
SignalKind::user_defined1()
SignalKind::user_defined2()
SignalKind::window_change()
```

Instance method: `SignalKind::as_raw_value(&self) -> c_int`.

For deployment graceful shutdown the relevant constructor is
`SignalKind::terminate()`, which corresponds to `SIGTERM`, the signal Docker,
Kubernetes, and systemd send on stop.

## 3. tower_http::timeout::TimeoutLayer

`TimeoutLayer` (from `tower-http`, feature `timeout`) caps how long a request
may run. Pairing it with graceful shutdown bounds the drain.

```rust
TimeoutLayer::new(Duration)                                  // 408-by-default timeout
TimeoutLayer::with_status_code(StatusCode, Duration)         // explicit status on timeout
```

The official `examples/graceful-shutdown` crate uses:

```rust
TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT, Duration::from_secs(10))
```

A request exceeding the duration receives the given status (here `408 Request
Timeout`) instead of hanging and blocking the shutdown drain.

## 4. sqlx Pool::close

```rust
pub async fn close(&self)
```

`close()` is async and waits for every checked-out connection to be returned
and closed. Call it AFTER `axum::serve(...).await` returns, so a rolling deploy
does not cause abrupt connection resets on the database server. See the
`axum-impl-database` skill for pool construction and tuning.

## 5. Build targets and profiles

Standard Cargo, not an Axum API, but required for deployment:

```bash
cargo build --release                                       # target/release/<crate>
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl     # static binary
```

```toml
[profile.release]
opt-level = 3
lto = "thin"
codegen-units = 1
strip = true
```

## 6. cargo-chef commands

`cargo-chef` is an ecosystem tool for Docker dependency-layer caching. Pin the
exact image tag against the current `cargo-chef` README.

```bash
cargo chef prepare --recipe-path recipe.json   # write the dependency recipe
cargo chef cook --release --recipe-path recipe.json   # build only dependencies
```

Base image: `lukemathwalker/cargo-chef:latest-rust-1`. For a musl build, pass
`--target x86_64-unknown-linux-musl` to both `cargo chef cook` and the final
`cargo build`.

## Sources

- https://docs.rs/axum/latest/axum/serve/struct.Serve.html
- https://github.com/tokio-rs/axum/blob/main/examples/graceful-shutdown/src/main.rs
- https://docs.rs/tokio/latest/tokio/signal/index.html
- https://docs.rs/tokio/latest/tokio/signal/fn.ctrl_c.html
- https://docs.rs/tokio/latest/tokio/signal/unix/fn.signal.html
- https://docs.rs/tokio/latest/tokio/signal/unix/struct.SignalKind.html
- https://docs.rs/tower-http/latest/tower_http/timeout/index.html
- https://docs.rs/sqlx/latest/sqlx/
