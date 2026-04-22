# Archive Policy

Rules for what to keep, compress, and remove when archiving completed harness tasks.

---

## Prerequisites

- **Full retro path (Entry A → Phase 4):** RETRO.md must exist before archiving (Phase 3 just created it)
- **Standalone archive path (Entry B → Phase 4A):** RETRO.md recommended but not required — user must acknowledge the warning if missing
- User must explicitly confirm the archive operation
- Only archive tasks with status `completed` or `failed` in state.json

---

## What Gets Moved

The entire task directory is moved as-is to `_archive/`. All files are preserved — nothing is compressed or deleted.

---

## What to Clean Up

| Item | Action |
|------|--------|
| `state.json` lock fields | Clear `running_step` and `pid` fields if present (stale process info) |
| DAGU symlink | **Update** (not remove) — re-point `~/.dagu/dags/harness-{task-name}.yaml` to the new `_archive/{task-name}/dag.yaml` path so execution history remains visible in the DAGU web UI |

---

## Archive Directory Structure

```
.harness-workspace/
├── _archive/
│   └── {task-name}/               ← entire task, moved here intact
│       ├── SPEC.md
│       ├── RETRO.md               (may be absent in standalone archive)
│       ├── progress.md
│       ├── EVAL_CRITERIA.md
│       ├── README.md
│       ├── state.json
│       ├── dag.yaml
│       ├── harness.sh
│       ├── eval-reports/          ← all reports preserved
│       ├── changelogs/
│       ├── prompts/
│       └── archived-at.txt        ← timestamp of archive operation
├── {active-task}/                  ← only active tasks remain here
```

DAGU symlink after archive:
```
~/.dagu/dags/harness-{task-name}.yaml → .harness-workspace/_archive/{task-name}/dag.yaml
```

---

## Archive Procedure

1. Check for RETRO.md in the task directory. If missing: warn and allow (standalone archive path) or abort (full retro path — should not happen since Phase 3 creates it)
2. Clear stale lock fields (`running_step`, `pid`) in state.json
3. Move `.harness-workspace/{task-name}/` → `.harness-workspace/_archive/{task-name}/`
4. Write `archived-at.txt` with ISO 8601 timestamp inside the archived directory
5. Update DAGU symlink: re-point `~/.dagu/dags/harness-{task-name}.yaml` → `_archive/{task-name}/dag.yaml` (if symlink existed)
6. Report summary: task name, file count, new location
