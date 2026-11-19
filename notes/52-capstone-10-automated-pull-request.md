# 52 · Capstone 10 · Automated Pull Request Creation (Stage 12)

> **Part H: Capstone** · Previous: [51 · DEV Environment and Automated DEV Deployment (Stage 11)](51-capstone-09-dev-deployment.md) · Next: [53 · PROD Environment and Automated PROD Deployment (Stage 13)](53-capstone-11-prod-deployment.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

**Stage 12** creates **`03-create-pr.yaml`**, a **reusable workflow** (`workflow_call` with inputs `head_branch`, `head_sha`). It checks out the branch, **verifies HEAD equals the DEV-deployed SHA**, and runs `gh pr create` (or `gh pr edit`) with `GITHUB_TOKEN` to open "Promote <branch> to main", **recording the SHA in the PR body**. `02-deploy-dev.yaml` calls it after a successful DEV deploy. The repo setting **"Allow GitHub Actions to create and approve pull requests"** is enabled.

## Why it exists

Automation proposes, humans decide: after DEV passes, a promotion PR appears automatically, but **review, approval and merge stay manual** (rulesets). The SHA check ensures the PR represents **exactly the commit that passed DEV**. If the branch moved on, no PR is created for untested code.

## Analogy

When a car passes the test track, the system **automatically fills in the release request form** with the car's **VIN** and puts it on the manager's desk. It **doesn't sign it**. A manager must review and sign (PR approval and merge).

## How it works inside

```text
02-deploy-dev.yaml  job create-pull-request-job:
   needs: deploy-to-dev-job
   uses: ./.github/workflows/03-create-pr.yaml       (local reusable workflow: SAME commit as caller)
   with: head_branch, head_sha (from workflow_run)
   permissions: contents: read, pull-requests: write  (caller job caps the callee)
03-create-pr.yaml:
   checkout ref: head_branch → test "$(git rev-parse HEAD)" = "$DEPLOYED_SHA"   (mismatch → fail)
   gh pr create --head … --base main --title … --body "DEV deployment succeeded for commit <SHA>…"
     || gh pr edit <branch> --body "…<SHA>…"        (PR already exists → update the body)
   author = github-actions[bot]  → events from GITHUB_TOKEN don't start new runs (anti-loop, topic 18)
```

Topics: 22 (reusable workflows, inputs, permissions), 18 (`GITHUB_TOKEN`, the "create PRs" setting), 30 (the ruleset governs the merge).

## Workflow YAML / commands: build it step by step

### Stage 12: Open the promotion PR automatically after a good DEV deploy

DEV deploy plus smoke test succeed → open a PR to `main` that records **the exact commit deployed**. **People** review, approve and merge. Stage 13 then deploys the same images to PROD. (Still build first, push later: nothing is pushed until Stage 14.)

#### 1. The reusable workflow `.github/workflows/03-create-pr.yaml`

```yaml
name: CWVJ PR | Create Pull Request

on:
  workflow_call:
    inputs:
      head_branch:
        required: true
        type: string
      head_sha:
        required: true
        type: string

jobs:
  create-pull-request-job:
    name: Create Pull Request
    runs-on: ubuntu-26.04

    permissions:
      contents: read
      pull-requests: write

    steps:
      - name: Check out the deployed branch
        uses: actions/checkout@v7
        with:
          ref: ${{ inputs.head_branch }}

      - name: Verify deployed commit
        env:
          DEPLOYED_SHA: ${{ inputs.head_sha }}
        run: test "$(git rev-parse HEAD)" = "$DEPLOYED_SHA"

      - name: Create or update Pull Request
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          HEAD_BRANCH: ${{ inputs.head_branch }}
          HEAD_SHA: ${{ inputs.head_sha }}
        run: |
          gh pr create --head "$HEAD_BRANCH" --base main --title "Promote $HEAD_BRANCH to main" --body "DEV deployment succeeded for commit $HEAD_SHA. Please review the changes before merging." \
            || gh pr edit "$HEAD_BRANCH" --body "DEV deployment succeeded for commit $HEAD_SHA. Please review the changes before merging."
```

| Piece | Why |
|---|---|
| `workflow_call` with **required string inputs** | It never runs on its own; a caller passes the branch and SHA (topic 22). |
| `contents: read` + `pull-requests: write` | Checkout, plus creating and editing PRs. That's all. It uses **`GITHUB_TOKEN`** (repo-scoped, per run), **not** OIDC (which is for AWS) and not a long-lived PAT (topic 18). |
| Checkout the branch, then `test "$(git rev-parse HEAD)" = "$DEPLOYED_SHA"` | If the branch moved on after DEV (a newer, **undeployed** commit), fail instead of proposing it. |
| `GH_TOKEN` | The `gh` CLI picks the token up from this env var. |
| Inputs passed via `env:` and quoted | Branch names are **untrusted input**, so never inline `${{ }}` into a script (topic 37). |
| The body records `$HEAD_SHA` | Reviewers can see which commit passed DEV. **Stage 13 also reads this SHA** to find the images. |
| `gh pr create … \|\| gh pr edit …` | A PR follows its branch, so later pushes update the **existing** PR, and `edit` refreshes the recorded SHA. |

> **Know the limits.** `||` runs `edit` on **any** `create` failure (missing permission, no diff, an API error), not only "PR already exists". A stricter version checks `gh pr list --head "$HEAD_BRANCH"` first. Also, a newly pushed commit shows up in the open PR **before** its DEV deploy finishes. Reviewers should confirm the body's SHA matches the PR head, and the required checks (Stage 14) add a safety net.

#### 2. Call it from `02-deploy-dev.yaml` (a second job)

```yaml
  # Calling job
  create-pull-request-job:
    name: Create Pull Request
    needs: deploy-to-dev-job

    permissions:
      contents: read
      pull-requests: write

    uses: ./.github/workflows/03-create-pr.yaml

    with:
      head_branch: ${{ github.event.workflow_run.head_branch }}
      head_sha: ${{ github.event.workflow_run.head_sha }}
```

- `needs: deploy-to-dev-job`: the deploy job contains the rollout checks **and** the smoke test, so the PR only appears after both pass. If they fail, this job is skipped.
- **Job-level `uses:`** (no `runs-on`, no `steps`). The called workflow has the runner and the logic. The caller decides **when** it runs and **with which inputs**.
- **The caller must grant the permissions too.** A called workflow can only keep or **reduce** what its caller grants, never raise it.

#### 3. Allow Actions to create PRs (a repository setting)

**Settings → Actions → General → Workflow permissions →** ☑ **Allow GitHub Actions to create and approve pull requests → Save.**

The YAML permissions aren't enough on their own: this repo-level switch must be on, or `gh pr create` fails with *"GitHub Actions is not permitted to create or approve pull requests."* The workflow only **creates or edits**. Approving and merging stay human, enforced by the `protect-main` ruleset.

---

## Hands-on lab

1. Create `03-create-pr.yaml`, add the calling job and enable the setting. After a DEV deploy, a PR appears authored by **github-actions[bot]** with the SHA in its body.
2. Push another commit to the branch: after DEV deploys it, the PR body is **updated** with the new SHA (the `gh pr edit` path).
3. Untick "Allow GitHub Actions to create and approve pull requests" and see the error, then restore it.

## Where you'll use it

- Promotion PRs (environment-to-environment), release PRs, dependency-bump PRs created by automation.

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `GitHub Actions is not permitted to create or approve pull requests` | Repo/org setting is off. | Settings → Actions → General → enable it. |
| `HTTP 403: Resource not accessible by integration` | Missing `pull-requests: write` (callee **and** caller job). | Add it to both. |
| "Verify deployed commit" fails | The branch advanced after the CI run. | Expected safety. The newer commit's own pipeline will create or update the PR. |
| No CI checks run on the bot's PR | Anti-loop rule. | The checks already exist on the head commit from the push CI. Fine for required checks. |

## Quick recap

- A reusable PR workflow called by the DEV deploy: verify SHA → `gh pr create || gh pr edit`, recording the SHA.
- Needs `pull-requests: write` plus the repo setting allowing Actions to create PRs.
- Humans still review, approve and merge (ruleset).

**Interview questions**

1. **Q: Why verify `git rev-parse HEAD` against the deployed SHA before creating the PR?**
   A: So the PR represents exactly the commit that passed DEV. If the branch moved, you don't promote untested code.
2. **Q: Why don't workflows trigger on the PR the bot creates?**
   A: Events caused by `GITHUB_TOKEN` don't start new workflow runs, to prevent loops.
3. **Q: Why make PR creation a reusable workflow?**
   A: It separates "when to run" (the caller) from "how to create the PR" (the callee), so it's reusable by other deploy workflows.
