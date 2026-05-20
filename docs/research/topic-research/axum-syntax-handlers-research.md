# Topic Research : axum-syntax-handlers

> Skill : `axum-syntax-handlers`
> Category : syntax/
> Target versions : Axum 0.7 and 0.8 (docs.rs current : 0.8.x)
> Researched : 2026-05-20
> Status : verified against official docs.rs content

This research file scopes the `axum-syntax-handlers` skill. It covers the handler
definition, the `Handler<T, S>` trait and its blanket impl, valid return types,
the 16-extractor limit, async closures as handlers, `HandlerWithoutStateExt`, and a
brief pointer to `axum-errors-handler-trait` for diagnosing invalid handlers.

All API-shape claims below are verified via WebFetch against `docs.rs/axum` on
2026-05-20. Where the source page returned `Handler<T, B>` historic naming, the
current trait is `Handler<T, S>` as confirmed by the `trait.Handler.html` page.

---

## 1. The Handler Blanket Impl — Exact Criteria

A handler is, in the official wording, "an async function that accepts zero or
more extractors as arguments and returns something that can be converted into a
response." Eligibility is enforced entirely by trait bounds (Axum's no-macros
design), specifically the `Handler` trait.

The `Handler<T, S>` trait has a blanket implementation for every function (and
closure) that meets ALL of the following criteria. Verified verbatim from
`docs.rs/axum/latest/axum/handler/index.html`:

1. **Declared `async fn`.** A plain `fn` that returns a `Future` does NOT qualify.
2. **At most 16 arguments**, and "all [arguments] implement `Send`."
3. **All arguments except the last implement `FromRequestParts<S>`** (parts-only
   extractors — method, URI, headers, path params, state).
4. **The last argument implements `FromRequest<S, M>`** (the single body-consuming
   extractor). A handler with zero arguments trivially satisfies this.
5. **The return type implements `IntoResponse`.**
6. **The produced future is `Send`** ("Returns a future that is `Send`").
7. **For closures**, the closure "must implement `Clone + Send` and be `'static`."

### The Handler trait shape (verified)

```rust
// axum 0.8 - verified from docs.rs/axum/latest/axum/handler/trait.Handler.html
pub trait Handler<T, S>: Clone + Send + Sync + Sized + 'static {
    type Future: Future<Output = Response> + Send + 'static;

    fn call(self, req: Request, state: S) -> Self::Future;

    fn layer<L>(self, layer: L) -> Layered<L, Self, T, S>
    where
        L: Layer<HandlerService<Self, T, S>> + Clone,
        L::Service: Service<Request>;

    fn with_state(self, state: S) -> HandlerService<Self, T, S>;
}
```

**Type parameters.** `T` is, in the docs' own words, a "workaround for trait
coherence rules" — it carries the handler's argument types as a discriminator so
the blanket impl does not collide. `S` is the application state type. A handler
author never names `T` directly; it is inferred.

The blanket-impl generic bounds, verified:

```
F:   FnOnce(...) -> Fut + Clone + Send + Sync + 'static
Fut: Future<Output = Res> + Send
Res: IntoResponse
S:   Send + Sync + 'static
```

The trait is implemented for `async fn()` with 0 arguments and for `async fn(T1,
..., Tn)` with 1 through 16 arguments, where every `Ti` except the last is
`FromRequestParts<S> + Send`, the final argument is `FromRequest<S, M> + Send`,
and the return type `Res` is `IntoResponse`.

A blanket impl also makes every `FromRequestParts` type usable as a `FromRequest`
type, so an "all-parts" handler still compiles cleanly — the body-extractor-last
rule only bites when an actual body extractor is present and misplaced.

---

## 2. Valid Handler Examples (verified)

Any return type implementing `IntoResponse` is valid. The following six handlers
are all valid; the first three return types match examples shown directly on the
`handler` module page.

```rust
// axum 0.8 - all of these are valid handlers

// (1) Unit return -> empty 200 OK. Verified example: `async fn unit_handler() {}`.
async fn unit() {}

// (2) &'static str -> text/plain; charset=utf-8 body, 200 OK.
async fn text() -> &'static str { "hello" }

// (3) impl IntoResponse -> status + JSON body via a tuple.
async fn created() -> impl IntoResponse {
    (StatusCode::CREATED, Json(serde_json::json!({ "ok": true })))
}

// (4) Result<T, E> where both T and E implement IntoResponse.
//     Verified example shape: `async fn echo(body: Bytes) -> Result<String, StatusCode>`.
async fn echo(body: Bytes) -> Result<String, StatusCode> {
    String::from_utf8(body.to_vec()).map_err(|_| StatusCode::BAD_REQUEST)
}

// (5) Handler with State + a parts extractor + a body extractor (last).
async fn create_user(
    State(state): State<AppState>,   // FromRequestParts
    Path(team_id): Path<u64>,        // FromRequestParts
    Json(payload): Json<NewUser>,    // FromRequest - MUST be last
) -> Result<Json<User>, AppError> {
    let user = state.repo.insert(team_id, payload).await?;
    Ok(Json(user))
}

// (6) Custom error type with the `?` operator. AppError: IntoResponse.
async fn fallible(Json(p): Json<Payload>) -> Result<String, AppError> {
    let result = process(p)?;        // `?` converts the error via IntoResponse
    Ok(result)
}
```

**The `Result<T, E>` handler model.** A handler returning `Result<T, E>` does not
"fail" at the `tower::Service` level — Axum services have `Infallible` as their
error type. `Err(E)` is turned into a valid HTTP response because `E: IntoResponse`.
This is the foundation of `?`-based error handling: define an `AppError` type,
implement `IntoResponse` for it, and `?` converts source errors automatically (via
`From` impls). The depth of the error-type pattern belongs to the errors skills,
not this syntax skill.

**Return type summary.** Valid handler return types include `()`, `&str` /
`String`, `Vec<u8>` / `Bytes`, `StatusCode`, `(StatusCode, T)`, `Json<T>`,
`Html<T>`, `Redirect`, `impl IntoResponse`, and `Result<T, E>` — anything whose
type implements `IntoResponse`. The full `IntoResponse` catalog belongs to
`axum-syntax-responses`.

---

## 3. The 16-Extractor Limit (verified)

Verified verbatim from the `handler` module docs: a handler must "Take no more
than 16 arguments that all implement `Send`."

This is a hard limit imposed by the macro that generates the blanket impls — Axum
generates the `Handler` impl for tuples of 0 through 16 extractor arguments. A
function with 17 or more arguments simply has no matching blanket impl and fails
the `Handler` trait bound.

The practical workaround when a handler genuinely needs more than 16 pieces of
request data: group related extractors into a single custom struct that implements
`FromRequestParts` (or `FromRequest`), reducing the argument count. In practice
real handlers rarely approach the limit; hitting it is usually a sign the handler
should be decomposed or a custom extractor introduced.

---

## 4. Async Closures, `HandlerWithoutStateExt`, `into_make_service`

### Async closures as handlers

A closure is a valid handler when it satisfies the same blanket-impl criteria plus
the closure-specific bound: it "must implement `Clone + Send` and be `'static`."
The closure body must be `async` (it must produce a `Send` future):

```rust
// axum 0.8 - async closures as handlers
let app = Router::new()
    .route("/", get(|| async { "Hello, World!" }))
    .route("/echo", post(|body: String| async move { body }));
```

A closure that captures a non-`Send` value, or a non-`'static` borrow, fails the
bound exactly as a named function would.

### `HandlerWithoutStateExt` and `into_make_service`

`HandlerWithoutStateExt` is an extension trait implemented for handlers whose state
type is `()` (handlers that need no application state). It lets a single stateless
handler be served directly, without constructing a `Router`:

```rust
// axum 0.8 - serving a single handler with no Router
use axum::handler::HandlerWithoutStateExt;

#[tokio::main]
async fn main() {
    async fn handler() -> &'static str { "Hello from a bare handler" }

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    // into_make_service() turns the handler directly into a MakeService
    axum::serve(listener, handler.into_make_service()).await.unwrap();
}
```

`into_make_service()` (and the connect-info variant
`into_make_service_with_connect_info::<C>()`) turn the handler into a
`MakeService` suitable for `axum::serve`. The handler answers every request,
regardless of path or method. This is the handler-level analogue of
`Router::into_make_service()`; it is useful for the smallest possible service or
for a catch-all responder. The general `Handler` trait itself also provides
`with_state`, `layer`, and `call` (see Section 1) for adapting a handler into a
service or wrapping it in middleware.

---

## 5. What Makes a Function NOT a Valid Handler (brief)

When any blanket-impl criterion fails, the compiler emits a variant of `the trait
bound 'fn(...) -> ... {handler}: Handler<_, _>' is not satisfied`. The error does
NOT name which criterion failed — that is the well-known pain point of the
no-macros design. The real causes:

1. **An argument is not an extractor** — e.g. `async fn h(flag: bool)`; `bool`
   implements neither `FromRequest` nor `FromRequestParts`.
2. **A body extractor is not the last argument** — e.g. `async fn h(body: Bytes,
   p: Path<u64>)`; two body extractors trip the same rule (the body is single-use).
3. **The return type does not implement `IntoResponse`** — e.g. `async fn h() ->
   i32`; wrap it in `Json`, return a `String`, or use a `StatusCode` tuple.
4. **The function is not `async`** — a synchronous `fn` returning `impl Future` is
   not accepted.
5. **The future is `!Send`** — holding an `Rc`, a `RefCell` borrow, or a
   `std::sync::MutexGuard` across an `.await` makes the future `!Send`, so the
   blanket impl no longer applies.
6. **More than 16 arguments** — no blanket impl exists for 17+ argument tuples.

The diagnostic remedy is the `#[debug_handler]` attribute macro from
`axum-macros` (also re-exported as `axum::debug_handler`), which generates code
producing a precise, plain-language error pointing at the offending argument or
return type. **Full coverage of the "Handler not satisfied" error, its causes,
and the `#[debug_handler]` diagnostic workflow belongs to the dedicated
`axum-errors-handler-trait` skill.** This syntax skill should point there and not
duplicate the depth.

---

## 6. Sources Verified

All URLs below were fetched and verified via WebFetch on **2026-05-20**.

- https://docs.rs/axum/latest/axum/handler/index.html — handler definition ("an
  async function that accepts zero or more extractors..."), the 6-point blanket
  impl criteria, the verbatim 16-argument limit ("Take no more than 16 arguments
  that all implement `Send`"), valid handler examples (`async fn unit_handler()
  {}`, `async fn string_handler() -> String`, `async fn echo(body: Bytes) ->
  Result<String, StatusCode>`), closure bound (`Clone + Send` and `'static`), and
  the `#[debug_handler]` pointer.
- https://docs.rs/axum/latest/axum/handler/trait.Handler.html — verbatim
  `Handler<T, S>` trait signature, supertraits (`Clone + Send + Sync + Sized +
  'static`), associated `Future` type (`Future<Output = Response> + Send +
  'static`), the `call` / `with_state` / `layer` methods, the blanket-impl
  signatures for 0 to 16 arguments (all-but-last `FromRequestParts<S> + Send`,
  last `FromRequest<S, M> + Send`, `Res: IntoResponse`), and the meaning of `T`
  ("a workaround for trait coherence rules") and `S` (application state).

Supporting prior research (already verified, see `vooronderzoek-axum.md` and
`fragments/research-a-core-routing-extractors.md`, both verified 2026-05-20):
- `HandlerWithoutStateExt::into_make_service` is referenced in fragment A §6 as
  the extension trait for serving a stateless handler without a `Router`.
- The extractor ordering rule (one `FromRequest`, must be last; all preceding
  arguments `FromRequestParts`) is verified in fragment A §4 against
  `docs.rs/axum/latest/axum/extract/index.html`.

Verification note: the `handler/index.html` page text historically references
`Handler<T, B>`; the current canonical trait is `Handler<T, S>` as confirmed
directly on `trait.Handler.html`. Use `Handler<T, S>` in the skill.
