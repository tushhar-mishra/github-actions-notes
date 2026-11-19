# 43 · Capstone 01 · Project, Repository, Governance and First Run (Stages 1–4)

> **Part H: Capstone** · Previous: [42 · Overview, Prerequisites and the Application](42-capstone-00-overview.md) · Next: [44 · Trivy Filesystem Security Scanning (Stage 5)](44-capstone-02-trivy-filesystem-scan.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

**Phase 1** sets up everything before real CI: the project layout, the first `ci.yaml`, a private GitHub repo with SSH access, the **`protect-main` ruleset**, the **`dev`/`prod` environments** with branch rules, and a first manual run.

## Why it exists

Governance first: if `main` isn't protected and environments don't exist **before** pipelines are added, nothing stops an unreviewed change from reaching production later. Building the workflow **incrementally** (a checkout-only job first) gives a known-good base to debug from.

## Analogy

Before building a house you **fence the site, set up the gate with a guard, and mark which trucks may enter which area** (ruleset + environments), then test that a delivery van can actually drive in (the first workflow run).

## How it works inside

Concepts used here (with where they're taught):

| Piece | What happens | Topic |
|---|---|---|
| `on.push.branches: feature/**, bugfix/**, hotfix/**` + `paths:` | CI runs only for development branches **and** relevant paths | 10 |
| `workflow_dispatch` | Manual run, used because a new branch with no changes is filtered out by `paths` | 09 |
| `concurrency: cwvj-ci-${{ github.ref }}` + `cancel-in-progress: true` | One CI run per branch, newest wins | 29 |
| `runs-on: ubuntu-26.04` | Pinned runner image (GA Sept 2026) | 07 |
| SSH key + `git push -u origin main` | Initial upload | 02 |
| Ruleset `protect-main` | PR required, 1 approval, dismiss stale approvals, last-push approval, conversation resolution, squash only, no force push/deletion. Required checks added **later** | 30 |
| Environments `dev` (feature/bugfix/hotfix branches), `prod` (`main`) | Deployment branch restrictions | 28 |

> **Plan note:** on GitHub Free, rulesets aren't **enforced** on private repos. The project makes the repo public later (Stage 14, after a security review) to demonstrate enforcement.

## Workflow YAML / commands: build it step by step

### Stage 1: Project layout and the first workflow

Only the **contents** of `project-files/` ([`code/capstone/`](code/capstone/)) go to the root of your new GitHub repo. Add files **only when a stage tells you to**; never copy the finished project in at the start.

```text
project-files/
├── .github/workflows/ci.yaml      # grows stage by stage (later renamed 01-ci.yaml)
├── argocd/                        # GitOps (a later video)
├── cwvj-3tier-mega-project/
│   ├── backend/                   # Java 21 / Spring Boot / Maven
│   ├── frontend/                  # React / Vite / npm
│   └── README.md
├── infrastructure/                # EKS config, networking scripts
├── kubernetes/                    # manifests and RBAC for dev/prod
└── README.md
```

> **Security:** secret templates with placeholder values stay **outside** `project-files/`. Never commit real credentials.

Create `.github/workflows/ci.yaml`:

```yaml
name: CWVJ CI | Build, Scan and Publish

on:
  push:
    branches:
      - 'feature/**'
      - 'bugfix/**'
      - 'hotfix/**'
    paths:
      - 'cwvj-3tier-mega-project/frontend/**'
      - 'cwvj-3tier-mega-project/backend/**'
      - '.github/workflows/**'
      - 'kubernetes/**'

  workflow_dispatch:

concurrency:
  group: cwvj-ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  workflow-foundation-job:
    name: Workflow Foundation
    runs-on: ubuntu-26.04
    steps:
      - name: Checkout source code
        uses: actions/checkout@v7
```

What's specific to this project (the general syntax is in topics 06–10 and 29):

- There's **no `push` trigger for `main`**. CI runs on development branches; `main` only changes through PRs.
- `paths:` covers the app, the workflows and the manifests, so a README-only change doesn't burn minutes. Use either `paths` or `paths-ignore` for one event, never both (topic 10).
- `group: cwvj-ci-${{ github.ref }}` puts one group per branch, and `cancel-in-progress: true` makes the newest push win. `github.ref` is the full ref (`refs/heads/feature/initial-ci`). Don't confuse it with `github.workspace`, the runner folder where the code is checked out, which Trivy uses later.

### Stage 2: Create the repo and push over SSH

1. Create a **private** repo `mega-project-demo` **without** a README, `.gitignore` or license.
2. Create a dedicated key and load it into the SSH agent (the agent signs on the client's behalf, so the private key never leaves it; see topic [02](02-git-and-github-recap.md)):

```bash
ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/cwvj_mega_project_ed25519   # -C is just a label
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/cwvj_mega_project_ed25519
ssh-add -l                                       # the key is listed
cat ~/.ssh/cwvj_mega_project_ed25519.pub         # copy this
```

3. Add the **public** key in **GitHub → Settings → SSH and GPG keys → New SSH key** (title `cwvj-mega-project`). Never share or commit the private key, and don't forward your agent to untrusted hosts.
4. From inside `project-files/`:

```bash
git init
git add .
git commit -m "ci: add initial GitHub Actions workflow"
git branch -M main
git remote add origin git@github.com:<you>/mega-project-demo.git
git remote -v
git push -u origin main          # -u sets the upstream so later pushes are just `git push`
```

This push **doesn't run CI** (there's no `main` push trigger). It's the one bootstrap push that creates `main`. The rules that block direct pushes come next, **before** any feature work.

### Stage 3: Governance: the ruleset and environments

**Ruleset** (**Settings → Rules → Rulesets → New branch ruleset**). Name it `protect-main`, set Enforcement to **Active**, and target the pattern `main`. Turn on:

| Rule | Setting |
|---|---|
| Restrict deletions | on |
| Block force pushes | on |
| Require a pull request before merging | **1** approval · dismiss stale approvals on new commits · require approval of the most recent reviewable push · require conversation resolution · allowed merge method **Squash** |
| Require status checks / up to date with base | **not yet**: added in Stage 14, after CI has reported once |
| Bypass list | **empty** |

> **Plan note:** GitHub Free enforces rulesets only on **public** repos. The project goes public in Stage 14 (after a security review) to prove enforcement. Docs: [available rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets).

**Environments** (**Settings → Environments → New environment**), each with **Deployment branches and tags → Selected branches and tags**:

| Environment | Allowed branches | Later holds |
|---|---|---|
| `dev` | `feature/*`, `bugfix/*`, `hotfix/*` (three separate rules) | DEV variables, AWS role, namespace |
| `prod` | `main` (changed to `refs/pull/*/merge` in Stage 14, topic 54) | PROD config plus required reviewers |

`dev` shows "0 branches" until you push a matching branch; `prod` already shows `main`. Branch rules only **allow** deployments. A job deploys only when it says `environment: dev` or `environment: prod` (topic 28).

### Stage 4: First run

```bash
git checkout -b feature/initial-ci
git push -u origin feature/initial-ci     # tip: <local>:<remote> to push under another name
```

**No run appears.** The branch filter matches, but the `paths` filter finds no changed files (the branch is identical to `main`). That's why `workflow_dispatch` is there: **Actions → CWVJ CI | Build, Scan and Publish → Run workflow → `feature/initial-ci`**. (A dispatch workflow must exist on the default branch, which it does since Stage 2.)

```text
push feature/initial-ci → branch filter ✓ → paths filter ✗ (no changes) → skipped
Run workflow (dispatch) → hosted runner (ubuntu-26.04) → Checkout source code → ✅ Workflow Foundation
```

Check that the run is green, the job **Workflow Foundation** ran on Ubuntu 26.04, and **Checkout source code** succeeded. That's your known-good base. Every later stage adds one tested piece to it.

---

## Hands-on lab

Follow Stages 1–4 above. Checkpoints:

1. `git remote -v` shows your SSH URL, and `main` is on GitHub with `.github/workflows/ci.yaml`.
2. **Settings → Rules → Rulesets** shows `protect-main` **Active** (on a public repo, a direct `git push origin main` is rejected with `GH013`).
3. **Settings → Environments** shows `dev` with three branch patterns and `prod` with `main`.
4. Pushing `feature/initial-ci` creates **no** run (paths filter). **Run workflow** on that branch succeeds with "Workflow Foundation".

## Where you'll use it

- Every new production repo: protect `main`, create environments, add a minimal workflow, then grow it.
- Trunk-based development with short-lived `feature/*`, `bugfix/*`, `hotfix/*` branches.

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Push to `feature/initial-ci` doesn't trigger CI | Expected: `paths` didn't match (no file changes). | **Run workflow** manually (that's why `workflow_dispatch` exists). |
| No **Run workflow** button | The workflow isn't on `main` yet. | Push it to `main` first (Stage 2 did). |
| `Permission denied (publickey)` | SSH key not loaded or not added. | Topic 02. |
| Ruleset not blocking pushes | Private repo on Free. | Expected until the repo is public (Stage 14). |
| `dev` shows "0 branches" | No matching branch pushed yet. | Normal. It updates after pushing `feature/*`. |

## Quick recap

- Minimal CI first: development-branch push + paths filter + `workflow_dispatch` + per-branch concurrency + pinned `ubuntu-26.04`.
- Governance before pipelines: the `protect-main` ruleset and `dev`/`prod` environments with branch rules.
- A new branch with no changes is skipped by `paths` filters, so use `workflow_dispatch`.

**Interview questions**

1. **Q: Why does pushing a brand-new feature branch not trigger this CI?**
   A: The `paths` filter only matches changed files, and a new branch from `main` has no changes under those paths.
2. **Q: Why enable required status checks only after the first CI run?**
   A: GitHub can only offer check names that have already reported. You must select the real job check names.
3. **Q: What does "require approval of the most recent reviewable push" prevent?**
   A: An author pushing new code after approval and merging it without someone else reviewing that latest push.
