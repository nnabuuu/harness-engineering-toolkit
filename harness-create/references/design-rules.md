# Design Rules

Mode definitions, critical design rules, and generation pipeline for Phase 2 (Build).

---

## Three Artifact Modes

**Document Mode** — articles, reports, data files. Versioned files in `drafts/v{N}.{ext}`.

**Code Mode** — live source code, UI components. Git commits as version snapshots, no `drafts/` directory.

**Investigation Mode** — bug root cause analysis, system debugging. Evidence files in `evidence/`, no `drafts/` directory. No Generator-Evaluator pair; uses a single Investigator agent that tests hypotheses sequentially.

Choose based on HARNESS_SPEC.md. When in doubt, ask.

---

## Critical Design Rules

1. **Fresh context**: Agents run via `claude -p` with ZERO memory. Generator prompt MUST list files to read as its complete memory. See `prompt-templates.md`.

2. **File-based communication**: All data extraction (score, changelog, top issue) reads from files agents write — NEVER from `claude -p` stdout.

3. **Separate invocations**: Generator and Evaluator in separate `claude -p` calls. Evaluator positioned as independent reviewer.

4. **Per-agent tools**: Each `claude -p` needs explicit `--allowedTools`. See tool table in `prompt-templates.md`.

5. **Git snapshots**: Commit after each step. For Code Mode, this IS the version system.

6. **Starting-point injection**: Orchestrator appends iteration-specific context (version, artifact path, changelog path) at invocation time. See `orchestrator-templates.md`.

7. **State tracking via state.json**: `state.json` is the single source of truth for harness progress. The orchestrator updates it after every sub-step (generator, git commit, evaluator, extract, exit check). On `--resume`, the orchestrator reads `state.json` to find the exact sub-step that failed and restarts from there — not the whole iteration. Requires `jq`. See `orchestrator-templates.md` for helper functions and the step-based state machine pattern.

---

## Generation Pipeline

Complete each step in order. Read the referenced files for templates and details.

### Step 1: SPEC.md
Frozen target extracted from HARNESS_SPEC.md. Contains objective, artifact description, frozen constraints.

### Step 2: EVAL_CRITERIA.md
Scoring rubric. Must be concrete enough for a stranger to score consistently. Verify detection methods are actionable.

### Step 3: Agent Prompts
Read `prompt-templates.md` for base templates.

**[Document/Code Mode]:**
- **Generator must have**: fresh context warning (top), explicit reading order, root cause analysis section (classify fixes as A/B/C type), single-focus strategy (max 1-2 fixes per round), starting-point directive, changelog file path, constraint reminder.
- **Evaluator must have**: independent reviewer role, rubric reference, anti-bias instruction, bug classification (`[COMPONENT]`/`[SYSTEM]`/`[DESIGN]`), actionable fix hints (file path + target value), file output (`eval-reports/v{N}-eval.md`), parseable score (`Total: XX/100`).

**[Investigation Mode]:**
- **Investigator must have**: fresh context warning (top), hypothesis-driven workflow, one-hypothesis-per-round constraint, evidence file output (`evidence/h{N}-{name}.md`), CONFIRMED/ELIMINATED/INCONCLUSIVE judgment requirement.

### Step 4: Orchestrator Script + DAGU DAG
Read `orchestrator-templates.md` for bash template and DAGU YAML template.

Must support: `--dry-run`, `--resume`, `--max-cost`, `--step <name> --iteration <N>`, `--status`. Must implement all exit conditions. Must extract data from files, not stdout. Must use `state.json` for sub-step tracking (requires `jq`). Generate `dag.yaml` alongside `harness.sh` for optional DAGU visualization.

### Step 5: progress.md
Initialize with v0 row. See `project-structure.md` for format.

### Step 6: README.md
How to run, prerequisites, morning review checklist.

### Step 7: Self-Review
Re-read the Self-Review Checklist in SKILL.md. Verify every applicable item.

---

## Adaptation Rules

**Code artifact**: Use Code Mode. Add validation step (typecheck/test) before eval. Revert on failure. See `orchestrator-templates.md` for patterns.

**Browser-visible UI**: Add Playwright tools to Generator + Evaluator. Include `screenshots/` directory.

**Subjective quality**: Weight rubric toward concrete detection methods. Consider human review gate.

**> 2 agents**: Pipeline extends linearly: Planner → Generator → [Specialist/Tool Agent] → Evaluator.

**Bug investigation**: Use Investigation Mode. Single Investigator agent. No eval rubric needed — exit condition is "root cause confirmed" not "score threshold". See `orchestrator-templates.md` for investigation orchestrator template.
