# Topic Research : axum-impl-websockets

> Skill : `axum-impl-websockets`
> Phase : 4 (topic research)
> Researched : 2026-05-20
> Target versions : Axum 0.7 and 0.8 (docs.rs current at research time : 0.8.9)
> All API signatures and code below WebFetch-verified against docs.rs and the
> tokio-rs/axum repository on 2026-05-20. No signatures are paraphrased.

This file scopes the `axum-impl-websockets` skill. It covers the WebSocket upgrade
flow, the `WebSocket` connection type, the `Message` enum, the high-impact 0.7 -> 0.8
`Message` type change, and the canonical concurrent (split + spawn + select) pattern.

---

## 1. The `ws` Cargo feature

The entire `axum::extract::ws` module is feature-gated. The docs.rs module page states
verbatim: "Available on crate feature `ws` only."

```toml
# Cargo.toml
[dependencies]
axum = { version = "0.8", features = ["ws"] }
```

ALWAYS enable the `ws` feature; without it `axum::extract::ws` does not exist and the
compile error is a missing-module error, not a missing-method error. The concurrent
pattern additionally needs `futures-util` (for `.split()`, `StreamExt`, `SinkExt`) and
`tokio` with the `macros` + `rt` features (for `tokio::spawn` and `tokio::select!`).

---

## 2. `WebSocketUpgrade` extractor and `on_upgrade`

`WebSocketUpgrade` is a `FromRequestParts` extractor (it reads request parts only, never
the body), so it can be combined freely with other parts extractors. The docs describe
it as the "Extractor for establishing WebSocket connections."

`on_upgrade` is the core method. Verified signature from docs.rs:

```rust
pub fn on_upgrade<C, Fut>(self, callback: C) -> Response
where
    C: FnOnce(WebSocket) -> Fut + Send + 'static,
    Fut: Future<Output = ()> + Send + 'static,
```

`on_upgrade` returns the `101 Switching Protocols` `Response` immediately. The `callback`
receives the live `WebSocket` once the upgrade actually completes and owns the entire
connection lifecycle. The callback's future must be `Send + 'static`, which is why every
WebSocket task must avoid `!Send` values held across `.await`.

### `on_failed_upgrade` and `OnFailedUpgrade`

When the protocol upgrade itself fails (after `on_upgrade` already returned its
response), the default behavior is to silently ignore the error. Override it:

```rust
pub fn on_failed_upgrade<C>(self, callback: C) -> WebSocketUpgrade<C>
where
    C: OnFailedUpgrade,
```

`OnFailedUpgrade` is a trait; `WebSocketUpgrade<F>` defaults `F` to `DefaultOnFailedUpgrade`,
which ignores failures. ALWAYS attach `on_failed_upgrade` in production if you need to
log or count failed upgrades, because the default is silent.

### Builder methods (verified signatures)

```rust
pub fn protocols<I>(self, protocols: I) -> Self
where I: IntoIterator, I::Item: Into<Cow<'static, str>>

pub fn max_message_size(self, max: usize) -> Self
pub fn max_frame_size(self, max: usize) -> Self
pub fn write_buffer_size(self, size: usize) -> Self
pub fn max_write_buffer_size(self, max: usize) -> Self
```

`max_message_size` / `max_frame_size` cap inbound payload sizes (DoS protection);
`protocols` negotiates a `Sec-WebSocket-Protocol` subprotocol.

---

## 3. The `WebSocket` connection type

`WebSocket` is the live, bidirectional connection delivered to the `on_upgrade`
callback. Three operations matter:

- `recv().await` returns `Option<Result<Message, Error>>`. `None` means the connection
  closed; `Some(Err(_))` is a protocol/transport error.
- `send(msg).await` returns `Result<(), Error>`. An `Err` means the peer is gone.
- `split()` consumes the `WebSocket` and returns a `(SplitSink, SplitStream)` pair via
  `futures_util::StreamExt::split`. `WebSocket` implements `Sink<Message>` and `Stream`,
  which is what makes `.split()`, `.send()` (via `SinkExt`) and `.next()` (via
  `StreamExt`) available.

---

## 4. The `Message` enum

Verified verbatim enum definition from docs.rs (Axum 0.8):

```rust
pub enum Message {
    Text(Utf8Bytes),
    Binary(Bytes),
    Ping(Bytes),
    Pong(Bytes),
    Close(Option<CloseFrame>),
}
```

Useful methods on `Message`: `Message::text(s)` and `Message::binary(b)` constructors,
`into_data()` (consume to binary `Bytes`), `into_text()` (consume to `Utf8Bytes`),
`to_text()` (borrow as `&str`). `Close` carries an optional `CloseFrame` with a close
code and reason.

### VERSION-CRITICAL : the 0.7 -> 0.8 `Message` type change

This is the single highest-impact 0.7 -> 0.8 break in WebSocket code. In 0.8 the
`Text`/`Binary`/`Ping`/`Pong` variants stopped wrapping owned `String`/`Vec<u8>` and now
wrap the cheap reference-counted buffer types `Utf8Bytes` and `Bytes`.

```rust
// axum 0.7 — variant payload types
Message::Text(String)
Message::Binary(Vec<u8>)
Message::Ping(Vec<u8>)
Message::Pong(Vec<u8>)

// axum 0.8 — variant payload types
Message::Text(Utf8Bytes)
Message::Binary(Bytes)
Message::Ping(Bytes)
Message::Pong(Bytes)
```

Construction migration (the most common break):

```rust
// axum 0.7
let m = Message::Text("hello".to_string());
let b = Message::Binary(vec![1, 2, 3]);

// axum 0.8 — String / Vec<u8> both impl Into for the new types
let m = Message::Text("hello".into());
let b = Message::Binary(vec![1, 2, 3].into());
// or the constructor helpers, version-stable in 0.8:
let m = Message::text("hello");
let b = Message::binary(vec![1, 2, 3]);
```

Pattern-match migration: on 0.7 `Message::Text(s)` binds `s: String`; on 0.8 it binds
`s: Utf8Bytes`. `Utf8Bytes` derefs to `&str`, so `println!("{s}")` and string comparison
keep working, but code that needed an owned `String` must call `.to_string()` /
`.as_str().to_owned()`. Likewise `Bytes` derefs to `&[u8]`.

ALWAYS annotate WebSocket code samples with the target version. NEVER write
`Message::Text(String)` for an 0.8 example.

---

## 5. The verified echo handler

Verbatim from the docs.rs `axum::extract::ws` module page (Axum 0.8):

```rust
async fn handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(msg) = socket.recv().await {
        let msg = if let Ok(msg) = msg {
            msg
        } else {
            // client disconnected
            return;
        };

        if socket.send(msg).await.is_err() {
            // client disconnected
            return;
        }
    }
}
```

Routed with (note `any`, since the upgrade request is a `GET` but routing it with `any`
is the documented convention):

```rust
use axum::{
    extract::ws::{WebSocket, WebSocketUpgrade},
    response::Response,
    routing::any,
    Router,
};

let app: Router = Router::new().route("/ws", any(handler));
```

This handler is sequential: it `recv`s and `send`s on the same `&mut socket`. That is
correct for a pure echo. It is NOT sufficient when the server must push messages
independently of what the client sends — see the concurrent pattern below.

---

## 6. The verified concurrent pattern : split + spawn + select

A single `WebSocket` cannot be borrowed mutably by two tasks at once, so it cannot
`recv` and `send` concurrently as-is. The official `examples/websockets/src/main.rs`
solves this by splitting the socket and driving each half in its own `tokio::spawn`
task, then using `tokio::select!` to abort the survivor when either side ends.

Verified fragments from `tokio-rs/axum/examples/websockets/src/main.rs` (2026-05-20):

```rust
let (mut sender, mut receiver) = socket.split();

let mut send_task = tokio::spawn(async move {
    for i in 0..n_msg {
        if sender
            .send(Message::Text(format!("Server message {i} ...").into()))
            .await
            .is_err()
        {
            return i;
        }
        tokio::time::sleep(std::time::Duration::from_millis(300)).await;
    }
    // ...
});

let mut recv_task = tokio::spawn(async move {
    let mut cnt = 0;
    while let Some(Ok(msg)) = receiver.next().await {
        cnt += 1;
        if process_message(msg, who).is_break() {
            break;
        }
    }
    cnt
});

tokio::select! {
    rv_a = (&mut send_task) => { recv_task.abort(); },
    rv_b = (&mut recv_task) => { send_task.abort(); }
}
```

Key points carried by this pattern:

- `socket.split()` yields `sender` (a `SplitSink`) and `receiver` (a `SplitStream`).
  `.send()` needs `SinkExt`, `.next()` needs `StreamExt`, both from `futures_util`.
- Each half is `move`d into its own `tokio::spawn`, so send and receive run truly
  concurrently on the Tokio runtime.
- `tokio::select!` borrows each `JoinHandle` mutably (`&mut send_task`). When one task
  finishes, the other is `.abort()`ed, guaranteeing the connection is fully torn down
  and no task leaks.
- The send task wraps a `String` with `.into()` to build a `Message::Text` — verified
  0.8 idiom (`format!(...).into()` produces a `Utf8Bytes`).

### The verified `Message`-matching helper

Verbatim from the same example (Axum 0.8):

```rust
fn process_message(msg: Message, who: SocketAddr) -> ControlFlow<(), ()> {
    match msg {
        Message::Text(t) => {
            println!(">>> {who} sent str: {t:?}");
        }
        Message::Binary(d) => {
            println!(">>> {who} sent {} bytes: {d:?}", d.len());
        }
        Message::Close(c) => {
            return ControlFlow::Break(());
        }
        Message::Pong(v) => {
            println!(">>> {who} sent pong with {v:?}");
        }
        Message::Ping(v) => {
            println!(">>> {who} sent ping with {v:?}");
        }
    }
    ControlFlow::Continue(())
}
```

`Ping`/`Pong` are handled automatically by the underlying tungstenite layer (the library
answers a `Ping` with a `Pong`); a handler still sees them and only needs custom logic
for application-level heartbeats. Receiving `Close` is the signal to stop the loop.

---

## 7. Combining with `State` and other extractors

`WebSocketUpgrade` is a `FromRequestParts` extractor, so it composes with any other
parts extractor in any order. The official example combines it with `Option<TypedHeader>`
and `ConnectInfo`:

```rust
async fn ws_handler(
    ws: WebSocketUpgrade,
    user_agent: Option<TypedHeader<headers::UserAgent>>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
) -> impl IntoResponse {
    // read user_agent / addr BEFORE the upgrade, then:
    ws.on_upgrade(move |socket| handle_socket(socket, addr))
}
```

To use application state, add a `State<AppState>` argument and `move` the state into the
`on_upgrade` closure:

```rust
async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
) -> Response {
    ws.on_upgrade(move |socket| handle_socket(socket, state))
}

async fn handle_socket(socket: WebSocket, state: AppState) {
    // state moved in; clone it before moving halves into the spawned tasks
}
```

`ConnectInfo<SocketAddr>` requires the server to be started with
`axum::serve(listener, app.into_make_service_with_connect_info::<SocketAddr>())`.

NEVER read the request body alongside `WebSocketUpgrade`; the upgrade consumes the
connection and a body extractor would conflict. All co-extractors must be
`FromRequestParts`.

---

## 8. Never block the socket task

The `on_upgrade` callback future and any task spawned from it run on Tokio worker
threads. Blocking calls inside them are a hard anti-pattern:

```rust
// WRONG — stalls a Tokio worker, the WebSocket stops answering pings
async fn handle_socket(mut socket: WebSocket) {
    std::thread::sleep(std::time::Duration::from_secs(5)); // blocks the worker
    let result = expensive_cpu_loop();                     // blocks the worker
}

// RIGHT — async sleep, CPU work offloaded
async fn handle_socket(mut socket: WebSocket) {
    tokio::time::sleep(std::time::Duration::from_secs(5)).await;
    let result = tokio::task::spawn_blocking(expensive_cpu_loop).await.unwrap();
}
```

ALWAYS use `tokio::time::sleep` instead of `std::thread::sleep`, async I/O instead of
synchronous I/O, and `tokio::task::spawn_blocking` for CPU-bound work, inside any
WebSocket handler or spawned socket task. Blocking one worker starves every other task
scheduled on it; the connection misses ping deadlines and the peer eventually times out.

---

## Suggested skill snippets (version-annotated)

1. `Cargo.toml` with the `ws` feature + `futures-util` (section 1).
2. The verified echo handler + `any` route (section 5) — Axum 0.8.
3. The 0.7 vs 0.8 `Message` construction before/after block (section 4).
4. The split + spawn + select concurrent block (section 6) — Axum 0.8.
5. The `process_message` `match` over all five `Message` variants (section 6) — 0.8.
6. The `WebSocketUpgrade` + `State` + `on_upgrade(move ...)` composition (section 7).
7. The block-vs-non-block contrast (section 8).

---

## Sources verified (2026-05-20)

| URL | Used for |
|-----|----------|
| https://docs.rs/axum/latest/axum/extract/ws/index.html | `ws` feature gate, verbatim echo handler, WebSocket recv/send/split |
| https://docs.rs/axum/latest/axum/extract/ws/struct.WebSocketUpgrade.html | `on_upgrade`, `on_failed_upgrade`, builder method signatures, `OnFailedUpgrade` |
| https://docs.rs/axum/latest/axum/extract/ws/enum.Message.html | verbatim `Message` enum definition, variant types, methods |
| https://github.com/tokio-rs/axum/blob/main/examples/websockets/src/main.rs | split + spawn + select pattern, `process_message`, `WebSocketUpgrade` + `ConnectInfo`/`TypedHeader` composition |
| docs/research/fragments/research-b-middleware-tower-realtime.md (section 6) | cross-check of prior verified WebSocket research |
| docs/research/vooronderzoek-axum.md (section 6, sub-topic 9) | 0.7 -> 0.8 `Message` type change flagged as high-impact break |

### Verification notes

- The `Message` enum, the echo handler, the `on_upgrade` signature, the builder method
  signatures, and the `process_message`/split/select fragments were all transcribed
  directly from fetched official content on 2026-05-20.
- The 0.7 payload types (`String`/`Vec<u8>`) are stated as the pre-0.8 shape per the
  Axum CHANGELOG breaking-change note ("`Message` now uses `Bytes` in place of
  `Vec<u8>`, and a new `Utf8Bytes` type in place of `String`"). The 0.8 shape is the
  verbatim docs.rs enum. Skill code samples for 0.7 should be marked clearly as 0.7.
- `Ping`/`Pong` auto-response behavior is library (tungstenite) behavior; the skill
  should present it as "handled by the underlying layer", not as an Axum API guarantee.
</content>
</invoke>
