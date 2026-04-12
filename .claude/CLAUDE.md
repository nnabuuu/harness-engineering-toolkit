# Harness Engineering Toolkit

## Available Skills

### Diagnose
- `/harness-self-check` — Interactive diagnostic. Probes five blind spots through conversation. Any industry.
- `/harness-audit` — Automated codebase scan. Detects five blind spots, offers auto-fix. Software projects.

### Build
- `/harness-plan` — Interactive interview to turn vague goals into structured HARNESS_SPEC.md. Two modes: iterative improvement, investigation/diagnostic.
- `/harness-build` — Generate runnable harness projects from a spec. Three modes: document, code, investigation.

## Skill Routing

When the user's request matches an available skill, invoke it:
- "check my harness", "diagnose", "self-check", "自检", "诊断" → `/harness-self-check`
- "audit", "scan my project", "审计", "扫描" → `/harness-audit`
- "plan harness", "define spec", "设计harness", "定义验收条件" → `/harness-plan`
- "build harness", "generate loop", "构建harness", "生成迭代脚本" → `/harness-build`

## Shared References

- `shared/4-layer-model.md` — The 4-layer harness model reference. Skills read this for consistent terminology.
