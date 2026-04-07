# Harness Engineering Toolkit

## Available Skills

### Diagnose
- `/harness-self-check` — Interactive diagnostic. Probes five self-deceptions through conversation. Any industry.
- `/harness-audit` — Automated codebase scan. Detects five self-deceptions, offers auto-fix. Software projects.

### Build (coming soon)
- `/harness-plan` — Design a harness spec for a task or project.
- `/harness-build` — Generate harness artifacts from a plan.

## Skill Routing

When the user's request matches an available skill, invoke it:
- "check my harness", "diagnose", "self-check", "自检", "诊断" → `/harness-self-check`
- "audit", "scan my project", "审计", "扫描" → `/harness-audit`

## Shared References

- `shared/4-layer-model.md` — The 4-layer harness model reference. Skills read this for consistent terminology.
