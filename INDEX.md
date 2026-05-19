# INDEX : Axum Skill Package Catalog

## Overview

0 skills across 5 categories for Axum 0.7,0.8.

## Summary

| Category | Skills | Focus |
|----------|:------:|-------|
| **core** | 0 | Architecture, cross-cutting concerns |
| **syntax** | 0 | API syntax, signatures, code patterns |
| **impl** | 0 | Development workflows, end-to-end implementations |
| **errors** | 0 | Error handling, debugging, anti-patterns |
| **agents** | 0 | Validation, code generation, orchestration |
| **Total** | **0** | |

## Core Skills (0)

| Skill | Description | Dependencies |
|-------|-------------|--------------|
| `axum-core-{topic}` | {{DESCRIPTION_FROM_FRONTMATTER}} | None |

## Syntax Skills (0)

| Skill | Description | Dependencies |
|-------|-------------|--------------|
| `axum-syntax-{topic}` | {{DESCRIPTION_FROM_FRONTMATTER}} | core-{topic} |

## Implementation Skills (0)

| Skill | Description | Dependencies |
|-------|-------------|--------------|
| `axum-impl-{topic}` | {{DESCRIPTION_FROM_FRONTMATTER}} | syntax-{topic} |

## Error Skills (0)

| Skill | Description | Dependencies |
|-------|-------------|--------------|
| `axum-errors-{topic}` | {{DESCRIPTION_FROM_FRONTMATTER}} | impl-{topic} |

## Agent Skills (0)

| Skill | Description | Dependencies |
|-------|-------------|--------------|
| `axum-agents-{topic}` | {{DESCRIPTION_FROM_FRONTMATTER}} | ALL |

## Dependency Graph

```
                    ┌─────────────────────┐
                    │   core-{topic}      │
                    └──────┬──────────────┘
                           │
                  ┌────────▼────────┐
                  │  syntax-{topic} │
                  └────────┬────────┘
                           │
                  ┌────────▼────────┐
                  │   impl-{topic}  │
                  └────────┬────────┘
                           │
                  ┌────────▼────────┐
                  │  errors-{topic} │
                  └────────┬────────┘
                           │
                  ┌────────▼────────┐
                  │  agents-{topic} │
                  └─────────────────┘
```

## Discovery

- npm-agentskills manifest : [package.json](package.json) `agents.skills[]`
- OpenAI Codex : [agents/openai.yaml](agents/openai.yaml)
- GitHub topic : `agentskills`

This INDEX.md is generated from skills frontmatter. To regenerate after adding skills :

```bash
node /path/to/Skill-Package-Workflow-Template/scripts/generate-index.js .
```
