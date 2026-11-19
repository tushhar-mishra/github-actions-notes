# 50 · Capstone 08 · Allow GitHub Actions to Deploy to EKS (Stage 10)

> **Part H: Capstone** · Previous: [49 · Publish and Promote Images to Amazon ECR with OIDC (Stage 9)](49-capstone-07-publish-images-to-ecr.md) · Next: [51 · DEV Environment and Automated DEV Deployment (Stage 11)](51-capstone-09-dev-deployment.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

**Phase 3 / Stage 10** gives the future deployment workflows access to EKS **in two layers**: **AWS IAM** (a read-only metadata policy + separate **DEV** and **PROD deploy roles** whose trust is tied to `environment:dev` / `environment:prod` OIDC subjects) and **Kubernetes** (EKS **access entries** that map each IAM role to a Kubernetes **group**, later bound to namespace permissions with RBAC in Stage 11).

## Why it exists

Deploying to Kubernetes needs **two** permissions: AWS must let the workflow call EKS APIs (for example to build a kubeconfig), and Kubernetes must let that identity create Deployments in the right **namespace only**. Separate DEV/PROD roles mean a DEV job can never touch PROD.

## Analogy

Getting into a **secure office tower**: the **building pass** (IAM role, issued only to people from the right department: the environment claim) gets you through the lobby. Your **floor badge** (access entry → Kubernetes group → RBAC) decides which **floor** (namespace) you can work on. DEV staff badges don't open the PROD floor.

## How it works inside

```text
deploy job (environment: dev) → OIDC sub = repo:…:environment:dev
   → assumes CWVJ…EKSDeployRole-DEV (trust: environment:dev) → AWS creds
   → aws eks update-kubeconfig  (needs eks:DescribeCluster from the metadata policy)
   → kubectl apply … → EKS API authenticates the IAM role via ACCESS ENTRY
          access entry: role → Kubernetes group "cwvj-dev-deployers"
   → Kubernetes RBAC: RoleBinding in namespace dev grants that group deploy rights (Stage 11)
```

- **Access entries vs the old `aws-auth` ConfigMap:** access entries are managed through the EKS API (auditable, no YAML editing in `kube-system`). Check the cluster's **authentication mode** (`API` or `API_AND_CONFIG_MAP`) first.
- An access entry **maps** an identity to groups. It **grants nothing** until RBAC binds the group.
- The environment-based `sub` is why the trust policy lists `…:environment:dev` / `…:environment:prod` (topics 28, 36).
- File: [`code/capstone/infrastructure/CWVJGitHubActionsEKSDeployPolicy.json`](code/capstone/infrastructure/CWVJGitHubActionsEKSDeployPolicy.json), roles `…EKSDeployRole.json`.

## Workflow YAML / commands: build it step by step

### Stage 10: Let GitHub Actions deploy to EKS (identity only, no deploy yet)

Three separate layers, each answering one question:

| Layer | Question | Object |
|---|---|---|
| IAM **trust policy** | *Which workflow* may become this AWS role? | `…DevRole` trusts `environment:dev`, `…ProdRole` trusts `environment:prod` |
| IAM **permissions policy** | *What AWS APIs* may the role call? | `eks:DescribeCluster`, `ecr:DescribeImages` (read-only) |
| **EKS access entry + Kubernetes RBAC** | *What may it do inside the cluster?* | role → K8s group → `Role`/`RoleBinding` in **one** namespace |

> Deploying straight from Actions is the starting point here. GitOps with Argo CD (Actions updates Git, Argo CD applies it) is covered in a later video.

#### 1. Shared permissions policy `CWVJGitHubActionsEKSDeployPolicy`

[`code/capstone/infrastructure/CWVJGitHubActionsEKSDeployPolicy.json`](code/capstone/infrastructure/CWVJGitHubActionsEKSDeployPolicy.json):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DescribeEKSCluster",
      "Effect": "Allow",
      "Action": "eks:DescribeCluster",
      "Resource": "arn:aws:eks:ap-south-1:<ACCOUNT_ID>:cluster/cwvj-mega-demo-eks"
    },
    {
      "Sid": "ReadPublishedECRImageMetadata",
      "Effect": "Allow",
      "Action": "ecr:DescribeImages",
      "Resource": [
        "arn:aws:ecr:ap-south-1:<ACCOUNT_ID>:repository/cwvj-mega-backend",
        "arn:aws:ecr:ap-south-1:<ACCOUNT_ID>:repository/cwvj-mega-frontend"
      ]
    }
  ]
}
```

`DescribeCluster` lets `aws eks update-kubeconfig` build the connection. `DescribeImages` resolves **tag → digest** before deploying. This grants **no** Kubernetes rights, and it can't push images: that's the separate `ECRPushRole` from Stage 9, so responsibilities stay split.

#### 2. Two roles with environment-based trust

`CWVJGitHubActionsEKSDeployDevRole` (the PROD role is identical except `:environment:prod`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:<owner>@<owner-id>/mega-project-demo@<repo-id>:environment:dev"
        }
      }
    }
  ]
}
```

Attach `CWVJGitHubActionsEKSDeployPolicy` to **both** roles.

- A job that declares `environment: dev` gets a token whose `sub` ends in `:environment:dev`, **not** `:ref:…`. That's why these roles trust the **environment** (topic 36). Take the exact prefix from **Settings → Actions → OIDC**, and replace the account ID with yours.
- Environment trust doesn't restrict **branches** by itself. The environment's **deployment branch rules** (Stage 3) and protection rules do that. Together: only `feature/*|bugfix/*|hotfix/*` can use `dev`, and only `main` (plus a reviewer) can use `prod`.

#### 3. EKS access entries (role → Kubernetes group)

**Why access entries, not `aws-auth`?** Access entries are managed through the EKS API, console or CLI instead of a hand-edited ConfigMap, and they're the recommended way for new setups. In `API_AND_CONFIG_MAP` mode, if one role is mapped in both, the **access entry wins** ([AWS migration docs](https://docs.aws.amazon.com/eks/latest/userguide/migrating-access-entries.html)).

**Check the mode:** EKS → `cwvj-mega-demo-eks` → **Access** shows `API_AND_CONFIG_MAP` (both work, so no change is needed). `API` means access entries only, and `CONFIG_MAP` means `aws-auth` only. Once the access-entry API is enabled, you can't switch back to a mode without it.

**Access → Access entries → Create access entry**, twice:

| IAM principal ARN | Type | Kubernetes group |
|---|---|---|
| `arn:aws:iam::<ACCOUNT_ID>:role/CWVJGitHubActionsEKSDeployDevRole` | Standard | `cwvj-github-actions-deployer-dev` |
| `arn:aws:iam::<ACCOUNT_ID>:role/CWVJGitHubActionsEKSDeployProdRole` | Standard | `cwvj-github-actions-deployer-prod` |

Don't associate an AWS-managed access policy. Permissions come from **our own RBAC** (`kubernetes/deployer-rbac.yaml`, applied in Stage 11), where the group names must match **exactly**. An access entry by itself grants **nothing** inside the cluster; it only gives the role a Kubernetes identity.

---

## Hands-on lab

1. Create the policy, both roles and both access entries. Then: `aws eks list-access-entries --cluster-name <name>` shows both roles.
2. `aws eks describe-cluster --name <name> --query cluster.accessConfig` shows the authentication mode.
3. (After Stage 11 RBAC) use `kubectl auth can-i create deployments -n prod --as-group cwvj-github-actions-deployer-dev --as x`, which should be **no**.

## Where you'll use it

- Every GitHub Actions → EKS integration. The same two-layer idea applies to AKS/GKE (cloud IAM + Kubernetes RBAC).

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `error: You must be logged in to the server (Unauthorized)` | No access entry for the role (or a wrong ARN). | Create the access entry for the exact role ARN. |
| `Forbidden: … cannot create resource "deployments" in namespace "dev"` | Access entry OK, RBAC missing. | Stage 11 RoleBinding for the group. |
| `AccessDeniedException: eks:DescribeCluster` | IAM metadata policy not attached. | Attach the policy to both roles. |
| Assume-role fails for the deploy job | Job lacks `environment: dev`, so the `sub` is the ref form, not the environment form. | Add `environment:` or adjust the trust. |

## Quick recap

- Two layers: **IAM** (OIDC trust per environment + DescribeCluster permissions) and **Kubernetes** (access entry → group → RBAC).
- Separate DEV/PROD roles trusted on `environment:dev` / `environment:prod`.
- Access entries replace `aws-auth`, and they grant nothing without RBAC.

**Interview questions**

1. **Q: Why does a GitHub job need both IAM and Kubernetes permissions to deploy to EKS?**
   A: IAM lets it authenticate and call the EKS APIs. Kubernetes RBAC decides what that authenticated identity may do inside the cluster.
2. **Q: How do you make sure the DEV workflow can't deploy to PROD?**
   A: Separate roles trusted on different environment `sub` claims, separate Kubernetes groups, and RoleBindings scoped to each namespace.
3. **Q: What replaced the `aws-auth` ConfigMap?**
   A: EKS access entries (the API-managed mapping of IAM principals to Kubernetes identities and groups).
