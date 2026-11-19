![GitHub Actions – Basics to Production](https://img.shields.io/badge/GitHub_Actions-Basics_to_Production-blue?style=for-the-badge&logo=githubactions&logoColor=white)

# GitHub Actions: Zero to Production — Reading Order

> One self-contained learning path. Read the topics **in order**: each one uses only ideas explained before it (or explains a new term on the spot).
>
> Every topic follows the same format: **What it is → Why it exists → Analogy → How it works inside → Workflow YAML / commands (with sample logs) → Hands-on lab → Where you'll use it → When it breaks → Quick recap + interview questions.**
>
> Corrections to outdated facts are marked **⚠️ Corrected: was X, now Y because Z**. Course videos may show older versions; these notes use the versions current in November 2026 (see the [cheat sheet](00-cheatsheet.md)).
>
> **About the sample logs:** the "sample log output" blocks are **illustrative**. They're reconstructed from GitHub's real log format to show what to look for, not captured from one specific run, so timestamps, SHAs, durations and tool versions in your runs will differ.
>
> **Short on time?** Start with [00 · Cheat sheet](00-cheatsheet.md) (two pages: versions, limits, gotchas, a full workflow skeleton), then the [interview bank](55-interview-bank-and-jenkins-map.md).

---

## Start here

| # | Topic | What you'll learn |
|---|---|---|
| 00 | [Cheat sheet](00-cheatsheet.md) | current versions (Nov 2026) and limits, a full workflow skeleton, 14 rules that trip people up, which tool to use, debugging in 60 s. Read it last for revision, or first for orientation. |

## Part A · Foundations

| # | Topic | What you'll learn |
|---|---|---|
| 01 | [What is CI/CD?](01-what-is-ci-cd.md) | CI vs Continuous Delivery vs Continuous Deployment, the SDLC, the 9-stage pipeline, Jenkins vs GitLab vs GitHub Actions |
| 02 | [Git and GitHub recap](02-git-and-github-recap.md) | repos, commits, branches, PRs, SSH keys — the one-time setup every lab uses |
| 03 | [YAML for GitHub Actions](03-yaml-for-github-actions.md) | YAML (YAML Ain't Markup Language): maps, lists, quoting, `\|` blocks, common errors |
| 04 | [Linux shell recap for runners](04-linux-shell-recap-for-runners.md) | how `run:` steps execute, exit codes, env vars, PATH, `set -euo pipefail` |
| 05 | [What is GitHub Actions?](05-what-is-github-actions.md) | event-driven automation, control plane vs data plane, why it's popular, vs Jenkins |
| 06 | [Workflows, jobs and steps](06-workflows-jobs-steps.md) | the building blocks, parallel jobs, `needs`, `run` vs `uses`, failure behaviour |
| 07 | [Runners](07-runners.md) | hosted vs self-hosted, sizes (public vs private), larger runners, ubuntu-latest → 26.04, strategy |
| 08 | [First workflow and what happens after `git push`](08-first-workflow-and-what-happens-after-push.md) | your first run + the full internal trace from push to green tick |

## Part B · Triggers and logic

| # | Topic | What you'll learn |
|---|---|---|
| 09 | [Workflow triggers](09-workflow-triggers.md) | push, PR, dispatch, schedule (UTC, default branch, 60-day rule), repository_dispatch, workflow_call |
| 10 | [Event filters and activity types](10-event-filters-and-activity-types.md) | branches/paths/tags filters, PR activity types, AND vs OR, `synchronize` |
| 11 | [Variables, contexts and expressions](11-variables-contexts-expressions.md) | `$VAR` vs `${{ }}`, default variables, `env`/`vars`, engine vs runner evaluation |
| 12 | [Functions and status checks](12-functions-and-status-checks.md) | contains/startsWith/format/hashFiles/fromJSON/toJSON, success/failure/always/cancelled |
| 13 | [Conditional execution](13-conditional-execution.md) | job vs step `if:`, `needs.<job>.result`, skip propagation |
| 14 | [Job control options](14-job-control-options.md) | timeout-minutes, continue-on-error, defaults, shell, working-directory |

## Part C · Actions, secrets and tokens

| # | Topic | What you'll learn |
|---|---|---|
| 15 | [Actions and the Marketplace](15-actions-and-marketplace.md) | what an action is, sources, choosing actions, tags vs SHA pinning |
| 16 | [Lab: Flask CI on a runner](16-lab-flask-ci.md) | Docker recap, build → run → smoke test in one job |
| 17 | [Secrets and variables](17-secrets-and-variables.md) | repo/org/environment scopes, masking, fork PRs, access tokens |
| 18 | [`GITHUB_TOKEN` and permissions](18-github-token-and-permissions.md) | the automatic token, `permissions:`, 403 fixes, anti-loop rule |
| 19 | [Lab: Docker build, test and push](19-lab-docker-build-push.md) | Docker Hub actions, Build → Test → Publish, `$GITHUB_ENV`, PR/WIP/draft logic |

## Part D · Moving data around

| # | Topic | What you'll learn |
|---|---|---|
| 20 | [Inputs](20-inputs.md) | workflow_dispatch inputs (types), action inputs, environment-aware pipeline |
| 21 | [Step and job outputs](21-step-and-job-outputs.md) | `$GITHUB_OUTPUT`, job outputs, `set-output` deprecation |
| 22 | [Reusable workflows](22-reusable-workflows.md) | `workflow_call` inputs/outputs/secrets, `secrets: inherit`, limits |
| 23 | [Workflow commands and job summaries](23-workflow-commands-and-summaries.md) | GITHUB_ENV/PATH/STEP_SUMMARY, annotations, `::add-mask::`, deprecated commands |
| 24 | [Artifacts](24-artifacts.md) | upload/download, retention, immutability, testing types |
| 25 | [Caching](25-caching.md) | cache keys, hits/misses, restore keys, branch scope, setup-* caching |
| 26 | [Matrix strategy](26-matrix-strategy.md) | multi-version testing, include/exclude, fail-fast, max-parallel, dynamic matrices |
| 27 | [Service containers and container jobs](27-service-and-container-jobs.md) | Postgres/Redis for tests, `container:` jobs, localhost vs service name |

## Part E · Deployment control

| # | Topic | What you'll learn |
|---|---|---|
| 28 | [Environments](28-environments.md) | environment variables/secrets, branch rules, reviewers, wait timer, deployment history |
| 29 | [Concurrency](29-concurrency.md) | groups, cancel-in-progress, pending rules, `queue: max`, dynamic groups |
| 30 | [Branch protection and rulesets](30-branch-protection-and-rulesets.md) | rulesets, required checks, CODEOWNERS, the bootstrap exception, org-wide governance (required workflows, templates, allowed actions, runner groups, budgets) |
| 31 | [Chaining workflows](31-chaining-workflows.md) | `workflow_run`, deploy-after-merge, dispatch, merge queues |

## Part F · Custom actions

| # | Topic | What you'll learn |
|---|---|---|
| 32 | [Custom actions and composite actions](32-custom-actions-and-composite.md) | action.yml, three types, local vs remote, two composite demos |
| 33 | [JavaScript actions](33-javascript-actions.md) | node24 runtime (node20 removed), Actions Toolkit, bundling |
| 34 | [Docker actions](34-docker-actions.md) | container actions, `/github/workspace`, choosing an action type |
| 35 | [Publishing and versioning actions](35-publishing-and-versioning-actions.md) | semver, major tags, Marketplace, immutable releases, Dependabot |

## Part G · Security and production

| # | Topic | What you'll learn |
|---|---|---|
| 36 | [OIDC cloud authentication](36-oidc-cloud-authentication.md) | OpenID Connect to AWS/Azure/GCP, trust policies, `sub` claims |
| 37 | [Security hardening](37-security-hardening.md) | script injection, pull_request_target (and checkout v7's new safe default), SHA pinning, least privilege |
| 38 | [Self-hosted runners deep dive](38-self-hosted-runners-deep-dive.md) | install, labels, groups, ephemeral runners, ARC on Kubernetes |
| 39 | [Debugging and troubleshooting](39-debugging-and-troubleshooting.md) | debug logging, actionlint, act, gh CLI, the error catalogue |
| 40 | [Supply chain: SBOM, provenance, attestations](40-supply-chain-attestations.md) | SBOMs, SLSA, artifact attestations, verification |
| 41 | [Billing, limits and cost](41-billing-limits-and-cost.md) | minutes, storage, limits, cost-saving patterns |

## Part H · Capstone: production DevSecOps pipeline

| # | Topic | What you'll learn |
|---|---|---|
| 42 | [Capstone 00 · Overview](42-capstone-00-overview.md) | the 3-tier app, the release flow, the capstone map |
| 43 | [Capstone 01 · Repo foundation (Stages 1–4)](43-capstone-01-repo-foundation.md) | first CI, repo + SSH, `protect-main` ruleset, dev/prod environments |
| 44 | [Capstone 02 · Trivy filesystem scan (Stage 5)](44-capstone-02-trivy-filesystem-scan.md) | vulnerability/secret/misconfig scanning gate |
| 45 | [Capstone 03 · Build and test (Stage 6)](45-capstone-03-build-and-test.md) | Maven + npm jobs, coverage, dependency caching |
| 46 | [Capstone 04 · AWS infrastructure labs](46-capstone-04-aws-infrastructure-labs.md) | EKS, networking, security groups, RDS, SonarQube server |
| 47 | [Capstone 05 · SonarQube analysis (Stage 7)](47-capstone-05-sonarqube-analysis.md) | quality gates in CI |
| 48 | [Capstone 06 · Containerize and scan (Stage 8)](48-capstone-06-containerize-and-scan.md) | hardened Dockerfiles, image scans, artifact hand-off |
| 49 | [Capstone 07 · Publish to ECR (Stage 9)](49-capstone-07-publish-images-to-ecr.md) | OIDC provider, IAM role, immutable tags, push jobs |
| 50 | [Capstone 08 · EKS deploy access (Stage 10)](50-capstone-08-eks-deploy-access.md) | deploy roles, access entries |
| 51 | [Capstone 09 · DEV deployment (Stage 11)](51-capstone-09-dev-deployment.md) | RBAC, secrets, `workflow_run` deploy by digest |
| 52 | [Capstone 10 · Automated PR (Stage 12)](52-capstone-10-automated-pull-request.md) | reusable PR workflow with SHA verification |
| 53 | [Capstone 11 · PROD deployment (Stage 13)](53-capstone-11-prod-deployment.md) | PR-merged trigger, prod approval, same-artifact promotion |
| 54 | [Capstone 12 · End-to-end validation (Stage 14)](54-capstone-12-end-to-end-validation.md) | go-live settings, bootstrap, full release, next steps |

## Part I · Review

| # | Topic | What you'll learn |
|---|---|---|
| 55 | [Interview bank and Jenkins map](55-interview-bank-and-jenkins-map.md) | GitHub Actions ↔ Jenkins table, 30-question drill, 10 production incident scenarios, all topic questions |

---

## Supporting folders

- [`code/`](code/) — every demo's source files, grouped by the topic that uses them (`code/<topic>-…/`, `code/capstone/`).
- [`images/`](images/) — diagrams used by the topics.
- The original lecture-by-lecture notes these topics were built from are in this repo's git history (commit `1bb40aa`): `git show 1bb40aa:<path>`.

---

## Course resources

> Learn modern CI/CD and workflow automation the way it works in real production environments.

This material is a structured, production-focused guide to **GitHub Actions and modern CI/CD workflows**, from foundational concepts to real-world automation patterns. It complements the video lectures with notes, diagrams, architecture explanations and hands-on workflow examples, building **strong conceptual understanding first** and then moving into **production-grade CI/CD implementation patterns**.

**Course introduction video:**

[![Watch the Course Introduction](https://img.youtube.com/vi/xRqFDrSmLSM/maxresdefault.jpg)](https://www.youtube.com/watch?v=xRqFDrSmLSM&ab_channel=CloudWithVarJosh)

**YouTube playlist:** [GitHub Actions: Basics to Production](https://www.youtube.com/playlist?list=PLmPit9IIdzwThF00pAAvktjkVtL7APhD5)

**Lecture videos** (also linked at the top of the matching topics):

| Lecture | Video | Topics |
|---|---|---|
| 01 What is GitHub Actions / first workflow | [w4c_NIjO3XI](https://www.youtube.com/watch?v=w4c_NIjO3XI) | 01, 05–08 |
| 02 Triggers & runners | [zpDH_tfOOqc](https://www.youtube.com/watch?v=zpDH_tfOOqc) | 07, 09 |
| 03 Actions, first production-style workflow | [IV2Focqf11Q](https://www.youtube.com/watch?v=IV2Focqf11Q) | 15, 16, 19 |
| 04 Filters, variables, expressions, contexts | [KIH79Y3xJ8c](https://www.youtube.com/watch?v=KIH79Y3xJ8c) | 10, 11 |
| 05 Functions | [Y0kDz3BEtM4](https://www.youtube.com/watch?v=Y0kDz3BEtM4) | 12, 19 |
| 06 Inputs | [fLZRgNhQq3w](https://youtu.be/fLZRgNhQq3w) | 20, 22 |
| 07 Outputs | [gkIXTCr1Iiw](https://www.youtube.com/watch?v=gkIXTCr1Iiw) | 21, 22 |
| 08 Artifacts & caching | [tw9e61Bct-E](https://www.youtube.com/watch?v=tw9e61Bct-E) | 24, 25 |
| 09 Matrix strategy | [B5XzHnO_guo](https://www.youtube.com/watch?v=B5XzHnO_guo) | 26 |
| 10 Environments | [p_qnXBlJg20](https://www.youtube.com/watch?v=p_qnXBlJg20) | 28 |
| 11 Custom actions: composite | [v6VdLk1gQSc](https://www.youtube.com/watch?v=v6VdLk1gQSc) | 32 |
| 12 JavaScript & Docker actions | [sjSmSzMJL5s](https://www.youtube.com/watch?v=sjSmSzMJL5s) | 33, 34 |
| 13 Concurrency | [0wXdpfNSWuI](https://www.youtube.com/watch?v=0wXdpfNSWuI) | 29 |
| Mega-project (capstone) | — | 42–54 |

**Course author:** Varun Joshi (Cloud With VarJosh). These notes are a personal rework of the course material. [LinkedIn: Varun Joshi](https://www.linkedin.com/in/varun-joshi-2b516752/) · [YouTube: Cloud With VarJosh](https://www.youtube.com/@CloudWithVarJosh) · [GitHub](https://github.com/CloudWithVarJosh)

**Related learning resources mentioned in the lectures:** [Git Masterclass playlist](https://www.youtube.com/playlist?list=PLmPit9IIdzwT6nq8hIuf7MBcPVzw3cX3K) · [Git Masterclass notes](https://github.com/CloudWithVarJosh/Git-Masterclass) · [YAML video](https://youtu.be/LT3U9fkOVDQ) · [Jenkins Basics to Production](https://github.com/CloudWithVarJosh/Jenkins-Basics-To-Production) · [CKA course](https://lnkd.in/g-7UEQfk) · [Kubernetes projects](https://lnkd.in/gg4tFfTD) · [Argo CD tutorial](https://youtu.be/m4lDTQwK1T8) · [Maven tutorial](https://youtu.be/3OKc5y_3wMM) · [DevSecOps & SonarQube](https://youtu.be/qyYsLVZDieU)
