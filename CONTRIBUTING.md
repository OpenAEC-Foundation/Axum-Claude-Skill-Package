# Contributing : Axum-Claude-Skill-Package

> This package follows the [Skill Package Workflow Template](https://github.com/OpenAEC-Foundation/Skill-Package-Workflow-Template) methodology. Read [BOOTSTRAP-RUNBOOK.md](https://github.com/OpenAEC-Foundation/Skill-Package-Workflow-Template/blob/main/BOOTSTRAP-RUNBOOK.md) before contributing.

## How to contribute

### Bug report

Open an issue using the bug-report template. Include :

- Skill name that misfires
- Exact user prompt
- Expected vs actual output

### New skill proposal

Open an issue using the feature-request template. Include :

- Proposed `axum-{cat}-{topic}` name
- Category (core / syntax / impl / errors / agents)
- Trigger scenarios ("Use when...")
- Why existing skills do not cover it

### Pull request

Branch convention : `feat/`, `fix/`, `docs/`, `refactor/`.

Before opening PR :

1. Run validators locally :
   ```bash
   node /path/to/Skill-Package-Workflow-Template/scripts/validate-frontmatter.js .
   node /path/to/Skill-Package-Workflow-Template/scripts/validate-line-count.js .
   node /path/to/Skill-Package-Workflow-Template/scripts/validate-structure.js .
   node /path/to/Skill-Package-Workflow-Template/scripts/validate-language.js .
   node /path/to/Skill-Package-Workflow-Template/scripts/validate-emdash.js .
   ```
2. All must exit 0.
3. Regenerate discovery manifests : `node scripts/generate-manifest.js .`
4. Regenerate INDEX : `node scripts/generate-index.js .`
5. Update CHANGELOG.md (Keep a Changelog format)
6. Commit per skill (`feat(skill): axum-{cat}-{topic}`)

## Hard rules

- English-only in skill content
- Deterministic language : ALWAYS / NEVER, no "you might consider"
- SKILL.md < 500 lines, overflow to `references/`
- Section headings use `:` not em-dash
- YAML folded scalar `>` for description, NEVER quoted strings
- 3 reference files per skill : `methods.md`, `examples.md`, `anti-patterns.md`
- NO `README.md` inside a skill folder
- Verify ALL code-snippets via WebFetch against official docs

## Methodology

This package was built using the [7-phase research-first methodology](https://github.com/OpenAEC-Foundation/Skill-Package-Workflow-Template/blob/main/WORKFLOW.md). New skills follow the same phases : research first, then create, then validate.

## Compliance

PRs that lower the compliance audit score below 90% will be requested for revision. Run :

```bash
node /path/to/Skill-Package-Workflow-Template/scripts/generate-audit-report.js .
```

## License

By contributing, you agree your work is licensed under the same MIT terms as the repository.
