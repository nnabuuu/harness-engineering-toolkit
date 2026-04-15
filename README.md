# Harness Engineering Toolkit

[![English](https://img.shields.io/badge/lang-English-blue)](README.md) [![简体中文](https://img.shields.io/badge/lang-简体中文-red)](README.zh-CN.md)

Skills for diagnosing and improving the harness around your AI agent — the system that determines whether it succeeds or fails.

Built on the **4-Layer Harness Model**:

> **L1 Goal Anchoring → L2 Context Engineering → L3 Execution Constraints → L4 Evaluation Loop**

---

## Skills

### Diagnose

| Skill | What it does | For whom |
|-------|-------------|----------|
| **`/harness-self-check`** | Interactive diagnostic. Assumes you already have a setup, then probes until the gap reveals itself. Five blind spots, one at a time. | Anyone — any industry |
| **`/harness-audit`** | Automated codebase scan. Detects five blind spots by checking files, CI config, and git history. Offers to auto-fix what it finds. | Software developers |

### Build

| Skill | What it does | For whom |
|-------|-------------|----------|
| **`/harness-create`** | End-to-end harness creation: interview → spec → runnable project. Two entry points: start from scratch (interviews you first) or from existing HARNESS_SPEC.md (skips to build). Three build modes: document, code, investigation. | Anyone |

### Reflect

| Skill | What it does | For whom |
|-------|-------------|----------|
| **`/harness-retro`** | Retrospective on completed harness tasks. Analyzes run data (scores, eval reports, changelogs) across 6 dimensions: convergence, bottleneck dimensions, repeated work, cost efficiency, prompt quality, and cross-task patterns. Generates concrete prompt/rubric improvements. Optionally archives completed tasks. | Anyone who has run a harness |

---

## The Five Blind Spots

| # | You think | Actually |
|---|-----------|----------|
| 1 | "I have a goal" | Goal doc is months old, agent is optimizing an outdated target |
| 2 | "I have instructions" | 150 lines, no priority structure, agent makes its own judgments |
| 3 | "I have checks" | CI runs defaults only, project-specific errors aren't caught |
| 4 | "I review everything" | No criteria, quality varies by how busy you are |
| 5 | "I set up my harness" | Nothing changed in 30+ days, flywheel stopped |

---

## Install

### Any agent (Claude Code, Codex, Cursor, Windsurf, Copilot, ...)

```bash
npx skills add nnabuuu/harness-engineering-toolkit
```

The CLI auto-detects your installed agents and places skills in the correct directories. Supports 41+ tools via the [open Agent Skills standard](https://agentskills.io).

Options:

```bash
npx skills add nnabuuu/harness-engineering-toolkit -g          # Global install
npx skills add nnabuuu/harness-engineering-toolkit -a claude-code  # Target specific agent
npx skills add nnabuuu/harness-engineering-toolkit --list       # Preview available skills
```

### Upload to claude.ai

1. Download the `SKILL.md` files from each skill directory
2. Create a Project in claude.ai → upload the SKILL.md files as Project Knowledge
3. Start chatting: "Check my harness setup"

---

## Progress Monitoring with DAGU

`/harness-create` generates a `dag.yaml` alongside every harness. If [DAGU](https://dagu.readthedocs.io) is installed, you get a web UI with dependency graphs, step status, and run history. [details →](docs/dagu.md)

---

## Quick Start

**Don't know where to start?** Run `/harness-self-check`. It asks you questions and tells you what to fix first.

**Software developer wanting a scan?** Run `/harness-audit` in your repo. It detects blind spots and offers fixes.

---

## Usage Examples

See what a diagnostic session and an automated audit look like in practice. [examples →](docs/examples.md)

---

## Learn More

- [Two Scopes](docs/concepts.md) — single-session vs. long-term harness
- [File Structure & Extending](docs/extending.md) — project layout and adding domain adapters

---

## Background

The 4-layer model comes from the「驯化 AI」article series. It adds **Goal Anchoring (L1)** as an explicit foundation — no existing framework foregrounds this. OpenAI starts from context, Hashimoto from errors, LangChain from primitives. This one starts from goals, because "everyone knows what we're doing" is the assumption that breaks first.

## License

MIT.
