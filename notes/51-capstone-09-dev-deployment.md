# 51 · Capstone 09 · DEV Environment and Automated DEV Deployment (Stage 11)

> **Part H: Capstone** · Previous: [50 · Allow GitHub Actions to Deploy to EKS (Stage 10)](50-capstone-08-eks-deploy-access.md) · Next: [52 · Automated Pull Request Creation (Stage 12)](52-capstone-10-automated-pull-request.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

**Phase 4 / Stage 11** prepares **DEV** (an app user and database in RDS, connectivity from EKS to RDS, `dev`/`prod` namespaces, RBAC from [`kubernetes/deployer-rbac.yaml`](code/capstone/kubernetes/deployer-rbac.yaml), DB **Secrets**) and writes **`02-deploy-dev.yaml`**: triggered by **`workflow_run`** when CI completes successfully on a development branch, it assumes the DEV role via OIDC, finds the **image digests** for the CI commit in ECR, fills them into the DEV manifests, `kubectl apply`s, waits for rollout and **smoke-tests** through a port-forward.

## Why it exists

Continuous Delivery to DEV: every green CI run becomes a running, testable deployment automatically. Deploying **by digest** (`image@sha256:…`) guarantees DEV runs exactly what CI built and scanned. A **separate workflow** keeps CI and CD responsibilities and permissions apart.

## Analogy

When the inspection line finishes a car (CI success), a **dispatcher automatically sends it to the test track** (`workflow_run`). The dispatcher identifies the car by its **VIN** (digest), not by a paint colour that could be repainted (tag), parks it in the DEV lane (namespace) and does a **quick test drive** (smoke test).

## How it works inside

```text
01-ci.yaml completes (feature/**) ── workflow_run(completed) ──► 02-deploy-dev.yaml  (from DEFAULT branch!)
  concurrency: cwvj-deploy-dev (no cancel)        environment: dev      timeout-minutes: 15
  if: github.event.workflow_run.conclusion == 'success'
  permissions: contents: read, id-token: write
  1. checkout ref = workflow_run.head_sha                    ← the commit CI tested
  2. OIDC → AWS_EKS_DEPLOY_ROLE_ARN_DEV
  3. aws ecr describe-images … tag ends with -<short sha> → BACKEND_DIGEST / FRONTEND_DIGEST
       → echo "BACKEND_IMAGE=<registry>/cwvj-mega-backend@<digest>" >> $GITHUB_ENV
  4. aws eks update-kubeconfig
  5. sed placeholders __BACKEND_IMAGE__, __DB_NAME__ in kubernetes/dev/*.yaml (on the runner only)
  6. kubectl apply -f kubernetes/dev/ → rollout status (5m) → get pods/services
  7. port-forward svc/cwvj-frontend 18080:80 & → curl --fail --retry … → trap kills port-forward
  8. job create-pull-request-job: uses ./.github/workflows/03-create-pr.yaml  (Stage 12)
```

Kubernetes recap: a **namespace** isolates resources. A **Deployment** keeps N pods of an image running and does rolling updates. A **Service** gives pods a stable address. A **Secret** stores sensitive config (here DB credentials, created outside the pipeline). **RBAC** = Role/ClusterRole (what) + RoleBinding (who, where).

Workflow topics used: 31 (`workflow_run`, default-branch requirement, `head_sha`), 29 (concurrency), 28 (environment), 36 (OIDC), 23 (`$GITHUB_ENV`), 22 (reusable workflow call), 04 (`set -euo pipefail`, `trap`).

## Workflow YAML / commands: build it step by step

### Stage 11: Prepare DEV and write the DEV deploy workflow

> **Don't push or run this yet.** Stages 11–13 write the DEV deploy, PR creation and PROD deploy workflows first. Stage 14 (topic 54) pushes them all together and runs the end-to-end flow.

#### 1. The DEV database (from the EC2 box, via `psql`)

| Item | Value |
|---|---|
| RDS instance / port | `cwvj-mega-postgres` / 5432 |
| Database | `cwvj_dev_db` |
| App user (owns the DB, **not** the master account) | `cwvj_dev_app_user` |

```bash
psql --host=<YOUR_RDS_ENDPOINT> --port=5432 --username=postgres --dbname=postgres --password
```

```sql
SELECT current_user;                                  -- the master user (postgres here)
CREATE ROLE cwvj_dev_app_user LOGIN;
\password cwvj_dev_app_user                           -- typed interactively: never in history or Git
GRANT cwvj_dev_app_user TO postgres;                  -- temporary: lets the master create a DB owned by the app user
CREATE DATABASE cwvj_dev_db OWNER cwvj_dev_app_user;
REVOKE cwvj_dev_app_user FROM postgres;               -- remove the temporary membership
\connect cwvj_dev_db
GRANT USAGE, CREATE ON SCHEMA public TO cwvj_dev_app_user;   -- Flyway creates the tables at app start-up
\conninfo
\l cwvj_dev_db
\du cwvj_dev_app_user
\q
```

If your master user isn't `postgres`, change it in the `GRANT` and `REVOKE` lines too.

#### 2. Let EKS pods reach RDS

**EC2 → Security Groups → `cwvj-rds-sg` → Inbound → Add rule:** PostgreSQL / 5432 / source = the **EKS cluster security group** (`eks-cluster-sg-cwvj-mega-demo-eks-<suffix>`; your suffix differs). Use an SG reference, never `0.0.0.0/0`.

> **Trade-off:** any pod using the node SG can now *reach* 5432. It still needs the DB **password** (in the backend's Secret) to log in. **Security Groups for Pods** would narrow network access to just the backend pods. That's skipped here to keep the focus on Actions.

#### 3. Namespaces, RBAC and Secrets (run once, as a cluster admin)

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl apply -f kubernetes/deployer-rbac.yaml
kubectl get namespaces dev prod
kubectl get clusterroles cwvj-github-actions-deployer-cluster-role
kubectl get rolebindings -n dev
kubectl get rolebindings -n prod
```

`deployer-rbac.yaml` = **one ClusterRole** (the deploy permissions, defined once) plus **two RoleBindings**: `dev` → group `cwvj-github-actions-deployer-dev`, and `prod` → group `…-prod`. A **RoleBinding** limits a ClusterRole to its own namespace. The ClusterRole deliberately **can't create or edit Secrets**, so database credentials stay outside the deploy identity.

**Prove the isolation** (impersonation needs admin rights):

```bash
kubectl auth can-i create deployments --as=t --as-group=cwvj-github-actions-deployer-dev  -n dev    # yes
kubectl auth can-i create deployments --as=t --as-group=cwvj-github-actions-deployer-prod -n prod   # yes
kubectl auth can-i create deployments --as=t --as-group=cwvj-github-actions-deployer-dev  -n prod   # no
kubectl auth can-i create deployments --as=t --as-group=cwvj-github-actions-deployer-prod -n dev    # no
```

RBAC is **additive**, so check an identity's *effective* rights, not one binding. You can run all of this because EKS makes the **cluster creator** a cluster-admin by default (`bootstrapClusterCreatorAdminPermissions`), which is separate from IAM admin. Run `aws sts get-caller-identity` and `kubectl config get-contexts` to see who you are.

**Database Secrets.** Put the real passwords into `dev-backend-db-secret.yaml` and `prod-backend-db-secret.yaml` (replacing `PASSWORD-HERE`), apply them, then **put the placeholders back before committing**:

```bash
kubectl apply -f dev-backend-db-secret.yaml
kubectl apply -f prod-backend-db-secret.yaml
kubectl get secret backend-db-secret -n dev
kubectl get secret backend-db-secret -n prod
```

To avoid the password ever touching a file:

```bash
read -r -s -p "Enter DEV database password: " DB_PASSWORD; printf '\n'
kubectl create secret generic backend-db-secret --namespace=dev \
  --from-literal=password="$DB_PASSWORD" --dry-run=client -o yaml | kubectl apply -f -
unset DB_PASSWORD
```

(`--from-literal` briefly shows up in the process list. In production, use **AWS Secrets Manager + ASCP / Secrets Store CSI** or Vault.)

#### 4. GitHub configuration this workflow reads

> 🆕 Added: the workflow below reads these variables, but the original steps never said where to create them.

| Where | Name | Value |
|---|---|---|
| Repository variable | `AWS_REGION`, `AWS_ACCOUNT_ID` | already set in Stage 9 |
| Repository variable | `EKS_CLUSTER_NAME` | `cwvj-mega-demo-eks` |
| **`dev` environment** variable | `AWS_EKS_DEPLOY_ROLE_ARN_DEV` | `arn:aws:iam::<ACCOUNT_ID>:role/CWVJGitHubActionsEKSDeployDevRole` |
| **`dev` environment** variable | `DB_NAME` | `cwvj_dev_db` |
| **`prod` environment** variable | `AWS_EKS_DEPLOY_ROLE_ARN_PROD` / `DB_NAME` | the PROD role ARN / `cwvj_prod_db` (Stage 13) |

Because `DB_NAME` exists in **both** environments, the same expression `vars.DB_NAME` resolves differently per job, depending on `environment:` (topic 28).

#### 5. The workflow files

From now on, the CI file is **`01-ci.yaml`**: `git mv .github/workflows/ci.yaml .github/workflows/01-ci.yaml`. Its `name:` doesn't change, so `workflow_run` still matches it.

| File | Trigger | Job |
|---|---|---|
| `01-ci.yaml` | push to feature/bugfix/hotfix, dispatch | scan, build, test, Sonar, image scan, push to ECR |
| **`02-deploy-dev.yaml`** | `workflow_run` of CI completed | deploy those images to `dev` |
| `03-create-pr.yaml` | `workflow_call` (Stage 12) | open the promotion PR |
| `04-deploy-prod.yaml` | PR closed (merged) (Stage 13) | deploy the same digests to `prod` |

Separate files give each one its own triggers, permissions, concurrency and environments. **Security** still comes from permissions, environments, IAM trust and RBAC, not from the file split.

**`.github/workflows/02-deploy-dev.yaml`** ([final version](code/capstone/workflow-yamls/02-deploy-dev.yaml); Stage 12 adds a PR job to it):

```yaml
name: CWVJ Deploy | DEV

on:
  workflow_run:
    workflows:
      - "CWVJ CI | Build, Scan and Publish"
    types:
      - completed
    branches:
      - "feature/**"
      - "bugfix/**"
      - "hotfix/**"

concurrency:
  group: cwvj-deploy-dev
  cancel-in-progress: false

jobs:
  deploy-to-dev-job:
    name: Deploy to DEV
    runs-on: ubuntu-26.04
    timeout-minutes: 15
    environment: dev

    if: >-
      github.event.workflow_run.conclusion == 'success'

    permissions:
      contents: read
      id-token: write

    steps:
      - name: Check out deployment manifests
        uses: actions/checkout@v7
        with:
          ref: ${{ github.event.workflow_run.head_sha }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.AWS_EKS_DEPLOY_ROLE_ARN_DEV }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Find the published image digests
        env:
          CI_SHA: ${{ github.event.workflow_run.head_sha }}
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

      - name: Prepare DEV manifests
        run: |
          sed -i \
            -e "s|__BACKEND_IMAGE__|${BACKEND_IMAGE}|g" \
            -e "s|__DB_NAME__|${{ vars.DB_NAME }}|g" \
            kubernetes/dev/backend.yaml

          sed -i \
            -e "s|__FRONTEND_IMAGE__|${FRONTEND_IMAGE}|g" \
            kubernetes/dev/frontend.yaml

      - name: Deploy to DEV
        run: kubectl apply -f kubernetes/dev/

      - name: Wait for deployments
        run: |
          kubectl rollout status deployment/cwvj-backend -n dev --timeout=5m
          kubectl rollout status deployment/cwvj-frontend -n dev --timeout=5m

      - name: Show deployed resources
        run: kubectl get deployments,pods,services -n dev -o wide

      - name: Smoke test DEV frontend
        run: |
          kubectl port-forward -n dev service/cwvj-frontend 18080:80 &
          PORT_FORWARD_PID=$!
          trap 'kill "$PORT_FORWARD_PID" 2>/dev/null || true' EXIT

          curl --fail --silent --show-error \
            --retry 10 --retry-connrefused \
            --max-time 10 \
            http://127.0.0.1:18080/
```

**The trigger and the guards** (topics 10, 13, 29 and 31):

| Line | Why |
|---|---|
| `workflows: ["CWVJ CI \| Build, Scan and Publish"]` | Matches CI **by its `name:`**. Without it, any workflow could trigger deploys. |
| `types: [completed]` | Fires when CI ends, **including failed or cancelled runs**… |
| `if: … conclusion == 'success'` | …so this guard is essential. |
| `branches: feature/** …` | Only development-branch CI runs deploy to DEV. A manual CI run on `main` won't. |
| Must exist on the **default branch** | `workflow_run` workflows are read from `main`. That's why Stage 14 has a bootstrap push. |
| `concurrency: cwvj-deploy-dev`, `cancel-in-progress: false` | Never interrupt a deploy halfway. The default keeps 1 running + 1 pending, and a newer pending run replaces the older one. Use `queue: max` if every deploy must run. |
| `timeout-minutes: 15` | A stuck rollout can't burn 6 hours. |
| `environment: dev` | Branch rules, `dev` variables, and the OIDC `sub` = `…:environment:dev` that the DEV role trusts (Stage 10). |
| `permissions: contents: read, id-token: write` | Once you list any permission, the others become `none`, so `contents: read` is needed for checkout. `id-token: write` only lets the job **request** the OIDC JWT. |

**The steps:**

| Step | Why |
|---|---|
| Checkout `ref: …workflow_run.head_sha` | A `workflow_run` run checks out the **default branch** by default. Pin it to the commit CI validated, so the manifests match the images even if the branch has moved on. |
| `CI_SHA` via `env:` | Keep `${{ }}` out of shell scripts. A SHA is safe, but branch names and PR titles aren't (topic 37). |
| `set -euo pipefail` | Stop on any failure, on unset variables, or on a failed pipe. |
| JMESPath `imageTags[?ends_with(@, '-<sha7>')] … imageDigest \| [0]` | Find the image whose tag ends with this commit's short SHA, and return its **digest**. |
| Empty or `null` check → `exit 1` | Never deploy only half of the app. |
| `>> "$GITHUB_ENV"` | `BACKEND_IMAGE=…/cwvj-mega-backend@sha256:…` is available to **later** steps, not the current one (topic 23). |
| `aws eks update-kubeconfig` | Writes `~/.kube/config` using `eks:DescribeCluster` (Stage 10). |
| `sed -i "s\|__BACKEND_IMAGE__\|…\|g"` | Uses `\|` as the delimiter because image refs contain `/`. The files change **only on the runner**. Credentials stay in the Secret; only the DB name is injected. |
| `kubectl apply` → `rollout status --timeout=5m` | Declarative apply, then **wait** until the rollout really finishes, or fail. |
| Port-forward + `curl --fail --retry 10 --retry-connrefused --max-time 10` | A real HTTP check of the frontend without exposing it. `trap … EXIT` always kills the background forward. (It doesn't test the backend or DB; that's Stage 14.) |

**Tag vs digest:** the tag **finds** the image, and the digest **pins** its exact content. DEV and PROD both deploy `@sha256:…`, which is build once, promote the same artifact.

> **Optional hardening:** to block manual CI runs on `main`, add `if: github.event_name != 'workflow_dispatch' || github.ref != 'refs/heads/main'` to the CI jobs. To limit **who** can trigger workflows, use **Settings → Actions → Policies → workflow execution protections** (restrict actors and events, per workflow file). Be careful: an actor restriction on the CI file also blocks developers' pushes. Put manual runs in a separate, restricted workflow that calls a reusable CI ([docs](https://docs.github.com/en/actions/how-tos/administer/control-workflow-execution)).

---

## Hands-on lab

1. Prepare the database, connectivity, namespaces, RBAC and Secrets, and verify with the four `kubectl auth can-i … --as-group` checks above.
2. Create the variables and `02-deploy-dev.yaml`. Remember it only triggers once it's on **`main`** (Stage 14 bootstraps that).
3. After a deploy: `kubectl get pods -n dev -o jsonpath='{..image}'` shows `…@sha256:…` digests, not tags.
4. Break the smoke test (wrong port) and see the job fail **after** deployment. Think about how you'd add automatic rollback (`kubectl rollout undo`).

## Where you'll use it

- CD to a dev/test cluster on every green CI. Deploy by digest. CI and CD as separate workflows.

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| DEV workflow never runs | Not on the default branch, or `workflows:` name mismatch. | Stage 14 bootstrap. Match `name:` exactly (topic 31). |
| `Could not find both published images for commit …` | The CI push jobs didn't run or failed, or the tag suffix differs. | Check the ECR tags end with `-<short-sha>`. |
| `rollout status … timed out` | Pods crash or don't become ready (image pull, DB secret, probes). | `kubectl describe pod`, `kubectl logs`. Check the Secret and RDS connectivity. |
| `Forbidden` from kubectl | RBAC/access entry mismatch. | Topic 50 and the RBAC file. |
| Smoke test `connection refused` | Port-forward not ready. | The `--retry-connrefused` loop (as in the workflow). Check the Service port. |

## Quick recap

- DEV prepared: DB user and DB, EKS↔RDS connectivity, namespaces, RBAC, Secrets (placeholders in Git, real values applied manually).
- `02-deploy-dev.yaml`: `workflow_run` on CI success → OIDC → find **digests** → fill manifests → `kubectl apply` → rollout → smoke test → call the PR workflow.
- Deploys exactly what CI built (`head_sha` + digest). One DEV deploy at a time (concurrency).

**Interview questions**

1. **Q: How does the DEV workflow know which images to deploy?**
   A: It takes `workflow_run.head_sha`, finds the ECR images whose tag ends with that short SHA, and deploys them by digest.
2. **Q: Why deploy by digest instead of tag?**
   A: A digest is the content hash. It can't change, so DEV (and later PROD) run exactly the scanned image.
3. **Q: Why is the DEV deploy a separate workflow from CI?**
   A: Different triggers, permissions and governance (environment, AWS role). It also keeps CI fast and focused, and lets CD run from the trusted default-branch definition.
