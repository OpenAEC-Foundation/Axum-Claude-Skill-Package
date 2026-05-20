# Changelog

All notable changes to the Axum Skill Package.

Format follows [Keep a Changelog](https://keepachangelog.com/).

## [1.0.0] - 2026-05-20

### Added
- 29 deterministic Claude skills for Axum 0.7 and 0.8, across 5 categories:
  - **core** (5): architecture, router, state, async-performance, version-migration
  - **syntax** (4): extractors, custom-extractors, handlers, responses
  - **impl** (15): middleware, tower-stack, static-files, websockets, sse, file-upload,
    auth-jwt, auth-session, authorization, database, validation, testing, tracing,
    openapi, deployment
  - **errors** (3): handling, extractor-rejections, handler-trait
  - **agents** (2): api-review, scaffold
- Each skill: a SKILL.md plus methods.md, examples.md, anti-patterns.md reference files
- Deep research basis: `docs/research/vooronderzoek-axum.md` plus per-skill topic research,
  all WebFetch-verified against docs.rs and the tokio-rs/axum repository
- Discovery manifests: `package.json` agents.skills[], `agents/openai.yaml`, `INDEX.md`
- Compliance audit: 100% (frontmatter, line count, structure, language checks)

### Project
- Built with the 7-phase research-first methodology (Skill Package Workflow Template)
- Phase 5 skill creation via tmux-orchestration with 3 workers and a per-batch quality gate
