# Retro Report Template

Output format for retrospective reports. Use this template when generating RETRO.md files.

---

## Single-Task Report

```markdown
# Harness Retro: {task-name}

**Mode:** {document|code|investigation} | **Iterations:** {N} | **Final Score:** {score}/{max} | **Period:** {start-date} — {end-date}
**Exit Condition:** {met|not met} — {reason}

---

## Score Trajectory

v1:  ████░░░░░░  42/100
v2:  ██████░░░░  55/100  (+13)
v3:  ████████░░  68/100  (+13)
v4:  █████████░  72/100  (+4)
v5:  █████████░  74/100  (+2)
v6:  █████████░  75/100  (+1)

Useful iterations: 6/6 (100%) | Cost per point: 0.18 iter/pt

---

## Findings

### 1. Convergence — {healthy|concern|problem}

**Evidence:** {specific numbers from the score trajectory}

**Impact:** {what this pattern cost — wasted iterations, missed exit, etc.}

**Suggestion:** {concrete change, if concern or problem}

### 2. Bottleneck Dimensions — {healthy|concern|problem}

**Evidence:** {dimension name, scores across iterations}

**Impact:** {how this dimension affected overall score}

**Suggestion:** {specific rubric or prompt change}

### 3. Repeated Work — {healthy|concern|problem}

**Evidence:** {repeated action, which changelogs, which eval reports}

**Impact:** {wasted effort}

**Suggestion:** {evaluator or generator prompt change}

### 4. Cost Efficiency — {healthy|concern|problem}

**Evidence:** {useful ratio, cost per point, diminishing returns inflection}

**Impact:** {wasted compute/time}

**Suggestion:** {exit condition or plateau detection change}

### 5. Prompt Quality — {healthy|concern|problem}

**Evidence:** {specific prompt issue found}

**Impact:** {downstream effect on generation or evaluation}

**Suggestion:** {before/after diff for the prompt file}

### 6. Cross-Task Patterns — N/A (single task)

---

## What Went Well

- {positive finding 1 — e.g., "Monotonic convergence with no regressions"}
- {positive finding 2 — e.g., "Evaluator feedback was consistently actionable — generator improved every iteration"}
- {positive finding 3}

---

## Recommended Actions (prioritized)

### 1. {highest impact action}

**Dimension:** D{N} {dimension name}
**File:** `{file path}`

```diff
- {old line}
+ {new line}
```

**Expected impact:** {what this change would improve}

### 2. {next action}

...

---

## Summary

| Dimension | Status | Key Finding |
|-----------|--------|-------------|
| Convergence | {status} | {one-line summary} |
| Bottleneck Dimensions | {status} | {one-line summary} |
| Repeated Work | {status} | {one-line summary} |
| Cost Efficiency | {status} | {one-line summary} |
| Prompt Quality | {status} | {one-line summary} |
| Cross-Task Patterns | N/A | Single task |
```

---

## Multi-Task Report

When analyzing multiple tasks, use this wrapper format. Save as `.harness-workspace/RETRO-{date}.md`.

```markdown
# Harness Retro: Cross-Task Analysis

**Tasks analyzed:** {N} | **Date:** {date}
**Scope:** {list of task names}

---

## Task Summary

| Task | Mode | Iterations | Final Score | Exit Met | Key Issue |
|------|------|-----------|-------------|----------|-----------|
| {name} | {mode} | {N} | {score}/{max} | {yes/no} | {one-line} |
| ... | ... | ... | ... | ... | ... |

---

## Per-Task Findings

### {task-name-1}

{abbreviated single-task report — trajectory + findings table only}

### {task-name-2}

...

---

## Cross-Task Patterns (D6)

### Pattern: {pattern name}

**Affected tasks:** {list}
**Evidence:** {what was observed across tasks}
**Root cause:** {why this happens — likely a template or default issue}
**Suggestion:** {concrete change to template files}

```diff
- {old template line}
+ {new template line}
```

---

## Recommended Actions (prioritized across all tasks)

### 1. {highest systemic impact}

**Affects:** {which tasks}
**File:** `{template file path}`

```diff
- {old}
+ {new}
```

### 2. ...

---

## Summary

| Dimension | Healthy | Concern | Problem |
|-----------|---------|---------|---------|
| Convergence | {N tasks} | {N tasks} | {N tasks} |
| Bottleneck Dimensions | {N} | {N} | {N} |
| Repeated Work | {N} | {N} | {N} |
| Cost Efficiency | {N} | {N} | {N} |
| Prompt Quality | {N} | {N} | {N} |
| Cross-Task Patterns | — | — | {status} |
```

---

## Formatting Rules

1. **Score trajectory bar**: Use full-width block characters. Each `█` = 10% of max score. Use `░` for remaining. Always show delta in parentheses after first iteration.

2. **Diffs in suggestions**: Use standard diff format (`-`/`+` lines). Reference the exact file path. Include enough context lines for the user to locate the change.

3. **Evidence**: Always cite specific iteration numbers, dimension names, file paths, or timestamps. Never say "some iterations had issues" — say "iterations 4-7 scored within 1 point of each other (74, 74, 75, 75)".

4. **Prioritization**: Order recommended actions by estimated impact. Systemic issues (affecting templates) outrank task-specific issues. Convergence and repeated-work fixes outrank cost-efficiency optimizations.

5. **Investigation mode tasks**: No score trajectory. Instead, show hypothesis status timeline:
   ```
   H1 (auth timeout):     ACTIVE → ELIMINATED (v3)
   H2 (race condition):   ACTIVE → ACTIVE → CONFIRMED (v4)
   H3 (memory leak):      ACTIVE → ELIMINATED (v2)
   ```
