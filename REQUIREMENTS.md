# REQUIREMENTS — Axum Skill Package

## Purpose
Define what the skill package must achieve and establish quality criteria for all skills.

## Target Users
- Claude instances (primary) — skills are instructions for AI code generation
- Open-source LLMs — deterministic skills compensate for weaker training coverage
- Developers using Claude Code — skills ensure correct, best-practice code generation

## Skill Format Requirements

### Structure
- Every skill has: `SKILL.md` + `references/` directory
- SKILL.md MUST be under 500 lines
- Reference files: `methods.md`, `examples.md`, `anti-patterns.md` (minimum)
- YAML frontmatter required: `name`, `description` (with trigger words)

### Content
- English-only (skills are instructions for Claude, not end-user documentation)
- Deterministic language: "ALWAYS use X when Y" / "NEVER do X because Y"
- No vague hedging: "you might consider" is BANNED
- All code examples verified against official documentation
- Version-explicit: annotate which versions each pattern applies to
- Anti-patterns documented with "WHY this fails" explanations

### Quality Guarantees
- Every code example compiles/runs against Axum {{TECH_VERSIONS}}
- All API references verified against official docs (SOURCES.md)
- Decision trees for common architectural choices
- Cross-references between related skills

## Per-Area Requirements

### syntax/ Skills
- Complete API signatures in references/methods.md
- Working code examples in references/examples.md
- Common pitfalls in references/anti-patterns.md
- Version compatibility annotations on all patterns

### impl/ Skills
- Step-by-step workflow patterns
- Decision trees for choosing approaches
- Working end-to-end examples
- Dependencies on syntax/ skills documented

### errors/ Skills
- Common error messages with root cause analysis
- Prevention strategies (how to avoid)
- Recovery patterns (how to fix)
- Real anti-patterns from GitHub issues

### core/ Skills
- Complete API overview (all modules/classes/methods)
- Architecture explanation
- Version matrix with breaking changes
- Runtime quirks and limitations

### agents/ Skills
- Validation rules that reference other skills
- Code quality checklist
- Cross-skill consistency checks
