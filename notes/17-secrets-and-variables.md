# 17 · Secrets and Configuration Variables

> **Part C: Actions, secrets and tokens** · Previous: [16 · Lab: Flask CI](16-lab-flask-ci.md) · Next: [18 · `GITHUB_TOKEN` and permissions](18-github-token-and-permissions.md) · [Back to INDEX](INDEX.md)
>
> Environment-scoped secrets are covered in depth in [topic 28](28-environments.md).

---

## What it is

- A **secret** is an **encrypted** value (password, token, API key) stored in GitHub and injected into a workflow **only while it runs**. GitHub **masks** it in logs (shows `***`).
- A **configuration variable** is a **plain-text**, non-sensitive value (region, username, app name) stored in GitHub and read with `vars.NAME` (topic 11).

Both live in **Settings → Secrets and variables → Actions**, and both can be defined at three levels:

| Level | Visible to | Typical use |
|---|---|---|
| **Organization** | selected repos in the org (all, private only, or a chosen list) | shared registry credentials, company-wide settings |
| **Repository** | all workflows in that repo | the repo's own deploy token |
| **Environment** | only jobs that use `environment: <name>` (topic 28) | separate `dev` / `prod` passwords |

When the same name exists at several levels, the **most specific wins**: **environment > repository > organization**.

---

## Why it exists

A runner must log in to external systems: Docker Hub, AWS, a database. You can't write the password in YAML, because anyone who can read the repo (and its **git history, forever**) would see it.

- Each job gets its **own runner**.
- Workflows run **inside** those runners.
- The runner must **authenticate** whenever it touches protected resources.

Secrets solve this: GitHub stores the value **encrypted**, injects it at runtime, and hides it in logs.

> **Security insight:** secrets must **never be hard-coded** in workflow YAML, source code, Dockerfiles or shell scripts.

**Jenkins comparison:** GitHub secrets ≈ Jenkins **Credentials** (with the Credentials Binding plugin, `withCredentials { … }`). Configuration variables ≈ Jenkins global environment properties.

---

## Analogy

A **hotel safe-deposit box**:

- You put your passport (the token) in the hotel safe (GitHub secrets). It's locked (**encrypted**).
- When the tour guide (a workflow job) needs it, the hotel hands it over **only for the trip** and takes it back afterwards.
- If the guide reads the passport number aloud (prints it to a log), a staff member covers their mouth: `***` (**masking**).
- The noticeboard in the lobby (configuration variables) is for things anyone may read, like checkout time and the Wi-Fi network name.

---

## How it works inside

```text
You: Settings → New secret  ──►  value encrypted in the browser with the repo's public key
                                  (libsodium "sealed box") → stored encrypted by GitHub
                                  → you can NEVER read it back, only overwrite or delete

Run starts → job needs ${{ secrets.X }} ──► GitHub decrypts it and sends it to the runner
             with the job (over TLS) ──► runner registers the value in its MASK LIST
step runs → any output line containing that exact value → replaced with ***
job ends → hosted VM destroyed with everything on it
```

Facts that follow from this:

- **Masking is plain text matching.** If you transform the secret (base64-encode it, reverse it, print it character by character), the transformed text **isn't masked**. Never print secrets, even "to check".
- **Secrets are not passed to workflows triggered by a `pull_request` from a fork.** (Except `GITHUB_TOKEN`, which is read-only for forks.) Otherwise anyone could open a PR that prints your secrets. More in topic 37.
- **You can't use `secrets.*` directly in an `if:`.** Map it to `env:` first, then test `env.MY_SECRET != ''`.
- Secrets are not passed to **reusable workflows** automatically. You pass them, or use `secrets: inherit` (topic 22).
- Limits: names may contain only letters, numbers and `_`, can't start with `GITHUB_`, and are case-insensitive. Max **48 KB** per secret. Store bigger files encrypted in the repo, with the key as the secret.

---

## Workflow YAML / commands

### Step 1: Create a Docker Hub access token (not your password)

Since the workflow must push to a private Docker Hub repo, the runner needs credentials. Use a **Docker Hub access token** (Docker Hub calls them **Personal Access Tokens, PATs**), **not** your password.

> **Important:** never use your real Docker Hub account password in CI/CD workflows. Always use access tokens.

In Docker Hub: **Profile icon → Account Settings → Personal Access Tokens → Generate New Token**:

| Property | Value |
|---|---|
| Description | `gha-demo` |
| Expiration | Custom → 1 Day |
| Access permissions | Read & Write |

Copy the token **immediately**. Docker Hub shows it **only once**.

> **Operational note:** Docker Hub tokens are created at the **account level**. Any user, system, runner or workflow using the token gets **the token's permissions** across the repos that account can access, within the token's configured scope.

> **Security insight:** production setups prefer **short-lived credentials, rotation policies, least-privilege access and central secret management** over long-lived static credentials. (The best version, **no stored cloud credentials at all**, is OIDC, topic 36.)

### Step 2: Store it in GitHub Secrets

**Repository → Settings → Secrets and variables → Actions → New repository secret**

```text
Repository Settings
  → Secrets and variables
      → Actions
```

| Secret name | Secret value |
|---|---|
| `DOCKERHUB_USERNAME` | your Docker Hub username |
| `DOCKERHUB_TOKEN` | the Docker Hub access token |

> **Note:** use **your own** Docker Hub username.

Or with the GitHub CLI:

```bash
gh secret set DOCKERHUB_TOKEN            # prompts for the value (never put it in shell history)
gh variable set DOCKERHUB_USERNAME --body "your-dockerhub-user"
gh secret list
```

```text
NAME              UPDATED
DOCKERHUB_TOKEN   less than a minute ago
```

**Why GitHub Secrets matter.** GitHub automatically:

- injects secrets securely into the run
- masks secrets in logs
- prevents accidental plain-text exposure in logs

> **Scope consideration:** secrets can be defined at several levels. **Repository** secrets are available only in that repo. **Organization** secrets can be shared across many repos in an org. Choose the scope from your security needs, ownership boundaries and reuse needs.

> **Security recommendation:** usernames are less sensitive and could be stored as normal **variables**. Many organizations still store them as secrets for consistency and central credential handling. Either is fine. Tokens, passwords, API keys and cloud credentials **must** be secrets.

### Step 3: Use them in a workflow

```yaml
name: Secrets Demo
on: workflow_dispatch

jobs:
  demo:
    runs-on: ubuntu-latest
    env:
      REGION: ${{ vars.AWS_REGION }}               # configuration variable (plain)
    steps:
      - name: Login to Docker Hub (secret passed as an action input)
        uses: docker/login-action@v4
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Use a secret in a script (secret passed as an env var)
        env:
          API_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
        run: |
          echo "Region is $REGION"
          echo "Token is $API_TOKEN"                # masked in the log
          echo "Length: ${#API_TOKEN}"              # safe: prints only the length

      - name: Skip when the secret is missing
        env:
          API_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
        if: env.API_TOKEN != ''                     # secrets.* can't be used directly in if:
        run: echo "secret is configured"
```

> ⚠️ Corrected: was `docker/login-action@v3` (lectures 03 and 05), now `docker/login-action@v4` because v4 is the current major release, already used by the later lectures and the capstone. One version is used across all topic files.

**Sample log output:**

```text
Run docker/login-action@v4
  with:
    username: your-dockerhub-user
    password: ***
Login Succeeded!
Run echo "Region is $REGION"
Region is ap-south-1
Token is ***
Length: 36
✓ Skip when the secret is missing
secret is configured
```

- `password: ***`: even the action's input listing is masked.
- `Token is ***`: printing a secret only prints `***`, but still **don't do it**.

### Best practice: pass secrets through `env:`, not straight into `run:`

```yaml
# ❌ risky: the secret is pasted into the script text before it runs
run: curl -H "Authorization: Bearer ${{ secrets.API_TOKEN }}" https://api.example.com

# ✅ better: the secret arrives as an environment variable
env:
  API_TOKEN: ${{ secrets.API_TOKEN }}
run: curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com
```

Both are masked, but the `env:` form keeps the secret out of the generated script file. It's also the same pattern that prevents script injection (topic 37).

### Masking a value you generate at runtime

```bash
TOKEN=$(aws ecr get-login-password)
echo "::add-mask::$TOKEN"     # from now on, this value is printed as ***
```

(`::add-mask::` is a **workflow command**, covered in topic 23.)

---

## Hands-on lab

1. Create the Docker Hub token and the two secrets as above (or `DOCKERHUB_USERNAME` as a variable).
2. Run the Secrets Demo with **Run workflow** and confirm `***` in the logs.
3. Try `echo "$API_TOKEN" | base64` and notice the output is **not** masked. Delete that run's logs afterwards (**⋯ → Delete all logs**) and **rotate the token**. That's why you never print secrets.
4. Fork your repo with a second account (or ask a friend), open a PR, and see `API_TOKEN` empty, so the `if:` step is skipped.
5. Create an organization (free) and an org secret limited to one repo, and test access from another repo.

---

## Where you'll use it

- **Registry logins:** Docker Hub token (topics 19, 28, 32).
- **SonarQube token** (`SONAR_TOKEN`) in the capstone.
- **Database passwords per environment** (topic 28 environment secrets).
- **Cloud access:** prefer **OIDC** (topic 36) over storing `AWS_SECRET_ACCESS_KEY`.
- **Audits:** "where are our secrets, who can use them, when were they rotated?"

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Error: Username and password required` (docker/login-action) | The secret is empty: wrong name, defined in another repo/env, or the run came from a **fork PR**. | Check the name (case-insensitive). Check the scope. For forks, see topic 37. |
| Works on push, fails on PR from a fork | Secrets aren't passed to fork PRs. | Run privileged steps only on `push`, or use an approval-gated flow. **Don't** switch to `pull_request_target` blindly (topic 37). |
| `Unrecognized named-value: 'secrets'` in `if:` | Secrets can't be used directly in `if:`. | Map to `env:` and test `env.X != ''`. |
| `unauthorized: incorrect username or password` | Expired or revoked token (the demo used a 1-day expiry!) or the wrong username. | Generate a new token and update the secret. |
| Secret visible in logs | It was transformed (base64, JSON-escaped, split). | Never print secrets. Use `::add-mask::` for derived values. Rotate immediately. |
| Environment secret empty | The job doesn't declare `environment: <name>`. | Add it (topic 28). |
| `denied: requested access to the resource is denied` on push | The token is read-only, or the namespace is wrong. | Recreate the token with Read & Write. Check the `user/repo` name. |

---

## Quick recap

- Secrets = encrypted, masked, injected only at runtime. Variables = plain text, `vars.*`.
- Levels: organization, repository, environment. The most specific wins.
- Use **access tokens, not passwords**, short expiry, least privilege, rotation, or OIDC.
- Pass secrets via `env:`. Never print them. Masking is exact text matching only.
- **Fork PRs don't get secrets.**

**Interview questions**

1. **Q: Can you read a GitHub secret's value after saving it?**
   A: No. It's write-only. You can only overwrite or delete it.
2. **Q: Why are secrets empty in a pull request from a fork?**
   A: To stop untrusted code from stealing them. GitHub doesn't pass secrets (and gives a read-only `GITHUB_TOKEN`) to fork-triggered `pull_request` runs.
3. **Q: Repository secret vs environment secret?**
   A: Repo secrets are available to every job in the repo. Environment secrets only reach jobs that target that environment, and can be protected by reviewers and branch rules.
