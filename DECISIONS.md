# Architectural Decisions

Numbered decisions (D-XXX) with rationale. Immutable once recorded — new decisions can supersede but never delete old ones.

---

## D-001: English-Only Skills

- **Date**: 2026-05-19
- **Decision**: All skill content MUST be in English only.
- **Rationale**: Skills are instructions FOR Claude, not for end users. Claude reads English and responds in the user's language. Bilingual skills double maintenance with zero functional benefit. Proven in ERPNext (28 skills) and Blender-Bonsai (73 skills).
- **Consequence**: No translations needed. All descriptions, code comments, and documentation in English.

---

## D-002: MIT License

- **Date**: 2026-05-19
- **Decision**: Project uses MIT License.
- **Rationale**: Most permissive license, maximizes adoption. Consistent with OpenAEC Foundation standards.
- **Consequence**: No commercial restrictions. Community-friendly.

---

## D-003: SKILL.md Under 500 Lines

- **Date**: 2026-05-19
- **Decision**: SKILL.md files MUST be under 500 lines.
- **Rationale**: Keeps main skill focused on decision trees and quick reference. Heavy content belongs in references/ directory. Proven optimal in ERPNext (180-427 lines per skill).
- **Consequence**: Complex topics split between SKILL.md (quick reference + patterns) and references/ (complete API, examples, anti-patterns).

---

## D-004: 7-Phase Research-First Methodology

- **Date**: 2026-05-19
- **Decision**: Follow the 7-phase research-first development methodology.
- **Rationale**: Proven in ERPNext (28 skills), Blender-Bonsai (73 skills), and Tauri 2 (27 skills). Research prevents hallucination. Deterministic skills require deep understanding.
- **Consequence**: No skill creation without prior deep research. Phases are sequential with defined exit criteria.

---

## D-005: ROADMAP.md as Single Source of Truth

- **Date**: 2026-05-19
- **Decision**: ROADMAP.md is the ONLY place for project status.
- **Rationale**: Multiple status locations cause drift and "which is current?" confusion. Single source of truth enables reliable session recovery.
- **Consequence**: Never duplicate status in CLAUDE.md or other files. All status references point to ROADMAP.md.

---

## D-006: WebFetch for Research Verification

- **Date**: 2026-05-19
- **Decision**: Use WebFetch to verify all code examples against latest official documentation.
- **Rationale**: Technology APIs evolve. Training data may be stale. WebFetch ensures latest official docs are consulted, not outdated cached knowledge.
- **Consequence**: All code examples must be verified against current official documentation before inclusion in skills.

---

## D-007: GitHub Publication Under OpenAEC Foundation

- **Date**: 2026-05-19
- **Decision**: Publish all skill packages under the OpenAEC Foundation GitHub organization.
- **Rationale**: Centralized, consistent branding. Community ownership. Discoverability.
- **Consequence**: All repos follow OpenAEC naming conventions and include social preview banners with OpenAEC branding.

---

## D-008: Final Skill Inventory : 29 Skills, 5 Categories, 10 Batches

- **Date**: 2026-05-20
- **Decision**: The Axum skill package ships 29 skills: core (5), syntax (4), impl (15),
  errors (3), agents (2). Executed in 10 batches of 3 (last batch 2).
- **Rationale**: Phase 2 research refined the raw ~28-topic inventory. Two structural
  changes (recorded in the masterplan Refinement Decisions table D-01 and D-02): SPLIT
  `axum-impl-auth` into `axum-impl-auth-jwt` + `axum-impl-auth-session` because JWT
  (stateless) and session auth (stateful, tower-sessions/axum-login) are structurally
  different and four auth methods exceed the 500-line bound; MERGE
  `axum-impl-graceful-shutdown` into `axum-impl-deployment` because graceful shutdown
  is a thin topic tightly coupled to deployment (SIGTERM = container stop signal).
  The 19 newly discovered sub-topics fold into existing skills as sections rather than
  fragmenting the catalog (masterplan D-05).
- **Consequence**: Phase 5 runs 10 batches via tmux-orchestration. Skill count is fixed
  unless a future decision supersedes this.

---

## D-009: Phase 4 Topic-Research Interleaved, Not Skipped

- **Date**: 2026-05-20
- **Decision**: Phase 4 topic-research runs as a focused supplement per batch
  (in-process opus agents writing `docs/research/topic-research/{skill}-research.md`),
  not as a full re-research and not skipped entirely.
- **Rationale**: The Phase 2 vooronderzoek is deep and WebFetch-verified, so a full
  Phase 4 re-research would be redundant (Docker L-001 skip-pattern). But each skill
  still needs exact, current API signatures and extra worked examples that the
  cluster-level vooronderzoek does not carry. A focused per-batch supplement is the
  middle path.
- **Consequence**: Before each batch the orchestrator spawns 3 in-process opus
  topic-research agents, then dispatches the batch to the tmux workers.
