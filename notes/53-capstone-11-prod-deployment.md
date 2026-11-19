# 53 · Capstone 11 · PROD Environment and Automated PROD Deployment (Stage 13)

> **Part H: Capstone** · Previous: [52 · Automated Pull Request Creation (Stage 12)](52-capstone-10-automated-pull-request.md) · Next: [54 · Push, Configure and Validate the Complete Automation (Stage 14) + Next Steps](54-capstone-12-end-to-end-validation.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

**Stage 13** prepares **PROD** (its RDS user and database) and writes **`04-deploy-prod.yaml`**, triggered by **`pull_request: closed`** into `main`. It runs only if the PR was **merged**, **created by `github-actions[bot]`**, and its **body contains `head.sha`**. It uses **`environment: prod`** (required reviewer with prevent-self-review, Stage 14), OIDC with the PROD role, and deploys the **same image digests** that ran in DEV, using the PROD manifests from the **merge commit**.

## Why it exists

Continuous **Delivery** to production with two human gates: **merge approval** (code review) and **deployment approval** (environment reviewer, a different person). Using the **same digests** as DEV means production runs exactly the tested artifact.

## Analogy

The signed release form (merged PR) goes to the **dispatch office for customer deliveries**. A **different** manager must stamp the delivery order (environment approval, no self-approval). The truck then delivers **the same VIN** that passed the test track, using the **latest delivery instructions** from head office (manifests in the merge commit).

## How it works inside

```text
PR "Promote feature/x to main" merged (by the reviewer account)
  └─► 04-deploy-prod.yaml  on: pull_request: types [closed], branches [main]
       concurrency: cwvj-deploy-prod (no cancel)     environment: prod      timeout 20m
       if: merged == true && user.login == 'github-actions[bot]' && contains(body, head.sha)
       ⏸ waits for the prod reviewer (prevent self-review; admins can't bypass)
       permissions: contents: read, id-token: write
       checkout merge_commit_sha             ← PROD manifests as merged into main
       images for head.sha                   ← the SAME images DEV deployed (by digest)
       OIDC → AWS_EKS_DEPLOY_ROLE_ARN_PROD (trust: environment:prod)
       fill kubernetes/prod/*.yaml → kubectl apply → rollout → smoke test
```

Key details:

- **Two commit references on purpose:** `merge_commit_sha` (manifests) vs `pull_request.head.sha` (the app commit whose images were built and tested).
- **Environment branch rule = `refs/pull/*/merge`:** for `pull_request` events the job runs on the PR merge ref, so the PROD environment must allow that ref (the original Stage 14 explains why `main` alone isn't the evaluated ref).
- Topics: 28 (environments, reviewers), 31 (deploy-after-merge pattern), 36 (OIDC environment `sub`), 29 (concurrency).

## Workflow YAML / commands: build it step by step

### Stage 13: Prepare PROD and write the PROD deploy workflow

PROD runs **only after a person merges the automated PR into `main`** (and, from Stage 14, after a reviewer approves the `prod` environment). It deploys the **same digests** that DEV ran, with no rebuild. (Still not pushed: that happens in Stage 14.)

#### 1. The PROD database (same RDS instance, separate DB and user)

Same steps as DEV (topic 51), with a **different password**:

```sql
-- psql --host=<YOUR_RDS_ENDPOINT> --port=5432 --username=postgres --dbname=postgres --password
SELECT current_user;
CREATE ROLE cwvj_prod_app_user LOGIN;
\password cwvj_prod_app_user
GRANT cwvj_prod_app_user TO postgres;
CREATE DATABASE cwvj_prod_db OWNER cwvj_prod_app_user;
REVOKE cwvj_prod_app_user FROM postgres;
\connect cwvj_prod_db
GRANT USAGE, CREATE ON SCHEMA public TO cwvj_prod_app_user;
\conninfo
\l cwvj_prod_db
\du cwvj_prod_app_user
\q
```

Separate databases and credentials keep PROD data and access apart from DEV. (A separate RDS instance per environment would isolate them even more.) The `prod` namespace, RBAC and `backend-db-secret` were created in Stage 11.

#### 2. `.github/workflows/04-deploy-prod.yaml`

```yaml
name: CWVJ Deploy | PROD

on:
  pull_request:
    types:
      - closed
    branches:
      - main

concurrency:
  group: cwvj-deploy-prod
  cancel-in-progress: false

jobs:
  deploy-to-prod-job:
    name: Deploy to PROD
    runs-on: ubuntu-26.04
    timeout-minutes: 20
    environment: prod

    # Deploy only when an automated DEV-approved PR is merged.
    if: >-
      github.event.pull_request.merged == true &&
      github.event.pull_request.user.login == 'github-actions[bot]' &&
      contains(github.event.pull_request.body, github.event.pull_request.head.sha)

    permissions:
      contents: read
      id-token: write

    steps:
      # Check out the merged main revision for production manifests.
      - name: Check out merged main
        uses: actions/checkout@v7
        with:
          ref: ${{ github.event.pull_request.merge_commit_sha }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.AWS_EKS_DEPLOY_ROLE_ARN_PROD }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Find the published image digests
        env:
          CI_SHA: ${{ github.event.pull_request.head.sha }}
        run: |
          set -euo pipefail

          SHORT_SHA="${CI_SHA::7}"

          BACKEND_DIGEST=$(aws ecr describe-images \
            --repository-name cwvj-mega-backend \
            --filter tagStatus=TAGGED \
            --query "imageDetails[?imageTags[?ends_with(@, '-${SHORT_SHA}')]].imageDigest | [0]" \
            --output json | tr -d '"')

          FRONTEND_DIGEST=$(aws ecr describe-images \
            --repository-name cwvj-mega-frontend \
            --filter tagStatus=TAGGED \
            --query "imageDetails[?imageTags[?ends_with(@, '-${SHORT_SHA}')]].imageDigest | [0]" \
            --output json | tr -d '"')

          if [[ -z "$BACKEND_DIGEST" || "$BACKEND_DIGEST" == "null" || \
                -z "$FRONTEND_DIGEST" || "$FRONTEND_DIGEST" == "null" ]]; then
            echo "Could not find both published images for commit ${CI_SHA}"
            exit 1
          fi

          echo "BACKEND_IMAGE=${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ vars.AWS_REGION }}.amazonaws.com/cwvj-mega-backend@${BACKEND_DIGEST}" >> "$GITHUB_ENV"
          echo "FRONTEND_IMAGE=${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ vars.AWS_REGION }}.amazonaws.com/cwvj-mega-frontend@${FRONTEND_DIGEST}" >> "$GITHUB_ENV"

      - name: Configure kubectl
        run: |
          aws eks update-kubeconfig \
            --name "${{ vars.EKS_CLUSTER_NAME }}" \
            --region "${{ vars.AWS_REGION }}"

      - name: Prepare PROD manifests
        run: |
          sed -i \
            -e "s|__BACKEND_IMAGE__|${BACKEND_IMAGE}|g" \
            -e "s|__DB_NAME__|${{ vars.DB_NAME }}|g" \
            kubernetes/prod/backend.yaml

          sed -i \
            -e "s|__FRONTEND_IMAGE__|${FRONTEND_IMAGE}|g" \
            kubernetes/prod/frontend.yaml

      - name: Deploy to PROD
        run: kubectl apply -f kubernetes/prod/

      - name: Wait for deployments
        run: |
          kubectl rollout status deployment/cwvj-backend -n prod --timeout=5m
          kubectl rollout status deployment/cwvj-frontend -n prod --timeout=5m

      - name: Show deployed resources
        run: kubectl get deployments,pods,services -n prod -o wide

      - name: Smoke test PROD frontend
        run: |
          kubectl port-forward -n prod service/cwvj-frontend 18080:80 &
          PORT_FORWARD_PID=$!
          trap 'kill "$PORT_FORWARD_PID" 2>/dev/null || true' EXIT

          curl --fail --silent --show-error \
            --retry 10 --retry-connrefused \
            --max-time 10 \
            http://127.0.0.1:18080/
```

The steps mirror DEV (topic 51). **Only the differences matter:**

| Difference | Why |
|---|---|
| `on: pull_request: types: [closed], branches: [main]` | Fires when a PR **into `main`** is closed, whether merged **or just closed**… |
| `if:` ① `merged == true` | …so only real merges deploy. |
| ② `user.login == 'github-actions[bot]'` | Only the **automated** promotion PRs from Stage 12, not PRs people opened by hand. |
| ③ `contains(body, head.sha)` | The PR body records the commit that passed DEV. If someone pushed a newer, **undeployed** commit to the branch, `head.sha` changes and isn't in the body, so **no PROD deploy** until that commit passes DEV and the body is updated. |
| `environment: prod` | `prod` variables (`AWS_EKS_DEPLOY_ROLE_ARN_PROD`, `DB_NAME=cwvj_prod_db`), the **required reviewer** (Stage 14), and the OIDC `sub` `…:environment:prod`, which only the PROD role trusts. |
| `timeout-minutes: 20`, group `cwvj-deploy-prod` | Its own concurrency lane, never cancelled halfway (the default is 1 running + 1 pending). |
| **Two SHAs** | `merge_commit_sha` = the squash commit on `main`, used for the **manifests** (the PROD source of truth). `head.sha` = the original branch commit, used to **find the images** that CI built and DEV ran. With squash merges these are always different. |
| PROD role and `kubernetes/prod/`, `-n prod` | A separate AWS identity → separate K8s group → RBAC only in `prod` (Stage 10). The DEV role **can't** deploy here. |

```text
DEV deploy ✓ + smoke ✓ → bot PR (body has the SHA) → human review + checks → squash merge
 → 04-deploy-prod (merged ✓, bot ✓, SHA in body ✓) → waits for the prod reviewer
 → OIDC PROD role → same ECR digests → kubectl apply -n prod → rollout ✓ → smoke ✓
```

The smoke test checks that the frontend answers over HTTP. Backend and DB behaviour need application-level tests, and a real release should also test through the actual ingress or load balancer.

---

## Hands-on lab

1. Create the PROD database and `04-deploy-prod.yaml`.
2. In Stage 14 you'll merge a promotion PR and **approve the deployment as the second account**, then confirm `kubectl get pods -n prod -o jsonpath='{..image}'` shows the **same digests** as `-n dev`.
3. Close a PR **without merging** and confirm the PROD job is **skipped**.
4. Open and merge a PR created by a human (not the bot) and confirm it's skipped (bot-only condition).

## Where you'll use it

- Production CD with change control: PR approval + environment approval + immutable artifact promotion.

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| PROD job skipped after merge | A condition is false (not the bot's PR, or the body lacks `head.sha` because the branch advanced). | Merge the latest bot-updated PR. Check the body SHA. |
| `Branch "…" is not allowed to deploy to prod` | The environment allows `main` only. | Add `refs/pull/*/merge` (Stage 14, topic 54). |
| Stuck "Waiting for review" | No eligible reviewer approved. Self-review blocked. | The other account approves under **Review deployments**. |
| Images not found | Looking up by the merge SHA instead of `head.sha`. | Use `pull_request.head.sha` for images. |

## Quick recap

- `pull_request: closed` + merged + bot author + SHA-in-body → PROD job.
- `environment: prod` with an independent reviewer (prevent self-review), OIDC PROD role on `environment:prod`.
- Manifests from `merge_commit_sha`, images (digests) from `head.sha`, the same artifact as DEV.

**Interview questions**

1. **Q: How does the capstone make sure PROD deploys exactly what DEV tested?**
   A: It deploys the images identified by the PR's head SHA (recorded in the PR body by the DEV pipeline), by digest.
2. **Q: Why separate the merge approval from the deployment approval?**
   A: Code review and release authorization are different decisions, often made by different people (segregation of duties).
3. **Q: Why does the PROD environment allow `refs/pull/*/merge` rather than `main`?**
   A: Jobs triggered by `pull_request` events run on the PR merge ref, and that's what the environment branch policy evaluates.
