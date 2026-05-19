# Axum Claude Skill Package

## Standing Orders — READ THIS FIRST

**Mission**: Build a complete, production-ready skill package for Axum and publish it under the OpenAEC Foundation on GitHub. This is your standing order for every session in this workspace.

**How**: Follow the 7-phase research-first methodology. Delegate ALL execution to agents. You are the ARCHITECT — you think, plan, validate, and delegate. Agents do the actual work.

**What you do on session start**:
1. Read ROADMAP.md → determine current phase and next steps
2. Read all core files (LESSONS.md, DECISIONS.md, REQUIREMENTS.md, SOURCES.md)
3. Continue where the previous session left off
4. If Phase 1 is incomplete → create the raw masterplan first
5. If Phase 2+ → follow the methodology, delegating in batches of 3 agents

**Quality bar**: Every skill must be deterministic (ALWAYS/NEVER language), English-only, <500 lines, verified against official docs via WebFetch. No hallucinated APIs. No vague language.

**End state**: A published GitHub repo at `https://github.com/OpenAEC-Foundation/{{GITHUB_REPO_NAME}}` with:
- All skills created, validated, and organized
- INDEX.md with complete skill catalog
- README.md with installation instructions and skill table
- Social preview banner (1280x640px) with OpenAEC branding
- Release tag (v1.0.0) and GitHub release
- Repository topics set (claude, skills, {{TECH_TOPIC}}, ai, deterministic, openaec)

**Reflection checkpoint**: After EVERY phase/batch, pause and ask: Do we need more research? Should we revise the plan? Are we meeting quality standards? Update core files before proceeding.

**Consolidate lessons**: Any workflow-level insight (not tech-specific) should also be noted for consolidation back to the Workflow Template repo (`{{WORKFLOW_TEMPLATE_PATH}}` : default `/home/freek/GitHub/Skill-Package-Workflow-Template`).

**Self-audit**: At Phase 6 or any time quality is in question, use Protocol P-010 to run a self-audit against the methodology. The audit template and CI/CD pipeline are in the Workflow Template repo.

**Masterplan template**: When creating your masterplan in Phase 3, follow the EXACT structure from:
- Template: `{{WORKFLOW_TEMPLATE_PATH}}/templates/masterplan.md.template`
- Proven example (single-tech, 27 skills, 10 batches, one session) : https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package/blob/main/docs/masterplan/tauri-masterplan.md
- Proven example (single-tech, 21 skills, 7 batches) : https://github.com/OpenAEC-Foundation/n8n-Claude-Skill-Package/blob/main/docs/masterplan/n8n-masterplan.md
- Proven example (multi-tech, 73 skills, 13+ batches) : https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package/blob/main/docs/masterplan/masterplan.md

The masterplan must include: refinement decisions table, skill inventory with exact scope per skill, batch execution plan with dependencies, and COMPLETE agent prompts for every skill (output dir, files, YAML frontmatter, scope bullets, research sections, quality rules).

**Reference projects** (study these for methodology, not content):
- ERPNext (28 skills): https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package
- Blender-Bonsai (73 skills): https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
- Tauri 2 (27 skills): https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package

---

## Project Identity
- Axum skill package for Claude — {{TECH_DESCRIPTION}}
- Technology: {{TECH_NAME}} {{TECH_VERSIONS}}
- Methodology: 7-phase research-first development (proven in ERPNext, Blender-Bonsai, and Tauri packages)
- Workflow reference: https://github.com/OpenAEC-Foundation/Skill-Package-Workflow-Template
- Reference projects:
  - ERPNext: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package
  - Blender-Bonsai: https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package
  - Tauri 2: https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package

## Core Files Map
| File | Domain | Role |
|------|--------|------|
| ROADMAP.md | Status | Single source of truth for project status, progress, next steps |
| LESSONS.md | Knowledge | Numbered lessons (L-XXX) discovered during development |
| DECISIONS.md | Architecture | Numbered decisions (D-XXX) with rationale, immutable once recorded |
| REQUIREMENTS.md | Scope | What skills must achieve, quality guarantees |
| SOURCES.md | References | Official documentation URLs, verification rules, last-verified dates |
| WAY_OF_WORK.md | Methodology | 7-phase process, skill structure, content standards |
| CHANGELOG.md | History | Version history in Keep a Changelog format |
| docs/masterplan/{{TECH_PREFIX}}-masterplan.md | Planning | Execution plan with phases, prompts, dependencies |
| README.md | Public | GitHub landing page |
| HANDOFF.md | Overdracht | Quick-start guide for new sessions, batch volgorde, bijzonderheden |
| INDEX.md | Catalog | Complete skill catalog with descriptions and dependency graph |

## Technology Scope
| Tech | Prefix | Versions |
|------|--------|----------|
| {{TECH_NAME}} | {{TECH_PREFIX}}- | {{TECH_VERSIONS}} |

## Skill Categories
| Category | Purpose | Naming |
|----------|---------|--------|
| syntax/ | API syntax, code patterns | {{TECH_PREFIX}}-syntax-{topic} |
| impl/ | Development workflows | {{TECH_PREFIX}}-impl-{topic} |
| errors/ | Error handling patterns | {{TECH_PREFIX}}-errors-{topic} |
| core/ | Cross-cutting concerns, architecture | {{TECH_PREFIX}}-core-{topic} |
| agents/ | Intelligent orchestration | {{TECH_PREFIX}}-agents-{topic} |

## Repository Structure
```
project-root/
├── CLAUDE.md                    # THIS FILE
├── ROADMAP.md                   # Status (single source of truth)
├── REQUIREMENTS.md              # Quality guarantees
├── DECISIONS.md                 # Architectural decisions
├── SOURCES.md                   # Official reference URLs
├── WAY_OF_WORK.md               # 7-phase methodology
├── LESSONS.md                   # Lessons learned
├── CHANGELOG.md                 # Version history
├── README.md                    # GitHub landing page
├── docs/
│   ├── masterplan/              # {{TECH_PREFIX}}-masterplan.md
│   └── research/                # vooronderzoek-{{TECH_PREFIX}}.md, topic-research/, fragments/
└── skills/
    └── source/
        ├── {{TECH_PREFIX}}-syntax/
        ├── {{TECH_PREFIX}}-impl/
        ├── {{TECH_PREFIX}}-errors/
        ├── {{TECH_PREFIX}}-core/
        └── {{TECH_PREFIX}}-agents/
```

---

## Privacy Protocol (P-000a)
**PROMPTS.md is PRIVATE** — it contains user session prompts and internal agent task data.

1. PROMPTS.md MUST be listed in `.gitignore` — NEVER commit or push it to GitHub
2. `.claude/` directory MUST be listed in `.gitignore`
3. `*.code-workspace` files MUST be listed in `.gitignore`
4. Before ANY `git push`, verify that `git status` does NOT show PROMPTS.md as staged
5. If PROMPTS.md was accidentally committed, remove it: `git rm --cached PROMPTS.md`

---

## Workspace Setup Protocol (P-000b)
On FIRST session in a new workspace, ensure permissions are configured for autonomous operation:

1. **Verify Bypass Permissions** — Check that `.claude/settings.json` has permissions allowing autonomous execution:
   ```json
   { "permissions": { "allow": ["Bash(*)", "Read", "Write", "Edit", "Glob", "Grep", "WebFetch", "WebSearch", "Agent"] } }
   ```
   If not configured, create `.claude/settings.json` with these permissions.
2. **Verify .gitignore** — Ensure PROMPTS.md, .claude/, and *.code-workspace are in `.gitignore`.
3. This enables agents to work without manual approval per tool call — critical for the batch delegation model.

---

## Session Start Protocol (P-001)
EVERY session begins with this sequence:

1. **Read ROADMAP.md** — Determine current phase, progress percentage, and "Next Steps" section
2. **Read LESSONS.md** — Check recent lessons that may affect your work
3. **Read DECISIONS.md** — Know all architectural decisions (D-001+) and their constraints
4. **Read REQUIREMENTS.md** — Understand quality guarantees and per-area requirements
5. **Read docs/masterplan/{{TECH_PREFIX}}-masterplan.md** — Know the execution plan and current phase details
6. **If researching**: Read **SOURCES.md** — Know approved sources, verification rules
7. **If creating skills**: Read **WAY_OF_WORK.md** — Know skill structure, content standards, naming
8. Identify next action from ROADMAP.md "Next Steps"
9. Confirm with user before proceeding

---

## Meta-Orchestrator Protocol (P-002)

### Identity
This Claude Code session + the human user together ARE the **meta-orchestrator**.
We are NOT a relay/passthrough. We are the strategic brain.

**What we do HERE (the brain):**
- THINK: Analyze problems, design solutions, make architectural decisions
- STRATEGIZE: Plan agent batches, define task decomposition, choose approaches
- DECIDE: Accept/reject agent output, resolve conflicts, set direction
- COMPOSE: Craft precise agent prompts with full context from core files

**What agents do THERE (the hands):**
- EXECUTE: Research, write, code, validate — the actual work
- CROSS-VALIDATE: Agents check each other's output before it comes back to us
- REPORT: Deliver refined, verified output to the meta-orchestrator

### Rules:
- Delegate EXECUTION via Claude Code Agent tool — thinking stays here
- Validate before accepting (validator-before-apply)
- Strategic reasoning, planning, and decision-making happen in THIS session
- Agents receive complete context (core file references) so they can work autonomously

### What to include in EVERY agent prompt:
- Quality criteria from **REQUIREMENTS.md** (relevant to their task)
- Approved source URLs from **SOURCES.md** (what docs to consult)
- Current status from **ROADMAP.md** (what's done, what's needed)
- Relevant constraints from **DECISIONS.md**
- Skill structure from **WAY_OF_WORK.md** (if writing skills)

### Delegation Flow:
1. **Think** — Define task scope, expected output, success criteria
2. **Compose** — Write task prompt with core file references (see above)
3. **Spawn Agent** — Use Claude Code Agent tool with complete prompt
4. **Collect** — Receive agent output automatically
5. **Judge** — VALIDATE output against **REQUIREMENTS.md** quality criteria
6. **Iterate** — Accept, or respawn with corrections

### Batch Strategy:
- 3 agents per batch (optimal for Claude Code Agent tool)
- Separated file scopes (NEVER two agents on same file)
- Quality gate after every batch
- Cross-validation: review agent output before final acceptance

---

## Quality Control Protocol (P-003)
### Validation criteria sourced from core files:

**From REQUIREMENTS.md:**
- Skill format requirements (YAML frontmatter, structure)
- Technology version coverage
- Language/framework specific requirements

**From DECISIONS.md:**
- D-001: English-only content
- D-002: MIT License
- D-003: SKILL.md < 500 lines

**From SOURCES.md:**
- All code verified against listed official sources only
- No unverified blog posts or outdated content

### Validator-Before-Apply checklist:
1. File exists and is complete
2. YAML frontmatter valid (name, description with trigger words)
3. Line count < 500 (SKILL.md)
4. English-only (no Dutch or other languages)
5. Deterministic language (ALWAYS/NEVER, not "you might consider")
6. All references/ files exist and are linked from SKILL.md
7. Sources traceable to **SOURCES.md** approved URLs

### Correction Flow:
If validation fails:
1. Document what failed in agent feedback
2. Spawn fix-agent with specific correction instructions
3. Re-validate after fix
4. NEVER accept below quality bar defined in **REQUIREMENTS.md**

---

## Research Protocol (P-004)
### Before ANY research:
1. Read **SOURCES.md** — Know approved sources
2. Read **REQUIREMENTS.md** — Know what the research must cover
3. Read **DECISIONS.md** — Know constraints

### During research:
- Use ONLY sources listed in **SOURCES.md** (or add new ones there)
- Verify code examples against official documentation
- Identify anti-patterns from real GitHub issues
- Use WebFetch to ensure latest documentation is consulted

### After research:
1. Update **SOURCES.md** "Last Verified" table with verification date
2. Log new discoveries in **LESSONS.md** (numbered L-XXX)
3. If new architectural decisions emerge, record in **DECISIONS.md** (numbered D-XXX)

### Research output location:
- Vooronderzoek: `docs/research/vooronderzoek-{{TECH_PREFIX}}.md`
- Topic research: `docs/research/topic-research/{skill-name}-research.md`
- Research fragments: `docs/research/fragments/`

---

## Skill Standards (P-005)
Defined in detail in **WAY_OF_WORK.md** and **REQUIREMENTS.md**. Quick reference:

- English-only (per **DECISIONS.md** D-001)
- Deterministic: "ALWAYS use X when Y" / "NEVER do X because Y"
- SKILL.md < 500 lines (per **DECISIONS.md** D-003), heavy content in references/
- YAML frontmatter: name + description with trigger words
- Structure: Quick Reference > Decision Trees > Patterns > Reference Links
- Naming: `{{TECH_PREFIX}}-{category}-{topic}`
- Verify against **SOURCES.md** approved URLs only

### SKILL.md YAML Frontmatter — Required Format

```yaml
---
name: axum-{{CATEGORY}}-{{TOPIC}}
description: >
  Use when [specific trigger scenario].
  Prevents the [common mistake / anti-pattern].
  Covers [key topics, API areas, version differences].
  Keywords: [comma-separated technical terms].
license: MIT
compatibility: "Designed for Claude Code. Requires Axum 0.7,0.8."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---
```

**CRITICAL FORMAT RULES:**
- Description MUST use folded block scalar `>` (NEVER quoted strings)
- Description MUST start with "Use when..."
- Description MUST include "Keywords:" line
- Name MUST be kebab-case, max 64 characters

---

## Reflection Checkpoint Protocol (P-006a)
MANDATORY after EVERY completed phase/batch — PAUSE and answer:

1. **Research sufficiency**: Did this phase reveal gaps? Do we need more research?
2. **Scope reassessment**: Should we add, merge, or remove skills?
3. **Plan revision**: Does the masterplan still make sense? Change batch order?
4. **Quality reflection**: Are we meeting our quality bar consistently?
5. **New discoveries**: Anything for LESSONS.md or DECISIONS.md?

If ANY answer is "yes" → update core files BEFORE continuing. If research needs expanding → return to Phase 2 or 4.

---

## Document Sync Protocol (P-006)
After EVERY completed phase/batch, update these files:

1. **REFLECTION CHECKPOINT** — Answer the 5 questions above (MANDATORY)
2. **ROADMAP.md** — Status, percentage, changelog entry, next steps (MANDATORY)
3. **LESSONS.md** — New patterns or discoveries (if any)
4. **DECISIONS.md** — New architectural decisions (if any)
5. **SOURCES.md** — New sources verified or dates updated (if researching)
6. **CHANGELOG.md** — Milestone entries (for significant completions)
7. Commit with message: `Phase X.Y: [action] [subject]`
8. **README.md** — Check if landing page needs updating

Timing: IMMEDIATE after completion, not deferred.

---

## Session End Protocol (P-007)
Before ending ANY session:

1. **ROADMAP.md** — Update current phase status + "Next Steps" section (CRITICAL)
2. **LESSONS.md** — Log anything learned during this session
3. **DECISIONS.md** — Record any decisions made
4. **CHANGELOG.md** — Add entry if milestone reached
5. Commit all changes with descriptive message
6. Verify **README.md** reflects current project state

---

## Inter-Agent Protocol (P-008)
Agents are spawned via the Agent tool within Claude Code. Results are collected automatically when the agent completes.

### How it works:
- Meta-orchestrator composes a prompt with full context
- Agent tool spawns a subagent with that prompt
- Subagent executes and returns results
- Meta-orchestrator validates output against REQUIREMENTS.md
- Accept or respawn with corrections

---

## GitHub Publication Protocol (P-009)

### Repository Setup:
```bash
# Create remote under OpenAEC Foundation
gh repo create OpenAEC-Foundation/{{GITHUB_REPO_NAME}} --public \
  --description "Deterministic Claude skills for Axum"

# Set remote and push
git remote add origin https://github.com/OpenAEC-Foundation/{{GITHUB_REPO_NAME}}.git
git push -u origin main

# Set topics
gh repo edit --add-topic claude,skills,{{TECH_TOPIC}},ai,deterministic,openaec
```

### Social Preview Banner:
Create `docs/social-preview-banner.html` with:
- 1280x640px dimensions
- Technology branding (brand colors, code samples)
- Skill count prominently displayed
- OpenAEC Foundation branding (bottom-right)
- Render to PNG for GitHub social preview

### Release:
```bash
git tag -a v1.0.0 -m "v1.0.0: X deterministic skills for Axum"
git push origin v1.0.0
gh release create v1.0.0 --title "v1.0.0 — Axum Skill Package" \
  --notes "Initial release with X deterministic skills across 5 categories."
```

---

## Self-Audit Protocol (P-010)

When a skill package reaches Phase 6 (Validation) or when requested, run a self-audit.

### How to audit:
1. Read the methodology audit template from the Workflow Template repo:
   `{{WORKFLOW_TEMPLATE_PATH}}/templates/methodology-audit.md.template`
2. Read the repo status overview for cross-package context:
   `{{WORKFLOW_TEMPLATE_PATH}}/REPO-STATUS-AUDIT.md`
3. Or use the ready-made audit prompt:
   `{{WORKFLOW_TEMPLATE_PATH}}/AUDIT-START-PROMPT.md`

### CI/CD Automated Validation:
Skill packages with GitHub remotes automatically run quality checks on push/PR via:
```yaml
# .github/workflows/quality.yml
name: Skill Quality
on: [push, pull_request]
jobs:
  quality:
    uses: OpenAEC-Foundation/Skill-Package-Workflow-Template/.github/workflows/skill-quality.yml@main
```

This validates: YAML frontmatter, line count, directory structure, English-only content, and generates a compliance score.

### Manual Full Audit:
For a comprehensive audit + auto-remediation, copy the prompt from `AUDIT-START-PROMPT.md` and run it in the target repo's Claude Code session.

---

## Discoverability Protocol (P-011)

Skills are only useful if users can find them. P-011 enforces three discovery surfaces.

### Required artifacts (Phase 6.5)

| Artifact | Standard | Auto-generated by |
|----------|----------|-------------------|
| `package.json` with `agents.skills[]` | npm-agentskills | `scripts/generate-manifest.js` |
| `agents/openai.yaml` | OpenAI Codex skill discovery | `scripts/generate-manifest.js` |
| GitHub topic `agentskills` | cross-platform discovery | `gh repo edit --add-topic agentskills` |
| `INDEX.md` with description tabel | human + LLM browse | `scripts/generate-index.js` |
| `Keywords:` line in every SKILL.md frontmatter | trigger-matching | manual / polish-pass agent |

### Keywords field rule

Every SKILL.md `description.Keywords:` MUST mix three term types :

1. **Technical** : exact API names, function names, error class names
2. **Symptom-based** : "blank screen", "nothing shows", "app is slow"
3. **Plain-language** : "how do I", "what is", "getting started"

Per Frappe v3.1.1 expansion : without symptom-based keywords, skills fail to trigger on real user prompts ("my page is slow" never matches "performance optimization" only).

### Verification

`scripts/validate-frontmatter.js` enforces presence of `Keywords:` line. `scripts/validate-emdash.js` enforces colon-based headings (em-dashes break some markdown parsers and conflict with global typografie-regel).

---

## V-N Upgrade Context Protocol (P-000c, optional)

When a major-version rewrite happens inside the same repo (Frappe v2 -> v3 pattern), add a "V-N Upgrade Context" block to CLAUDE.md immediately after Standing Orders. Contents :

- What changed in V-N (rename / new skills / removed skills)
- Migration script reference (e.g. `tools/rename_skills.py`)
- Key documents updated (MIGRATION-AGENT-SKILLS-STANDARD.md, ROADMAP.md, etc.)
- Reference to the V-N release tag

Prevents new sessions from operating on stale assumptions about pkg-structure. See `https://github.com/OpenAEC-Foundation/Frappe_Claude_Skill_Package/blob/main/CLAUDE.md` (lines around V2 Upgrade Context) for reference.

---

## tmux-orchestration Protocol (P-012, packages >15 skills)

For Phase 5 skill creation with >15 skills, use `tmux-orchestration` skill instead of in-process Agent calls. Persistent workers with file-scope-isolation + quality-gate-loop scale better.

### Required workspace files (auto-created by `bootstrap-new-package.sh`)

- `.vscode/tasks.json` : worker-1/2/3 spawn tasks
- `.claude/settings.json` : bypass permissions ON
- `state/` (gitignored) : `messages.jsonl`, `sessions.yaml` created by `tmo init`

### Workflow

1. Main session invokes `tmux-orchestration` skill
2. Answers : 3 workers, VS Code panels, every-reply QG, Nederlands reply-language, `skill-builder` role
3. Per batch : 3 in-process opus agents for topic-research → bundle-inject batch to workers → quality-gate loop
4. Workers run with `TMO_SESSION=worker-N TMO_STATE_DIR=<ws>/state`
5. Quality-gate verdicts via `tmo task approve/reject`

See BOOTSTRAP-RUNBOOK.md §6 for full integration.
