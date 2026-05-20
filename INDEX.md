# INDEX : Axum Skill Package Catalog

## Overview

29 deterministic Claude skills across 5 categories for the Axum Rust web framework
(versions 0.7 and 0.8).

## Summary

| Category | Skills | Focus |
|----------|:------:|-------|
| **core** | 5 | Architecture, routing, state, performance, cross-cutting concerns |
| **syntax** | 4 | API syntax, extractors, handlers, responses |
| **impl** | 15 | Development workflows, end-to-end implementations |
| **errors** | 3 | Error handling, extractor rejections, debugging |
| **agents** | 2 | Code review and service scaffolding orchestration |
| **Total** | **29** | |

## core (5)

| Skill | Description |
|-------|-------------|
| `axum-core-architecture` | Starting an Axum project, choosing Axum, understanding how tokio + hyper + tower compose, or debugging why a server process exits immediately. |
| `axum-core-router` | Defining routes, mounting sub-routers, method-routers, fallbacks, and the 404-vs-405 distinction; fixing startup routing panics. |
| `axum-core-state` | Wiring shared application state (DB pools, config, caches) through `with_state`, the `State` extractor, and `FromRef` substates. |
| `axum-core-async-performance` | An Axum service is slow, freezes, or stalls under load; running CPU-bound or blocking work without starving the Tokio runtime. |
| `axum-core-version-migration` | Upgrading an Axum project from 0.7 to 0.8: the complete breaking-change matrix and an ordered migration checklist. |

## syntax (4)

| Skill | Description |
|-------|-------------|
| `axum-syntax-extractors` | Reading request data with the built-in extractors: `Path`, `Query`, `Json`, `Form`, `State`, `Extension`, headers, body. |
| `axum-syntax-custom-extractors` | Implementing a custom extractor with `FromRequest` / `FromRequestParts`, including the 0.7-vs-0.8 `async_trait` change. |
| `axum-syntax-handlers` | Writing or fixing an Axum handler function: the argument list, return types, and the `Handler` trait bound. |
| `axum-syntax-responses` | Returning a status code, headers, JSON, HTML, raw bytes, a redirect, or a cookie through `IntoResponse`. |

## impl (15)

| Skill | Description |
|-------|-------------|
| `axum-impl-middleware` | Adding middleware with `from_fn` / `from_fn_with_state` / `map_request` / `map_response`, and getting the ordering right. |
| `axum-impl-tower-stack` | Adding tower and tower-http layers: `TraceLayer`, `CorsLayer`, `CompressionLayer`, `TimeoutLayer`, `ServiceBuilder`, rate limiting. |
| `axum-impl-static-files` | Serving static files and the single-page-app fallback with `ServeDir`, `ServeFile`, and `SetStatus`. |
| `axum-impl-websockets` | Adding WebSocket endpoints with `WebSocketUpgrade`, including the 0.8 `Message` type change and concurrent send/receive. |
| `axum-impl-sse` | Pushing a one-way server-to-client stream over plain HTTP with `Sse`, `Event`, and `KeepAlive`. |
| `axum-impl-file-upload` | Accepting `multipart/form-data` uploads with `Multipart`, streaming large files, and the `DefaultBodyLimit` footgun. |
| `axum-impl-auth-jwt` | Stateless JWT authentication: a `Claims` extractor, a login route issuing a bearer token, env-var secrets. |
| `axum-impl-auth-session` | Server-side session auth (`tower-sessions`, `axum-login`), HTTP Basic auth, and the OAuth2 authorization-code flow. |
| `axum-impl-authorization` | Role-based and permission-based access control as `route_layer` middleware and newtype guard extractors. |
| `axum-impl-database` | Wiring a `sqlx` connection pool into router state, running queries and transactions in handlers. |
| `axum-impl-validation` | Validating an incoming JSON body with the `validator` crate and a `ValidatedJson` custom extractor. |
| `axum-impl-testing` | Testing an Axum app with `axum-test` `TestServer` or the `tower::ServiceExt::oneshot` pattern. |
| `axum-impl-tracing` | Structured logging and request tracing with `tracing-subscriber`, `TraceLayer`, `#[instrument]`, and OpenTelemetry. |
| `axum-impl-openapi` | Generating an OpenAPI 3 spec and Swagger UI with `utoipa`, `utoipa-swagger-ui`, and `utoipa-axum`. |
| `axum-impl-deployment` | Production deployment: release builds, musl static binaries, multi-stage Docker, and SIGTERM-aware graceful shutdown. |

## errors (3)

| Skill | Description |
|-------|-------------|
| `axum-errors-handling` | The infallible-handler model, a custom `AppError` enum with `IntoResponse`, the `?` operator, `thiserror` vs `anyhow`. |
| `axum-errors-extractor-rejections` | Controlling the HTTP error when an extractor fails: customizing rejections, `WithRejection`, unified API error types. |
| `axum-errors-handler-trait` | Diagnosing the opaque "the trait bound ... `Handler` is not satisfied" compile error with `#[debug_handler]`. |

## agents (2)

| Skill | Description |
|-------|-------------|
| `axum-agents-api-review` | Reviewing Axum code for correctness, reliability, and security defects, cross-referencing the other 28 skills. |
| `axum-agents-scaffold` | Building a complete Axum service from scratch: the ordered build sequence and a production-readiness checklist. |

## Dependency Order

Skills are designed to be read in this order: `core` -> `syntax` -> `impl` -> `errors`
-> `agents`. The two `agents` skills cross-reference every other skill.

## Discovery

- npm-agentskills manifest : [package.json](package.json) `agents.skills[]`
- OpenAI Codex : [agents/openai.yaml](agents/openai.yaml)
- GitHub topic : `agentskills`

Skill source : `skills/source/axum-{category}/{skill-name}/SKILL.md`
