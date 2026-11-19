# 49 · Capstone 07 · Publish and Promote Images to Amazon ECR with OIDC (Stage 9)

> **Part H: Capstone** · Previous: [48 · Containerize and Scan the Application (Stage 8)](48-capstone-06-containerize-and-scan.md) · Next: [50 · Allow GitHub Actions to Deploy to EKS (Stage 10)](50-capstone-08-eks-deploy-access.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

**Stage 9** publishes the scanned images to **Amazon ECR** (Elastic Container Registry): create two ECR repos with **immutable tags**, register **GitHub's OIDC provider** in IAM, create the **ECR push role** (trust: this repo's feature/bugfix/hotfix branches) and a **least-privilege push policy**, then add `backend-ecr-push-job` / `frontend-ecr-push-job` that load the `.tar`, assume the role via OIDC, `docker login` to ECR, tag and push.

## Why it exists

A registry is where deployable artifacts live. **OIDC** means no AWS keys in GitHub (topic 36). **Immutable tags** guarantee a tag always means the same image, which is the foundation of "promote the same artifact".

## Analogy

The sealed shipping container (scanned image) goes to the **bonded warehouse** (ECR). The truck driver shows a **one-shift ID** (OIDC → temporary role credentials), and the warehouse only accepts deliveries for **two named bays** (permissions policy). Once a bay label (tag) is used, it **can never be reused** for a different container (immutable tags).

## How it works inside

```text
backend-ecr-push-job (needs: backend-image-build-scan-job; permissions: id-token: write)
  download-artifact (.tar) → docker load
  aws-actions/configure-aws-credentials@v6 (role-to-assume: vars.AWS_ROLE_ARN) ── OIDC JWT ──► STS
       trust policy: aud = sts.amazonaws.com, sub ~ repo:Owner@id/repo@id:ref:refs/heads/feature/*
  aws ecr get-login-password | docker login --password-stdin <acct>.dkr.ecr.<region>.amazonaws.com
  docker tag cwvj-mega-backend:<tag> <registry>/cwvj-mega-backend:<tag>     (same tag as the scanned image)
  docker push → ECR (tag immutable; digest sha256:… recorded)
```

- `permissions: id-token: write` **only** allows requesting the token. AWS access comes from the **trust policy** (who) + **permissions policy** (what).
- No `contents: read` is needed, because there's no checkout (topic 18).
- Files: [`code/capstone/infrastructure/CWVJGitHubActionsECRPushRole.json`](code/capstone/infrastructure/CWVJGitHubActionsECRPushRole.json), [`…ECRPushPolicy.json`](code/capstone/infrastructure/CWVJGitHubActionsECRPushPolicy.json).
- The `sub` values use the **immutable subject format** with numeric owner and repo IDs. Copy **your** IDs from a printed token (topic 36), not the original's.

## Workflow YAML / commands: build it step by step

### Stage 9: Publish the exact scanned images to ECR

**Goal:** push the images that **passed Trivy**, without rebuilding them. This is "build once, promote many": DEV and PROD will later deploy these same images.

### Prepare AWS

#### 1. ECR repositories

**ECR → Repositories → Create repository**, twice:

| Setting | Value |
|---|---|
| Names | `cwvj-mega-backend`, `cwvj-mega-frontend` |
| Visibility | Private |
| **Image tag mutability** | **Immutable** |
| Encryption | AES-256 (default) |

Tags like `0.0.1-SNAPSHOT-8048730` are **unique per commit by convention**, and **immutable because ECR enforces it**. Re-running the workflow for the same commit makes the same tag, so the push **fails instead of overwriting**. That's intentional; handle it explicitly rather than turning immutability off. A tag is a name. The **digest** (`sha256:…`) is the exact content, and it's what DEV and PROD deploy later.

#### 2. GitHub as an OIDC identity provider

**IAM → Identity providers → Add provider:** type **OpenID Connect**, URL `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com` ([GitHub docs](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)). This only **registers trust**. It grants nothing by itself.

```text
job (permissions: id-token: write) ──► GitHub OIDC provider issues a JWT (aud, sub = repo + ref …)
configure-aws-credentials ──► STS AssumeRoleWithWebIdentity(role-to-assume, JWT)
STS checks the role's TRUST policy (aud == sts.amazonaws.com, sub matches) ──► temporary credentials
AWS CLI / docker use them ──► allowed only what the role's PERMISSIONS policy grants
```

> You'll also see `oidc.eks.ap-south-1.amazonaws.com/…` in IAM. That's the **EKS** provider (from `withOIDC: true`), used by **pods** via IRSA. The GitHub provider is used by **workflows**. Both are federation, for different identity sources (topic 36).

#### 3. Permissions policy `CWVJGitHubActionsECRPushPolicy`

[`code/capstone/infrastructure/CWVJGitHubActionsECRPushPolicy.json`](code/capstone/infrastructure/CWVJGitHubActionsECRPushPolicy.json):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuthentication",
      "Effect": "Allow",
      "Action": ["ecr:GetAuthorizationToken"],
      "Resource": "*"
    },
    {
      "Sid": "PushApplicationImages",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:UploadLayerPart"
      ],
      "Resource": [
        "arn:aws:ecr:ap-south-1:<ACCOUNT_ID>:repository/cwvj-mega-backend",
        "arn:aws:ecr:ap-south-1:<ACCOUNT_ID>:repository/cwvj-mega-frontend"
      ]
    }
  ]
}
```

`GetAuthorizationToken` **can't** be scoped to a repository, so it needs `"Resource": "*"`. The upload actions are scoped to the two repos. That's least privilege.

#### 4. Role `CWVJGitHubActionsECRPushRole` (custom trust policy), with the policy above attached

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
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:<owner>@<owner-id>/mega-project-demo@<repo-id>:ref:refs/heads/feature/*",
            "repo:<owner>@<owner-id>/mega-project-demo@<repo-id>:ref:refs/heads/bugfix/*",
            "repo:<owner>@<owner-id>/mega-project-demo@<repo-id>:ref:refs/heads/hotfix/*"
          ]
        }
      }
    }
  ]
}
```

- **Copy your subject prefix** from **repo → Settings → Actions → OIDC** ("Default subject claim prefix"). This repo's prefix includes the **immutable owner and repo IDs** (`name@id`), so a renamed or re-created repo with the same name can't match. If you build the `sub` from the name alone, the role assumption fails with "Not authorized to perform sts:AssumeRoleWithWebIdentity".
- Only **this repo's development branches** can assume the role. Trust policy = **who** may assume it; permissions policy = **what** it can do.

### Image job changes (backend shown; the frontend is identical with `frontend` names)

At **job level** in `backend-image-build-scan-job` (beside `runs-on`, not inside `steps:`):

```yaml
    outputs:
      image-tag: ${{ steps.image-tag.outputs.image_tag }}
```

**After the Trivy step**, save and upload the exact image that was scanned:

```yaml
      # Save the exact image that passed the security scan as a Docker archive.
      - name: Save backend container image
        run: |
          docker save \
            -o cwvj-mega-backend.tar \
            cwvj-mega-backend:${{ steps.image-tag.outputs.image_tag }}

      # Upload the exact scanned image so a later job can use the same image.
      - name: Upload backend container image
        uses: actions/upload-artifact@v7
        with:
          name: backend-container-image
          path: cwvj-mega-backend.tar
          if-no-files-found: error
```

These run only if the scan passed, so only a **scanned** image travels on. **Small metadata (the tag) goes as a job output; files (the `.tar`) go as artifacts** (topics 21 and 24). The `.tar` lands in `$GITHUB_WORKSPACE`, hence the bare filename. The artifact name (`backend-container-image`) isn't the file name.

### The push job

```yaml
  backend-ecr-push-job:
    name: Push Backend Image to Amazon ECR
    runs-on: ubuntu-26.04
    needs: backend-image-build-scan-job

    permissions:
      id-token: write

    steps:
      - name: Download backend container image
        uses: actions/download-artifact@v8
        with:
          name: backend-container-image
          path: .

      # Load the exact Docker image produced and scanned by the previous job.
      - name: Load backend container image
        run: docker load -i cwvj-mega-backend.tar

      # Verify that the expected image and tag are available.
      - name: Verify backend container image
        env:
          IMAGE_TAG: ${{ needs.backend-image-build-scan-job.outputs.image-tag }}
        run: docker images cwvj-mega-backend:${IMAGE_TAG}

      # Authenticate to AWS using GitHub OIDC and the IAM role created earlier.
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      # Authenticate Docker with the Amazon ECR registry.
      - name: Login to Amazon ECR
        run: |
          aws ecr get-login-password \
            --region "${{ vars.AWS_REGION }}" \
            | docker login \
                --username AWS \
                --password-stdin \
                "${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ vars.AWS_REGION }}.amazonaws.com"

      # Tag the local image with its ECR repository name while preserving the existing image tag.
      - name: Tag backend image for Amazon ECR
        env:
          IMAGE_TAG: ${{ needs.backend-image-build-scan-job.outputs.image-tag }}
        run: |
          ECR_REGISTRY="${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ vars.AWS_REGION }}.amazonaws.com"

          docker tag \
            "cwvj-mega-backend:${IMAGE_TAG}" \
            "${ECR_REGISTRY}/cwvj-mega-backend:${IMAGE_TAG}"

      # Push the exact image that was built and scanned earlier.
      - name: Push backend image to Amazon ECR
        env:
          IMAGE_TAG: ${{ needs.backend-image-build-scan-job.outputs.image-tag }}
        run: |
          ECR_REGISTRY="${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ vars.AWS_REGION }}.amazonaws.com"

          docker push \
            "${ECR_REGISTRY}/cwvj-mega-backend:${IMAGE_TAG}"
```

| Piece | Point |
|---|---|
| `needs: backend-image-build-scan-job` | A failed scan **skips** the push. `backend-job` is implied through the chain. |
| Job-level `permissions: id-token: write` | Lets the job **request** an OIDC token, nothing more. Every unlisted scope becomes `none`, and that's fine because this job never checks out code (topic 18). |
| `docker load` | Restores the scanned image with its original name and tag. No rebuild. |
| Verify step | `docker images` only *lists*, so a missing image prints nothing and still exits 0. For a strict check, use `docker image inspect "cwvj-mega-backend:${IMAGE_TAG}"`, which fails if it's missing. |
| `configure-aws-credentials@v6` | OIDC → STS → temporary credentials for later steps. It does **not** log Docker in. |
| `get-login-password \| docker login --password-stdin` | A separate **registry** login (user `AWS`, password via stdin so it never appears in argv). |
| `docker tag` then `docker push` | Adds the ECR name to the **same image** (no new layers) and pushes it with the **same tag** from upstream. |

**Repository variables** (**Settings → Secrets and variables → Actions → Variables**). These are configuration, not secrets, because OIDC removed the keys:

| Variable | Example |
|---|---|
| `AWS_REGION` | `ap-south-1` |
| `AWS_ACCOUNT_ID` | `<ACCOUNT_ID>` |
| `AWS_ROLE_ARN` | `arn:aws:iam::<ACCOUNT_ID>:role/CWVJGitHubActionsECRPushRole` |

**Frontend:** add the same `outputs:` block and the save/upload steps (`cwvj-mega-frontend.tar`, artifact `frontend-container-image`) to `frontend-image-build-scan-job`. Then copy the push job as `frontend-ecr-push-job` (`needs: frontend-image-build-scan-job`), replacing every `backend` with `frontend`. The same role and variables work because the policy covers both repos.

> ⚠️ Corrected: was an `if: github.event_name != 'pull_request'` line shown on the push jobs in the explanation and the frontend snippet but not in the backend YAML, and a verify step using `docker image inspect` in one place and `docker images` in another. Now both match the final [`01-ci.yaml`](code/capstone/workflow-yamls/01-ci.yaml): **no `if:`** (CI never runs on `pull_request` events, see topic 43) and `docker images <repo>:<tag>`, with the stricter alternative mentioned above.

### Push and verify

```bash
git branch --show-current          # feature/initial-ci
git add . && git commit -m "ci(backend-frontend): publish container images to Amazon ECR" && git push origin feature/initial-ci
```

Two chains, side by side: `backend-job → backend-image-build-scan-job → backend-ecr-push-job` and the same for the frontend. Check that each image passed Trivy before it was saved, that the push job assumed the role through OIDC, and that ECR shows the tags (for example `0.0.1-SNAPSHOT-8048730` and `0.1.0-8048730`) with their digests:

```bash
for r in cwvj-mega-backend cwvj-mega-frontend; do
  aws ecr describe-images --repository-name "$r" --region ap-south-1 \
    --query 'imageDetails[*].[imageTags,imageDigest,imagePushedAt]' --output table
done
```

| Failure | Check |
|---|---|
| `Configure AWS credentials` | The provider exists, `AWS_ROLE_ARN` is right, the trust `sub` matches your prefix and branch, `aud` is right, and `id-token: write` is set. |
| ECR login | `AWS_REGION`, `AWS_ACCOUNT_ID`, and `ecr:GetAuthorizationToken` on `*`. |
| Push denied | The repo exists in that account and region, and the upload actions are scoped to it. |
| `tag invalid: … already exists` | An immutable tag from an earlier run of the same commit. That's expected; don't disable immutability. |
| Artifact or load errors | The upload and download artifact names match, and the `.tar` name is right. |

---

## Hands-on lab

1. Prepare AWS, change the image jobs and add both push jobs. Then in ECR, see both images with tags ending in the short commit SHA.
2. Re-run the same CI run: the push fails because the **tag is immutable**. That's expected, not a bug.
3. Print your OIDC claims (topic 36) and confirm your trust policy's `sub` matches.
4. Temporarily remove `id-token: write` and read the error. Then restore it.

## Where you'll use it

- Every container pipeline that publishes to a cloud registry, done without stored keys.

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Trust `sub` mismatch (branch, repo ID format) or wrong role ARN. | Print the claims. Fix the policy. |
| `no basic auth credentials` on push | Docker not logged in to ECR (credentials ≠ Docker login). | Run the `get-login-password \| docker login` step. |
| `tag invalid: The image tag '…' already exists … and cannot be overwritten` | Immutable tags on a re-run. | Expected. Push from a new commit. **Don't disable immutability** to "fix" it. |
| `AccessDeniedException` on `ecr:InitiateLayerUpload` | Permissions policy missing an action or the repo ARN. | Fix the policy (least privilege, the right ARNs). |

## Quick recap

- ECR repos with immutable tags. A GitHub OIDC provider in IAM. A push role with a strict `sub` trust and a least-privilege push policy.
- Push jobs: load the scanned `.tar`, OIDC → AWS, `docker login` to ECR, tag and push the **same** image.
- Credentials ≠ Docker login. Immutable tags make re-pushes fail on purpose.

**Interview questions**

1. **Q: How does the workflow push to ECR without AWS keys?**
   A: OIDC: the job gets a GitHub-signed JWT, STS validates it against the role's trust policy and returns temporary credentials, then `docker login` to ECR with a token from `get-login-password`.
2. **Q: Why immutable tags?**
   A: A tag always refers to one specific image digest, so deployments and rollbacks are trustworthy and can't be silently overwritten.
3. **Q: What's the difference between the trust policy and the permissions policy?**
   A: Trust = who may assume the role (repo, branch, environment claims). Permissions = what the role may do once assumed.
