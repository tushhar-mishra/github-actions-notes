# 44 · Capstone 02 · Trivy Filesystem Security Scanning (Stage 5)

> **Part H: Capstone** · Previous: [43 · Project, Repository, Governance and First Run (Stages 1–4)](43-capstone-01-repo-foundation.md) · Next: [45 · Build and Validate the Backend and Frontend (Stage 6)](45-capstone-03-build-and-test.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

**Stage 5** replaces the foundation job with **`trivy-scan-job`**: **Trivy** (an open-source scanner by Aqua Security) scans the **checked-out repository filesystem** for **vulnerable dependencies, exposed secrets and misconfigurations**, and fails CI on **HIGH/CRITICAL** findings.

## Why it exists

"Shift left": find a leaked AWS key or a vulnerable library **before** anything is built or shipped. It's the cheapest point to fix it. The stage also proves the gate works by **planting** a fake secret and an old Log4j dependency, then removing them.

## Analogy

An **airport bag scanner at the entrance**: before your luggage (code) goes anywhere, it's X-rayed for forbidden items (secrets, vulnerable libraries). Anything dangerous and you don't get past the gate.

## How it works inside

```text
trivy-scan-job (ubuntu-26.04)
  actions/checkout@v7 → repo at $GITHUB_WORKSPACE
  aquasecurity/trivy-action@v0.36.0
     scan-type: fs, scan-ref: ${{ github.workspace }}
     scanners: vuln,secret,misconfig   severity: HIGH,CRITICAL   exit-code: '1'
     → Trivy downloads its vulnerability DB, inspects lockfiles/manifests (pom.xml, package-lock.json),
       searches for secret patterns, checks IaC/K8s files
     → findings ≥ HIGH → exit code 1 → job ❌
other jobs (later) run in parallel unless they `need` this job
```

> ⚠️ **Security note (added):** `aquasecurity/trivy-action` itself was **compromised in March 2026** (most version tags were rewritten to malicious code). `v0.36.0` is a post-incident release, but for production pin the **full commit SHA** of the release you've reviewed: `uses: aquasecurity/trivy-action@<sha>  # v0.36.0` (topic 37).

## Workflow YAML / commands: build it step by step

### Stage 5: Replace the foundation job with a Trivy scan

In `.github/workflows/ci.yaml`, **delete `workflow-foundation-job`** and add this job under `jobs:` ([Trivy action docs](https://github.com/marketplace/actions/aqua-security-trivy)):

```yaml
  trivy-scan-job:
    name: Repository Filesystem Security Scan
    runs-on: ubuntu-26.04

    steps:
      - name: Checkout source code
        uses: actions/checkout@v7

      - name: Run Trivy repository filesystem scan
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          scan-type: fs
          scan-ref: ${{ github.workspace }}
          severity: HIGH,CRITICAL
          exit-code: '1'
```

| Line | Why |
|---|---|
| Its own checkout | Every job is a fresh runner, so each one fetches the code itself. |
| `scan-type: fs`, `scan-ref: ${{ github.workspace }}` | Scan the checked-out repo: source, manifests and lockfiles (vulnerabilities, plus secrets and misconfigurations depending on the scanners and files). |
| `severity: HIGH,CRITICAL` | Report only serious findings for now. |
| `exit-code: '1'` | A finding fails the step, so the job and the workflow fail too. That's the **gate**. |
| `@v0.36.0` | An exact version. In production, pin the **full commit SHA**: this action's tags were hijacked in March 2026 (topic 37). |

> Jobs without `needs:` run **in parallel**. Once the build jobs exist (Stage 6), a failed scan won't cancel them. Later stages make the image jobs depend on the gates.

```bash
git add . && git commit -m "ci: add Trivy filesystem security scan" && git push
```

In **Actions**, open **Repository Filesystem Security Scan**. It should pass on the clean repo.

### Prove the gate works, then clean up

**1. Plant findings.** Add a known-vulnerable dependency to `<dependencies>` in `cwvj-3tier-mega-project/backend/pom.xml`:

```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.14.0</version>   <!-- Log4Shell, CVE-2021-44228 -->
</dependency>
```

Add a **fake** credential in `secrets.txt` at the repo root (a synthetic value, not a real key):

```text
AWS_ACCESS_KEY_ID=AKIAAAAAAAAAAAAAAAAA
```

```bash
git branch --show-current            # feature/initial-ci
git add . && git commit -m "test: simulate Trivy security findings" && git push origin feature/initial-ci
```

**Result in the course run:** Trivy reported **3 vulnerabilities** in `backend/pom.xml` and **1 secret** in `secrets.txt`, and the job failed ❌. The frontend lockfile was clean.

**2. Remove them** (delete the Log4j block from `pom.xml`):

```bash
git rm secrets.txt
git add . && git commit -m "chore: remove intentional Trivy test findings" && git push origin feature/initial-ci
```

The next run passes ✅.

---

## Hands-on lab

1. Add the Trivy job and confirm the scan passes on clean code.
2. **Prove the gate works:** add the synthetic `secrets.txt` and the old Log4j dependency, push, and **read the Trivy table** in the log (CVE IDs, severity, fixed version). The job must be ❌.
3. Remove them, push, and it's ✅ again.
4. Extra: add `format: sarif` + `output: trivy.sarif` and upload with `github/codeql-action/upload-sarif` (needs `security-events: write`) to see results in **Security → Code scanning**.

## Where you'll use it

- Every repo's first CI gate (dependency + secret + IaC scanning).
- Compliance evidence that every change was scanned.

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Job ❌ with a CVE table | Real HIGH/CRITICAL findings. | Upgrade the dependency to the "Fixed Version". Don't lower severity just to pass. |
| `Fatal error … failed to download vulnerability DB` | Network or rate limit on the DB download. | Re-run. Cache the Trivy DB (`cache: true` in newer trivy-action) or mirror it. |
| Secret found in history after deleting the file | Old commits still contain it. | **Rotate** the secret. Rewrite history if needed. |
| Scan passes but the image later has CVEs | The FS scan can't see OS packages inside the image. | That's why Stage 8 adds **image** scans. |

## Quick recap

- Trivy `fs` scan of the checkout: vulnerabilities + secrets + misconfigurations, failing on HIGH/CRITICAL.
- Proven with planted test findings, then cleaned up.
- Pin scanners and actions by SHA. The scanner action itself was a 2026 supply-chain target.

**Interview questions**

1. **Q: Filesystem scan vs image scan?**
   A: An FS scan checks source, manifests and lockfiles in the repo. An image scan checks the built image, including OS packages and the files actually packaged. You need both.
2. **Q: How do you make a scanner block the pipeline?**
   A: Set it to exit non-zero (`exit-code: '1'`) for chosen severities, so the job fails and dependent jobs are skipped.
3. **Q: A scan finds a committed AWS key. What do you do?**
   A: Revoke and rotate the key immediately, remove it from the code and history, and investigate its use. Deleting the file alone isn't enough.
