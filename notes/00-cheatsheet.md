# 00 · Cheat Sheet: GitHub Actions on Two Pages

> **Use this for revision, not for learning.** Each line links to the topic that explains it. [Back to INDEX](INDEX.md)
>
> All versions and dates here are **as of November 2026** and were checked against the GitHub changelog. This is the **one place** to update when versions change. Topics use the same values.

---

## 1. Versions and platform facts (as of Nov 2026)

| Thing | Current value | Topic | Source |
|---|---|---|---|
| `actions/checkout` | **v7** (v7.0.0 June 2026). By default it now **refuses to check out fork PR code** under `pull_request_target` / `workflow_run`. | [15](15-actions-and-marketplace.md), [37](37-security-hardening.md) | [changelog](https://github.blog/changelog/2026-06-18-safer-pull_request_target-defaults-for-github-actions-checkout/) |
| `actions/cache` | **v6** (v6.0.0 June 2026, v6.1.0 latest) | [25](25-caching.md) | [releases](https://github.com/actions/cache/releases) |
| `actions/upload-artifact` / `download-artifact` | **v4+** (v3 is dead: immutable artifacts, unique names per run) | [24](24-artifacts.md) | |
| JavaScript action runtime | **`node24`**. Node 20 was **removed from runners on 23 Sep 2026**, and the opt-out env var no longer works. | [33](33-javascript-actions.md) | [changelog](https://github.blog/changelog/2026-09-23-node-20-is-no-longer-available-in-github-actions) |
| `ubuntu-latest` | **Ubuntu 26.04**. Gradual rollout from 19 Oct to 19 Nov 2026, so a job may still land on 24.04 during the rollout. The `Set up job` log shows which image ran. | [07](07-runners.md) | [changelog](https://github.blog/changelog/2026-09-17-ubuntu-26-generally-available-and-latest-migration/) |
| Concurrency `queue: max` | Up to **100 pending** runs per group, FIFO. **Not** allowed with `cancel-in-progress: true`. Shipped 7 May 2026. | [29](29-concurrency.md) | [changelog](https://github.blog/changelog/2026-05-07-github-actions-concurrency-groups-now-allow-larger-queues/) |
| Reusable workflows | Up to **10 levels** of nesting, **50** reusable workflows per run (Nov 2025) | [22](22-reusable-workflows.md) | |
| Supply-chain alert | `aquasecurity/trivy-action` tags were **hijacked (Mar 2026)** → pin actions by **full commit SHA** | [37](37-security-hardening.md), [44](44-capstone-02-trivy-filesystem-scan.md) | |

## 2. Limits you'll be asked about

| Limit | Value | Topic |
|---|---|---|
| Job run time (hosted) | **6 h** (default `timeout-minutes: 360`) | [14](14-job-control-options.md) |
| Workflow run time | **35 days** (including waiting for approval) | [41](41-billing-limits-and-cost.md) |
| Matrix | **256 jobs** per matrix | [26](26-matrix-strategy.md) |
| Cache | **10 GB per repo** by default (more can be bought), evicted after **7 days** unused | [25](25-caching.md) |
| Artifacts / logs retention | **90 days** by default (configurable) | [24](24-artifacts.md) |
| Secret size | **48 KB** | [17](17-secrets-and-variables.md) |
| Job outputs | **1 MB** per job, 50 MB per run | [21](21-step-and-job-outputs.md) |
| `schedule` | Minimum every **5 min**, UTC, default branch only, disabled after **60 days** with no activity (public repos) | [09](09-workflow-triggers.md) |
| `GITHUB_TOKEN` | Expires when the job ends (24 h max) | [18](18-github-token-and-permissions.md) |

## 3. Workflow skeleton (every key you need)

```yaml
name: CI                                   # shown in the Actions tab
run-name: CI for ${{ github.ref_name }}    # per-run title
on:
  push:            { branches: [main], paths: ['src/**'] }   # filters are AND across types
  pull_request:    { types: [opened, synchronize, reopened] }
  workflow_dispatch:
    inputs: { env: { type: choice, options: [dev, prod], default: dev } }
permissions: { contents: read }            # least privilege; unlisted scopes → none
concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }
env: { APP: demo }                         # workflow-level env
jobs:
  build:
    runs-on: ubuntu-26.04                  # pin the image in production
    timeout-minutes: 15
    outputs: { tag: ${{ steps.meta.outputs.tag }} }
    strategy: { fail-fast: false, matrix: { py: ['3.12', '3.13'] } }
    steps:
      - uses: actions/checkout@v7          # in production: pin @<full-sha> # v7.x.y
      - uses: actions/cache@v6
        with: { path: ~/.cache/pip, key: pip-${{ runner.os }}-${{ hashFiles('**/requirements.txt') }} }
      - id: meta
        run: echo "tag=${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main' && success()
    environment: production                # approvals + env secrets
    permissions: { id-token: write, contents: read }   # OIDC
    runs-on: ubuntu-26.04
    steps:
      - run: echo "deploying ${{ needs.build.outputs.tag }}"
```

## 4. The rules that trip people up

| # | Rule | Topic |
|---|---|---|
| 1 | `${{ }}` is evaluated by **GitHub before the job runs**. `$VAR` is evaluated by **the shell on the runner**. | [11](11-variables-contexts-expressions.md) |
| 2 | A step `if:` silently includes `success()`. Use `always()` or `failure()` to run after a failure. | [12](12-functions-and-status-checks.md) |
| 3 | If a job is skipped, every job that `needs` it is skipped too, unless that job's `if:` uses `always()` / `!cancelled()`. | [13](13-conditional-execution.md) |
| 4 | Jobs run **in parallel** and on **separate machines**. Pass data with outputs (small) or artifacts (files). | [06](06-workflows-jobs-steps.md), [21](21-step-and-job-outputs.md), [24](24-artifacts.md) |
| 5 | Cache = speed (it may miss). Artifact = handover (it must exist). Never use a cache to pass build output. | [24](24-artifacts.md), [25](25-caching.md) |
| 6 | Listing **any** `permissions:` scope sets every other scope to `none`. | [18](18-github-token-and-permissions.md) |
| 7 | Events created with `GITHUB_TOKEN` **don't trigger** new workflow runs (except `workflow_dispatch` / `repository_dispatch`). | [18](18-github-token-and-permissions.md) |
| 8 | Fork PRs get **no secrets** and a read-only token. `pull_request_target` has secrets, so **never run PR code there**. | [37](37-security-hardening.md) |
| 9 | Secrets **can't** be used directly in `if:`. Map them to `env` first. Reusable workflows need `secrets: inherit` or explicit secrets. | [17](17-secrets-and-variables.md), [22](22-reusable-workflows.md) |
| 10 | `schedule`, `workflow_run` and `pull_request_target` run the workflow file **from the default branch**. | [09](09-workflow-triggers.md), [31](31-chaining-workflows.md) |
| 11 | Default concurrency keeps **1 running + 1 pending**, and a newer run replaces the pending one. Use `queue: max` if every run matters. | [29](29-concurrency.md) |
| 12 | Writing to `$GITHUB_ENV` only affects **later** steps, not the current one. | [23](23-workflow-commands-and-summaries.md) |
| 13 | Matrix `include` adds or extends combinations. `exclude` is applied **before** `include`. | [26](26-matrix-strategy.md) |
| 14 | Tags like `@v7` can be moved. A full commit SHA can't. | [35](35-publishing-and-versioning-actions.md), [37](37-security-hardening.md) |

## 5. Choose the right tool

| Need | Use |
|---|---|
| Reuse **steps** inside a job | Composite action ([32](32-custom-actions-and-composite.md)) |
| Reuse **whole jobs / a pipeline** | Reusable workflow `workflow_call` ([22](22-reusable-workflows.md)) |
| Logic in code, fast start, runs on any OS | JavaScript action ([33](33-javascript-actions.md)) |
| Own toolchain, any language, Linux only | Docker action ([34](34-docker-actions.md)) |
| Run B after workflow A finishes | `workflow_run` ([31](31-chaining-workflows.md)) |
| Cloud credentials | **OIDC** (no stored keys) ([36](36-oidc-cloud-authentication.md)) |
| Human approval before deploy | Environment + required reviewers ([28](28-environments.md)) |
| Block merges without green CI | Ruleset → required status checks ([30](30-branch-protection-and-rulesets.md)) |
| Prove where an image came from | `actions/attest-build-provenance` ([40](40-supply-chain-attestations.md)) |

## 6. Debug in 60 seconds

1. Read the **first red step**, not the last one. Then open `Set up job` to see the image, the token permissions and the action SHAs.
2. Re-run with **debug logging** (`ACTIONS_STEP_DEBUG=true` or "Enable debug logging").
3. The workflow didn't start? Check the trigger, branch/path filters, the YAML of the **default branch** (for schedule and workflow_run), whether Actions is disabled, and whether the run came from `GITHUB_TOKEN`.
4. A job is stuck "Queued"? Check that the `runs-on` label exists and that a self-hosted runner is online, then check concurrency groups and environment approvals.
5. Got `Resource not accessible by integration`? Add the missing `permissions:` scope.

Full guide: [39 · Debugging](39-debugging-and-troubleshooting.md). Interview drill: [55 · Interview bank](55-interview-bank-and-jenkins-map.md).
