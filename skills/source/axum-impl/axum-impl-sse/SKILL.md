---
name: axum-impl-sse
description: >
  Use when an Axum handler must push a one-way server-to-client stream over
  plain HTTP: live notifications, log tailing, progress updates, an LLM token
  feed, or any "the browser only ever receives" channel consumed by an
  EventSource.
  Prevents the compile error from returning a bare Stream of Event instead of
  the required Result<Event, E> item type, prevents the dropped-connection bug
  from omitting KeepAlive on a quiet long-lived stream, prevents the runtime
  panic from calling a single-use Event builder method twice, and prevents
  reaching for WebSockets when the client never sends anything back.
  Covers axum::response::sse::Sse, Event, KeepAlive, the TryStream<Ok = Event>
  bound on Sse::new, the .map(Ok) infallible lift, the Event builder methods
  (.data, .json_data, .event, .id, .retry, .comment) with their documented
  panics, the json and tokio feature flags, and the SSE versus WebSocket
  decision.
  Keywords: axum SSE, server-sent events, axum::response::sse, Sse::new,
  Event, KeepAlive, EventSource, text/event-stream, TryStream, map(Ok),
  Result Event Infallible, BoxError, keep_alive, throttle, broadcast stream,
  json_data, the trait bound Stream is not satisfied, expected Result found
  Event, my SSE connection keeps dropping, events stop after a minute,
  connection closed by proxy, panicked at data already set, how do I stream
  to the browser, how do I push live updates, what is SSE, SSE vs websocket,
  one-way push, live feed not updating.
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# axum-impl-sse

## Overview

Server-Sent Events (SSE) is a one-way, server-to-client push channel over a
single long-lived HTTP response with body type `text/event-stream`. The browser
consumes it with `new EventSource(url)` and reconnects automatically when the
connection drops.

Axum exposes SSE through the `axum::response::sse` module. The module has
exactly three public types:

| Type | Role |
|------|------|
| `Sse<S>` | The response. Wraps a `Stream` and implements `IntoResponse`. |
| `Event` | One SSE record, built fluently from `Event::default()`. |
| `KeepAlive` | Idle-gap filler that holds quiet connections open. |

The `axum::response::sse` module is API-stable between Axum 0.7 and 0.8. None of
the 0.7-to-0.8 breaking changes (path syntax, WebSocket `Message` types, removed
`axum::Server`) touch this module, so every snippet in this skill compiles
unchanged on both versions. There are no version-divergent code paths here.

## Quick Reference

| Need | Do this |
|------|---------|
| Return SSE from a handler | Return `Sse<impl Stream<Item = Result<Event, E>>>` |
| Build the response | `Sse::new(stream)` |
| Lift an infallible `Event` stream | `.map(Ok)` so items become `Result<Event, Infallible>` |
| Hold a quiet connection open | `.keep_alive(KeepAlive::new())` |
| One text message | `Event::default().data("text")` |
| A named event | `Event::default().event("name").data("text")` |
| A JSON payload | `Event::default().json_data(value)?` (feature `json`) |
| Resumable event | `Event::default().id("42").data("text")` |
| Pace an instant stream | `.throttle(Duration)` from `tokio_stream::StreamExt` |

| `Sse` method | Signature |
|--------------|-----------|
| `Sse::new` | `fn new(stream: S) -> Self where S: TryStream<Ok = Event> + Send + 'static, S::Error: Into<BoxError>` |
| `Sse::keep_alive` | `fn keep_alive(self, keep_alive: KeepAlive) -> Sse<KeepAliveStream<S>>` (feature `tokio`, on by default) |

Full API signatures and feature-flag detail: `references/methods.md`.

## Decision Trees

### SSE or WebSocket

```
Does the client ever send messages on this connection?
├── NO  (client only receives: notifications, feeds, logs, progress, token stream)
│        -> Use SSE. This skill. axum::response::sse.
│           Plain HTTP, passes proxies, free browser auto-reconnect.
└── YES (chat, collaborative editing, multiplayer, interactive control)
         -> Use WebSockets. See axum-impl-websockets.
            Bidirectional, carries binary frames.
```

ALWAYS choose SSE when the data flow is push-only. NEVER reach for WebSockets
for a receive-only client: SSE is simpler, has less overhead, needs no protocol
upgrade, and gives free reconnection with `Last-Event-ID` resumption.

### What item type does my stream yield

```
Can the stream itself fail at runtime?
├── NO  (a timer, a counter, repeat_with, a fixed iterator)
│        -> Stream<Item = Event>, then .map(Ok)
│           -> Result<Event, Infallible>
│           Return: Sse<impl Stream<Item = Result<Event, Infallible>>>
└── YES (a database cursor, a broadcast::Receiver, a channel)
         -> map each item into Result<Event, YourError>
            YourError MUST satisfy Into<BoxError>
            Return: Sse<impl Stream<Item = Result<Event, axum::BoxError>>>
```

`Sse::new` requires `S: TryStream<Ok = Event>`, which is exactly
`Stream<Item = Result<Event, E>>`. A bare `Stream<Item = Event>` does NOT
satisfy this bound. This is the single most common SSE compile error.

### Do I need KeepAlive

```
Will this stream ever stay idle longer than a few seconds?
├── YES (almost always: event feeds, notifications, log tails)
│        -> ALWAYS attach .keep_alive(KeepAlive::new())
└── NO  (a short fixed burst that completes in under a second)
         -> KeepAlive is optional, but attaching it is still safe
```

When unsure, attach `KeepAlive`. The default 15-second comment interval is a
safe baseline and costs nothing on an active stream.

## Patterns

### Pattern: minimal infallible SSE handler

The canonical shape. Verified against the official `tokio-rs/axum` SSE example.

```rust
use axum::response::sse::{Event, KeepAlive, Sse};
use futures_util::stream::{self, Stream};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt as _;

async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok)                              // Event -> Ok(Event); error type = Infallible
        .throttle(Duration::from_secs(1));    // pace the otherwise-instant stream

    Sse::new(stream).keep_alive(KeepAlive::default())
}

// Router::new().route("/sse", get(sse_handler))   // identical on axum 0.7 and 0.8
```

`.map(Ok)` is load-bearing: it converts `Stream<Item = Event>` into
`Stream<Item = Result<Event, Infallible>>`, which satisfies the
`TryStream<Ok = Event>` bound on `Sse::new`.

### Pattern: build a multi-field Event

Each single-use builder method is called at most once per `Event`.

```rust
let evt = Event::default()
    .event("user-joined")          // EventSource: addEventListener("user-joined", ...)
    .id("42")                      // becomes the client's lastEventId, resent on reconnect
    .data("Alice joined the room"); // the visible payload
```

NEVER call `.data`, `.event`, `.id`, or `.retry` twice on the same `Event`:
each panics on the second call. Build one fresh `Event` per logical message.
Only `.comment` may be called repeatedly.

### Pattern: fallible stream from a broadcast channel

When the stream genuinely can fail, map each item into `Result<Event, E>` where
`E: Into<BoxError>`.

```rust
use axum::response::sse::{Event, KeepAlive, Sse};
use tokio_stream::{wrappers::BroadcastStream, StreamExt as _};
use futures_util::Stream;

// rx: tokio::sync::broadcast::Receiver<String>, taken from shared State
async fn events(rx: tokio::sync::broadcast::Receiver<String>)
    -> Sse<impl Stream<Item = Result<Event, axum::BoxError>>>
{
    let stream = BroadcastStream::new(rx).map(|msg| {
        msg.map(|text| Event::default().data(text)) // Ok branch -> Event
           .map_err(axum::BoxError::from)            // Err branch -> BoxError
    });
    Sse::new(stream).keep_alive(KeepAlive::new())
}
```

### Pattern: JSON payload event

`Event::json_data` requires the `json` feature and returns `Result` because
serialization can fail.

```rust
#[derive(serde::Serialize)]
struct Update { progress: u8 }

// json_data returns Result<Event, axum::Error>
let evt: Result<Event, axum::Error> =
    Event::default().json_data(Update { progress: 73 });
```

### Pattern: KeepAlive on a long-lived stream

A quiet stream that emits nothing for minutes looks dead to proxies and load
balancers, which then drop the idle connection. `KeepAlive` injects an invisible
comment record during idle gaps so the connection stays open and the client can
tell "live but quiet" from "dead".

```rust
use axum::response::sse::{Event, KeepAlive};
use std::time::Duration;

// default: 15-second interval, empty-comment payload
let ka = KeepAlive::new();

// custom interval and a named-event payload instead of a bare comment
let ka = KeepAlive::new()
    .interval(Duration::from_secs(10))
    .event(Event::default().comment("ping"));
```

ALWAYS attach `KeepAlive` to a long-lived SSE stream. Lower the interval below
15 seconds only when a proxy has a shorter idle timeout.

### Pattern: keep heavy work off the poll path

The SSE stream is polled on a Tokio worker thread. NEVER run blocking or
CPU-heavy work inside the stream's `poll_next`. ALWAYS do async work in a
`tokio::spawn` task and sync or CPU work in `tokio::task::spawn_blocking`, then
feed results through a channel and stream that channel as `Event`s. See
`references/examples.md` for the channel-fed pattern.

## Common Mistakes

| Mistake | Result | Fix |
|---------|--------|-----|
| Return `Stream<Item = Event>` | Compile error: `TryStream<Ok = Event>` bound unsatisfied | `.map(Ok)` |
| Omit `.keep_alive(...)` | Idle connection dropped by proxy; client cannot detect a dead stream | Attach `KeepAlive` |
| Call `.data` / `.event` / `.id` / `.retry` twice | Runtime panic | One builder chain per `Event` |
| Newline in `.event` / `.id` / `.comment` | Runtime panic | Only `.data` accepts newlines |
| Non-`Send` or non-`'static` stream (`Rc`, borrowed data) | Compile error on `Sse::new` | Own captured data; use `Arc` |
| Blocking work in the stream poll path | Stalls the Tokio worker | `spawn` / `spawn_blocking` + channel |

Full anti-pattern analysis with WHY each fails: `references/anti-patterns.md`.

## Reference Links

- `references/methods.md` : complete API signatures for `Sse`, `Event`,
  `KeepAlive`, the stream item type, feature flags, and `BoxError`.
- `references/examples.md` : working version-annotated code, including the
  verbatim official handler, a full router with `main`, the channel-fed
  pattern, and the EventSource client side.
- `references/anti-patterns.md` : the six SSE anti-patterns with root-cause
  explanations and fixes.

Related skills:

- `axum-impl-websockets` : the bidirectional alternative; use when the client
  must also send messages.
- `axum-syntax-handlers` : handler signatures, extractors, and return types.

Verified sources (2026-05-20):

- https://docs.rs/axum/latest/axum/response/sse/index.html
- https://docs.rs/axum/latest/axum/response/sse/struct.Sse.html
- https://docs.rs/axum/latest/axum/response/sse/struct.Event.html
- https://docs.rs/axum/latest/axum/response/sse/struct.KeepAlive.html
- https://github.com/tokio-rs/axum/blob/main/examples/sse/src/main.rs
