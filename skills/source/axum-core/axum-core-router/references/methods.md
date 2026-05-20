# axum-core-router : Methods Reference

Complete API surface for `Router<S>` and `MethodRouter`. Signatures are verbatim
from docs.rs (`axum` 0.8.x, verified 2026-05-20). Axum 0.7 shares these signatures
except where a 0.8-only note is given. Source URLs are listed at the end.

## Router type

`Router<S>` is the central routing type. `S` is the state type the router still
requires. A fully configured, serveable router has type `Router<()>`. `Router`
implements `Clone` and `tower::Service`.

## Router constructors and route registration

```rust
pub fn new() -> Self
```
Creates an empty router. An empty router answers `404 Not Found` to every request
until routes or a fallback are added.

```rust
pub fn route(self, path: &str, method_router: MethodRouter<S>) -> Self
```
Registers `path` to a `MethodRouter`. Panics:
- if `path` is empty;
- if the route overlaps with another already-registered route.
A static segment and a dynamic capture at the same position do NOT overlap; the
static route takes precedence. Two captures at the same position DO overlap.

```rust
pub fn route_service<T>(self, path: &str, service: T) -> Self
where T: Service<Request, Error = Infallible> + Clone + Send + Sync + 'static
```
Registers `path` to a raw `tower::Service`. Panics for the same reasons as
`route`, and additionally panics if you pass a `Router` to it. For a nested
`Router`, use `nest` instead.

## Sub-router composition

```rust
pub fn nest(self, path: &str, router: Router<S>) -> Self
```
Mounts `router` under the prefix `path`. The matched prefix is stripped from the
request URI before the child router sees it, so child routes are written relative
to the mount point. Panics:
- if `path` is empty;
- if `path` contains a wildcard (`*`);
- if the nested route overlaps with another route.

```rust
pub fn nest_service<T>(self, path: &str, service: T) -> Self
where T: Service<Request, Error = Infallible> + Clone + Send + Sync + 'static
```
The `tower::Service` variant of `nest`. Common use: mounting a `ServeDir`.

```rust
pub fn merge<R>(self, other: R) -> Self
where R: Into<Router<S>>
```
Flattens `other`'s routes into this router at the same level (no prefix added).
Both routers must share the state type `S`. Panics if two routers that each have
a fallback are merged, because a `Router` allows only a single fallback.

## Fallbacks

```rust
pub fn fallback<H, T>(self, handler: H) -> Self
where H: Handler<T, S>, T: 'static
```
Sets the handler invoked when NO route path matches at all (a true `404`). A path
that matched but with an unregistered HTTP method does NOT reach this handler.

```rust
pub fn fallback_service<T>(self, service: T) -> Self
where T: Service<Request, Error = Infallible> + Clone + Send + Sync + 'static
```
The `tower::Service` variant of `fallback`.

```rust
pub fn method_not_allowed_fallback<H, T>(self, handler: H) -> Self
where H: Handler<T, S>, T: 'static
```
Sets a router-wide `405 Method Not Allowed` handler. It applies to every
method-router registered so far, so the handler does not have to be repeated per
route.

```rust
pub fn reset_fallback(self) -> Self
```
Clears any fallback set on the router. Required on one of two routers that both
have a fallback before they can be merged.

## State

```rust
pub fn with_state<S2>(self, state: S) -> Router<S2>
```
Provides the state value. In practice this turns a `Router<S>` into a
`Router<()>`. `axum::serve` accepts only `Router<()>`. The compiler enforces that
a router needing state cannot be served without it. See the `axum-core-state`
skill for `State<S>` and `FromRef`.

## Middleware

```rust
pub fn layer<L>(self, layer: L) -> Router<S>
where L: Layer<Route> + Clone + Send + Sync + 'static
```
Applies a `tower::Layer` to ALL routes registered so far. Routes added after the
`layer` call are NOT wrapped. Middleware added this way runs after routing, and
also runs for requests that do not match any route. Stacked `layer` calls nest
bottom-to-top: the last `layer` is the outermost.

```rust
pub fn route_layer<L>(self, layer: L) -> Self
where L: Layer<Route> + Clone + Send + Sync + 'static
```
Applies a `tower::Layer` that runs ONLY when a request matches a route. Intended
for middleware that returns early, such as authorization. Because it never runs on
an unmatched path, it cannot turn a genuine `404` into a `401`. Panics if no
routes have been added to the router yet.

## Version compatibility

```rust
pub fn without_v07_checks(self) -> Self
```
Disables the axum 0.8 panic that rejects 0.7-style path strings (paths beginning
with `:` or `*`). With this enabled, such segments are treated as literal. Use it
ONLY when a literal `:` or `*` path segment is genuinely required, NEVER as a
shortcut to avoid migrating path syntax.

## Path parameter syntax

| Capture kind | axum 0.7 | axum 0.8 |
|--------------|----------|----------|
| Single segment | `/users/:id` | `/users/{id}` |
| Catch-all wildcard | `/files/*rest` | `/files/{*rest}` |

axum 0.8 uses `matchit` 0.8. A 0.7-style path on a 0.8 router panics at router
construction. A literal `{` or `}` in a 0.8 path is escaped by doubling: `{{` and
`}}`. A catch-all `/{*key}` does NOT match the empty segment.

## MethodRouter

`MethodRouter<S>` is the per-path type that fans out by HTTP method. It is the
second argument to `Router::route`.

### Free functions in `axum::routing`

Handler-taking constructors, each returning a `MethodRouter`:

`get`, `post`, `put`, `delete`, `patch`, `head`, `options`, `connect`, `trace`,
`any`, `on`.

- `any(handler)` matches every HTTP method.
- `on(filter: MethodFilter, handler)` matches the methods selected by `filter`.

Each constructor has a `*_service` sibling that accepts a `tower::Service` instead
of a handler: `get_service`, `post_service`, `put_service`, `delete_service`,
`patch_service`, `head_service`, `options_service`, `connect_service`,
`trace_service`, `any_service`, `on_service`.

### Chaining methods on a MethodRouter

```rust
pub fn get<H, T>(self, handler: H) -> Self
// identical shape for: post, put, delete, patch, head, options
pub fn on<H, T>(self, filter: MethodFilter, handler: H) -> Self
```
The same names that exist as free functions also exist as chaining methods on a
`MethodRouter`, so several methods register on one path:
`get(list).post(create)`. The `*_service` chaining variants exist as well.

### MethodRouter behavior

- A freshly constructed `MethodRouter` responds `405 Method Not Allowed` to every
  request by default.
- A `MethodRouter` sets the `Allow` header when returning `405`.
- A `get` route also answers `HEAD` requests, with the response body removed.

```rust
pub fn fallback<H, T>(self, handler: H) -> Self
```
`MethodRouter::fallback` customizes the `405` response for that one path only.
For a router-wide `405` handler use `Router::method_not_allowed_fallback`.

`MethodRouter` also exposes `layer`, `route_layer`, `merge`, and `with_state`,
mirroring the `Router` methods of the same name.

## Documented panics summary

| Call | Panics when |
|------|-------------|
| `route` | `path` empty, or route overlaps another route |
| `route_service` | same as `route`, or a `Router` is passed as the service |
| `nest` | `path` empty, `path` contains `*`, or route overlaps |
| `merge` | both routers have a fallback |
| `route_layer` | no routes added yet |
| `route` / `nest` (0.8) | a 0.7-style `:`/`*` path string is used |

All routing panics occur at router construction (startup), never at request time.

## Sources verified (fetched 2026-05-20)

- https://docs.rs/axum/latest/axum/routing/struct.Router.html : `Router` method
  signatures, path syntax, route precedence, overlap and empty-path panics, merge
  double-fallback panic, nest wildcard panic, `layer` versus `route_layer`.
- https://docs.rs/axum/latest/axum/routing/method_routing/struct.MethodRouter.html
  : `MethodRouter` chaining methods, `fallback`, default `405` behavior, `Allow`
  header, the `get`-also-answers-`HEAD` note.
- https://docs.rs/axum/latest/axum/routing/index.html : free-function inventory
  (`get`/`post`/.../`any`/`on` plus `*_service` variants), exported types.
- axum 0.8.0 CHANGELOG breaking-changes section : `matchit` 0.8 path-syntax change.
