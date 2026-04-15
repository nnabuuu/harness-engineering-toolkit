# File Structure & Extending

## File Structure

```
harness-engineering-toolkit/
├── .claude/
│   └── CLAUDE.md                        ← Skill routing for Claude Code
├── .gitignore
├── LICENSE
├── README.md                            ← English
├── README.zh-CN.md                      ← 简体中文
├── docs/                                ← Detailed documentation
├── shared/
│   └── 4-layer-model.md                 ← Model reference (used by all skills)
├── harness-self-check/
│   └── SKILL.md                         ← Interactive diagnostic (any industry)
├── harness-audit/
│   ├── SKILL.md                         ← Automated audit (generic framework)
│   └── docs/
│       ├── scoring-rubric.md            ← 5 blind spots, 0-5 each
│       └── software-engineering.md      ← Software domain adapter + auto-fix templates
├── harness-create/
│   ├── SKILL.md                         ← End-to-end: interview → spec → project
│   └── references/
│       ├── planning-interview.md        ← Interview question sequences
│       ├── spec-templates.md            ← HARNESS_SPEC.md output templates
│       ├── design-rules.md              ← Build mode rules and pipeline
│       ├── project-structure.md         ← Directory layout, file roles
│       ├── prompt-templates.md          ← Agent prompt templates
│       └── orchestrator-templates.md    ← Bash orchestration + DAGU DAG patterns
├── harness-retro/
│   ├── SKILL.md                         ← Retrospective analysis on completed tasks
│   └── references/
│       ├── analysis-playbook.md         ← 6 analysis dimensions, detection procedures
│       ├── retro-report-template.md     ← RETRO.md output format
│       └── archive-policy.md            ← What to keep, compress, remove
├── .harness-workspace/                  ← Generated harness projects (gitignored)
└── examples/                            ← Example outputs
```

## Extending to Other Domains

`software-engineering.md` is the first domain adapter. To add your own:

1. Create `harness-audit/docs/your-domain.md`
2. Define what to check and where to look, in your domain's terms
3. Add scoring examples for each blind spot
4. The generic rubric stays the same — only the evidence changes
