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

### 复盘

| 工具 | 功能 | 适用对象 |
|------|------|---------|
| **`/harness-retro`** | 对已完成的 harness 任务进行复盘。分析运行数据（分数、评估报告、变更日志），覆盖 6 个维度：收敛性、瓶颈维度、重复劳动、成本效率、提示词质量、跨任务模式。生成具体的提示词/评分标准改进建议。可选归档已完成任务。 | 运行过 harness 的人 |

---

## 五种盲区

| # | 你以为的 | 实际上的 |
|---|---------|---------|
| 1 | "我有目标" | 目标文档几个月没更新，agent 在对齐一个过时的方向 |
| 2 | "我有指令" | 150 行全堆在一起，没有优先级结构，agent 自己做判断 |
| 3 | "我有检查" | CI 跑的是默认规则，项目特有的错误全靠肉眼发现 |
| 4 | "我会 review" | 没有验收标准，质量取决于你有多忙 |
| 5 | "我搭了 harness" | 30 天没更新过，飞轮没在转 |

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

`/harness-create` 在生成 harness 的同时会生成 `dag.yaml`。如果本地装了 [DAGU](https://dagu.readthedocs.io)，你可以在 Web UI 中查看依赖图、步骤状态和运行历史。[详情 →](docs/dagu.zh-CN.md)

---

## 快速开始

**不知道从哪开始？** 跑 `/harness-self-check`，它会问你问题，告诉你最该改的一件事。

**软件开发者想要自动扫描？** 在仓库里跑 `/harness-audit`，检测盲区，提供修复。

---

## 使用示例

看看交互式诊断和自动化审计实际运行的效果。[示例 →](docs/examples.zh-CN.md)

---

## 了解更多

- [两个尺度](docs/concepts.zh-CN.md) — 单次任务 vs. 长期运转
- [文件结构与扩展](docs/extending.zh-CN.md) — 项目布局和添加行业适配器

---

## 背景

四层模型来自「驯化 AI」系列文章。它在业界已有框架的基础上增加了 **L1 目标锚定** 作为显式地基——现有框架中没有一个把目标对齐放在第一层。OpenAI 从上下文切入，Hashimoto 从错误修复切入，LangChain 从系统原语切入。本框架从目标切入，因为在跨团队、跨行业场景中，"大家都知道我们要干嘛"这个假设最先崩塌。

## 许可

MIT。
