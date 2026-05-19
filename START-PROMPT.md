# Start Prompt : Axum-Claude-Skill-Package

> Plak deze prompt in een nieuwe Claude Code sessie geopend in deze workspace.

## Volledige bootstrap (lege workspace)

```
Lees BOOTSTRAP-RUNBOOK.md van /home/freek/GitHub/Skill-Package-Workflow-Template/ en bootstrap deze workspace voor Axum 0.7,0.8.

Inputs:
- tech: axum
- prefix: axum
- versions: 0.7,0.8
- languages: Rust
- license: MIT
- estimated skills: {{ESTIMATED_SKILLS}}

Ga door alle phases tot v1.0.0 published. Gebruik tmux-orchestration voor Phase 5. Bypass permissions ON. Geen stilvallen.
```

## Hervatten lopende sessie

```
Lees HANDOFF.md en hervat vanaf de daar genoemde phase. Gebruik BOOTSTRAP-RUNBOOK.md uit /home/freek/GitHub/Skill-Package-Workflow-Template/ als methodologie-referentie.
```

## Alleen specifieke fase

```
Lees BOOTSTRAP-RUNBOOK.md sectie {{SECTION_NUM}} (Phase {{PHASE_NUM}}) en voer alleen die fase uit. Werk op huidige staat van repo. Verify-conditie eind fase moet groen.
```

## Audit-only

```
Lees AUDIT-START-PROMPT.md van Skill-Package-Workflow-Template en draai een compliance audit op deze workspace. Auto-remediate alles onder 90%.
```

## Hard rules in deze workspace

1. Bypass permissions ON ({{.claude/settings.json}})
2. tmux-orchestration voor parallel skill-creation in Phase 5
3. Commits per fase + push
4. Geen fallback-paden, root-cause-fix bij failure
5. Compliance score >= 90% verplicht voor v1.0.0
6. Em-dashes verboden in section headings (gebruik `:`)
7. Quoted YAML descriptions verboden (folded scalar `>` only)
