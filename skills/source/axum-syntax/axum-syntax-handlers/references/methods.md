# axum-syntax-handlers: methods

Complete API signatures for the handler surface. Verified via WebFetch
against `docs.rs/axum/latest/axum/handler/` on 2026-05-20. The handler API
is identical on Axum 0.7 and 0.8.

## The Handler trait

```rust
// axum 0.7 / 0.8 - docs.rs/axum/latest/axum/handler/trait.Handler.html
pub trait Handler<T, S>: Clone + Send + Sync + Sized + 'static {
    type Future: Future<Output = Response> + Send + 'static;

    // required method
    fn call(self, req: Request, state: S) -> Self::Future;

    // provided method: wrap the handler in tower middleware
    fn layer<L>(self, layer: L) -> Layered<L, Self, T, S>
    where
        L: Layer<HandlerService<Self, T, S>> + Clone,
        L::Service: Service<Request>;

    // provided method: turn the handler into a Service by supplying state
    fn with_state(self, state: S) -> HandlerService<Self, T, S>;
}
```

### Type parameters

| Parameter | Meaning |
|-----------|---------|
| `T` | A coherence-rule workaround. Carries the handler's argument types as a discriminator so the blanket impls for different arities do not collide. Always inferred, never named by the author. |
| `S` | The application state type the handler reads through `State<S>` and other `FromRequestParts<S>` extractors. |

### Supertraits

`Clone + Send + Sync + Sized + 'static`. A handler value must be cheaply
cloneable (Axum clones it per request) and safe to move across threads.

### Associated type

```rust
type Future: Future<Output = Response> + Send + 'static;
```

The future a handler invocation returns. It MUST be `Send` and `'static` and
resolve to a `Response`. A handler body that produces a `!Send` future does
not satisfy this and is not a handler.

### Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `call` | `fn call(self, req: Request, state: S) -> Self::Future` | Invoke the handler with a request and state. Required method. |
| `layer` | `fn layer<L>(self, layer: L) -> Layered<L, Self, T, S>` | Wrap the handler in one `tower::Layer`. Returns a `Layered` value that is itself a handler. |
| `with_state` | `fn with_state(self, state: S) -> HandlerService<Self, T, S>` | Provide the state and obtain a `tower::Service`. |

## The blanket implementation

Axum implements `Handler<T, S>` for plain functions and closures through a
blanket impl, generated for tuples of 0 to 16 argument types. The bounds:

```text
// axum 0.7 / 0.8 - blanket-impl bounds, verified
F:   FnOnce(T1, ..., Tn) -> Fut + Clone + Send + Sync + 'static
Fut: Future<Output = Res> + Send
Res: IntoResponse
S:   Send + Sync + 'static

For each argument Ti:
  every Ti except the last:  FromRequestParts<S> + Send
  the last argument Tn:      FromRequest<S, M> + Send
A 0-argument function satisfies the parts/body rules trivially.
```

The six eligibility criteria that follow from these bounds:

1. The function is `async fn` (it returns a `Future`).
2. At most 16 arguments, every argument `Send`.
3. Every argument except the last implements `FromRequestParts<S>`.
4. The last argument implements `FromRequest<S, M>`.
5. The return type `Res` implements `IntoResponse`.
6. The produced future `Fut` is `Send`.

For a closure handler, criterion 7: the closure implements `Clone + Send`
and is `'static`.

A blanket impl also makes every `FromRequestParts` type usable where a
`FromRequest` type is expected, so an all-parts handler still compiles. The
body-extractor-last rule only bites when an actual body extractor is present
and placed before another argument.

## HandlerWithoutStateExt

An extension trait implemented for handlers whose state type is `()`. It
serves a single stateless handler without a `Router`.

```rust
// axum 0.7 / 0.8 - docs.rs/axum/latest/axum/handler/trait.HandlerWithoutStateExt.html
pub trait HandlerWithoutStateExt<T>: Handler<T, ()> {
    // turn the handler into a MakeService for axum::serve
    fn into_make_service(self) -> IntoMakeService<HandlerService<Self, T, ()>>;

    // same, but the handler can also extract ConnectInfo<C>
    fn into_make_service_with_connect_info<C>(
        self,
    ) -> IntoMakeServiceWithConnectInfo<HandlerService<Self, T, ()>, C>;
}
```

| Method | Use |
|--------|-----|
| `into_make_service` | Serve one handler as the entire service. It answers every request, all paths and methods. |
| `into_make_service_with_connect_info::<C>` | Same, when the handler needs the peer address through a `ConnectInfo<C>` extractor. |

A handler that needs application state cannot use `HandlerWithoutStateExt`;
use a `Router` with `.with_state(...)` instead.

## Supporting types

| Type | Role |
|------|------|
| `HandlerService<H, T, S>` | A `tower::Service` adapter wrapping a handler plus its state. Produced by `Handler::with_state`. |
| `Layered<L, H, T, S>` | A handler wrapped in a `tower::Layer`. Produced by `Handler::layer`. Itself implements `Handler`. |
| `IntoMakeService<S>` | A `MakeService` produced by `into_make_service`, accepted by `axum::serve`. |

## Related diagnostic API

```rust
// axum 0.7 / 0.8 - axum-macros, re-exported as axum::debug_handler
#[axum::debug_handler]
async fn my_handler(/* ... */) { /* ... */ }
```

`#[debug_handler]` is an attribute macro from `axum-macros`. Applied to a
handler, it generates code that turns the opaque
`the trait bound ... Handler<_, _> is not satisfied` error into a precise
message pointing at the offending argument or return type. It is a
development-only aid and a no-op concern on release builds. Full coverage of
its diagnostic workflow is in the `axum-errors-handler-trait` skill.

## Sources

Verified via WebFetch on 2026-05-20:

- https://docs.rs/axum/latest/axum/handler/index.html
- https://docs.rs/axum/latest/axum/handler/trait.Handler.html
- https://docs.rs/axum/latest/axum/handler/trait.HandlerWithoutStateExt.html
