# Lessons Learned

Observations and findings captured during skill package development.

---

## L-001: WebFetch summarizer is unreliable for transcribing exact dates

- **Date**: 2026-05-20
- **Context**: Phase 2 research agents fetched the Axum CHANGELOG.md to extract v0.7
  and v0.8 release dates.
- **Finding**: The WebFetch summarization model returned chronologically impossible
  release dates (e.g. "0.8.1 (29. November, 2022)"). API-shape claims (signatures,
  trait bodies, breaking-change bullets) WERE transcribed correctly.
- **How to apply**: For exact dates and version numbers, cross-check rather than trust
  the WebFetch summary. For API signatures and structural facts, WebFetch is reliable.
  Skill authors should quote trait definitions and code, not parenthesized CHANGELOG
  dates.

---

## L-002: Axum's no-macros design concentrates pain into one cryptic error

- **Date**: 2026-05-20
- **Context**: Phase 2 research across all three clusters.
- **Finding**: Because handler eligibility is enforced purely by the `Handler` trait
  bound, almost every handler-shape mistake (non-extractor argument, body extractor
  not last, non-`IntoResponse` return, `!Send` future) surfaces as the same opaque
  "the trait bound ... Handler is not satisfied" error.
- **How to apply**: The errors category needs a dedicated skill for diagnosing this
  single error via `#[debug_handler]`. Every syntax skill should cross-reference it.

---

## L-003: The v0.7 -> v0.8 break is pervasive, not isolated

- **Date**: 2026-05-20
- **Context**: Phase 2 research.
- **Finding**: The 0.8 path-syntax change (`:id` -> `{id}`) affects EVERY code example
  in the package; the `#[async_trait]` removal affects EVERY custom-extractor example
  (auth, database, validation); the WebSocket `Message` type change affects realtime
  code. A single migration skill is not enough.
- **How to apply**: Every skill must annotate version-divergent code `// axum 0.8` /
  `// axum 0.7`. The migration skill carries the complete removed-API checklist.
