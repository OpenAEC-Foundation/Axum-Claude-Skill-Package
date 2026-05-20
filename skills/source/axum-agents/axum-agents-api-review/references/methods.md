# axum-agents-api-review: Complete Checklist Reference

This file is the full review checklist. Where SKILL.md Pattern 1 lists the
headline checks per category, this file gives one row per check: what to look
for, the failure symptom, and the skill that owns the fix. Skill names are the
exact package inventory names.

This skill introduces no new API claims. Every check traces to a verified
anti-pattern in `docs/research/vooronderzoek-axum.md` section 9, owned by
another skill.

## 1. Routing

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Path syntax matches the target version | `:id` / `*rest` versus `{id}` / `{*rest}` in `.route()` strings | 0.7 syntax panics at router construction on 0.8 | `axum-core-router`, `axum-core-version-migration` |
| Routes registered before `.layer()` | A `.layer()` call sequenced before all `.route()` calls | Later routes are not wrapped by the middleware | `axum-core-router`, `axum-impl-middleware` |
| 405 distinguished from 404 | Only `.fallback()` present where a method-not-allowed path matters | Wrong-method requests return 404 instead of 405 | `axum-core-router` |
| No double-fallback panic after `.merge()` | Two merged routers each set `.fallback()` | Panic at construction; `reset_fallback` needed | `axum-core-router` |
| `NestedPath` used for nested-router prefixes | Manual prefix string-building inside nested routers | Fragile; breaks when the mount point changes | `axum-core-router` |

## 2. Extractors

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Body extractor is the LAST argument | `Json<T>`, `Form<T>`, `Multipart`, `Bytes`, `String`, `Request` not in final position | `Handler` trait bound fails to compile (the body is single-use) | `axum-syntax-extractors`, `axum-errors-handler-trait` |
| At most one body extractor per handler | Two `FromRequest` extractors in one signature | Compile failure | `axum-syntax-extractors` |
| `Option<Path<T>>` behavior is 0.8-aware | Code assuming `Option<Path<T>>` swallows all errors | 0.8 rejects instead of yielding `None` | `axum-syntax-extractors`, `axum-core-version-migration` |
| Custom extractors use the right base trait | `FromRequest` used where no body is consumed | Forces final-argument placement unnecessarily | `axum-syntax-custom-extractors` |
| Custom extractor `Rejection` implements `IntoResponse` | The `Rejection` associated type missing the impl | Compile failure | `axum-syntax-custom-extractors`, `axum-errors-extractor-rejections` |

## 3. Handlers

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Handler return type is `IntoResponse` | `async fn h() -> i32` or another non-response type | `Handler` trait bound not satisfied | `axum-syntax-handlers`, `axum-errors-handler-trait` |
| Handler is declared `async` | A sync `fn` passed to a method-router | `Handler` trait bound not satisfied | `axum-syntax-handlers`, `axum-errors-handler-trait` |
| Handler future is `Send` | `Rc`, `RefCell`, `std::sync::MutexGuard` held across `.await` | `!Send` future fails the bound; cryptic error | `axum-errors-handler-trait`, `axum-core-async-performance` |
| At most 16 extractor arguments | A handler with 17 or more arguments | The `Handler` blanket impl does not cover it | `axum-syntax-handlers` |
| `#[debug_handler]` available for diagnosis | A cryptic "Handler is not satisfied" error with no debug aid | Slow, guess-driven debugging | `axum-errors-handler-trait` |

## 4. Middleware

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Layer ordering is intentional | Stacked `Router::layer()` calls with no ordering awareness | Bottom-to-top order surprises the author; wrong wrap order | `axum-impl-middleware`, `axum-impl-tower-stack` |
| `ServiceBuilder` versus `.layer()` direction known | A `ServiceBuilder` stack assumed to share the `Router::layer()` direction | `ServiceBuilder` is top-to-bottom, opposite of stacked `.layer()` | `axum-impl-tower-stack` |
| Auth middleware uses `route_layer` | Auth applied with `.layer()` instead of `.route_layer()` | A 404 is turned into a 401, leaking route existence | `axum-impl-middleware`, `axum-impl-authorization` |
| `from_fn` argument order correct | `Next` not the final parameter; parts after `Request` | Compile failure in the `from_fn` middleware | `axum-impl-middleware` |

## 5. State

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| `State` preferred over `Extension` for app data | `Extension<T>` for application-wide config or pools | Runtime 500 if the extension is missing; no compile check | `axum-core-state` |
| State is `Clone` and cheap to clone | Heavy, non-`Arc` data in state | Expensive per-request clone | `axum-core-state` |
| `FromRef` used for substate composition | Handlers extract a giant monolithic state struct | Handlers coupled to unrelated state fields | `axum-core-state` |
| Interior mutability uses `tokio::sync::Mutex` | `std::sync::Mutex` guard held across `.await` | `!Send` future; runtime starvation | `axum-core-state`, `axum-core-async-performance` |

## 6. Errors

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| No `.unwrap()` or `.expect()` in handlers | `.unwrap()` on request-derived or fallible values | A panic turns a recoverable error into a crashed task | `axum-errors-handling`, `axum-core-async-performance` |
| A typed `AppError` with `IntoResponse` exists | Ad-hoc `StatusCode` returns scattered everywhere | Inconsistent error responses; no `?` ergonomics | `axum-errors-handling` |
| `From` impls wire `?` conversion | Manual `.map_err()` everywhere | Verbose, error-prone error plumbing | `axum-errors-handling` |
| Extractor rejections customized where needed | Default rejection bodies leaking internal detail | A poor API contract for clients | `axum-errors-extractor-rejections` |
| Fallible `tower::Service` middleware adapted | A failing middleware service with no `HandleError` | A service-level error never becomes an HTTP response | `axum-errors-handling` |

## 7. Auth

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| JWT and session secret read from env | A literal secret string in source | Credentials baked into the binary and git history | `axum-impl-auth-jwt` |
| `Claims` mandates `exp` | A `Claims` struct without an expiry field | Tokens never expire | `axum-impl-auth-jwt` |
| Auth enforced via a `FromRequestParts` extractor | Per-handler manual token parsing | Easy to forget on a new route; not type-enforced | `axum-impl-auth-jwt`, `axum-syntax-custom-extractors` |
| Authorization is permission-based | Hardcoded coarse role string comparisons | Endpoints tightly coupled to a role hierarchy | `axum-impl-authorization` |
| CORS allowlist explicit in production | `CorsLayer::permissive()` or `very_permissive()` shipped | Cross-origin protection disabled | `axum-impl-tower-stack` |
| Session auth uses an established crate | A hand-rolled session store | Reinvented, unaudited session handling | `axum-impl-auth-session` |

## 8. Database

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Exactly one pool, built in `main` | A new `PgPool` or connection created inside a handler | Database connection exhaustion under load | `axum-impl-database`, `axum-core-async-performance` |
| Pool shared via `with_state`, no extra `Arc` | An `Arc<PgPool>` wrapper | Redundant; `Pool` is already a cheap `Clone` | `axum-impl-database`, `axum-core-state` |
| `acquire_timeout` set | `PgPoolOptions` without `acquire_timeout` | A saturated pool hangs instead of failing fast | `axum-impl-database`, `axum-core-async-performance` |
| `max_connections` sized against the server limit | A default or oversized pool times many instances | Exceeds the database server connection cap | `axum-impl-database` |
| Transactions rely on drop-rollback or explicit commit | A `begin()` with neither `commit` nor a clear drop path | A silent rollback or a leaked transaction | `axum-impl-database` |

## 9. Performance

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| No blocking or CPU work in an async handler | Sync file I/O, `std::thread::sleep`, heavy CPU loops, blocking DB drivers | Starves a Tokio worker thread; latency spikes for all requests | `axum-core-async-performance` |
| Blocking work uses `spawn_blocking` | CPU-bound work inline in the handler future | Worker starvation | `axum-core-async-performance` |
| WebSocket and SSE tasks never block | Blocking calls inside a socket task | A starved socket task; dropped connections | `axum-impl-websockets`, `axum-impl-sse` |
| `DefaultBodyLimit` raised for large uploads | A `Multipart` upload path with the default 2 MB limit | Uploads silently rejected at 2 MB | `axum-impl-file-upload` |
| Upload path keeps a hard ceiling | `DefaultBodyLimit::disable()` with no `RequestBodyLimitLayer` | Unbounded body memory-exhaustion DoS | `axum-impl-file-upload` |

## 10. Deployment

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Graceful shutdown wired | `axum::serve(...)` with no `.with_graceful_shutdown()` | SIGTERM kills in-flight requests on every deploy | `axum-impl-deployment` |
| Shutdown signal covers SIGTERM | A signal future selecting only `ctrl_c()` | A container or k8s SIGTERM is not handled | `axum-impl-deployment` |
| Pool closed and tracing flushed on shutdown | A shutdown path that drops resources abruptly | Lost in-flight DB work and unflushed traces | `axum-impl-deployment`, `axum-impl-tracing`, `axum-impl-database` |
| Binds `0.0.0.0` in a container | Binds `127.0.0.1` inside Docker or k8s | The service is unreachable from outside the container | `axum-impl-deployment` |
| Config from env vars | Literal config values in source | Not portable across environments | `axum-impl-deployment` |
| No `Span::enter()` guard across `.await` | A held span guard spanning an await point | Incorrect, interleaved trace output | `axum-impl-tracing`, `axum-core-async-performance` |

## The 16 Anti-Patterns

The 16 verified anti-patterns from `vooronderzoek-axum.md` section 9. Each maps
to a checklist category and to the skill that owns the corrective guidance.
The agent cites the `AP-<n>` number on any finding that matches a row here.

| AP | Anti-pattern | Severity | Checklist category | Owning skill |
|----|--------------|----------|--------------------|--------------|
| AP-1 | Body extractor not last | blocker | 2 Extractors | `axum-syntax-extractors` |
| AP-2 | 0.7 path syntax on 0.8 (startup panic) | blocker | 1 Routing | `axum-core-router`, `axum-core-version-migration` |
| AP-3 | Non-`IntoResponse` handler return | blocker | 3 Handlers | `axum-syntax-handlers`, `axum-errors-handler-trait` |
| AP-4 | `!Send` value held across `.await` | blocker | 3 Handlers, 5 State | `axum-errors-handler-trait`, `axum-core-async-performance` |
| AP-5 | `State` versus `Extension` confusion | suggestion | 5 State | `axum-core-state` |
| AP-6 | Assuming `Option<Path<T>>` swallows errors | warning | 2 Extractors | `axum-syntax-extractors`, `axum-core-version-migration` |
| AP-7 | `.layer()` before routes registered | warning | 1 Routing, 4 Middleware | `axum-core-router`, `axum-impl-middleware` |
| AP-8 | `ServiceBuilder` versus `.layer()` ordering | warning | 4 Middleware | `axum-impl-tower-stack` |
| AP-9 | `CorsLayer::permissive()` in production | warning | 7 Auth | `axum-impl-tower-stack` |
| AP-10 | Blocking or CPU work in an async handler or WS task | warning | 9 Performance | `axum-core-async-performance` |
| AP-11 | `DefaultBodyLimit` silently blocking uploads | warning | 9 Performance | `axum-impl-file-upload` |
| AP-12 | New DB pool per request | warning | 8 Database | `axum-impl-database` |
| AP-13 | `.unwrap()` or `.expect()` in a handler | blocker | 6 Errors | `axum-errors-handling` |
| AP-14 | Missing graceful shutdown | warning | 10 Deployment | `axum-impl-deployment` |
| AP-15 | Hardcoded JWT or DB secret | blocker | 7 Auth, 10 Deployment | `axum-impl-auth-jwt`, `axum-impl-deployment` |
| AP-16 | `tracing` span guard across `.await` | suggestion | 10 Deployment | `axum-impl-tracing` |

Severity rationale: items that fail compilation or panic at startup (AP-1,
AP-2, AP-3) and confirmed security holes (AP-15) are blockers. `.unwrap()` in a
handler (AP-13) is a blocker because it converts a recoverable error into a
crashed task. A `!Send` value across an await (AP-4) is a blocker because it
fails the `Handler` bound at compile time. The remaining operational defects
are warnings; pure best-practice divergences (AP-5, AP-16) are suggestions.

## Sources

This skill performs no WebFetch. Every fact is verified upstream:

- `docs/research/vooronderzoek-axum.md` section 9: the 16 anti-patterns, and
  sections 1 to 8 for the routing, extractor, handler, middleware, state,
  error, auth, database, performance, and deployment context.
- `docs/masterplan/axum-masterplan.md`: the Skill Inventory, providing the
  exact names of the 28 cross-reference target skills.
