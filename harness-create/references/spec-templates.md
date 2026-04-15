# HARNESS_SPEC.md Templates

Output templates for Phase 1. Generate the spec using the appropriate template after completing all interview phases.

---

## Route A: Iterative Improvement Template

```markdown
# Harness Specification

## Task
- **Artifact**: [what is being worked on]
- **Current state**: [starting point]
- **Target audience**: [who judges the output]
- **Goal**: [one-sentence optimization target]

## Frozen Constraints
[Things that must NOT change during the run]
- ...

## Eval Rubric

### Scoring Dimensions
| # | Dimension | Weight | Detection Method |
|---|-----------|--------|------------------|
| 1 | ... | /100 | ... |

### Dimension Details
#### [Dimension 1 Name]
- **5/5**: ...
- **3/5**: ...
- **1/5**: ...

### Penalty Rules
- [Specific penalizable patterns with point deductions]

### Threshold
- **Pass score**: X/100
- **Target score**: Y/100

## Agent Architecture
### Generator
- **Role**: ...
- **Perspective**: ...
- **Input**: ...
- **Output**: ...

### Evaluator
- **Role**: ...
- **Perspective**: ...
- **Input**: ...
- **Output**: ...
- **Isolation**: Separate context window (mandatory)

[Additional agents if needed]

## Exit Conditions
- ...

## Progress Tracking
- **Log file**: progress.md
- **Per-iteration record**: version number, timestamp, total score, per-dimension scores, key changes summary, evaluator's top unresolved issue

## Estimated Resource Usage
- **Iterations**: ~N expected
- **Tokens per iteration**: ~X (generator) + ~Y (evaluator)
- **Total estimated cost**: ~$Z
```

---

## Route B: Investigation Template

```markdown
# Investigation Harness Specification

## Symptom
[Precise, verifiable description of the bug]

## Expected Behavior
[What should happen instead]

## Reproduction Steps
[Exact steps to trigger the bug]

## Code Path
[Complete call chain from user action to bug manifestation]
```
[user action]
  → [layer 1]: [file/function]
    → [layer 2]: [file/function]
      → ... → [bug manifests here]
```

## Hypotheses (ranked by likelihood)

### H1: [Name] (Likelihood: high)
- **Claim**: ...
- **Verification method**: ...
- **Expected evidence if TRUE**: ...
- **Expected evidence if FALSE**: ...
- **Files to inspect**: ...

### H2: [Name] (Likelihood: high)
...

### H3: [Name] (Likelihood: medium)
...

## Evidence Collection Plan
[Ordered list of verification steps, referencing hypotheses]

## Agent Architecture

### Investigator
- **Role**: Systematically verify hypotheses by collecting code evidence
- **Perspective**: You are a debugger who tests hypotheses against evidence. You do NOT fix bugs — you find root causes.
- **Input**: SPEC.md (symptoms + hypotheses), source code files
- **Output**: `evidence/h{N}-{name}.md` per hypothesis + `root-cause-report.md`
- **Isolation**: Fresh context per round (mandatory)
- **Key constraint**: One hypothesis per round. Do not skip to fixing.

### Fix Verifier (optional, if fix is in scope)
- **Role**: Verify that a proposed fix resolves the original symptom
- **Perspective**: Skeptical tester who tries to reproduce the original bug after fix
- **Input**: Root cause report + fix patch
- **Output**: Pass/fail with evidence

## Exit Conditions
- **Root cause confirmed**: At least 1 hypothesis has CONFIRMED status with code evidence
- **Max rounds**: N (typically 3-5 for investigation tasks)
- **Dead end**: All hypotheses ELIMINATED → generate new hypotheses or escalate to human
- **Combination root cause**: Multiple hypotheses confirmed → document the interaction

## Modifiable Files
[Files the investigator may add debug logs to — clearly separated from source files]

## Frozen Files
[Files that must NOT be modified, even for debug logging]
```

After generating either template, review the spec with the user. Confirm each hypothesis or dimension makes sense. Make adjustments before handing off to Phase 2.
