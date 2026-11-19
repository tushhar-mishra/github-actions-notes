# 08 · Your First Workflow, and What Happens After `git push`

> **Part A: Foundations** · Previous: [07 · Runners](07-runners.md) · Next: [09 · Workflow triggers](09-workflow-triggers.md) · [Back to INDEX](INDEX.md)
>
> **Video:** [What is GitHub Actions | Build Your First Workflow from Scratch](https://www.youtube.com/watch?v=w4c_NIjO3XI) (demo section)

---

## What it is

This is the first real demo. You'll write a workflow, push it, and watch it run. Then you'll follow **every internal step** GitHub takes between your `git push` and the green tick.

The goal of this demo is **not** a production pipeline yet. It's to understand:

- repository setup
- GitHub authentication
- where workflow files go
- workflow syntax
- how jobs and steps run
- the GitHub Actions execution flow

The workflow logic is kept simple on purpose, so you can focus on the architecture and execution model.

---

## Why it exists

Most "my workflow didn't run" problems come from not knowing what GitHub does behind the scenes. If you know the chain (**event → match → queue → runner → steps → result**), you can tell exactly which link broke.

---

## Analogy

Posting a letter:

1. You drop it in the post box (**git push**).
2. The post office stamps it with the time and type (**event recorded**).
3. They read the address (**which workflows listen to `push`?**).
4. They put it in the sorting queue (**jobs queued**).
5. A free postman with the right route picks it up (**runner whose labels match**).
6. The postman delivers it step by step (**steps run**).
7. You get a delivery receipt (**logs, artifacts, ✅/❌ status**).

---

## How it works inside

### The full trace after `git push`

```text
① YOU: git push origin main
       │
② GITHUB receives the new commits on branch main.
   It records a "push" EVENT with a payload (a JSON document): who pushed, which branch,
   before/after commit SHAs, the list of commits. (SHA = the 40-character ID of a commit.)
       │
③ WORKFLOW MATCHING
   GitHub reads every file in .github/workflows/ — AT THE PUSHED COMMIT (not the default branch).
   For each file: parse YAML → validate → does on: include "push"?
   Do branch/path filters (topic 10) match? → if yes, the workflow is MATCHED.
       │
④ WORKFLOW RUN CREATED
   One "run" per matched workflow, numbered #1, #2… It gets GITHUB_SHA = pushed commit,
   GITHUB_REF = refs/heads/main, and a fresh GITHUB_TOKEN (topic 18).
       │
⑤ JOB GRAPH BUILT
   Jobs with no needs: are ready now. Jobs with needs: wait.
   Job-level if: conditions are evaluated here (before any runner is involved).
       │
⑥ JOBS QUEUED
   Each ready job enters the queue with its runs-on labels.
       │
⑦ RUNNER PICKED
   Hosted label → GitHub boots a FRESH VM from the ubuntu-26.04 (what ubuntu-latest means since Nov 2026) image.
   Self-hosted label → an idle registered runner with matching labels takes it.
       │
⑧ "Set up job" on the runner
   Runner app downloads the job definition, prints runner/image info,
   downloads every action used in uses: (e.g. actions/checkout), prepares the workspace
   /home/runner/work/<repo>/<repo>. NOTE: your code is NOT there yet — checkout does that.
       │
⑨ STEPS RUN in order, each as a new shell process (topic 04).
   Output streams live to the Actions tab. Secrets are masked as ***.
   A non-zero exit code fails the step → remaining steps skipped.
       │
⑩ "Post" steps and "Complete job": actions clean up (post-checkout etc.),
   the runner reports the job result, the hosted VM is destroyed.
       │
⑪ RESULTS: job status → dependent jobs (needs) start or are skipped →
   workflow status ✅/❌ → commit gets a status check (visible on the commit/PR) →
   logs kept (default 90 days), artifacts uploaded if any (topic 24),
   notifications sent (e.g. email on failure).
```

Key facts this trace explains:

- **Your code is not on the runner** until a step runs `actions/checkout`. A fresh VM is empty.
- The workflow file used is **the one in the commit you pushed**, so a workflow on a feature branch runs that branch's version.
- **Jobs are separate VMs**, so `job-1` and `job-2` don't share files.
- YAML errors fail at step ③, so no runner time is used.

---

## Workflow YAML / commands

### Step 1: Repository and authentication

Do the setup from [topic 02](02-git-and-github-recap.md): a private repo `cwvj-gha-practice` (no README, `.gitignore` or license), an SSH key added to GitHub, and the SSH URL copied, e.g. `git@github.com:<YOUR_GITHUB_USERNAME>/cwvj-gha-practice.git`.

> Private repos are used because most enterprise apps live in private repos, so GitHub Actions behaves the way it does in real organizations.

### Step 2: Create the workflow structure

Workflows **must** be under `.github/workflows/`.

```bash
# Create project directory
mkdir project-files

# Move into the project directory
cd project-files

# Initialize local Git repository
git init

# Create GitHub Actions workflow directory structure
mkdir -p .github/workflows

# Create workflow file
touch .github/workflows/my-workflow.yaml
```

Put this in `my-workflow.yaml`:

```yaml
name: My First Workflow
on: push
jobs:
  job-1:
    runs-on: ubuntu-latest
    steps:
      - name: Step 1
        run: echo "Hello from Step 1"
      - name: Step 2
        run: echo "Hello from Step 2"
  job-2:
    runs-on: ubuntu-latest
    steps:
      - name: Final Step
        run: echo "Hello from Job 2"
```

- `name`: the workflow name shown in the Actions UI.
- `on: push`: run whenever code is pushed.
- `jobs`: the units of work in this workflow.
- `runs-on`: which runner OS runs each job.
- `steps`: the operations in each job, run in order.
- `run`: a shell command on the runner.

Execution behaviour:

- `job-1` and `job-2` run on **separate runners**.
- Jobs run **in parallel by default** (there's no `needs` here).
- Steps inside a job run **in the order written**.

### Step 3: Push to GitHub

```bash
# Create a sample README file
touch README.md

# Add all files to the Git staging area
git add .

# Create the first commit
git commit -m "ci: add initial GitHub Actions workflow"

# Rename the default branch to main
git branch -M main

# Configure GitHub repository as remote origin
git remote add origin git@github.com:<YOUR_GITHUB_USERNAME>/cwvj-gha-practice.git

# Push code to GitHub
git push -u origin main
```

When the push succeeds:

- GitHub receives the workflow file in `.github/workflows/`.
- The `push` event triggers the workflow automatically.
- GitHub provisions runners and starts the jobs.

It also proves that SSH authentication works, GitHub trusts your machine, the remote is configured, and workflow triggering works.

### Step 4: Verify the run on GitHub

Go to **Repository → Actions**. You'll see a run of **My First Workflow**. Open it to see:

- the run status
- the individual jobs
- the steps inside each job
- the runner OS
- the logs

You should notice:

- `job-1` and `job-2` running **independently**
- steps inside each job running **in order**
- GitHub provisioning runners **automatically**
- logs **streaming in real time**

Click a job or step to inspect the shell commands, step output, duration, execution order, and failure/debug information.

> The **Actions tab is your central window into CI/CD**. You'll spend a lot of time here.

**Sample log output (job-1):**

```text
▶ Set up job
    Current runner version: '2.3xx.x'
    Operating System: Ubuntu 26.04 LTS
    Runner Image: ubuntu-26.04
    GITHUB_TOKEN Permissions: Contents: read, Metadata: read ...
    Secret source: Actions
    Complete job name: job-1
▶ Step 1
    Run echo "Hello from Step 1"
    Hello from Step 1
▶ Step 2
    Run echo "Hello from Step 2"
    Hello from Step 2
▶ Complete job
    Cleaning up orphan processes
```

| Log line | Maps to trace step |
|---|---|
| `Current runner version`, `Runner Image` | ⑦–⑧ a fresh hosted VM was picked and prepared |
| `GITHUB_TOKEN Permissions` | ④ the automatic token for this run (topic 18) |
| `Run echo …` / `Hello from Step 1` | ⑨ the step's command and its output |
| `Cleaning up orphan processes` | ⑩ the job finished. The VM is about to be destroyed |

### Bonus step: trigger it again

Add a step to `job-2`:

```yaml
- name: Another Step
  run: echo "Workflow executed again"
```

Updated `job-2`:

```yaml
job-2:
  runs-on: ubuntu-latest
  steps:
    - name: Final Step
      run: echo "Hello from Job 2"
    - name: Another Step
      run: echo "Workflow executed again"
```

```bash
git add .
git commit -m "workflow: add another step to job-2"
git push
```

After the push:

- GitHub triggers the workflow **again**.
- A **new run** (#2) appears in the Actions tab.
- The new step shows up in `job-2`.

That's because the trigger is `on: push`, which means **every push** starts the workflow.

---

## Hands-on lab

1. Do Steps 1–4 above and find each trace step (②–⑪) in the UI: the commit, the run number, the queued state (refresh quickly), the "Set up job" section, the step logs, and the status tick next to the commit on the repo page.
2. Add `- run: ls -la` as the first step. It lists **nothing** except hidden folders, which proves the code isn't on the runner yet.
3. Add `- uses: actions/checkout@v7` before it and push again. Now your files appear.
4. Break the YAML (remove a space after a colon), push, and watch the run fail at step ③ with **"Invalid workflow file"**. Note that no job ever started.

---

## Where you'll use it

- Debugging any "didn't trigger" / "stuck" / "file not found" problem: find which trace step failed.
- **Interview classic:** "What happens when you push code to a repo with GitHub Actions?" Answer with the trace above.

---

## When it breaks

| Symptom | Trace step | Meaning and fix |
|---|---|---|
| No run appears at all | ③ | File not in `.github/workflows/`, `on:` doesn't include this event, or a branch/path filter excluded it (topic 10). Also check Actions is enabled in **Settings → Actions**. |
| Run appears with ❌ **Invalid workflow file** | ③ | YAML/schema error. The message gives the line number (topic 03). |
| Job stuck **Queued** | ⑥–⑦ | No matching runner (self-hosted offline, wrong label) or concurrency/plan limits (topics 07, 29, 38). |
| `No such file or directory` for your source files | ⑧ | You forgot `actions/checkout`. |
| `Error: Process completed with exit code 1.` | ⑨ | A command failed. Read the lines above it. |
| Pushed but an *old* version of the workflow ran | ③ | The workflow comes from the pushed commit. Did you push the branch you edited? |
| Workflow file is on a feature branch but `workflow_dispatch` button isn't shown | ③ | Manual and some other triggers only appear once the workflow file exists on the **default branch** (topic 09). |

---

## Quick recap

- push → event → workflows matched (from the pushed commit) → run created → jobs queued → runner picked → fresh VM → steps → results.
- The runner starts **empty**: use `actions/checkout` to get your code.
- Every push with `on: push` creates a **new numbered run**.
- The Actions tab shows status, jobs, steps and live logs.

**Interview questions**

1. **Q: Walk me through what happens after `git push`.**
   A: GitHub records a push event, finds workflows in `.github/workflows` at that commit whose `on:` and filters match, creates a run, queues the jobs, assigns each to a runner matching `runs-on` (a fresh VM for hosted), runs the steps in order with live logs, then reports status checks, keeps logs and artifacts, and destroys the VM.
2. **Q: Why does `ls` show no files at the start of a job?**
   A: The runner is a fresh machine. The repo is only cloned when `actions/checkout` runs.
3. **Q: Does a YAML syntax error use runner minutes?**
   A: No. Validation happens before any job is queued.

---

### References

- [GitHub Actions documentation](https://docs.github.com/actions)
- [GitHub-hosted runners](https://docs.github.com/actions/using-github-hosted-runners/about-github-hosted-runners)
- [Self-hosted runners](https://docs.github.com/actions/hosting-your-own-runners)
- [Workflow syntax](https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions)
- [Events that trigger workflows](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows)
