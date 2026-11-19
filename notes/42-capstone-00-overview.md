# 42 · Capstone 00 · Overview, Prerequisites and the Application

> **Part H: Capstone** · Previous: [41 · Billing, limits and cost](41-billing-limits-and-cost.md) · Next: [43 · Project, Repository, Governance and First Run (Stages 1–4)](43-capstone-01-repo-foundation.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

The **capstone** is one end-to-end, production-style pipeline that uses almost every topic in these notes. It ships a **three-tier user-management app**:

| Tier | Technology | Recap |
|---|---|---|
| Frontend | **React** UI, built with **Vite**, dependencies via **npm**, served by **Nginx** | static files in a container |
| Backend | **Java 21 + Spring Boot** REST API at `/api/v1/users`, built with **Maven** into a JAR | JAR in a container |
| Database | **Amazon RDS for PostgreSQL** (a managed database) | outside the cluster |

It runs on **Amazon EKS** (Elastic Kubernetes Service, AWS's managed Kubernetes) in **DEV** and **PROD** namespaces.

The finished release process:

```text
Feature Branch → CI Validation → Container Build and Scan → Amazon ECR → DEV Deployment
→ Automated Pull Request → Human Review and Merge → PROD Approval → PROD Deployment
```

**Capstone map:**

| File | Stage(s) | You build |
|---|---|---|
| [42](42-capstone-00-overview.md) | Phase 0 | understand the app and the plan |
| [43](43-capstone-01-repo-foundation.md) | 1–4 | repo, SSH, first workflow, `protect-main` ruleset, `dev`/`prod` environments |
| [44](44-capstone-02-trivy-filesystem-scan.md) | 5 | Trivy filesystem scan job |
| [45](45-capstone-03-build-and-test.md) | 6 | backend (Maven) and frontend (npm/Vite) build-and-test jobs with caching |
| [46](46-capstone-04-aws-infrastructure-labs.md) | Labs 1–5 | EKS cluster, networking, security groups, RDS, SonarQube server |
| [47](47-capstone-05-sonarqube-analysis.md) | 7 | SonarQube analysis and Quality Gates in CI |
| [48](48-capstone-06-containerize-and-scan.md) | 8 | Dockerfiles, image build and Trivy image scans |
| [49](49-capstone-07-publish-images-to-ecr.md) | 9 | ECR repos, GitHub OIDC provider, IAM role, push jobs |
| [50](50-capstone-08-eks-deploy-access.md) | 10 | IAM deploy roles, EKS access entries |
| [51](51-capstone-09-dev-deployment.md) | 11 | DEV DB, RBAC, secrets, `workflow_run` DEV deploy |
| [52](52-capstone-10-automated-pull-request.md) | 12 | reusable PR-creation workflow |
| [53](53-capstone-11-prod-deployment.md) | 13 | PROD DB and the `pull_request: closed` PROD deploy |
| [54](54-capstone-12-end-to-end-validation.md) | 14 + next steps | go-live configuration, end-to-end validation, future improvements |

Final workflows (in [`code/capstone/workflow-yamls/`](code/capstone/workflow-yamls/)): `01-ci.yaml` (CI), `02-deploy-dev.yaml` (DEV, `workflow_run`), `03-create-pr.yaml` (reusable PR creation, `workflow_call`), `04-deploy-prod.yaml` (PROD, PR merged).

## Why it exists

Each topic so far taught one feature in isolation. Real pipelines combine them, and **most interview "design a pipeline" questions are this project**. Building it shows how triggers, filters, jobs, caching, artifacts, outputs, secrets, OIDC, environments, rulesets, concurrency, reusable workflows and `workflow_run` work **together**, plus the DevSecOps tools (**Trivy**, **SonarQube**) and the platforms (**ECR**, **EKS**, **RDS**).

## Analogy

Building a **car factory line** after learning each machine separately: the parts arrive (code), get inspected (Trivy, tests, SonarQube), assembled into a sealed unit (container image), inspected again (image scan), stored in the warehouse with a serial number (ECR + immutable tag/digest), test-driven on the private track (DEV), signed off by a manager (PR review + PROD approval), then delivered to customers (PROD). The **same car** (image digest) goes all the way. Nothing is rebuilt between the test track and the customer.

## How it works inside

```text
developer pushes feature/x
  │
  ▼  01-ci.yaml  (push on feature/**|bugfix/**|hotfix/**, paths filter, concurrency cwvj-ci-<ref>)
  trivy-scan-job ─┐
  backend-job ──┼─► backend-image-build-scan-job ─► backend-ecr-push-job   (OIDC → ECR, immutable tag)
  frontend-job ─┘─► frontend-image-build-scan-job ─► frontend-ecr-push-job
  │ (SonarQube quality gate inside backend-job / frontend-job)
  ▼  workflow_run: completed + conclusion == success
  02-deploy-dev.yaml  (environment: dev, OIDC → EKS role, find image digests, kubectl apply, smoke test)
  │
  ▼  uses: ./.github/workflows/03-create-pr.yaml   (GITHUB_TOKEN → gh pr create, body contains the SHA)
  PR "Promote feature/x to main"  ── human review + required CI checks (protect-main ruleset) ── squash merge
  │
  ▼  04-deploy-prod.yaml  (pull_request: closed, merged, bot-created, body contains head.sha)
  environment: prod  ── required reviewer approves (prevent self-review) ── deploy SAME digests to prod
```

## Workflow YAML / commands: build it step by step

**Prerequisites.** Basic Git (topic [02](02-git-and-github-recap.md)), CI/CD (topic [01](01-what-is-ci-cd.md)), GitHub Actions (topics 05–41), Docker (topic [16](16-lab-flask-ci.md)), and a little Maven, SonarQube, Kubernetes and AWS. You don't need to be an expert, because each stage introduces what it needs. Optional deeper material: [Maven](https://www.youtube.com/watch?v=3OKc5y_3wMM) · [SonarQube](https://www.youtube.com/watch?v=qyYsLVZDieU) · [Kubernetes (CKA course)](https://lnkd.in/g-7UEQfk) · [Modern SDLC](https://youtu.be/imEHsgvJbYo).

**Install these tools** on your machine or a jump host:

| Tool | Used for | Install |
|---|---|---|
| Git | clone, branch, commit, push | [git-scm.com](https://git-scm.com/install) |
| AWS CLI v2 | talk to AWS | [AWS docs](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) |
| eksctl | create and manage the EKS cluster | [eksctl.io](https://eksctl.io/installation/) |
| kubectl | manage and check Kubernetes resources | [kubernetes.io](https://kubernetes.io/docs/tasks/tools/) |

**The five phases:**

| Phase | Goal | Topics |
|---|---|---|
| 0 · Understand the app | the three tiers and how a request flows | 42 |
| 1 · Repository foundation | repo, SSH, first workflow, ruleset, environments | 43 |
| 2 · CI and publishing | Trivy, build/test, SonarQube gates, image build and scan, push to ECR | 44–49 |
| 3 · AWS and EKS foundation | EKS, RDS, networking, OIDC, IAM roles, access entries, RBAC | 46, 49–51 |
| 4 · CD and release management | DEV/PROD deploys, automated PR, approvals, go-live, end-to-end check | 51–54 |

### Stage 0: the application

| Tier | Technology | What it does |
|---|---|---|
| Frontend | **React** (UI components), **Vite** (dev server plus production build into static files), **npm** (dependencies and scripts: `npm run dev`, `npm run build`), **Nginx** (serves the static files) | Users view, add, edit and delete user records. It calls the backend API and shows the response. |
| Backend | **Java 21 + Spring Boot**, built by **Maven** into a JAR | A REST API at `/api/v1/users`. It validates input, applies business rules and reads or writes the database. |
| Database | **Amazon RDS for PostgreSQL** | Stores the `users` table persistently, outside the containers. Constraints stop invalid or duplicate rows. |

**Request flow:** browser → Nginx (frontend) → `/api/v1/users` on the backend → PostgreSQL → response → the UI updates. Listing, adding, editing and deleting users all follow this same path.

**Deployment:** the frontend and backend run as separate workloads on **Amazon EKS** across several Availability Zones, in `dev` and `prod` namespaces. RDS lives outside the cluster, so data survives pod restarts.

## Hands-on lab

1. Read Stage 0 above and draw the three tiers and the request flow (browser → Nginx → backend → RDS) yourself.
2. Install the tools listed (Git, AWS CLI v2, eksctl, kubectl) and check them: `git --version; aws --version; eksctl version; kubectl version --client`.
3. Browse [`code/capstone/`](code/capstone/): find the backend `pom.xml`, the frontend `package.json`, the four workflow YAMLs and the Kubernetes manifests.
4. **Cost warning:** EKS, RDS and EC2 cost money. Plan to finish in a few days and **delete everything** afterwards (eksctl delete cluster, delete RDS/EC2/ECR).

## Where you'll use it

- **Interviews:** "Walk me through a production CI/CD pipeline you built." This is it.
- **Portfolio:** a public repo with this pipeline is strong evidence for DevOps/SRE roles.
- **Real work:** the same shape (scan → build/test → quality gate → image scan → registry → DEV → PR → approval → PROD) is used in most companies.

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Unexpected AWS bill | Cluster, RDS or EC2 left running. | Delete the resources after each session (topic 41 mindset applies to AWS too). |
| Lost in the stages | Copying the final workflow too early. | Follow the stages **in order**. Each one adds a tested piece. |
| A tool command not found | Missing local CLI. | Install from the links in "Install Required Tools". |

## Quick recap

- Three tiers: React/Vite/Nginx · Spring Boot/Maven · RDS PostgreSQL, on EKS DEV/PROD.
- Four workflows: CI → (workflow_run) DEV → (workflow_call) PR → (PR merged) PROD.
- Key principles: DevSecOps gates, **OIDC** (no stored AWS keys), **immutable tags and digests** (build once, promote the same artifact), **human approvals** for merge and PROD.

**Interview questions**

1. **Q: Describe the capstone's release flow.**
   A: Feature-branch push → CI (Trivy, build/test, SonarQube, image build and scan, push to ECR) → `workflow_run` DEV deploy → automated PR → human review and merge → PROD environment approval → deploy the same image digests to PROD.
2. **Q: Why separate CI, DEV deploy, PR creation and PROD deploy into four workflow files?**
   A: Clear responsibilities, different triggers and permissions, and separate governance (environments and approvals) for each step.
3. **Q: What does "build once, promote the same artifact" mean here?**
   A: Images are built and scanned once in CI, then DEV and PROD deploy those exact images by digest. Nothing is rebuilt per environment.
