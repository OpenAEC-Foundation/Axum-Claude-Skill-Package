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

---

## L-004: generate-index.js miscategorizes prefixed category directories

- **Date**: 2026-05-20
- **Context**: Phase 6.5, INDEX.md generation.
- **Finding**: `scripts/generate-index.js` expects skill directories at
  `skills/source/{category}/` but this package uses `skills/source/axum-{category}/`.
  The script put all 29 skills under an "Other" bucket and reported 0 per category.
- **How to apply**: After running `generate-index.js`, verify the per-category counts.
  When directories are prefixed (`axum-core`, not `core`), regenerate INDEX.md by hand
  or patch the script. Workflow-Template candidate: make the script strip a known prefix.

---

## L-005: generate-banner-png.js needs a Chrome executable

- **Date**: 2026-05-20
- **Context**: Phase 7, social preview banner.
- **Finding**: `generate-banner-png.js` launches Puppeteer without an `executablePath`,
  so it fails when Puppeteer's bundled Chrome is not installed. `npx puppeteer browsers
  install chrome` was blocked by the sandbox.
- **How to apply**: A system Chrome (`/usr/bin/google-chrome`) works via a one-off
  Puppeteer script with `executablePath` set. Workflow-Template candidate: have
  `generate-banner-png.js` fall back to a system Chrome when the bundled one is absent.
