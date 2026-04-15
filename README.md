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

---

## The Five Blind Spots

Both diagnostic skills detect the same five patterns — the most common ways teams think they have a harness but actually don't:

| # | You think | Actually |
|---|-----------|----------|
| 1 | "I have a goal" | Goal doc is months old, agent is optimizing an outdated target |
| 2 | "I have instructions" | 150 lines, no priority structure, agent makes its own judgments |
| 3 | "I have checks" | CI runs defaults only, project-specific errors aren't caught |
| 4 | "I review everything" | No criteria, quality varies by how busy you are |
| 5 | "I set up my harness" | Nothing changed in 30+ days, flywheel stopped |

---

## Two Scopes

The toolkit addresses harness at two scopes:

**Single-Session** — Making one agent do one task right. Goal is clear, context is loaded, checks run before delivery, output is evaluated. Self-deceptions 1-3 live here.

**Long-Term** — Making the system stay right over weeks and months. Goals stay current, context doesn't rot, checks evolve with new error patterns, the flywheel keeps turning. Self-deception 5 lives here. Self-deception 4 spans both.

Current skills cover both scopes. `/harness-create` includes long-term harness design (startup rituals, progress files, scheduled reviews, harness versioning) as part of the generated harness project.

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

`/harness-create` generates a `dag.yaml` alongside every harness. If [DAGU](https://dagu.readthedocs.io) is installed locally, the harness auto-registers itself so you can monitor progress in a web UI.

**What you get:**
- Dependency graph showing each step (generator → evaluator → exit check)
- Per-step status (running / succeeded / failed), duration, and stdout logs
- Run history across iterations
- Manual retry and re-run from the UI

**How it works:** At harness creation time, a symlink is created from DAGU's DAGs directory (`~/.dagu/dags/harness-{task-name}.yaml`) to your harness's `dag.yaml`. DAGU picks up the symlink automatically — no copying, no config editing. The harness stays in `.harness-workspace/` and DAGU reads it in place.

You can also manage registration manually:

```bash
bash harness.sh --register-dagu     # Re-create symlink
bash harness.sh --unregister-dagu   # Remove symlink
```

**Install DAGU (optional):**

```bash
brew install dagu-org/brew/dagu   # macOS
dagu server                        # Start UI at http://localhost:8080
```

If DAGU is not installed, the harness runs standalone via `bash harness.sh` with no loss of functionality.

---

## Quick Start

**Don't know where to start?** Run `/harness-self-check`. It asks you questions and tells you what to fix first.

**Software developer wanting a scan?** Run `/harness-audit` in your repo. It detects blind spots and offers fixes.

---

## Usage Examples

### /harness-self-check

```
You:    /harness-self-check
Agent:  Tell me briefly: what AI agent are you using, and what's it doing for you?
You:    Claude Code, building a B2B platform...
Agent:  You probably have some kind of goal doc — a PRD, a spec, something.
        When was the last time you opened it and checked whether it still
        matches what you're building today?
You:    Hmm... it's been a while.
Agent:  So your agent has been faithfully working toward a goal you wrote
        three months ago, while your actual product has moved on. Every
        hour it spends is optimizing the wrong thing.
        ...
Agent:  ## Harness Diagnostic Report
        Self-deceptions found: 4/5
        The biggest gap: The Stale Goal...
        The fix: Open your PRD, rewrite the first paragraph to match
        what you're actually building. 30 minutes.
```

### /harness-audit

```
You:    /harness-audit
Agent:  [scans files, CI config, git history]

        # Harness Audit Report

        ## Findings

        ### ⚠️ The Stale Goal — DETECTED
        docs/PRD.md last modified 94 days ago. CLAUDE.md does not
        reference it. Your agent doesn't know this file exists.

        ### ✅ The Monolith — NOT DETECTED
        CLAUDE.md is 72 lines, links to 4 sub-documents. Healthy.

        ### ⚠️ The Paper Rule — DETECTED
        5 "don't do X" rules in CLAUDE.md. Only 2 have automated checks.
        3 are paper-only.

        Want me to create a check script for the 3 paper rules?
```

---

## File Structure

```
harness-engineering-toolkit/
├── .claude/
│   └── CLAUDE.md                        ← Skill routing for Claude Code
├── .gitignore
├── LICENSE
├── README.md                            ← English
├── README.zh-CN.md                      ← 简体中文
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
├── .harness-workspace/                  ← Generated harness projects (gitignored)
└── examples/                            ← Example outputs
```

## Extending to Other Domains

`software-engineering.md` is the first domain adapter. To add your own:

1. Create `harness-audit/docs/your-domain.md`
2. Define what to check and where to look, in your domain's terms
3. Add scoring examples for each blind spot
4. The generic rubric stays the same — only the evidence changes

---

## Background

The 4-layer model comes from the「驯化 AI」article series. It adds **Goal Anchoring (L1)** as an explicit foundation — no existing framework foregrounds this. OpenAI starts from context, Hashimoto from errors, LangChain from primitives. This one starts from goals, because "everyone knows what we're doing" is the assumption that breaks first.

## License

MIT.
