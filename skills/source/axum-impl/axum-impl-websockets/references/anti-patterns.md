# Axum WebSockets : Anti-Patterns

Each entry is a real mistake, the exact symptom it produces, and WHY it fails. Verified
against docs.rs and the tokio-rs/axum repository on 2026-05-20.

## 1: Forgetting the ws cargo feature

```rust
use axum::extract::ws::WebSocketUpgrade; // unresolved import
```

Symptom: `error[E0432]: unresolved import axum::extract::ws` or
`could not find ws in extract`.

WHY it fails: the entire `axum::extract::ws` module is gated behind the `ws` feature.
Without `axum = { features = ["ws"] }` the module is not compiled at all. This is a
missing-module error, not a missing-method error, so it is easy to misread as a typo.

Fix: add `features = ["ws"]` to the `axum` dependency.

## 2: Sequential echo loop for a server that must push

```rust
// WRONG when the server pushes on its own schedule
async fn handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.recv().await {
        socket.send(msg).await.ok();
    }
}
```

Symptom: no compiler error. At runtime the server only ever replies to client input
and never delivers timer ticks, broadcasts, or events from other clients.

WHY it fails: the task is parked inside `recv().await` until the client sends
something. While parked it cannot run `send`. A pure echo is fine; anything that pushes
unsolicited messages is not.

Fix: use `socket.split()` + two `tokio::spawn` tasks + `tokio::select!` (SKILL.md
Pattern 4).

## 3: Constructing Message::Text with a String on Axum 0.8

```rust
// axum 0.7 code compiled against axum 0.8
let m = Message::Text("hello".to_string());
```

Symptom: `error[E0308]: mismatched types: expected Utf8Bytes, found String`.
For `Binary`: `expected Bytes, found Vec<u8>`.

WHY it fails: Axum 0.8 changed the variant payloads from owned `String` / `Vec<u8>` to
`Utf8Bytes` / `Bytes`. The 0.7 literal no longer matches the variant type.

Fix: `Message::text("hello")`, or `Message::Text("hello".into())` since `String` and
`&str` both `impl Into<Utf8Bytes>`.

## 4: Pattern matching Message::Text expecting an owned String on 0.8

```rust
// axum 0.8
if let Message::Text(s) = msg {
    let owned: String = s; // mismatched types: expected String, found Utf8Bytes
}
```

Symptom: `error[E0308]: mismatched types: expected String, found Utf8Bytes`.

WHY it fails: on 0.8 the bound value is `Utf8Bytes`, not `String`. `Utf8Bytes` derefs
to `&str`, so `println!`, comparison, and slicing keep working, but assigning it to a
`String` binding does not.

Fix: keep it as `Utf8Bytes` where a `&str` is enough, or call `s.to_string()` when an
owned `String` is genuinely required.

## 5: Calling recv and send on one socket from two tasks

```rust
// WRONG
let socket2 = socket;
tokio::spawn(async move { socket.recv().await; });  // moved twice
tokio::spawn(async move { socket2.send(m).await; });
```

Symptom: `error[E0382]: use of moved value` or `cannot borrow socket as mutable more
than once`.

WHY it fails: `recv` and `send` both take `&mut self`. A single `WebSocket` cannot be
borrowed mutably by two tasks. The type system blocks the unsound code.

Fix: `let (sender, receiver) = socket.split();` then move each half into its own task.

## 6: Blocking the socket task

```rust
// WRONG
async fn handle_socket(mut socket: WebSocket) {
    std::thread::sleep(std::time::Duration::from_secs(5));
    let result = expensive_cpu_loop();
}
```

Symptom: no compiler error. The connection freezes, stops answering pings, and the peer
eventually times out and closes. Other connections on the same worker also stall.

WHY it fails: `std::thread::sleep` and synchronous CPU loops block the Tokio worker
thread. A blocked worker runs no other task, so every connection scheduled on it
misses its deadlines.

Fix: `tokio::time::sleep(...).await` for delays, `tokio::task::spawn_blocking(...)` for
CPU-bound work, async I/O for I/O.

## 7: Not aborting the survivor task in tokio::select

```rust
// WRONG
tokio::select! {
    _ = (&mut send_task) => {},  // recv_task left running
    _ = (&mut recv_task) => {},  // send_task left running
}
```

Symptom: no compiler error. Half-open connections accumulate; the still-running task
holds its socket half open after the other half is gone.

WHY it fails: `tokio::select!` returns as soon as one branch completes but does not
cancel the others. The orphaned task keeps its `SplitSink` or `SplitStream` alive.

Fix: abort the survivor in each branch: `_ = (&mut send_task) => recv_task.abort()`.

## 8: Holding a !Send value across an await in the callback

```rust
// WRONG : Rc is !Send
async fn handle_socket(mut socket: WebSocket) {
    let counter = std::rc::Rc::new(0);
    socket.recv().await; // counter is held across the await
    println!("{counter}");
}
```

Symptom: `error: future cannot be sent between threads safely` pointing at the
`on_upgrade` callback.

WHY it fails: `on_upgrade` requires `Fut: Future<Output = ()> + Send + 'static`. A
`!Send` value (`Rc`, `RefCell`, a raw pointer) held across an `.await` makes the whole
future `!Send`.

Fix: use `Send` equivalents (`Arc` instead of `Rc`, `tokio::sync::Mutex` instead of
`RefCell`), or drop the `!Send` value before the first `.await`.

## 9: Reading the request body alongside WebSocketUpgrade

```rust
// WRONG
async fn handler(ws: WebSocketUpgrade, body: String) -> Response {
    ws.on_upgrade(handle_socket)
}
```

Symptom: a compile error, because a body extractor must be the last argument and
implement `FromRequest`, which consumes the request that `WebSocketUpgrade` also needs.

WHY it fails: the upgrade takes over the connection. `WebSocketUpgrade` is a
`FromRequestParts` extractor; a body extractor is `FromRequest` and consumes the
request body, which conflicts with the upgrade.

Fix: only co-extract `FromRequestParts` types (`State`, `Query`, `ConnectInfo`,
`TypedHeader`, `HeaderMap`). Never extract the body next to `WebSocketUpgrade`.

## 10: Ignoring on_failed_upgrade

```rust
// silent : a failed protocol upgrade is discarded
ws.on_upgrade(handle_socket)
```

Symptom: no compiler error. Failed upgrades vanish with no log line and no metric, so
intermittent client failures are invisible in production.

WHY it fails: after `on_upgrade` returns the `101` response, a later failure of the
background upgrade is handled by `DefaultOnFailedUpgrade`, which ignores the error.

Fix: chain `.on_failed_upgrade(|error| tracing::error!("upgrade failed: {error}"))`
before `on_upgrade` whenever failures must be observed.

## 11: Extracting ConnectInfo without the matching make-service

```rust
// WRONG : plain serve, but the handler extracts ConnectInfo
axum::serve(listener, app).await?;
```

Symptom: at runtime the WebSocket handshake fails with a missing-extension rejection
for `ConnectInfo<SocketAddr>`.

WHY it fails: `ConnectInfo` is populated by `into_make_service_with_connect_info`. With
plain `app` the extension is never inserted, so the extractor rejects every request.

Fix: `axum::serve(listener, app.into_make_service_with_connect_info::<SocketAddr>())`.

## 12: Treating recv returning None as an error

```rust
// WRONG : logs a clean disconnect as a failure
match socket.recv().await {
    Some(Ok(msg)) => { /* ... */ }
    _ => tracing::error!("socket error"), // None is not an error
}
```

Symptom: no compiler error. Every normal client disconnect is logged as an error,
flooding logs.

WHY it fails: `recv()` returns `Option<Result<Message, Error>>`. `None` means the
connection closed cleanly. Only `Some(Err(_))` is a real protocol or transport error.

Fix: match the three cases distinctly: `Some(Ok(msg))` is data, `Some(Err(e))` is an
error, `None` is a clean close.
