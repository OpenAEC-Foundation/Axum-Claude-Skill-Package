# Axum WebSockets : API Reference

All signatures verified against docs.rs (Axum 0.8.9) and the tokio-rs/axum repository on
2026-05-20. The entire `axum::extract::ws` module requires the `ws` cargo feature.

## Module gate

```toml
# Cargo.toml : without the ws feature axum::extract::ws does not exist
axum = { version = "0.8", features = ["ws"] }
```

The `ws` feature gate is identical in Axum 0.7 and 0.8.

## WebSocketUpgrade

`WebSocketUpgrade<F>` is the extractor for establishing WebSocket connections. It
implements `FromRequestParts<S>` (it reads request parts only, never the body), so it
composes with any other parts extractor. The type parameter `F` defaults to
`DefaultOnFailedUpgrade`.

### on_upgrade

```rust
pub fn on_upgrade<C, Fut>(self, callback: C) -> Response
where
    C: FnOnce(WebSocket) -> Fut + Send + 'static,
    Fut: Future<Output = ()> + Send + 'static,
```

Finalizes the upgrade. Returns the `101 Switching Protocols` `Response` immediately. The
`callback` runs once the upgrade completes and owns the connection lifecycle. The
callback future MUST be `Send + 'static`, so a socket task cannot hold a `!Send` value
across an `.await`. Signature identical in 0.7 and 0.8.

### on_failed_upgrade

```rust
pub fn on_failed_upgrade<C>(self, callback: C) -> WebSocketUpgrade<C>
where
    C: OnFailedUpgrade,
```

Registers a callback invoked if the background upgrade task fails after `on_upgrade`
already returned its response. The default `DefaultOnFailedUpgrade` ignores the error.
`OnFailedUpgrade` is implemented for any `FnOnce(Error) + Send + 'static`.

### Builder methods

```rust
pub fn protocols<I>(self, protocols: I) -> Self
where
    I: IntoIterator,
    I::Item: Into<Cow<'static, str>>,

pub fn max_message_size(self, max: usize) -> Self   // default 64 MiB
pub fn max_frame_size(self, max: usize) -> Self     // default 16 MiB
pub fn write_buffer_size(self, size: usize) -> Self
pub fn max_write_buffer_size(self, max: usize) -> Self
```

- `protocols` negotiates a `Sec-WebSocket-Protocol` subprotocol; the selected protocol
  is readable on the `WebSocket` via `protocol()`.
- `max_message_size` / `max_frame_size` cap inbound payload sizes (DoS protection).
- `write_buffer_size` / `max_write_buffer_size` tune outbound buffering and backpressure.

All builder method signatures are identical in 0.7 and 0.8.

## WebSocket

`WebSocket` is the live bidirectional connection delivered to the `on_upgrade` callback.
It implements `Sink<Message>` and `Stream<Item = Result<Message, Error>>`.

```rust
pub async fn recv(&mut self) -> Option<Result<Message, Error>>
pub async fn send(&mut self, msg: Message) -> Result<(), Error>
pub async fn close(self) -> Result<(), Error>
pub fn protocol(&self) -> Option<&HeaderValue>
```

- `recv().await` : `None` means the connection closed cleanly; `Some(Err(_))` is a
  protocol or transport error.
- `send(msg).await` : an `Err` means the peer is gone.
- `split()` is `futures_util::StreamExt::split`; it consumes the `WebSocket` and returns
  `(SplitSink<WebSocket, Message>, SplitStream<WebSocket>)`. The split sink uses
  `SinkExt::send` and the split stream uses `StreamExt::next`, both from `futures-util`.

`recv` / `send` signatures are identical in 0.7 and 0.8; only the `Message` payload
types differ between versions (see below).

## Message enum

```rust
// axum 0.8 : verbatim docs.rs enum
pub enum Message {
    Text(Utf8Bytes),
    Binary(Bytes),
    Ping(Bytes),
    Pong(Bytes),
    Close(Option<CloseFrame>),
}
```

```rust
// axum 0.7 : variant payload types
pub enum Message {
    Text(String),
    Binary(Vec<u8>),
    Ping(Vec<u8>),
    Pong(Vec<u8>),
    Close(Option<CloseFrame>),
}
```

The 0.7 to 0.8 change replaced owned `String` / `Vec<u8>` with the reference-counted
`Utf8Bytes` / `Bytes`. The Axum CHANGELOG states the breaking change as: `Message` now
uses `Bytes` in place of `Vec<u8>`, and a new `Utf8Bytes` type in place of `String`.

### Message methods (Axum 0.8)

```rust
pub fn text<S>(string: S) -> Message where S: Into<Utf8Bytes>
pub fn binary<B>(bin: B) -> Message where B: Into<Bytes>
pub fn into_data(self) -> Bytes
pub fn into_text(self) -> Result<Utf8Bytes, Error>
pub fn to_text(&self) -> Result<&str, Error>
```

- `Message::text` / `Message::binary` are the clearest 0.8 constructors and accept any
  `Into<Utf8Bytes>` / `Into<Bytes>`, which includes `&str`, `String`, `&[u8]`, `Vec<u8>`.
- `into_data` consumes any variant into its `Bytes` payload.
- `into_text` consumes into `Utf8Bytes`; `to_text` borrows as `&str`.

### Utf8Bytes and Bytes

- `Utf8Bytes` (Axum 0.8) is a cheaply cloneable, reference-counted UTF-8 string buffer.
  It derefs to `&str`, so formatting, comparison, and slicing work directly. Build an
  owned `String` with `.to_string()`.
- `Bytes` (from the `bytes` crate) is a cheaply cloneable reference-counted byte buffer.
  It derefs to `&[u8]`. Build an owned `Vec<u8>` with `.to_vec()`.
- Both `String -> Utf8Bytes` and `Vec<u8> -> Bytes` are available via `Into`, so
  `.into()` migrates most 0.7 construction sites.

## CloseFrame

```rust
pub struct CloseFrame {
    pub code: u16,        // axum 0.8 : a u16 close code
    pub reason: Utf8Bytes // axum 0.8 ; was String in 0.7
}
```

`Message::Close(None)` closes without a status code. `Message::Close(Some(frame))`
sends a code and reason. Standard codes: `1000` normal closure, `1001` going away.

## Routing

```rust
use axum::{routing::any, Router};

let app: Router = Router::new().route("/ws", any(handler));
```

The upgrade request is a `GET`, but routing the endpoint with `any` is the documented
convention in the official examples. `get(handler)` also works.

## ConnectInfo wiring

`ConnectInfo<SocketAddr>` as a co-extractor requires:

```rust
axum::serve(listener, app.into_make_service_with_connect_info::<SocketAddr>()).await?;
```

Without `into_make_service_with_connect_info`, extracting `ConnectInfo<SocketAddr>`
fails at runtime with a missing-extension rejection.
