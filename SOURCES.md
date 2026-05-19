# Sources — Axum Skill Package

## Approved Sources

All skill content MUST be verified against these approved sources. No unverified blog posts or AI-generated content.

### Primary Sources

| Source | URL | Type | Last Verified |
|--------|-----|------|---------------|
| Axum docs (docs.rs) | https://docs.rs/axum/latest/axum/ | Official API Documentation | Not yet |
| Axum repository | https://github.com/tokio-rs/axum | GitHub Repository | Not yet |
| Axum examples | https://github.com/tokio-rs/axum/tree/main/examples | Official Examples | Not yet |
| Axum CHANGELOG | https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md | Version History | Not yet |
| tower docs | https://docs.rs/tower/latest/tower/ | Official API Documentation | Not yet |
| tower-http docs | https://docs.rs/tower-http/latest/tower_http/ | Official API Documentation | Not yet |
| tokio docs | https://docs.rs/tokio/latest/tokio/ | Official API Documentation | Not yet |
| hyper docs | https://docs.rs/hyper/latest/hyper/ | Official API Documentation | Not yet |

### Secondary Sources (use only when primary is insufficient)

| Source | URL | Type | Last Verified |
|--------|-----|------|---------------|
| sqlx docs | https://docs.rs/sqlx/latest/sqlx/ | Companion crate (DB) | Not yet |
| sqlx repository | https://github.com/launchbadge/sqlx | Companion crate (DB) | Not yet |
| axum-test docs | https://docs.rs/axum-test/latest/axum_test/ | Companion crate (testing) | Not yet |
| utoipa docs | https://docs.rs/utoipa/latest/utoipa/ | Companion crate (OpenAPI) | Not yet |
| validator docs | https://docs.rs/validator/latest/validator/ | Companion crate (validation) | Not yet |
| tracing docs | https://docs.rs/tracing/latest/tracing/ | Companion crate (logging) | Not yet |
| Axum tutorial blog | https://github.com/tokio-rs/axum/blob/main/ECOSYSTEM.md | Ecosystem reference | Not yet |

## Verification Rules

1. **Primary sources ONLY**: Official docs > source code > official examples
2. **NEVER use**: Random blog posts, unverified StackOverflow answers, AI-generated content without verification
3. **Version-check**: Ensure source matches target versions (Axum 0.7, 0.8)
4. **Date-check**: Note last verification date per source
5. **Cross-reference**: If official docs are sparse, verify against source code and examples
6. **WebFetch**: ALWAYS use WebFetch to verify against latest official documentation (D-006)

## Source Addition Protocol

When discovering a new source during research:
1. Verify it's official or maintained by the core team (tokio-rs org)
2. Add to the appropriate table above
3. Set "Last Verified" to current date
4. Record in LESSONS.md if the source revealed significant insights
