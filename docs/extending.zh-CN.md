# 文件结构与扩展

## 文件结构

```
harness-engineering-toolkit/
├── .claude/
│   └── CLAUDE.md                        ← Claude Code skill 路由
├── .gitignore
├── LICENSE
├── README.md                            ← English
├── README.zh-CN.md                      ← 简体中文
├── docs/                                ← 详细文档
├── shared/
│   └── 4-layer-model.md                 ← 四层模型参考（所有 skill 共用）
├── harness-self-check/
│   └── SKILL.md                         ← 交互式诊断（任何行业）
├── harness-audit/
│   ├── SKILL.md                         ← 自动化审计（通用框架）
│   └── docs/
│       ├── scoring-rubric.md            ← 五种盲区评分标准
│       └── software-engineering.md      ← 软件领域适配器 + 自动修复模板
├── harness-create/
│   ├── SKILL.md                         ← 端到端：访谈 → 规格 → 项目
│   └── references/
│       ├── planning-interview.md        ← 访谈问题序列
│       ├── spec-templates.md            ← HARNESS_SPEC.md 输出模板
│       ├── design-rules.md              ← 构建模式规则和流水线
│       ├── project-structure.md         ← 目录结构、文件角色
│       ├── prompt-templates.md          ← Agent 提示词模板
│       └── orchestrator-templates.md    ← Bash 编排 + DAGU DAG 模式
├── harness-retro/
│   ├── SKILL.md                         ← 复盘分析已完成的任务
│   └── references/
│       ├── analysis-playbook.md         ← 6 个分析维度、检测流程
│       ├── retro-report-template.md     ← RETRO.md 输出格式
│       └── archive-policy.md            ← 保留、压缩、移除策略
├── .harness-workspace/                  ← 生成的 harness 项目（已 gitignore）
└── examples/                            ← 示例输出
```

## 扩展到其他行业

`software-engineering.md` 是第一个行业适配器。添加你自己的行业：

1. 创建 `harness-audit/docs/your-domain.md`
2. 用你行业的术语定义每种盲区要检查什么、去哪里检查
3. 添加每种盲区的评分示例
4. 通用评分标准不变——变的只是证据来源
