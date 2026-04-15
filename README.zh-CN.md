# Harness Engineering Toolkit

[![English](https://img.shields.io/badge/lang-English-blue)](README.md) [![简体中文](https://img.shields.io/badge/lang-简体中文-red)](README.zh-CN.md)

一组用于诊断和改进 AI agent harness 的工具——harness 是围绕 agent 构建的系统环境，决定了 agent 能否可靠地完成任务。

基于**四层 Harness 模型**：

> **L1 目标锚定 → L2 上下文工程 → L3 执行约束 → L4 评估反馈**

---

## 工具

### 诊断

| 工具 | 功能 | 适用对象 |
|------|------|---------|
| **`/harness-self-check`** | 交互式诊断。假设你已经有了 harness，然后通过追问暴露你以为有但其实没有的部分。五种盲区，逐个探测。 | 任何行业 |
| **`/harness-audit`** | 自动化审计。扫描代码仓库中的文件、CI 配置和 git 历史，检测五种盲区，发现能自动修的问题会问你要不要当场修掉。 | 软件开发者 |

### 构建

| 工具 | 功能 | 适用对象 |
|------|------|---------|
| **`/harness-create`** | 端到端 harness 创建：访谈 → 规格 → 可运行项目。两个入口：从零开始（先访谈）或从已有 HARNESS_SPEC.md（直接构建）。三种构建模式：文档、代码、调查。 | 任何行业 |

---

## 五种盲区

两个诊断工具检测的是同一组模式——团队以为自己有 harness 但其实没有的五种最常见方式：

| # | 你以为的 | 实际上的 |
|---|---------|---------|
| 1 | "我有目标" | 目标文档几个月没更新，agent 在对齐一个过时的方向 |
| 2 | "我有指令" | 150 行全堆在一起，没有优先级结构，agent 自己做判断 |
| 3 | "我有检查" | CI 跑的是默认规则，项目特有的错误全靠肉眼发现 |
| 4 | "我会 review" | 没有验收标准，质量取决于你有多忙 |
| 5 | "我搭了 harness" | 30 天没更新过，飞轮没在转 |

---

## 两个尺度

Toolkit 在两个尺度上工作：

**单次任务** — 让一个 agent 把一件事做对。目标清晰、上下文到位、检查在交付前运行、产出被评估。盲区 1-3 主要在这个尺度。

**长期运转** — 让系统在数周、数月后仍然正确。目标保持最新、上下文不腐烂、检查规则跟上新的错误模式、飞轮持续转动。盲区 5 在这个尺度。盲区 4 横跨两者。

当前工具覆盖两个尺度。`/harness-create` 在生成的 harness 项目中包含长期设计（启动仪式、进度文件、定期巡检、harness 版本化）。

---

## 安装

### 任何 Agent（Claude Code、Codex、Cursor、Windsurf、Copilot……）

```bash
npx skills add nnabuuu/harness-engineering-toolkit
```

CLI 自动检测已安装的 agent，将 skill 放到正确的目录。支持 41+ 种工具，基于开放的 [Agent Skills 标准](https://agentskills.io)。

选项：

```bash
npx skills add nnabuuu/harness-engineering-toolkit -g          # 全局安装
npx skills add nnabuuu/harness-engineering-toolkit -a claude-code  # 指定目标 agent
npx skills add nnabuuu/harness-engineering-toolkit --list       # 预览可用 skill
```

### 上传到 claude.ai

1. 下载各 skill 目录下的 `SKILL.md` 文件
2. 在 claude.ai 创建 Project → 上传 SKILL.md 文件作为 Project Knowledge
3. 开始对话："检查一下我的 harness 设置"

---

## 用 DAGU 监控进度

`/harness-create` 在生成 harness 的同时会生成 `dag.yaml`。如果本地安装了 [DAGU](https://dagu.readthedocs.io)，harness 会自动注册，你可以在 Web UI 中实时监控运行进度。

**你能看到什么：**
- 步骤依赖图（generator → evaluator → exit check）
- 每步的状态（运行中 / 成功 / 失败）、耗时、stdout 日志
- 跨迭代的运行历史
- 从 UI 手动重试和重新运行

**原理：** 创建 harness 时，会在 DAGU 的 DAGs 目录（`~/.dagu/dags/harness-{task-name}.yaml`）建一个指向 harness 中 `dag.yaml` 的符号链接（symlink）。DAGU 自动发现这个链接——不需要复制文件，也不需要改配置。Harness 留在 `.harness-workspace/` 里，DAGU 原地读取。

也可以手动管理注册：

```bash
bash harness.sh --register-dagu     # 重新创建 symlink
bash harness.sh --unregister-dagu   # 移除 symlink
```

**安装 DAGU（可选）：**

```bash
brew install dagu-org/brew/dagu   # macOS
dagu server                        # 启动 UI，默认 http://localhost:8080
```

不装 DAGU 也不影响使用，`bash harness.sh` 独立运行，功能完全相同。

---

## 快速开始

**不知道从哪开始？** 跑 `/harness-self-check`，它会问你问题，告诉你最该改的一件事。

**软件开发者想要自动扫描？** 在仓库里跑 `/harness-audit`，检测盲区，提供修复。

---

## 使用示例

### /harness-self-check

```
你：    /harness-self-check
Agent： 简单说说：你在用什么 AI agent，让它做什么事？
你：    Claude Code，帮我做一个 B2B 平台……
Agent： 你大概有一份目标文档——产品需求文档、项目简介之类的。
        上次打开它是什么时候？它还跟你现在做的事对得上吗？
你：    嗯……好像是几个月前写的。
Agent： 你的 agent 一直在忠实地对齐你三个月前写的目标，而你的产品方向
        已经变了。它每个小时的工作都在优化一个过时的方向。
        ……
Agent： ## Harness 诊断报告
        盲区命中：4/5
        最大缺口：过期目标……
        修复建议：打开你的目标文档，把第一段改成你今天在做的事。30分钟。
```

### /harness-audit

```
你：    /harness-audit
Agent： [扫描文件、CI 配置、git 历史]

        # Harness 审计报告

        ## 发现

        ### ⚠️ 过期目标 — 检测到
        docs/PRD.md 上次修改是 94 天前。CLAUDE.md 没有引用它。
        你的 agent 不知道这份文件的存在。

        ### ✅ 巨石指令 — 未检测到
        CLAUDE.md 72 行，链接到 4 份子文档。健康。

        ### ⚠️ 纸面规则 — 检测到
        CLAUDE.md 有 5 条"不要做 X"规则，只有 2 条有自动检查。
        3 条是纸面规则。

        要我为这 3 条纸面规则生成检查脚本吗？
```

---

## 文件结构

```
harness-engineering-toolkit/
├── .claude/
│   └── CLAUDE.md                        ← Claude Code skill 路由
├── .gitignore
├── LICENSE
├── README.md                            ← English
├── README.zh-CN.md                      ← 简体中文
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
├── .harness-workspace/                  ← 生成的 harness 项目（已 gitignore）
└── examples/                            ← 示例输出
```

## 扩展到其他行业

`software-engineering.md` 是第一个行业适配器。添加你自己的行业：

1. 创建 `harness-audit/docs/your-domain.md`
2. 用你行业的术语定义每种盲区要检查什么、去哪里检查
3. 添加每种盲区的评分示例
4. 通用评分标准不变——变的只是证据来源

---

## 背景

四层模型来自「驯化 AI」系列文章。它在业界已有框架的基础上增加了 **L1 目标锚定** 作为显式地基——现有框架中没有一个把目标对齐放在第一层。OpenAI 从上下文切入，Hashimoto 从错误修复切入，LangChain 从系统原语切入。本框架从目标切入，因为在跨团队、跨行业场景中，"大家都知道我们要干嘛"这个假设最先崩塌。

## 许可

MIT。
