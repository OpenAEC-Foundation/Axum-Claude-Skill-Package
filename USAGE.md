# Usage : Axum Skill Package

## How skills activate

Each skill has a `description` field starting with "Use when...". Claude matches the user's prompt against all skill descriptions and loads the most relevant ones. You do not invoke skills manually : describe what you want in natural language.

## Typical workflows

### Workflow 1 : {{COMMON_TASK_1}}

Ask Claude :

> {{EXAMPLE_PROMPT_1}}

Claude will load : `axum-{{CAT}}-{{TOPIC}}` and produce code following the skill guidance.

### Workflow 2 : {{COMMON_TASK_2}}

Ask Claude :

> {{EXAMPLE_PROMPT_2}}

Claude loads : `axum-{{CAT}}-{{TOPIC_2}}`.

## Discovering which skill applies

Browse [INDEX.md](INDEX.md) for the full catalog with descriptions and dependency graph.

Or grep frontmatter Keywords :

```bash
grep -r "Keywords:" skills/source/ | grep -i "{{YOUR_TERM}}"
```

## When multiple skills apply

Claude loads multiple skills if descriptions match. Skills are designed to compose : `core` provides architecture, `syntax` adds API patterns, `impl` chains them into workflows.

## Anti-patterns triggered by error messages

Many `errors-*` skills activate on specific error-message strings. Paste the error verbatim and Claude loads the right diagnostic skill.

## Cross-tech usage

For multi-tech stacks : install the relevant single-tech packages plus [Cross-Tech-AEC-Claude-Skill-Package](https://github.com/OpenAEC-Foundation/Cross-Tech-AEC-Claude-Skill-Package) for boundary integration patterns.
