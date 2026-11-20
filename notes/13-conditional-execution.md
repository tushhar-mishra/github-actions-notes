# 13 · Conditional Execution with `if:`

> **Part B: Triggers and logic** · Previous: [12 · Functions and status checks](12-functions-and-status-checks.md) · Next: [14 · Job control options](14-job-control-options.md) · [Back to INDEX](INDEX.md)
>
> The production-style PR pipeline (Docker Hub, secrets, PR title checks, draft PRs) needs secrets (topic 17) and Docker (topic 19), so it lives in [topic 19](19-lab-docker-build-push.md). This topic teaches the `if:` mechanics on their own, with no secrets.

---

## What it is

`if:` lets a **job** or a **step** decide **whether to run**, using an expression (topic 11) and functions (topic 12).

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'   # job-level condition
    runs-on: ubuntu-latest
    steps:
      - if: github.event_name == 'push'   # step-level condition
        run: echo "deploying"
```

If the condition is **false**, the job or step is **skipped**. A skipped job or step is **not a failure**.

---

## Why it exists

One workflow often serves several situations:

- PR → run tests only
- push to `main` → test **and** deploy
- draft PR → run lightweight checks only
- failure → send a notification

Without `if:` you'd need many nearly-identical workflow files. Events and filters (topic 10) decide whether the **workflow starts**. `if:` decides **which parts run** once it has started.

**Jenkins comparison:** `when { branch 'main' }` on a stage ≈ job-level `if:`. Groovy `if` blocks inside `steps` ≈ step-level `if:`. `post { failure { } }` ≈ `if: failure()`.

---

## Analogy

An **airport**:

- **Event filters** = "is there a flight today at all?" (does the workflow start?)
- **Job `if:`** = the **gate check**: "is this passenger flying to London?" If not, they never board. No seat (runner) is used.
- **Step `if:`** = the **flight attendant on board**: "vegetarian meal? → serve it. Otherwise skip."
- **`if: failure()`** = the **emergency procedure** that only runs when something went wrong.

---

## How it works inside

```text
Workflow run starts
 │
 ├─ For each job (when its needs: are done):
 │     evaluate job if:   ← GitHub engine, NO runner yet, only contexts available
 │        false → job "skipped" (grey), no runner, no minutes
 │        true  → job queued → runner
 │
 └─ On the runner, for each step:
       evaluate step if:   ← contexts + steps.<id>.* + status functions
          (no if written → if: success())
          false → step skipped
          true  → step runs
```

What each level can see:

| Where | Can use | Cannot use |
|---|---|---|
| **Job `if:`** | `github`, `needs`, `vars`, `inputs`, status functions | `env` (job-level), `steps`, `runner`, `$SHELL_VARS` |
| **Step `if:`** | all of the above, plus `env`, `steps.<id>.outputs/outcome/conclusion`, `runner`, `job`, `matrix` | shell variables (`$X`) |

🆕 **Results you can test:**

- `needs.<job>.result` → `success`, `failure`, `cancelled` or `skipped`
- `steps.<id>.outcome` → the result **before** `continue-on-error` is applied
- `steps.<id>.conclusion` → the result **after** `continue-on-error` (topic 14)

> 🆕 **Skips spread down the chain.** If job B `needs` job A and A is **skipped**, B is skipped too (implicit `success()`). To run B anyway, use `if: ${{ !cancelled() && needs.A.result != 'failure' }}` or `if: always()`.

---

## Workflow YAML / commands

### A pipeline that changes behaviour by event, branch and result

`.github/workflows/conditional-demo.yml`:

```yaml
name: Conditional Demo

on:
  push:
    branches: [main, "feature/**"]
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        id: tests                                   # id lets later steps read this step's result
        env:
          COMMIT_MSG: ${{ github.event.head_commit.message }}   # attacker-controllable text: pass via env, never inline
        run: |
          echo "running tests"
          # simulate failure when commit message contains [fail]
          if [[ "$COMMIT_MSG" == *"[fail]"* ]]; then exit 1; fi

      - name: Notify on failure
        if: failure()
        run: echo "::warning::Tests failed on ${{ github.ref_name }}"

  lint-full:
    # skip heavy lint for draft PRs; always run for pushes
    if: github.event_name == 'push' || github.event.pull_request.draft == false
    runs-on: ubuntu-latest
    steps:
      - run: echo "full lint"

  deploy:
    needs: [test, lint-full]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ github.sha }}"

  report:
    needs: [test, lint-full, deploy]
    if: ${{ !cancelled() }}                          # runs even if earlier jobs failed or were skipped
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "test:   ${{ needs.test.result }}"
          echo "lint:   ${{ needs.lint-full.result }}"
          echo "deploy: ${{ needs.deploy.result }}"
```

| Line | Meaning |
|---|---|
| `id: tests` | Names the step so others can use `steps.tests.*`. |
| `if: failure()` | Runs only because the previous step failed. |
| `::warning::…` | A **workflow command**: shows a yellow annotation in the UI (topic 23). |
| `github.event.pull_request.draft == false` | Draft PRs skip the heavy job. On a `push`, `github.event.pull_request` doesn't exist, which is why the `||` with `push` comes first. |
| `needs.lint-full.result` | Reads the result of the job with ID `lint-full`. Hyphens are fine in IDs, because expressions have no minus operator. `needs['lint-full'].result` (bracket syntax) also works. |
| `if: ${{ !cancelled() }}` | Report every time unless the run was cancelled. The braces are needed because the expression starts with `!`. |

**Sample log output: push to `feature/x`:**

```text
✓ test
✓ lint-full
⊘ deploy        (skipped: condition false — not main)
✓ report
    test:   success
    lint:   success
    deploy: skipped
```

**Sample log output: push to `main` with message "fix [fail]":**

```text
✗ test
    Run tests → Error: Process completed with exit code 1.
    ⚠ Tests failed on main
✓ lint-full
⊘ deploy        (skipped: a needed job failed)
✓ report
    test:   failure
    lint:   success
    deploy: skipped
Workflow: ✗ failure
```

**Sample: draft PR:**

```text
✓ test
⊘ lint-full     (skipped)
⊘ deploy        (skipped)
✓ report  → lint: skipped, deploy: skipped
```

---

## Hands-on lab

1. Push the workflow to a feature branch and confirm `deploy` is skipped.
2. Merge to `main` and confirm `deploy` runs.
3. Push to `main` with `[fail]` in the commit message and confirm `deploy` is skipped and `report` shows `failure`.
4. Open a **draft** PR, then click **Ready for review**, and compare which jobs run.
5. Change `report` to `if: success()` and repeat step 3. It no longer runs. Explain why.

---

## Where you'll use it

- **Deploy only from `main`** or tags, test everywhere.
- **Skip expensive jobs for draft PRs** (the topic 19 pipeline does this).
- **Failure notifications** and **always-run reports**.
- **Environment selection:** `if: github.ref == 'refs/heads/main'` → prod job, `if: github.ref == 'refs/heads/develop'` → dev job (topic 28).
- The capstone runs push-to-ECR jobs only for `push` to `main` and builds and scans only for PRs.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Job always skipped | Condition false, or a `needs` job was skipped (skips spread down the chain). | Dump `${{ toJSON(github) }}` / `${{ toJSON(needs) }}` in a debug job. Use `!cancelled()` where needed. |
| `Unrecognized named-value: 'env'` in a job `if:` | Job-level `env` isn't available there. | Use `vars.*` or move the condition to a step. |
| `if: ${{ github.ref_name }} == 'main'` runs on every branch | Only part of the condition is inside the braces. GitHub substitutes the value and gets a non-empty **string** like `feature/x == 'main'`, which is "truthy". | Put the **whole** comparison inside one `${{ }}`, or drop the braces: `if: github.ref_name == 'main'`. |
| `needs.lint-full.result` empty | The job isn't listed in this job's `needs:` (only direct dependencies are visible), or there's a typo in the ID. | Add it to `needs:`. Check the ID spelling. |
| Report job doesn't run after a failure | Implicit `success()`. | Add `!cancelled()` or `always()`. |
| Skipped required check blocks a PR merge | A skipped job counts as **success** for required checks… unless the whole **workflow** was filtered out. | Use job `if:` (shows "skipped") rather than `paths:` filters on required workflows (topic 10). |

---

## Quick recap

- **Job `if:`** is evaluated by GitHub before a runner is assigned. **Step `if:`** is evaluated on the runner, between steps.
- No `if:` = `if: success()`. Adding a status function replaces it.
- Use `needs.<job>.result` and `steps.<id>.outcome/conclusion` to react to results.
- Skips spread down the `needs` chain. Use `!cancelled()` to continue anyway.
- `needs.<job>.result` is only visible for jobs listed **directly** in `needs:`.

**Interview questions**

1. **Q: Does a skipped job use runner minutes?**
   A: No. Job-level `if:` is evaluated before the job is queued to a runner.
2. **Q: How do you run a summary job after other jobs no matter whether they passed?**
   A: `needs: [a, b]` plus `if: ${{ !cancelled() }}` (or `always()`), then read `needs.a.result`.
3. **Q: Why might `deploy` be skipped even though its condition is true?**
   A: A job it needs failed or was skipped. The implicit `success()` is also part of the condition.
