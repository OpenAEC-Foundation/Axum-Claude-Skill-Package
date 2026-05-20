# axum-core-state: Methods and Signatures

All signatures verified via WebFetch against docs.rs on 2026-05-20 (docs.rs
current at research time: axum 0.8.9). State management APIs are identical
across Axum 0.7 and 0.8.

Sources:

- https://docs.rs/axum/latest/axum/extract/struct.State.html
- https://docs.rs/axum/latest/axum/extract/trait.FromRef.html
- https://docs.rs/axum/latest/axum/struct.Router.html

## State<S>

```rust
// axum 0.7 / 0.8 - verbatim from docs.rs
pub struct State<S>(pub S);
```

`State<S>` is a tuple struct with one public field. It is a `FromRequestParts`
extractor: it reads no request body, so it may appear in any handler argument
position before a body extractor. Destructure it directly in the handler
signature: `State(state): State<AppState>`.

Documented purpose, verbatim: "State is global and used in every request a
router with state receives. For accessing data derived from requests, such as
authorization data, see Extension."

## Router::new

```rust
// axum 0.7 / 0.8 - verbatim
pub fn new() -> Self
```

Creates an empty `Router`. Its type parameter `S` is inferred from the
handlers added to it and the eventual `with_state` call.

## Router::with_state

```rust
// axum 0.7 / 0.8 - verbatim from docs.rs
pub fn with_state<S2>(self, state: S) -> Router<S2>
```

Consumes the router and the state value. Documented, verbatim: "Provide the
state for the router. State passed to this method is global and will be used
for all requests this router receives."

`Router<S>` means a router that is missing a state of type `S` to be able to
handle requests. After `with_state` the router is no longer missing the state,
so its type becomes `Router<()>`. Only `Router<()>` implements
`into_make_service` and can be passed to `axum::serve`.

## FromRef<T>

```rust
// axum 0.7 / 0.8 - verbatim from docs.rs
pub trait FromRef<T> {
    fn from_ref(input: &T) -> Self;
}
```

Documented purpose, verbatim: "Used to do reference-to-value conversions thus
not consuming the input value." Implement it so a handler can extract a
substate out of a larger parent state.

### The blanket impl

```rust
// axum 0.7 / 0.8 - documented blanket impl
impl<T> FromRef<T> for T where T: Clone
```

Every `Clone` type implements `FromRef` for itself. This is why a handler can
extract `State<AppState>` directly from a router holding `AppState`: the whole
state implements `FromRef` for itself.

## #[derive(FromRef)]

`#[derive(FromRef)]` comes from the `axum-macros` crate and is re-exported as
`axum::extract::FromRef`. Applied to a parent state struct, it generates one
`impl FromRef<Parent> for FieldType` per field, each cloning that field. Every
field type that a handler extracts as `State<FieldType>` MUST be `Clone`,
because `from_ref` returns the value by move.

The manual equivalent the derive expands to, per field:

```rust
// axum 0.7 / 0.8 - shape verbatim from the State docs
impl FromRef<AppState> for ApiState {
    fn from_ref(app_state: &AppState) -> ApiState {
        app_state.api_state.clone()
    }
}
```

## State requirements summary

| Requirement | Reason |
|-------------|--------|
| Application state MUST be `Clone` | Axum clones the state into every request |
| Heavy or large data SHOULD be wrapped in `Arc` | Keeps the per-request clone a refcount bump |
| Each `FromRef` substate field MUST be `Clone` | `from_ref` returns the field by value |
| The served router MUST be `Router<()>` | Only `Router<()>` has `into_make_service` |
