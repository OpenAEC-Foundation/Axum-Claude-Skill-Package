# WAY OF WORK

## Overview
This project follows the 7-phase research-first methodology proven in three production skill packages (ERPNext 28 skills, Blender-Bonsai 73 skills, Tauri 2 27 skills). The methodology ensures deterministic, high-quality skills by mandating deep research before any skill creation.

**Core principle**: You cannot create deterministic skills for something you don't deeply understand.

**Full methodology**: See https://github.com/OpenAEC-Foundation/Skill-Package-Workflow-Template/blob/main/WORKFLOW.md

## The 7 Phases

### Phase 1: Raw Masterplan
- Define project scope and Axum coverage areas
- Create preliminary skill inventory (estimate, not final)
- Set up repository structure and core files
- Output: `docs/masterplan/{{TECH_PREFIX}}-masterplan.md`, all core files

### Phase 2: Deep Research (Vooronderzoek)
- One comprehensive research document for Axum
- Cover: API surface, architecture, patterns, anti-patterns, version differences
- Minimum 2000 words, verify with WebFetch
- Output: `docs/research/vooronderzoek-{{TECH_PREFIX}}.md`

### Phase 3: Masterplan Refinement
- Review research against preliminary inventory
- Add, merge, or remove skills based on findings
- Define dependencies and batch execution order
- Write ready-to-use agent prompts for each skill
- Output: updated masterplan (definitive)

### Phase 4: Topic-Specific Research
- Per-skill focused research (concurrent with Phase 5)
- Only the information that specific skill needs
- Verify against official docs using WebFetch
- Output: `docs/research/topic-research/{skill-name}-research.md`

### Phase 5: Skill Creation
- Transform research into deterministic skills
- Execute in batches of 3 agents via Claude Code Agent tool
- Quality gate after every batch
- Output: `skills/source/{{TECH_PREFIX}}-{category}/{skill-name}/`

### Phase 6: Validation
- Structural, content, cross-reference, and functional validation
- All skills must pass quality gate before publication
- Output: validation report

### Phase 7: Publication
- INDEX.md, updated README.md, social preview banner
- GitHub remote under OpenAEC Foundation
- Release tag (v1.0.0)

## Skill Structure

### Directory Layout
```
skill-name/
├── SKILL.md              # Main file, < 500 lines
└── references/
    ├── methods.md        # Complete API signatures
    ├── examples.md       # Working code examples
    └── anti-patterns.md  # What NOT to do
```

### Naming Convention
- `{{TECH_PREFIX}}-{category}-{topic}`
- Categories: syntax, impl, errors, core, agents

## Content Standards

### DO:
- Use deterministic language: "ALWAYS use X when Y" / "NEVER do X because Y"
- Verify ALL code against official documentation (WebFetch)
- Version-explicit code examples
- Document anti-patterns with explanations
- Keep SKILL.md under 500 lines

### DON'T:
- Vague language: "you might consider" is BANNED
- Assumptions about API behavior
- Outdated or unverified code
- Non-English content
- Training data without WebFetch verification

## Orchestration Model

- Main session = ORCHESTRATOR (coordinates, validates)
- Agents = WORKERS (research, write, validate)
- 3 agents per batch, quality gate between batches
- Each agent writes to unique directory (no conflicts)

## Version Control
- Commit after EVERY phase: `Phase X.Y: [action] [subject]`
- ROADMAP.md updated with every commit
- Push to GitHub after every phase
