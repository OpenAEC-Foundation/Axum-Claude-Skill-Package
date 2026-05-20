# Methods Reference: axum-errors-handler-trait

All signatures verified against docs.rs on 2026-05-20. Sources are listed at the
end of this file.

## The `Handler<T, S>` trait

`axum::handler::Handler<T, S>` is the trait that makes a function eligible as a
route handler. It is defined in the `axum` crate.

### Supertraits

A type implementing `Handler<T, S>` MUST also be:

```
Clone + Send + Sync + Sized + 'static
```

### Associated type

```rust
type Future: Future<Output = Response> + Send + 'static;
```

The handler's future MUST produce a `Response` and MUST itself be `Send` and
`'static`.

### Methods

```rust
// Call the handler with the request and the state.
fn call(self, req: Request, state: S) -> Self::Future;

// Bind the state to the handler, producing a Service.
fn with_state(self, state: S) -> HandlerService<Self, T, S>;

// Apply a tower layer to the handler.
fn layer<L>(self, layer: L) -> Layered<L, Self, T, S>
where
    L: Layer<HandlerService<Self, T, S>> + Clone,
    L::Service: Service<Request>;
```

`call` and `layer` are the methods most code touches indirectly; route
registration calls them for you. `with_state` is used to turn a single handler
into a service without a `Router`.

The `Handler<T, S>` trait, its supertraits, the associated `Future` type, and
these method signatures are identical in Axum 0.7 and Axum 0.8.

## The blanket implementation

Axum provides ONE blanket `impl` of `Handler` for functions. A function or
closure implements `Handler<T, S>` when ALL of these hold (verbatim criteria from
the trait docs):

1. The function is declared `async fn`. A plain `fn` that returns a `Future`
   does NOT qualify.
2. It takes no more than 16 arguments, and every argument implements `Send`.
3. All arguments except the last implement `FromRequestParts<S>`.
4. The last argument implements `FromRequest<S, M>`.
5. The return type implements `IntoResponse`.
6. For closures, the closure is `Clone + Send` and `'static`.
7. The future the function returns is `Send`.

Failing ANY single criterion makes the blanket impl inapplicable. The compiler
then reports the entire `impl` as unsatisfied via `error[E0277]` and cannot
indicate which criterion failed. This is the source of the opaque
`the trait bound ...{handler}: Handler<_, _> is not satisfied` error.

There is no blanket impl for tuples of 17 or more arguments, so a 17-argument
handler fails criterion 2 outright.

### The opaque error shape

```
error[E0277]: the trait bound `fn(bool) -> impl Future<Output = ()> {handler}:
              Handler<_, _>` is not satisfied
   --> src/main.rs:8:25
    |
  8 |     Router::new().route("/", get(handler))
    |                              --- ^^^^^^^ the trait `Handler<_, _>` is not
    |                              |           implemented for fn item `...`
    |                              required by a bound introduced by this call
    |
    = help: the following other types implement trait `Handler<T, S>`:
              `MethodRouter<S>` implements `Handler<(), S>`
              `axum::handler::Layered<L, H, T, S>` implements `Handler<T, S>`
note: required by a bound in `axum::routing::get`
```

The `help` block lists unrelated `Handler` impls; it is NOT a hint about the
fault. The placeholders `Handler<_, _>` are inference variables, not concrete
types.

## The `#[debug_handler]` macro

### Identity and location

`#[debug_handler]` is an attribute macro. It is defined in the `axum-macros`
crate as `axum_macros::debug_handler` (verified at axum-macros 0.5.1) and is
re-exported from the main `axum` crate as `axum::debug_handler`.

### The `macros` feature

`axum::debug_handler` is "Available on crate feature `macros` only" (verbatim).
The `macros` feature MUST be enabled on the `axum` dependency:

```toml
# axum 0.8
axum = { version = "0.8", features = ["macros"] }

# axum 0.7
axum = { version = "0.7", features = ["macros"] }
```

Alternatively, depend on `axum-macros` directly and import
`axum_macros::debug_handler`.

### Purpose

`#[debug_handler]` "Generates better error messages when applied to handler
functions" (verbatim). It transforms the opaque trait-bound error into a precise,
plain-language error that points at the exact argument or return type at fault.

### How it produces precise errors

Applied to a handler, the macro generates synthetic checking items, such as
`__axum_macros_check_handler_0_from_request_check`. Each criterion is asserted by
its own item, so the compiler can pin a failure to one argument or to the return
type rather than reporting the whole `route(...)` call.

### Verbatim macro error: not async

```
error: handlers must be async functions
  --> main.rs:xx:1
   |
xx | fn handler() -> &'static str {
   | ^^
```

For a non-extractor argument the macro reports that the argument does not
implement `FromRequestParts` / `FromRequest`. For a bad return type it reports
that the return type does not implement `IntoResponse`. For a `!Send` future it
reports the `Send` violation. Each diagnostic points at a precise span.

### State type

By default the macro assumes a state type of `()`. If the handler has a
`State<T>` argument, the macro infers `T` automatically. For handlers with
multiple or ambiguous state arguments, set the state explicitly:

```rust
#[debug_handler(state = AppState)]
async fn handler(State(state): State<AppState>) {}
```

### Release-profile behavior

Verbatim: "This macro has no effect when compiled with the release profile.
(eg. `cargo build --release`)". The macro adds zero cost to release builds, so
leaving it on a handler permanently is safe. For strict hygiene it may be gated:

```rust
#[cfg_attr(debug_assertions, axum::debug_handler)]
async fn handler() {}
```

This gating is optional, since the macro is already a release no-op.

### Limitation: impl-block methods without `self`

`#[debug_handler]` does NOT work on a method inside an `impl` block that lacks a
`self` parameter. It then emits:

```
error[E0425]: cannot find function
`__axum_macros_check_handler_0_from_request_check` in this scope
```

This `E0425` is a macro limitation, not a handler bug. The fix is to move the
logic to a free `async fn`, or to diagnose the handler without the macro.

The `#[debug_handler]` macro, its `state =` override, its release-profile
no-op behavior, and the `E0425` limitation are identical in Axum 0.7 and 0.8.

## Sources verified

Fetched and verified via WebFetch on 2026-05-20:

- `https://docs.rs/axum/latest/axum/handler/trait.Handler.html`: the
  `Handler<T, S>` trait, supertraits (`Clone + Send + Sync + Sized + 'static`),
  the associated `Future` type (`Future<Output = Response> + Send + 'static`),
  the `call` / `with_state` / `layer` method signatures, and the seven
  blanket-impl criteria.
- `https://docs.rs/axum-macros/latest/axum_macros/attr.debug_handler.html`: the
  `#[debug_handler]` purpose, the verbatim "handlers must be async functions"
  error, default `()` state with `State`-argument inference, the
  `#[debug_handler(state = ...)]` override, the impl-block-without-`self`
  limitation and its `E0425` error, and the verbatim release-profile sentence.
- `https://docs.rs/axum/latest/axum/attr.debug_handler.html`: confirms
  `debug_handler` is re-exported by the `axum` crate from `axum-macros` and is
  "Available on crate feature `macros` only".
