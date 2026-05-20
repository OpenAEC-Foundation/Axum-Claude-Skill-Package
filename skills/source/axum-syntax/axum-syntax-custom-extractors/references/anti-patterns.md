# Anti-Patterns : axum-syntax-custom-extractors

Real mistakes when writing custom extractors, each with the root cause and the
fix. Every signature referenced was WebFetch-verified against docs.rs on
2026-05-20.

## AP-1 : #[async_trait] left on an Axum 0.8 impl

```rust
// axum 0.8 - WRONG: #[async_trait] on a 0.8 extractor impl
use async_trait::async_trait;

#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for MyExtractor {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S)
        -> Result<Self, Self::Rejection> { /* ... */ }
}
```

WHY THIS FAILS: Axum 0.8 converted `FromRequest` and `FromRequestParts` to
native `async fn` in traits (RPITIT, Rust 1.75). The trait method now returns
a bare `impl Future + Send`. The `#[async_trait]` macro rewrites the method to
return `Pin<Box<dyn Future>>`, which no longer matches the trait signature, so
the impl does not satisfy the trait. The error is a signature mismatch on
`from_request_parts`.

FIX: Delete the `#[async_trait]` attribute and remove the `async-trait`
dependency from `Cargo.toml`. The plain `async fn` body is unchanged.

```rust
// axum 0.8 - CORRECT: no attribute, native async fn in trait
impl<S: Send + Sync> FromRequestParts<S> for MyExtractor {
    type Rejection = (StatusCode, &'static str);
    async fn from_request_parts(parts: &mut Parts, _: &S)
        -> Result<Self, Self::Rejection> { /* ... */ }
}
```

The mirror mistake is removing `#[async_trait]` on Axum 0.7, where it IS
required. On 0.7 the macro is mandatory; on 0.8 it is forbidden.

## AP-2 : implementing both FromRequest and FromRequestParts for one type

```rust
// axum 0.7 / 0.8 - WRONG: both traits on the same concrete type
impl<S: Send + Sync> FromRequestParts<S> for AuthToken {
    type Rejection = StatusCode;
    async fn from_request_parts(parts: &mut Parts, _: &S)
        -> Result<Self, Self::Rejection> { /* ... */ }
}

impl<S: Send + Sync> FromRequest<S> for AuthToken {     // second impl
    type Rejection = StatusCode;
    async fn from_request(req: Request, _: &S)
        -> Result<Self, Self::Rejection> { /* ... */ }
}
```

WHY THIS FAILS: the blanket impl
`impl<S, T> FromRequest<S, ViaParts> for T where T: FromRequestParts<S>`
already gives `AuthToken` a `FromRequest` impl through the `ViaParts` marker.
The hand-written `FromRequest<S>` resolves through the default `ViaRequest`
marker. `AuthToken` now has two `FromRequest` impls reachable through two
different markers. The compiler cannot infer which marker `M` a handler
argument needs, so the extractor is ambiguous: any handler taking `AuthToken`
fails the `Handler` trait bound.

FIX: pick exactly ONE trait for a concrete extractor. If `AuthToken` only
reads headers, keep `FromRequestParts` and delete the `FromRequest` impl. The
parts extractor still works as the last handler argument through the blanket
impl.

The only valid exception is a GENERIC wrapper extractor, for example
`axum-extra`'s `WithRejection<E, R>`, where the two impls cover different
inner extractors `E` and never collide for one concrete type.

## AP-3 : a body extractor placed before another argument

```rust
// axum 0.7 / 0.8 - WRONG: body extractor is not the last argument
async fn handler(body: ValidatedJson<NewUser>, ua: ExtractUserAgent) {}
```

WHY THIS FAILS: `ValidatedJson` implements `FromRequest`, so it consumes the
body. The request body is a single-use async stream. Axum enforces that the
last handler argument is the only one allowed to be a genuine `FromRequest`
extractor; every earlier argument must be `FromRequestParts`. A `FromRequest`
extractor in a non-last position fails the `Handler` trait bound with the
opaque `the trait bound ... Handler<_, _> is not satisfied` error.

FIX: move the body extractor to the last position.

```rust
// axum 0.7 / 0.8 - CORRECT: parts extractor first, body extractor last
async fn handler(ua: ExtractUserAgent, body: ValidatedJson<NewUser>) {}
```

Apply `#[debug_handler]` from `axum-macros` to the handler for a precise
diagnostic instead of the opaque trait-bound error.

## AP-4 : a Rejection type that does not implement IntoResponse

```rust
// axum 0.7 / 0.8 - WRONG: Rejection is a plain error with no IntoResponse
struct MyError(String);   // no IntoResponse impl

impl<S: Send + Sync> FromRequestParts<S> for MyExtractor {
    type Rejection = MyError;     // does not satisfy `Rejection: IntoResponse`
    async fn from_request_parts(_: &mut Parts, _: &S)
        -> Result<Self, Self::Rejection> { /* ... */ }
}
```

WHY THIS FAILS: both traits declare `type Rejection: IntoResponse`. The
rejection is the value Axum turns directly into an HTTP response when
extraction fails. A type with no `IntoResponse` impl breaks the associated
type bound and the impl does not compile.

FIX: implement `IntoResponse` for the rejection type, or use a type that
already has the impl, such as `(StatusCode, &'static str)`,
`(StatusCode, Json<serde_json::Value>)`, `StatusCode`, or `Response`.

```rust
// axum 0.7 / 0.8 - CORRECT
impl axum::response::IntoResponse for MyError {
    fn into_response(self) -> axum::response::Response {
        (axum::http::StatusCode::BAD_REQUEST, self.0).into_response()
    }
}
```

## AP-5 : reading the body stream by hand instead of delegating

```rust
// axum 0.8 - WRONG: hand-rolling body parsing in a FromRequest impl
impl<S: Send + Sync> FromRequest<S> for MyJson {
    type Rejection = StatusCode;
    async fn from_request(req: Request, _: &S)
        -> Result<Self, Self::Rejection>
    {
        let bytes = axum::body::to_bytes(req.into_body(), usize::MAX)
            .await
            .map_err(|_| StatusCode::BAD_REQUEST)?;
        let value = serde_json::from_slice(&bytes)
            .map_err(|_| StatusCode::BAD_REQUEST)?;
        Ok(MyJson(value))
    }
}
```

WHY THIS FAILS: `usize::MAX` removes the body-size ceiling, so a large request
can exhaust memory (a denial-of-service vector). It also reimplements
content-type checking, charset handling, and error classification that `Json`
already does correctly, and it loses the structured `JsonRejection` variants.

FIX: delegate body consumption to the existing `Json` extractor and remap its
`JsonRejection`. `Json` respects `DefaultBodyLimit` and produces the correct
rejection variants.

```rust
// axum 0.8 - CORRECT: delegate to Json, remap the rejection
impl<S, T> FromRequest<S> for MyJson<T>
where
    S: Send + Sync,
    T: serde::de::DeserializeOwned,
    axum::Json<T>: FromRequest<S, Rejection = axum::extract::rejection::JsonRejection>,
{
    type Rejection = (StatusCode, String);
    async fn from_request(req: Request, state: &S)
        -> Result<Self, Self::Rejection>
    {
        let axum::Json(value) = axum::Json::<T>::from_request(req, state)
            .await
            .map_err(|r| (r.status(), r.body_text()))?;
        Ok(MyJson(value))
    }
}
```

## AP-6 : a custom derive rejection missing a From impl

```rust
// axum 0.7 / 0.8 - WRONG: ApiError has no From<JsonRejection>
#[derive(FromRequest)]
#[from_request(rejection(ApiError))]
struct CreateUser {
    #[from_request(via(Json))]
    payload: NewUserPayload,
}

struct ApiError(StatusCode, String);
impl IntoResponse for ApiError { /* ... */ }
// MISSING: impl From<JsonRejection> for ApiError
```

WHY THIS FAILS: `#[from_request(rejection(ApiError))]` tells the derive macro
to use `ApiError` as the one rejection type. The macro converts each field's
own rejection into `ApiError` through a `From` impl. The `payload` field is
extracted `via(Json)`, whose rejection is `JsonRejection`. Without
`From<JsonRejection> for ApiError`, the generated code has no conversion path
and fails to compile.

FIX: implement `From<R>` for every `via`-field rejection `R`, and implement
`IntoResponse` for the rejection type.

```rust
// axum 0.7 / 0.8 - CORRECT
impl From<axum::extract::rejection::JsonRejection> for ApiError {
    fn from(rej: axum::extract::rejection::JsonRejection) -> Self {
        ApiError(rej.status(), rej.body_text())
    }
}
```

## AP-7 : using #[derive(FromRequestParts)] when a field consumes the body

```rust
// axum 0.7 / 0.8 - WRONG: a parts-derive struct with a body field
#[derive(FromRequestParts)]
struct Bundle {
    headers: HeaderMap,
    #[from_request(via(Json))]      // Json consumes the body
    payload: NewUser,
}
```

WHY THIS FAILS: `#[derive(FromRequestParts)]` generates a `FromRequestParts`
impl, and `FromRequestParts` cannot consume the request body. `Json` is a body
extractor. The `via(Json)` field needs body access the parts derive cannot
provide, so the code does not compile.

FIX: use `#[derive(FromRequest)]` for any struct that has a body-consuming
field. The body field must be the last field.

```rust
// axum 0.7 / 0.8 - CORRECT: FromRequest derive, body field last
#[derive(FromRequest)]
struct Bundle {
    headers: HeaderMap,
    #[from_request(via(Json))]
    payload: NewUser,
}
```
