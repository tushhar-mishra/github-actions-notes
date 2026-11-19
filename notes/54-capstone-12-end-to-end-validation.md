# 54 · Capstone 12 · Push, Configure and Validate the Complete Automation (Stage 14) + Next Steps

> **Part H: Capstone** · Previous: [53 · PROD Environment and Automated PROD Deployment (Stage 13)](53-capstone-11-prod-deployment.md) · Next: [55 · Interview bank and Jenkins map](55-interview-bank-and-jenkins-map.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

**Stage 14** goes live: security-review the repo history and make it **public** (to enforce rulesets on GitHub Free), verify `protect-main`, add an **independent reviewer** collaborator, enable **Actions may create PRs**, configure the **PROD environment** (required reviewer, prevent self-review, no admin bypass, `refs/pull/*/merge`), observe the **default-branch requirement**, add **required CI status checks**, **bootstrap** the workflows into `main` (a temporary ruleset disable), run the full release (CI → DEV → PR → review/merge → PROD approval → PROD), clean up the branch, and repeat as a **normal developer flow**. Then: **ideas to evolve the project**, the conclusion and references.

## Why it exists

This is where all the pieces have to work **together**, including the GitHub settings that YAML can't express. It also teaches two real-world gotchas: **`workflow_run` listeners must be on the default branch**, and **required checks can only be chosen after they've reported once**.

## Analogy

**Opening day** of the factory: final safety inspection (security review), handing out manager keys (reviewer, environment approvers), a one-time ribbon-cutting where the gates are briefly opened to bring the machines in (bootstrap), then the **first real car goes through every station** to a customer, and finally a normal working day to prove it runs without special help.

## How it works inside

```text
Settings (no YAML):  repo public → ruleset verify → collaborator (Write) → Actions: create PRs ✓
                     prod env: reviewer, prevent self-review, no admin bypass, refs/pull/*/merge
push feature/initial-ci → only CI runs (02 isn't on main yet)          ← default-branch requirement
ruleset: add required checks (real names from that CI run)
bootstrap: ruleset Disabled → git push origin HEAD:main (fast-forward) → ruleset Active
fresh commit on feature/initial-ci → CI ✓ → workflow_run → DEV ✓ → PR (bot)
reviewer account: review + approve + squash merge   (required checks + approval enforced)
→ pull_request closed/merged → PROD job waits → owner account approves → PROD ✓
cleanup: delete remote/local branch; then a new feature branch → the whole flow again, untouched
```

## Workflow YAML / commands: build it step by step

### Stage 14: Go live: governance, bootstrap, first release, normal flow

#### 1. Governance before going public

**Security review first.** Scan the files **and the whole Git history** for keys, passwords, tokens, `.env` files and private keys. Deleting a file in the latest commit doesn't remove it from history. Rotate anything exposed and clean the history before changing visibility. Then **Settings → General → Danger Zone → Change visibility → Public**. (Free plan: rulesets and environment reviewers are enforced only on public repos. Code, history, forks and Actions logs all become public.)

**Check `protect-main`** (**Settings → Rules → Rulesets**): Active, target `main`, PR required, 1 approval, stale approvals dismissed, most-recent-push approval. In **Settings → General → Pull Requests**, **Allow squash merging** must be on.

**Add an independent reviewer.** In **Settings → Collaborators → Add people**, invite a **second account** with **Write** access, and have that account **accept** the invite. A pending invite, or a reviewer without write access, doesn't count as an approval.

**Allow Actions to create PRs** (if you haven't already, see Stage 12): **Settings → Actions → General → Workflow permissions →** ☑ *Allow GitHub Actions to create and approve pull requests*.

**`prod` environment** (**Settings → Environments → prod**):

| Setting | Value | Why |
|---|---|---|
| Required reviewers | the **owner** account | A second human gate, separate from the PR approval |
| Prevent self-review | ☑ | Whoever triggered the run (the person who merged) can't approve their own deploy |
| Allow administrators to bypass | ☐ | No silent skips |
| Deployment branches and tags | **Selected** → `refs/pull/*/merge` (instead of `main`) | See the correction below |

> ⚠️ Corrected (clarified): was "add `main` as the allowed deployment branch" in Stage 3, now `refs/pull/*/merge`, because `04-deploy-prod.yaml` runs on a **`pull_request`** event, and for that event the run's ref is `refs/pull/<n>/merge`, not `main`. With only `main` allowed, the PROD job is rejected by the branch rule. `branches: [main]` in the workflow already ensures that only PRs **into `main`** trigger it.

**Two people, two decisions:** the *reviewer* account reviews, approves and **merges** the PR, so code enters `main`. The *owner* account **approves the PROD deployment**, so production access is authorized. Merging ≠ releasing.

#### 2. The default-branch requirement, and the one-time bootstrap

Commit everything to `feature/initial-ci`, after checking that the Secret files contain only the `PASSWORD-HERE` placeholders:

```bash
git status && git branch --show-current
git add .
git diff --cached --name-only        # review exactly what's staged
git diff --cached
git commit -m "Deploy: Complete DEV and PROD deployment automation"
git push origin feature/initial-ci
```

**Only CI runs.** Here's why:

| Workflow | Trigger | Why it didn't run |
|---|---|---|
| `01-ci.yaml` | `push` | It **did** run: the push matched. |
| `02-deploy-dev.yaml` | `workflow_run` | It must exist on the **default branch**, and it isn't in `main` yet. |
| `03-create-pr.yaml` | `workflow_call` | It only runs when called. A local `./…` call is resolved **from the caller's commit**, so it must be in `main` too. |
| `04-deploy-prod.yaml` | `pull_request: closed` | No PR has been merged yet. |

**Why GitHub insists:** a `workflow_run` workflow can use **secrets and write permissions** even when the upstream run had none. If any branch could add such a listener, an unreviewed change could leak secrets or deploy. The default branch is the trusted source (topics 31 and 37). In a mature repo you never hit this, because workflows arrive in `main` through PRs.

**Required status checks:** now that CI has reported, edit `protect-main` → ☑ **Require status checks to pass** → **select the real check names from the list** (don't type them by hand) → ☑ require branches to be up to date. Save.

**Bootstrap (one time only):**

1. Set `protect-main` to **Disabled**.
2. `git push origin HEAD:main`. This must be a **fast-forward**; never use `--force`.
3. Set `protect-main` back to **Active** immediately.
4. Check that `main` has all four workflow files plus `kubernetes/dev/` and `kubernetes/prod/`.

The earlier CI run **isn't replayed** (its `workflow_run` event has already passed). Push a fresh commit to `feature/initial-ci` to start the full chain.

#### 3. The first release

```text
push → CI ✓ → (workflow_run) DEV deploy ✓ + smoke ✓ → bot opens "Promote feature/initial-ci to main"
```

**As the reviewer account:** open the PR. Check source → `main`, the **Files changed** tab, and that the **body's SHA equals the head commit**. Confirm the required checks are green. **Review changes → Approve → Squash and merge.** (Here the diff is mostly workflows and manifests. In normal work it's application code reviewed by senior developers.)

**As the owner account:** `CWVJ Deploy | PROD` waits at the `prod` gate → **Review deployments → Approve**. Only then does the job get the environment's variables and secrets and the PROD OIDC identity.

**Verify:**

```bash
kubectl get deployments,pods,services -n prod -o wide     # images are …@sha256:<same digests as DEV>
```

Check the run log (rollouts ✓, smoke ✓, digests resolved), then use the app (create, read, update and delete users) to prove the backend and DB work. A release is verified by **artifact identity + availability + function**, not just green pods.

**Clean up the short-lived branch:** click **Delete branch** on the merged PR, then:

```bash
git switch main && git pull origin main && git fetch --prune
git branch -D feature/initial-ci     # -D: after a squash merge, git can't see the commits as merged
git branch
```

#### 4. The normal developer flow (no workflow changes needed)

```bash
git switch main && git pull origin main
git switch -c feature/normal-developer-workflow
# make a harmless change (e.g. a comment in an app file)
git add <path-to-modified-file>
git commit -m "Demonstrate normal developer workflow"
git push -u origin feature/normal-developer-workflow
```

`push → CI → DEV deploy → smoke → automated PR → human review and merge → PROD approval → PROD deploy`. Developers just push code. Automation does the rest, and humans keep the two decisions that matter.

#### 5. Tear down (stop the bill)

> 🆕 Added: the original lab ended without cleanup steps.

```bash
eksctl delete cluster -f code/capstone/infrastructure/eks-config.yaml   # cluster, node group, eksctl VPC stacks
```

Then in the console, delete: the **RDS** instance (and the snapshot, if you don't need it) and its **DB subnet group**; the **SonarQube EC2** instance; the **ECR** repositories (`--force` deletes their images); the **security groups and subnets** you created in topic 46 (before the VPC goes, if eksctl reports dependencies); and, optionally, the IAM roles, policies and the GitHub OIDC provider.

#### 6. Where to take it next (optional, not gaps)

| Direction | What to add |
|---|---|
| Manifests | **Kustomize** overlays or **Helm** charts instead of `sed` placeholders |
| GitOps | **Argo CD**: Actions updates Git, and Argo syncs, detects drift and self-heals (a follow-up video) |
| Runners | **Ephemeral self-hosted runners / ARC** in the VPC for private EKS and SonarQube access (topic 38) |
| Scale and availability | HPA (pods), Karpenter or Cluster Autoscaler (nodes), PodDisruptionBudgets ([EKS best practices](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-autoscaling.html)) |
| Secrets | AWS Secrets Manager or Vault via **ASCP / Secrets Store CSI** or **External Secrets Operator**, with IRSA or EKS Pod Identity ([AWS](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrate_eks.html), [ESO](https://external-secrets.io/)) |
| Supply chain | **SBOMs**, signing (Cosign or AWS Signer), provenance verification, **ECR enhanced scanning** ([docs](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-enhanced.html), topic 40) |
| IaC | **Terraform or CDK** for the VPC, EKS, RDS, IAM and ECR, with remote state, plans and drift detection |
| Testing | **Testcontainers** integration tests, **Playwright** or API end-to-end smoke tests. Raise coverage back up instead of lowering gates (they were lowered to 60/40 **for the demo only**). |
| Observability | CloudWatch Container Insights, OpenTelemetry tracing, dashboards, **SLIs/SLOs** with actionable alerts |
| Releases | **Blue-green or canary** with automated verification and rollback (Argo Rollouts, Flagger) |

**The thread through the whole project:** build once → scan → publish by **immutable digest** → promote the **same artifact** DEV → PROD. **OIDC** instead of keys, **least privilege** at every layer (token, IAM, EKS, RBAC), and **automation with human gates** where it matters.

---

## Hands-on lab

Follow sections 1–5 in order. Verify at each step:

1. The ruleset blocks a direct push to `main` once public (`GH013`).
2. Before the bootstrap, only CI runs on the feature push. After the bootstrap plus a fresh commit, DEV runs.
3. The PR can't be merged until the required checks pass **and** the reviewer approves.
4. PROD waits for approval, and the merging account **can't** approve it (prevent self-review).
5. `kubectl get deploy -n prod` shows the new rollout, with the same image digests as DEV.
6. Normal flow: a brand-new feature branch flows end to end with **no** workflow changes.
7. Afterwards, **tear everything down** (section 5) and consider making the repo private again.

## Where you'll use it

- Going live with any new pipeline: settings + bootstrap + a first controlled release.
- Interviews: the default-branch requirement, required checks, segregation of duties, build-once-promote-same-artifact.

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| DEV workflow doesn't run after the feature push | `workflow_run` listener not on `main`. | The bootstrap (section 2), then a fresh commit. |
| Can't select required checks | They haven't reported yet. | Push once so CI reports, then select the real names. |
| `git push origin HEAD:main` rejected (non-fast-forward) | `main` moved ahead. | Reconcile history (rebase/merge). **Never `--force`**. |
| Reviewer can't approve the PR | The invitation wasn't accepted, or no Write access. | Accept the invitation. Grant Write. |
| PROD approver blocked | Prevent self-review (the merger triggered the deploy). | The other account approves. |
| Forgot to re-enable the ruleset | Branch unprotected. | Re-enable immediately after the bootstrap. |

## Quick recap

- Settings complete the YAML: public repo (Free), ruleset + required checks, collaborator reviewer, Actions-create-PRs, PROD environment protections.
- `workflow_run` must live on the default branch, hence the one-time bootstrap with the ruleset briefly disabled.
- The full flow: CI → DEV → bot PR → human review/merge → independent PROD approval → PROD, then normal development needs no changes.
- Next steps (from the original): Kustomize/Helm, Argo CD GitOps, self-hosted/ARC runners, HPA/PDB/Karpenter, Secrets Manager/External Secrets, SBOM/signing/provenance (topic 40), Terraform/CDK, Testcontainers/Playwright, observability (CloudWatch, OpenTelemetry, SLOs), blue-green/canary.

**Interview questions**

1. **Q: Your new `workflow_run` deploy workflow doesn't trigger. Why?**
   A: It must exist on the default branch. Merge it, then generate a new upstream run (old events aren't replayed).
2. **Q: How do you enforce "CI must pass and someone else must review" before merging to main?**
   A: A ruleset with a required PR, approvals, last-push approval and the CI jobs as required status checks.
3. **Q: How does the project separate duties for production?**
   A: One person reviews and merges, and a different person approves the PROD environment deployment (prevent self-review, no admin bypass).
