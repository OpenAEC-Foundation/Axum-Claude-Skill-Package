# axum-impl-sse : Examples

Working, version-annotated SSE code. Every snippet compiles on Axum 0.7 and
0.8: the `axum::response::sse` module is API-stable across both versions, so
there are no version-divergent code paths. Snippets verified against the
official `tokio-rs/axum` SSE example and docs.rs on 2026-05-20.

## Example 1: the verbatim official handler

From `tokio-rs/axum/examples/sse/src/main.rs`. This is the canonical SSE shape.

```rust
use axum::{
    response::sse::{Event, Sse},
    routing::get,
    Router,
};
use futures_util::stream::{self, Stream};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt as _;

async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok)                            // Event -> Ok(Event); error type = Infallible
        .throttle(Duration::from_secs(1));  // pace the otherwise-instant stream

    Sse::new(stream).keep_alive(
        axum::response::sse::KeepAlive::new()
            .interval(Duration::from_secs(1))
            .text("keep-alive-text"),
    )
}

// route registration
let app = Router::new().route("/sse", get(sse_handler));
```

Three load-bearing details:

1. `.map(Ok)` lifts the infallible `Event` stream into `Result<Event, Infallible>`.
2. `.throttle(...)` from `tokio_stream::StreamExt` paces an otherwise-instant
   `repeat_with` stream.
3. `KeepAlive` is attached unconditionally, even on this once-per-second demo.

## Example 2: minimal handler with KeepAlive::default()

```rust
use axum::response::sse::{Event, KeepAlive, Sse};
use futures_util::stream::{self, Stream};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt as _;

async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok)
        .throttle(Duration::from_secs(1));

    Sse::new(stream).keep_alive(KeepAlive::default())
}
```

`KeepAlive::default()` equals `KeepAlive::new()`: a 15-second interval with an
empty-comment payload.

## Example 3: full runnable app with main

```rust
use axum::{
    response::sse::{Event, KeepAlive, Sse},
    routing::get,
    Router,
};
use futures_util::stream::{self, Stream};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt as _;

#[tokio::main]
async fn main() {
    let app = Router::new().route("/sse", get(sse_handler));

    // axum 0.7 and 0.8: serve with tokio's TcpListener (axum::Server was removed in 0.7)
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let mut counter = 0u32;
    let stream = stream::repeat_with(move || {
        counter += 1;
        Event::default().data(format!("tick {counter}"))
    })
    .map(Ok)
    .throttle(Duration::from_secs(1));

    Sse::new(stream).keep_alive(KeepAlive::default())
}
```

## Example 4: multi-field Event

```rust
use axum::response::sse::Event;

// each single-use builder method is called at most once; calling twice panics
let evt = Event::default()
    .event("user-joined")           // EventSource: addEventListener("user-joined", ...)
    .id("42")                       // becomes the client's lastEventId, resent on reconnect
    .data("Alice joined the room"); // the visible payload
```

## Example 5: retry hint and comment-carrying keep-alive

```rust
use axum::response::sse::{Event, KeepAlive};
use std::time::Duration;

// a retry hint tells the browser to wait 5s before reconnecting after a drop
let evt = Event::default()
    .retry(Duration::from_secs(5))
    .data("reconnect hint set");

// keep-alive carrying a named event instead of a bare comment
let ka = KeepAlive::new()
    .interval(Duration::from_secs(10))
    .event(Event::default().comment("ping"));
```

## Example 6: fallible stream from a Tokio broadcast channel

A broadcast receiver can lag and miss messages, so its stream genuinely fails.
The item type is `Result<Event, axum::BoxError>`.

```rust
use axum::response::sse::{Event, KeepAlive, Sse};
use futures_util::Stream;
use tokio::sync::broadcast;
use tokio_stream::wrappers::BroadcastStream;
use tokio_stream::StreamExt as _;

// rx: broadcast::Receiver<String>, taken from shared State
async fn events(rx: broadcast::Receiver<String>)
    -> Sse<impl Stream<Item = Result<Event, axum::BoxError>>>
{
    let stream = BroadcastStream::new(rx).map(|msg| {
        // BroadcastStream yields Result<String, BroadcastStreamRecvError>
        msg.map(|text| Event::default().data(text)) // Ok branch -> Event
           .map_err(axum::BoxError::from)            // Err branch -> BoxError
    });
    Sse::new(stream).keep_alive(KeepAlive::new())
}
```

## Example 7: JSON payload event (feature `json`)

```rust
use axum::response::sse::Event;

#[derive(serde::Serialize)]
struct Update {
    progress: u8,
    label: String,
}

// json_data returns Result<Event, axum::Error> because serialization can fail
fn progress_event(progress: u8) -> Result<Event, axum::Error> {
    Event::default().json_data(Update {
        progress,
        label: "rendering".to_string(),
    })
}
```

`Cargo.toml` must enable the `json` feature:

```toml
axum = { version = "0.8", features = ["json"] }
```

## Example 8: channel-fed stream, heavy work off the poll path

NEVER run blocking or CPU-heavy work inside the SSE stream's poll path. Spawn
the work in a separate task and stream the channel instead.

```rust
use axum::response::sse::{Event, KeepAlive, Sse};
use futures_util::Stream;
use tokio::sync::mpsc;
use tokio_stream::wrappers::ReceiverStream;
use tokio_stream::StreamExt as _;
use std::convert::Infallible;

async fn jobs() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let (tx, rx) = mpsc::channel::<Event>(16);

    // producer task: does the slow work, never touches the SSE poll path
    tokio::spawn(async move {
        for step in 1..=5u8 {
            // CPU-heavy or blocking work belongs in spawn_blocking, not here
            let result = tokio::task::spawn_blocking(move || step * step)
                .await
                .unwrap();
            let evt = Event::default().data(format!("step {step} -> {result}"));
            if tx.send(evt).await.is_err() {
                break; // client disconnected, receiver dropped
            }
        }
    });

    let stream = ReceiverStream::new(rx).map(Ok);
    Sse::new(stream).keep_alive(KeepAlive::default())
}
```

## Example 9: the EventSource client side

The browser consumes the SSE endpoint with `EventSource`. Included for context;
this is the client the Rust handler streams to.

```javascript
// default "message" events: anything sent without .event(...)
const source = new EventSource("/sse");
source.onmessage = (e) => console.log("data:", e.data);

// named events: matches Event::default().event("user-joined")
source.addEventListener("user-joined", (e) => console.log("joined:", e.data));

// the browser reconnects automatically and resends the last id as
// the Last-Event-ID header, which maps to Event::default().id(...)
source.onerror = () => console.log("connection lost, browser will retry");
```

## Cargo.toml dependencies

```toml
[dependencies]
axum = "0.8"                       # or "0.7"; SSE API is identical
tokio = { version = "1", features = ["full"] }
futures-util = "0.3"
tokio-stream = { version = "0.1", features = ["sync"] }  # "sync" enables BroadcastStream
serde = { version = "1", features = ["derive"] }         # only for json_data examples
# enable axum's json feature for Event::json_data:
# axum = { version = "0.8", features = ["json"] }
```
