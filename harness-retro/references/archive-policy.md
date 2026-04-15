# Archive Policy

Rules for what to keep, compress, and remove when archiving completed harness tasks.

---

## Prerequisites

- Retro report (RETRO.md) must be generated BEFORE archiving
- User must explicitly confirm the archive operation
- Only archive tasks with status `completed` or `failed` in state.json

---

## What to Keep in Place

These files stay in their original location within the task directory — they are the permanent record:

| File | Reason |
|------|--------|
| `SPEC.md` | The frozen target — needed for future reference |
| `RETRO.md` | The retrospective findings — the learning output |
| `progress.md` | Score history — quick reference for convergence |
| `EVAL_CRITERIA.md` | Rubric used — context for understanding the retro |
| `README.md` | Task documentation |
| Final eval report | The last `eval-reports/eval-v{N}.md` — the terminal assessment |

---

## What to Compress

Bundle into `_archive/{task-name}/artifacts.tar.gz`:

| Directory/Files | Reason for compressing |
|----------------|----------------------|
| `eval-reports/` (except final) | Intermediate evaluations — useful for deep analysis but not day-to-day |
| `changelogs/` | Iteration-by-iteration changes — bulky but occasionally needed |
| `prompts/` | Agent prompts — useful to compare against future templates |
| `drafts/` or `output/` | Intermediate artifacts — the largest files, rarely re-read |
| `evidence/` (investigation mode) | Hypothesis evidence files |
| `screenshots/` | Visual artifacts if present |

---

## What to Remove

Remove only after compression is verified:

| Item | Condition |
|------|-----------|
| DAGU symlink | Run `bash harness.sh --unregister-dagu` to remove `~/.dagu/dags/harness-{task-name}.yaml` |
| `state.json` lock fields | Clear `running_step` and `pid` fields if present (stale process info) |

**Never delete** the task directory itself, state.json, or any file listed in "Keep in Place" above.

---

## Archive Directory Structure

```
.harness-workspace/
├── _archive/
│   └── {task-name}/
│       ├── artifacts.tar.gz      ← compressed intermediate files
│       └── archived-at.txt       ← timestamp of archive operation
├── {task-name}/                   ← original directory (kept files remain)
│   ├── SPEC.md
│   ├── RETRO.md
│   ├── progress.md
│   ├── EVAL_CRITERIA.md
│   ├── README.md
│   ├── state.json
│   └── eval-reports/
│       └── eval-v{final}.md      ← only the final report remains
```

---

## Archive Procedure

1. Verify RETRO.md exists in the task directory
2. Create `_archive/{task-name}/` directory
3. Run `tar -czf _archive/{task-name}/artifacts.tar.gz` on compressible files
4. Write `archived-at.txt` with ISO 8601 timestamp
5. Verify tarball is valid: `tar -tzf artifacts.tar.gz > /dev/null`
6. Remove compressed source files (only after step 5 succeeds)
7. Run `bash harness.sh --unregister-dagu` if harness.sh exists
8. Report summary: files kept, files compressed, bytes saved
