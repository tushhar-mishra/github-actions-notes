# 40 · Supply Chain Security: SBOMs, Provenance and Artifact Attestations

> **Part G: Security and production** · Previous: [39 · Debugging](39-debugging-and-troubleshooting.md) · Next: [41 · Billing, limits and cost](41-billing-limits-and-cost.md) · [Back to INDEX](INDEX.md)
>
> Topic 01's pipeline stage 6 mentions "SBOMs, signatures and provenance metadata". This topic shows how to produce and verify them with GitHub Actions.

---

## What it is

**Software supply-chain security** answers: *"Is this artifact really what we built, from the code we reviewed, with known ingredients?"*

| Term | Plain meaning |
|---|---|
| **SBOM** (Software Bill of Materials) | The **ingredient list** of an artifact: every library and version inside it. Formats: **SPDX**, **CycloneDX**. |
| **Provenance** | A signed record of **how and where** an artifact was built: repo, commit, workflow, runner, inputs. |
| **Attestation** | A **signed statement** about an artifact (its provenance, its SBOM, a scan result), cryptographically bound to the artifact's **digest**. |
| **SLSA** ("salsa", Supply-chain Levels for Software Artifacts) | A framework of levels for build integrity. Signed provenance from a hosted build platform like GitHub Actions supports **SLSA Build Level 2–3**. |
| **Sigstore** | Free, open signing infrastructure. GitHub artifact attestations use it (public repos use the public instance). |

---

## Why it exists

Attackers increasingly target the **build**, not the source: swapped base images, compromised actions (topic 37), tampered registries. If production only runs artifacts with a **valid signed provenance from your repo's CI**, a tampered or locally built image is **rejected**.

SBOMs let you answer *"are we affected by CVE-X?"* in minutes instead of days. Regulations and customers (for example US federal software suppliers) increasingly **require** SBOMs.

**Jenkins comparison:** Jenkins needs plugins and your own signing setup. GitHub Actions provides first-party attestations tied to the workflow's OIDC identity, with no keys to manage.

---

## Analogy

A **food package**:

- **SBOM** = the **ingredients label**.
- **Provenance** = **"Made in factory X, line 3, batch #42, on 19 Nov, from recipe v7"**.
- **Attestation** = that label **sealed with the factory's tamper-proof hologram**. If anyone repackages the food, the hologram doesn't match.
- **Verification** = the supermarket scanning the hologram before putting it on the shelf (the deploy gate).

---

## How it works inside

```text
build job (permissions: id-token: write, attestations: write, contents: read)
  1. build artifact (image/JAR) → compute its SHA-256 digest
  2. actions/attest-build-provenance:
       - gets the job's OIDC token (proves: repo, workflow, ref, commit, runner)
       - Sigstore issues a short-lived signing certificate bound to that identity
       - signs an in-toto/SLSA provenance statement {subject: digest, builder, source, …}
       - stores the attestation in GitHub (and pushes it to the registry if push-to-registry)
  3. later (deploy or anyone): gh attestation verify <artifact> --repo OWNER/REPO
       - checks the signature, certificate identity (this repo's workflow) and digest match
```

No private keys are stored anywhere. The identity **is** the workflow's OIDC token (topic 36).

---

## Workflow YAML / commands

### Generate an SBOM, attest provenance and the SBOM for a container image

```yaml
name: Build and attest image
on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write          # push to GHCR (topic 18)
  id-token: write          # OIDC for signing (topic 36)
  attestations: write      # store attestations

jobs:
  build:
    runs-on: ubuntu-26.04
    steps:
      - uses: actions/checkout@v7

      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: push
        uses: docker/build-push-action@v7
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Generate SBOM (CycloneDX)
        uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/${{ github.repository }}:${{ github.sha }}
          format: cyclonedx-json
          output-file: sbom.cdx.json

      - name: Attest build provenance
        uses: actions/attest-build-provenance@v3
        with:
          subject-name: ghcr.io/${{ github.repository }}
          subject-digest: ${{ steps.push.outputs.digest }}
          push-to-registry: true

      - name: Attest SBOM
        uses: actions/attest-sbom@v3
        with:
          subject-name: ghcr.io/${{ github.repository }}
          subject-digest: ${{ steps.push.outputs.digest }}
          sbom-path: sbom.cdx.json
          push-to-registry: true
```

> Versions shown are the majors current at writing. As with any third-party action (here `anchore/sbom-action`), pin SHAs in production (topic 37). The repo name must be lowercase for GHCR. `${{ github.repository }}` works when your owner and repo names are lowercase.

**Sample log output:**

```text
Run actions/attest-build-provenance@v3
  Attestation created for ghcr.io/you/app@sha256:6b1f…
  Uploaded attestation to https://github.com/you/app/attestations/1234567
  Attestation signed using certificate from Sigstore Public Good instance
```

### Verify before deploying

```bash
gh attestation verify oci://ghcr.io/you/app:3f9a1c2 --repo you/app
```

```text
Loaded digest sha256:6b1f… for oci://ghcr.io/you/app:3f9a1c2
Loaded 1 attestation from GitHub API
✓ Verification succeeded!

sha256:6b1f… was attested by:
REPO     PREDICATE_TYPE                  WORKFLOW
you/app  https://slsa.dev/provenance/v1  .github/workflows/build.yml@refs/heads/main
```

In Kubernetes, admission policies (for example **Kyverno** or **Sigstore policy-controller**) can enforce "only run images with a valid attestation from `you/app`".

### Where this connects to the capstone

The capstone scans **before** pushing (Trivy filesystem + image scans, topics 44 and 48) and promotes the **exact scanned image** by digest/tag (topic 49). Adding `attest-build-provenance` after its ECR push makes that promotion **verifiable**.

---

## Hands-on lab

1. In a public test repo, run the workflow above (GHCR needs no extra setup).
2. Open **Actions → Attestations** (or the repo's **Attestations** page) and see the entry.
3. Run `gh attestation verify` locally and get ✓.
4. Build the same Dockerfile **on your laptop**, push it under another tag, and run verify: it **fails** (no attestation for that digest). That's the protection.
5. Open `sbom.cdx.json` (upload it as an artifact) and find the Flask version.

---

## Where you'll use it

- **Release pipelines** for containers, binaries and packages.
- **Compliance:** SBOMs for customers and regulators. SLSA levels in security questionnaires.
- **Deploy gates:** verify attestations before `kubectl apply`.
- **Incident response:** "which images contain log4j 2.14?" Query the SBOMs.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Error: Failed to get ID token` | Missing `id-token: write`. | Add it. |
| `Resource not accessible by integration` creating an attestation | Missing `attestations: write`. | Add it. |
| Attestations unavailable on a private repo | Plan restriction (private repos need GitHub Enterprise Cloud). | Use a public repo or the right plan. |
| `gh attestation verify` → `no attestations found` | Different digest (a re-tagged or rebuilt image), wrong `--repo`. | Verify the exact digest produced by CI. |
| `subject-digest` empty | `build-push-action` didn't push (no `id`, or `push: false`). | Give the step an `id`, push, and use `steps.<id>.outputs.digest`. |

---

## Quick recap

- **SBOM** = the ingredient list. **Provenance** = how and where it was built. **Attestation** = a signed statement bound to the artifact's digest.
- `actions/attest-build-provenance` / `actions/attest-sbom` + `id-token: write` + `attestations: write`. Keyless signing via Sigstore and OIDC.
- Verify with `gh attestation verify`, and enforce in the cluster with admission policies.
- Supports SLSA build levels and helps detect tampered artifacts.

**Interview questions**

1. **Q: What is an SBOM and why generate it in CI?**
   A: A machine-readable list of every component in an artifact. CI generates it for every build so you can quickly find affected artifacts when a CVE appears.
2. **Q: How does GitHub sign attestations without stored private keys?**
   A: It uses the job's OIDC identity to get a short-lived Sigstore certificate, so the workflow identity itself is the signer.
3. **Q: How would you stop a manually built image from being deployed to production?**
   A: Require a valid build-provenance attestation from the repo's CI workflow (`gh attestation verify` in the deploy job, or an admission policy).
