# 55 · Interview Bank and the GitHub Actions ↔ Jenkins Map

> **Part I: Review** · Previous: [54 · Capstone 12: end-to-end validation](54-capstone-12-end-to-end-validation.md) · [Back to INDEX](INDEX.md)
>
> 🆕 New topic. Part 1 maps every GitHub Actions concept to its Jenkins equivalent (useful because you already have Jenkins notes). Part 2 is a **revision drill**: one-line answers to the questions interviewers ask most. Part 3 lists **every topic's interview questions** with links to the answers in each topic.

---

## What it is

A single page to revise from before an interview or an on-call shift.

## Why it exists

Interviewers jump between topics ("how do runners work?" → "how do you secure secrets?" → "design a pipeline"). Revising one page beats re-reading 54 files. Mapping to Jenkins lets you explain GitHub Actions in terms an interviewer with a Jenkins background will recognise, and the other way round.

## Analogy

The **index cards** you make before an exam: one card per question, the answer in your own words, and a "translation" card for the other language you know (Jenkins).

## How it works inside

Use it in three passes: (1) read the Jenkins map and say each row aloud, (2) cover the answers in the drill and answer from memory, (3) for any miss, jump to the linked topic.

---

## Workflow YAML / commands

### Part 1: GitHub Actions ↔ Jenkins map

| GitHub Actions | Jenkins | Notes | Topic |
|---|---|---|---|
| Workflow file `.github/workflows/*.yml` | `Jenkinsfile` | YAML data vs Groovy code | 06 |
| Event trigger `on:` | Build triggers (webhook, SCM polling, cron, upstream) | Built in, no webhook setup | 09 |
| `workflow_dispatch` + inputs | "This project is parameterized" | | 09, 20 |
| `schedule` (cron, UTC) | "Build periodically" | | 09 |
| `repository_dispatch` | "Trigger builds remotely" | | 09 |
| Job | Stage (+ its own agent) | Jobs are parallel by default | 06 |
| Step | `steps { sh … }` | | 06 |
| `needs:` | Stage order / `parallel` | | 06, 13 |
| `if:` | `when { … }` | | 13 |
| `if: failure()` / `always()` | `post { failure { } always { } }` | | 12 |
| Runner | Agent / node | Hosted runners need no servers | 07 |
| `runs-on:` labels | `agent { label '…' }` | | 07, 38 |
| Self-hosted runner + ARC | Agent + Kubernetes plugin | | 38 |
| Action (`uses:`) | Plugin / shared-library step | Versioned per workflow | 15 |
| Composite / JS / Docker action | Shared-library step / plugin | | 32–34 |
| Reusable workflow (`workflow_call`) | Shared Library pipeline | | 22 |
| `env:`, `vars.*` | `environment { }`, global properties | | 11 |
| Contexts (`github.*`) | `env.BRANCH_NAME`, `currentBuild` | Structured vs flat | 11 |
| `secrets.*` | Credentials + `withCredentials` | | 17 |
| `GITHUB_TOKEN` + `permissions:` | GitHub PAT/App credential | Automatic, per run | 18 |
| OIDC (`id-token: write`) | Instance profile / Vault plugin | No stored keys | 36 |
| Step/job outputs | Groovy variables between stages | Jobs run on separate machines | 21 |
| Artifacts | `archiveArtifacts` / `stash` | | 24 |
| Cache (`actions/cache`) | Persistent workspace / `~/.m2` on the agent | Explicit on ephemeral runners | 25 |
| Matrix strategy | `matrix { axes { } }` | | 26 |
| `services:` / `container:` | `docker.image().withRun` / `agent { docker }` | | 27 |
| Environments (approvals) | `input` step + folder credentials + Lockable Resources | | 28 |
| `concurrency:` | `disableConcurrentBuilds()` / `lock()` | | 29 |
| Rulesets + required checks | GitHub branch protection fed by the Jenkins status | | 30 |
| `workflow_run` | "Build after other projects" / `upstream()` | | 31 |
| Re-run with debug logging | Replay + `set -x` | | 39 |
| `actionlint` / `act` | Pipeline linter / (no local runner) | | 39 |

### Part 2: The 30-second answers (revision drill)

| # | Question | Short answer | Topic |
|---|---|---|---|
| 1 | CI vs Continuous Delivery vs Deployment? | CI = merge and test often. Delivery = always releasable with a manual prod gate. Deployment = automatic to prod. | 01 |
| 2 | What happens after `git push`? | Event → workflows matched (from that commit) → run → jobs queued → runner by label → fresh VM → steps → logs, status, artifacts. | 08 |
| 3 | Hosted vs self-hosted runners? | Hosted: GitHub-managed ephemeral VMs. Self-hosted: yours (private network, GPUs), persistent unless ephemeral, never on public repos. | 07, 38 |
| 4 | Job stuck in Queued? | Labels, runner offline, group access, concurrency, plan limits. | 07, 38 |
| 5 | Workflow didn't trigger? | Path/branch filters, wrong folder, not on the default branch (schedule/workflow_run/dispatch), Actions disabled. | 10, 31 |
| 6 | `$VAR` vs `${{ }}`? | The shell expands `$VAR` on the runner. GitHub substitutes `${{ }}` before the step runs (and in job-level fields). | 11 |
| 7 | Pass a value job → job? | `$GITHUB_OUTPUT` → `jobs.<id>.outputs` → `needs.<id>.outputs`. | 21 |
| 8 | Files job → job? | Artifacts (upload/download). | 24 |
| 9 | Speed up installs? | `actions/cache` or setup-* `cache:` with `hashFiles` keys. | 25 |
| 10 | Test on many OS/versions? | `strategy.matrix` (include/exclude, fail-fast, max-parallel). | 26 |
| 11 | Reusable workflow vs composite action? | Reuses jobs (own runners, secrets, environments) vs reuses steps inside one job. | 22, 32 |
| 12 | Secrets in fork PRs? | Not passed. `GITHUB_TOKEN` is read-only. Don't use `pull_request_target` with PR code. | 17, 37 |
| 13 | Least privilege for `GITHUB_TOKEN`? | Workflow `permissions: contents: read`, grant more per job. Listing any scope sets the rest to none. | 18 |
| 14 | Deploy to AWS without keys? | OIDC: `id-token: write` + an IAM role trusting `repo:…:environment:prod` + configure-aws-credentials. | 36 |
| 15 | Manual approval before prod? | Environment with required reviewers (+ prevent self-review), `environment: prod`. | 28 |
| 16 | Avoid overlapping deploys? | `concurrency: group: deploy-prod, cancel-in-progress: false` (or `queue: max`). | 29 |
| 17 | Cancel outdated PR runs? | `group: ${{ github.workflow }}-${{ github.head_ref \|\| github.ref }}`, `cancel-in-progress: true`. | 29 |
| 18 | Enforce CI before merge? | A ruleset with a required PR, approvals and required status checks. | 30 |
| 19 | Run CD after CI? | `workflow_run` (on the default branch) + check `conclusion` + check out `head_sha`. | 31 |
| 20 | Script injection? | Untrusted `${{ github.event.* }}` inlined in `run:`. Pass it via `env:`. | 37 |
| 21 | Why pin SHAs? | Tags are mutable (trivy-action 2026, tj-actions 2025). SHAs are immutable. Dependabot updates them. | 15, 37 |
| 22 | `set-output` is deprecated, so what now? | `echo "k=v" >> "$GITHUB_OUTPUT"`. | 21, 23 |
| 23 | Node 20 actions? | Removed Sep 2026. Use `using: node24` / the latest majors. | 33 |
| 24 | Composite vs JS vs Docker action? | Reuse steps / custom logic and API / own runtime (Linux only, slower). | 34 |
| 25 | Debug a failing job? | Locate the stage, re-run with debug logging, `toJSON` dumps, `set -x`, actionlint, act. | 39 |
| 26 | Reduce CI cost? | Filters, concurrency cancel, caching, smaller PR matrix, Linux/arm64, timeouts, short retention. | 41 |
| 27 | Prove an image came from your CI? | Build provenance attestation + `gh attestation verify`. | 40 |
| 28 | Integration tests with a DB? | `services: postgres` + a health check (localhost+ports, or the service name in a container job). | 27 |
| 29 | Build once, deploy many? | Build and scan once, promote the same image digest to every environment. | 28, 51, 53 |
| 30 | Design a production pipeline? | The capstone flow: scan → build/test → quality gate → image scan → registry (OIDC, immutable) → DEV → PR → review → prod approval → prod. | 42–54 |

### Part 2b: 🆕 Production scenarios ("tell me about a time…")

Senior interviews rarely ask "what is a matrix?". They describe an incident and watch **how you think**. Answer each one in the same 4 beats: **Symptom → How I narrowed it down → Root cause → Fix plus the guardrail that stops it coming back.** The scenarios below are realistic composites built from the failure modes in the topics. Practice them aloud, then swap in your own lab experience.

**S1. "Production got deployed twice, and the older commit won."**
- **Narrow down:** Two runs of the deploy workflow overlapped, because two merges landed 30 s apart. The older run finished last.
- **Root cause:** There was no `concurrency` on the deploy job, so both runs executed in parallel.
- **Fix and guardrail:** `concurrency: { group: deploy-prod, cancel-in-progress: false }` so deploys **queue** instead of racing. Use `queue: max` if every commit must deploy in order. Deploy an **immutable image digest**, and have the deploy step refuse to roll back to a commit older than the current one. (Topics 29, 53)

**S2. "Our dependency cache never hits. Every run takes 8 minutes."**
- **Narrow down:** The `Post Cache` log said `Cache saved with key …` on every run, and the keys differed each time.
- **Root cause:** The key included `${{ github.sha }}` (or `hashFiles` matched a lockfile rewritten during the build), so it was unique on every commit. A second cause: PR branches can't read caches created on *other* feature branches, only their own and the base branch's.
- **Fix and guardrail:** Key = `os + hashFiles('**/package-lock.json')`, plus `restore-keys` for partial hits. Warm the cache with a run on `main`. Watch hit rate in the cache tab. (Topic 25)

**S3. "A contributor's fork PR exfiltrated our secrets."**
- **Narrow down:** The audit log showed a run triggered by `pull_request_target` that checked out `github.event.pull_request.head.sha` and ran `npm install`.
- **Root cause:** A "pwn request". Privileged trigger plus untrusted code meant the attacker's `postinstall` script read the env and the token.
- **Fix and guardrail:** Rotate every secret **first**. Build fork PRs under `pull_request` (no secrets), and do privileged follow-ups in `workflow_run` on **data only**. Upgrade to `checkout@v7`, which refuses this by default. Use CODEOWNERS on `.github/workflows/` and `permissions: contents: read` as the default. (Topics 17, 31, 37)

**S4. "CI broke overnight and nobody changed anything."**
- **Narrow down:** The last green run and the first red run had the same commit. Diffing the `Set up job` logs showed `Runner Image: ubuntu-24.04` → `ubuntu-26.04`.
- **Root cause:** `ubuntu-latest` moved to a new OS image (Oct–Nov 2026), and a pre-installed tool changed version. (The same pattern happens when an action's **major tag** moves.)
- **Fix and guardrail:** Pin `runs-on: ubuntu-26.04` (or 24.04 temporarily) and pin actions by SHA with Dependabot PRs, so every upgrade arrives as a reviewed PR. (Topics 07, 37)

**S5. "The production deploy job sat 'Waiting' for two days, then failed."**
- **Narrow down:** The job was waiting on the `production` environment's required reviewers, and the only reviewer was on leave.
- **Root cause:** A single-person approval gate. Pending runs also count toward the 35-day workflow limit and block the concurrency group.
- **Fix and guardrail:** Use a reviewer **team** (at least 2 people), "prevent self-review", a deployment branch policy, and alerting on pending approvals. (Topics 28, 29)

**S6. "Our self-hosted runner leaked data between builds."**
- **Narrow down:** A job found files and a Docker login from a previous job in the workspace and in `~/.docker/config.json`.
- **Root cause:** Persistent (non-ephemeral) self-hosted runners reuse the machine, so state survives between jobs.
- **Fix and guardrail:** Use **ephemeral** runners (`--ephemeral` or ARC). Put them in runner groups limited to trusted repos, never on public repos. Use OIDC instead of stored credentials. (Topic 38)

**S7. "The required check is stuck on 'Expected' and nobody can merge."**
- **Narrow down:** The PR only touched `docs/`. The CI workflow has `paths: ['src/**']`, so it never ran, and the required check never reported.
- **Root cause:** A path filter on a workflow that produces a **required** check.
- **Fix and guardrail:** Remove the workflow-level path filter and skip the expensive *steps* with a changed-files check or a job-level `if:`. A skipped job reports success, so the check still appears. (Topics 10, 13, 30)

**S8. "The CD workflow didn't start after CI succeeded."**
- **Narrow down:** The `workflow_run` workflow existed only on the feature branch, and the CI workflow's `name:` had been renamed.
- **Root cause:** `workflow_run` is read **from the default branch** and matches the CI workflow **by name**.
- **Fix and guardrail:** Merge CD to `main` first, keep workflow names stable, and check `github.event.workflow_run.conclusion == 'success'`. (Topic 31)

**S9. "Our Actions bill tripled this month."**
- **Narrow down:** The usage report showed one repo running a 24-job matrix on macOS on every push to every branch, plus artifacts kept for 90 days.
- **Root cause:** No filters, no cancellation, and the most expensive OS used for the full matrix.
- **Fix and guardrail:** A small Linux matrix on PRs and the full matrix only on `main` or nightly. Add `cancel-in-progress` for PRs, `timeout-minutes`, path filters (not on required checks, see S7), short artifact retention, and **budgets** with alerts. (Topics 26, 29, 41)

**S10. "A third-party action we use was compromised."**
- **Narrow down:** A security advisory (like trivy-action in March 2026) said version tags were re-pointed to malicious commits.
- **Root cause:** Workflows referenced `@v1`-style **mutable tags**.
- **Fix and guardrail:** Search the org for the action, check run logs for the malicious SHA in the window, and **rotate any secrets** those jobs could reach. Pin every action to a full SHA. Use an allowed-actions policy, Dependabot, and least-privilege `permissions:`, so the blast radius is small next time. (Topics 15, 37, 30)

### Part 3: Every topic's interview questions (with answers in each topic)

Each topic ends with **Quick recap → Interview questions**, and the answers live there. The table lists the questions so you can quiz yourself, then click through to check your answer. **162 questions across 54 topics.**

| Topic (opens its Quick recap) | Qs | Questions |
|---|---:|---|
| [01 · What is CI/CD?](01-what-is-ci-cd.md#quick-recap) | 3 | What's the difference between Continuous Delivery and Continuous Deployment? · Does a CI/CD tool compile your code? · Why add a Quality Gate? |
| [02 · Git and GitHub Recap (Everything GitHub Actions Needs)](02-git-and-github-recap.md#quick-recap) | 3 | Why doesn't GitHub Actions need a webhook like Jenkins does? · What does `git push -u` do? · Which part of an SSH key pair do you upload to GitHub? |
| [03 · YAML for GitHub Actions](03-yaml-for-github-actions.md#quick-recap) | 3 | Why might `python-version: 3.10` install Python 3.1? · What's the difference between `\|` and `>`? · Can a YAML error consume runner minutes? |
| [04 · Linux Shell Recap for Runners](04-linux-shell-recap-for-runners.md#quick-recap) | 3 | How does GitHub Actions decide that a `run:` step failed? · Why might a failing command in a pipe not fail the step? · I `export`ed a variable in step 1. Why is it empty in step 2? |
| [05 · What is GitHub Actions?](05-what-is-github-actions.md#quick-recap) | 3 | Why is GitHub Actions called event-driven? · What's the main operational difference from Jenkins? · Can GitHub Actions run on your own servers? |
| [06 · Workflows, Jobs and Steps](06-workflows-jobs-steps.md#quick-recap) | 3 | Do two jobs in a workflow share files? · How do you make deploy wait for test? · What's the difference between `run` and `uses`? |
| [07 · Runners: GitHub-Hosted vs Self-Hosted](07-runners.md#quick-recap) | 3 | Why are GitHub-hosted runners considered secure and reproducible? · What's the main security risk of self-hosted runners? · Your job is stuck in "Queued". What do you check? |
| [08 · Your First Workflow, and What Happens After `git push`](08-first-workflow-and-what-happens-after-push.md#quick-recap) | 3 | Walk me through what happens after `git push`. · Why does `ls` show no files at the start of a job? · Does a YAML syntax error use runner minutes? |
| [09 · Workflow Triggers (Events)](09-workflow-triggers.md#quick-recap) | 3 | How do you let an external monitoring tool start a workflow? · Your nightly schedule didn't run. Name three possible reasons. · What's the Jenkins equivalent of `workflow_call`? |
| [10 · Event Filters and Activity Types](10-event-filters-and-activity-types.md#quick-recap) | 3 | What does `synchronize` mean? · A workflow has `push: branches: [develop]` and `paths: [app/ · Why might path filters be dangerous on required status checks? |
| [11 · Variables, Contexts and Expressions](11-variables-contexts-expressions.md#quick-recap) | 3 | Difference between `$GITHUB_REF` and `${{ github.ref }}`? · Why can a job be skipped without using any runner minutes? · Where would you store a non-secret value used by 20 repos? |
| [12 · Functions and Status Check Functions](12-functions-and-status-checks.md#quick-recap) | 3 | How do you run a step only when an earlier step failed? · Difference between `always()` and `!cancelled()`? · How would you build a cache key that changes only when dependencies change? |
| [13 · Conditional Execution with `if:`](13-conditional-execution.md#quick-recap) | 3 | Does a skipped job use runner minutes? · How do you run a summary job after other jobs no matter whether they passed? · Why might `deploy` be skipped even though its condition is true? |
| [14 · Job Control Options: `timeout-minutes`, `continue-on-error`, `defaults`, `shell`, `working-directory`](14-job-control-options.md#quick-recap) | 3 | What's the default job timeout on GitHub-hosted runners? · Difference between `steps.x.outcome` and `steps.x.conclusion`? · How do you make every `run:` step in a job start in `backend/`? |
| [15 · Actions and the GitHub Marketplace](15-actions-and-marketplace.md#quick-recap) | 3 | Difference between GitHub Actions and an action? · Why pin actions to a commit SHA instead of `@v1`? · What does `fetch-depth: 0` do? |
| [16 · Lab: Build and Smoke-Test a Flask App on a Runner](16-lab-flask-ci.md#quick-recap) | 3 | Why are build, run and smoke test in one job? · What does `-p 5000:5000` mean? · Why does `EXPOSE` alone not make the app reachable? |
| [17 · Secrets and Configuration Variables](17-secrets-and-variables.md#quick-recap) | 3 | Can you read a GitHub secret's value after saving it? · Why are secrets empty in a pull request from a fork? · Repository secret vs environment secret? |
| [18 · `GITHUB_TOKEN` and `permissions:`](18-github-token-and-permissions.md#quick-recap) | 3 | How do you apply least privilege to `GITHUB_TOKEN`? · Why didn't my workflow run after another workflow pushed a commit? · What does `id-token: write` do? |
| [19 · Lab: Build, Test and Push a Docker Image (Production-Style)](19-lab-docker-build-push.md#quick-recap) | 3 | Why build with `push: false` and push later? · How do you share a value generated in one step with later steps? · A PR's title is edited from "WIP …" to "…". Why didn't CI run? |
| [20 · Inputs: Workflow Inputs and Action Inputs](20-inputs.md#quick-recap) | 3 | How do you let an operator choose dev/qa/prod when running a workflow? · Why might a boolean input of `false` still pass an `if:`? · Can a workflow with inputs be started without the UI? |
| [21 · Outputs: Step Outputs and Job Outputs](21-step-and-job-outputs.md#quick-recap) | 3 | How do you pass a value from job A to job B? · Why was `::set-output` deprecated? · Outputs or artifacts for a test report? |
| [22 · Reusable Workflows (`workflow_call`): Inputs, Outputs and Secrets](22-reusable-workflows.md#quick-recap) | 3 | Reusable workflow vs composite action? · How does a caller read a value produced deep inside a reusable workflow? · Are secrets available inside a reusable workflow automatically? |
| [23 · Workflow Commands, Environment Files and Job Summaries](23-workflow-commands-and-summaries.md#quick-recap) | 3 | How do you make a tool you installed available to later steps? · Why were `set-env` and `add-path` disabled? · How do you show a result table on the workflow run page? |
| [24 · Artifacts: Sharing and Keeping Files](24-artifacts.md#quick-recap) | 3 | How do you pass a built JAR from the build job to the deploy job? · Outputs vs artifacts? · How long are artifacts kept? |
| [25 · Caching Dependencies](25-caching.md#quick-recap) | 3 | How do you avoid re-downloading Maven dependencies on every run? · Cache vs artifact? · Why might a PR never hit the cache that `main` built? |
| [26 · Matrix Strategy](26-matrix-strategy.md#quick-recap) | 3 | How many jobs does `os: [a,b,c]` × `node: [20,22]` with one `exclude` create? · What does `fail-fast: false` do? · How do you build a matrix from data computed at runtime? |
| [27 · Service Containers and Container Jobs](27-service-and-container-jobs.md#quick-recap) | 3 | How do you give integration tests a real PostgreSQL in GitHub Actions? · Why can't my step reach `postgres:5432`? · When would you use `container:` for a job? |
| [28 · Environments: Variables, Secrets, Approvals and Protection Rules](28-environments.md#quick-recap) | 3 | How do you require manual approval before a production deploy in GitHub Actions? · `if: github.ref == 'refs/heads/main'` vs a deployment branch restriction? · Repo secret and env secret have the same name. Which wins? |
| [29 · Concurrency: Controlling Overlapping Runs](29-concurrency.md#quick-recap) | 3 | How do you cancel outdated CI runs when a developer pushes again to the same PR? · Three runs hit a group with `cancel-in-progress: false`. What happens? · Why not `cancel-in-progress: true` on a Terraform apply job? |
| [30 · Branch Protection, Rulesets, Required Checks and CODEOWNERS](30-branch-protection-and-rulesets.md#quick-recap) | 3 | How do you ensure nothing merges to `main` without passing CI? · Why is a required check sometimes stuck at "Expected"? · How would you stop people from editing CI workflows without DevOps review? |
| [31 · Chaining Workflows: `workflow_run`, `merge_group` and Cross-Workflow Patterns](31-chaining-workflows.md#quick-recap) | 3 | Why didn't my `workflow_run` deploy workflow trigger? · Why is `workflow_run` considered safer for privileged actions on fork PRs? · In a `workflow_run` job, what does `actions/checkout` check out by default? |
| [32 · Custom Actions and Composite Actions](32-custom-actions-and-composite.md#quick-recap) | 3 | When would you choose a composite action over a JavaScript action? · Why does my local composite action fail with "can't find action.yml"? · How do you pass a secret into a composite action? |
| [33 · JavaScript Actions](33-javascript-actions.md#quick-recap) | 3 | When is a JavaScript action better than a composite action? · Why do JavaScript actions start faster than Docker actions? · A JavaScript action works locally but fails when used from another repo with "cannot find module". Why? |
| [34 · Docker Container Actions (and Which Action Type to Choose)](34-docker-actions.md#quick-recap) | 3 | Why are Docker actions Linux-only? · How does a Docker action access the repository files? · Composite, JavaScript or Docker for "run our internal Go CLI with a specific Terraform version"? |
| [35 · Publishing and Versioning Your Actions](35-publishing-and-versioning-actions.md#quick-recap) | 3 | How do popular actions let users write `@v4` but still get fixes? · Why might a security team forbid `@v4` and require SHAs? · How do you keep SHA-pinned actions up to date without manual work? |
| [36 · OIDC: Logging In to the Cloud Without Stored Keys](36-oidc-cloud-authentication.md#quick-recap) | 3 | Why use OIDC instead of storing AWS keys as GitHub secrets? · What does the IAM trust policy check? · How would you allow only production-approved jobs to deploy? |
| [37 · Security Hardening for GitHub Actions](37-security-hardening.md#quick-recap) | 3 | What is script injection in GitHub Actions and how do you prevent it? · Why is `pull_request_target` dangerous? · What lesson does the 2026 trivy-action compromise teach? |
| [38 · Self-Hosted Runners Deep Dive (Install, Labels, Groups, Ephemeral, ARC)](38-self-hosted-runners-deep-dive.md#quick-recap) | 3 | A job is stuck in "Queued" on self-hosted runners. What do you check? · Why prefer ephemeral runners? · What is ARC? |
| [39 · Debugging and Troubleshooting Workflows](39-debugging-and-troubleshooting.md#quick-recap) | 3 | How do you get more detailed logs for a failing GitHub Actions job? · How can you catch workflow syntax and expression errors before pushing? · A workflow with `on: push: branches: [main]` didn't run on your feature branch push. Bug? |
| [40 · Supply Chain Security: SBOMs, Provenance and Artifact Attestations](40-supply-chain-attestations.md#quick-recap) | 3 | What is an SBOM and why generate it in CI? · How does GitHub sign attestations without stored private keys? · How would you stop a manually built image from being deployed to production? |
| [41 · Billing, Limits and Cost Optimization](41-billing-limits-and-cost.md#quick-recap) | 3 | Name four ways to reduce GitHub Actions costs. · Are self-hosted runners free? · Why can 10 tiny jobs cost more than 1 job? |
| [42 · Capstone 00 · Overview, Prerequisites and the Application](42-capstone-00-overview.md#quick-recap) | 3 | Describe the capstone's release flow. · Why separate CI, DEV deploy, PR creation and PROD deploy into four workflow files? · What does "build once, promote the same artifact" mean here? |
| [43 · Capstone 01 · Project, Repository, Governance and First Run (Stages 1–4)](43-capstone-01-repo-foundation.md#quick-recap) | 3 | Why does pushing a brand-new feature branch not trigger this CI? · Why enable required status checks only after the first CI run? · What does "require approval of the most recent reviewable push" prevent? |
| [44 · Capstone 02 · Trivy Filesystem Security Scanning (Stage 5)](44-capstone-02-trivy-filesystem-scan.md#quick-recap) | 3 | Filesystem scan vs image scan? · How do you make a scanner block the pipeline? · A scan finds a committed AWS key. What do you do? |
| [45 · Capstone 03 · Build and Validate the Backend and Frontend (Stage 6)](45-capstone-03-build-and-test.md#quick-recap) | 3 | Why `npm ci` instead of `npm install` in CI? · How does `setup-java`'s `cache: maven` work? · What does `mvn verify` run? |
| [46 · Capstone 04 · AWS Infrastructure Labs: EKS, Networking, RDS, SonarQube](46-capstone-04-aws-infrastructure-labs.md#quick-recap) | 3 | Why reference security groups instead of IPs in rules? · How would you let CI reach a private SonarQube without opening it to the internet? · What's the EKS OIDC provider for? |
| [47 · Capstone 05 · Code Analysis and Quality Gates with SonarQube (Stage 7)](47-capstone-05-sonarqube-analysis.md#quick-recap) | 3 | Does SonarQube run your tests? · How does a Quality Gate stop a pipeline? · Why use the Maven plugin for Java instead of the generic scanner? |
| [48 · Capstone 06 · Containerize and Scan the Application (Stage 8)](48-capstone-06-containerize-and-scan.md#quick-recap) | 3 | Why scan both the repo and the image? · Why move the image to the push job as a `.tar` artifact instead of rebuilding? · Name three Dockerfile hardening practices used here. |
| [49 · Capstone 07 · Publish and Promote Images to Amazon ECR with OIDC (Stage 9)](49-capstone-07-publish-images-to-ecr.md#quick-recap) | 3 | How does the workflow push to ECR without AWS keys? · Why immutable tags? · What's the difference between the trust policy and the permissions policy? |
| [50 · Capstone 08 · Allow GitHub Actions to Deploy to EKS (Stage 10)](50-capstone-08-eks-deploy-access.md#quick-recap) | 3 | Why does a GitHub job need both IAM and Kubernetes permissions to deploy to EKS? · How do you make sure the DEV workflow can't deploy to PROD? · What replaced the `aws-auth` ConfigMap? |
| [51 · Capstone 09 · DEV Environment and Automated DEV Deployment (Stage 11)](51-capstone-09-dev-deployment.md#quick-recap) | 3 | How does the DEV workflow know which images to deploy? · Why deploy by digest instead of tag? · Why is the DEV deploy a separate workflow from CI? |
| [52 · Capstone 10 · Automated Pull Request Creation (Stage 12)](52-capstone-10-automated-pull-request.md#quick-recap) | 3 | Why verify `git rev-parse HEAD` against the deployed SHA before creating the PR? · Why don't workflows trigger on the PR the bot creates? · Why make PR creation a reusable workflow? |
| [53 · Capstone 11 · PROD Environment and Automated PROD Deployment (Stage 13)](53-capstone-11-prod-deployment.md#quick-recap) | 3 | How does the capstone make sure PROD deploys exactly what DEV tested? · Why separate the merge approval from the deployment approval? · Why does the PROD environment allow `refs/pull/*/merge` rather than `main`? |
| [54 · Capstone 12 · Push, Configure and Validate the Complete Automation (Stage 14) + Next Steps](54-capstone-12-end-to-end-validation.md#quick-recap) | 3 | Your new `workflow_run` deploy workflow doesn't trigger. Why? · How do you enforce "CI must pass and someone else must review" before merging to main? · How does the project separate duties for production? |

---

## Hands-on lab

1. Answer one Part 2b scenario aloud in under 2 minutes, using the 4 beats.
1. Cover the "Short answer" column in Part 2 and answer all 30 aloud. Mark the misses.
2. For each miss, open the linked topic and redo its hands-on lab.
3. Explain the Jenkins map rows to a friend (or to yourself) for a Jenkins → GitHub Actions migration.
4. Do a mock "design a pipeline" answer using the capstone flow in 5 minutes.

## Where you'll use it

Interviews (DevOps/SRE/platform), team onboarding, and Jenkins → GitHub Actions migration planning.

## When it breaks

| Symptom | Fix |
|---|---|
| You know the term but can't explain "what happens inside" | Re-read that topic's **How it works inside** section, then draw the flow from memory. |
| Answers sound memorized | Tie each answer to something you ran in a lab, with the log line you saw. |

## Quick recap

- Part 1 translates every concept to Jenkins. Part 2 is the 30-question drill. Part 2b is 10 production incident scenarios (answer them in 4 beats: symptom → narrow down → root cause → fix plus guardrail). Part 3 lists all 162 topic questions, linking to their answers in each topic.
- Revise with the drill, deepen with the topics, prove it with the labs.

**Interview questions**

1. **Q: How would you explain GitHub Actions to a Jenkins expert in one sentence?**
   A: It's a managed, event-driven CI/CD service where Jenkinsfile ≈ workflow YAML, agents ≈ runners, plugins ≈ actions, shared libraries ≈ reusable workflows, and credentials ≈ secrets/OIDC, all built into GitHub.
2. **Q: What's the biggest operational difference?**
   A: GitHub runs the control plane (and optionally the runners), so there's no controller to patch or scale.
3. **Q: What's the biggest security difference?**
   A: Per-run scoped tokens, OIDC to clouds, and environments/rulesets as first-class governance, versus long-lived credentials and plugins on a self-managed server.
