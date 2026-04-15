# Progress Monitoring with DAGU

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
