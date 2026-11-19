# 31 · Chaining Workflows: `workflow_run`, `merge_group` and Cross-Workflow Patterns

> **Part E: Deployment control** · Previous: [30 · Branch protection and rulesets](30-branch-protection-and-rulesets.md) · Next: [32 · Custom actions and composite actions](32-custom-actions-and-composite.md) · [Back to INDEX](INDEX.md)
>
> 🆕 New topic. The capstone's DEV deployment (`02-deploy-dev.yaml`) is triggered by `workflow_run`, and its Stage 14 explains the "default branch requirement". This topic teaches the mechanism before you meet it there.

---

## What it is

Ways to make **one workflow start another** (or react to another):

| Mechanism | Starts when… | Typical use |
|---|---|---|
| **`workflow_run`** | another workflow (by **name**) is `requested`, `in_progress` or `completed` | CI finished → deploy (capstone DEV) |
| **`workflow_call`** | a job calls it with `uses:` | shared pipeline steps (topic 22) |
| **`workflow_dispatch` via API/CLI** | a step runs `gh workflow run other.yml` | fan-out to other workflows/repos |
| **`repository_dispatch`** | someone POSTs `/dispatches` | another repo or an external system (topic 09) |
| **`pull_request: types: [closed]`** | a PR is merged/closed | deploy after merge (capstone PROD) |
| **`merge_group`** | a PR enters the **merge queue** | run required checks on the queued merge |

---

## Why it exists

- You want **CI and CD in separate files**, owned by different people, but CD must start **only after CI succeeded**. `needs` only works **inside one workflow**.
- Untrusted PR code (forks) must never get deploy secrets. A **follow-up workflow** running from the trusted default branch can do the privileged part safely.
- Busy repos use a **merge queue**: PRs merge one after another, each tested against the latest `main`.

**Jenkins comparison:** "Build after other projects are built" / the `upstream(...)` trigger ≈ `workflow_run`. The `build job: 'deploy'` step ≈ dispatching another workflow.

---

## Analogy

A **relay race between two teams in different stadiums**: team CI runs its leg. When it **crosses the finish line** (`completed`), a referee phones stadium CD, which starts its own leg (`workflow_run`). The **phone number** CD listens on must be written in the **official rulebook kept at headquarters** (the **default branch**), not on a sticky note a random runner brought along (a feature branch).

---

## How it works inside

### `workflow_run`

```text
CI workflow run finishes (any branch)
  → GitHub emits a workflow_run event, payload = the CI run (conclusion, head_sha, head_branch, …)
  → GitHub looks for workflows with on.workflow_run.workflows: ["<CI workflow NAME>"]
       ★ it reads those listener workflows FROM THE DEFAULT BRANCH only
  → listener run starts on the DEFAULT BRANCH's latest commit (GITHUB_SHA = default branch!)
       with full repository secrets and write-capable GITHUB_TOKEN
  → you must check out the CI commit yourself: ref: ${{ github.event.workflow_run.head_sha }}
```

Rules that follow:

1. **The listener workflow must exist on the default branch**, or it never runs. A feature branch can't add a new privileged listener, which is the security reason the capstone explains.
2. It fires on **completed**, whether the upstream run succeeded or failed, so **always check `github.event.workflow_run.conclusion == 'success'`**.
3. `workflows:` matches the upstream workflow's **`name:`**, not its file name.
4. Chains can be at most **3 levels deep** (A → B → C).
5. The run starts from the **default branch**, so `actions/checkout` without `ref:` checks out `main`, **not** the code CI tested.

### Why the PR the capstone creates doesn't re-trigger CI

The capstone opens its promotion PR using `GITHUB_TOKEN` (`github-actions[bot]`). Events made with `GITHUB_TOKEN` **don't start new workflows** (topic 18). That's fine, because the PR's head commit **already has the CI checks** from the earlier `push` run (checks belong to **commits**, not PRs). Merging the PR is done by a **human**, so `pull_request: closed` **does** fire and starts the PROD workflow.

---

## Workflow YAML / commands

### `workflow_run`: the capstone's DEV pattern (simplified)

```yaml
name: CWVJ Deploy | DEV

on:
  workflow_run:
    workflows:
      - "CWVJ CI | Build, Scan and Publish"   # upstream workflow's name:
    types:
      - completed
    branches:                                # filter by the UPSTREAM run's branch
      - "feature/**"
      - "bugfix/**"
      - "hotfix/**"

concurrency:
  group: cwvj-deploy-dev
  cancel-in-progress: false

jobs:
  deploy-to-dev-job:
    runs-on: ubuntu-26.04
    environment: dev
    if: >-
      github.event.workflow_run.conclusion == 'success'
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Check out the exact commit CI tested
        uses: actions/checkout@v7
        with:
          ref: ${{ github.event.workflow_run.head_sha }}
      - run: |
          echo "CI run:     ${{ github.event.workflow_run.html_url }}"
          echo "CI branch:  ${{ github.event.workflow_run.head_branch }}"
          echo "CI commit:  ${{ github.event.workflow_run.head_sha }}"
          echo "This run's GITHUB_SHA (default branch): $GITHUB_SHA"
```

| Line | Meaning |
|---|---|
| `workflows: ["CWVJ CI \| …"]` | The listener matches the upstream **display name**. |
| `types: [completed]` | Fire when the upstream run ends (success **or** failure). |
| `branches:` | Only upstream runs from these branches. |
| `if: …conclusion == 'success'` | Deploy only if CI passed. |
| `ref: …head_sha` | Deploy the **tested** commit, not `main`. |
| `if: >-` | Folded YAML string, so a long condition fits on several lines. |

**Sample log output:**

```text
CI run:     https://github.com/you/mega-project-demo/actions/runs/1234567890
CI branch:  feature/initial-ci
CI commit:  9f2c8a7e…
This run's GITHUB_SHA (default branch): 1a2b3c4d…     ← different! proves the run started from main
```

### Downloading the upstream run's artifacts

```yaml
- uses: actions/download-artifact@v8
  with:
    name: backend-container-image
    run-id: ${{ github.event.workflow_run.id }}
    github-token: ${{ secrets.GITHUB_TOKEN }}     # needs permissions: actions: read
```

### Deploy-after-merge: the capstone's PROD trigger

```yaml
on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  deploy-to-prod-job:
    environment: prod
    if: >-
      github.event.pull_request.merged == true &&
      github.event.pull_request.user.login == 'github-actions[bot]' &&
      contains(github.event.pull_request.body, github.event.pull_request.head.sha)
```

- `merged == true`: closed **and merged**, not just closed.
- `user.login == 'github-actions[bot]'`: only PRs created by the automation (DEV-validated).
- `contains(body, head.sha)`: the PR body records the exact SHA that DEV tested, so PROD deploys that verified commit (detailed in capstone topics 52–53).

### Dispatching another workflow from a step

```yaml
- name: Trigger the docs site build
  env:
    GH_TOKEN: ${{ secrets.DOCS_REPO_PAT }}      # GITHUB_TOKEN can't trigger runs (anti-loop) except dispatch in the same repo
  run: gh workflow run build.yml --repo my-org/docs-site -f version=${{ github.ref_name }}
```

> `GITHUB_TOKEN` **can** trigger `workflow_dispatch`/`repository_dispatch` **in the same repo** (the exception to the anti-loop rule, topic 18). For **another repo**, use a PAT or a GitHub App token.

### Merge queue (`merge_group`)

```yaml
on:
  pull_request:
  merge_group:            # runs when a PR is added to the merge queue
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: make test
```

Enable **Require merge queue** in the ruleset. Required checks must run on **both** `pull_request` and `merge_group`, or queued PRs wait forever.

---

## Hands-on lab

1. Create `ci.yml` (`name: CI`, on push, `run: echo build`) and `deploy.yml` (`on: workflow_run: workflows: [CI], types: [completed]`). Push both to a **feature branch** only and observe that deploy **doesn't run**.
2. Merge `deploy.yml` into `main`, push to the feature branch again, and now deploy runs. Compare `GITHUB_SHA` with `workflow_run.head_sha`.
3. Make CI fail and confirm deploy runs but its job is **skipped** by the `conclusion` check.
4. Upload an artifact in CI and download it in deploy with `run-id`.
5. Rename CI's `name:` and see deploy stop triggering (it matches by name).

---

## Where you'll use it

- **CI → CD separation** (capstone `01-ci.yaml` → `02-deploy-dev.yaml`).
- **Secure handling of fork PRs:** an untrusted `pull_request` run uploads results, and a trusted `workflow_run` comments on the PR with write permissions.
- **Release fan-out:** tag → build → dispatch docs, SDK and Helm-chart repos.
- **Merge queues** in busy monorepos.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `workflow_run` workflow **never triggers** | It isn't on the **default branch**, or `workflows:` doesn't match the upstream `name:` exactly. | Merge it to `main`. Copy the exact name. |
| Deploys even when CI failed | Missing `conclusion == 'success'` check. | Add the `if:`. |
| Deployed the wrong code | Checked out `main` instead of the CI commit. | `ref: ${{ github.event.workflow_run.head_sha }}`. |
| Old CI runs aren't "replayed" after adding the listener | Events that already happened aren't re-sent. | Push a fresh commit (capstone Stage 14). |
| PR created by a workflow has no CI running on it | Anti-loop rule for `GITHUB_TOKEN`. | Rely on checks already on the head commit, or create the PR with a GitHub App token. |
| `Resource not accessible by integration` downloading upstream artifacts | Missing `actions: read` or `github-token`. | Add both. |
| Merge queue stuck | The required workflow lacks `merge_group:`. | Add the trigger. |

---

## Quick recap

- `workflow_run` = run after another workflow (matched by **name**), from the **default branch**, with secrets. Check `conclusion`, check out `head_sha`.
- `pull_request: closed` + `merged == true` = deploy after merge.
- Events made with `GITHUB_TOKEN` don't trigger runs (except same-repo dispatch).
- `merge_group` is needed for merge queues.
- Use `needs` inside one workflow, and these mechanisms across workflows.

**Interview questions**

1. **Q: Why didn't my `workflow_run` deploy workflow trigger?**
   A: It must exist on the default branch, and `workflows:` must match the upstream workflow's `name`.
2. **Q: Why is `workflow_run` considered safer for privileged actions on fork PRs?**
   A: It runs from the trusted default-branch definition with the base repo's secrets, while the untrusted PR code ran earlier in a restricted `pull_request` run. (You still must not execute the PR's code with those secrets.)
3. **Q: In a `workflow_run` job, what does `actions/checkout` check out by default?**
   A: The default branch, not the upstream commit. Pass `ref: github.event.workflow_run.head_sha`.
