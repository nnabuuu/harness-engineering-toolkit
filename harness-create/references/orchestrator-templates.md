# Orchestrator Templates

## Key Patterns

Before reading the templates, understand these critical patterns:

### 1. Starting-Point Injection

The orchestrator MUST append iteration-specific context when invoking each agent. The agent prompt file is static; dynamic context (version number, starting point, changelog path) is injected at invocation time:

```bash
run_generator() {
  local version=$1
  local prev=$((version - 1))
  local prompt=$(cat "$PROMPTS_DIR/generator.md")

  # CRITICAL: Inject iteration-specific context
  if [[ $version -eq 1 ]]; then
    prompt="${prompt}

This is the FIRST iteration. Create the initial version based on SPEC.md.
Save to: [ARTIFACT_OUTPUT_PATH]
Save changelog to: ${CHANGELOG_DIR}/v1-changelog.md"
  else
    prompt="${prompt}

This is iteration ${version}.
Your STARTING POINT is: [ARTIFACT_LOCATION] — read it first, then improve it.
Read eval-reports/v${prev}-eval.md for specific feedback to address.
Save to: [ARTIFACT_OUTPUT_PATH]
Save changelog to: ${CHANGELOG_DIR}/v${version}-changelog.md"
  fi

  claude -p "$prompt" --allowedTools "$GENERATOR_TOOLS"
}
```

### 2. File-Based Data Extraction (NOT stdout)

`claude -p` stdout is noisy and unreliable. Extract ALL data from files the agents write:

```bash
# Score — from eval report FILE
score=$(grep -oE '(总分|Total)[：:][[:space:]]*[0-9]+' "$EVAL_DIR/v${i}-eval.md" | grep -oE '[0-9]+' | head -1)

# Key changes — from changelog FILE
changes=$(grep "^- " "$CHANGELOG_DIR/v${i}-changelog.md" | head -3 | tr '\n' '; ')

# Top issue — from eval report FILE
top_issue=$(grep -A1 "Priority Fix" "$EVAL_DIR/v${i}-eval.md" | tail -1)
```

### 3. Git Snapshots Per Iteration

**IMPORTANT:** If the repo uses commitlint/husky, the commit message MUST follow conventional format.
Check `.commitlintrc` or `package.json` for allowed types/scopes. Example:
- `feat(frontend): harness-name v${VERSION} iteration`
- `feat(frontend): harness-name v${VERSION} eval — score ${SCORE}/100`

```bash
# After Generator + validation passes:
git add -A
git commit -m "feat(frontend): ${HARNESS_NAME} v${VERSION} iteration"

# After Evaluator completes:
git add eval-reports/ ${CHANGELOG_DIR}/ progress.md
git commit -m "feat(frontend): ${HARNESS_NAME} v${VERSION} eval — score ${SCORE}/100"
```

### 4. Code Mode: Validation + Revert

For code artifacts, run validation (typecheck/tests) before eval. On failure, revert and skip:

```bash
# Step 2: Validation (Code Mode only)
echo "Running validation..."
cd "$SOURCE_DIR" && npx tsc --noEmit && npx vitest run
if [[ $? -ne 0 ]]; then
  echo "Validation failed. Reverting..."
  git checkout -- "$SOURCE_DIR"
  append_progress $i "FAIL" "Validation failed" "typecheck/test error"
  continue  # skip eval, try next iteration
fi
```

### 5. Frozen File Violation Gate (Code Mode)

Check for modifications to files that MUST NOT be changed. Run after Generator, before validation:

```bash
# FROZEN_FILES is a newline-separated list from SPEC.md, e.g.:
# FROZEN_FILES="src/tokens.css
# src/theme.ts"
check_frozen_files() {
  local violations=0
  while IFS= read -r f; do
    [[ -z "$f" ]] && continue
    if git diff --name-only | grep -qF "$f"; then
      echo "FROZEN FILE VIOLATION: $f was modified. Reverting."
      git checkout -- "$f"
      violations=$((violations + 1))
    fi
  done <<< "$FROZEN_FILES"
  return $violations
}

# In the main loop, after Generator:
check_frozen_files
frozen_violations=$?
if [[ $frozen_violations -gt 0 ]]; then
  echo "WARNING: ${frozen_violations} frozen file(s) reverted."
fi
```

### 6. Regression Detection + Auto-Revert

After scoring, check if score dropped significantly. If so, revert Generator's changes:

```bash
# After score extraction, check for regression
if [[ $prev_score -gt 0 && $score -lt $((prev_score - 5)) ]]; then
  echo "REGRESSION detected: ${prev_score} → ${score} (Δ < -5). Reverting."
  # Code Mode: revert source files
  git checkout HEAD~1 -- "$SOURCE_DIR"
  # Document Mode: would revert draft file instead
  append_progress $i "REVERTED" "Regression from ${prev_score} to ${score}" "Auto-reverted"
  continue  # skip to next iteration
fi
```

### 7. Code Mode: Dev Server Lifecycle

```bash
# Start dev server before loop
start_dev_server() {
  cd "$SOURCE_DIR" && npm run dev &
  DEV_SERVER_PID=$!
  sleep 5  # wait for server to start
}

# Kill on exit
cleanup() {
  [[ -n "${DEV_SERVER_PID:-}" ]] && kill "$DEV_SERVER_PID" 2>/dev/null
}
trap cleanup EXIT
```

### 8. state.json State Machine

`state.json` is the single source of truth for harness progress. The orchestrator updates it after every sub-step. On `--resume`, it reads `state.json` to find the exact failed sub-step and restarts from there.

**Dependency:** Requires `jq`. Check at startup:

```bash
if ! command -v jq &>/dev/null; then
  echo "ERROR: jq is required but not installed."
  echo "  macOS:  brew install jq"
  echo "  Ubuntu: sudo apt-get install jq"
  echo "  Arch:   sudo pacman -S jq"
  exit 1
fi
```

**state.json schema:**

```json
{
  "harness": "task-name",
  "mode": "document|code|investigation",
  "status": "running|paused|completed|failed",
  "current_iteration": 3,
  "current_step": "evaluator",
  "max_iterations": 10,
  "score_threshold": 85,
  "started_at": "2026-04-16T10:00:00Z",
  "updated_at": "2026-04-16T10:45:00Z",
  "iterations": [
    {
      "version": 1,
      "status": "completed",
      "steps": {
        "generator": { "status": "completed", "started_at": "...", "completed_at": "..." },
        "git_post_gen": { "status": "completed", "commit": "abc123" },
        "evaluator": { "status": "completed", "started_at": "...", "completed_at": "..." },
        "extract": { "status": "completed", "score": 62 },
        "git_post_eval": { "status": "completed", "commit": "def456" },
        "exit_check": { "status": "completed", "result": "continue" }
      },
      "score": 62,
      "changelog_summary": "added intro; rewrote conclusion"
    }
  ]
}
```

**Step definitions per mode:**

- **Document Mode:** `generator` → `git_post_gen` → `evaluator` → `extract` → `git_post_eval` → `exit_check`
- **Code Mode:** `generator` → `frozen_check` → `validation` → `git_post_gen` → `evaluator` → `extract` → `git_post_eval` → `exit_check`
- **Investigation Mode:** `investigator` → `git_snapshot` → `extract_evidence` → `exit_check`

**Helper functions:**

```bash
STATE_FILE="${HARNESS_DIR}/state.json"

# Initialize state.json at harness setup
init_state() {
  cat > "$STATE_FILE" <<STATEEOF
{
  "harness": "${TASK_NAME}",
  "mode": "${MODE}",
  "status": "running",
  "current_iteration": 1,
  "current_step": "${STEPS[0]}",
  "max_iterations": ${MAX_ITERATIONS},
  "score_threshold": ${SCORE_THRESHOLD},
  "started_at": "$(date -u '+%Y-%m-%dT%H:%M:%SZ')",
  "updated_at": "$(date -u '+%Y-%m-%dT%H:%M:%SZ')",
  "iterations": []
}
STATEEOF
}

# Read a field from state.json
state_read() {
  jq -r "$1" "$STATE_FILE"
}

# Update state.json (top-level fields)
state_update() {
  local tmp="${STATE_FILE}.tmp"
  jq "$1" "$STATE_FILE" > "$tmp" && mv "$tmp" "$STATE_FILE"
}

# Initialize a new iteration entry
state_init_iteration() {
  local version=$1
  state_update "
    .current_iteration = ${version} |
    .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\" |
    .iterations += [{
      \"version\": ${version},
      \"status\": \"in_progress\",
      \"steps\": {}
    }]"
}

# Update a step within the current iteration
# Usage: state_step_update <iteration_index> <step_name> '{"status":"completed"}'
state_step_update() {
  local idx=$1 step=$2 data=$3
  state_update "
    .current_step = \"${step}\" |
    .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\" |
    .iterations[${idx}].steps.${step} = (.iterations[${idx}].steps.${step} // {}) * ${data}"
}

# Mark a step as running
state_step_start() {
  local idx=$1 step=$2
  state_step_update "$idx" "$step" "{\"status\":\"running\",\"started_at\":\"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\"}"
}

# Mark a step as completed
state_step_complete() {
  local idx=$1 step=$2
  local extra="${3:-"{}"}"
  state_step_update "$idx" "$step" "({\"status\":\"completed\",\"completed_at\":\"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\"} * ${extra})"
}

# Mark a step as failed
state_step_fail() {
  local idx=$1 step=$2 error=$3
  local tmp="${STATE_FILE}.tmp"
  jq --arg err "$error" "
    .current_step = \"${step}\" |
    .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\" |
    .iterations[${idx}].steps[\"${step}\"] = (.iterations[${idx}].steps[\"${step}\"] // {}) * {\"status\":\"failed\",\"error\":\$err}
  " "$STATE_FILE" > "$tmp" && mv "$tmp" "$STATE_FILE"
}

# Get step status for a given iteration
state_get_step_status() {
  local idx=$1 step=$2
  jq -r ".iterations[${idx}].steps[\"${step}\"].status // \"pending\"" "$STATE_FILE"
}

# Mark harness as completed/failed
state_finish() {
  local status=$1
  state_update ".status = \"${status}\" | .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\""
}

# Print a summary of current state (for --status flag)
state_print_summary() {
  echo "╔══════════════════════════════════════════════════╗"
  echo "║  Harness Status: $(state_read '.harness')"
  echo "║  Mode: $(state_read '.mode')"
  echo "║  Status: $(state_read '.status')"
  echo "║  Iteration: $(state_read '.current_iteration') / $(state_read '.max_iterations')"
  echo "║  Current step: $(state_read '.current_step')"
  echo "║  Started: $(state_read '.started_at')"
  echo "║  Updated: $(state_read '.updated_at')"
  echo "╚══════════════════════════════════════════════════╝"
  echo ""
  local count=$(jq '.iterations | length' "$STATE_FILE")
  if [[ "$count" -gt 0 ]]; then
    echo "Iterations:"
    jq -r '.iterations[] | "  v\(.version): \(.status) — score \(.score // "N/A")"' "$STATE_FILE"
  fi
}
```

### 9. run_step Wrapper

The `run_step` function wraps every sub-step. It checks state.json, skips completed steps, updates state on start/success/failure, and exits on failure.

```bash
# Run a single step with state tracking
# Usage: run_step <iteration_index> <step_name> <step_function> [args...]
run_step() {
  local idx=$1 step_name=$2 step_fn=$3
  shift 3

  # Check if already completed (for --resume)
  local step_status
  step_status=$(state_get_step_status "$idx" "$step_name")
  if [[ "$step_status" == "completed" ]]; then
    echo "  [skip] ${step_name} (already completed)"
    return 0
  fi

  echo "  [run]  ${step_name}..."
  state_step_start "$idx" "$step_name"

  if $step_fn "$@"; then
    state_step_complete "$idx" "$step_name"
    echo "  [done] ${step_name}"
    return 0
  else
    state_step_fail "$idx" "$step_name" "step function returned non-zero"
    echo "  [FAIL] ${step_name}"
    return 1
  fi
}
```

Both standalone loop mode and DAGU-driven `--step` mode use the same step functions, so behavior is identical.

### 10. DAGU Auto-Registration

Auto-register `dag.yaml` with a local DAGU instance by symlinking into its DAGs directory. Called once at harness setup time (Phase 2, Step 2.4), not on every run. If DAGU is not installed, skip silently.

```bash
# Resolve DAGU's DAGs directory.
# Priority: $DAGU_DAGS_DIR → `dagu config` output → ~/.dagu/dags (fallback)
_dagu_dags_dir() {
  if [[ -n "${DAGU_DAGS_DIR:-}" ]]; then
    echo "$DAGU_DAGS_DIR"
  else
    dagu config 2>/dev/null | awk '/DAGs directory/{print $NF}' || echo "${HOME}/.dagu/dags"
  fi
}

# Auto-register dag.yaml with local DAGU instance (if installed).
# Called once at harness setup time, not on every run.
register_dagu() {
  if ! command -v dagu &>/dev/null; then
    echo "[dagu] Not installed — skipping DAG registration."
    echo "       Install: https://dagu.readthedocs.io"
    return 0
  fi

  local dags_dir
  dags_dir="$(_dagu_dags_dir)"
  mkdir -p "$dags_dir"

  local link_name="${dags_dir}/harness-${TASK_NAME}.yaml"
  local target="${HARNESS_DIR}/dag.yaml"

  if [[ ! -f "$target" ]]; then
    echo "[dagu] ERROR: ${target} not found. Generate dag.yaml first."
    return 1
  fi

  if [[ -L "$link_name" ]]; then
    echo "[dagu] Symlink already exists: ${link_name}"
    return 0
  fi

  ln -s "$target" "$link_name"
  echo "[dagu] Registered: ${link_name} → ${target}"
  echo "       View at: http://localhost:8080 (default DAGU UI)"
}

# Remove the symlink created by register_dagu.
unregister_dagu() {
  local dags_dir
  dags_dir="$(_dagu_dags_dir)"
  local link_name="${dags_dir}/harness-${TASK_NAME}.yaml"

  if [[ -L "$link_name" ]]; then
    rm "$link_name"
    echo "[dagu] Unregistered: ${link_name}"
  else
    echo "[dagu] No symlink found at ${link_name}"
  fi
}
```

The orchestrator should also support `--register-dagu` and `--unregister-dagu` flags so users can re-run registration manually after setup (see CLI Flags below).

---

## CLI Flags

All orchestrator scripts support these flags:

```
harness.sh                                # Run full loop (standalone mode)
harness.sh --resume                       # Resume from state.json (exact sub-step)
harness.sh --step <name> --iteration <N>  # Run single step (for DAGU)
harness.sh --status                       # Print current state.json summary
harness.sh --dry-run                      # Estimate cost without running
harness.sh --max-cost <USD>               # Set cost cap
harness.sh --register-dagu                # Symlink dag.yaml into DAGU's DAGs dir
harness.sh --unregister-dagu              # Remove the DAGU symlink
```

Flag parsing template (requires `register_dagu`/`unregister_dagu` defined above — see Key Patterns §10):

```bash
DRY_RUN=false
RESUME=false
SINGLE_STEP=""
SINGLE_ITERATION=""
SHOW_STATUS=false

while [[ $# -gt 0 ]]; do
  case "$1" in
    --dry-run)    DRY_RUN=true; shift ;;
    --resume)     RESUME=true; shift ;;
    --step)       SINGLE_STEP="$2"; shift 2 ;;
    --iteration)  SINGLE_ITERATION="$2"; shift 2 ;;
    --status)     SHOW_STATUS=true; shift ;;
    --max-cost)   MAX_COST_USD="$2"; shift 2 ;;
    --max-cost=*) MAX_COST_USD="${1#*=}"; shift ;;
    --register-dagu)   register_dagu; exit 0 ;;
    --unregister-dagu) unregister_dagu; exit 0 ;;
    *) echo "Unknown flag: $1"; exit 1 ;;
  esac
done

# Handle --status
if $SHOW_STATUS; then
  if [[ ! -f "$STATE_FILE" ]]; then
    echo "No state.json found. Harness has not been started."
    exit 0
  fi
  state_print_summary
  exit 0
fi

# Handle --step (DAGU mode)
if [[ -n "$SINGLE_STEP" ]]; then
  if [[ -z "$SINGLE_ITERATION" ]]; then
    echo "ERROR: --step requires --iteration" >&2
    exit 1
  fi
  step_idx=$((SINGLE_ITERATION - 1))
  # Validate step name
  if ! declare -f "step_${SINGLE_STEP}" >/dev/null 2>&1; then
    echo "ERROR: Unknown step '${SINGLE_STEP}'. Valid steps: ${STEPS[*]}" >&2
    exit 1
  fi
  # Ensure iteration entry exists
  step_iter_count=$(jq '.iterations | length' "$STATE_FILE")
  if [[ $step_iter_count -le $step_idx ]]; then
    state_init_iteration "$SINGLE_ITERATION"
  fi
  # Run the single step
  run_step "$step_idx" "$SINGLE_STEP" "step_${SINGLE_STEP}" "$SINGLE_ITERATION"
  exit $?
fi
```

---

## Resume Logic

On startup with `--resume`:

1. Read `state.json`
2. Find `current_iteration`
3. Look at last iteration's `steps` — find the first step with status != "completed"
4. Resume from that exact step, skipping all completed steps
5. Special handling: if `generator` or `evaluator` failed, check if their output files exist (partial completion) — if output exists, mark as completed and skip to next step

```bash
if $RESUME; then
  if [[ ! -f "$STATE_FILE" ]]; then
    echo "No state.json found. Starting fresh."
    RESUME=false
  else
    START_VERSION=$(state_read '.current_iteration')
    echo "Resuming from v${START_VERSION}, step: $(state_read '.current_step')..."
    # The run_step wrapper handles skipping completed steps automatically
  fi
fi
```

---

## Bash Orchestrator Template (Document Mode)

```bash
#!/usr/bin/env bash
set -euo pipefail

# ============================================================
# [TASK_NAME] Harness — Overnight Iteration Loop
# Generated by harness-create
# ============================================================

# --- Dependency Check ---
if ! command -v jq &>/dev/null; then
  echo "ERROR: jq is required but not installed."
  echo "  macOS:  brew install jq"
  echo "  Ubuntu: sudo apt-get install jq"
  echo "  Arch:   sudo pacman -S jq"
  exit 1
fi

# --- Configuration ---
TASK_NAME="[TASK_NAME]"
MODE="document"
ARTIFACT_EXT="[EXT]"           # md, ts, py, json, etc.
MAX_ITERATIONS=[MAX_ITER]       # Hard cap on iterations
SCORE_THRESHOLD=[THRESHOLD]     # Stop when score >= this
MIN_IMPROVEMENT=[MIN_DELTA]     # Stop if improvement < this for 2 consecutive runs
MAX_COST_USD="${MAX_COST:-999}" # Override with --max-cost flag

# Agent tool permissions
GENERATOR_TOOLS="Read,Write,Edit,Grep,Glob"
EVALUATOR_TOOLS="Read,Grep,Glob"

# Step sequence for Document Mode
STEPS=(generator git_post_gen evaluator extract git_post_eval exit_check)

# Rough token estimates per iteration (adjust based on task)
EST_TOKENS_GENERATOR=[GEN_TOKENS]
EST_TOKENS_EVALUATOR=[EVAL_TOKENS]
COST_PER_1K_TOKENS=0.003       # Adjust for your model/tier

# --- Directories ---
HARNESS_DIR="$(cd "$(dirname "$0")" && pwd)"
DRAFTS_DIR="${HARNESS_DIR}/drafts"
EVAL_DIR="${HARNESS_DIR}/eval-reports"
PROMPTS_DIR="${HARNESS_DIR}/prompts"
PROGRESS_FILE="${HARNESS_DIR}/progress.md"
STATE_FILE="${HARNESS_DIR}/state.json"

# --- DAGU registration helpers (Key Patterns §10) ---
_dagu_dags_dir() {
  if [[ -n "${DAGU_DAGS_DIR:-}" ]]; then echo "$DAGU_DAGS_DIR"
  else dagu config 2>/dev/null | awk '/DAGs directory/{print $NF}' || echo "${HOME}/.dagu/dags"; fi
}
register_dagu() {
  if ! command -v dagu &>/dev/null; then
    echo "[dagu] Not installed — skipping DAG registration."
    return 0
  fi
  local dags_dir; dags_dir="$(_dagu_dags_dir)"; mkdir -p "$dags_dir"
  local link_name="${dags_dir}/harness-${TASK_NAME}.yaml"
  local target="${HARNESS_DIR}/dag.yaml"
  if [[ ! -f "$target" ]]; then echo "[dagu] ERROR: ${target} not found."; return 1; fi
  if [[ -L "$link_name" ]]; then echo "[dagu] Symlink already exists: ${link_name}"; return 0; fi
  ln -s "$target" "$link_name"
  echo "[dagu] Registered: ${link_name} → ${target}"
}
unregister_dagu() {
  local dags_dir; dags_dir="$(_dagu_dags_dir)"
  local link_name="${dags_dir}/harness-${TASK_NAME}.yaml"
  if [[ -L "$link_name" ]]; then rm "$link_name"; echo "[dagu] Unregistered: ${link_name}"
  else echo "[dagu] No symlink found at ${link_name}"; fi
}

# --- Flags ---
DRY_RUN=false
RESUME=false
SINGLE_STEP=""
SINGLE_ITERATION=""
SHOW_STATUS=false

while [[ $# -gt 0 ]]; do
  case "$1" in
    --dry-run)    DRY_RUN=true; shift ;;
    --resume)     RESUME=true; shift ;;
    --step)       SINGLE_STEP="$2"; shift 2 ;;
    --iteration)  SINGLE_ITERATION="$2"; shift 2 ;;
    --status)     SHOW_STATUS=true; shift ;;
    --max-cost)   MAX_COST_USD="$2"; shift 2 ;;
    --max-cost=*) MAX_COST_USD="${1#*=}"; shift ;;
    --register-dagu)   register_dagu; exit 0 ;;
    --unregister-dagu) unregister_dagu; exit 0 ;;
    *) echo "Unknown flag: $1"; exit 1 ;;
  esac
done

# ============================================================
# state.json helpers (see Key Patterns §8)
# ============================================================

init_state() {
  cat > "$STATE_FILE" <<STATEEOF
{
  "harness": "${TASK_NAME}",
  "mode": "${MODE}",
  "status": "running",
  "current_iteration": 1,
  "current_step": "${STEPS[0]}",
  "max_iterations": ${MAX_ITERATIONS},
  "score_threshold": ${SCORE_THRESHOLD},
  "started_at": "$(date -u '+%Y-%m-%dT%H:%M:%SZ')",
  "updated_at": "$(date -u '+%Y-%m-%dT%H:%M:%SZ')",
  "iterations": []
}
STATEEOF
}

state_read() { jq -r "$1" "$STATE_FILE"; }

state_update() {
  local tmp="${STATE_FILE}.tmp"
  jq "$1" "$STATE_FILE" > "$tmp" && mv "$tmp" "$STATE_FILE"
}

state_init_iteration() {
  local version=$1
  state_update "
    .current_iteration = ${version} |
    .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\" |
    .iterations += [{\"version\":${version},\"status\":\"in_progress\",\"steps\":{}}]"
}

state_step_update() {
  local idx=$1 step=$2 data=$3
  state_update "
    .current_step = \"${step}\" |
    .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\" |
    .iterations[${idx}].steps.${step} = (.iterations[${idx}].steps.${step} // {}) * ${data}"
}

state_step_start() {
  state_step_update "$1" "$2" "{\"status\":\"running\",\"started_at\":\"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\"}"
}

state_step_complete() {
  local extra="${3:-"{}"}"
  state_step_update "$1" "$2" "({\"status\":\"completed\",\"completed_at\":\"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\"} * ${extra})"
}

state_step_fail() {
  local tmp="${STATE_FILE}.tmp"
  jq --arg err "$3" "
    .current_step = \"$2\" |
    .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\" |
    .iterations[$1].steps[\"$2\"] = (.iterations[$1].steps[\"$2\"] // {}) * {\"status\":\"failed\",\"error\":\$err}
  " "$STATE_FILE" > "$tmp" && mv "$tmp" "$STATE_FILE"
}

state_get_step_status() {
  jq -r ".iterations[${1}].steps[\"${2}\"].status // \"pending\"" "$STATE_FILE"
}

state_finish() {
  state_update ".status = \"${1}\" | .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\""
}

state_print_summary() {
  echo "╔══════════════════════════════════════════════════╗"
  echo "║  Harness: $(state_read '.harness')"
  echo "║  Mode: $(state_read '.mode')"
  echo "║  Status: $(state_read '.status')"
  echo "║  Iteration: $(state_read '.current_iteration') / $(state_read '.max_iterations')"
  echo "║  Current step: $(state_read '.current_step')"
  echo "║  Started: $(state_read '.started_at')"
  echo "║  Updated: $(state_read '.updated_at')"
  echo "╚══════════════════════════════════════════════════╝"
  echo ""
  local count
  count=$(jq '.iterations | length' "$STATE_FILE")
  if [[ "$count" -gt 0 ]]; then
    echo "Iterations:"
    jq -r '.iterations[] | "  v\(.version): \(.status) — score \(.score // "N/A")"' "$STATE_FILE"
  fi
}

# ============================================================
# run_step wrapper (see Key Patterns §9)
# ============================================================

run_step() {
  local idx=$1 step_name=$2 step_fn=$3
  shift 3

  local step_status
  step_status=$(state_get_step_status "$idx" "$step_name")
  if [[ "$step_status" == "completed" ]]; then
    echo "  [skip] ${step_name} (already completed)"
    return 0
  fi

  echo "  [run]  ${step_name}..."
  state_step_start "$idx" "$step_name"

  if $step_fn "$@"; then
    state_step_complete "$idx" "$step_name"
    echo "  [done] ${step_name}"
    return 0
  else
    state_step_fail "$idx" "$step_name" "step function returned non-zero"
    echo "  [FAIL] ${step_name}"
    return 1
  fi
}

# ============================================================
# Step functions
# ============================================================

get_last_version() {
  local last=$(ls "${DRAFTS_DIR}"/v*.${ARTIFACT_EXT} 2>/dev/null | sort -V | tail -1)
  if [[ -n "$last" ]]; then
    basename "$last" | grep -oE '[0-9]+' | head -1
  else
    echo "0"
  fi
}

extract_score() {
  local eval_file="$1"
  grep -oE '(总分|Total)[：:][[:space:]]*[0-9]+' "$eval_file" | grep -oE '[0-9]+' | head -1
}

estimate_cost() {
  local iterations=$1
  local total_tokens=$(( iterations * (EST_TOKENS_GENERATOR + EST_TOKENS_EVALUATOR) ))
  echo "scale=2; $total_tokens * $COST_PER_1K_TOKENS / 1000" | bc
}

append_progress() {
  local version=$1 score=$2 changes=$3 top_issue=$4
  local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
  echo "| v${version} | ${timestamp} | ${score}/100 | ${changes} | ${top_issue} |" >> "$PROGRESS_FILE"
}

# --- Step: generator ---
step_generator() {
  local version=$1
  local prev=$((version - 1))
  local prompt=$(cat "$PROMPTS_DIR/generator.md")

  if [[ $version -eq 1 ]]; then
    prompt="${prompt}

This is the FIRST iteration. Create the initial version based on SPEC.md.
Save to: drafts/v1.${ARTIFACT_EXT}
Save changelog to: drafts/v1-changelog.md"
  else
    prompt="${prompt}

This is iteration ${version}.
Your STARTING POINT is: drafts/v${prev}.${ARTIFACT_EXT} — read it first, then modify it.
Read eval-reports/v${prev}-eval.md for specific feedback to address.
Save improved version to: drafts/v${version}.${ARTIFACT_EXT}
Save changelog to: drafts/v${version}-changelog.md"
  fi

  claude -p "$prompt" --allowedTools "$GENERATOR_TOOLS" 2>&1 || return 1
}

# --- Step: git_post_gen ---
step_git_post_gen() {
  local version=$1
  git add -A && git commit -m "harness: v${version} iteration" --allow-empty 2>/dev/null || true
}

# --- Step: evaluator ---
step_evaluator() {
  local version=$1
  local prompt=$(cat "$PROMPTS_DIR/evaluator.md")
  prompt=$(echo "$prompt" | sed "s/{N}/${version}/g")

  claude -p "$prompt" --allowedTools "$EVALUATOR_TOOLS" 2>&1 || return 1
}

# --- Step: extract ---
step_extract() {
  local version=$1
  local idx=$((version - 1))

  local eval_file="${EVAL_DIR}/v${version}-eval.md"
  if [[ ! -f "$eval_file" ]]; then
    echo "Eval report not found at ${eval_file}."
    return 1
  fi

  local score
  score=$(extract_score "$eval_file")
  if [[ -z "$score" ]]; then
    echo "Could not extract score from eval report."
    return 1
  fi

  local changelog_file="${DRAFTS_DIR}/v${version}-changelog.md"
  local changes
  if [[ -f "$changelog_file" ]]; then
    changes=$(grep "^- " "$changelog_file" | head -3 | tr '\n' '; ' | cut -c1-100)
  else
    changes="(no changelog file)"
  fi
  local top_issue
  top_issue=$(grep -A1 "Priority Fix" "$eval_file" 2>/dev/null | tail -1 | cut -c1-80 || echo "N/A")

  append_progress "$version" "$score" "$changes" "$top_issue"
  echo "   Score: ${score}/100"

  # Store score in state.json for this iteration
  local tmp="${STATE_FILE}.tmp"
  jq --arg ch "$changes" ".iterations[${idx}].score = ${score} | .iterations[${idx}].changelog_summary = \$ch" "$STATE_FILE" > "$tmp" && mv "$tmp" "$STATE_FILE"
}

# --- Step: git_post_eval ---
step_git_post_eval() {
  local version=$1
  local score
  score=$(state_read ".iterations[$((version - 1))].score // 0")
  git add eval-reports/ progress.md && git commit -m "harness: v${version} eval — score ${score}/100" 2>/dev/null || true
}

# --- Step: exit_check ---
step_exit_check() {
  local version=$1
  local idx=$((version - 1))
  local score
  score=$(state_read ".iterations[${idx}].score // 0")

  # Threshold check
  if [[ "$score" -ge "$SCORE_THRESHOLD" ]]; then
    echo "Score threshold reached! (${score} >= ${SCORE_THRESHOLD})"
    state_update ".iterations[${idx}].steps.exit_check.result = \"threshold_reached\""
    state_update ".iterations[${idx}].status = \"completed\""
    state_finish "completed"
    return 0
  fi

  # Diminishing returns check
  if [[ $version -gt 1 ]]; then
    local prev_score
    prev_score=$(state_read ".iterations[$((idx - 1))].score // 0")
    if [[ $prev_score -gt 0 ]]; then
      local improvement=$((score - prev_score))
      if [[ $improvement -lt $MIN_IMPROVEMENT ]]; then
        # Check if previous iteration also had low improvement
        local prev_prev_score=0
        if [[ $version -gt 2 ]]; then
          prev_prev_score=$(state_read ".iterations[$((idx - 2))].score // 0")
        fi
        if [[ $prev_prev_score -gt 0 ]] && [[ $((prev_score - prev_prev_score)) -lt $MIN_IMPROVEMENT ]]; then
          echo "Diminishing returns (< ${MIN_IMPROVEMENT} for 2 consecutive iterations). Stopping."
          state_update ".iterations[${idx}].steps.exit_check.result = \"diminishing_returns\""
          state_finish "completed"
          return 0
        fi
      fi
    fi
  fi

  state_update ".iterations[${idx}].steps.exit_check.result = \"continue\""
  state_update ".iterations[${idx}].status = \"completed\""
  return 0
}

# ============================================================
# Handle --status
# ============================================================

if $SHOW_STATUS; then
  if [[ ! -f "$STATE_FILE" ]]; then
    echo "No state.json found. Harness has not been started."
    exit 0
  fi
  state_print_summary
  exit 0
fi

# ============================================================
# Handle --step (DAGU single-step mode)
# ============================================================

if [[ -n "$SINGLE_STEP" ]]; then
  if [[ -z "$SINGLE_ITERATION" ]]; then
    echo "ERROR: --step requires --iteration" >&2
    exit 1
  fi
  step_idx=$((SINGLE_ITERATION - 1))
  # Validate step name
  if ! declare -f "step_${SINGLE_STEP}" >/dev/null 2>&1; then
    echo "ERROR: Unknown step '${SINGLE_STEP}'. Valid steps: ${STEPS[*]}" >&2
    exit 1
  fi
  # Ensure iteration entry exists
  step_iter_count=$(jq '.iterations | length' "$STATE_FILE")
  if [[ $step_iter_count -le $step_idx ]]; then
    state_init_iteration "$SINGLE_ITERATION"
  fi
  run_step "$step_idx" "$SINGLE_STEP" "step_${SINGLE_STEP}" "$SINGLE_ITERATION"
  exit $?
fi

# ============================================================
# Main Loop (standalone mode)
# ============================================================

echo "╔══════════════════════════════════════════════════╗"
echo "║  ${TASK_NAME} Harness                           ║"
echo "║  Max iterations: ${MAX_ITERATIONS}              ║"
echo "║  Score threshold: ${SCORE_THRESHOLD}/100        ║"
echo "║  Cost cap: \$${MAX_COST_USD}                    ║"
echo "╚══════════════════════════════════════════════════╝"

# Initialize or resume
if $RESUME && [[ -f "$STATE_FILE" ]]; then
  START_VERSION=$(state_read '.current_iteration')
  echo "Resuming from v${START_VERSION}, step: $(state_read '.current_step')..."
else
  START_VERSION=1
  init_state
fi

if $DRY_RUN; then
  remaining=$((MAX_ITERATIONS - START_VERSION + 1))
  est=$(estimate_cost $remaining)
  echo "[DRY RUN] Would run ${remaining} iterations, estimated cost: \$${est}"
  exit 0
fi

for i in $(seq $START_VERSION $MAX_ITERATIONS); do
  echo ""
  echo "━━━ Iteration $i / $MAX_ITERATIONS ━━━"

  # Cost check
  completed=$((i - START_VERSION))
  spent=$(estimate_cost $completed)
  if (( $(echo "$spent > $MAX_COST_USD" | bc -l) )); then
    echo "Cost cap reached (\$${spent} > \$${MAX_COST_USD}). Stopping."
    state_finish "paused"
    break
  fi

  # Initialize iteration in state.json (if not resuming into an existing one)
  idx=$((i - 1))
  iter_count=$(jq '.iterations | length' "$STATE_FILE")
  if [[ $iter_count -le $idx ]]; then
    state_init_iteration "$i"
  fi

  # Run each step through the state machine
  run_step "$idx" "generator"     "step_generator"     "$i" || { state_finish "failed"; break; }
  run_step "$idx" "git_post_gen"  "step_git_post_gen"  "$i" || { state_finish "failed"; break; }
  run_step "$idx" "evaluator"     "step_evaluator"     "$i" || { state_finish "failed"; break; }
  run_step "$idx" "extract"       "step_extract"       "$i" || { state_finish "failed"; break; }
  run_step "$idx" "git_post_eval" "step_git_post_eval" "$i" || { state_finish "failed"; break; }
  run_step "$idx" "exit_check"    "step_exit_check"    "$i" || break  # exit_check returns 0 even when stopping

  # Check if harness was completed by exit_check
  harness_status=$(state_read '.status')
  if [[ "$harness_status" != "running" ]]; then
    break
  fi
done

# Final status
if [[ "$(state_read '.status')" == "running" ]]; then
  state_finish "completed"
fi

echo ""
echo "━━━ Run Complete ━━━"
echo "Progress: ${PROGRESS_FILE}"
echo "Latest: drafts/v$(get_last_version).${ARTIFACT_EXT}"
echo "State: ${STATE_FILE}"
echo "History: git log --grep='harness:' --oneline"
```

---

## Template Customization Notes

When generating from these templates:

1. **Replace ALL `[PLACEHOLDERS]`** — search for `[` and verify each is filled
2. **Adjust token estimates** based on task:
   - Short article: ~3K generator, ~2K evaluator
   - Long document: ~8K generator, ~4K evaluator
   - Code refactoring: ~5K generator, ~3K evaluator + test output
   - Code + browser: ~80K generator, ~60K evaluator (screenshots are expensive)
3. **Set `--allowedTools`** per agent — see prompt-templates.md for the table
4. **The score extraction regex** must match the eval prompt's output format exactly
5. **For Code Mode**: adapt the template using the patterns at the top of this file (validation + revert, dev server lifecycle, changelogs/ directory). Add `frozen_check` and `validation` steps between `generator` and `git_post_gen`. Set `STEPS=(generator frozen_check validation git_post_gen evaluator extract git_post_eval exit_check)`.
6. **For browser tasks**: add Playwright MCP tools to allowedTools and screenshot directory management
7. **jq is required**: Orchestrator checks at startup and prints install instructions if missing. No fallback — state.json requires correct JSON handling.

---

## Bash Orchestrator Template (Investigation Mode)

```bash
#!/usr/bin/env bash
set -euo pipefail

# ============================================================
# [TASK_NAME] Investigation Harness
# Generated by harness-create (Investigation Mode)
# ============================================================

# --- Dependency Check ---
if ! command -v jq &>/dev/null; then
  echo "ERROR: jq is required but not installed."
  echo "  macOS:  brew install jq"
  echo "  Ubuntu: sudo apt-get install jq"
  echo "  Arch:   sudo pacman -S jq"
  exit 1
fi

# --- Configuration ---
TASK_NAME="[TASK_NAME]"
MODE="investigation"
MAX_ROUNDS=[MAX_ROUNDS]        # Typically 3-5 for investigation
MAX_COST_USD="${MAX_COST:-999}" # Override with --max-cost flag
INVESTIGATOR_TOOLS="Read,Grep,Glob,Bash"

# Rough token estimates per round (adjust based on task)
EST_TOKENS_INVESTIGATOR=[INVEST_TOKENS]
COST_PER_1K_TOKENS=0.003       # Adjust for your model/tier

# Step sequence for Investigation Mode
STEPS=(investigator git_snapshot extract_evidence exit_check)

# --- Directories ---
HARNESS_DIR="$(cd "$(dirname "$0")" && pwd)"
EVIDENCE_DIR="${HARNESS_DIR}/evidence"
PROMPTS_DIR="${HARNESS_DIR}/prompts"
PROGRESS_FILE="${HARNESS_DIR}/progress.md"
SPEC_FILE="${HARNESS_DIR}/SPEC.md"
STATE_FILE="${HARNESS_DIR}/state.json"

# --- DAGU registration helpers (Key Patterns §10) ---
_dagu_dags_dir() {
  if [[ -n "${DAGU_DAGS_DIR:-}" ]]; then echo "$DAGU_DAGS_DIR"
  else dagu config 2>/dev/null | awk '/DAGs directory/{print $NF}' || echo "${HOME}/.dagu/dags"; fi
}
register_dagu() {
  if ! command -v dagu &>/dev/null; then
    echo "[dagu] Not installed — skipping DAG registration."
    return 0
  fi
  local dags_dir; dags_dir="$(_dagu_dags_dir)"; mkdir -p "$dags_dir"
  local link_name="${dags_dir}/harness-${TASK_NAME}.yaml"
  local target="${HARNESS_DIR}/dag.yaml"
  if [[ ! -f "$target" ]]; then echo "[dagu] ERROR: ${target} not found."; return 1; fi
  if [[ -L "$link_name" ]]; then echo "[dagu] Symlink already exists: ${link_name}"; return 0; fi
  ln -s "$target" "$link_name"
  echo "[dagu] Registered: ${link_name} → ${target}"
}
unregister_dagu() {
  local dags_dir; dags_dir="$(_dagu_dags_dir)"
  local link_name="${dags_dir}/harness-${TASK_NAME}.yaml"
  if [[ -L "$link_name" ]]; then rm "$link_name"; echo "[dagu] Unregistered: ${link_name}"
  else echo "[dagu] No symlink found at ${link_name}"; fi
}

# --- Flags ---
DRY_RUN=false
RESUME=false
SINGLE_STEP=""
SINGLE_ITERATION=""
SHOW_STATUS=false

while [[ $# -gt 0 ]]; do
  case "$1" in
    --dry-run)    DRY_RUN=true; shift ;;
    --resume)     RESUME=true; shift ;;
    --step)       SINGLE_STEP="$2"; shift 2 ;;
    --iteration)  SINGLE_ITERATION="$2"; shift 2 ;;
    --status)     SHOW_STATUS=true; shift ;;
    --max-cost)   MAX_COST_USD="$2"; shift 2 ;;
    --max-cost=*) MAX_COST_USD="${1#*=}"; shift ;;
    --register-dagu)   register_dagu; exit 0 ;;
    --unregister-dagu) unregister_dagu; exit 0 ;;
    *) echo "Unknown flag: $1"; exit 1 ;;
  esac
done

mkdir -p "$EVIDENCE_DIR"

# ============================================================
# state.json helpers
# ============================================================

init_state() {
  cat > "$STATE_FILE" <<STATEEOF
{
  "harness": "${TASK_NAME}",
  "mode": "${MODE}",
  "status": "running",
  "current_iteration": 1,
  "current_step": "${STEPS[0]}",
  "max_iterations": ${MAX_ROUNDS},
  "score_threshold": 0,
  "started_at": "$(date -u '+%Y-%m-%dT%H:%M:%SZ')",
  "updated_at": "$(date -u '+%Y-%m-%dT%H:%M:%SZ')",
  "iterations": []
}
STATEEOF
}

state_read() { jq -r "$1" "$STATE_FILE"; }

state_update() {
  local tmp="${STATE_FILE}.tmp"
  jq "$1" "$STATE_FILE" > "$tmp" && mv "$tmp" "$STATE_FILE"
}

state_init_iteration() {
  local version=$1
  state_update "
    .current_iteration = ${version} |
    .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\" |
    .iterations += [{\"version\":${version},\"status\":\"in_progress\",\"steps\":{}}]"
}

state_step_update() {
  local idx=$1 step=$2 data=$3
  state_update "
    .current_step = \"${step}\" |
    .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\" |
    .iterations[${idx}].steps.${step} = (.iterations[${idx}].steps.${step} // {}) * ${data}"
}

state_step_start() {
  state_step_update "$1" "$2" "{\"status\":\"running\",\"started_at\":\"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\"}"
}

state_step_complete() {
  local extra="${3:-"{}"}"
  state_step_update "$1" "$2" "({\"status\":\"completed\",\"completed_at\":\"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\"} * ${extra})"
}

state_step_fail() {
  local tmp="${STATE_FILE}.tmp"
  jq --arg err "$3" "
    .current_step = \"$2\" |
    .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\" |
    .iterations[$1].steps[\"$2\"] = (.iterations[$1].steps[\"$2\"] // {}) * {\"status\":\"failed\",\"error\":\$err}
  " "$STATE_FILE" > "$tmp" && mv "$tmp" "$STATE_FILE"
}

state_get_step_status() {
  jq -r ".iterations[${1}].steps[\"${2}\"].status // \"pending\"" "$STATE_FILE"
}

state_finish() {
  state_update ".status = \"${1}\" | .updated_at = \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\""
}

state_print_summary() {
  echo "╔══════════════════════════════════════════════════╗"
  echo "║  Harness: $(state_read '.harness')"
  echo "║  Mode: $(state_read '.mode')"
  echo "║  Status: $(state_read '.status')"
  echo "║  Round: $(state_read '.current_iteration') / $(state_read '.max_iterations')"
  echo "║  Current step: $(state_read '.current_step')"
  echo "║  Started: $(state_read '.started_at')"
  echo "║  Updated: $(state_read '.updated_at')"
  echo "╚══════════════════════════════════════════════════╝"
}

# ============================================================
# run_step wrapper
# ============================================================

run_step() {
  local idx=$1 step_name=$2 step_fn=$3
  shift 3

  local step_status
  step_status=$(state_get_step_status "$idx" "$step_name")
  if [[ "$step_status" == "completed" ]]; then
    echo "  [skip] ${step_name} (already completed)"
    return 0
  fi

  echo "  [run]  ${step_name}..."
  state_step_start "$idx" "$step_name"

  if $step_fn "$@"; then
    state_step_complete "$idx" "$step_name"
    echo "  [done] ${step_name}"
    return 0
  else
    state_step_fail "$idx" "$step_name" "step function returned non-zero"
    echo "  [FAIL] ${step_name}"
    return 1
  fi
}

# ============================================================
# Step functions
# ============================================================

append_progress() {
  local round=$1 hypothesis=$2 status=$3 summary=$4
  local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
  echo "| ${round} | ${timestamp} | ${hypothesis} | ${status} | ${summary} |" >> "$PROGRESS_FILE"
}

# --- Step: investigator ---
step_investigator() {
  local round=$1
  local prompt=$(cat "$PROMPTS_DIR/investigator.md")

  prompt="${prompt}

This is investigation round ${round} of ${MAX_ROUNDS}.
Read SPEC.md for symptoms and hypotheses.
Read progress.md for previous investigation results.
Read evidence/ directory for previously collected evidence.
Write your findings to evidence/ directory."

  claude -p "$prompt" --allowedTools "$INVESTIGATOR_TOOLS" 2>&1 || return 1
}

# --- Step: git_snapshot ---
step_git_snapshot() {
  local round=$1
  git add -A && git commit -m "investigate: round ${round}" --allow-empty 2>/dev/null || true
}

# --- Step: extract_evidence ---
step_extract_evidence() {
  local round=$1
  local idx=$((round - 1))

  local latest_evidence
  latest_evidence=$(ls -t "$EVIDENCE_DIR"/*.md 2>/dev/null | head -1)
  if [[ -n "$latest_evidence" ]]; then
    local hypothesis
    hypothesis=$(basename "$latest_evidence" .md)
    local status
    status=$(grep -oE '(CONFIRMED|ELIMINATED|INCONCLUSIVE)' "$latest_evidence" 2>/dev/null | head -1 || echo "UNKNOWN")
    local summary
    summary=$(grep -A1 -E '理由|Reasoning' "$latest_evidence" 2>/dev/null | tail -1 | cut -c1-80 || echo "N/A")
    append_progress "$round" "$hypothesis" "$status" "$summary"
    echo "   Hypothesis: ${hypothesis}"
    echo "   Status: ${status}"

    # Store in state.json
    state_update ".iterations[${idx}].hypothesis = \"${hypothesis}\" | .iterations[${idx}].hypothesis_status = \"${status}\""
  else
    append_progress "$round" "N/A" "NO_OUTPUT" "No evidence file produced"
    echo "   WARNING: No evidence file produced"
  fi
}

# --- Step: exit_check ---
step_exit_check() {
  local round=$1
  local idx=$((round - 1))

  # 1. Root cause found
  if grep -q "CONFIRMED" "$EVIDENCE_DIR"/*.md 2>/dev/null; then
    echo ""
    echo "Root cause CONFIRMED! Generating root cause report..."

    local confirmed_files
    confirmed_files=$(grep -l "CONFIRMED" "$EVIDENCE_DIR"/*.md 2>/dev/null)
    {
      echo "# Root Cause Report"
      echo ""
      echo "## Confirmed Root Cause(s)"
      echo ""
      for f in $confirmed_files; do
        echo "### $(basename "$f" .md)"
        cat "$f"
        echo ""
      done
      echo "## Investigation Summary"
      echo ""
      cat "$PROGRESS_FILE"
    } > "${HARNESS_DIR}/root-cause-report.md"

    git add -A && git commit -m "investigate: root cause confirmed" 2>/dev/null || true
    state_update ".iterations[${idx}].steps.exit_check.result = \"confirmed\""
    state_update ".iterations[${idx}].status = \"completed\""
    state_finish "completed"
    return 0
  fi

  # 2. All hypotheses eliminated
  local total_hypotheses
  total_hypotheses=$(grep -cE "^### H[0-9]" "$SPEC_FILE" 2>/dev/null || echo "0")
  local eliminated
  eliminated=$(grep -cl "ELIMINATED" "$EVIDENCE_DIR"/*.md 2>/dev/null | wc -l | tr -d ' ' || echo "0")
  if [[ "$eliminated" -ge "$total_hypotheses" && "$total_hypotheses" -gt 0 ]]; then
    echo ""
    echo "All ${total_hypotheses} hypotheses ELIMINATED. Dead end reached."
    echo "Check evidence/ for details. Consider generating new hypotheses."
    state_update ".iterations[${idx}].steps.exit_check.result = \"all_eliminated\""
    state_finish "completed"
    return 0
  fi

  state_update ".iterations[${idx}].steps.exit_check.result = \"continue\""
  state_update ".iterations[${idx}].status = \"completed\""
  return 0
}

# ============================================================
# Handle --status
# ============================================================

if $SHOW_STATUS; then
  if [[ ! -f "$STATE_FILE" ]]; then
    echo "No state.json found. Harness has not been started."
    exit 0
  fi
  state_print_summary
  exit 0
fi

# ============================================================
# Handle --step (DAGU single-step mode)
# ============================================================

if [[ -n "$SINGLE_STEP" ]]; then
  if [[ -z "$SINGLE_ITERATION" ]]; then
    echo "ERROR: --step requires --iteration" >&2
    exit 1
  fi
  step_idx=$((SINGLE_ITERATION - 1))
  # Validate step name
  if ! declare -f "step_${SINGLE_STEP}" >/dev/null 2>&1; then
    echo "ERROR: Unknown step '${SINGLE_STEP}'. Valid steps: ${STEPS[*]}" >&2
    exit 1
  fi
  # Ensure iteration entry exists
  step_iter_count=$(jq '.iterations | length' "$STATE_FILE")
  if [[ $step_iter_count -le $step_idx ]]; then
    state_init_iteration "$SINGLE_ITERATION"
  fi
  run_step "$step_idx" "$SINGLE_STEP" "step_${SINGLE_STEP}" "$SINGLE_ITERATION"
  exit $?
fi

# ============================================================
# Main Loop (standalone mode)
# ============================================================

echo "╔══════════════════════════════════════════════════╗"
echo "║  ${TASK_NAME} Investigation Harness              ║"
echo "║  Max rounds: ${MAX_ROUNDS}                       ║"
echo "╚══════════════════════════════════════════════════╝"

# Initialize or resume
if $RESUME && [[ -f "$STATE_FILE" ]]; then
  START_ROUND=$(state_read '.current_iteration')
  echo "Resuming from round ${START_ROUND}, step: $(state_read '.current_step')..."
else
  START_ROUND=1
  init_state
fi

if $DRY_RUN; then
  echo "[DRY RUN] Would run ${MAX_ROUNDS} investigation rounds"
  echo "Hypotheses from SPEC.md:"
  grep -E "^### H[0-9]" "$SPEC_FILE" 2>/dev/null || echo "(none found)"
  exit 0
fi

estimate_cost() {
  local rounds=$1
  local total_tokens=$(( rounds * EST_TOKENS_INVESTIGATOR ))
  echo "scale=2; $total_tokens * $COST_PER_1K_TOKENS / 1000" | bc
}

for i in $(seq $START_ROUND $MAX_ROUNDS); do
  echo ""
  echo "━━━ Investigation Round $i / $MAX_ROUNDS ━━━"

  # Cost check
  completed=$((i - START_ROUND))
  spent=$(estimate_cost $completed)
  if (( $(echo "$spent > $MAX_COST_USD" | bc -l) )); then
    echo "Cost cap reached (\$${spent} > \$${MAX_COST_USD}). Stopping."
    state_finish "paused"
    break
  fi

  # Initialize iteration in state.json
  idx=$((i - 1))
  iter_count=$(jq '.iterations | length' "$STATE_FILE")
  if [[ $iter_count -le $idx ]]; then
    state_init_iteration "$i"
  fi

  # Run each step through the state machine
  run_step "$idx" "investigator"     "step_investigator"     "$i" || { state_finish "failed"; break; }
  run_step "$idx" "git_snapshot"     "step_git_snapshot"     "$i" || { state_finish "failed"; break; }
  run_step "$idx" "extract_evidence" "step_extract_evidence" "$i" || { state_finish "failed"; break; }
  run_step "$idx" "exit_check"       "step_exit_check"       "$i" || break

  # Check if harness was completed by exit_check
  harness_status=$(state_read '.status')
  if [[ "$harness_status" != "running" ]]; then
    break
  fi
done

# Final status
if [[ "$(state_read '.status')" == "running" ]]; then
  state_finish "completed"
fi

echo ""
echo "━━━ Investigation Complete ━━━"
echo "Progress: ${PROGRESS_FILE}"
echo "Evidence: ${EVIDENCE_DIR}/"
echo "State: ${STATE_FILE}"
if [[ -f "${HARNESS_DIR}/root-cause-report.md" ]]; then
  echo "Root cause report: ${HARNESS_DIR}/root-cause-report.md"
fi
```

---

## DAGU YAML Templates

Generate `dag.yaml` alongside `harness.sh`. Both files read/write the same `state.json`. DAGU is optional — the harness works standalone without it.

### Document Mode dag.yaml

```yaml
# ============================================================
# [TASK_NAME] — DAGU DAG Definition
# Generated by harness-create
# Run: dagu start dag.yaml
# ============================================================

name: harness-[TASK_NAME]
description: "Overnight iteration harness for [TASK_NAME]"
tags: [harness, document]
type: graph

params:
  - ITERATION: "1"

env:
  - HARNESS_DIR: "[ABSOLUTE_HARNESS_DIR]"

steps:
  - name: generator
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step generator --iteration ${ITERATION}
    retry_policy:
      limit: 1
      interval_sec: 30

  - name: git_post_gen
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step git_post_gen --iteration ${ITERATION}
    depends: [generator]

  - name: evaluator
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step evaluator --iteration ${ITERATION}
    depends: [git_post_gen]
    retry_policy:
      limit: 1
      interval_sec: 30

  - name: extract
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step extract --iteration ${ITERATION}
    depends: [evaluator]

  - name: git_post_eval
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step git_post_eval --iteration ${ITERATION}
    depends: [extract]

  - name: exit_check
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step exit_check --iteration ${ITERATION}
    depends: [git_post_eval]
```

### Investigation Mode dag.yaml

```yaml
# ============================================================
# [TASK_NAME] Investigation — DAGU DAG Definition
# Generated by harness-create (Investigation Mode)
# Run: dagu start dag.yaml
# ============================================================

name: investigate-[TASK_NAME]
description: "Investigation harness for [TASK_NAME]"
tags: [harness, investigation]
type: graph

params:
  - ITERATION: "1"

env:
  - HARNESS_DIR: "[ABSOLUTE_HARNESS_DIR]"

steps:
  - name: investigator
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step investigator --iteration ${ITERATION}
    retry_policy:
      limit: 1
      interval_sec: 30

  - name: git_snapshot
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step git_snapshot --iteration ${ITERATION}
    depends: [investigator]

  - name: extract_evidence
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step extract_evidence --iteration ${ITERATION}
    depends: [git_snapshot]

  - name: exit_check
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step exit_check --iteration ${ITERATION}
    depends: [extract_evidence]
```

### Code Mode dag.yaml

```yaml
# ============================================================
# [TASK_NAME] — DAGU DAG Definition (Code Mode)
# Generated by harness-create
# Run: dagu start dag.yaml
# ============================================================

name: harness-[TASK_NAME]
description: "Overnight iteration harness for [TASK_NAME] (Code Mode)"
tags: [harness, code]
type: graph

params:
  - ITERATION: "1"

env:
  - HARNESS_DIR: "[ABSOLUTE_HARNESS_DIR]"

steps:
  - name: generator
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step generator --iteration ${ITERATION}
    retry_policy:
      limit: 1
      interval_sec: 30

  - name: frozen_check
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step frozen_check --iteration ${ITERATION}
    depends: [generator]

  - name: validation
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step validation --iteration ${ITERATION}
    depends: [frozen_check]

  - name: git_post_gen
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step git_post_gen --iteration ${ITERATION}
    depends: [validation]

  - name: evaluator
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step evaluator --iteration ${ITERATION}
    depends: [git_post_gen]
    retry_policy:
      limit: 1
      interval_sec: 30

  - name: extract
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step extract --iteration ${ITERATION}
    depends: [evaluator]

  - name: git_post_eval
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step git_post_eval --iteration ${ITERATION}
    depends: [extract]

  - name: exit_check
    command: bash
    script: |
      cd "${HARNESS_DIR}" && bash harness.sh --step exit_check --iteration ${ITERATION}
    depends: [git_post_eval]
```

### DAGU Usage Notes

- **Running a single iteration:** `dagu start dag.yaml --params "ITERATION=3"`
- **Running a loop:** Use DAGU's scheduler or a wrapper script that increments `ITERATION` and checks `state.json` status after each DAG run.
- **Retries:** DAGU handles per-step retries. The `retry_policy` config on `generator` and `evaluator` steps allows one automatic retry with a 30s delay.
- **Visualization:** Each step appears as a node in the DAGU web UI graph. Step logs, durations, and statuses are visible per-node.
- **Without DAGU:** Just run `bash harness.sh` — the standalone loop works identically, using the same step functions and state.json.
- **Auto-registration:** At harness creation time, `register_dagu` symlinks `dag.yaml` into DAGU's DAGs directory (see Key Patterns §10). Re-run manually with `harness.sh --register-dagu`. Remove with `harness.sh --unregister-dagu`.
