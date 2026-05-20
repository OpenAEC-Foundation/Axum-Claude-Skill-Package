# axum-impl-sse : Methods Reference

Complete API signatures for the `axum::response::sse` module. All signatures
WebFetch-verified against docs.rs on 2026-05-20. The module is API-stable across
Axum 0.7 and 0.8: every signature below applies unchanged to both versions.

## Module: axum::response::sse

The module has exactly three public types and one wrapper stream type:

| Item | Kind | Purpose |
|------|------|---------|
| `Sse<S>` | struct | The SSE response. Implements `IntoResponse`. |
| `Event` | struct | One SSE record. |
| `KeepAlive` | struct | Idle-gap keep-alive configuration. |
| `KeepAliveStream<S>` | struct | Returned by `Sse::keep_alive`; the wrapped stream. |

Import path:

```rust
use axum::response::sse::{Event, KeepAlive, Sse};
```

## Sse<S>

```rust
pub struct Sse<S> { /* private fields */ }
```

A response that streams `Event`s as `text/event-stream`. Generic over the inner
stream type `S`.

### Sse::new

```rust
pub fn new(stream: S) -> Self
where
    S: TryStream<Ok = Event> + Send + 'static,
    S::Error: Into<BoxError>
```

Creates an SSE response from a stream.

- `S: TryStream<Ok = Event>` means `S` is a `Stream<Item = Result<Event, E>>`.
  A bare `Stream<Item = Event>` does NOT satisfy this bound.
- `S::Error: Into<BoxError>` means the error type must convert into
  `axum::BoxError` (`Box<dyn std::error::Error + Send + Sync>`).
- `Send + 'static` is required because the connection task may outlive the
  handler and move across Tokio worker threads.

For an infallible stream, the error type is `std::convert::Infallible`, which
satisfies `Into<BoxError>`.

### Sse::keep_alive

```rust
pub fn keep_alive(self, keep_alive: KeepAlive) -> Sse<KeepAliveStream<S>>
```

Available on crate feature `tokio` only. The `tokio` feature is enabled by
default in Axum. Wraps the inner stream in a `KeepAliveStream` that injects
keep-alive records during idle gaps. Without this call there is NO keep-alive.

### IntoResponse impl

```rust
impl<S, E> IntoResponse for Sse<S>
where
    S: Stream<Item = Result<Event, E>> + Send + 'static,
    E: Into<BoxError>
```

Because `Sse<S>` implements `IntoResponse`, a handler returns it directly. The
canonical handler return type is:

```rust
Sse<impl Stream<Item = Result<Event, Infallible>>>   // infallible stream
Sse<impl Stream<Item = Result<Event, axum::BoxError>>> // fallible stream
```

Note: `new` requires a `TryStream`; the `IntoResponse` impl accepts any
`Stream` yielding `Result<Event, E>`. In practice both describe the same
`Result<Event, E>` item stream.

## Event

```rust
pub struct Event { /* private fields */ }
```

One SSE record. Built fluently starting from `Event::default()`. An `Event` with
an empty `data` field is ignored by the browser; ALWAYS set `.data` (or
`.comment` for a keep-alive payload).

| Method | Signature | Behavior |
|--------|-----------|----------|
| `default` | `fn default() -> Self` | New empty event, the builder start point. |
| `data` | `fn data<T: AsRef<str>>(self, data: T) -> Self` | Sets the `data:` field. Newlines split into multiple `data:` lines. |
| `json_data` | `fn json_data<T: Serialize>(self, data: T) -> Result<Self, axum::Error>` | Feature `json`. Sets `data:` to compact JSON. Returns `Result`. |
| `event` | `fn event<T: AsRef<str>>(self, event: T) -> Self` | Sets the `event:` name used by `addEventListener`. |
| `id` | `fn id<T: AsRef<str>>(self, id: T) -> Self` | Sets the `id:` field; becomes the client's `lastEventId`. |
| `retry` | `fn retry(self, duration: Duration) -> Self` | Sets the client reconnect hint (sent as milliseconds). |
| `comment` | `fn comment<T: AsRef<str>>(self, comment: T) -> Self` | Adds a `:comment` line, ignored by clients. |

### Documented panics

| Method | Panics when |
|--------|-------------|
| `data` | data was already set on this `Event` |
| `json_data` | data was already set on this `Event` |
| `event` | value contains a newline or carriage return, OR called more than once |
| `id` | value contains a newline, carriage return, or null character, OR called more than once |
| `retry` | called more than once |
| `comment` | value contains a newline or carriage return |

Determinism rules:

- NEVER call `.data` or `.json_data` twice on the same `Event`.
- NEVER call `.event`, `.id`, or `.retry` more than once on the same `Event`.
- `.comment` is the only repeatable builder method.
- NEVER put a newline or carriage return into `.event`, `.id`, or `.comment`.
- `.data` is the only field that accepts newlines; it splits them into multiple
  `data:` lines.

## KeepAlive

```rust
pub struct KeepAlive { /* private fields */ }
```

Configures the keep-alive records injected during idle gaps. Also implements
`Default`, equivalent to `KeepAlive::new()`.

| Method | Signature | Behavior |
|--------|-----------|----------|
| `new` | `fn new() -> Self` | New `KeepAlive`. Default interval 15 seconds, default payload an empty comment. |
| `interval` | `fn interval(self, time: Duration) -> Self` | Sets the gap between keep-alive records. Default 15s. |
| `text` | `fn text<I: AsRef<str>>(self, text: I) -> Self` | Keep-alive payload as comment text. Panics on newline or carriage return. |
| `event` | `fn event(self, event: Event) -> Self` | Keep-alive payload as a full `Event` instead of a comment. Panics on newline or carriage return. |

`KeepAlive` is consumed by `Sse::keep_alive`.

## BoxError

```rust
pub type BoxError = Box<dyn std::error::Error + Send + Sync>;
```

`axum::BoxError` is the error type the SSE stream's `E` must convert into. Any
error type that implements `std::error::Error + Send + Sync` satisfies
`Into<BoxError>`. `std::convert::Infallible` also satisfies the bound, which is
why an infallible stream lifted with `.map(Ok)` works.

## Feature flags

| Feature | Default | Gates |
|---------|---------|-------|
| `tokio` | enabled | `Sse::keep_alive` and the `KeepAliveStream` wrapper |
| `json` | not enabled by default | `Event::json_data` |

To use `Event::json_data`, enable the `json` feature in `Cargo.toml`:

```toml
axum = { version = "0.8", features = ["json"] }
```

## Companion stream APIs

These types are NOT part of `axum::response::sse` but are needed to build SSE
streams. They come from `futures-util` and `tokio-stream`.

| Item | Crate | Use |
|------|-------|-----|
| `futures_util::stream::repeat_with` | `futures-util` | Build an infinite stream from a closure. |
| `futures_util::Stream` | `futures-util` | The stream trait named in handler return types. |
| `tokio_stream::StreamExt::map` | `tokio-stream` | `.map(Ok)` lift and item transforms. |
| `tokio_stream::StreamExt::throttle` | `tokio-stream` | Pace an instant stream by a `Duration`. |
| `tokio_stream::wrappers::BroadcastStream` | `tokio-stream` | Wrap a `broadcast::Receiver` as a `Stream`. |
| `tokio_stream::wrappers::ReceiverStream` | `tokio-stream` | Wrap an `mpsc::Receiver` as a `Stream`. |

`StreamExt` must be in scope with `use tokio_stream::StreamExt as _;` for
`.map` and `.throttle` to be callable.
