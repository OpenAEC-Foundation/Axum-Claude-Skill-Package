# Sources — Axum Skill Package

## Approved Sources

All skill content MUST be verified against these approved sources. No unverified blog posts or AI-generated content.

### Primary Sources

| Source | URL | Type | Last Verified |
|--------|-----|------|---------------|
| Axum docs (docs.rs) | https://docs.rs/axum/latest/axum/ | Official API Documentation | 2026-05-20 |
| Axum repository | https://github.com/tokio-rs/axum | GitHub Repository | 2026-05-20 |
| Axum examples | https://github.com/tokio-rs/axum/tree/main/examples | Official Examples | 2026-05-20 |
| Axum CHANGELOG | https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md | Version History | 2026-05-20 |
| tower docs | https://docs.rs/tower/latest/tower/ | Official API Documentation | 2026-05-20 |
| tower-http docs | https://docs.rs/tower-http/latest/tower_http/ | Official API Documentation | 2026-05-20 |
| tokio docs | https://docs.rs/tokio/latest/tokio/ | Official API Documentation | 2026-05-20 |
| hyper docs | https://docs.rs/hyper/latest/hyper/ | Official API Documentation | 2026-05-20 |

### Secondary Sources (companion crates, verified during Phase 2)

| Source | URL | Type | Last Verified |
|--------|-----|------|---------------|
| axum-extra docs | https://docs.rs/axum-extra/latest/axum_extra/ | Cookies, TypedHeader, WithRejection | 2026-05-20 |
| axum-macros docs | https://docs.rs/axum-macros/latest/axum_macros/ | debug_handler, derive(FromRequest) | 2026-05-20 |
| sqlx docs | https://docs.rs/sqlx/latest/sqlx/ | Database (pool, queries, transactions) | 2026-05-20 |
| axum-test docs | https://docs.rs/axum-test/latest/axum_test/ | Testing (TestServer) | 2026-05-20 |
| utoipa docs | https://docs.rs/utoipa/latest/utoipa/ | OpenAPI generation | 2026-05-20 |
| validator docs | https://docs.rs/validator/latest/validator/ | Request validation | 2026-05-20 |
| tracing docs | https://docs.rs/tracing/latest/tracing/ | Structured logging | 2026-05-20 |
| jsonwebtoken docs | https://docs.rs/jsonwebtoken/latest/jsonwebtoken/ | JWT auth | 2026-05-20 |
| tokio 0.8 announcement | https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0 | v0.8 breaking changes | 2026-05-20 |

### Ecosystem crates (reference / mention only)

`oauth2`, `tower-sessions`, `axum-login`, `tower_governor`, `garde`, `cargo-chef`,
`opentelemetry`, `tracing-opentelemetry`. Cite their docs.rs page when a skill needs
exact API detail; otherwise mention as ecosystem standard.

## Verification Rules

1. **Primary sources ONLY**: Official docs > source code > official examples
2. **NEVER use**: Random blog posts, unverified StackOverflow answers, AI-generated content without verification
3. **Version-check**: Ensure source matches target versions (Axum 0.7, 0.8)
4. **Date-check**: Note last verification date per source
5. **Cross-reference**: If official docs are sparse, verify against source code and examples
6. **WebFetch**: ALWAYS use WebFetch to verify against latest official documentation (D-006)

## Verification caveat (Phase 2)

WebFetch's summarization model could not reliably transcribe exact CHANGELOG release
*dates*. Release dates (Axum 0.7.0 = 2023-11-27, 0.8.0 = 2024-12-02) are cross-checked
knowledge; all API-shape and breaking-change claims are verified against fetched
official content.

## Source Addition Protocol

When discovering a new source during research:
1. Verify it's official or maintained by the core team (tokio-rs org)
2. Add to the appropriate table above
3. Set "Last Verified" to current date
4. Record in LESSONS.md if the source revealed significant insights
