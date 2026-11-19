# 30 · Branch Protection, Rulesets, Required Checks and CODEOWNERS

> **Part E: Deployment control** · Previous: [29 · Concurrency](29-concurrency.md) · Next: [31 · Chaining workflows](31-chaining-workflows.md) · [Back to INDEX](INDEX.md)
>
> Concept topic built from the capstone's Stage 3 ("Protect the main branch") and Stage 14 ("Verify the ruleset", "Configure the required CI status checks", "Bootstrap"). The capstone's exact click-by-click steps are kept in [topic 43](43-capstone-01-repo-foundation.md) and [topic 54](54-capstone-12-end-to-end-validation.md). CODEOWNERS is 🆕.

---

## What it is

GitHub Actions can tell you whether a change is good. **Rulesets** (and the older **branch protection rules**) make GitHub **enforce** it: *"nobody may change `main` unless the PR is reviewed and CI passed."*

- A **ruleset** is a named set of rules applied to **target branches or tags** (for example `main`), with an **enforcement status** (`Active`, `Evaluate`, `Disabled`) and a **bypass list** (who may skip the rules).
- A **required status check** is a CI job whose result must be ✅ before a PR can merge. Each **job** in your workflow reports a **check** on the commit.
- **CODEOWNERS** is a file that says **who must review** changes to which paths.

**Classic branch protection rules** (Settings → Branches) still work. **Rulesets** (Settings → Rules → Rulesets) are the newer, more flexible version: several can stack, they can target tags, they can be set org-wide, and they have an "Evaluate" dry-run mode.

---

## Why it exists

Without protection, anyone with write access can push straight to `main`, skipping review **and** CI, so your carefully built pipeline is optional. Production code lines need guarantees:

- changes arrive **only through PRs**
- at least **one independent reviewer** approves
- **CI must pass** on the latest commit
- **history can't be rewritten** (no force pushes) and the branch can't be deleted

> **Production insight (from the capstone):** protecting `main` from **direct pushes** and requiring **independent code review** creates a controlled path into the production codebase. Controls get stronger as the CI/CD workflows take shape.

**Jenkins comparison:** Jenkins can't enforce this on its own. It reports a build status to GitHub, and GitHub's branch protection decides. That's the same in GitHub Actions, except the checks come from your own workflows.

---

## Analogy

A **bank vault door** (`main`):

- **Require a pull request** = you can't walk in. You must submit a request form.
- **Required approvals: 1** = a second employee must sign the form.
- **Dismiss stale approvals** = if you change the form after it was signed, the signature is cancelled.
- **Require status checks** = the metal detector (CI) must beep green.
- **Block force pushes / restrict deletions** = nobody can rewrite the ledger or remove the vault.
- **Bypass list** = the bank manager's master key. Keep it empty.
- **CODEOWNERS** = "changes to the safe-deposit section must be signed by the safe-deposit manager".

---

## How it works inside

```text
PR opened/updated → workflows run → each JOB reports a check run on the head commit
                                     (name shown as "<workflow name> / <job name>")
"Merge" clicked → GitHub evaluates all ACTIVE rulesets targeting the base branch:
   ✓ PR exists (no direct push)        ✓ N approvals from users with write access
   ✓ approvals not stale               ✓ most recent push approved by someone else
   ✓ conversations resolved            ✓ required checks = success (or skipped/neutral)
   ✓ branch up to date with base (if required)   ✓ allowed merge method (e.g. squash)
   any ✗ → merge button blocked, reason shown
direct `git push origin main` → rejected by the server: "GH013: Repository rule violations found"
```

Key facts:

- A required check must have **reported at least once** before you can select it, and the name must match **exactly**. That's why the capstone adds required checks only **after the first CI run**.
- If a required check's workflow **never runs** for a PR (filtered out by `paths:`), the PR waits forever ("Expected — Waiting for status to be reported"). See topic 10.
- A job **skipped** by `if:` reports as **skipped**, which counts as passing.
- **Plan note:** on **GitHub Free**, rulesets and branch protection are **enforced only on public repos**. In a private Free repo you can create them, but they're **not enforced**. The capstone makes the repo public for that reason, after a security review of its history.

---

## Workflow YAML / commands

### A ruleset like the capstone's `protect-main`

**Settings → Rules → Rulesets → New ruleset → New branch ruleset**:

| Setting | Value | Why |
|---|---|---|
| Ruleset name | `protect-main` | |
| Enforcement status | `Active` | (`Evaluate` = log only, `Disabled` = off) |
| Target branches | Include by pattern: `main` | |
| **Restrict deletions** | ✓ | Only bypass actors can delete `main`. |
| **Block force pushes** | ✓ | No history rewrites. |
| **Require a pull request before merging** | ✓ | No direct pushes. |
| ↳ Required approvals | `1` | At least one approving review from an eligible reviewer. |
| ↳ Dismiss stale approvals when new commits are pushed | ✓ | New commits invalidate old approvals. |
| ↳ Require approval of the most recent reviewable push | ✓ | The latest push must be approved by **someone other than** the pusher. |
| ↳ Require conversation resolution | ✓ | All review threads resolved. |
| ↳ Allowed merge methods | `Squash` | Each PR becomes one commit on `main`. |
| **Require status checks to pass** | ✓ **after the first CI run** | Choose the real check names. |
| ↳ Require branches to be up to date | ✓ (with the checks) | The PR must include the latest `main`. |
| Bypass list | **empty** | Rules apply to everyone. |

(Leave "Require review from specific teams" and the "additional approval for unattributed Copilot pull requests" options off unless you need them.)

Docs: [Available rules for rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets) · [Managing rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets)

### What the check names look like

```yaml
name: CWVJ CI | Build, Scan and Publish
jobs:
  backend-job:
    name: Backend Build & Test      # ← the check name uses this display name
```

On the PR's **Checks** tab:

```text
✓ CWVJ CI | Build, Scan and Publish / Backend Build & Test (pull_request)
✓ CWVJ CI | Build, Scan and Publish / Frontend Build & Test (pull_request)
```

In the ruleset you search for and pick **`Backend Build & Test`**. **Don't guess or type check names by hand.** Select the ones GitHub reports.

### What a blocked direct push looks like

```text
$ git push origin main
remote: error: GH013: Repository rule violations found for refs/heads/main.
remote: - Changes must be made through a pull request.
remote: - Required status check "Backend Build & Test" is expected.
 ! [remote rejected] main -> main (push declined due to repository rule violations)
```

### 🆕 CODEOWNERS

Create `.github/CODEOWNERS`:

```text
# Pattern                     Owners (users or @org/team)
*                             @platform-team-lead
/.github/workflows/           @your-org/devops
/cwvj-3tier-mega-project/backend/   @your-org/backend
*.tf                          @your-org/infra
```

Then in the ruleset, enable **Require review from Code Owners**. Now a PR touching `.github/workflows/` can't merge without a DevOps approval. That's a key control: **changing the pipeline is as sensitive as changing the code** (topic 37).

### The "bootstrap" exception (capstone Stage 14)

Event-triggered workflows like `workflow_run` must **exist on the default branch** (topic 31). The capstone's one-time fix: set `protect-main` to **Disabled** → `git push origin HEAD:main` (a fast-forward, never `--force`) → set it back to **Active** immediately. In an established repo you **never** need this, because workflows arrive in `main` through normal PRs.

### 🆕 Running Actions across a whole organization (hundreds of repos)

Everything above protects **one** repo. A platform team looking after 300 repos can't click through 300 settings pages, so they set rules **once at the organization level** and let every repo inherit them. These are the pieces, and the topic that explains each building block:

| Problem at org scale | Control | Where it lives |
|---|---|---|
| Every repo must run the security scan before merge | **Org ruleset** targeting repos by name pattern or custom property, with the **"Require workflows to pass before merging"** rule pointing at one central workflow | Org → Settings → Repository → Rulesets |
| Teams copy and paste old pipelines | **Central reusable workflows** in one `platform/ci-templates` repo, called with `uses: org/ci-templates/.github/workflows/build.yml@v3` and versioned with tags (topic 22) | A dedicated repo |
| New repos start with no CI | **Workflow templates** (starter workflows) in the org's `.github` repo under `workflow-templates/`. They show up in every repo's "New workflow" page. | `org/.github` repo |
| Anyone can use any Marketplace action | **Allowed actions policy:** GitHub-owned plus verified creators plus an explicit allow-list, and **require SHA pinning** (topic 37) | Org → Settings → Actions → General |
| Old action versions everywhere | **Dependabot** `package-ecosystem: github-actions` in every repo (or org-wide via security configurations) | `.github/dependabot.yml` |
| Shared self-hosted runners leak between teams | **Runner groups** restricted to selected repos and workflows, **ephemeral** runners, ARC autoscaling (topic 38) | Org → Settings → Actions → Runner groups |
| The bill grows quietly | **Budgets and spending limits** per org or cost center, usage reports, larger runners only where they're proven to help, and arm64 / Linux by default (topic 41) | Billing → Budgets |
| Default token is too powerful | Org default **workflow permissions = read**. Repos can't widen it. | Org → Settings → Actions → General |

**Interview line:** "At org scale I'd standardize with an org ruleset that requires a central security workflow, publish versioned reusable workflows from one templates repo, lock down allowed actions with SHA pinning, isolate runners with runner groups, and set budgets. The goal is to give teams a paved road and enforce only the few rules that really matter."

---

## Hands-on lab

Use a **public** test repo (or a paid plan):

1. Create `protect-main` as above **without** required checks. Try `git push origin main` and read the `GH013` rejection.
2. Open a PR. CI runs. Add its job as a **required check**, push a commit that fails CI, and the merge button is blocked.
3. Approve the PR from a second account, push one more commit, and watch the approval being **dismissed**.
4. Add `.github/CODEOWNERS` for `.github/workflows/` and enable code-owner review.
5. Add a `paths:` filter to the required workflow, open a PR changing only `README.md`, and see the check stuck at "Expected". Fix it (remove the path filter, or use a job-level `if:`).

---

## Where you'll use it

- **Every production repo:** `main` protected, PRs plus a passing CI required.
- **Capstone:** `protect-main` (Stage 3), required CI checks (Stage 14), an independent reviewer, squash merges.
- **Org-wide rulesets:** the same rules for all repos (Enterprise/Team).
- **Tag rulesets:** only release managers can create `v*` tags that trigger release pipelines.
- **Audits:** "show that no one can deploy unreviewed code" = rulesets + environment reviewers (topic 28).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Ruleset ignored in a private Free repo | Not enforced on that plan. | Make the repo public (after a security review) or upgrade. |
| Required check stuck **"Expected — Waiting for status to be reported"** | The workflow was filtered out (`paths`, `branches`), renamed, or never triggered on PRs. | Make sure the workflow runs on every PR. Re-select the check after renames. |
| Can't find a check to require | It has never reported on this repo. | Run CI once, then search for it. |
| `GH013: Repository rule violations` on push | Direct push to a protected branch. | Open a PR. (Bootstrap only with the documented temporary disable.) |
| PR approved but still blocked | Stale approval dismissed, last push not approved by another person, unresolved conversations, branch behind base, or a code owner hasn't reviewed. | Read the merge box: it lists the failing rule. |
| Reviewer's approval doesn't count | The reviewer lacks **write** access (or the invitation wasn't accepted). | Grant write access. |
| Workflow can't push its version bump to `main` | `GITHUB_TOKEN` isn't on the bypass list. | Use a PR, or add a GitHub App to the bypass list deliberately. |

---

## Quick recap

- Rulesets = enforceable rules on branches or tags: PR required, approvals, stale dismissal, conversation resolution, required checks, up-to-date branch, no force pushes or deletions, merge method.
- Required checks = **job names** from your workflows. Add them after they've reported once.
- CODEOWNERS = path-based required reviewers. Protect `.github/workflows/` too.
- Free plan: enforced on **public** repos only.
- Keep the bypass list empty. Temporary disables only for a documented bootstrap.

**Interview questions**

1. **Q: How do you ensure nothing merges to `main` without passing CI?**
   A: A ruleset (or branch protection) on `main` requiring a PR, approvals, and the CI jobs as required status checks, with "require branches to be up to date".
2. **Q: Why is a required check sometimes stuck at "Expected"?**
   A: The workflow that produces it didn't run for that PR (path or branch filters, or a rename), so the check never reported.
3. **Q: How would you stop people from editing CI workflows without DevOps review?**
   A: A CODEOWNERS entry for `/.github/workflows/` plus "Require review from Code Owners" in the ruleset.
