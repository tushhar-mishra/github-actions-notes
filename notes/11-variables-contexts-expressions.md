# 11 · Variables, Contexts and Expressions

> **Part B: Triggers and logic** · Previous: [10 · Event filters and activity types](10-event-filters-and-activity-types.md) · Next: [12 · Functions and status checks](12-functions-and-status-checks.md) · [Back to INDEX](INDEX.md)
>
> **Video:** [GitHub Actions Workflow Logic Explained | Filters, Contexts, Variables & Expressions](https://www.youtube.com/watch?v=KIH79Y3xJ8c) (second half)

---

## What it is

GitHub Actions gives you three ways to work with information and logic:

1. **Variables** (`$VARIABLE` syntax): used inside **shell commands and scripts on the runner**, while the job is running.
2. **Contexts** (`github.*`, `runner.*`, `env.*`, `vars.*`, `secrets.*` …): objects that hold **run information**: repo, event, job, steps, runner, secrets, variables, matrix values.
3. **Expressions** (`${{ }}` syntax): small formulas inside the **workflow YAML** (at workflow, job and step level) used for **conditions, dynamic values and execution control**.

They overlap, and that's what confuses people:

| You write | What it is |
|---|---|
| `github` | a **context** (an object) |
| `github.ref` | a **context property** |
| `${{ github.ref }}` | an **expression** that reads a context value |
| `${{ github.ref == 'refs/heads/main' }}` | a **conditional expression** (gives `true`/`false`) |
| `$GITHUB_REF` | a **variable** on the runner, with the same value |

> **Note:** GitHub's docs say **variables**. Many engineers say **environment variables**, because they end up in the runner's shell environment while the job runs.

**Rule of thumb:**

- use **variables (`$VARIABLE`)** inside shell commands and scripts whenever possible
- use **contexts and expressions (`${{ }}`)** for workflow logic, conditions, orchestration and dynamic values
- if a value exists **only** as a context, you **must** use an expression

---

## Why it exists

- You need **configuration values** (app name, region) without hard-coding them everywhere → **variables**.
- You need to **know facts** about the run (which branch? who pushed? which OS?) → **contexts**.
- You need **decisions** (deploy only from `main`) → **expressions**.

**Jenkins comparison:** Jenkins gives flat environment variables (`BUILD_NUMBER`, `JOB_NAME`, `WORKSPACE`) and you write logic in Groovy (`when { branch 'main' }`). GitHub gives **structured, nested contexts** (`github.event.pull_request.title`), and logic lives in `${{ }}` expressions that GitHub evaluates **before** the runner is involved.

---

## Analogy

Think of a **theatre play**:

- **Contexts** = the **director's clipboard**: the full script, the cast list, which theatre, tonight's date. The director (GitHub's workflow engine) reads it **before** the show.
- **Expressions** = the director's **decisions** from the clipboard: "If tonight is the premiere, add the extra scene."
- **Variables** = **notes taped inside the actor's costume**. The actor (the runner's shell) can only read them **once on stage**.

The director can cancel a scene before any actor walks on stage (a job-level `if:` skips the job **before a runner is assigned**). An actor can only use what's taped inside their costume.

---

## How it works inside

```text
GitHub workflow engine (control plane)                  Runner (data plane)
──────────────────────────────────────                  ───────────────────
1. Build contexts: github, vars, secrets, inputs, needs…
2. Evaluate job-level if: ${{ … }}
       false → job SKIPPED (never reaches a runner)
       true  → job queued
3. When the job runs, for each step:
   - evaluate step if:
   - REPLACE every ${{ … }} in run:/with:/env: ──────────► 4. receives the RESULT text,
     with its computed value (text substitution)             e.g. echo "Repo: me/app"
                                                          5. shell expands $VARIABLES
                                                             from its environment
                                                             (GITHUB_*, RUNNER_*, env:)
```

Important consequences:

- **The runner never sees `${{ }}`.** By the time the command reaches Bash, `${{ github.actor }}` has already become `octocat`.
- **`$VARIABLES` only exist on the runner.** You **can't** use `$GITHUB_REF` in a job-level `if:`, because there's no shell there yet.

> **Contexts vs environment variables:**
>
> - Contexts belong to the **control plane** (the orchestration side). Runners are the **data plane** that runs scripts and commands.
> - Environment variables such as `RUNNER_NAME`, `RUNNER_OS` or `GITHUB_REF` exist **inside the runner OS** and are read with shell syntax: `$VAR` (Bash) or `$env:VAR` (PowerShell).
> - Contexts such as `${{ github.event_name }}` are **structured metadata objects** evaluated by the workflow engine with `${{ }}`.
> - Contexts can often be read **before a job is routed to a runner**. That's why they're used in `if:` conditions, matrix definitions, job configuration and conditional execution.
> - The **evaluated values** are injected into generated scripts, step metadata, temporary files or environment variables, which the runner then executes.
> - Runners **don't understand contexts**. They only receive and run the **resolved values**.

---

## Workflow YAML / commands

### 1. Variables (`$VARIABLE` syntax)

Docs: [Variables concepts](https://docs.github.com/en/actions/concepts/workflows-and-actions/variables) · [Variables reference](https://docs.github.com/en/actions/reference/workflows-and-actions/variables)

Variables are used inside **runner shell commands, scripts and runtime execution**. They store **non-sensitive configuration**: compiler flags, usernames, server names, environment-specific settings, app-specific values.

**Where you can define them (scope):**

- **`env:`** in the YAML, at **workflow**, **job** or **step** level. Only visible inside that scope.
- **Repository-level variables** (set in the UI): visible to **all workflows in that repo**.
- **Organization-level variables** (set in the UI): shared across **many repos** in a GitHub organization.
- **Environment-level variables**: tied to a deployment environment like `dev` or `prod` (topic 28).

Guideline:

- `env:` → **workflow-specific** configuration
- repository variables → values shared by **several workflows in one repo**
- organization variables → values shared by **several repos**

> **Note:** repository and organization variables are configured in the **GitHub UI** (**Settings → Secrets and variables → Actions → Variables**). `env:` variables are written in the **workflow YAML**.

> 🆕 **Precedence:** if the same name is defined at several `env:` levels, the **most specific wins**: **step > job > workflow**.

Variables come in **two types**:

#### A. Default variables provided by GitHub

GitHub sets many variables in **every** run, available to every step on the runner:

```bash
$GITHUB_EVENT_NAME
$GITHUB_REF
$GITHUB_SHA
$GITHUB_ACTOR
$RUNNER_OS
```

They describe the event, branch, commit, repo and runner.

```yaml
run: |
  echo "Event Type: $GITHUB_EVENT_NAME"
  echo "Branch Name: $GITHUB_REF_NAME"
  echo "Runner OS: $RUNNER_OS"
```

🆕 The default variables you'll use most:

| Variable | Example value | Meaning |
|---|---|---|
| `GITHUB_EVENT_NAME` | `push` | which event started the run |
| `GITHUB_REF` | `refs/heads/main` | full Git ref (branch or tag) |
| `GITHUB_REF_NAME` | `main` | short branch or tag name |
| `GITHUB_HEAD_REF` / `GITHUB_BASE_REF` | `feature/x` / `main` | PR source / target branch (PR events only) |
| `GITHUB_SHA` | `ffac537e…` | commit being built |
| `GITHUB_ACTOR` | `octocat` | who triggered the run |
| `GITHUB_REPOSITORY` | `me/app` | owner/name |
| `GITHUB_RUN_ID` / `GITHUB_RUN_NUMBER` | `91234567` / `42` | unique run ID / per-workflow counter |
| `GITHUB_WORKSPACE` | `/home/runner/work/app/app` | where checkout puts your code |
| `RUNNER_OS` / `RUNNER_ARCH` / `RUNNER_NAME` | `Linux` / `X64` / `GitHub Actions 2` | runner info |
| `RUNNER_TEMP` | `/home/runner/work/_temp` | temp folder, emptied after every job |
| `CI` | `true` | many tools check this to switch to "CI mode" |

> **Important:** default environment variables exist **only on the runner machine** running the job. They're mainly for **actual command execution**.

#### B. Custom variables created by you

You can define your own with `env:`, repository variables, organization variables and environment variables.

```yaml
env:
  APPLICATION_NAME: payment-service
  OWNER: CloudWithVarJosh
```

```yaml
run: |
  echo "Application: $APPLICATION_NAME"
  echo "Owner: $OWNER"
```

Used for reusable config, environment-specific settings, app names, deployment config and build config.

UI-defined variables are read through the **`vars` context**: `${{ vars.AWS_REGION }}`. Map one into the shell with `env:` if you want `$AWS_REGION`.

> **Security warning:** variables are **not masked** in logs. For passwords, API keys or tokens, always use **GitHub Secrets** (topic 17).

### 2. Contexts

Docs: [Contexts concepts](https://docs.github.com/en/actions/concepts/workflows-and-actions/contexts) · [Contexts reference](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)

Contexts are evaluated by the **GitHub Actions workflow engine** and give access to:

- **`github`** → workflow and repository information
- **`env`** → custom environment variables (from `env:`)
- **`vars`** → repository/organization/environment variables
- **`secrets`** → GitHub secrets
- **`runner`**, **`job`**, **`steps`**, **`matrix`** → runner, job, step and matrix information
- 🆕 also **`inputs`** (topic 20), **`needs`** (outputs of earlier jobs, topic 21), **`strategy`** (topic 26)

Used for:

- `if:` conditions
- workflow orchestration
- matrix strategies
- reusable workflow configuration
- dynamic execution logic

This topic mostly uses the **`github`** context:

```yaml
${{ github.event_name }}
${{ github.ref }}
${{ github.sha }}
${{ github.actor }}
```

```yaml
if: ${{ github.event_name == 'push' }}
```

> **Important:** contexts can often be evaluated **before the job is even sent to a runner**. That's why they're used so much for execution control and orchestration.

> **Practical note:** you'll often see contexts used inside runner commands too, because GitHub also exposes many context values as variables. Because of that overlap, both ways can look interchangeable. Rule of thumb: `$VARIABLE` in shell, `${{ }}` for workflow logic, and `${{ }}` is required when a value only exists as a context.

🆕 **Debug tip:** dump a whole context to see what's inside:

```yaml
- run: echo '${{ toJSON(github.event) }}'
```

### 3. Expressions (`${{ }}`)

Docs: [Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions)

Expressions give **dynamic evaluation, conditional logic and runtime decisions** inside workflows.

```yaml
${{ <expression> }}
```

Used for:

- `if:` conditions
- branch-based execution
- computing values dynamically
- matrix logic
- workflow orchestration
- calling functions

Examples:

```yaml
${{ github.event_name }}
${{ github.ref == 'refs/heads/main' }}
${{ startsWith(github.ref_name, 'feature/') }}
${{ runner.os }}
```

```yaml
if: ${{ github.event_name == 'push' }}
```

- `github.event_name` is a **context value**
- `== 'push'` is the **conditional logic**
- `${{ }}` is the **expression syntax**

🆕 **Operators:** `==`, `!=`, `<`, `>`, `&&` (and), `||` (or), `!` (not), `( )`. Strings use **single quotes** `'main'`. Comparisons of strings ignore upper/lower case.

Built-in functions include:

- `startsWith()`
- `contains()`
- `endsWith()`
- `success()`
- `failure()`
- `always()`

```yaml
if: ${{ startsWith(github.ref_name, 'feature/') }}
```

This runs the job only if the branch name starts with:

```text
feature/
```

Functions are covered fully in **topic 12**.

> **Important:** expressions are evaluated by the **workflow engine**, often **before** the job reaches a runner. That makes them central to orchestration and conditional execution.

> 🆕 **Note:** in `if:` you may leave out `${{ }}`: `if: github.ref == 'refs/heads/main'` works. GitHub wraps it for you. (Keep the braces when the expression starts with `!`.)

### When to use variables, contexts or expressions

They're meant for different **stages** of the run:

- **Variables (`$VARIABLE`)**: inside the **runner, during command execution**
- **Contexts (`github.*`, `runner.*` …)**: to read run metadata
- **Expressions (`${{ }}`)**: workflow logic, conditions, orchestration, execution control

The key difference:

- **contexts and expressions** can be evaluated **before** the job reaches the runner
- **variables** exist **only after** the runner starts the job

This matters for:

- `if:` conditions
- orchestration
- dynamic job execution
- runtime shell commands

```yaml
name: Contexts vs Variables Demo

on:
  push:

jobs:
  prod-check:
    if: ${{ github.ref == 'refs/heads/main' }}

    runs-on: ubuntu-latest

    steps:
      - name: Print Workflow Information
        run: |
          echo "Event Name: $GITHUB_EVENT_NAME"
          echo "Branch Name: $GITHUB_REF_NAME"
```

- `github.ref` is a **context property**.
- `${{ github.ref == 'refs/heads/main' }}` is an **expression** checking whether the run came from **`main`**.
- The **workflow engine** evaluates it first.
- The job goes to a runner **only if it's `true`**.
- Once the runner starts, `$GITHUB_EVENT_NAME` and `$GITHUB_REF_NAME` exist in the shell.
- From **`main`** → the job runs.
- From **`feature/login`** → the job is **skipped completely, before reaching a runner**.

The common pattern this shows: **contexts** give information, **expressions** control execution, and **variables** are used at runtime in scripts.

> **Production recommendation:**
>
> - **Variables (`$VARIABLE`)** → runtime command execution on the runner
> - **Contexts (`github.*`)** → reading run metadata
> - **Expressions (`${{ }}`)** → conditions, orchestration, execution control
>
> In real workflows, all three are used together.

---

### Demo: variables, contexts and expressions together

You'll see default variables, custom `env:` variables, `github.*` and `runner.*` contexts, `${{ }}` evaluation, conditional `if:`, the variable vs context difference, and the behaviour on `main` vs `feature/*`.

#### Step 1: Project structure and Git

```bash
# create the project directory
mkdir project-files

# move inside the project directory
cd project-files

# initialize a local git repository
git init

# ensure the primary branch is named main
git branch -M main

# create the github actions workflow directory structure
mkdir -p .github/workflows
```

You now have a local repo on branch `main` with the workflow folder.

#### Step 2: The workflow file

Create `.github/workflows/02-expressions-variables-contexts-demo.yaml`:

```yaml
name: 02 - Expressions Variables Contexts Demo

on:
  push:
    branches:
      - main
      - feature/*

env:
  APPLICATION_NAME: payment-service
  DEPLOYMENT_ENVIRONMENT: production
  AWS_REGION: ap-south-1

jobs:
  demo-job:
    runs-on: ubuntu-latest

    steps:
      - name: Print Variables Contexts and Expressions
        run: |
          echo "========== DEFAULT VARIABLES =========="
          echo "Event Name: $GITHUB_EVENT_NAME"
          echo "Branch Name: $GITHUB_REF_NAME"
          echo "Commit SHA: $GITHUB_SHA"
          echo "Runner OS: $RUNNER_OS"

          echo ""

          echo "========== CUSTOM VARIABLES =========="
          echo "Application Name: $APPLICATION_NAME"
          echo "Deployment Environment: $DEPLOYMENT_ENVIRONMENT"
          echo "AWS Region: $AWS_REGION"

          echo ""

          echo "========== CONTEXT VALUES =========="
          echo "Repository: ${{ github.repository }}"
          echo "Actor: ${{ github.actor }}"
          echo "Workflow: ${{ github.workflow }}"
          echo "Runner OS Context: ${{ runner.os }}"

          echo ""

          echo "========== CONTEXT VS VARIABLES =========="
          echo "Context Ref: ${{ github.ref }}"
          echo "Variable Ref: $GITHUB_REF"

          echo ""

          echo "========== EXPRESSION EVALUATION =========="
          echo "Is Main Branch: ${{ github.ref == 'refs/heads/main' }}"
          echo "Is Feature Branch: ${{ startsWith(github.ref_name, 'feature/') }}"

      - name: Demonstrate Feature Branch Condition
        if: ${{ startsWith(github.ref_name, 'feature/') }}
        run: |
          echo "Feature branch detected successfully"

      - name: Demonstrate Main Branch Condition
        if: ${{ github.ref == 'refs/heads/main' }}
        run: |
          echo "Main branch deployment logic triggered"
```

**What this shows:**

- **Default variables** `$GITHUB_EVENT_NAME`, `$GITHUB_REF_NAME`, `$GITHUB_SHA`, `$RUNNER_OS` are available automatically on the runner and describe the event, branch, commit and runner.
- **Custom variables** defined with workflow-level `env:` (`APPLICATION_NAME`, `DEPLOYMENT_ENVIRONMENT`, `AWS_REGION`) centralize reusable config and are read as `$APPLICATION_NAME`, `$DEPLOYMENT_ENVIRONMENT`, `$AWS_REGION`.
- **Contexts** `${{ github.repository }}`, `${{ github.actor }}`, `${{ github.workflow }}`, `${{ runner.os }}` give repo, workflow, actor and runner metadata.
- **Expressions** give dynamic evaluation: `${{ github.ref == 'refs/heads/main' }}` checks for `main`, and `${{ startsWith(github.ref_name, 'feature/') }}` checks for a name starting with `feature/`.
- **`if:`** gives conditional execution. The `main` step runs only on `main`, and the feature step runs only on feature branches.
- Workflows can change behaviour by **branch, event, environment and runtime conditions**.
- This conditional execution is used heavily for **production deployments, feature validation, branch-specific testing, release orchestration and environment-based automation**.

#### Step 3: Add remote, commit and push

```bash
# connect the local repository to the remote github repository
git remote add origin git@github.com:<YOUR_GITHUB_USERNAME>/<YOUR_REPOSITORY_NAME>.git

# add all changes to the staging area
git add .

# create a commit for the workflow changes
git commit -m "feat: add variables contexts expressions demo"

# push the local main branch to github
git push -u origin main
```

The repo must exist and authentication must work (topic 02). `-u` sets the upstream.

Because `on.push.branches` includes `main`, the workflow runs: a valid push happened, it targeted `main`, and all trigger conditions matched.

#### Step 4: Observe the run

**Sample log output (push to `main`):**

```text
========== DEFAULT VARIABLES ==========
Event Name: push
Branch Name: main
Commit SHA: 3f9a1c2e8b7d4e6f0a1b2c3d4e5f60718293a4b5
Runner OS: Linux

========== CUSTOM VARIABLES ==========
Application Name: payment-service
Deployment Environment: production
AWS Region: ap-south-1

========== CONTEXT VALUES ==========
Repository: your-user/cwvj-gha-practice
Actor: your-user
Workflow: 02 - Expressions Variables Contexts Demo
Runner OS Context: Linux

========== CONTEXT VS VARIABLES ==========
Context Ref: refs/heads/main
Variable Ref: refs/heads/main

========== EXPRESSION EVALUATION ==========
Is Main Branch: true
Is Feature Branch: false

⊘ Demonstrate Feature Branch Condition   (skipped)
✓ Demonstrate Main Branch Condition
    Main branch deployment logic triggered
```

It shows default variables, custom `env:` variables, context values via `${{ }}`, the overlap between contexts and variables (both refs are identical), and expression results.

- `Is Main Branch: true` and `Is Feature Branch: false`, because the run is on `main`.
- `Main branch deployment logic triggered` is printed because that step has `if: ${{ github.ref == 'refs/heads/main' }}`.
- **Demonstrate Feature Branch Condition** is **skipped** because this isn't a `feature/*` branch.

> 🆕 Look at the raw log line for the expression step. GitHub shows `Run echo "Is Main Branch: true"`, which proves `${{ }}` was replaced **before** Bash ran.

---

## Hands-on lab

1. Push the demo to `main` and compare your log with the sample.
2. Create `feature/ctx`, push, and confirm the two `if:` steps swap.
3. Add a job-level `env: AWS_REGION: us-east-1` and a step-level `env: AWS_REGION: eu-west-1`. Which value prints? (Step wins.)
4. Create a **repository variable** `TEAM=platform` in **Settings → Secrets and variables → Actions → Variables** and print `${{ vars.TEAM }}`.
5. Try `if: $GITHUB_REF == 'refs/heads/main'` at job level and read the error. Why can't a job-level `if:` see shell variables?

---

## Where you'll use it

- **Branch-based deploys:** `if: github.ref == 'refs/heads/main'`.
- **Config without hard-coding:** `vars.AWS_REGION`, `vars.ECR_REPOSITORY` (the capstone uses repository variables a lot).
- **Image tags:** `${{ github.sha }}` as the immutable image tag (topic 19 and the capstone).
- **Debugging:** `toJSON(github)` to see why a condition didn't match.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Unrecognized named-value: 'GITHUB_REF'` | You used a shell variable inside `${{ }}`. | Use the context: `github.ref`. |
| `if:` is always true | You wrote `if: "${{ … }} == main"` or a non-empty string. A non-empty string is "truthy". | Put the whole comparison **inside** one `${{ }}` (or none). |
| Variable prints empty | Wrong scope (defined in another job), a typo, or a `vars.*` name not created in the UI. | Check the scope. Names are case-sensitive in shell. |
| `secrets.X` empty in a fork PR | Secrets aren't passed to fork PRs. | Topic 17. |
| Value looks wrong for PRs | `github.ref` is `refs/pull/N/merge` on PR events. | Use `github.head_ref` / `github.base_ref`. |
| Script injection risk | Untrusted text (PR title) inserted with `${{ }}` straight into `run:`. | Pass it through `env:` and use `"$VAR"` (topic 37). |

---

## Quick recap

- **Variables** = runner shell (`$X`). **Contexts** = structured run data (`github.*`). **Expressions** = `${{ }}` logic.
- `${{ }}` is replaced **before** the runner sees the command. `$X` is expanded **by the shell** on the runner.
- Job-level `if:` runs **before** a runner is assigned, so only contexts work there.
- `env:` scope: step > job > workflow. UI variables → `vars`. Secrets → `secrets` (masked).
- Variables are not masked. Never put secrets in them.

**Interview questions**

1. **Q: Difference between `$GITHUB_REF` and `${{ github.ref }}`?**
   A: Same value. The first is expanded by the shell on the runner. The second is substituted by GitHub's engine before the step runs and can also be used where there's no shell (job `if:`, `runs-on`, `with:`).
2. **Q: Why can a job be skipped without using any runner minutes?**
   A: Its `if:` is evaluated by the control plane before the job is queued to a runner.
3. **Q: Where would you store a non-secret value used by 20 repos?**
   A: An organization-level variable, read with `${{ vars.NAME }}`.

---

### References

- [Workflow syntax](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions)
- [Events](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
- [Variables concepts](https://docs.github.com/en/actions/concepts/workflows-and-actions/variables) · [Variables reference](https://docs.github.com/en/actions/reference/workflows-and-actions/variables)
- [Contexts concepts](https://docs.github.com/en/actions/concepts/workflows-and-actions/contexts) · [Contexts reference](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)
- [Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions)
- [Multiple events](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#using-activity-types-and-filters-with-multiple-events)
