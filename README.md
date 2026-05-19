# Axum : Claude Skill Package

<p align="center">
  <img src="docs/social-preview.png" alt="0 Deterministic Skills for Axum" width="100%">
</p>

![Claude Code Ready](https://img.shields.io/badge/Claude_Code-Ready-blue?style=flat-square)
![Axum](https://img.shields.io/badge/Axum-0.7,0.8-0A66C2?style=flat-square)
![Skills](https://img.shields.io/badge/Skills-0-green?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)
![Agent Skills](https://img.shields.io/badge/agent--skills-compatible-purple?style=flat-square)

**0 deterministic Claude AI skills for Axum. Deterministic Claude skills for Axum : Rust async web framework, routing, extractors, middleware, tower stack, sqlx integration. Companion to Rust and PostgreSQL**

Built on the [Agent Skills](https://agentskills.org) open standard. Discoverable via npm-agentskills manifest and OpenAI Codex skill discovery.

## Why This Exists

Without skills, Claude lacks deterministic guidance for Axum patterns:

```{{LANGUAGE}}
// Wrong : {{WRONG_PATTERN_DESCRIPTION}}
{{WRONG_CODE_EXAMPLE}}
```

With this skill package, Claude produces correct patterns:

```{{LANGUAGE}}
// Correct : {{CORRECT_PATTERN_DESCRIPTION}}
{{CORRECT_CODE_EXAMPLE}}
```

## What's Inside

| Category | Count | Purpose |
|----------|:-----:|---------|
| **core/** | 0 | Architecture, cross-cutting concerns |
| **syntax/** | 0 | API syntax, code patterns, signatures |
| **impl/** | 0 | Step-by-step development workflows |
| **errors/** | 0 | Error handling, debugging, anti-patterns |
| **agents/** | 0 | Validation, code generation, orchestration |
| **Total** | **0** | |

See [INDEX.md](INDEX.md) for the complete skill catalog with descriptions and dependency graph.

## Installation

### Claude Code (recommended)

```bash
# Clone the full package
git clone https://github.com/OpenAEC-Foundation/Axum-Claude-Skill-Package.git
cp -r Axum-Claude-Skill-Package/skills/source/ ~/.claude/skills/axum/
```

### As git submodule

```bash
git submodule add https://github.com/OpenAEC-Foundation/Axum-Claude-Skill-Package.git .claude/skills/axum
```

### Via npm-agentskills standard

```bash
npx skills add @openaec/axum-claude-skill-package
```

### Claude.ai (web)

Upload individual SKILL.md files as project knowledge.

## Skill Structure

Every skill follows 3-level progressive disclosure:

```
axum-{category}-{topic}/
├── SKILL.md              # Main guidance (< 500 lines)
└── references/
    ├── methods.md        # Complete API signatures
    ├── examples.md       # Working code examples
    └── anti-patterns.md  # What NOT to do (with explanations)
```

YAML frontmatter uses folded scalar `>`, "Use when..." opener, and a `Keywords:` line with technical + symptom-based + plain-language terms for maximum discoverability.

## Quality Guarantees

- **Deterministic language** : ALWAYS / NEVER, no "you might consider"
- **Version-explicit code** : every example annotated with applicable versions
- **WebFetch-verified** : all code-snippets validated against official docs
- **CI/CD validated** : frontmatter, line count, structure, language, em-dash checks on every push
- **Compliance audit** : score >= 90% required for releases

## Companion Skills : Cross-Technology Integration

For projects combining Axum with other AEC technologies, see [Cross-Tech-AEC-Claude-Skill-Package](https://github.com/OpenAEC-Foundation/Cross-Tech-AEC-Claude-Skill-Package).

## Related Skill Packages (OpenAEC Foundation)

| Package | Skills | Repo |
|---------|--------|------|
| Blender-Bonsai-ifcOpenshell-Sverchok | 73 | [Link](https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package) |
| Frappe | 61 | [Link](https://github.com/OpenAEC-Foundation/Frappe_Claude_Skill_Package) |
| Speckle | 25 | [Link](https://github.com/OpenAEC-Foundation/Speckle-Claude-Skill-Package) |

See full list at [OpenAEC-Foundation](https://github.com/OpenAEC-Foundation).

## License

MIT : OpenAEC Foundation

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Built with the [Skill Package Workflow Template](https://github.com/OpenAEC-Foundation/Skill-Package-Workflow-Template) methodology.
