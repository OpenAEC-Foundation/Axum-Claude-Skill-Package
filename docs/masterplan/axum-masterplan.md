# Masterplan : Axum Skill Package

> Status : Phase 3 refined : executable
> Generated : 2026-05-19
> Refined : 2026-05-20
> Research basis : `docs/research/vooronderzoek-axum.md` + `docs/research/fragments/`

## Scope

- Technology : Axum (Rust async web framework)
- Versions : 0.7, 0.8
- Languages : Rust
- Prefix : axum
- License : MIT
- Companion stack : tokio, hyper, tower, tower-http, sqlx, serde, tracing
- Focus boundary : Axum ONLY. NOT Actix-Web, NOT Rocket. Companion crates covered
  only where they integrate directly with Axum.

## Final Skill Count : 29 skills, 5 categories, 10 batches

| Category | Count |
|----------|-------|
| core/    | 5     |
| syntax/  | 4     |
| impl/    | 15    |
| errors/  | 3     |
| agents/  | 2     |
| **Total** | **29** |

---

## Refinement Decisions

Research-driven changes from the raw pre-research inventory. Per BOOTSTRAP-RUNBOOK
§5.1, research without consequences is not thorough.

| ID | Decision | Reason | Source |
|----|----------|--------|--------|
| D-01 | SPLIT `axum-impl-auth` -> `axum-impl-auth-jwt` + `axum-impl-auth-session` | Research showed JWT (stateless, self-contained token) and session-based auth (stateful, `tower-sessions`/`axum-login`) are structurally different audiences and patterns; 4 auth methods in one skill exceeds the 500-line bound | vooronderzoek §5; fragment C §4 |
| D-02 | MERGE `axum-impl-graceful-shutdown` -> `axum-impl-deployment` | Graceful shutdown is a thin topic (one `with_graceful_shutdown` call + a signal future) tightly coupled to deployment: SIGTERM is the Docker/k8s/systemd stop signal, the official example pairs them, and shutdown closes the DB pool and flushes tracing | vooronderzoek §6, §8; fragment C §9, §11 |
| D-03 | RENAME `axum-errors-trait-bounds` -> `axum-errors-handler-trait`, expand scope to absorb the `#[debug_handler]` diagnostic workflow | Research (L-002) showed the no-macros design concentrates every handler-shape mistake into one cryptic "Handler is not satisfied" error; the dedicated skill must own the diagnostic tool, not just describe symptoms | LESSONS L-002; fragment A §6 |
| D-04 | RENAME `axum-core-performance` -> `axum-core-async-performance`, lead with the "never block the runtime" rule | Research established the blocking-call-starves-the-runtime mistake as the single most important Axum operational rule, broader than raw "performance" framing | vooronderzoek §9 AP-10; fragment C §11 |
| D-05 | KEEP 19 newly discovered sub-topics as folded sections, not standalone skills | Sub-topics (cookies/axum-extra, rate limiting, `route_layer` vs `layer`, `SetStatus`, `anyhow` newtype, `utoipa-axum`, `cargo-chef`, `garde`, OpenTelemetry, `WithRejection`, `NestedPath`, `FromRef`, `OptionalFromRequest`, etc.) each belong inside an existing skill's scope; standalone skills would fragment the catalog | vooronderzoek "Newly Discovered Sub-Topics" |

Net effect vs raw inventory : SPLIT (+1) and MERGE (-1) cancel; final count 29.

---

## Skill Inventory

Dependency chain : `core -> syntax -> impl -> errors -> agents`.

### core/ (5)

| # | Skill | Batch |
|---|-------|-------|
| 1 | axum-core-architecture | 1 |
| 2 | axum-core-router | 1 |
| 3 | axum-core-state | 1 |
| 4 | axum-core-async-performance | 2 |
| 5 | axum-core-version-migration | 2 |

### syntax/ (4)

| # | Skill | Batch |
|---|-------|-------|
| 6 | axum-syntax-extractors | 2 |
| 7 | axum-syntax-custom-extractors | 3 |
| 8 | axum-syntax-handlers | 3 |
| 9 | axum-syntax-responses | 3 |

### impl/ (15)

| # | Skill | Batch |
|---|-------|-------|
| 10 | axum-impl-middleware | 4 |
| 11 | axum-impl-tower-stack | 4 |
| 12 | axum-impl-static-files | 4 |
| 13 | axum-impl-websockets | 5 |
| 14 | axum-impl-sse | 5 |
| 15 | axum-impl-file-upload | 5 |
| 16 | axum-impl-auth-jwt | 6 |
| 17 | axum-impl-auth-session | 6 |
| 18 | axum-impl-authorization | 6 |
| 19 | axum-impl-database | 7 |
| 20 | axum-impl-validation | 7 |
| 21 | axum-impl-testing | 7 |
| 22 | axum-impl-tracing | 8 |
| 23 | axum-impl-openapi | 8 |
| 24 | axum-impl-deployment | 8 |

### errors/ (3)

| # | Skill | Batch |
|---|-------|-------|
| 25 | axum-errors-handling | 9 |
| 26 | axum-errors-extractor-rejections | 9 |
| 27 | axum-errors-handler-trait | 9 |

### agents/ (2)

| # | Skill | Batch |
|---|-------|-------|
| 28 | axum-agents-api-review | 10 |
| 29 | axum-agents-scaffold | 10 |

---

## Execution Plan : Batches

Batch size 3 (Claude Code Agent / tmux-orchestration optimum). File-scope per batch :
no two skills share a directory. Quality gate after every batch.

| Batch | Skills | Count | Dependencies | Category |
|-------|--------|-------|--------------|----------|
| 1 | axum-core-architecture, axum-core-router, axum-core-state | 3 | none | core |
| 2 | axum-core-async-performance, axum-core-version-migration, axum-syntax-extractors | 3 | batch 1 | core/syntax |
| 3 | axum-syntax-custom-extractors, axum-syntax-handlers, axum-syntax-responses | 3 | batch 2 | syntax |
| 4 | axum-impl-middleware, axum-impl-tower-stack, axum-impl-static-files | 3 | batch 3 | impl |
| 5 | axum-impl-websockets, axum-impl-sse, axum-impl-file-upload | 3 | batch 3 | impl |
| 6 | axum-impl-auth-jwt, axum-impl-auth-session, axum-impl-authorization | 3 | batch 3 | impl |
| 7 | axum-impl-database, axum-impl-validation, axum-impl-testing | 3 | batch 3 | impl |
| 8 | axum-impl-tracing, axum-impl-openapi, axum-impl-deployment | 3 | batch 3 | impl |
| 9 | axum-errors-handling, axum-errors-extractor-rejections, axum-errors-handler-trait | 3 | batch 4-8 | errors |
| 10 | axum-agents-api-review, axum-agents-scaffold | 2 | ALL | agents |

Phase 4 topic-research is interleaved : before dispatching each batch, the orchestrator
spawns in-process opus agents to write `docs/research/topic-research/{skill}-research.md`
for that batch's skills. Skip-criterion (Docker L-001) : the Phase 2 vooronderzoek is
deep and verified, so topic-research is a focused supplement (exact signatures, extra
examples), not a full re-research.

Worker context overflow protection (L-007) : respawn the 3 tmux workers between every
batch for a clean token budget.

---

## Common Agent Prompt Block

Every worker prompt = this COMMON BLOCK + the skill-specific block below it.

```
You are a skill-builder worker for the Axum Claude Skill Package.

Workspace : /home/freek/GitHub/Axum-Claude-Skill-Package/
Read first : docs/research/vooronderzoek-axum.md, the relevant fragment in
docs/research/fragments/, the skill's topic-research file in
docs/research/topic-research/, REQUIREMENTS.md, WAY_OF_WORK.md, SOURCES.md.

Files to create (exactly these, in the skill's output dir) :
  - SKILL.md            (max 500 lines)
  - references/methods.md      (complete API signatures)
  - references/examples.md     (working, version-annotated code)
  - references/anti-patterns.md (real mistakes + WHY each fails)

SKILL.md YAML frontmatter (folded scalar, NEVER quoted) :
  ---
  name: axum-{category}-{topic}
  description: >
    Use when [trigger scenario].
    Prevents [the anti-pattern].
    Covers [topics, API areas, version differences].
    Keywords: [technical terms + symptom phrases + plain-language synonyms].
  license: MIT
  compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
  metadata:
    author: OpenAEC-Foundation
    version: "1.0"
  ---

SKILL.md structure : Quick Reference -> Decision Trees -> Patterns ->
Reference Links (to the 3 reference files).

Quality rules (HARD) :
  - English only. Deterministic language : "ALWAYS X", "NEVER Y". No "you might".
  - Every code example verified against SOURCES.md approved URLs via WebFetch.
    NO hallucinated APIs. NO unverified signatures.
  - Annotate version-divergent code : `// axum 0.8` / `// axum 0.7`.
  - NO em-dashes anywhere. Section headings use a colon, not an em-dash.
  - SKILL.md < 500 lines; overflow goes to references/.
  - NO README.md inside the skill folder.
  - Quoted YAML description = FORBIDDEN (use folded `>`).
  - Keywords line MUST mix 3 types : technical (API/error names), symptom-based
    ("nothing shows", "panics on startup", "401 instead of 404"), plain-language
    ("how do I", "what is").

Quality gate before done : the skill must pass
  node $TEMPLATE/scripts/validate-frontmatter.js <skill-dir>
  node $TEMPLATE/scripts/validate-language.js <skill-dir>
  node $TEMPLATE/scripts/validate-line-count.js <skill-dir>
  node $TEMPLATE/scripts/validate-structure.js <skill-dir>
  node $TEMPLATE/scripts/validate-emdash.js <skill-dir>
($TEMPLATE = /home/freek/GitHub/Skill-Package-Workflow-Template)
```

---

## Skill Blocks

Each block below is the skill-specific half of the worker prompt. Output dir pattern :
`skills/source/axum-{category}/{skill-name}/`.

### Skill: axum-core-architecture

- Output dir : skills/source/axum-core/axum-core-architecture/
- Research input : vooronderzoek §1, §2; fragment A §1
- Scope :
  - Axum as a composition over tokio (runtime) + hyper 1.0 (HTTP) + tower (Service trait)
  - The request lifecycle from TcpListener to handler to response
  - The "no macros" design philosophy and its trade-off (cryptic Handler errors)
  - `axum::serve` as the glue; `#[tokio::main]` entry point
  - The `axum-core` crate boundary (foundational traits) vs `axum`
  - When to choose Axum (decision context, not a framework comparison)
- Cross-refs : axum-core-router, axum-syntax-handlers, axum-errors-handler-trait

### Skill: axum-core-router

- Output dir : skills/source/axum-core/axum-core-router/
- Research input : vooronderzoek §2; fragment A §2, §3
- Scope :
  - `Router<S>`, `.route()`, method-routers (get/post/put/delete/patch/head/options/any)
  - Chaining method-routers on one path; `MethodRouter` and the 405-vs-404 distinction
    (`.fallback()` true 404 vs `MethodRouter::fallback` / `.method_not_allowed_fallback()`)
  - `.nest()`, `.nest_service()`, `.merge()` (and the double-fallback panic + `reset_fallback`)
  - Route matching precedence, overlap panics, catch-all `/{*key}` semantics
  - `.layer()` vs `.route_layer()` placement rule (register routes before layering)
  - `NestedPath` extractor for nested routers
  - Path syntax : 0.8 `{id}`/`{*rest}` vs 0.7 `:id`/`*rest` (annotate both)
- Cross-refs : axum-core-version-migration, axum-impl-middleware, axum-core-state

### Skill: axum-core-state

- Output dir : skills/source/axum-core/axum-core-state/
- Research input : vooronderzoek §2, §4; fragment A §4; fragment B §5
- Scope :
  - `Router::with_state`, the `State<S>` extractor, `Router<S>` -> `Router<()>` typing
  - State must be `Clone`; `Arc` for heavy/shared-mutable data
  - `FromRef` / `#[derive(FromRef)]` substate composition
  - `State` vs `Extension` : compile-time-checked vs runtime, when to use each
  - The "missing state" compile error and how to read it
  - Interior mutability : `tokio::sync::Mutex` (never `std::sync::Mutex` across `.await`)
- Cross-refs : axum-syntax-extractors, axum-impl-database, axum-errors-handler-trait

### Skill: axum-core-async-performance

- Output dir : skills/source/axum-core/axum-core-async-performance/
- Research input : vooronderzoek §8, §9; fragment C §11; fragment B §6
- Scope :
  - The cardinal rule : NEVER block the Tokio runtime in an async handler
  - `tokio::task::spawn_blocking` for CPU/sync work; `tokio::spawn` for concurrent async
  - Async I/O equivalents (`tokio::fs`, `sqlx`, `tokio::time::sleep`)
  - `!Send` future traps (`Rc`, `RefCell`, `std::sync::MutexGuard` across `.await`)
  - Connection pool sizing principles; `acquire_timeout` fail-fast
  - tokio runtime tuning basics (worker threads, blocking pool)
- Cross-refs : axum-impl-database, axum-errors-handler-trait, axum-impl-websockets

### Skill: axum-core-version-migration

- Output dir : skills/source/axum-core/axum-core-version-migration/
- Research input : vooronderzoek §3; fragment A §3, §5; fragment B §6; fragment C version note
- Scope :
  - Complete v0.7 -> v0.8 breaking-change matrix
  - Path syntax `:param`/`*wildcard` -> `{param}`/`{*wildcard}` (panics on old syntax);
    `Router::without_v07_checks` escape hatch
  - `#[async_trait]` removal from `FromRequest`/`FromRequestParts` (native RPITIT, MSRV 1.75)
  - WebSocket `Message` : `String`/`Vec<u8>` -> `Utf8Bytes`/`Bytes`
  - `Option<Path<T>>` rejection-behavior change (`OptionalFromRequest`)
  - Removed-API checklist : `RawBody`, `BodyStream`, `axum::Server`, `B` body param,
    `hyper::Body` re-export, `headers`/`TypedHeader` moved to `axum-extra`
  - Step-by-step migration checklist
- Cross-refs : all skills (version annotation), axum-syntax-custom-extractors

### Skill: axum-syntax-extractors

- Output dir : skills/source/axum-syntax/axum-syntax-extractors/
- Research input : vooronderzoek §2; fragment A §3, §4
- Scope :
  - Built-in extractors : `Path<T>`, `Query<T>`, `Json<T>`, `Form<T>`, `State<T>`,
    `Extension<T>`, `HeaderMap`, `Method`, `Uri`, `Bytes`, `String`, `Request`
  - `FromRequestParts` (no body) vs `FromRequest` (body) classification table
  - The extractor ordering rule : at most one body extractor, MUST be last
  - `Path<T>` deserialization (single, tuple, struct, HashMap)
  - `Option<Extractor>` and `OptionalFromRequest` (0.8 behavior)
  - decision tree : which extractor for which input
- Cross-refs : axum-syntax-custom-extractors, axum-errors-extractor-rejections, axum-core-state

### Skill: axum-syntax-custom-extractors

- Output dir : skills/source/axum-syntax/axum-syntax-custom-extractors/
- Research input : vooronderzoek §2, §3; fragment A §5; fragment C §2
- Scope :
  - Implementing `FromRequestParts<S>` (no body) and `FromRequest<S>` (body)
  - The `Rejection` associated type (must implement `IntoResponse`)
  - Native `async fn` in trait : 0.8 has NO `#[async_trait]`, 0.7 needs it (show both)
  - Wrapping an inner extractor and remapping its rejection
  - `#[derive(FromRequest)]` / `#[derive(FromRequestParts)]` from axum-macros
  - The rule : do NOT implement both traits for the same concrete type (unless a generic wrapper)
- Cross-refs : axum-core-version-migration, axum-errors-extractor-rejections, axum-impl-validation

### Skill: axum-syntax-handlers

- Output dir : skills/source/axum-syntax/axum-syntax-handlers/
- Research input : vooronderzoek §2; fragment A §6
- Scope :
  - `async fn` handler signature, the `Handler<T, S>` trait, the 16-extractor limit
  - Valid return types (anything `IntoResponse`); `Result<T, E>` handlers
  - The blanket-impl criteria for what IS a valid handler
  - Async closures as handlers; `HandlerWithoutStateExt::into_make_service`
  - Brief pointer to axum-errors-handler-trait for diagnosing invalid handlers
- Cross-refs : axum-syntax-responses, axum-errors-handler-trait, axum-syntax-extractors

### Skill: axum-syntax-responses

- Output dir : skills/source/axum-syntax/axum-syntax-responses/
- Research input : vooronderzoek §2; fragment A §7
- Scope :
  - `IntoResponse` and `IntoResponseParts` traits; the built-in impl table
  - `StatusCode`, `(StatusCode, T)`, `(StatusCode, headers, T)` tuples
  - `Json<T>`, `Html<T>`, raw `Vec<u8>`/`Bytes` bodies
  - `Redirect::to`/`permanent`/`temporary`; `AppendHeaders`
  - Headers via `[(HeaderName, value); N]`
  - Cookies : raw `Set-Cookie` headers AND the `axum-extra` `CookieJar`/`SignedCookieJar`/
    `PrivateCookieJar` extractors
  - 0.8 note : response body type must be `axum::body::Body`
- Cross-refs : axum-syntax-handlers, axum-errors-handling

### Skill: axum-impl-middleware

- Output dir : skills/source/axum-impl/axum-impl-middleware/
- Research input : vooronderzoek §2; fragment B §1, §2
- Scope :
  - `axum::middleware::from_fn`, `from_fn_with_state` (exact argument order : parts
    extractors, then `Request`, then `Next`)
  - `map_request`, `map_response` and `_with_state` variants
  - The `Next` type and `next.run(request).await`; deliberate short-circuit for auth gates
  - Middleware ordering : stacked `Router::layer()` is bottom-to-top (last = outermost)
  - `.route_layer()` vs `.layer()` : the 404-vs-401 decision tree
  - When to write a `tower::Layer` instead of `from_fn`
- Cross-refs : axum-impl-tower-stack, axum-impl-authorization, axum-core-router

### Skill: axum-impl-tower-stack

- Output dir : skills/source/axum-impl/axum-impl-tower-stack/
- Research input : vooronderzoek §2; fragment B §3, §4
- Scope :
  - The `tower::Service` trait (`poll_ready` backpressure, `call`) and `tower::Layer`
  - `ServiceBuilder` ordering : top-to-bottom (first = outermost) : OPPOSITE of stacked
    `Router::layer()`. Provide the deterministic rule.
  - tower-http layers : `TraceLayer`, `CorsLayer`, `CompressionLayer`, `TimeoutLayer`,
    `RequestBodyLimitLayer` (with feature flags)
  - Rate limiting : no first-party tower-http layer; `tower::limit` or the
    `tower_governor` crate (recommended)
- Cross-refs : axum-impl-middleware, axum-impl-tracing, axum-impl-file-upload

### Skill: axum-impl-static-files

- Output dir : skills/source/axum-impl/axum-impl-static-files/
- Research input : fragment B §9
- Scope :
  - `tower_http::services::ServeDir`, `ServeFile` (Services, not Layers : mount via
    `nest_service`/`fallback_service`)
  - `.not_found_service()` and `.fallback()` on `ServeDir`
  - The SPA fallback pattern : `SetStatus` wrapping `ServeFile` to return index.html with 404
  - `nest_service` prefix stripping
- Cross-refs : axum-core-router, axum-impl-tower-stack

### Skill: axum-impl-websockets

- Output dir : skills/source/axum-impl/axum-impl-websockets/
- Research input : vooronderzoek §6; fragment B §6
- Scope :
  - `WebSocketUpgrade` extractor, `on_upgrade`, `OnFailedUpgrade`
  - `WebSocket` : `recv`, `send`, `.split()` into sink + stream
  - `Message` enum variants; the 0.8 type change `String`/`Vec<u8>` -> `Utf8Bytes`/`Bytes`
  - Concurrent send/receive : split + two `tokio::spawn` tasks + `tokio::select!`
  - Combining with `State`; NEVER block the socket task
- Cross-refs : axum-core-async-performance, axum-core-version-migration, axum-impl-sse

### Skill: axum-impl-sse

- Output dir : skills/source/axum-impl/axum-impl-sse/
- Research input : vooronderzoek §6; fragment B §7
- Scope :
  - `axum::response::sse::Sse`, `Event`, `KeepAlive`
  - Building an `Event` stream from a futures `Stream`; the `Result<Event, E>` item type
  - `Event` builder : `.data`, `.event`, `.id`, `.retry`, `.comment`
  - ALWAYS attach `KeepAlive` for long-lived connections
  - SSE vs WebSockets decision (one-way push vs bidirectional)
- Cross-refs : axum-impl-websockets, axum-syntax-handlers

### Skill: axum-impl-file-upload

- Output dir : skills/source/axum-impl/axum-impl-file-upload/
- Research input : fragment B §8
- Scope :
  - `axum::extract::Multipart`, `next_field()` loop, `Field` methods
  - `Multipart` is a `FromRequest` extractor : MUST be the last handler argument
  - Streaming large files chunk by chunk to `tokio::fs::File` vs buffering with `.bytes()`
  - `DefaultBodyLimit` (2 MB default) : `max(n)` / `disable()` + `RequestBodyLimitLayer`
  - ALWAYS keep a hard ceiling (memory-exhaustion DoS)
- Cross-refs : axum-impl-tower-stack, axum-core-async-performance

### Skill: axum-impl-auth-jwt

- Output dir : skills/source/axum-impl/axum-impl-auth-jwt/
- Research input : vooronderzoek §5; fragment C §4
- Scope :
  - Stateless JWT auth with the `jsonwebtoken` crate (v10, aws_lc_rs)
  - `Claims` struct (mandatory `exp`), `encode`/`decode`, `Validation`
  - Token verification inside a `FromRequestParts` extractor : any handler taking
    `Claims` is automatically protected
  - `EncodingKey`/`DecodingKey` from an env-var secret (NEVER a literal)
  - The login route that issues a token; `AuthError` enum + `IntoResponse`
- Cross-refs : axum-syntax-custom-extractors, axum-impl-authorization, axum-errors-handling

### Skill: axum-impl-auth-session

- Output dir : skills/source/axum-impl/axum-impl-auth-session/
- Research input : vooronderzoek §5; fragment C §4
- Scope :
  - Session-based auth : cookie session id + server-side session store
  - `tower-sessions` (`SessionManagerLayer`, pluggable stores) and `axum-login`
    (`AuthSession`, `login_required!` guard)
  - HTTP Basic auth via `axum-extra` `TypedHeader<Authorization<Basic>>` (TLS-only)
  - OAuth2 authorization-code flow with the `oauth2` crate (redirect + callback routes)
  - Decision tree : JWT vs session vs basic vs OAuth2
- Cross-refs : axum-impl-auth-jwt, axum-syntax-responses, axum-impl-authorization

### Skill: axum-impl-authorization

- Output dir : skills/source/axum-impl/axum-impl-authorization/
- Research input : vooronderzoek §5; fragment C §5
- Scope :
  - Role-based and permission-based access control as middleware
  - `from_fn_with_state` checking claims; apply with `route_layer` (404 not -> 401)
  - Permission checks vs coarse roles (decoupling endpoints from a role hierarchy)
  - Per-route guard via a dedicated newtype extractor (`AdminClaims`)
- Cross-refs : axum-impl-middleware, axum-impl-auth-jwt, axum-syntax-custom-extractors

### Skill: axum-impl-database

- Output dir : skills/source/axum-impl/axum-impl-database/
- Research input : vooronderzoek §6; fragment C §6
- Scope :
  - `sqlx` connection pool (`PgPool`/`MySqlPool`) created once, shared via `with_state`
  - `Pool` is a cheap reference-counted `Clone` : NO `Arc` wrapper needed
  - `PgPoolOptions` (`max_connections`, `acquire_timeout`)
  - Query macros (`query!`, `query_as!`, `query_scalar!`, compile-time checked) vs
    runtime functions; `fetch_one`/`fetch_optional`/`fetch_all`/`execute`
  - Transactions : `pool.begin()`, `commit`/`rollback`, drop = auto-rollback
  - A `DatabaseConnection` `FromRequestParts` extractor; pool sizing
- Cross-refs : axum-core-state, axum-core-async-performance, axum-errors-handling

### Skill: axum-impl-validation

- Output dir : skills/source/axum-impl/axum-impl-validation/
- Research input : fragment C §3
- Scope :
  - The `validator` crate `#[derive(Validate)]` and attributes (email, url, length,
    range, custom, nested, regex, must_match)
  - A `ValidatedJson<T>` custom extractor : deserialize then `.validate()`, return 422
  - `garde` as a context-aware alternative (mention)
  - 0.7 vs 0.8 form of the extractor impl (`#[async_trait]` difference)
- Cross-refs : axum-syntax-custom-extractors, axum-errors-extractor-rejections

### Skill: axum-impl-testing

- Output dir : skills/source/axum-impl/axum-impl-testing/
- Research input : fragment C §7
- Scope :
  - `axum-test` `TestServer` : `.json()`, `assert_status_ok`, `assert_json`, `assert_text`
  - The `tower::ServiceExt::oneshot` pattern (Router IS a Service, no extra crate)
  - `http_body_util::BodyExt::collect` to read response bodies
  - Shared `app()` constructor for unit + integration tests; `tests/` layout
- Cross-refs : axum-core-architecture, axum-impl-database

### Skill: axum-impl-tracing

- Output dir : skills/source/axum-impl/axum-impl-tracing/
- Research input : fragment C §8
- Scope :
  - `tracing-subscriber` : `registry()`, `EnvFilter`, `fmt::layer()`
  - `tower_http::trace::TraceLayer::new_for_http()` and its customization hooks
  - Event macros (`info!`/`error!`/...), `#[instrument]`, the `?`/`%` field sigils
  - NEVER hold a `Span::enter()` guard across `.await`
  - OpenTelemetry export (`opentelemetry`, `opentelemetry-otlp`, `tracing-opentelemetry`)
- Cross-refs : axum-impl-tower-stack, axum-core-async-performance

### Skill: axum-impl-openapi

- Output dir : skills/source/axum-impl/axum-impl-openapi/
- Research input : fragment C §10
- Scope :
  - `utoipa` : `#[derive(ToSchema)]`, `#[utoipa::path(...)]`, `#[derive(OpenApi)]`
  - `utoipa-swagger-ui` `SwaggerUi` integration
  - `utoipa-axum` `OpenApiRouter` : unifies routing + spec so they cannot drift
  - Note : utoipa's `{param}` path syntax matches Axum 0.8 exactly
- Cross-refs : axum-core-router, axum-syntax-handlers

### Skill: axum-impl-deployment

- Output dir : skills/source/axum-impl/axum-impl-deployment/
- Research input : vooronderzoek §8; fragment C §9, §11
- Scope :
  - Release builds; static binaries via `x86_64-unknown-linux-musl`
  - Multi-stage Docker builds; `cargo-chef` dependency-layer caching; runtime stage
    (`debian:bookworm-slim` / `distroless` / `scratch`)
  - Graceful shutdown : `axum::serve(...).with_graceful_shutdown(shutdown_signal())`,
    a signal future selecting `ctrl_c()` and Unix `SignalKind::terminate()` (SIGTERM)
  - On shutdown : close the DB pool, flush tracing; pair with `TimeoutLayer`
  - Container hygiene : bind `0.0.0.0`, env-var config
- Cross-refs : axum-impl-tracing, axum-impl-database, axum-core-async-performance

### Skill: axum-errors-handling

- Output dir : skills/source/axum-errors/axum-errors-handling/
- Research input : vooronderzoek §7; fragment C §1
- Scope :
  - The infallible-handler model : services have `Infallible` error type
  - The `AppError` enum + `IntoResponse` + `From` impls + `?` operator pattern
  - `thiserror` (structured, status-mapped enum) vs `anyhow` (newtype wrapper, orphan-rule)
  - Combining both : `thiserror` for known errors + an `anyhow` catch-all variant
  - Fallible `tower::Service` middleware : `HandleError` / `HandleErrorLayer`
- Cross-refs : axum-syntax-responses, axum-errors-extractor-rejections, axum-impl-database

### Skill: axum-errors-extractor-rejections

- Output dir : skills/source/axum-errors/axum-errors-extractor-rejections/
- Research input : vooronderzoek §7; fragment C §2; fragment A §5
- Scope :
  - The `Rejection` associated type; the `axum::extract::rejection` module
    (`JsonRejection`, `PathRejection`, `QueryRejection`, ... all `#[non_exhaustive]`)
  - Customizing rejections : `Result<T, T::Rejection>` handler arguments
  - `axum-extra` `WithRejection<E, R>` for global rejection remapping
  - Derive-based custom rejections (`#[from_request(rejection(...))]`)
  - `OptionalFromRequest` and `Option<T>` extractor behavior (0.8)
- Cross-refs : axum-syntax-extractors, axum-syntax-custom-extractors, axum-errors-handling

### Skill: axum-errors-handler-trait

- Output dir : skills/source/axum-errors/axum-errors-handler-trait/
- Research input : vooronderzoek §7; fragment A §6; LESSONS L-002
- Scope :
  - Diagnosing "the trait bound ... Handler is not satisfied"
  - The 5 root causes : non-extractor argument, body extractor not last, return type
    not `IntoResponse`, function not `async`, `!Send` future
  - `#[debug_handler]` from axum-macros : the diagnostic workflow
  - A decision tree mapping symptom to cause to fix
- Cross-refs : axum-syntax-handlers, axum-syntax-extractors, axum-core-async-performance

### Skill: axum-agents-api-review

- Output dir : skills/source/axum-agents/axum-agents-api-review/
- Research input : whole vooronderzoek; all anti-patterns
- Scope :
  - A review checklist for Axum code, cross-referencing the other 28 skills
  - Verify : extractor ordering, path syntax matches target version, handler validity,
    middleware ordering, no blocking calls, no per-request pools, error handling,
    secrets from env, graceful shutdown present
  - Output : structured findings with severity + the skill that covers the fix
- Cross-refs : all skills

### Skill: axum-agents-scaffold

- Output dir : skills/source/axum-agents/axum-agents-scaffold/
- Research input : whole vooronderzoek
- Scope :
  - Orchestrate building a complete Axum service from a requirement
  - Decision sequence : routing -> state -> extractors -> middleware -> errors ->
    auth -> database -> testing -> deployment
  - Which skill to consult at each step; a Cargo.toml dependency baseline
  - A minimal-to-production checklist
- Cross-refs : all skills

---

## Phase 5 Orchestration

Per BOOTSTRAP-RUNBOOK §6 : `tmux-orchestration` skill, 3 workers, VS Code panels,
quality-gate every reply, Nederlands reply-language, `skill-builder` role.

Batch loop : (1) in-process opus topic-research for the batch's 3 skills ->
(2) verify topic-research files exist -> (3) `tmo task add` + bundle injection to
worker-1/2/3 -> (4) quality-gate loop -> (5) all workers done -> (6) run validators ->
(7) commit `feat(phase-5): batch-N skills [...]` -> (8) update ROADMAP -> (9) reflection.
Respawn workers between batches (L-007 context overflow protection).

STOP after Phase 5. No GitHub push, no remote (per session instruction).
