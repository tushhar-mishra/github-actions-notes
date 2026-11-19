# 33 · JavaScript Actions

> **Part F: Custom actions** · Previous: [32 · Custom and composite actions](32-custom-actions-and-composite.md) · Next: [34 · Docker actions](34-docker-actions.md) · [Back to INDEX](INDEX.md)
>
> **Video:** [GitHub JavaScript & Docker Actions Explained | Hands-on Demos](https://www.youtube.com/watch?v=sjSmSzMJL5s) (Demo 1)
>
> [![Watch the video](https://img.youtube.com/vi/sjSmSzMJL5s/maxresdefault.jpg)](https://www.youtube.com/watch?v=sjSmSzMJL5s)
>
> **Code:** [`code/33-javascript-actions/`](code/33-javascript-actions/)

---

## What it is

A **JavaScript action** is a custom action whose logic is a **JavaScript (or TypeScript) program**, run by the **Node.js runtime** that comes with the runner application.

```yaml
runs:
  using: node24
  main: index.js
```

When GitHub sees this, it starts **Node.js 24** on the runner and runs the file named in **`main`**.

> **What is TypeScript?** TypeScript is a **superset of JavaScript** (every valid JavaScript program is valid TypeScript) that adds **static type checking, interfaces, classes, generics** and better **IDE support** (code completion, navigation, error detection). It's **compiled to plain JavaScript** before Node.js runs it. These notes use JavaScript for simplicity, but many production actions use TypeScript.

> **About runtimes:** every language needs an execution environment. **JavaScript, Python, Java and C#** use a **language runtime**. **Go, Rust and C++** usually compile into standalone executables that don't need one. That's why a JavaScript action must state its Node.js version.

Many of the most widely used actions are JavaScript actions: **`actions/checkout`, `actions/setup-node`, `actions/setup-java`, `actions/setup-python`, `actions/cache`, `actions/upload-artifact`, `actions/download-artifact`, `aws-actions/configure-aws-credentials`, `docker/login-action`**. Millions of developers use them every day without knowing how they work inside.

> **Mental model:**
>
> ```text
> Reuse Existing Workflow Steps → Composite Action
>
> Write New Automation Logic → JavaScript Action
> ```

---

## Why it exists

Composite actions **orchestrate** existing steps. Sometimes you need **real programming**: loops, functions, modules, error handling, libraries, JSON processing, and especially **calling GitHub's APIs**.

GitHub exposes **REST** and **GraphQL** APIs (GraphQL is a query-style API where you ask for exactly the fields you want) for **repositories, pull requests, issues, releases, workflows, branches** and more. A JavaScript action can read data, create or update resources, process responses and decide dynamically, which makes it good for:

```text
Read Workflow Inputs → Call GitHub REST API → Process Response → Generate Outputs
```

```text
Automatically Label Pull Requests → Generate Release Notes → Create GitHub Releases → Update Issues
```

> **Note:** many of these tasks **could** be done with shell scripts in a composite action. But as automation gets complex, a **programming language** is simpler, more structured and easier to extend. Both have their place. Choose by complexity.

> **Recommendation:** choose a JavaScript action for **custom programming logic, GitHub API interactions, dynamic data processing**, or anything that becomes awkward in shell.

> **Production insight:** JavaScript actions run **directly on the runner's Node.js**, so they're **lighter and faster to start than Docker actions** (no image to build or pull). They're common for GitHub-centric automation: PRs, issues, releases, repo settings, release notes, approval flows, internal developer platforms.

> **Key takeaway:** you **don't need to be a JavaScript developer** to use GitHub Actions well. Most DevOps engineers **consume** JavaScript actions. Understanding their structure lets you use, customize and, when needed, build them.

**Jenkins comparison:** a JavaScript action is like a small **Jenkins plugin**, but per-repo, versioned, and running only for the job that uses it.

---

## Analogy

A composite action is a **recipe card that tells the cook which appliances to use**. A JavaScript action is a **programmable kitchen robot**: you write its program once (`index.js`), it can **phone the supplier** (call the GitHub API), **make decisions** ("if the order has more than 10 items, use the big pot"), and **report problems** (`setFailed`). The kitchen already has the robot's power socket (Node.js on the runner). You just plug in your program.

---

## How it works inside

```text
step: uses: ./.github/actions/workflow-summary
  1. runner reads action.yml → runs.using: node24, main: index.js
  2. inputs from with: are exported as environment variables INPUT_<NAME>  (INPUT_NAME=Cloud With VarJosh)
  3. runner starts its BUNDLED Node 24: node /path/to/action/index.js
       (the bundled Node is separate from any Node you install with setup-node)
  4. index.js uses @actions/core / @actions/github:
       core.getInput('name')      → reads INPUT_NAME
       core.info()/warning()/error() → log lines and annotations
       core.setOutput('x', v)      → writes to $GITHUB_OUTPUT
       github.context              → parsed GITHUB_* env vars + event payload
       github.getOctokit(token)    → authenticated REST/GraphQL client
  5. exit code 0 = success; core.setFailed(msg) sets exit code 1 → step fails
  6. optional pre:/post: scripts run before/after (e.g. checkout's credential cleanup)
```

**Runtime history (why versions matter):**

| `using:` | Status (Nov 2026) |
|---|---|
| `node12` | Removed (2023) |
| `node16` | Removed (2024) |
| `node20` | **Deprecated Sep 2025, removed from runners 23 Sep 2026** |
| `node24` | **Current.** The default since June 2026 |

> ⚠️ Corrected: was `using: node20`, now **`using: node24`**, because Node 20 actions no longer run on GitHub-hosted runners. The temporary opt-out variable `ACTIONS_ALLOW_USE_UNSECURE_NODE_VERSION` no longer works. Self-hosted runners need runner **v2.327.1+** for Node 24, and **ARM32** self-hosted runners lost support (Node 24 has no official ARM32 build).

**Dependencies are NOT installed automatically.** For a **local** action you can `npm install` in an earlier step (as the demo does). For a **remote/published** action GitHub just downloads the repo, so you must ship dependencies **in the repo**: either commit `node_modules/` or, the standard way, **bundle** everything into one file (`dist/index.js`) with a tool like `@vercel/ncc` or `rollup`, and point `main:` at it.

---

## Workflow YAML / commands

### Demo 1: your first JavaScript action

Actions run application code on Node.js and use the **Actions Toolkit**, GitHub APIs and the workflow context.

#### Step 1: Repository

Repo and SSH as in [topic 02](02-git-and-github-recap.md). GitHub discovers workflows under `.github/workflows`, provisions a runner, downloads the repo and runs the steps (including local or remote actions).

#### Step 2: Directory structure

```text
.github
├── actions
│   └── workflow-summary
│       ├── action.yml
│       ├── index.js
│       └── package.json
└── workflows
    └── javascript-action-demo.yml
```

- Each sub-folder of `.github/actions` is **one local action** with its own `action.yml`.
- Unlike composite actions, JavaScript actions contain **source files**. Here everything is in **`index.js`**, the **entry point**.

  > **Note:** these notes use `index.js`. Production actions often use a compiled file like **`dist/index.js`**. The concept is the same.

- **`package.json`** describes the project and its libraries.

  > **Note:** very simple JavaScript actions can run without `package.json`, but including one is **standard practice** and matches what you'll see in production and open-source actions.

- Production actions often also have **`src/`** (source), **`dist/`** (compiled JavaScript that GitHub runs), extra modules (`utils.js`, `helpers.js`, `logger.js`), **`README.md`**, **`.gitignore`** and **`LICENSE`**.

> **Note:** however many files there are, GitHub **always starts from the entry point** in `action.yml`.
>
> **Note:** the focus is how GitHub runs a JavaScript action, not learning JavaScript. The code is deliberately simple.

#### Step 3: `package.json`

```json
{
  "name": "workflow-summary-action",
  "version": "1.0.0",
  "description": "Simple GitHub JavaScript Action",
  "main": "index.js",
  "dependencies": {
    "@actions/core": "^1.11.1",
    "@actions/github": "^6.0.1"
  }
}
```

A JavaScript action is a **Node.js project**:

- **`name`**: identifies the project.
- **`version`**: the project version.
- **`description`**: a short summary.
- **`main`**: the entry point (`index.js`).
- **`dependencies`**: external libraries. `^1.11.1` means "1.11.1 or any newer 1.x" (**semver** caret range).

Both libraries come from the **GitHub Actions Toolkit** ([`actions/toolkit`](https://github.com/actions/toolkit)), GitHub's collection of libraries for reading inputs, setting outputs, logging, reading the workflow context and calling the GitHub API:

- **`@actions/core`**: read inputs, write logs, set outputs, mark failure.
- **`@actions/github`**: the **workflow context** plus an authenticated **REST/GraphQL client** (Octokit).

> **Note:** `package.json` only **declares** dependencies. The workflow runs **`npm install`** later.
>
> **Production insight:** most production JavaScript actions use toolkit libraries instead of reimplementing common features.
>
> 🆕 Check the toolkit's current major versions before starting a new action (`npm view @actions/core version`). Newer majors may change the module format (CommonJS `require` vs ESM `import`).

#### Step 4: `index.js`

```javascript
const core = require("@actions/core");
const github = require("@actions/github");

try {

    const name = core.getInput("name");

    core.info("========================================");
    core.info("GitHub JavaScript Action");
    core.info("========================================");

    core.info("");
    core.info(`Hello ${name}!`);
    core.info("");

    core.info("Workflow Information");
    core.info("----------------------------------------");
    core.info(`Repository : ${github.context.repo.owner}/${github.context.repo.repo}`);
    core.info(`Workflow   : ${github.context.workflow}`);
    core.info(`Actor      : ${github.context.actor}`);
    core.info(`Event      : ${github.context.eventName}`);
    core.info(`Branch     : ${github.context.ref}`);
    core.info(`Run Number : ${github.context.runNumber}`);
    core.info(`SHA        : ${github.context.sha}`);

}
catch (error) {
    core.setFailed(error.message);
}
```

Focus on **how it gets inputs, uses the toolkit, reads the context and handles failures**, not on the syntax.

- **Imports:** `const` declares a variable (`core`, `github` are just names). `require()` imports a library so its functions are available.
- **`core.getInput("name")`**: reads the `name` input from the caller. It's the toolkit equivalent of `${{ inputs.name }}` in composite actions.
- **`core.info(...)`**: writes informational lines to the workflow log.
- **`github.context`**: run metadata: repository, workflow, actor, event, branch (`ref`), run number, commit SHA. These are the same values as `${{ github.actor }}` / `${{ github.repository }}` in YAML, but accessed **programmatically**.
- **`try { … } catch (error) { core.setFailed(error.message); }`**: if anything throws, `setFailed` **marks the action failed** and shows the message, a standard production pattern.

> **Key takeaway:** understand **how a JavaScript action is organized, uses the toolkit, receives inputs, writes logs, reads the context and handles failures**. More advanced actions build on exactly this.

#### Step 5: `action.yml` and the calling workflow

[`action.yml`](code/33-javascript-actions/.github/actions/workflow-summary/action.yml):

```yaml
name: Workflow Summary
description: Display information about the current GitHub Actions workflow.

inputs:
  name:
    description: Name of the user.
    required: true

runs:
  using: node24
  main: index.js
```

[`.github/workflows/javascript-action-demo.yml`](code/33-javascript-actions/.github/workflows/javascript-action-demo.yml):

```yaml
name: JavaScript Action Demo

on:
  workflow_dispatch:

jobs:
  demo-job:
    runs-on: ubuntu-latest

    steps:

      - name: Checkout Repository
        uses: actions/checkout@v7

      - name: Install Action Dependencies
        working-directory: ./.github/actions/workflow-summary
        run: npm install

      - name: Execute JavaScript Action
        uses: ./.github/actions/workflow-summary
        with:
          name: Cloud With VarJosh
```

> **Note:** declaring dependencies in `package.json` **doesn't install them**. At first the action has only source code and declarations:
>
> ```text
> workflow-summary/
> ├── action.yml
> ├── index.js
> └── package.json          ← no node_modules/ yet
> ```
>
> `npm install` (run inside the action folder via `working-directory`, topic 14) downloads them into `node_modules/` **on this runner**, just before the action runs. (This works for **local** actions only. Published actions must bundle their dependencies, see "How it works inside".)

> **Observation:** to the caller, invoking a JavaScript action looks **exactly like** invoking a composite or remote action. Implementation details stay inside the action.

#### Steps 6–7: Commit, push, run

```bash
git add .
git commit -m "feat: add javascript workflow-summary action"
git push
```

**Actions → JavaScript Action Demo → Run workflow.**

#### Step 8: Observe (sample log output)

```text
✓ Install Action Dependencies
    added 25 packages, and audited 26 packages in 2s
✓ Execute JavaScript Action
    Run ./.github/actions/workflow-summary
      with:
        name: Cloud With VarJosh
    ========================================
    GitHub JavaScript Action
    ========================================

    Hello Cloud With VarJosh!

    Workflow Information
    ----------------------------------------
    Repository : your-user/gha-practice-demo
    Workflow   : JavaScript Action Demo
    Actor      : your-user
    Event      : workflow_dispatch
    Branch     : refs/heads/main
    Run Number : 3
    SHA        : 4dbbbfe44d5e2a4d…
```

> **Observation:** to the workflow, a JavaScript action is just another `uses:`. That keeps workflows simple and focused on orchestration.

> **Exercise:** change the `name` value, commit and run again. You'll see the new value read by `core.getInput("name")`, which shows how data flows from the **workflow** into a **JavaScript action**.

### 🆕 Going further: outputs, the GitHub API, and a job summary

```javascript
const core = require("@actions/core");
const github = require("@actions/github");

async function run() {
  try {
    const token = core.getInput("github-token", { required: true });
    const octokit = github.getOctokit(token);
    const { owner, repo } = github.context.repo;

    const { data: prs } = await octokit.rest.pulls.list({ owner, repo, state: "open" });
    core.setOutput("open-prs", String(prs.length));          // → steps.<id>.outputs.open-prs

    await core.summary                                         // → $GITHUB_STEP_SUMMARY (topic 23)
      .addHeading("Open pull requests")
      .addTable([["#", "Title"], ...prs.map(p => [String(p.number), p.title])])
      .write();

    if (prs.length > 20) core.warning("Too many open PRs");   // yellow annotation
  } catch (err) {
    core.setFailed(err.message);
  }
}
run();
```

Add to `action.yml` (`inputs: github-token: {required: true}`, `outputs: open-prs: {description: …}`), and call it with `with: github-token: ${{ secrets.GITHUB_TOKEN }}` plus `permissions: pull-requests: read` (topic 18).

---

## Hands-on lab

1. Build the demo and match the log.
2. Remove the `npm install` step and read `Error: Cannot find module '@actions/core'`.
3. Change `using: node24` to `using: node20` and read the error. That's why old actions break.
4. Add the "going further" code and see the summary table on the run page.
5. Bundle it: `npx @vercel/ncc build index.js -o dist`, set `main: dist/index.js`, delete the `npm install` step, and it still works.

---

## Where you'll use it

- **GitHub automation:** labelling, PR comments, release notes, issue triage, repo settings.
- **Internal developer platform** actions that call internal APIs.
- **Reading** the source of popular actions (checkout, cache) when debugging their behaviour.
- **Audits:** upgrading every action off removed Node runtimes (the Sep 2026 Node 20 removal).

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Error: Cannot find module '@actions/core'` | Dependencies not installed or bundled. | `npm install` first (local), or bundle into `dist/` and commit it. |
| `The 'node20' runtime is no longer supported` / the action fails to start | Removed runtime. | `using: node24`. Upgrade third-party actions to their latest major. |
| `Input required and not supplied: name` | `required: true` input missing from `with:`. | Pass it. |
| `HttpError: Resource not accessible by integration` | The token lacks permissions. | Add `permissions:` (topic 18). |
| Step passes although the logic failed | Errors swallowed without `core.setFailed`. | Always `setFailed` in `catch`, and `await` async calls. |
| `SyntaxError: Cannot use import statement outside a module` | ESM vs CommonJS mismatch. | Use `require`, or `"type": "module"` in package.json with `import`. |

---

## Quick recap

- JavaScript action = `action.yml` with `runs.using: node24` + `main: <file>`, run by the runner's **bundled Node.js**.
- Toolkit: `@actions/core` (inputs, logs, outputs, failure, summary), `@actions/github` (context + Octokit API client).
- Inputs arrive as `INPUT_*` env vars. `core.setFailed` fails the step.
- Dependencies: `npm install` (local demo) or **bundle into `dist/`** (published).
- `node20` is gone (Sep 2026). Use `node24`.

**Interview questions**

1. **Q: When is a JavaScript action better than a composite action?**
   A: When you need real programming logic or GitHub API calls (loops, JSON, error handling) that would be messy in shell.
2. **Q: Why do JavaScript actions start faster than Docker actions?**
   A: They run on the runner's existing Node.js, with no image to build or pull.
3. **Q: A JavaScript action works locally but fails when used from another repo with "cannot find module". Why?**
   A: Remote actions aren't `npm install`-ed. Dependencies must be committed or bundled (e.g. with ncc into `dist/index.js`).
