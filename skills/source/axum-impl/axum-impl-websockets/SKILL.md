---
name: axum-impl-websockets
description: >
  Use when adding WebSocket endpoints to an Axum service, when migrating WebSocket
  code from Axum 0.7 to 0.8, or when a socket can receive client input but cannot
  push messages on its own schedule.
  Prevents the sequential-only echo trap, blocking the Tokio worker inside a socket
  task, the silent dropped upgrade failure, and the 0.7 to 0.8 Message payload-type
  break.
  Covers the ws cargo feature, WebSocketUpgrade, on_upgrade, on_failed_upgrade, the
  WebSocket connection, the Message enum, split plus tokio::spawn plus tokio::select,
  and combining the upgrade with State and ConnectInfo.
  Keywords: axum websocket, WebSocketUpgrade, on_upgrade, on_failed_upgrade,
  Message::Text, Utf8Bytes, Bytes, split sink stream, tokio::select, ws feature,
  server cannot push messages, websocket only echoes, connection freezes,
  Message::Text expects String, mismatched types String Utf8Bytes, how do I add
  websockets to axum, what is on_upgrade.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# Axum WebSockets

## Overview

A WebSocket endpoint in Axum is an ordinary handler that takes the `WebSocketUpgrade`
extractor, returns `ws.on_upgrade(callback)`, and runs the live connection inside the
callback. The HTTP `101 Switching Protocols` response is returned immediately; the
callback owns the connection for its entire lifetime.

Two facts dominate correct WebSocket code:

1. A single `WebSocket` cannot be borrowed mutably by two tasks at once, so it cannot
   `recv` and `send` concurrently as-is. Any server that pushes messages on its own
   schedule MUST split the socket.
2. Axum 0.8 changed the `Message` payload types from owned `String` / `Vec<u8>` to the
   reference-counted `Utf8Bytes` / `Bytes`. Code carried over from 0.7 without change
   fails to compile.

The entire `axum::extract::ws` module is gated behind the `ws` cargo feature.

## Quick Reference

| Need | API | Notes |
|------|-----|-------|
| Enable WebSockets | `axum = { features = ["ws"] }` | Without it `axum::extract::ws` does not exist |
| Accept an upgrade | `WebSocketUpgrade` extractor | `FromRequestParts`, composes with other parts extractors |
| Start the connection | `ws.on_upgrade(callback)` | Returns the `101` response immediately |
| Log upgrade failures | `.on_failed_upgrade(cb)` | Default behavior is silent |
| Receive a message | `socket.recv().await` | `Option<Result<Message, Error>>`; `None` means closed |
| Send a message | `socket.send(msg).await` | `Result<(), Error>`; `Err` means the peer is gone |
| Concurrent send + receive | `socket.split()` + 2x `tokio::spawn` + `tokio::select!` | One socket cannot recv and send at once |
| Build a text message (0.8) | `Message::text("hi")` or `Message::Text("hi".into())` | 0.8 wraps `Utf8Bytes`, not `String` |
| Cap inbound size | `.max_message_size(n)` / `.max_frame_size(n)` | DoS protection on the upgrade |
| Route the endpoint | `Router::new().route("/ws", any(handler))` | `any` is the documented convention |

Required crates for the concurrent pattern: `axum` with `ws`, `tokio` with `macros` and
`rt`, and `futures-util` for `.split()`, `StreamExt`, and `SinkExt`.

## Decision Trees

### Sequential echo vs concurrent split

```
Does the server send messages on its own schedule
(timers, broadcasts, events from other clients)?
├── NO  : server only replies to client input
│         -> sequential loop on &mut socket           (Pattern 2)
└── YES : server pushes independently of recv
          -> split + 2 tokio::spawn + tokio::select!  (Pattern 4)
```

ALWAYS use the concurrent pattern when the server is a chat hub, a live feed, or any
source of unsolicited messages. The sequential loop can only answer; it cannot push,
because it is blocked inside `recv().await` while waiting for the client.

### WebSocket vs Server-Sent Events

```
Does the client need to send messages to the server over this channel?
├── YES : bidirectional      -> WebSocket (this skill)
└── NO  : server-to-client   -> Server-Sent Events (see axum-impl-sse)
```

### Picking a Message constructor (Axum 0.8)

```
Building a Message::Text or Message::Binary?
├── from a string/byte literal  -> Message::text("..") / Message::binary(b"..")
├── from format!(...)           -> Message::Text(format!("..").into())
└── from an owned String/Vec    -> Message::Text(s.into()) / Message::Binary(v.into())
```

## Patterns

### Pattern 1: Enable the ws feature

```toml
# Cargo.toml
[dependencies]
axum = { version = "0.8", features = ["ws"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
futures-util = "0.3"
```

ALWAYS enable the `ws` feature. Without it `axum::extract::ws` does not exist and the
compiler reports a missing-module error, not a missing-method error.

### Pattern 2: Sequential echo handler

Verified against the docs.rs `axum::extract::ws` module page (Axum 0.8).

```rust
// axum 0.8
use axum::{
    extract::ws::{WebSocket, WebSocketUpgrade},
    response::Response,
    routing::any,
    Router,
};

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

let app: Router = Router::new().route("/ws", any(handler));
```

`recv().await` returns `Option<Result<Message, Error>>`. `None` means the connection
closed; `Some(Err(_))` is a protocol or transport error. This handler is correct ONLY
for a pure echo. It can never push a message on its own, because it is parked inside
`recv().await` until the client speaks.

### Pattern 3: Migrate Message construction from 0.7 to 0.8

This is the single highest-impact 0.7 to 0.8 break in WebSocket code. The
`Text` / `Binary` / `Ping` / `Pong` variants stopped wrapping owned `String` / `Vec<u8>`
and now wrap the reference-counted `Utf8Bytes` / `Bytes`.

```rust
// axum 0.7 : variant payload types
Message::Text(String)
Message::Binary(Vec<u8>)
Message::Ping(Vec<u8>)
Message::Pong(Vec<u8>)

// axum 0.8 : variant payload types
Message::Text(Utf8Bytes)
Message::Binary(Bytes)
Message::Ping(Bytes)
Message::Pong(Bytes)
```

Construction migration:

```rust
// axum 0.7
let m = Message::Text("hello".to_string());
let b = Message::Binary(vec![1, 2, 3]);

// axum 0.8 : String and Vec<u8> both impl Into for the new types
let m = Message::Text("hello".into());
let b = Message::Binary(vec![1, 2, 3].into());

// axum 0.8 : constructor helpers, the clearest form
let m = Message::text("hello");
let b = Message::binary(vec![1, 2, 3]);
```

Pattern-match migration: on 0.7 `Message::Text(s)` binds `s: String`; on 0.8 it binds
`s: Utf8Bytes`. `Utf8Bytes` derefs to `&str` and `Bytes` derefs to `&[u8]`, so
formatting and comparison keep working. Code that needed an owned `String` MUST add an
explicit `.to_string()`.

NEVER write `Message::Text(String)` for an 0.8 example. ALWAYS annotate WebSocket code
with its target version.

### Pattern 4: Concurrent send and receive (split + spawn + select)

Verified against `tokio-rs/axum/examples/websockets/src/main.rs` (Axum 0.8).

```rust
// axum 0.8
use axum::extract::ws::{Message, WebSocket};
use futures_util::{sink::SinkExt, stream::StreamExt};

async fn handle_socket(socket: WebSocket) {
    let (mut sender, mut receiver) = socket.split();

    let mut send_task = tokio::spawn(async move {
        for i in 0..10 {
            if sender
                .send(Message::Text(format!("server msg {i}").into()))
                .await
                .is_err()
            {
                return;
            }
            tokio::time::sleep(std::time::Duration::from_millis(300)).await;
        }
    });

    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(msg)) = receiver.next().await {
            if matches!(msg, Message::Close(_)) {
                break;
            }
        }
    });

    tokio::select! {
        _ = (&mut send_task) => recv_task.abort(),
        _ = (&mut recv_task) => send_task.abort(),
    }
}
```

`socket.split()` yields a `SplitSink` (`sender`) and a `SplitStream` (`receiver`); it is
`futures_util::StreamExt::split`. Each half is `move`d into its own `tokio::spawn`, so
send and receive run truly concurrently. `tokio::select!` borrows each `JoinHandle`
mutably; when one task ends, the other is `.abort()`ed so no task leaks. ALWAYS abort the
survivor, otherwise a half-open connection lingers.

### Pattern 5: Match every Message variant

Verified against `tokio-rs/axum/examples/websockets/src/main.rs` (Axum 0.8).

```rust
// axum 0.8
use axum::extract::ws::Message;
use std::ops::ControlFlow;

fn process_message(msg: Message) -> ControlFlow<(), ()> {
    match msg {
        Message::Text(t) => println!("text: {t}"),       // t: Utf8Bytes, derefs to &str
        Message::Binary(d) => println!("{} bytes", d.len()), // d: Bytes
        Message::Ping(_) => {}  // answered automatically by the underlying layer
        Message::Pong(_) => {}  // answered automatically by the underlying layer
        Message::Close(_) => return ControlFlow::Break(()),
    }
    ControlFlow::Continue(())
}
```

`Ping` is answered with a `Pong` automatically by the underlying tungstenite layer; a
handler still sees both and only needs custom logic for application-level heartbeats.
Receiving `Close` is the signal to stop the loop.

### Pattern 6: Combine WebSocketUpgrade with State and ConnectInfo

`WebSocketUpgrade` is a `FromRequestParts` extractor, so it composes with any other
parts extractor in any order. Read every co-extractor BEFORE the upgrade, then `move`
the values into the `on_upgrade` closure.

```rust
// axum 0.8
use axum::extract::{ws::WebSocketUpgrade, ConnectInfo, State};
use axum::response::Response;
use std::net::SocketAddr;

async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
) -> Response {
    ws.on_upgrade(move |socket| handle_socket(socket, state, addr))
}
```

`ConnectInfo<SocketAddr>` requires the server to be started with
`axum::serve(listener, app.into_make_service_with_connect_info::<SocketAddr>())`.

NEVER read the request body alongside `WebSocketUpgrade`; the upgrade consumes the
connection, so a body extractor conflicts with it. Every co-extractor MUST be
`FromRequestParts`.

### Pattern 7: Log failed upgrades

After `on_upgrade` returns its response, a later failure of the protocol upgrade is
silently ignored by default. Override it to observe failures.

```rust
// axum 0.8 (signature identical in 0.7)
async fn handler(ws: WebSocketUpgrade) -> Response {
    ws.on_failed_upgrade(|error| {
        tracing::error!("websocket upgrade failed: {error}");
    })
    .on_upgrade(handle_socket)
}
```

ALWAYS attach `on_failed_upgrade` in production when failed upgrades must be logged or
counted; the default `DefaultOnFailedUpgrade` discards the error.

### Pattern 8: NEVER block the socket task

The `on_upgrade` callback and any task spawned from it run on Tokio worker threads.

```rust
// WRONG : stalls a Tokio worker, the connection misses ping deadlines
async fn handle_socket(mut socket: WebSocket) {
    std::thread::sleep(std::time::Duration::from_secs(5));
    let result = expensive_cpu_loop();
}

// RIGHT : async sleep, CPU work offloaded
async fn handle_socket(mut socket: WebSocket) {
    tokio::time::sleep(std::time::Duration::from_secs(5)).await;
    let result = tokio::task::spawn_blocking(expensive_cpu_loop).await.unwrap();
}
```

ALWAYS use `tokio::time::sleep` instead of `std::thread::sleep`, async I/O instead of
synchronous I/O, and `tokio::task::spawn_blocking` for CPU-bound work inside any socket
task. Blocking one worker starves every other task on it; the peer eventually times out.

## Reference Links

- `references/methods.md` : complete API signatures for `WebSocketUpgrade`, `WebSocket`,
  the `Message` enum, `CloseFrame`, and the builder methods, with 0.7 / 0.8 differences.
- `references/examples.md` : full working examples (chat hub, stateful socket, graceful
  close, subprotocol negotiation), each version-annotated.
- `references/anti-patterns.md` : real mistakes with the exact compiler or runtime
  symptom and WHY each fails.

## Related Skills

- `axum-core-async-performance` : Tokio runtime, `spawn`, `spawn_blocking`, never block.
- `axum-core-version-migration` : the full 0.7 to 0.8 breaking-change catalog.
- `axum-impl-sse` : Server-Sent Events for server-to-client only streaming.
