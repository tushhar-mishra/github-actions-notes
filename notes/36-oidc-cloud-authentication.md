# 36 · OIDC: Logging In to the Cloud Without Stored Keys

> **Part G: Security and production** · Previous: [35 · Publishing and versioning actions](35-publishing-and-versioning-actions.md) · Next: [37 · Security hardening](37-security-hardening.md) · [Back to INDEX](INDEX.md)
>
> Concept topic. The capstone's Stage 9 ("Create a GitHub OIDC Identity Provider", IAM role and trust policy) and Stage 10 (EKS deploy roles) apply it. Their full click-by-click AWS steps are in [topic 49](49-capstone-07-publish-images-to-ecr.md) and [topic 50](50-capstone-08-eks-deploy-access.md).

---

## What it is

**OIDC = OpenID Connect.** It's an open standard that lets one system **prove who it is** to another using a short-lived, signed **token** instead of a password.

In GitHub Actions:

1. Your job asks GitHub for an **OIDC token**: a **JWT** (JSON Web Token, a signed piece of JSON) that says, roughly, *"I am a workflow in repo `me/app`, running on branch `main`, for environment `prod`"*.
2. The job gives that token to the cloud (AWS STS, Azure AD, Google STS).
3. The cloud checks the signature and your **trust rules**, then hands back **temporary credentials** (valid for about an hour).

**No cloud keys are stored in GitHub at all.**

---

## Why it exists

The old way: create an AWS **IAM user**, generate **access keys** (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`), and store them as GitHub secrets. Problems:

- **Long-lived:** valid until someone rotates them, often forever.
- **Leakable:** a printed log, a malicious PR or a compromised action can steal them (that's exactly what the March 2026 trivy-action malware looked for).
- **Hard to scope:** the same key works from anywhere, any branch, any repo that has the secret.

OIDC fixes all three: credentials are **temporary**, **issued per job**, and only if the token's claims (**repo, branch, environment**) match the cloud's trust policy.

**Background (from the capstone):** AWS has two relevant identity types:

- **IAM users:** identities created in AWS IAM, with a **username and password** (console) and/or **access keys** (programmatic). These are **long-term credentials**.
- **IAM roles:** identities that authorized principals **assume** (IAM users, AWS services, **federated identities**). Roles have **no** long-term passwords or keys. When a role is assumed, **AWS STS** (Security Token Service) issues **temporary security credentials**.

**Federation** = letting identities managed **elsewhere** access a system without creating separate identities there:

- **SAML 2.0**: common for corporate workforce login.
- **OIDC**: common for applications and workloads, **including GitHub Actions**.

(These are common patterns, not exclusive rules. AWS supports both.)

**Jenkins comparison:** Jenkins usually stores AWS keys in its credential store, or runs on an EC2 instance with an **instance profile** role. OIDC gives GitHub-hosted runners the equivalent of an instance profile without owning the machine.

---

## Analogy

A **hotel key card** vs a **master key**:

- **Stored access keys** = giving the cleaner a **permanent master key**. If it's copied, every room is open forever.
- **OIDC** = the cleaner shows their **staff ID** (the OIDC token, signed by the agency) at reception. Reception checks the **list** ("agency X, staff on the 3rd floor, this shift only", the **trust policy**) and prints a **key card that expires at the end of the shift** (temporary credentials) for **specific rooms** (the permissions policy).

---

## How it works inside

```text
job (permissions: id-token: write)
  1. aws-actions/configure-aws-credentials asks GitHub's OIDC provider
     https://token.actions.githubusercontent.com  for a JWT with audience "sts.amazonaws.com"
     claims include:
       iss = https://token.actions.githubusercontent.com
       aud = sts.amazonaws.com
       sub = repo:OWNER/REPO:ref:refs/heads/main          (no environment)
          or repo:OWNER/REPO:environment:prod             (job uses environment: prod)
          or repo:OWNER/REPO:pull_request                 (PR runs)
       repository, ref, sha, workflow, actor, run_id, job_workflow_ref, environment …
  2. action calls AWS STS AssumeRoleWithWebIdentity(role-to-assume, JWT)
  3. STS: verify signature against GitHub's public keys (via the registered OIDC provider)
          check trust policy conditions on aud and sub
  4. OK → temporary AccessKeyId/SecretAccessKey/SessionToken (default ~1h) → exported as env vars
  5. later steps (aws, kubectl, docker login to ECR…) use them; IAM PERMISSIONS policy limits what they can do
```

The process (from the capstone):

1. **Request an OIDC token:** the workflow, with `id-token: write`, requests a JWT from GitHub's provider.
2. **Request role assumption:** the AWS credentials action presents the token to **AWS STS** for the role in `role-to-assume`.
3. **Validate the trust relationship:** STS validates the token and evaluates the role's **trust policy**, including the **audience (`aud`)** and **subject (`sub`)** conditions.
4. **Issue temporary credentials** if authorized.
5. **Authorize AWS operations:** the role's **permissions policies** decide which actions and resources are allowed.

> **Remember:** the **OIDC provider establishes trust**, the **role's trust policy controls who can assume the role**, the **permissions policies decide what the role can do**, and **STS issues temporary credentials**.

### The three AWS pieces

| Piece | What it is | Capstone example |
|---|---|---|
| **Identity provider** (once per AWS account) | Registers GitHub as trusted: URL `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com` | IAM → Identity providers → Add provider → OpenID Connect |
| **Role + trust policy** | **Who** may assume it (which repo, branch or environment) | `CWVJGitHubActionsECRPushRole`, `…EKSDeployRole` |
| **Permissions policy** | **What** it may do | `CWVJGitHubActionsECRPushPolicy` (push only to two ECR repos) |

> **Important:** registering the provider **grants nothing by itself**. Access comes only from roles whose trust policy matches the token.

> **Not to be confused with the EKS OIDC provider:** you may also see `oidc.eks.<region>.amazonaws.com/...` in IAM, created by `withOIDC: true` in the EKS cluster config. That one lets **pods in the cluster** get AWS permissions (**IRSA**, IAM Roles for Service Accounts). The GitHub one lets **workflows** assume roles.
>
> `EKS Pods → EKS OIDC → IAM Role → AWS Services`
> `GitHub Actions → GitHub OIDC → IAM Role → AWS Services`

---

## Workflow YAML / commands

### The trust policy (capstone ECR push role)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<YOUR_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": [
          "repo:<OWNER>@<OWNER_ID>/<REPO>@<REPO_ID>:ref:refs/heads/feature/*",
          "repo:<OWNER>@<OWNER_ID>/<REPO>@<REPO_ID>:ref:refs/heads/bugfix/*",
          "repo:<OWNER>@<OWNER_ID>/<REPO>@<REPO_ID>:ref:refs/heads/hotfix/*"
        ]
      }
    }
  }]
}
```

| Part | Meaning |
|---|---|
| `Principal.Federated` | Trust tokens from **this** registered GitHub OIDC provider in this AWS account. |
| `aud = sts.amazonaws.com` | The token must be meant for AWS STS. |
| `sub` patterns | Only these **repos and branches** may assume the role. `StringLike` allows `*`. |
| `Owner@146525273/repo@1386577280` | The newer **immutable subject** format that includes the numeric **owner ID** and **repository ID**. It stops someone who deletes and recreates a repo with the same name (or renames an org) from matching your policy. The classic format is `repo:OWNER/REPO:…`. **Print your token's claims (below) and copy the exact `sub`** before writing the policy. ⚠️ *Whether your repo gets the ID-based or the classic format depends on the repo/org OIDC settings: never assume, copy it from a real token.* |

The **EKS deploy role** in the capstone also allows `…:environment:dev` and `…:environment:prod`. When a job declares `environment:`, the `sub` claim becomes `repo:…:environment:<name>` **instead of** the `ref` form. That's how a cloud role can be locked to the **protected prod environment** (topic 28).

> ⚠️ **Never use `"sub": "repo:*"` or omit the `sub` condition.** That lets **any GitHub repo in the world** assume your role.

### The workflow side

```yaml
jobs:
  push-to-ecr:
    runs-on: ubuntu-26.04
    permissions:
      id-token: write          # allow requesting the OIDC JWT (topic 18)
      contents: read
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}     # arn:aws:iam::<acct>:role/CWVJGitHubActionsECRPushRole
          aws-region: ${{ vars.AWS_REGION }}           # ap-south-1

      - name: Who am I?
        run: aws sts get-caller-identity

      - name: Login to Amazon ECR (same as the capstone)
        run: |
          aws ecr get-login-password --region "${{ vars.AWS_REGION }}" \
            | docker login --username AWS --password-stdin \
                "${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ vars.AWS_REGION }}.amazonaws.com"
```

**Sample log output:**

```text
Run aws-actions/configure-aws-credentials@v6
  Assuming role with OIDC
  Authenticated as assumedRoleId AROA…:GitHubActions
Run aws sts get-caller-identity
{
    "UserId": "AROA…:GitHubActions",
    "Account": "<YOUR_ACCOUNT_ID>",
    "Arn": "arn:aws:sts::<YOUR_ACCOUNT_ID>:assumed-role/CWVJGitHubActionsECRPushRole/GitHubActions"
}
Run aws ecr get-login-password --region "ap-south-1" | docker login …
  Login Succeeded
```

- `assumed-role/…/GitHubActions`: temporary credentials for the role. **No secret was stored.**

### Debug: see your token's claims

```yaml
- name: Show OIDC claims (debug only)
  uses: actions/github-script@v8
  with:
    script: |
      const t = await core.getIDToken('sts.amazonaws.com');
      const claims = JSON.parse(Buffer.from(t.split('.')[1], 'base64url').toString());
      core.info(JSON.stringify({sub: claims.sub, aud: claims.aud, ref: claims.ref, environment: claims.environment}, null, 2));
```

(It prints only the claims, not the token itself. Never print the raw token.)

### Other clouds (same idea)

| Cloud | Action | Trust configured in |
|---|---|---|
| AWS | `aws-actions/configure-aws-credentials` | IAM OIDC provider + role trust policy |
| Azure | `azure/login` (with `client-id`, `tenant-id`, `subscription-id`) | Entra ID app → **Federated credentials** |
| Google Cloud | `google-github-actions/auth` (`workload_identity_provider`) | **Workload Identity Federation** pool and provider |
| HashiCorp Vault | `hashicorp/vault-action` (`method: jwt`) | Vault JWT auth role bound to claims |

---

## Hands-on lab

(Uses a free-tier AWS account. Delete the role when done.)

1. **IAM → Identity providers → Add provider → OpenID Connect**: URL `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com`.
2. Create role `gha-oidc-demo` with a trust policy limited to `repo:<you>/<repo>:ref:refs/heads/main` and **no** permissions policy.
3. Workflow: `id-token: write` + `configure-aws-credentials` + `aws sts get-caller-identity`. Run it on `main` → success.
4. Run it from a feature branch → `Not authorized to perform sts:AssumeRoleWithWebIdentity`. That's the trust policy working.
5. Add the claims-debug step and compare `sub` on `main` vs a job with `environment: prod`.
6. Remove `id-token: write` and read the error.

---

## Where you'll use it

- **Every cloud deploy:** ECR push and EKS deploy in the capstone (topics 49–53), Terraform plan/apply, S3 uploads.
- **Least privilege per stage:** separate roles for "push image" and "deploy to prod", each with its own trust (branch vs `environment:prod`).
- **Security reviews:** "do we have any long-lived cloud keys in GitHub secrets?" The goal answer is "no".

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Could not load credentials from any providers` / `Credentials could not be loaded` | No OIDC token: `id-token: write` missing. | Add it (job or workflow level). |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Trust policy `sub`/`aud` doesn't match the token (branch, environment, repo ID format), or the wrong role ARN. | Print the claims. Fix the `sub` pattern (ref vs environment form). |
| `No OpenIDConnect provider found in your account for https://token.actions.githubusercontent.com` | Identity provider not registered. | Add the provider. |
| Works on push, fails on fork PRs | Fork PRs can't get `id-token: write`. | Deploy only from trusted events. |
| Role assumed but `AccessDenied` on ECR/EKS | The **permissions** policy is too narrow, or the resource ARN is wrong. | Fix the permissions policy (trust ≠ permissions). |
| Session expires mid-job | Default ~1 h. | `role-duration-seconds` (within the role's max session). |

---

## Quick recap

- OIDC = a short-lived signed token (JWT) from GitHub → exchanged for **temporary cloud credentials**. **No stored keys.**
- Needs `permissions: id-token: write`.
- AWS pieces: **OIDC provider** (trust GitHub), **role trust policy** (which repo/branch/environment, via `aud` + `sub`), **permissions policy** (what it can do).
- The `sub` claim changes form with `environment:` (`…:environment:prod`), which lets you tie cloud access to protected environments.
- Never trust `repo:*`. Print the claims to debug.

**Interview questions**

1. **Q: Why use OIDC instead of storing AWS keys as GitHub secrets?**
   A: There are no long-lived secrets to leak or rotate. Credentials are temporary, per job, and scoped by trust conditions such as repo, branch and environment.
2. **Q: What does the IAM trust policy check?**
   A: The token issuer (via the federated provider), `aud = sts.amazonaws.com`, and the `sub` claim pattern (repo plus branch, environment or event).
3. **Q: How would you allow only production-approved jobs to deploy?**
   A: Put the deploy job in a protected `prod` environment, and have the cloud role trust only `sub = repo:org/app:environment:prod`.
