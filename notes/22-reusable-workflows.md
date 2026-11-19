# 22 · Reusable Workflows (`workflow_call`): Inputs, Outputs and Secrets

> **Part D: Moving data around** · Previous: [21 · Step and job outputs](21-step-and-job-outputs.md) · Next: [23 · Workflow commands and job summaries](23-workflow-commands-and-summaries.md) · [Back to INDEX](INDEX.md)
>
> Videos: [Inputs](https://youtu.be/fLZRgNhQq3w) · [Outputs](https://www.youtube.com/watch?v=gkIXTCr1Iiw)
>
> **Code:** [`code/22-reusable-workflows/`](code/22-reusable-workflows/)

---

## What it is

A **reusable workflow** is a normal workflow file whose trigger is **`on: workflow_call`**. Other workflows **call it like a function**:

```yaml
jobs:
  deploy:
    uses: ./.github/workflows/deploy.yaml      # call it
    with:                                       # pass inputs
      application_name: payment-service
```

- It can **receive inputs** (`on.workflow_call.inputs`).
- It can **return outputs** (`on.workflow_call.outputs`).
- It can **receive secrets** (`on.workflow_call.secrets`, or `secrets: inherit`), 🆕.

> **Important:** think of **`workflow_dispatch`** as **user-to-workflow** communication, and **`workflow_call`** as **workflow-to-workflow** communication.

Unlike `workflow_dispatch`, a reusable workflow **can't be started directly by a user** (unless it also lists another trigger). It runs only when another workflow invokes it. The inputs from the caller appear in its **`inputs`** context.

---

## Why it exists

Without reuse, 50 repos each copy the same deployment YAML, and a fix must be made 50 times.

**Production example:** a company sets standards such as:

- all apps deploy through a **common workflow**
- all deployments run **security validation**
- all deployments follow a **standard approval process**
- all deployments produce **metadata and audit logs**

Instead of every team implementing these, the **platform engineering** team publishes **one reusable deployment workflow** that every team calls.

```text
GitHub Organization

├── payment-service
├── inventory-service
├── notification-service
├── order-service
├── ...
└── shared-platform-workflows
      └── deploy.yaml (Reusable Workflow)
```

| Without reusable workflows | With reusable workflows |
|---|---|
| 50 repositories, 50 deployment workflows | 50 repositories, 50 lightweight calling workflows, 1 reusable deployment workflow |
| Deployment logic duplicated across repositories | Deployment logic centralized in a reusable workflow |
| Higher maintenance overhead | Easier maintenance and governance |
| Changes require updates across multiple repositories | Changes implemented once and inherited by all consumers |

This improves **standardization, reusability, security and operational consistency**.

> **Production insight:** in large **GitHub Organizations**, platform teams keep reusable workflows in **central repos**. App repos keep small **calling workflows** with app-specific config, while the shared repo holds the deployment, security, compliance and governance logic. That cuts duplication and maintenance and enforces consistent CI/CD standards.

**Jenkins comparison:** reusable workflows are conceptually like **Jenkins Shared Libraries**: central, reusable pipeline logic that many pipelines call, for **standardization, reusability, governance and maintainability**. A Jenkins pipeline calls functions from a shared library. A GitHub workflow calls a reusable workflow with `uses:`. The details differ, but the goal is the same.

---

## Analogy

A **restaurant franchise's central kitchen manual**:

- Head office (the platform team) writes **one** recipe, "how to make the signature burger" (`deploy.yaml`).
- Each branch (each app repo) just says "make the signature burger, **with cheese, 5 portions**" (`uses:` + `with:` inputs).
- The recipe **hands back** a receipt: "order #42 ready at counter 3" (**outputs**).
- Branches can hand over their own **key to the storeroom** (**secrets**), either one key at a time (`secrets:` mapping) or the whole key ring (`secrets: inherit`).
- When head office improves the recipe, **every branch gets the improvement** on its next order.

---

## How it works inside

```text
Calling workflow run (triggered by push / dispatch / anything)
 │
 ├─ job A (normal job, runs on a runner)
 ├─ job "deploy" with  uses: org/platform/.github/workflows/deploy.yaml@v1
 │      │   (a "caller job": it has NO runs-on and NO steps of its own)
 │      ▼
 │   GitHub loads deploy.yaml at ref v1 → validates the inputs against on.workflow_call.inputs
 │   → its jobs become part of THIS run (shown nested: "deploy / deploy-application-job")
 │   → each of its jobs runs on its own runner, as usual
 │   → when done, on.workflow_call.outputs are evaluated from ITS jobs' outputs
 │      ▼
 │   values attached to the caller job: needs.deploy.outputs.<name>
 │
 └─ job "notify" (needs: deploy) reads needs.deploy.outputs.deployment_url
```

Facts that follow (🆕 unless noted):

- The called workflow runs **in the caller's run**: same `github` context (event, SHA, actor), and the **`GITHUB_TOKEN` permissions** can only be equal to or **lower** than the caller's.
- **`env:` defined in the caller is NOT passed** into the reusable workflow. Use inputs.
- **Secrets are NOT passed automatically.** Map them, or use `secrets: inherit` (same org/enterprise only).
- **Location:** must be in `.github/workflows/` (no subfolders). Same repo: `./.github/workflows/x.yaml`. Other repo: `owner/repo/.github/workflows/x.yaml@ref`. For **private** repos, the repo holding the reusable workflow must allow access (**Settings → Actions → General → Access**).
- **Nesting limits:** a reusable workflow can call others. In **November 2025** GitHub raised the limits on GitHub.com to **10 levels deep** and **50 unique reusable workflows** per top-level caller (counting the whole nested tree). They used to be 4 and 20, which still apply on older GitHub Enterprise Server versions.
- The caller job can use `needs`, `if`, `strategy.matrix`, `permissions`, `concurrency` and `secrets`, but **not** `runs-on`, `steps`, `env` or `timeout-minutes`.

---

## Workflow YAML / commands

### Part 1: Reusable workflow inputs

Reusable workflow inputs pass values **from a calling workflow to a reusable workflow**. They're defined with:

```yaml
on:
  workflow_call:
```

Unlike **workflow inputs** (usually from **users**, topic 20), reusable workflow inputs come **from another workflow** through **`with:`**. That lets companies **standardize common logic** across repos, apps and teams while still supplying app-specific values at runtime.

The reusable workflow declares its inputs:

```yaml
on:
  workflow_call:
    inputs:
      application_name:
        required: true
        type: string

      run_security_scan:
        required: true
        type: boolean

      replicas:
        required: true
        type: number
```

- **`application_name`** (`string`): which app (for example `payment-service`)
- **`run_security_scan`** (`boolean`): run security validation before deploying?
- **`replicas`** (`number`): how many app instances to deploy

Supported types for reusable workflow inputs:

- **`string`** → app names, image tags, environment names, release IDs
- **`boolean`** → switch optional behaviour (`run_security_scan: true`)
- **`number`** → replica counts, timeouts, scaling parameters

(There's no `choice` type here. That one is only for `workflow_dispatch`.) The right type improves **validation, readability and design**.

The inputs are **organization-specific**: one company needs `application_name`, `run_security_scan` and `replicas`, another also requires `change_ticket`, `cost_center` or `region` for governance, compliance or operations.

The **calling workflow** invokes it:

```yaml
jobs:
  deploy:
    uses: ./.github/workflows/deploy.yaml
    with:
      application_name: payment-service
      run_security_scan: true
      replicas: 5
```

- **`deploy.yaml`** is the reusable workflow.
- The caller passes **deployment-specific values**.
- The reusable workflow reads them through the **`inputs`** context:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Display Deployment Information
        run: |
          echo "Application: ${{ inputs.application_name }}"
          echo "Run Security Scan: ${{ inputs.run_security_scan }}"
          echo "Replicas: ${{ inputs.replicas }}"

      - name: Deploy Application
        run: |
          echo "Deploying application..."
```

```text
Application: payment-service
Run Security Scan: true
Replicas: 5
```

#### Demo 2: centralize deployment logic with reusable workflow inputs

Platform teams build reusable workflows containing common deployment standards, security checks, compliance controls and operational practices. App teams call them and pass app-specific values.

```text
Payment Service Repository

run-unit-tests-job
          ↓
build-payment-artifacts-job
          ↓
deploy-payment-service-job
          ↓
Reusable Deployment Workflow
          ↓
deploy-application-job
```

- The **calling workflow** holds **app-specific** logic.
- The **reusable workflow** holds **shared** deployment logic.
- Inputs flow from caller to callee, and the callee behaves dynamically.

You'll learn: how reusable workflows are created and invoked, how inputs pass between workflows, how `needs` creates dependencies, how platform teams standardize deployments, and how companies cut duplication.

**Step 1: the reusable workflow** `.github/workflows/deploy.yaml`:

```yaml
name: Reusable Deployment Workflow

on:
  workflow_call:
    inputs:
      application_name:
        required: true
        type: string

      run_security_scan:
        required: true
        type: boolean

      replicas:
        required: true
        type: number

jobs:
  deploy-application-job:
    runs-on: ubuntu-latest

    steps:
      - name: Display Deployment Parameters
        run: |
          echo "Application: ${{ inputs.application_name }}"
          echo "Run Security Scan: ${{ inputs.run_security_scan }}"
          echo "Replicas: ${{ inputs.replicas }}"

      - name: Security Validation
        if: ${{ inputs.run_security_scan }}
        run: |
          echo "Running security scan..."
          echo "No critical vulnerabilities found."

      - name: Deploy Application
        run: |
          echo "Deploying ${{ inputs.application_name }}"
          echo "Desired Replicas: ${{ inputs.replicas }}"

      - name: Deployment Complete
        run: |
          echo "Deployment completed successfully."
```

- **`name:`**: shown in the UI. A reusable workflow is still a workflow and appears in history. Clear names matter as you create reusable workflows for deployment, security, testing, compliance and platform automation.

  > **Production insight:** platform teams maintain **dozens** of reusable workflows. Consistent naming makes them easy to discover, maintain and govern.

- **`on: workflow_call:`**: the reusable trigger. Users can't start it directly. It runs only when invoked, and caller inputs appear in `inputs`.

  > **Production insight:** reusable workflows are a **foundation of platform engineering**, enforcing common standards across hundreds of repos from one central workflow.

- **`inputs:`**: the **contract** between caller and callee. The caller must provide all required inputs, and GitHub **validates them before execution**.

  | Input | Type | Purpose |
  |---|---|---|
  | `application_name` | string | Application being deployed |
  | `run_security_scan` | boolean | Enable or disable security validation |
  | `replicas` | number | Desired deployment replica count |

  > **Production insight:** contracts often include **application name, environment, change ticket, region, cost center, deployment strategy and approval IDs**.

- **`deploy-application-job`**: all deployment activity runs here, fed by `inputs`. Because it's central, many repos reuse the same process.

  > **Production insight:** improvements are made **once** and every consuming repo gets them.

- **Display Deployment Parameters**: confirms the values arrived (application, security scan, replicas). That helps troubleshooting.

  > **Operational tip:** logging runtime parameters at the start of a reusable workflow simplifies **troubleshooting, change reviews and audits**. Many companies record app, environment, version, region and strategy.

- **Security Validation** (`if: ${{ inputs.run_security_scan }}`): runs only if the caller passed `true`. It shows the callee changing behaviour based on the caller.

  > **Production insight:** centralizing security validation means every app follows the same security process, without relying on each team.

- **Deploy Application**: uses `application_name` and `replicas`. One workflow, many apps, app-specific parameters.

  > **Production insight:** in reality this step would run **Helm deployments, Kubernetes manifest updates, Argo CD syncs, Terraform** or cloud deployments. The mechanism is central, the parameters vary.

- **Deployment Complete**: a clear end point. Such steps often record status, publish metadata or trigger downstream automation.

  > **Production insight:** production deploy workflows often add **notifications, audit logs, change-management updates, release tracking, monitoring integration and post-deployment validation**.

**Step 2: the calling workflow** `.github/workflows/payment-service-deployment.yaml`:

```yaml
name: Payment Service Deployment

on:
  workflow_dispatch:
  push:

jobs:
  run-unit-tests-job:
    runs-on: ubuntu-latest

    steps:
      - name: Execute Unit Tests
        run: |
          echo "Running payment service unit tests..."
          echo "All tests passed."

  build-payment-artifacts-job:
    runs-on: ubuntu-latest

    needs:
      - run-unit-tests-job

    steps:
      - name: Build Payment Artifacts
        run: |
          echo "Building payment service artifacts..."
          echo "Artifacts generated successfully."

  deploy-payment-service-job:
    needs:
      - build-payment-artifacts-job

    uses: ./.github/workflows/deploy.yaml

    with:
      application_name: payment-service
      run_security_scan: true
      replicas: 5
```

> **Job and step execution (recap of topic 06):** jobs run **in parallel** by default, each on its own runner. **`needs`** creates dependencies and order. Steps run **in order** inside a job. If a **step fails**, the rest of its job is skipped and the job fails. If a **job fails**, jobs that `need` it are skipped, unless they use a status function like **`always()`** (topic 12).
>
> ```text
> Run Unit Tests
>        ↓
> Build Artifacts
>        ↓
> Deploy Application
> ```
>
> If **Run Unit Tests** fails, **Build Artifacts** and **Deploy Application** don't run. If a step in **Build Artifacts** fails, the deploy job never starts. A step or job with **`always()`** would still run.
>
> **Production insight:** this "stop early" behaviour is a key reason pipelines are reliable. Problems are caught early and don't reach packaging, publishing or production. `always()` is kept for clean-up, notifications, log collection and reporting.

- **`name: Payment Service Deployment`**: identifies the workflow's purpose (deploying payment-service).

  > **Operational tip:** repos often have separate workflows for **CI, CD, security scanning, dependency management, release automation and operations**. Good names help during troubleshooting.

- **`on: workflow_dispatch:` + `push:`**: runs on manual start **or** on push. Several triggers let one workflow serve different scenarios, and either way it ends up calling the reusable workflow.

  > **Production insight:** organizations commonly support **automated and manual** paths. Deploy automatically after a push, and also allow manual deploys, rollbacks, maintenance or emergency procedures.

  > **Important:** reusable workflows are **independent of the caller's trigger**. The caller may be triggered by `workflow_dispatch`, `push`, `pull_request`, `release`, `schedule` or anything else.

- **`run-unit-tests-job`**: validates the app before any build or deploy. It's **app-specific** logic owned by the app team. It runs on a fresh, isolated Ubuntu runner.

  > **Production insight:** unit tests are one of the **earliest quality gates**: fast feedback that stops defective code early.
  >
  > **Operational note:** each job has its own runner, so files from one job aren't in another unless shared via **artifacts, caches, outputs or external storage**.

- **Execute Unit Tests**: simulated. A real app runs framework-specific test commands.

  > **Production insight:** mature teams enforce minimum **test coverage, quality gates and PR validation** before deployments proceed.

- **`build-payment-artifacts-job`** with **`needs: [run-unit-tests-job]`**: generates deployment artifacts **after** unit tests succeed. It's skipped automatically if tests fail. Building separately keeps testing and packaging responsibilities apart. Also app-specific.

  > **Important:** **`needs`** creates **job-level** dependencies, not step-level. Steps are already sequential.
  >
  > **Production insight:** artifacts include **Docker images, JAR, WAR and ZIP files, Helm charts** and other deployable bundles. Pipelines such as **Build → Test → Security Scan → Package → Deploy** are chained with `needs`. **Build Once, Deploy Many**: promote one validated artifact through Dev, QA, Stage and Prod.

- **`deploy-payment-service-job`**: **delegates** deployment to the reusable workflow instead of containing steps. It's the bridge between app-specific logic and shared organizational standards.

  > **Production insight:** platform teams encourage app teams to use central deployment workflows instead of maintaining their own.

  - **`needs: [build-payment-artifacts-job]`**: deployment can't start until artifacts exist. GitHub evaluates the dependency before scheduling. If the build fails, deployment never starts.

    > **Operational note:** chaining with `needs` gives **predictable, auditable** pipelines.

  - **`uses: ./.github/workflows/deploy.yaml`**: invokes the reusable workflow. GitHub **hands control** to it. This is the core mechanism of workflow reuse.

    > **Production insight:** companies keep reusable workflows for **deployments, security scanning, compliance, testing, release management** and more.

  - **`with:`**: passes values into the callee's **`inputs`** context. The same reusable workflow then supports many apps and scenarios.

    | Input | Purpose |
    |---|---|
    | **application_name** | Identifies the application being deployed |
    | **run_security_scan** | Determines whether security validation executes |
    | **replicas** | Specifies the desired deployment replica count |

    > **Production insight:** this is why reusable workflows are so powerful. Logic is central, and teams provide **environment, version, region, replica count, strategy or approval IDs**.

**Step 3: commit and push**

```bash
# Add all workflow files and related changes to the Git staging area
git add .

# Create a commit containing the Reusable Workflow Inputs demo implementation
git commit -m "feat: add reusable workflow inputs demo"

# Associate the local repository with the remote GitHub repository (one-time setup)
git remote add origin git@github.com:<YOUR_GITHUB_USERNAME>/cwvj-gha-practice.git

# Push the code to GitHub and configure the local branch to track origin/main
git push -u origin main
```

Because the caller has `push`, it runs automatically. It can also be run with **Run workflow**.

> **Git best practice:** many teams follow the **Conventional Commits** specification (`feat:`, `fix:`, `docs:`, `refactor:`, `chore:`) to standardize history and enable automated releases, **semantic versioning** (MAJOR.MINOR.PATCH, like `v2.4.1`) and automated changelogs (topic 35).

> **Key observation:** the reusable workflow isn't triggered directly. GitHub runs the **caller**, which invokes it with **`uses:`**.

**Step 4: observe and verify.** **Actions → Payment Service Deployment**:

```text
run-unit-tests-job
          ↓
build-payment-artifacts-job
          ↓
deploy-payment-service-job
          ↓
Reusable Deployment Workflow
          ↓
deploy-application-job
```

**Sample log output (by job):**

```text
run-unit-tests-job
    Running payment service unit tests...
    All tests passed.
build-payment-artifacts-job
    Building payment service artifacts...
    Artifacts generated successfully.
deploy-payment-service-job / deploy-application-job      ← nested name = caller job / called job
    Application: payment-service
    Run Security Scan: true
    Replicas: 5
    Running security scan...
    No critical vulnerabilities found.
    Deploying payment-service
    Desired Replicas: 5
    Deployment completed successfully.
```

Check that:

- **`needs`** controls the job order
- the deploy job **calls** `uses: ./.github/workflows/deploy.yaml` instead of running deploy steps itself
- the **inputs propagated** (`payment-service`, `true`, `5`)
- **Security Validation** ran because `run_security_scan: true` was supplied
- the deploy stage consumed the inputs
- both the caller and the reusable workflow **completed successfully**

> **Key observation:** the **caller** holds **app-specific logic**, and the **reusable workflow** holds **shared organizational logic**. **Reusable workflow inputs are the bridge.**

---

### Part 2: Reusable workflow outputs

Reusable workflow outputs let a reusable workflow **return information to the caller**.

Reusable workflows centralize **deployments, security validation, compliance, platform operations and infrastructure provisioning**. But often the callee produces information that the caller needs **afterwards**:

```text
Calling Workflow
         ↓
Invokes Reusable Deployment Workflow

Reusable Workflow
         ↓
Deploys Application
         ↓
Creates Deployment URL

Calling Workflow
         ↓
Needs Deployment URL
```

The callee produces `https://cwvj-demo.com`. The caller may need it to:

- send deployment notifications
- create change-management records
- update monitoring systems
- configure synthetic health checks (automated probes that act like a user)
- record deployment metadata for audits

**How can a reusable workflow return information to the caller?** With **reusable workflow outputs**. Values produced inside it are exposed and returned **after it completes successfully**.

**Reusable workflow:**

```yaml
on:
  workflow_call:
    outputs:
      deployment_url:
        value: ${{ jobs.deploy-job.outputs.deployment_url }}

jobs:
  deploy-job:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Application
        id: deploy
        run: |
          echo "deployment_url=https://cwvj-demo.com" >> $GITHUB_OUTPUT

    outputs:
      deployment_url: ${{ steps.deploy.outputs.deployment_url }}
```

**Calling workflow:**

```yaml
jobs:
  deploy-job:
    uses: ./.github/workflows/deploy.yml

  notify-job:
    needs:
      - deploy-job
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "${{ needs.deploy-job.outputs.deployment_url }}"
```

**Inside the reusable workflow, three layers:**

1. **Step output.** The `deploy` step (`id: deploy`) writes `deployment_url=https://cwvj-demo.com` to `$GITHUB_OUTPUT` (topic 21). Output name `deployment_url`, value `https://cwvj-demo.com`. Visible only to steps in that job.
2. **Job output.** `outputs: deployment_url: ${{ steps.deploy.outputs.deployment_url }}` maps it to a job output on `deploy-job` (`steps` = earlier steps, `deploy` = the ID, `outputs` = its values, `deployment_url` = the name). It now crosses the step boundary, **but it's still confined to the reusable workflow.**
3. **Reusable workflow output.** Under **`on.workflow_call.outputs`**, because it's what the workflow **returns**:

   ```yaml
   on:
     workflow_call:
       outputs:
         deployment_url:
           value: ${{ jobs.deploy-job.outputs.deployment_url }}
   ```

   `jobs` = the jobs **in this reusable workflow**, `deploy-job` = the job that made it, `outputs` = its job outputs, `deployment_url` = the one to return. GitHub resolves it to `https://cwvj-demo.com` and makes it available to the caller **after the reusable workflow completes successfully**.

> **Important:** reusable workflow outputs **generate nothing**. They expose **job outputs** to the caller. The URL is created **once**, in the deploy step, and each layer just widens its visibility:
>
> ```text
> Step Output
>       ↓
> Job Output
>       ↓
> Reusable Workflow Output
> ```
>
> **Connection to topic 21:** job outputs cross **job boundaries**. Reusable workflow outputs cross **workflow boundaries**.

**In the caller:**

- `deploy-job` calls the reusable workflow with **`uses`**.
- When it finishes, GitHub **automatically exposes its outputs through the job that invoked it**, here `deploy-job`.

  > **Important:** in the **caller**, `deploy-job` has **no `outputs:` block**. The reusable workflow's outputs are mapped onto it automatically.

- `notify-job` (`needs: [deploy-job]`, so it waits for the deployment and can read its outputs) reads `${{ needs.deploy-job.outputs.deployment_url }}`: `needs` = dependencies, `deploy-job` = **the caller job that invoked the reusable workflow**, `outputs` = what that workflow returned, `deployment_url` = the name. It resolves to `https://cwvj-demo.com`.

> **Key observation:** step outputs = between steps, job outputs = between jobs, reusable workflow outputs = between workflows. Each layer **widens visibility**.

> **Connection to Part 1:** reusable workflow **inputs** pass values **into** a reusable workflow. **Outputs** are the **reverse**: values back to the caller.

**Common uses:** returning **deployment URLs, release versions, security validation results, compliance results, generated resource IDs, environment-specific deployment info** and other platform metadata.

> **Important:** reusable workflow outputs are for **values and metadata**, not files, reports, binaries or logs. Use **artifacts** (topic 24). Artifacts uploaded inside a reusable workflow are part of the **same run**, so the caller's later jobs can download them.

> **Production insight:** platform teams hide complex logic inside reusable workflows but still **return critical information** to app teams.

> **Mental model:**
>
> ```text
> Input
>   ↓
> environment = prod
>
> Workflow Executes
>
> Output
>   ↓
> deployment_url = https://cwvj-demo.com
> ```
>
> ```text
> Inputs  → Data entering the workflow
> Outputs → Data leaving the workflow
> ```
>
> Not a strict technical definition, but a useful picture: **inputs** feed workflows, jobs, actions and reusable workflows, and **outputs** expose what was generated to downstream consumers.

#### Demo 3: return deployment information with reusable workflow outputs

> **Learning observation:** the **same deployment URL** example continues. The business scenario stays the same, and only the **scope** of sharing changes (step → job → workflow).

**Step 1: the reusable workflow.** [`code/22-reusable-workflows/deploy-application.yaml`](code/22-reusable-workflows/deploy-application.yaml), saved as `.github/workflows/deploy-application.yaml`:

```yaml
name: Centralized Deployment Workflow

on:
  workflow_call:
    outputs:
      deployment_url:
        description: Deployment URL
        value: ${{ jobs.deploy-job.outputs.deployment_url }}

jobs:
  deploy-job:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Application
        id: deploy
        run: |
          echo "Deploying application..."
          sleep 3
          echo "deployment_url=https://cwvj-demo.com" >> $GITHUB_OUTPUT

    outputs:
      deployment_url: ${{ steps.deploy.outputs.deployment_url }}
```

> **Important:** job outputs don't generate values, they expose step values. Reusable workflow outputs don't generate values, they expose job outputs. (`description:` documents the output for consumers.)

**Step 2: the calling workflow.** [`code/22-reusable-workflows/03-reusable-workflow-outputs-demo.yaml`](code/22-reusable-workflows/03-reusable-workflow-outputs-demo.yaml):

```yaml
name: 03 - Reusable Workflow Outputs Demo

on:
  workflow_dispatch:

jobs:
  deploy-job:
    uses: ./.github/workflows/deploy-application.yaml

  notification-job:
    runs-on: ubuntu-latest
    needs:
      - deploy-job
    steps:
      - name: Send Notification
        run: |
          echo "Sending notification..."
          echo "Application URL:"
          echo "${{ needs.deploy-job.outputs.deployment_url }}"
          echo "Notification sent successfully."
```

> **Important:** the caller's `deploy-job` defines **no `outputs` block**.

> **Key observation:** the URL is generated inside the reusable workflow, exposed as a reusable workflow output, **mapped automatically onto `deploy-job`**, and read by `notification-job` through `needs`.

**Steps 3–4: commit, push and run**

```bash
# Add the workflow files to Git tracking
git add .

# Create a commit
git commit -m "feat: add reusable workflow outputs demo"

# Push changes to GitHub
git push
```

**Actions → 03 - Reusable Workflow Outputs Demo → Run workflow**.

**Step 5: observe (sample log output)**

```text
deploy-job / deploy-job
    Deploying application...
notification-job
    Sending notification...
    Application URL:
    https://cwvj-demo.com
    Notification sent successfully.
```

> **Key observation:** step outputs = between steps, job outputs = between jobs, reusable workflow outputs = between workflows. Same idea throughout: **one execution unit creates a value, another consumes it.**

---

### Part 3 (🆕): Passing secrets, calling across repos, versioning

```yaml
# reusable workflow: declare the secrets it needs
on:
  workflow_call:
    inputs:
      environment: { type: string, required: true }
    secrets:
      DOCKERHUB_TOKEN:
        required: true

jobs:
  push:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}       # environments work inside reusable workflows (topic 28)
    steps:
      - run: echo "token length ${#T}"
        env: { T: ${{ secrets.DOCKERHUB_TOKEN }} }
```

```yaml
# caller, option A: pass explicitly (works across orgs, clearest)
jobs:
  push:
    uses: my-org/platform-workflows/.github/workflows/push.yaml@v1
    with:
      environment: dev
    secrets:
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}

# caller, option B: hand over all of the caller's secrets (same org/enterprise only)
jobs:
  push:
    uses: my-org/platform-workflows/.github/workflows/push.yaml@v1
    with:
      environment: dev
    secrets: inherit
```

- `@v1` / `@<sha>` versions the reusable workflow exactly like an action (topic 15). Platform teams tag releases so app repos can upgrade on purpose.
- `secrets: inherit` is convenient but gives the callee **every** secret. Prefer explicit mapping for third-party or cross-team workflows.

### Reusable workflow vs composite action (preview of topic 32)

| | Reusable workflow | Composite action |
|---|---|---|
| Unit of reuse | **Jobs** (whole pipeline stages) | **Steps** inside one job |
| Called with | `jobs.<id>.uses:` | `steps[*].uses:` |
| Own runners | Yes, each job picks `runs-on` | No, runs on the caller's runner |
| Secrets | `secrets:` / `inherit` | Passed as inputs |
| Environments / approvals | Yes | No |
| Shows in logs | Separate jobs | One step (with sub-steps) |

---

## Hands-on lab

1. Build Demo 2 (inputs) and Demo 3 (outputs) in your practice repo.
2. Combine them: make `deploy.yaml` take `application_name` **and** return `deployment_url`, then print it in a notify job.
3. Call it with a **matrix** (topic 26 preview): `strategy: matrix: app: [payment, orders]` on the caller job, and `with: application_name: ${{ matrix.app }}`.
4. Move `deploy.yaml` to a second repo, tag it `v1`, allow access (**Settings → Actions → General → Access**), and call it with `owner/repo/.github/workflows/deploy.yaml@v1`.
5. Add a required secret and try calling without passing it. Read the error, then fix it with `secrets: inherit`.

---

## Where you'll use it

- **Platform engineering:** one golden CI/CD workflow for 100+ repos.
- **Capstone (topic 52):** `03-create-pr.yaml` is a reusable workflow called by the DEV deploy workflow to open the promotion PR.
- **Per-environment deploys:** the same reusable deploy, called for dev, staging and prod with different `environment` inputs.
- **Interviews:** "reusable workflow vs composite action" and "how do you pass secrets to a reusable workflow?"

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `error parsing called workflow … workflow is not reusable as it is missing a "on.workflow_call" trigger` | The callee lacks `workflow_call`. | Add it. |
| `Invalid input, replicas is not defined in the referenced workflow` | You passed an undeclared input. | Declare it under `on.workflow_call.inputs`. |
| `Input required and not supplied: application_name` | Missing required input. | Add it in `with:`. |
| `Unexpected value 'runs-on'` / `'steps'` on the caller job | Caller jobs can't have runs-on or steps. | Put the steps in a separate job. |
| Secret empty inside the reusable workflow | Secrets aren't passed automatically. | `secrets: NAME: ${{ secrets.NAME }}` or `secrets: inherit`. |
| `Unable to find reusable workflow` / `… not allowed to be used` | Wrong path or ref, a subfolder under workflows, or a private repo without access. | `owner/repo/.github/workflows/file.yml@ref`. Enable **Access** in the source repo. |
| `needs.deploy-job.outputs.x` empty | `on.workflow_call.outputs` missing, or the inner job output isn't mapped. | Map all three layers: step → job → workflow. |
| Caller's `env:` value missing inside | `env` isn't propagated. | Pass it as an input. |
| `The nested reusable workflow depth exceeded` | Too many nested levels. | Flatten the design. |

---

## Quick recap

- Reusable workflow = `on: workflow_call`. Call it with `jobs.<id>.uses: path-or-repo@ref`.
- **Inputs** (string/boolean/number) via `with:` → `inputs.*`. **Outputs**: step → job → `on.workflow_call.outputs` → `needs.<caller-job>.outputs.*`.
- **Secrets** must be passed (`secrets:` map or `secrets: inherit`). Caller `env` doesn't pass.
- The caller holds app-specific logic, the callee holds shared standards. ≈ Jenkins Shared Libraries.
- Version them with tags or SHAs like actions.

**Interview questions**

1. **Q: Reusable workflow vs composite action?**
   A: A reusable workflow reuses whole **jobs** (own runners, environments, secrets). A composite action reuses **steps** inside a job on the caller's runner.
2. **Q: How does a caller read a value produced deep inside a reusable workflow?**
   A: Step output → job output in the callee → `on.workflow_call.outputs` → `needs.<caller-job>.outputs.<name>` in the caller.
3. **Q: Are secrets available inside a reusable workflow automatically?**
   A: No. Pass each one under `secrets:`, or use `secrets: inherit` (same org/enterprise).

---

### Original lecture conclusions (06 and 07)

**Lecture 06:** inputs are a foundational building block. They let workflows, reusable workflows and actions receive runtime values and adapt. The lecture covered **workflow inputs, reusable workflow inputs and action inputs** with production-style demos.

```text
Inputs provide data
        ↓
Workflows consume data
        ↓
Behavior changes dynamically
```

Inputs combine with **variables, contexts, expressions, functions and custom actions** to build increasingly reusable automation platforms.

**Lecture 07:** outputs move runtime values across execution boundaries without hard-coding or regenerating them: **step outputs** (steps), **job outputs** (jobs), **reusable workflow outputs** (workflows).

```text
One Execution Unit Creates a Value
                 ↓
Another Execution Unit Consumes the Value
```

```text
Step Output              → Available to Subsequent Steps
Job Output               → Available to Downstream Jobs
Reusable Workflow Output → Available to Calling Workflows
```

Outputs enable modular design, less duplication, better maintainability, and every stage working on the same runtime information.

> **Key takeaway:** the difference between the output types is not the value but **the execution boundary it must cross**: Step → Step (step output), Job → Job (job output), Workflow → Workflow (reusable workflow output).

Next: **artifacts**, for sharing files, reports, binaries, logs and other deliverables ([topic 24](24-artifacts.md)).

### References

- Lecture 06: [Actions docs](https://docs.github.com/en/actions) · [workflow_dispatch](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#workflow_dispatch) · [Reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) · [Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions) · [Functions](https://docs.github.com/en/actions/learn-github-actions/expressions#functions) · [Contexts](https://docs.github.com/en/actions/learn-github-actions/contexts) · [Variables](https://docs.github.com/en/actions/learn-github-actions/variables) · [Marketplace (actions)](https://github.com/marketplace?type=actions)
- Lecture 07: [Defining outputs for jobs](https://docs.github.com/actions/using-jobs/defining-outputs-for-jobs) · [Workflow syntax](https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions) · [Reusing workflows](https://docs.github.com/actions/using-workflows/reusing-workflows) · [Workflow commands](https://docs.github.com/actions/using-workflows/workflow-commands-for-github-actions) · [Contexts](https://docs.github.com/actions/learn-github-actions/contexts) · [Expressions](https://docs.github.com/actions/learn-github-actions/expressions)
