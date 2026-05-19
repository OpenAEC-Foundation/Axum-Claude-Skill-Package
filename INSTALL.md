# Install : Axum Skill Package

## Claude Code (local)

```bash
git clone https://github.com/OpenAEC-Foundation/Axum-Claude-Skill-Package.git
cp -r Axum-Claude-Skill-Package/skills/source/ ~/.claude/skills/axum/
```

Restart Claude Code if running. Skills auto-discover from `~/.claude/skills/`.

## As git submodule (recommended for project-specific use)

```bash
cd your-project
git submodule add https://github.com/OpenAEC-Foundation/Axum-Claude-Skill-Package.git .claude/skills/axum
git commit -m "feat: add axum skills as submodule"
```

## Via npm-agentskills

```bash
npx skills add @openaec/axum-claude-skill-package
```

## Claude.ai (web)

1. Open a Claude.ai project
2. Upload individual `SKILL.md` files from `skills/source/**/` as project knowledge
3. Skills activate based on description triggers

## Verification

After install, ask Claude :

> {{VERIFICATION_PROMPT}}

If skill activates : install successful. If not : check `~/.claude/skills/axum/` exists and contains category-folders.

## Requirements

- Axum 0.7,0.8
- Claude Code (latest)
- {{ADDITIONAL_REQUIREMENTS}}
