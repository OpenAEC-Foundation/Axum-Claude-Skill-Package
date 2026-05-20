# Topic Research : axum-impl-sse

> Skill : `axum-impl-sse`
> Cluster : B (Realtime)
> Researched : 2026-05-20
> Target versions : Axum 0.7 and 0.8 (docs.rs current : axum 0.8.9)
> Module : `axum::response::sse`

All API signatures, builder methods and code examples below were WebFetch-verified
against docs.rs and the official `tokio-rs/axum` SSE example on 2026-05-20. No API is
inferred. The SSE module is API-stable between 0.7 and 0.8; the breaking changes in
those releases (path syntax, WebSocket `Message` types, removed `axum::Server`) do NOT
touch `axum::response::sse`.

---

## 1. What SSE is, and SSE vs WebSocket

Server-Sent Events is a **one-way, server-to-client** push channel over a single
long-lived HTTP response. The body is a `text/event-stream` of newline-delimited
records; the browser consumes it with `new EventSource(url)`. Axum's `axum::response::sse`
module wraps a Rust `Stream` as such a response.

Deterministic decision rule:

- **Use SSE** when the data flow is push-only server-to-client: notifications, live
  feeds, log tailing, progress updates, LLM token streaming. SSE is plain HTTP, so it
  passes proxies and load balancers unchanged, auto-reconnects in the browser (with
  `Last-Event-ID` resumption), and needs no protocol upgrade.
- **Use WebSockets** (`axum::extract::ws`, skill `axum-impl-websockets`) when the client
  must also send messages on the same connection: chat, collaborative editing,
  multiplayer, interactive control. WebSockets are bidirectional and carry binary
  frames; SSE is text-only and one-way.
- NEVER reach for WebSockets when the client only ever receives. SSE is simpler, has
  less overhead, and gives free browser-side reconnection.

---

## 2. The `Sse<S>` struct (verified)

```rust
pub struct Sse<S> { /* private fields */ }
```

### `Sse::new` (verified signature)

```rust
pub fn new(stream: S) -> Self
where
    S: TryStream<Ok = Event> + Send + 'static,
    S::Error: Into<BoxError>
```

`Sse::new` takes a stream whose successful item is an `Event`. A `TryStream<Ok = Event>`
is exactly a `Stream<Item = Result<Event, E>>` where `E: Into<BoxError>`. The stream
must be `Send + 'static` because the connection task may outlive the handler and move
across Tokio worker threads.

### `Sse::keep_alive` (verified signature)

```rust
pub fn keep_alive(self, keep_alive: KeepAlive) -> Sse<KeepAliveStream<S>>
```

Available with the `tokio` feature (on by default). Wraps the inner stream in a
`KeepAliveStream` that injects keep-alive records during idle gaps. Default behavior
without this call is **no keep-alive**.

### `IntoResponse` impl (verified)

```rust
impl<S, E> IntoResponse for Sse<S>
where
    S: Stream<Item = Result<Event, E>> + Send + 'static,
    E: Into<BoxError>
```

Because `Sse<S>` implements `IntoResponse`, a handler simply returns it. The canonical
return type is `Sse<impl Stream<Item = Result<Event, E>>>`.

---

## 3. The stream item type : `Result<Event, E>` and the `.map(Ok)` lift

This is the single most important mechanic of the skill. `Sse::new` requires a stream
of `Result<Event, E>`, NOT a stream of bare `Event`. The `Result` exists so a stream
that genuinely can fail (a database cursor, a broadcast receiver) can surface its error.

When the stream is infallible, you must still produce `Result` items. ALWAYS lift an
infallible `Stream<Item = Event>` with `.map(Ok)`, which turns each `Event` into
`Ok(event)`. The error type is then `std::convert::Infallible`, and the verified return
type is `Sse<impl Stream<Item = Result<Event, Infallible>>>`.

```rust
// infallible stream of Events -> required Result<Event, Infallible> items
let stream = stream::repeat_with(|| Event::default().data("hi!"))
    .map(Ok);   // lift: Event -> Ok(Event), error type = Infallible
```

For a fallible stream, the item is `Result<Event, YourError>` and `YourError` must
satisfy `Into<BoxError>` (a `Box<dyn std::error::Error + Send + Sync>`). NEVER return a
bare `Stream<Item = Event>` from the handler; it will not satisfy `Sse::new`'s bound.

---

## 4. The `Event` builder (verified)

```rust
pub struct Event { /* private fields */ }
```

`Event` is built fluently starting from `Event::default()`. All verified builder methods:

| Method | Signature | Behavior |
|--------|-----------|----------|
| `default` | `fn default() -> Self` | Empty event, the builder start point. |
| `data` | `fn data<T: AsRef<str>>(self, data: T) -> Self` | Sets the `data:` field. Newlines split into multiple `data:` lines. **Panics if data was already set.** |
| `json_data` | `fn json_data<T: Serialize>(self, data: T) -> Result<Self, axum::Error>` | (feature `json`) Sets `data:` to compact JSON. Returns `Result`. **Panics if data was already set.** |
| `event` | `fn event<T: AsRef<str>>(self, event: T) -> Self` | Sets the `event:` name (the `addEventListener` type). **Panics on newlines/CR or if called twice.** |
| `id` | `fn id<T: AsRef<str>>(self, id: T) -> Self` | Sets the `id:` field (becomes the client's `lastEventId`). **Panics on newline/CR/NUL or if called twice.** |
| `retry` | `fn retry(self, duration: Duration) -> Self` | Sets the client reconnect hint (milliseconds on the wire). **Panics if called twice.** |
| `comment` | `fn comment<T: AsRef<str>>(self, comment: T) -> Self` | Sets a `:comment` line, ignored by clients. **May be called multiple times.** Panics on newline/CR. |

Determinism rules from the verified panic documentation:

- NEVER call `.data` or `.json_data` twice on the same `Event`; it panics. Build one
  `Event` per logical message.
- NEVER call `.event`, `.id`, or `.retry` more than once on the same `Event`; each
  panics on the second call. Only `.comment` is repeatable.
- NEVER put a newline or carriage return into `.event`, `.id`, or `.comment` values; it
  panics. `.data` is the only field that accepts newlines (it splits them into multiple
  `data:` lines).
- An `Event` with an empty `data` field is ignored by the browser. ALWAYS set `.data`
  (or `.comment` for a keep-alive) so the event is delivered.

---

## 5. The `KeepAlive` struct (verified)

```rust
pub struct KeepAlive { /* private fields */ }
```

| Method | Signature | Behavior |
|--------|-----------|----------|
| `new` | `fn new() -> Self` | New `KeepAlive`. Default interval **15 seconds**, default payload an empty comment. |
| `interval` | `fn interval(self, time: Duration) -> Self` | Sets the gap between keep-alive records. Default 15s. |
| `text` | `fn text<I: AsRef<str>>(self, text: I) -> Self` | Keep-alive payload as comment text. **Panics on newline/CR.** |
| `event` | `fn event(self, event: Event) -> Self` | Keep-alive payload as a full `Event` instead of a comment. **Panics on newline/CR.** |

`KeepAlive` also implements `Default` (equivalent to `KeepAlive::new()`).

ALWAYS attach a `KeepAlive` to any long-lived SSE stream via `.keep_alive(...)`. A quiet
stream that emits nothing for minutes looks dead to intermediary proxies and load
balancers, which then drop the idle connection; the keep-alive comment is invisible
traffic that holds the connection open and lets the client distinguish "live but quiet"
from "dead". The default 15-second interval is a safe baseline; lower it only if a
proxy has a shorter idle timeout.

---

## 6. The verified official SSE handler

From `tokio-rs/axum/examples/sse/src/main.rs`, fetched 2026-05-20 (verbatim handler):

```rust
use axum::{
    response::sse::{Event, Sse},
    routing::get,
    Router,
};
use axum_extra::TypedHeader;
use futures_util::stream::{self, Stream};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt as _;

async fn sse_handler(
    TypedHeader(user_agent): TypedHeader<headers::UserAgent>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    println!("`{}` connected", user_agent.as_str());

    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok)
        .throttle(Duration::from_secs(1));

    Sse::new(stream).keep_alive(
        axum::response::sse::KeepAlive::new()
            .interval(Duration::from_secs(1))
            .text("keep-alive-text"),
    )
}

// route registration:
// Router::new().route("/sse", get(sse_handler))
```

Note three load-bearing details verified in the example: (1) `.map(Ok)` lifts the
infallible `Event` stream into `Result<Event, Infallible>`; (2) `.throttle(...)` from
`tokio_stream::StreamExt` paces an otherwise-instant `repeat_with` stream; (3)
`KeepAlive` is attached unconditionally even though this demo stream emits every second.

---

## 7. Additional verified snippets

### 7a. Minimal handler with the module-doc `KeepAlive::default()` form

```rust
use axum::response::sse::{Event, KeepAlive, Sse};
use std::convert::Infallible;
use tokio_stream::StreamExt as _;
use futures_util::stream::{self, Stream};

async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::repeat_with(|| Event::default().data("hi!"))
        .map(Ok)
        .throttle(std::time::Duration::from_secs(1));

    Sse::new(stream).keep_alive(KeepAlive::default())
}
```

### 7b. Multi-field event : `event`, `id`, `data`

```rust
// each builder method called at most once (calling twice panics)
let evt = Event::default()
    .event("user-joined")          // EventSource addEventListener("user-joined", ...)
    .id("42")                      // becomes client lastEventId, sent on reconnect
    .data("Alice joined the room");
```

### 7c. Fallible stream from a Tokio broadcast channel

```rust
use tokio::sync::broadcast;
use tokio_stream::wrappers::BroadcastStream;
use tokio_stream::StreamExt as _;
use axum::response::sse::{Event, KeepAlive, Sse};

// rx: broadcast::Receiver<String> taken from shared State
async fn events(rx: broadcast::Receiver<String>)
    -> Sse<impl futures_util::Stream<Item = Result<Event, axum::BoxError>>>
{
    let stream = BroadcastStream::new(rx).map(|msg| {
        // BroadcastStream yields Result<String, BroadcastStreamRecvError>
        msg.map(|text| Event::default().data(text))
           .map_err(axum::BoxError::from)
    });
    Sse::new(stream).keep_alive(KeepAlive::new())
}
```

### 7d. JSON payload event (feature `json`)

```rust
#[derive(serde::Serialize)]
struct Update { progress: u8 }

// json_data returns Result because serialization can fail
let evt: Result<Event, axum::Error> =
    Event::default().json_data(Update { progress: 73 });
```

### 7e. Comment-only keep-alive event and a retry hint

```rust
// a retry hint tells the browser to wait 5s before reconnecting after a drop
let evt = Event::default()
    .retry(std::time::Duration::from_secs(5))
    .data("reconnect hint set");

// keep-alive carrying a named event instead of a bare comment
let ka = KeepAlive::new()
    .interval(std::time::Duration::from_secs(10))
    .event(Event::default().comment("ping"));
```

---

## 8. Anti-patterns for this skill

1. **Returning `Stream<Item = Event>` from the handler.** Fails `Sse::new`'s
   `TryStream<Ok = Event>` bound. FIX: `.map(Ok)` to produce `Result<Event, Infallible>`.
2. **Omitting `.keep_alive(...)` on a long-lived stream.** Idle connections get dropped
   by proxies and the client cannot detect a dead stream. FIX: always attach a
   `KeepAlive`.
3. **Calling `.data` / `.event` / `.id` / `.retry` twice on one `Event`.** Each panics
   at runtime. FIX: one builder chain per `Event`, each single-use method once.
4. **Newlines in `.event`, `.id`, or `.comment`.** Panics (SSE comments/names forbid
   CR/LF). Only `.data` accepts newlines.
5. **Blocking or CPU-heavy work inside the SSE stream's poll path.** Stalls the Tokio
   worker. FIX: `tokio::spawn` for async work, `spawn_blocking` for sync/CPU work; feed
   results through a channel and stream that.
6. **Non-`Send` / non-`'static` stream.** `Sse::new` requires `Send + 'static`; an `Rc`
   or borrowed data in the stream breaks the bound. FIX: own all captured data, use
   `Arc` for shared state.

---

## Sources verified (2026-05-20)

| URL | Used for |
|-----|----------|
| https://docs.rs/axum/latest/axum/response/sse/index.html | Module item list, `KeepAlive::default()` example, module overview |
| https://docs.rs/axum/latest/axum/response/sse/struct.Sse.html | `Sse<S>` definition, `Sse::new` + `keep_alive` signatures, `IntoResponse` impl bounds |
| https://docs.rs/axum/latest/axum/response/sse/struct.Event.html | `Event` builder methods (`data`, `json_data`, `event`, `id`, `retry`, `comment`) + documented panics |
| https://docs.rs/axum/latest/axum/response/sse/struct.KeepAlive.html | `KeepAlive` definition, `new`/`interval`/`text`/`event`, 15s default interval |
| https://github.com/tokio-rs/axum/blob/main/examples/sse/src/main.rs | Verbatim official `sse_handler`, imports, `.map(Ok)` + `.throttle` + `KeepAlive` usage |

Cross-reference: `docs/research/vooronderzoek-axum.md` section 6, and
`docs/research/fragments/research-b-middleware-tower-realtime.md` section 7.

### Verification notes

- The SSE module is API-stable across Axum 0.7 and 0.8; none of the 0.7-to-0.8 breaking
  changes touch `axum::response::sse`. Examples above apply unchanged to both versions.
- `KeepAlive::keep_alive` and the `KeepAliveStream` wrapper are gated behind the `tokio`
  feature, which is enabled by default in Axum.
- `Event::json_data` is gated behind the `json` feature.
- `BoxError` is `axum::BoxError` (`Box<dyn std::error::Error + Send + Sync>`), the error
  type the SSE stream's `E` must convert into.
