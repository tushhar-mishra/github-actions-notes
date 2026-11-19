# 14 · Job Control Options: `timeout-minutes`, `continue-on-error`, `defaults`, `shell`, `working-directory`

> **Part B: Triggers and logic** · Previous: [13 · Conditional execution](13-conditional-execution.md) · Next: [15 · Actions and the Marketplace](15-actions-and-marketplace.md) · [Back to INDEX](INDEX.md)
>
---

## What it is

A few small keys that control **how** jobs and steps run:

| Key | Level | What it does |
|---|---|---|
| `timeout-minutes` | job or step | Kill it if it runs longer than N minutes. |
| `continue-on-error` | job or step | If it fails, **don't** fail the job/workflow. |
| `defaults.run.shell` | workflow or job | Default shell for all `run:` steps. |
| `defaults.run.working-directory` | workflow or job | Default folder for all `run:` steps. |
| `shell` | step | Shell for **this** step only. |
| `working-directory` | step | Folder for **this** step only. |
| `name` | job or step | The label shown in the UI. |
| `id` | step | Name used in `steps.<id>.*` (topic 13, 21). |
| `env` | workflow/job/step | Variables (topic 11). |

---

## Why it exists

- A hung test or a stuck `docker pull` would otherwise wait for the **default 360 minutes (6 hours)**, using minutes and blocking deploys → **`timeout-minutes`**.
- Some checks are "nice to know", not "must pass" (an experimental linter, a flaky beta test) → **`continue-on-error`**.
- Monorepos have code in `backend/` and `frontend/`. Writing `cd backend &&` in every step is noisy and error-prone → **`working-directory`**.
- Windows defaults to PowerShell and Linux to Bash. You want one consistent shell → **`shell` / `defaults`**.

**Jenkins comparison:** `options { timeout(time: 10, unit: 'MINUTES') }` ≈ `timeout-minutes`. `catchError(buildResult: 'SUCCESS')` ≈ `continue-on-error`. `dir('backend') { … }` ≈ `working-directory`.

---

## Analogy

Running a **marathon relay team**:

- `timeout-minutes` = "if a runner hasn't finished in 30 minutes, pull them off the track."
- `continue-on-error` = "this runner is in training. If they drop out, the team result still counts."
- `defaults.working-directory` = "everyone starts from the north gate unless told otherwise."
- `shell` = "this leg must be run in spiked shoes instead of trainers."

---

## How it works inside

- **Timeouts:** the runner tracks the wall-clock time. When the limit is reached, it **cancels** the step or job (sends a stop signal to the processes). The result is **failure** for a step timeout and **cancelled** for a job timeout, and the log shows `The job running on runner … has exceeded the maximum execution time of N minutes.`
- **continue-on-error at step level:** the step's **outcome** = `failure`, but its **conclusion** = `success`. Later steps see `success()` as true. `steps.<id>.outcome` still tells you it failed.
- **continue-on-error at job level:** the job shows failed (red), but the **workflow run** is not marked failed because of it, and jobs that `need` it still run.
- **defaults:** a step's own `shell:` / `working-directory:` wins over job `defaults`, which wins over workflow `defaults`. Defaults apply **only to `run:` steps**, not to `uses:` actions.
- **shell values:** `bash`, `sh`, `pwsh` (PowerShell 7), `powershell` (Windows PowerShell 5), `cmd`, `python`, or a custom template like `perl {0}` (`{0}` = the temp script file).

---

## Workflow YAML / commands

```yaml
name: Job Control Demo
on: workflow_dispatch

defaults:
  run:
    shell: bash                      # every run: step uses bash (also on Windows → Git Bash)

jobs:
  backend:
    runs-on: ubuntu-latest
    timeout-minutes: 15              # whole job killed after 15 min (default 360)
    defaults:
      run:
        working-directory: backend   # every run: step starts in ./backend
    steps:
      - uses: actions/checkout@v7

      - name: Show folder
        run: pwd                     # → /home/runner/work/repo/repo/backend

      - name: Experimental linter (allowed to fail)
        id: lint
        continue-on-error: true
        run: exit 1

      - name: Check what happened
        run: |
          echo "outcome    = ${{ steps.lint.outcome }}"
          echo "conclusion = ${{ steps.lint.conclusion }}"

      - name: Script in another folder
        working-directory: scripts   # step-level override (relative to the workspace, not backend)
        run: ls

      - name: Hanging step
        timeout-minutes: 1           # this step alone is limited to 1 minute
        run: sleep 120

      - name: Python as the shell
        shell: python
        run: |
          import platform
          print("Python on", platform.system())
```

**Sample log output:**

```text
✓ Show folder
    /home/runner/work/repo/repo/backend
⚠ Experimental linter (allowed to fail)
    Error: Process completed with exit code 1.     ← shown, but job continues
✓ Check what happened
    outcome    = failure
    conclusion = success
✓ Script in another folder
    deploy.sh
✗ Hanging step
    Error: The action 'Hanging step' has timed out after 1 minutes.
⊘ Python as the shell (skipped)
```

| Line | Meaning |
|---|---|
| `outcome = failure` | What really happened. |
| `conclusion = success` | What GitHub counts after `continue-on-error`. |
| `has timed out after 1 minutes` | The step timeout fired. The step failed, so later steps were skipped. |

### Job-level `continue-on-error` with a matrix (preview of topic 26)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    continue-on-error: ${{ matrix.experimental }}   # only experimental versions may fail
    strategy:
      matrix:
        python: ["3.12", "3.13"]
        experimental: [false]
        include:
          - python: "3.14"
            experimental: true
```

---

## Hands-on lab

1. Run the demo with **Run workflow** (`workflow_dispatch`). Create `backend/` and `scripts/` folders first.
2. Remove `timeout-minutes: 1` from the hanging step and set the **job** timeout to `2`. Now the whole job is cancelled. Compare the messages.
3. Change `continue-on-error: true` to `false` and see the job fail at the linter.
4. Add a `windows-latest` job with no `defaults`, run `echo $HOME`, then add `shell: bash`. Compare.

---

## Where you'll use it

- **Always set `timeout-minutes`** on production jobs. A hung deploy at 6 hours is a real incident.
- **Monorepos:** `defaults.run.working-directory: frontend` (the capstone's frontend and backend jobs do this).
- **Non-blocking checks:** new linters, beta OS versions, advisory security scans.
- **Cross-platform scripts:** `defaults.run.shell: bash`.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Job ran 6 hours and was cancelled | No `timeout-minutes`, and the default is 360. | Set a sensible timeout. |
| `No such file or directory` with `working-directory` | The folder doesn't exist **yet** (before checkout), or the path is wrong. Step-level paths are relative to `$GITHUB_WORKSPACE`. | Check out first. Check the path. |
| `working-directory` ignored for an action | `defaults` / `working-directory` apply only to `run:` steps. | Pass the path as an input to the action (for example `with: path:`). |
| Red step but green workflow, and the team ignores real failures | `continue-on-error` overused. | Use it only for truly optional checks. Review it regularly. |
| `$env:X` syntax error on Linux | You used PowerShell syntax with Bash. | Match the syntax to the `shell`. |

---

## Quick recap

- `timeout-minutes` (default 360) on jobs and steps. Always set it.
- `continue-on-error`: outcome = failure, conclusion = success.
- `defaults.run.shell` / `working-directory` apply only to `run:` steps. Step settings override job settings, which override workflow settings.
- `shell:` can be bash, pwsh, cmd, python or a custom template.

**Interview questions**

1. **Q: What's the default job timeout on GitHub-hosted runners?**
   A: 360 minutes (6 hours).
2. **Q: Difference between `steps.x.outcome` and `steps.x.conclusion`?**
   A: `outcome` is the raw result. `conclusion` is the result after `continue-on-error` is applied.
3. **Q: How do you make every `run:` step in a job start in `backend/`?**
   A: `defaults: run: working-directory: backend` on the job.
