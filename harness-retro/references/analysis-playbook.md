# Analysis Playbook

Detailed procedures for the 6 retrospective analysis dimensions. Each dimension follows the same structure: what to look for, how to detect it, how to classify severity, and what to recommend.

---

## Severity Classification

Every dimension produces a status:

| Status | Meaning | Action |
|--------|---------|--------|
| **healthy** | No issues detected, or minor variance within acceptable bounds | Note as positive finding |
| **concern** | Suboptimal pattern detected, but task still converged | Generate improvement suggestion |
| **problem** | Pattern actively hurt task quality, cost, or convergence | Generate improvement suggestion, flag as priority |

All findings must cite **specific evidence**: iteration numbers, scores, file paths, timestamps. Never report a status without data to support it.

---

## Dimension 1: Convergence

**Question:** Did the score trajectory move efficiently toward the exit condition?

### Data Sources
- `progress.md` — score history per iteration
- `state.json` — iteration count, status, timestamps

### Detection Procedure

1. Extract the score sequence from progress.md (e.g., `[42, 55, 68, 72, 74, 75]`)
2. Compute iteration-over-iteration deltas (e.g., `[+13, +13, +4, +2, +1]`)
3. Check for these patterns:

**Plateau** — 3+ consecutive iterations with delta <= 1 point:
- Evidence: "Iterations 4-7 scored 74, 74, 75, 75 — plateau for 4 iterations"
- Impact: Wasted iterations after diminishing returns
- Suggestion: Lower exit threshold, add plateau-detection exit condition, or improve evaluator feedback specificity

**Regression** — Score drops by > 3 points between consecutive iterations:
- Evidence: "Score dropped from 72 (v5) to 64 (v6)"
- Impact: Generator undid previous good work
- Suggestion: Add regression detection to orchestrator (`--revert-on-regression`), tighten generator constraints

**Oscillation** — Score alternates up/down for 4+ iterations (no net improvement):
- Evidence: "v3: 65, v4: 58, v5: 67, v6: 60 — oscillating without net gain"
- Impact: Generator is fighting itself; evaluator feedback may be contradictory
- Suggestion: Review evaluator rubric for conflicting dimensions, add "do not undo previous improvements" constraint

**Slow convergence** — Task used > 70% of max_iterations but reached exit condition:
- Evidence: "Used 14 of 15 max iterations to reach 80/100"
- Impact: Cutting it close; task was nearly abandoned
- Suggestion: Review starting prompt for better initial quality, review evaluator for overly harsh scoring

**Fast convergence** — Task reached exit condition in <= 2 iterations:
- Evidence: "Reached 85/100 in 2 iterations"
- Impact: Positive — but may indicate exit threshold is too low or task was too easy
- Suggestion: Consider raising exit threshold, or note as well-calibrated if quality is genuinely high

### Classification
- **healthy**: Monotonic improvement, reached exit in reasonable iterations, no plateaus/regressions
- **concern**: One plateau OR one minor regression (< 5 points) OR slow convergence
- **problem**: Multiple regressions OR oscillation OR never reached exit condition

---

## Dimension 2: Bottleneck Dimensions

**Question:** Were there eval dimensions that never improved, dragging the overall score?

### Data Sources
- `eval-reports/` — per-dimension scores across iterations
- `EVAL_CRITERIA.md` — dimension definitions and weights

### Detection Procedure

1. Parse each eval report to extract per-dimension scores across all iterations
2. Build a matrix: rows = dimensions, columns = iterations
3. For each dimension, compute:
   - Starting score (v1)
   - Final score (last iteration)
   - Max score ever achieved
   - Net change (final - starting)

4. Flag dimensions where:

**Stuck** — Net change <= 0 across all iterations:
- Evidence: "Dimension 'Error Handling' scored 2/5 in v1 and 2/5 in v8 (final)"
- Impact: This dimension contributed 0 improvement despite {N} iterations of effort
- Suggestion: Review evaluator feedback for this dimension — is the feedback actionable? Does the generator know how to improve it? Consider adding specific examples to the generator prompt.

**Capped** — Dimension reached a ceiling early and never improved past it:
- Evidence: "Dimension 'Test Coverage' hit 4/5 in v2, remained 4/5 through v8"
- Impact: If 5/5 is achievable, the evaluator may not be giving specific enough feedback on what's missing
- Suggestion: Add concrete criteria for full marks in evaluator rubric

**Sacrificed** — Dimension improved then regressed while other dimensions improved:
- Evidence: "Dimension 'Readability' was 4/5 in v3, dropped to 2/5 in v5 when 'Performance' jumped from 2/5 to 4/5"
- Impact: Generator is trading off dimensions instead of improving all
- Suggestion: Add "do not sacrifice existing dimension scores" constraint to generator prompt; consider making this dimension frozen once it reaches a threshold

### Classification
- **healthy**: All dimensions improved or maintained, no stuck dimensions
- **concern**: 1 stuck dimension, or 1 capped dimension at >= 80% of max
- **problem**: 2+ stuck dimensions, or any sacrificed dimension

---

## Dimension 3: Repeated Work

**Question:** Did the generator attempt the same fix multiple times? (Indicates evaluator feedback isn't actionable.)

### Data Sources
- `changelogs/` — what changed in each iteration
- `eval-reports/` — feedback given after each iteration

### Detection Procedure

1. Read all changelogs in order
2. For each changelog, extract the list of changes made (summarize to action + target, e.g., "added error handling to API module", "refactored auth flow")
3. Cross-reference: if the same action+target appears in 3+ changelogs, flag as repeated work

**Same fix, different iteration:**
- Evidence: "Added input validation to form handler' appears in changelogs for v2, v4, and v6"
- Impact: Generator keeps applying the same fix but evaluator keeps flagging it — either the fix doesn't stick (generator overwrites it) or evaluator doesn't recognize the fix
- Suggestion: Check evaluator criteria — is the detection method specific enough to recognize the fix? Check generator prompt — does it have access to previous changelogs?

**Revert-and-redo cycles:**
- Evidence: "v3 changelog: 'Removed caching layer'. v5 changelog: 'Added caching layer'. v7 changelog: 'Removed caching layer'"
- Impact: Generator is oscillating on a design decision; evaluator feedback is contradictory
- Suggestion: Add architectural decision constraint to generator prompt ("once a design decision is made, do not reverse it unless evaluator explicitly flags it as harmful")

4. Also check: does the evaluator feedback for iteration N reference the same issue it flagged in iteration N-2? If so, the generator isn't reading or acting on the feedback.

### Classification
- **healthy**: No repeated actions across changelogs
- **concern**: 1 action repeated 2 times
- **problem**: Any action repeated 3+ times, or 2+ revert-and-redo cycles

---

## Dimension 4: Cost Efficiency

**Question:** Were iterations spent productively, or was effort wasted?

### Data Sources
- `state.json` — iteration count, timestamps, step durations
- `progress.md` — score per iteration

### Detection Procedure

1. Compute **useful iteration ratio**: iterations that improved the score / total iterations
   - An iteration is "useful" if its score > previous iteration's score

2. Compute **cost per point**: total iterations / total score improvement
   - Example: 10 iterations, score went from 40 to 80 → 10/40 = 0.25 iterations per point

3. Check for **diminishing returns inflection**: the iteration after which no iteration improved score by more than 2 points
   - Example: "After iteration 5, no single iteration improved score by more than 2 points. Final 5 iterations contributed only 6 points."

4. If timestamps available, compute **step duration trends**:
   - Generator step getting slower each iteration (context accumulation)
   - Evaluator step time stable (good) vs growing (reading too much history)

**Low useful ratio** (< 50%):
- Evidence: "Only 4 of 10 iterations improved the score (40% useful ratio)"
- Impact: 60% of compute/time was wasted
- Suggestion: Add early termination on plateau, improve generator prompt to avoid regressions

**High cost per point** (> 0.5 iterations/point for tasks scoring out of 100):
- Evidence: "0.8 iterations per point gained"
- Impact: Inefficient — generator is making small incremental changes
- Suggestion: Review generator prompt for boldness constraints, consider batch feedback approach

**Late-stage waste** — Final 30% of iterations contributed < 10% of total improvement:
- Evidence: "Last 3 of 10 iterations contributed 4 of 40 total points (10%)"
- Impact: Could have stopped earlier with similar result
- Suggestion: Add plateau exit condition, lower exit threshold if current one is rarely reached

### Classification
- **healthy**: Useful ratio >= 70%, no late-stage waste
- **concern**: Useful ratio 50-69%, or mild late-stage waste
- **problem**: Useful ratio < 50%, or severe diminishing returns (>40% of iterations wasted)

---

## Dimension 5: Prompt Quality

**Question:** Did the prompts (generator, evaluator, spec) serve the task well?

### Data Sources
- `prompts/` — generator.md, evaluator.md, and any other agent prompts
- `eval-reports/` — evaluator output quality
- `SPEC.md` — the frozen target
- `EVAL_CRITERIA.md` — rubric

### Detection Procedure

1. **Spec drift check**: Compare SPEC.md to what the generator actually produced (final artifact). Does the final artifact address all spec requirements?
   - Evidence: "SPEC.md requires 'mobile-responsive design' but no eval report mentions mobile testing"
   - Suggestion: Add missing requirement to evaluator rubric

2. **Fresh context warning**: Check if generator prompt includes a fresh context warning (instruction to re-read relevant files at the start of each iteration)
   - Evidence: "Generator prompt does not include fresh context injection"
   - Suggestion: Add "IMPORTANT: You are starting with no prior context..." block to generator prompt

3. **Evaluator feedback quality**: Sample 3 eval reports and check:
   - Does feedback cite specific locations/lines/sections? (actionable)
   - Does feedback say what to change, not just what's wrong? (directive)
   - Does feedback prioritize issues? (structured)
   - Evidence: "Evaluator feedback in v3 says 'error handling needs improvement' without specifying which functions or what kind of error handling"
   - Suggestion: Add "cite specific file:line" and "provide concrete fix suggestion" instructions to evaluator prompt

4. **Tool permission check**: Does the generator prompt include `--allowedTools` appropriate for the task?
   - Evidence: "Generator has Write tool but no Read tool — can create files but can't read existing code"
   - Suggestion: Add missing tools to allowedTools list

5. **Starting-point injection**: Does the orchestrator inject context about what changed in previous iterations?
   - Evidence: "Generator prompt does not receive previous evaluator feedback or changelog"
   - Suggestion: Add iteration context injection to orchestrator

### Classification
- **healthy**: Fresh context present, evaluator feedback is actionable, spec fully covered
- **concern**: 1-2 minor prompt issues (e.g., missing fresh context warning)
- **problem**: Evaluator feedback consistently non-actionable, or spec drift detected, or missing tool permissions

---

## Dimension 6: Cross-Task Patterns

**Question:** Are there systemic issues that appear across multiple tasks? (Multi-task analysis only.)

### Data Sources
- All of the above, across all selected tasks

### Detection Procedure

> **Gate**: Only run this dimension when 2+ tasks are selected. For single-task retros, mark as "N/A — single task selected".

1. **Template issues**: Do multiple tasks show the same problem (e.g., all generators lack fresh context, all evaluators give non-actionable feedback)?
   - Evidence: "3 of 4 tasks had non-actionable evaluator feedback (D3 repeated work flagged in all)"
   - Impact: The prompt templates in `harness-create/references/` may need updating
   - Suggestion: Patch specific template file with concrete diff

2. **Mode-specific issues**: Do all tasks of the same mode (document/code/investigation) share a problem?
   - Evidence: "Both code-mode tasks had regression issues (D1) — the code mode revert threshold may be too lenient"
   - Suggestion: Adjust revert threshold in orchestrator template

3. **Convergence patterns**: What's the average useful iteration ratio across tasks? Average iterations to convergence?
   - Evidence: "Average useful ratio: 55%. Average iterations: 8.5 of 10 max."
   - Impact: System-wide efficiency could improve
   - Suggestion: Review default max_iterations and exit thresholds in spec template

4. **Rubric calibration**: Do tasks with similar goals have similar score distributions?
   - If scores cluster differently for similar tasks, rubric calibration may be inconsistent

### Classification
- **healthy**: No shared patterns across tasks, or shared patterns are positive
- **concern**: 1 shared concern pattern across tasks
- **problem**: Same problem dimension flagged in 3+ tasks, indicating systemic template issue
