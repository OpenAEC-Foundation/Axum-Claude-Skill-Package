# Axum WebSockets : Working Examples

Every example is version-annotated. Code is verified against the docs.rs
`axum::extract::ws` module page and `tokio-rs/axum/examples/websockets/src/main.rs`
(2026-05-20). Examples target Axum 0.8 unless marked otherwise.

## Example 1: Minimal echo server

```rust
// axum 0.8
use axum::{
    extract::ws::{WebSocket, WebSocketUpgrade},
    response::Response,
    routing::any,
    Router,
};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/ws", any(handler));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(msg) = socket.recv().await {
        let Ok(msg) = msg else {
            return; // client disconnected or protocol error
        };
        if socket.send(msg).await.is_err() {
            return; // client disconnected
        }
    }
}
```

`Cargo.toml`:

```toml
[dependencies]
axum = { version = "0.8", features = ["ws"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread", "net"] }
```

## Example 2: The same echo handler, 0.7 vs 0.8

Only the `Message` construction differs. The `recv` / `send` loop is identical.

```rust
// axum 0.7
use axum::extract::ws::{Message, WebSocket};

async fn reply(mut socket: WebSocket) {
    socket
        .send(Message::Text("ready".to_string())) // Text wraps String
        .await
        .ok();
}
```

```rust
// axum 0.8
use axum::extract::ws::{Message, WebSocket};

async fn reply(mut socket: WebSocket) {
    socket
        .send(Message::text("ready")) // helper builds Utf8Bytes
        .await
        .ok();
}
```

## Example 3: Concurrent socket (split + spawn + select)

A server that pushes a counter every 300 ms while still reading client input. The
sequential loop cannot do this.

```rust
// axum 0.8
use axum::extract::ws::{Message, WebSocket};
use futures_util::{sink::SinkExt, stream::StreamExt};
use std::time::Duration;

async fn handle_socket(socket: WebSocket) {
    let (mut sender, mut receiver) = socket.split();

    let mut send_task = tokio::spawn(async move {
        let mut i = 0u32;
        loop {
            if sender
                .send(Message::Text(format!("tick {i}").into()))
                .await
                .is_err()
            {
                break; // peer gone
            }
            i += 1;
            tokio::time::sleep(Duration::from_millis(300)).await;
        }
    });

    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(msg)) = receiver.next().await {
            match msg {
                Message::Text(t) => println!("client said: {t}"),
                Message::Close(_) => break,
                _ => {}
            }
        }
    });

    tokio::select! {
        _ = (&mut send_task) => recv_task.abort(),
        _ = (&mut recv_task) => send_task.abort(),
    }
}
```

`Cargo.toml` adds `futures-util = "0.3"`.

## Example 4: Stateful socket with State and ConnectInfo

```rust
// axum 0.8
use axum::{
    extract::{ws::WebSocketUpgrade, ConnectInfo, State},
    response::Response,
    routing::any,
    Router,
};
use std::{net::SocketAddr, sync::Arc};
use std::sync::atomic::{AtomicUsize, Ordering};

#[derive(Clone)]
struct AppState {
    connections: Arc<AtomicUsize>,
}

#[tokio::main]
async fn main() {
    let state = AppState { connections: Arc::new(AtomicUsize::new(0)) };
    let app = Router::new()
        .route("/ws", any(ws_handler))
        .with_state(state);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .await
    .unwrap();
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
) -> Response {
    // co-extractors are read BEFORE the upgrade, then moved into the closure
    ws.on_upgrade(move |socket| handle_socket(socket, state, addr))
}

async fn handle_socket(
    mut socket: axum::extract::ws::WebSocket,
    state: AppState,
    addr: SocketAddr,
) {
    let n = state.connections.fetch_add(1, Ordering::SeqCst) + 1;
    println!("{addr} connected, {n} live connections");

    while let Some(Ok(_msg)) = socket.recv().await {
        // handle messages
    }

    state.connections.fetch_sub(1, Ordering::SeqCst);
    println!("{addr} disconnected");
}
```

`into_make_service_with_connect_info::<SocketAddr>()` is mandatory when any handler
extracts `ConnectInfo<SocketAddr>`.

## Example 5: Broadcast chat hub

Each socket receives every other socket's messages through a `tokio::sync::broadcast`
channel held in shared state. This needs the concurrent pattern: the receive half feeds
the channel, the send half drains it.

```rust
// axum 0.8
use axum::extract::{ws::{Message, WebSocket, WebSocketUpgrade}, State};
use axum::{response::Response, routing::any, Router};
use futures_util::{sink::SinkExt, stream::StreamExt};
use tokio::sync::broadcast;

#[derive(Clone)]
struct ChatState {
    tx: broadcast::Sender<String>,
}

#[tokio::main]
async fn main() {
    let (tx, _rx) = broadcast::channel::<String>(100);
    let app = Router::new()
        .route("/ws", any(ws_handler))
        .with_state(ChatState { tx });
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn ws_handler(ws: WebSocketUpgrade, State(state): State<ChatState>) -> Response {
    ws.on_upgrade(move |socket| handle_socket(socket, state))
}

async fn handle_socket(socket: WebSocket, state: ChatState) {
    let (mut sender, mut receiver) = socket.split();
    let mut rx = state.tx.subscribe();
    let tx = state.tx.clone();

    // send half : drain the broadcast channel to this client
    let mut send_task = tokio::spawn(async move {
        while let Ok(line) = rx.recv().await {
            if sender.send(Message::Text(line.into())).await.is_err() {
                break;
            }
        }
    });

    // receive half : feed this client's messages into the broadcast channel
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(t))) = receiver.next().await {
            let _ = tx.send(t.to_string()); // Utf8Bytes -> owned String
        }
    });

    tokio::select! {
        _ = (&mut send_task) => recv_task.abort(),
        _ = (&mut recv_task) => send_task.abort(),
    }
}
```

Note `t.to_string()`: `t` is a `Utf8Bytes`, and the broadcast channel here carries
owned `String`. On Axum 0.7 `t` would already be a `String` and `.to_string()` would be
a redundant clone.

## Example 6: Graceful close with a CloseFrame

```rust
// axum 0.8
use axum::extract::ws::{CloseFrame, Message, WebSocket};

async fn close_with_reason(mut socket: WebSocket) {
    socket
        .send(Message::Close(Some(CloseFrame {
            code: 1000, // normal closure
            reason: "server shutting down".into(), // Utf8Bytes
        })))
        .await
        .ok();
    // after sending Close, stop using the socket
}
```

On Axum 0.7 `CloseFrame.reason` is a `String`, so use `"...".into()` or
`.to_string()`; the `.into()` form shown above compiles on both versions.

## Example 7: Subprotocol negotiation and size limits

```rust
// axum 0.8 (builder signatures identical in 0.7)
use axum::extract::ws::WebSocketUpgrade;
use axum::response::Response;

async fn handler(ws: WebSocketUpgrade) -> Response {
    ws.protocols(["graphql-ws", "graphql-transport-ws"])
        .max_message_size(1024 * 1024) // 1 MiB inbound cap
        .max_frame_size(64 * 1024)     // 64 KiB frame cap
        .on_failed_upgrade(|error| {
            tracing::error!("upgrade failed: {error}");
        })
        .on_upgrade(handle_socket)
}

async fn handle_socket(socket: axum::extract::ws::WebSocket) {
    if let Some(proto) = socket.protocol() {
        println!("negotiated subprotocol: {proto:?}");
    }
}
```
