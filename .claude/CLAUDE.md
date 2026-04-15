# Harness Engineering Toolkit

## Available Skills

### Diagnose
- `/harness-self-check` — Interactive diagnostic. Probes five blind spots through conversation. Any industry.
- `/harness-audit` — Automated codebase scan. Detects five blind spots, offers auto-fix. Software projects.

### Build
- `/harness-create` — End-to-end: interview → spec → runnable harness project. Two entry points: start from scratch or from existing HARNESS_SPEC.md.

### Reflect
- `/harness-retro` — Retrospective on completed harness tasks. Analyzes convergence, bottlenecks, cost. Suggests improvements. Optionally archives.

## Skill Routing

When the user's request matches an available skill, invoke it:
- "check my harness", "diagnose", "self-check", "自检", "诊断" → `/harness-self-check`
- "audit", "scan my project", "审计", "扫描" → `/harness-audit`
- "plan harness", "build harness", "create harness", "harness", "define spec", "generate loop", "设计harness", "构建harness", "定义验收条件", "生成迭代脚本" → `/harness-create`
- "retro", "retrospective", "review harness", "analyze harness", "复盘", "回顾", "总结harness" → `/harness-retro`

## Shared References

- `shared/4-layer-model.md` — The 4-layer harness model reference. Skills read this for consistent terminology.
