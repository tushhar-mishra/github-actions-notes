# 18 · `GITHUB_TOKEN` and `permissions:`

> **Part C: Actions, secrets and tokens** · Previous: [17 · Secrets and variables](17-secrets-and-variables.md) · Next: [19 · Lab: Docker build and push](19-lab-docker-build-push.md) · [Back to INDEX](INDEX.md)
>
---

## What it is

For **every workflow run**, GitHub automatically creates a temporary access token called **`GITHUB_TOKEN`**. Your workflow can use it to call the GitHub API for **its own repository**: read code, comment on PRs, create releases, push packages to GHCR, open pull requests, and so on.

The **`permissions:`** key decides **what that token is allowed to do**.

```yaml
permissions:
  contents: read        # read the repo
  pull-requests: write  # comment on / create PRs
```

You read the token as `${{ secrets.GITHUB_TOKEN }}` or `${{ github.token }}`. You **don't** create it. GitHub does.

---

## Why it exists

Before this, CI tools needed a long-lived **PAT** (Personal Access Token) belonging to a real user. That's risky: it never expires, often has full access to every repo the user can see, and stops working when the person leaves.

`GITHUB_TOKEN` is:

- **automatic**: no setup
- **short-lived**: expires when the job ends (and after 24 hours at most)
- **scoped**: only to the repo running the workflow
- **least privilege**: when you set `permissions:` correctly

**Jenkins comparison:** in Jenkins you store a GitHub PAT or GitHub App credential and bind it into the job. In GitHub Actions the platform hands each run a fresh, scoped token.

---

## Analogy

A **visitor badge** at an office:

- Printed **fresh for each visitor** (each run), with **today's date** (it expires).
- It opens **only certain doors** (`permissions:`): the meeting room (`contents: read`), maybe the print room (`pull-requests: write`), never the server room unless someone explicitly allowed it.
- It works **only in this building** (this repo).
- A visitor badge **can't sign up new visitors** (by design, events created with `GITHUB_TOKEN` don't trigger new workflow runs).

---

## How it works inside

```text
run starts
  └─ GitHub creates an installation token for the built-in "GitHub Actions" app on this repo
       scopes = repo/org default  ⊕  workflow-level permissions:  ⊕  job-level permissions:
  └─ token sent to each job (masked) → available as secrets.GITHUB_TOKEN / github.token
  └─ actions like actions/checkout use it automatically (input: token, default github.token)
job ends → token revoked
```

**Default permissions** (set in **Settings → Actions → General → Workflow permissions**):

| Setting | Meaning |
|---|---|
| **Read repository contents and packages permissions** (restricted, the default for new repos and orgs) | `contents: read`, `packages: read`, `metadata: read` |
| **Read and write permissions** (permissive, older repos) | Read/write for most scopes |
| ☐ **Allow GitHub Actions to create and approve pull requests** | Off by default. Must be ticked for workflows that open PRs (capstone topic 52). |

**The `permissions:` key, rules you must know:**

1. It can be set at **workflow** level (all jobs) or **job** level (overrides it for that job).
2. **As soon as you list any permission, every permission you didn't list becomes `none`.** (The `metadata: read` permission is always granted.)
3. Shorthand values: `permissions: read-all`, `permissions: write-all`, `permissions: {}` (= nothing at all).
4. For **pull requests from forks**, the token is **read-only** no matter what you write.

The scopes you'll meet most:

| Scope | Needed for |
|---|---|
| `contents` | read the repo (checkout of private repos), push commits/tags, create releases (`write`) |
| `pull-requests` | comment on, label or create PRs |
| `issues` | create or comment on issues |
| `packages` | push or pull GHCR images |
| `id-token: write` | request an **OIDC** token for cloud login (topic 36). Doesn't touch GitHub data. |
| `actions` | cancel or re-run workflows, manage caches |
| `checks` / `statuses` | report custom checks |
| `security-events` | upload code-scanning results (SARIF) |
| `attestations` | create build attestations (topic 40) |
| `deployments` | create deployment statuses |

**Anti-loop rule:** events caused by `GITHUB_TOKEN` (a push, a new PR) **don't start new workflow runs**, except `workflow_dispatch` and `repository_dispatch`. This prevents infinite loops. If you **need** a follow-up workflow, use `workflow_run` (topic 31) or a **GitHub App token**.

---

## Workflow YAML / commands

### Least-privilege workflow

```yaml
name: Token Demo
on:
  pull_request:

permissions:
  contents: read              # workflow default: read only

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7          # uses github.token with contents: read
      - run: echo "tests pass"

  comment:
    needs: test
    runs-on: ubuntu-latest
    permissions:                           # job-level override
      pull-requests: write                 # contents is now NONE for this job (rule 2)
    steps:
      - name: Comment on the PR with the gh CLI
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}   # gh reads GH_TOKEN automatically
        run: |
          gh pr comment ${{ github.event.pull_request.number }} \
            --repo "$GITHUB_REPOSITORY" \
            --body "✅ Tests passed for ${GITHUB_SHA::7}"
```

**Sample log output (`Set up job` of the comment job):**

```text
GITHUB_TOKEN Permissions
  Metadata: read
  PullRequests: write
...
Run gh pr comment 12 --repo your-user/cwvj-gha-practice --body "✅ Tests passed for 3f9a1c2"
https://github.com/your-user/cwvj-gha-practice/pull/12#issuecomment-2345678901
```

- `GITHUB_TOKEN Permissions` in **Set up job** shows the **actual** permissions. It's the first place to look when you get a 403.
- Only `Metadata` and `PullRequests` are listed, which proves rule 2 (everything else became `none`).

### The capstone patterns

```yaml
# Pushing to Amazon ECR with OIDC: no GitHub data needed, just the OIDC token
permissions:
  id-token: write

# Opening a PR with the gh CLI (capstone 03-create-pr.yaml)
permissions:
  contents: read
  pull-requests: write
# plus: Settings → Actions → General → ☑ Allow GitHub Actions to create and approve pull requests
```

> **Note:** the ECR job doesn't need `contents: read` because it doesn't check out code. `actions/download-artifact` downloads artifacts **from the same run** with a separate internal runtime token, not `GITHUB_TOKEN`.

---

## Hands-on lab

1. Push the Token Demo and open a PR. Find **GITHUB_TOKEN Permissions** in **Set up job** for both jobs.
2. Remove `pull-requests: write` and re-run. Watch the comment step fail with **403** (`Resource not accessible by integration`).
3. In the `test` job, set `permissions: {}`. Checkout of a **private** repo now fails. Why? (`contents: none`.)
4. Add a step that pushes a tag using `GITHUB_TOKEN` (`contents: write`). Confirm that a workflow listening on `push: tags` is **not** triggered (the anti-loop rule).

---

## Where you'll use it

- **PR bots:** comments, labels, auto-created PRs (capstone topic 52).
- **Releases:** `contents: write` to create GitHub Releases and tags.
- **GHCR:** `packages: write` to push images to `ghcr.io`.
- **Cloud login:** `id-token: write` (topic 36).
- **Security reviews:** "set workflow-level `contents: read`, grant more only per job" is a standard hardening rule (topic 37).

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `HttpError: Resource not accessible by integration` / **HTTP 403** | The token lacks a permission. | Check **Set up job → GITHUB_TOKEN Permissions**. Add the scope at job level. |
| 403 only on PRs from forks | Fork tokens are always read-only. | Do write actions in a `workflow_run` follow-up (topic 31) that runs in the base repo. |
| `GitHub Actions is not permitted to create or approve pull requests` | Repo/org setting is off. | **Settings → Actions → General → ☑ Allow GitHub Actions to create and approve pull requests**. |
| Checkout fails `repository not found` on a private repo after adding `permissions:` | You listed some scopes, so `contents` became `none`. | Add `contents: read`. |
| `Error: Could not get ID token` / `Unable to get ACTIONS_ID_TOKEN_REQUEST_URL` | `id-token: write` is missing. | Add it to the job (topic 36). |
| Workflow pushed a commit but CI didn't run on it | The anti-loop rule. | Use a GitHub App token or PAT for that push, or chain with `workflow_run`. |
| Need to access **another** repo | `GITHUB_TOKEN` only covers the current repo. | Use a GitHub App installation token (`actions/create-github-app-token`) or a fine-grained PAT stored as a secret. |

---

## Quick recap

- `GITHUB_TOKEN` = automatic, per-run, repo-scoped, short-lived token.
- `permissions:` at workflow or job level. **Listing any scope sets the rest to `none`.**
- Default for new repos: read-only. Fork PRs: always read-only.
- Events made with `GITHUB_TOKEN` don't trigger new runs (anti-loop).
- 403 → read **GITHUB_TOKEN Permissions** in **Set up job**.

**Interview questions**

1. **Q: How do you apply least privilege to `GITHUB_TOKEN`?**
   A: Set `permissions: contents: read` at workflow level, and grant extra scopes (for example `pull-requests: write`) only on the jobs that need them.
2. **Q: Why didn't my workflow run after another workflow pushed a commit?**
   A: Events triggered by `GITHUB_TOKEN` don't create new runs, to prevent loops. Use `workflow_run`, a GitHub App token, or a PAT.
3. **Q: What does `id-token: write` do?**
   A: It lets the job request an OIDC token to authenticate to a cloud provider. It doesn't grant access to repository data.
