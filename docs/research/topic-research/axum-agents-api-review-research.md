# Topic Research : axum-agents-api-review

> Skill : axum-agents-api-review (category : agents, batch 10)
> Status : Phase 4 topic-research
> Generated : 2026-05-20
> Research basis : `docs/research/vooronderzoek-axum.md` (whole file, especially
> §9 Anti-Patterns) + `docs/masterplan/axum-masterplan.md` (Skill Inventory)

This skill is an orchestration skill. It does NOT introduce any new API claims. It
synthesizes the already-verified package research into a deterministic review
checklist for Axum code, where every finding cross-references the exact skill that
owns the fix. No WebFetch is required for this skill.

---

## 1. Purpose and Position

`axum-agents-api-review` is the last analytical skill in the package. Where the other
28 skills teach how to write Axum code correctly, this skill teaches how to AUDIT
existing Axum code against the package's combined knowledge. The agent reads a
codebase (or a diff, or a single file), runs the checklist below category by
category, and emits structured findings. Each finding names the owning skill so the
user can jump straight to the corrective guidance.

The skill must NEVER attempt to teach the fix inline. It detects, classifies by
severity, and delegates to the owning skill. Keeping the fix knowledge in the owning
skill is what keeps this skill short and prevents drift.

---

## 2. Severity Model

Three levels. The agent MUST assign exactly one severity per finding.

| Severity | Definition | Examples |
|----------|------------|----------|
| **blocker** | Code will not compile, panics at startup/runtime, or has a confirmed security hole. Ship-stopping. | 0.7 path syntax on 0.8 (startup panic), body extractor not last (compile fail), hardcoded JWT secret, `.unwrap()` on external input. |
| **warning** | Code compiles and runs but has a latent correctness, reliability, or operational defect that WILL bite under load or in production. | Blocking call in async handler, new DB pool per request, missing graceful shutdown, `CorsLayer::permissive()` in production, `.layer()` before routes. |
| **suggestion** | Code works correctly but diverges from a package best practice; improving it raises maintainability or clarity. | `Extension` where `State` would be compile-checked, coarse roles instead of permission checks, missing `#[instrument]`, no `acquire_timeout` on the pool. |

Rule for the agent : when uncertain between two levels, choose the HIGHER severity.
A false blocker costs a review comment; a missed blocker costs an outage.

---

## 3. Review Checklist by Category

Each check : what to look for, the failure symptom, the owning skill. Skill names are
the exact masterplan inventory names.

### 3.1 Routing

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Path syntax matches target version | `:id` / `*rest` vs `{id}` / `{*rest}` in `.route()` strings | 0.7 syntax panics at router construction on 0.8 | axum-core-router, axum-core-version-migration |
| Routes registered before `.layer()` | `.layer()` call sequenced before all `.route()` calls | Later routes are not wrapped by the middleware | axum-core-router, axum-impl-middleware |
| 405 vs 404 handled distinctly | Only `.fallback()` present where a method-not-allowed path matters | Wrong-method requests return 404 instead of 405 | axum-core-router |
| No double-fallback panic after `.merge()` | Two merged routers each set `.fallback()` | Panic at construction; `reset_fallback` needed | axum-core-router |
| `NestedPath` used for nested-router prefixes | Manual prefix string-building inside nested routers | Fragile, breaks when the mount point changes | axum-core-router |

### 3.2 Extractors

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Body extractor is the LAST argument | `Json<T>`, `Form<T>`, `Multipart`, `Bytes`, `String`, `Request` not in final position | `Handler` trait bound fails to compile (body is single-use) | axum-syntax-extractors, axum-errors-handler-trait |
| At most one body extractor per handler | Two `FromRequest` extractors in one signature | Compile failure | axum-syntax-extractors |
| `Option<Path<T>>` behavior is 0.8-aware | Code assuming `Option<Path<T>>` swallows all errors | 0.8 rejects instead of yielding `None` | axum-syntax-extractors, axum-core-version-migration |
| Custom extractors use the right base trait | `FromRequest` used where no body is consumed | Forces final-argument placement unnecessarily | axum-syntax-custom-extractors |
| Custom extractor `Rejection` implements `IntoResponse` | `Rejection` associated type missing the impl | Compile failure | axum-syntax-custom-extractors, axum-errors-extractor-rejections |

### 3.3 Handlers

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Handler return type is `IntoResponse` | `async fn h() -> i32` or other non-response type | `Handler` trait bound not satisfied | axum-syntax-handlers, axum-errors-handler-trait |
| Handler is declared `async` | A sync `fn` passed to a method-router | `Handler` trait bound not satisfied | axum-syntax-handlers, axum-errors-handler-trait |
| Handler future is `Send` | `Rc`, `RefCell`, `std::sync::MutexGuard` held across `.await` | `!Send` future fails the bound; cryptic error | axum-errors-handler-trait, axum-core-async-performance |
| <= 16 extractor arguments | Handler with 17+ arguments | `Handler` blanket impl does not cover it | axum-syntax-handlers |
| `#[debug_handler]` available for diagnosis | Cryptic "Handler is not satisfied" with no debug aid | Slow, guess-driven debugging | axum-errors-handler-trait |

### 3.4 Middleware

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Layer ordering is intentional | Stacked `Router::layer()` calls without ordering awareness | Bottom-to-top order surprises the author; wrong wrap order | axum-impl-middleware, axum-impl-tower-stack |
| `ServiceBuilder` vs `.layer()` direction known | A `ServiceBuilder` stack assumed to share `Router::layer()` direction | `ServiceBuilder` is top-to-bottom, opposite of stacked `.layer()` | axum-impl-tower-stack |
| Auth middleware uses `route_layer` | Auth applied with `.layer()` instead of `.route_layer()` | A 404 is turned into a 401, leaking route existence | axum-impl-middleware, axum-impl-authorization |
| `from_fn` argument order correct | `Next` not the final parameter, parts after `Request` | Compile failure in `from_fn` middleware | axum-impl-middleware |

### 3.5 State

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| `State` preferred over `Extension` for app data | `Extension<T>` for app-wide config or pools | Runtime 500 if the extension is missing; no compile check | axum-core-state |
| State is `Clone` and cheap to clone | Heavy non-`Arc` data in state | Expensive per-request clone | axum-core-state |
| `FromRef` used for substate composition | Handlers extract a giant monolithic state struct | Handlers coupled to unrelated state fields | axum-core-state |
| Interior mutability uses `tokio::sync::Mutex` | `std::sync::Mutex` guard held across `.await` | `!Send` future; runtime starvation | axum-core-state, axum-core-async-performance |

### 3.6 Errors

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| No `.unwrap()` / `.expect()` in handlers | `.unwrap()` on request-derived or fallible values | Panic turns a recoverable error into a crashed task | axum-errors-handling, axum-core-async-performance |
| A typed `AppError` with `IntoResponse` exists | Ad-hoc `StatusCode` returns scattered everywhere | Inconsistent error responses; no `?` ergonomics | axum-errors-handling |
| `From` impls wire `?` conversion | Manual `.map_err()` everywhere | Verbose, error-prone error plumbing | axum-errors-handling |
| Extractor rejections customized where needed | Default rejection bodies leaking internal detail | Poor API contract for clients | axum-errors-extractor-rejections |
| Fallible `tower::Service` middleware adapted | A failing middleware service with no `HandleError` | Service-level error never becomes an HTTP response | axum-errors-handling |

### 3.7 Auth

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| JWT/session secret read from env | A literal secret string in source | Credentials baked into the binary and git history | axum-impl-auth-jwt |
| `Claims` mandates `exp` | A `Claims` struct without an expiry field | Tokens never expire | axum-impl-auth-jwt |
| Auth enforced via a `FromRequestParts` extractor | Per-handler manual token parsing | Easy to forget on a new route; not type-enforced | axum-impl-auth-jwt, axum-syntax-custom-extractors |
| Authorization is permission-based | Hardcoded coarse role string comparisons | Endpoints tightly coupled to a role hierarchy | axum-impl-authorization |
| CORS allowlist is explicit in production | `CorsLayer::permissive()` / `very_permissive()` shipped | Cross-origin protection disabled | axum-impl-tower-stack |
| Session auth uses an established crate | Hand-rolled session store | Reinvented, unaudited session handling | axum-impl-auth-session |

### 3.8 Database

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Exactly one pool, built in `main` | A new `PgPool`/connection created inside a handler | Database connection exhaustion under load | axum-impl-database, axum-core-async-performance |
| Pool shared via `with_state`, no extra `Arc` | `Arc<PgPool>` wrapper | Redundant; `Pool` is already a cheap `Clone` | axum-impl-database, axum-core-state |
| `acquire_timeout` set | `PgPoolOptions` without `acquire_timeout` | A saturated pool hangs instead of failing fast | axum-impl-database, axum-core-async-performance |
| `max_connections` sized against the server limit | Default or oversized pool x many instances | Exceeds the database server connection cap | axum-impl-database |
| Transactions rely on drop-rollback or explicit commit | A `begin()` with neither `commit` nor a clear drop path | Silent rollback or a leaked transaction | axum-impl-database |

### 3.9 Performance

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| No blocking / CPU work in an async handler | Sync file I/O, `std::thread::sleep`, heavy CPU loops, blocking DB drivers | Starves a Tokio worker thread; latency spikes for all requests | axum-core-async-performance |
| Blocking work uses `spawn_blocking` | CPU-bound work inline in the handler future | Worker starvation | axum-core-async-performance |
| WebSocket / SSE tasks never block | Blocking calls inside a socket task | Starved socket task; dropped connections | axum-impl-websockets, axum-impl-sse |
| `DefaultBodyLimit` raised for large uploads | `Multipart` upload path with the default 2 MB limit | Uploads silently rejected at 2 MB | axum-impl-file-upload |
| Upload path keeps a hard ceiling | `DefaultBodyLimit::disable()` with no `RequestBodyLimitLayer` | Unbounded body memory-exhaustion DoS | axum-impl-file-upload |

### 3.10 Deployment

| Check | Look for | Symptom of failure | Owning skill |
|-------|----------|--------------------|--------------|
| Graceful shutdown wired | `axum::serve(...)` with no `.with_graceful_shutdown()` | SIGTERM kills in-flight requests on every deploy | axum-impl-deployment |
| Shutdown signal covers SIGTERM | A signal future selecting only `ctrl_c()` | Container/k8s SIGTERM is not handled | axum-impl-deployment |
| Pool closed + tracing flushed on shutdown | Shutdown path that drops resources abruptly | Lost in-flight DB work and unflushed traces | axum-impl-deployment, axum-impl-tracing, axum-impl-database |
| Binds `0.0.0.0` in a container | Binds `127.0.0.1` inside Docker/k8s | Service unreachable from outside the container | axum-impl-deployment |
| Config from env vars | Literal config values in source | Not portable across environments | axum-impl-deployment |
| No `Span::enter()` guard across `.await` | A held span guard spanning an await point | Incorrect, interleaved trace output | axum-impl-tracing, axum-core-async-performance |

---

## 4. The 16 Anti-Patterns Mapped to Checks and Owning Skills

The 16 verified anti-patterns from vooronderzoek §9. Each maps to a checklist row
above and to the skill that owns the corrective guidance.

| # | Anti-pattern | Severity | Check (section) | Owning skill |
|---|--------------|----------|-----------------|--------------|
| 1 | Body extractor not last | blocker | Extractors 3.2 | axum-syntax-extractors |
| 2 | 0.7 path syntax on 0.8 (startup panic) | blocker | Routing 3.1 | axum-core-router, axum-core-version-migration |
| 3 | Non-`IntoResponse` handler return | blocker | Handlers 3.3 | axum-syntax-handlers, axum-errors-handler-trait |
| 4 | `!Send` value held across `.await` | blocker | Handlers 3.3 / State 3.5 | axum-errors-handler-trait, axum-core-async-performance |
| 5 | `State` vs `Extension` confusion | suggestion | State 3.5 | axum-core-state |
| 6 | Assuming `Option<Path<T>>` swallows errors | warning | Extractors 3.2 | axum-syntax-extractors, axum-core-version-migration |
| 7 | `.layer()` before routes registered | warning | Routing 3.1 / Middleware 3.4 | axum-core-router, axum-impl-middleware |
| 8 | `ServiceBuilder` vs `.layer()` ordering | warning | Middleware 3.4 | axum-impl-tower-stack |
| 9 | `CorsLayer::permissive()` in production | warning | Auth 3.7 | axum-impl-tower-stack |
| 10 | Blocking / CPU work in an async handler or WS task | warning | Performance 3.9 | axum-core-async-performance |
| 11 | `DefaultBodyLimit` silently blocking uploads | warning | Performance 3.9 | axum-impl-file-upload |
| 12 | New DB pool per request | warning | Database 3.8 | axum-impl-database |
| 13 | `.unwrap()` / `.expect()` in a handler | blocker | Errors 3.6 | axum-errors-handling |
| 14 | Missing graceful shutdown | warning | Deployment 3.10 | axum-impl-deployment |
| 15 | Hardcoded JWT / DB secret | blocker | Auth 3.7 / Deployment 3.10 | axum-impl-auth-jwt, axum-impl-deployment |
| 16 | `tracing` span guard across `.await` | suggestion | Deployment 3.10 | axum-impl-tracing |

Severity guidance for the agent : items that fail compilation or panic at startup
(1, 2, 3) and confirmed security holes (15) are blockers. `.unwrap()` in a handler
(13) is a blocker because it converts a recoverable error into a crashed task.
`!Send` across await (4) is a blocker because it fails the `Handler` bound at
compile time. The remaining operational defects are warnings; pure best-practice
divergences (5, 16) are suggestions.

---

## 5. Structured Findings Output Format

The agent MUST emit findings in this exact structure so output is machine-readable
and consistently scannable.

### 5.1 Per-finding block

```
[<SEVERITY>] <short title>
  Location : <file>:<line> (or "design-level" if not line-specific)
  Symptom  : <what goes wrong, observable behavior>
  Anti-pattern : AP-<n> (if it maps to one of the 16; omit otherwise)
  Fix skill : axum-<category>-<topic>
  Action   : <one-line corrective direction; consult the fix skill for detail>
```

### 5.2 Summary header

The agent MUST precede the findings with a count summary :

```
Axum API Review : <N> findings
  blockers : <b>   warnings : <w>   suggestions : <s>
  Target Axum version : <0.7 | 0.8 | unknown>
```

### 5.3 Ordering rule

Findings MUST be ordered : all blockers first, then warnings, then suggestions.
Within a severity, order by file then line. A review with zero findings MUST still
emit the summary header with all counts at 0 and an explicit "No issues found"
line, so an empty result is never ambiguous.

### 5.4 Determinism rule

The agent NEVER reports a finding it cannot tie to a specific check in section 3.
Every finding cites either an AP number or a checklist category. The agent NEVER
invents a severity outside {blocker, warning, suggestion}.

---

## 6. Cross-Reference Coverage

The skill cross-references all 28 other skills. Coverage map (skill : the checklist
categories that point to it) :

- axum-core-architecture : (referenced by axum-agents-scaffold, not a check target)
- axum-core-router : 3.1
- axum-core-state : 3.5, 3.8
- axum-core-async-performance : 3.3, 3.5, 3.6, 3.8, 3.9, 3.10
- axum-core-version-migration : 3.1, 3.2
- axum-syntax-extractors : 3.2, 3.3
- axum-syntax-custom-extractors : 3.2, 3.7
- axum-syntax-handlers : 3.3
- axum-syntax-responses : (no dedicated check; covered via errors)
- axum-impl-middleware : 3.1, 3.4
- axum-impl-tower-stack : 3.4, 3.7
- axum-impl-static-files : (no dedicated check; low audit risk)
- axum-impl-websockets : 3.9
- axum-impl-sse : 3.9
- axum-impl-file-upload : 3.9
- axum-impl-auth-jwt : 3.7
- axum-impl-auth-session : 3.7
- axum-impl-authorization : 3.4, 3.7
- axum-impl-database : 3.8
- axum-impl-validation : (no dedicated check; covered via extractor rejections)
- axum-impl-testing : (no dedicated check; review verifies code, not tests)
- axum-impl-tracing : 3.10
- axum-impl-openapi : (no dedicated check)
- axum-impl-deployment : 3.10
- axum-errors-handling : 3.6
- axum-errors-extractor-rejections : 3.2, 3.6
- axum-errors-handler-trait : 3.2, 3.3

Skills with no dedicated checklist row (architecture, responses, static-files,
validation, testing, openapi) are still listed so the SKILL.md can present a complete
"all 28 skills" cross-reference table. The review checklist concentrates on the
categories where verified anti-patterns exist.

---

## 7. Sources

This skill introduces NO new API claims. It is synthesized entirely from the verified
package research :

- `docs/research/vooronderzoek-axum.md` : §9 (the 16 anti-patterns), §1-§8 (the
  category context for routing, extractors, handlers, middleware, state, errors,
  auth, database, performance, deployment).
- `docs/masterplan/axum-masterplan.md` : the Skill Inventory section, providing the
  exact names of the 28 other skills used as cross-reference targets.

All anti-pattern claims trace back to the WebFetch-verified fragments behind
vooronderzoek §9. No WebFetch is performed for this skill : every fact is already
verified upstream. The skill's job is composition and routing, not new research.
