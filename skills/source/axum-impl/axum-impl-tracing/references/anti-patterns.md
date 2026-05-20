# axum-impl-tracing: Anti-Patterns

Each entry is a real mistake, why it fails, and the corrected form. Verified
2026-05-20 against `https://docs.rs/tracing/latest/tracing/`,
`https://docs.rs/tracing-subscriber/latest/tracing_subscriber/`, and the
`tokio-rs/axum` `examples/tracing-aka-logging` crate.

## 1. Holding a Span::enter guard across an await

### Wrong

```rust
async fn handler() {
    let span = tracing::info_span!("handler");
    let _guard = span.enter();
    do_async_work().await; // guard still alive while the task is suspended
}
```

### Why it fails

`Span::enter()` returns a drop guard that keeps the span entered on the current
thread until the guard is dropped. When the future yields at the `.await`, the
async runtime runs other tasks on that same thread while the guard is still
alive. Those unrelated tasks are recorded as running inside the span, so span
timings are inflated and the parent/child structure is wrong. The official
`tracing` documentation states this directly: `Span::enter` may produce
incorrect traces if the guard is held across an await point. The symptom is
traces where unrelated requests appear nested under one another.

### Correct

Use `#[tracing::instrument]` on the async fn, or `Future::instrument` on an
async block. Both enter the span only while the future is actively polled.

```rust
#[tracing::instrument]
async fn handler() {
    do_async_work().await;
}
```

`span.enter()` is correct ONLY in synchronous code with no `.await` in scope.

## 2. Calling .init() more than once

### Wrong

```rust
tracing_subscriber::fmt::init();          // installs a global default
tracing_subscriber::registry()
    .with(tracing_subscriber::fmt::layer())
    .init();                              // second install: fails
```

### Why it fails

`.init()` installs the composed subscriber as the process-wide global default.
The global default can be set only once for the lifetime of the process. A
second `.init()` returns an error or panics because the slot is already taken.
The symptom is a startup panic, or, with two competing setups, missing or
duplicated log lines.

### Correct

Initialize the subscriber exactly once, in `main`, before `axum::serve`. Pick
one setup path and delete the other.

## 3. Forgetting the SubscriberExt and SubscriberInitExt imports

### Wrong

```rust
// no extension-trait imports
tracing_subscriber::registry()
    .with(tracing_subscriber::fmt::layer()) // error: no method named `with`
    .init();                                // error: no method named `init`
```

### Why it fails

`.with()` and `.init()` are not inherent methods on the registry. `.with()`
comes from the `SubscriberExt` trait and `.init()` from the `SubscriberInitExt`
trait. A trait method is callable only when its trait is in scope. Without the
imports the compiler reports that no method named `with` or `init` exists.

### Correct

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
// or: use tracing_subscriber::prelude::*;
```

## 4. Spanning on the raw URI instead of MatchedPath

### Wrong

```rust
TraceLayer::new_for_http().make_span_with(|request: &Request<_>| {
    tracing::info_span!("http_request", uri = %request.uri())
})
```

### Why it fails

`request.uri()` is the concrete request path: `/users/42`, `/users/43`,
`/users/44`. Spanning on it produces a distinct span identity for every concrete
URL, so span and metric cardinality grows without bound and per-route
aggregation becomes impossible. A tracing backend either drops data or bills for
the explosion.

### Correct

Read `MatchedPath` from the request extensions. It is the route template
(`/users/{id}`), so all requests to that route share one bounded span identity.

```rust
TraceLayer::new_for_http().make_span_with(|request: &Request<_>| {
    let matched_path = request
        .extensions()
        .get::<MatchedPath>()
        .map(MatchedPath::as_str);
    tracing::info_span!("http_request", matched_path)
})
```

## 5. Recording a span field that was never declared

### Wrong

```rust
let span = tracing::info_span!("http_request"); // no `result` field declared
// ... later ...
span.record("result", "ok"); // silently does nothing
```

### Why it fails

A span's field set is fixed at creation. `span.record(name, value)` can only
fill a field that already exists on the span; a `record` call for an undeclared
field name is silently ignored. There is no compile error and no runtime error,
so the missing field is easy to overlook. The symptom is a field that never
appears in the output even though `record` is called.

### Correct

Declare the field at span creation with `tracing::field::Empty`, then fill it.

```rust
let span = tracing::info_span!("http_request", result = tracing::field::Empty);
// ... later ...
span.record("result", "ok"); // now the field is updated
```

## 6. Interpolating values into the log message

### Wrong

```rust
tracing::info!("user {} fetched in {}ms", user_id, elapsed);
```

### Why it fails

`tracing` is a structured logging system: each event carries typed `key = value`
fields that a backend can filter, group, and aggregate. Baking the values into
the message string with positional interpolation collapses them into opaque
text. A query for "all events where `user_id = 42`" can no longer match,
because there is no `user_id` field, only a sentence.

### Correct

Pass the values as fields and keep the message a constant description.

```rust
tracing::info!(user_id, elapsed_ms = elapsed, "user fetched");
```

Use the `%` sigil for values with a clean `Display` and the `?` sigil for values
with only `Debug`.
