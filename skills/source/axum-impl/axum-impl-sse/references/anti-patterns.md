# axum-impl-sse : Anti-Patterns

The six SSE mistakes that cause compile errors, runtime panics, and dropped
connections. Each entry states WHAT goes wrong, WHY it fails, and the FIX.

## Anti-pattern 1: returning a bare Stream of Event from the handler

```rust
// WRONG: item type is Event, not Result<Event, E>
async fn sse_handler() -> Sse<impl Stream<Item = Event>> {
    let stream = stream::repeat_with(|| Event::default().data("hi!"));
    Sse::new(stream)
}
```

WHY it fails: `Sse::new` requires `S: TryStream<Ok = Event>`, which is exactly
`Stream<Item = Result<Event, E>>`. A `Stream<Item = Event>` yields bare `Event`s
and does NOT satisfy the bound. The compiler reports a trait bound error such as
`the trait bound ... TryStream ... is not satisfied` or `expected Result, found
Event`.

FIX: lift the stream with `.map(Ok)` so each `Event` becomes `Ok(event)`. The
error type is then `std::convert::Infallible`.

```rust
// CORRECT
async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::repeat_with(|| Event::default().data("hi!")).map(Ok);
    Sse::new(stream)
}
```

## Anti-pattern 2: omitting KeepAlive on a long-lived stream

```rust
// WRONG: a quiet long-lived stream with no keep-alive
async fn notifications(rx: Receiver<Event>) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = ReceiverStream::new(rx).map(Ok);
    Sse::new(stream) // no .keep_alive(...)
}
```

WHY it fails: when no event is emitted for tens of seconds, intermediary proxies
and load balancers see an idle TCP connection and drop it. The browser cannot
distinguish a live-but-quiet stream from a dead one, so it may sit on a connection
that is already gone. Notifications silently stop arriving.

FIX: ALWAYS attach a `KeepAlive`. It injects an invisible comment record during
idle gaps that holds the connection open.

```rust
// CORRECT
Sse::new(stream).keep_alive(KeepAlive::new())
```

## Anti-pattern 3: calling a single-use Event builder method twice

```rust
// WRONG: .data called twice on one Event
let evt = Event::default()
    .data("first line")
    .data("second line"); // panics: data already set
```

WHY it fails: `Event::data` and `Event::json_data` panic if data was already
set. `Event::event`, `Event::id`, and `Event::retry` panic if called more than
once. The panic message resembles `data already set` and crashes the connection
task at runtime.

FIX: build one `Event` per logical message and call each single-use method at
most once. To put multiple lines in one event, pass a single string with
newlines to `.data`: it splits newlines into multiple `data:` lines. Only
`.comment` may be called repeatedly.

```rust
// CORRECT: newlines inside one .data call
let evt = Event::default().data("first line\nsecond line");
```

## Anti-pattern 4: newlines in .event, .id, or .comment

```rust
// WRONG: newline in an event name
let evt = Event::default()
    .event("user\njoined") // panics: contains a carriage return or newline
    .data("payload");
```

WHY it fails: the SSE wire format uses newlines as record delimiters, so the
`event:`, `id:`, and comment fields forbid CR and LF. `Event::event`,
`Event::id`, and `Event::comment` panic when their value contains a newline or
carriage return. `Event::id` also panics on a null character.

FIX: keep `.event`, `.id`, and `.comment` values single-line. `.data` is the
only field that accepts newlines.

```rust
// CORRECT
let evt = Event::default().event("user-joined").data("payload");
```

## Anti-pattern 5: blocking or CPU-heavy work in the stream poll path

```rust
// WRONG: blocking call inside the stream closure
let stream = stream::repeat_with(|| {
    let data = std::fs::read_to_string("/big/file").unwrap(); // blocks the worker
    Event::default().data(data)
}).map(Ok);
```

WHY it fails: the SSE stream is polled on a Tokio worker thread. A blocking call
(`std::fs`, a synchronous database driver, a `std::thread::sleep`) or a long CPU
loop inside `poll_next` stalls that worker, which delays every other task it was
scheduled to run. Throughput collapses across the whole server.

FIX: do async work in a `tokio::spawn` task and synchronous or CPU-heavy work in
`tokio::task::spawn_blocking`. Feed the results through an `mpsc` channel and
stream the channel. See `references/examples.md` Example 8.

## Anti-pattern 6: a non-Send or non-'static stream

```rust
// WRONG: Rc is not Send
use std::rc::Rc;
let shared = Rc::new(load_config());
let stream = stream::repeat_with(move || {
    Event::default().data(shared.name.clone()) // Rc captured: stream is not Send
}).map(Ok);
Sse::new(stream) // compile error: Send bound unsatisfied
```

WHY it fails: `Sse::new` requires `S: Send + 'static`. The connection task may
outlive the handler and move across Tokio worker threads. An `Rc`, a borrowed
reference, or any non-`Send` capture breaks the `Send` bound; a borrowed value
breaks the `'static` bound. The compiler reports `... cannot be sent between
threads safely` or `borrowed value does not live long enough`.

FIX: own all captured data and use `Arc` instead of `Rc` for shared state.

```rust
// CORRECT
use std::sync::Arc;
let shared = Arc::new(load_config());
let stream = stream::repeat_with(move || {
    Event::default().data(shared.name.clone())
}).map(Ok);
Sse::new(stream)
```

## Bonus anti-pattern: using WebSockets for a receive-only client

WHY it fails: WebSockets require an HTTP upgrade handshake, carry the overhead
of bidirectional framing, and need manual reconnection logic on the client. When
the client only ever receives (notifications, feeds, logs, progress, an LLM
token stream), all of that is wasted complexity.

FIX: use SSE. It is plain HTTP, passes proxies and load balancers unchanged, and
the browser `EventSource` reconnects automatically with `Last-Event-ID`
resumption. Reach for WebSockets (`axum-impl-websockets`) only when the client
must also send messages on the same connection.
