# Methods : axum-syntax-custom-extractors

Complete API signatures for implementing custom extractors. Every definition
below was WebFetch-verified against docs.rs on 2026-05-20. Sources are listed
at the end of this file.

## The two extractor traits

### FromRequestParts (no body)

```rust
// axum 0.8 - verbatim from docs.rs, native async fn in trait form
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;

    fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

- `S` is the application state type. Most hand-written impls are generic over
  `S` with the bound `S: Send + Sync`.
- `Parts` is `http::request::Parts`. Import it as
  `axum::http::request::Parts`. It carries the method, URI, HTTP version,
  headers, and extensions, but NOT the body.
- `parts` is `&mut` because some built-in extractors take their value out of
  `parts.extensions`. Reading headers needs no mutation, but the signature
  requires the `&mut`.
- `state` is `&S`, an immutable borrow.
- The return type is a bare `impl Future<...> + Send`. There is no
  `#[async_trait]`. Implement the method as a plain `async fn`.

Docs.rs guidance, verbatim: "Extractors that implement `FromRequestParts`
cannot consume the request body and can thus be run in any order for
handlers." And: "If your extractor needs to consume the request body then you
should implement `FromRequest` and not `FromRequestParts`."

### FromRequest (consumes the body)

```rust
// axum 0.8 - verbatim from docs.rs
pub trait FromRequest<S, M = ViaRequest>: Sized {
    type Rejection: IntoResponse;

    fn from_request(
        req: Request<Body>,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

- `M = ViaRequest` is a default generic marker parameter. A hand-written impl
  writes `FromRequest<S>` and never names `M`; the marker exists only so the
  blanket impl below can coexist.
- `Request<Body>` is `http::Request<axum::body::Body>`. The body type is
  always `axum::body::Body`; Axum 0.8 removed the generic `B` body parameter.
  `axum::extract::Request` is the alias used in most code.
- `req` is consumed by value because reading the body takes ownership of the
  single-use async stream.

Docs.rs guidance, verbatim: "Extractors that implement `FromRequest` can
consume the request body and can thus only be run once for handlers. If your
extractor doesn't need to consume the request body then you should implement
`FromRequestParts` and not `FromRequest`."

### The blanket impl

```rust
// axum 0.8 - verbatim from docs.rs
impl<S, T> FromRequest<S, ViaParts> for T
where
    S: Send + Sync,
    T: FromRequestParts<S>,
{
    type Rejection = <T as FromRequestParts<S>>::Rejection;
    // ...
}
```

This impl makes every `FromRequestParts` type automatically a `FromRequest`
type through the `ViaParts` marker. Consequence: a handler whose arguments are
all `FromRequestParts` extractors compiles, because each parts extractor also
satisfies `FromRequest` via `ViaParts`. The "last argument must be
`FromRequest`" rule therefore only fails when a genuine body extractor is not
last.

## The Rejection associated type

- Both traits declare `type Rejection: IntoResponse`.
- The rejection is the error returned when extraction fails. Because it
  implements `IntoResponse`, Axum turns a failed extraction directly into an
  HTTP response without reaching the handler.
- Values that satisfy the bound include `StatusCode`, `(StatusCode, T)` where
  `T: IntoResponse`, `(StatusCode, Json<serde_json::Value>)`, `Response`, and
  any of the built-in `*Rejection` types in `axum::extract::rejection`.
- `Infallible` is the rejection for extractors that cannot fail.

## The async mechanism : 0.7 versus 0.8

| Version | Mechanism | Impl block attribute | Cargo dependency |
|---------|-----------|----------------------|------------------|
| 0.7 | `#[async_trait]` macro | `#[async_trait]` REQUIRED | `async-trait` REQUIRED |
| 0.8 | native `async fn` in trait (RPITIT) | NONE | none (remove `async-trait`) |

Axum 0.8 MSRV is Rust 1.75, the release that stabilized return-position
`impl Trait` in traits. The `async fn` body is byte-for-byte identical between
versions; only the attribute and the dependency change.

## JsonRejection accessors

`JsonRejection` lives in `axum::extract::rejection`. It is `#[non_exhaustive]`.
Used when a body extractor wraps `Json` and remaps the inner rejection.

| Method | Returns | Use |
|--------|---------|-----|
| `.status()` | `StatusCode` | the HTTP status the rejection would produce |
| `.body_text()` | `String` | a human-readable error message |

Variants: `MissingJsonContentType`, `JsonDataError`, `JsonSyntaxError`,
`BytesRejection`. Because the type is `#[non_exhaustive]`, any `match` on the
variants needs a catch-all arm.

The same `.status()` / `.body_text()` accessor pair exists on the other
rejection types in the module (`PathRejection`, `QueryRejection`,
`FormRejection`, `BytesRejection`, `StringRejection`, `ExtensionRejection`).

## FromRef : reading a substate

```rust
// axum 0.8 / 0.7 - signature identical
pub trait FromRef<T> {
    fn from_ref(input: &T) -> Self;
}
```

To make a `FromRequestParts` extractor depend on one field of a composite
state, bound the impl `where SomeResource: FromRef<S>, S: Send + Sync`, then
call `SomeResource::from_ref(state)` inside `from_request_parts`. See
`axum-core-state` for the substate model and `#[derive(FromRef)]`.

## axum-macros derive attributes

`axum-macros` provides `#[derive(FromRequest)]` and
`#[derive(FromRequestParts)]` (also re-exported as `axum::FromRequest` /
`axum::FromRequestParts` when Axum's `macros` feature is enabled). Attribute
syntax is identical on Axum 0.7 and 0.8.

| Attribute | Placement | Effect |
|-----------|-----------|--------|
| `#[from_request(via(SomeExtractor))]` | field or container | extract that field, or the whole type, through `SomeExtractor` |
| `#[from_request(rejection(MyRejection))]` | container | use `MyRejection` as the one unified `Rejection` type |
| `#[from_request(state(MyState))]` | container | pin the concrete state type instead of generic `S` |

Verified derive rules from docs.rs:

- Default behavior: "By default `#[derive(FromRequest)]` will call
  `FromRequest::from_request` for each field." A field with no attribute must
  itself already be an extractor type.
- Body limit: "only the last field can consume the request body" when using
  `#[derive(FromRequest)]`. `#[derive(FromRequestParts)]` cannot consume the
  body at all.
- Custom rejection: `MyRejection` must implement `IntoResponse`, and must
  implement `From<R>` for the rejection `R` of each `via` field, so Axum knows
  how to convert each field rejection into `MyRejection`.
- `via` requirement: the `via` extractor must be a generic newtype struct (a
  tuple struct with exactly one public field) that implements the trait.

## The both-traits rule

NEVER implement both `FromRequest` and `FromRequestParts` for one concrete
type. The blanket impl resolves `FromRequest` for a parts extractor through
the `ViaParts` marker; a hand-written `FromRequest<S>` resolves through the
default `ViaRequest` marker. Two reachable impls through two markers make the
marker `M` impossible to infer, so the extractor is ambiguous and the
`Handler` trait bound fails. The only valid exception is a generic wrapper extractor
(for example `axum-extra`'s `WithRejection<E, R>`), where the two impls cover
different inner extractors and never collide for one concrete type.

## Sources verified

All URLs fetched and verified via WebFetch on 2026-05-20.

- https://docs.rs/axum/latest/axum/extract/trait.FromRequestParts.html
- https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html
- https://docs.rs/axum-macros/latest/axum_macros/
- https://docs.rs/axum-macros/latest/axum_macros/derive.FromRequest.html
- https://docs.rs/axum-macros/latest/axum_macros/derive.FromRequestParts.html
