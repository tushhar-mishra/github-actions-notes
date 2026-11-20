# 35 · Publishing and Versioning Your Actions

> **Part F: Custom actions** · Previous: [34 · Docker actions](34-docker-actions.md) · Next: [36 · OIDC cloud authentication](36-oidc-cloud-authentication.md) · [Back to INDEX](INDEX.md)
>
> 🆕 New topic. Lectures 11–12 built **local** actions and showed `owner/repo@v1` references. This topic covers how to turn a local action into a **remote, versioned** one that other repos (or the Marketplace) can safely use.

---

## What it is

**Publishing** an action means putting it in its **own repository** with **version tags**, so other workflows can call `owner/repo@v1`.

**Versioning** means following **semantic versioning** (**semver**, `MAJOR.MINOR.PATCH`, like `v2.4.1`):

| Part | Bump when… | Example |
|---|---|---|
| **MAJOR** | breaking change (renamed input, removed output, new runtime) | `v1.9.0 → v2.0.0` |
| **MINOR** | new backwards-compatible feature | `v2.0.0 → v2.1.0` |
| **PATCH** | bug fix | `v2.1.0 → v2.1.1` |

The GitHub convention: also keep a **moving major tag** (`v2`) that always points to the **latest `v2.x.y`**, so users of `@v2` get fixes automatically.

---

## Why it exists

- Copying a local action into 50 repos brings back the duplication problem. A remote action is **one source of truth**.
- Users need **stability**: they shouldn't get a breaking change overnight. Semver and major tags let them choose between "auto-update fixes" (`@v2`) and "frozen" (`@v2.1.1` or a SHA).
- Security teams need **immutability**: releases that can't be silently changed (immutable releases, below).

**Jenkins comparison:** like releasing a Shared Library with `@Library('my-lib@2.1.0')` tags, or publishing a versioned plugin.

---

## Analogy

A **book series**:

- **v2.1.1** = a specific **printing** of edition 2 (exact).
- **v2** = "the latest printing of **edition 2**". The library swaps it for the newest corrected printing.
- **v3** = a **new edition** with chapters reorganized (breaking). Readers must decide to switch.
- A **commit SHA** = the exact **ISBN plus page scan**. It can never be changed.

---

## How it works inside

```text
action repo: my-org/build-info-action
  action.yml (at repo ROOT for owner/repo@ref; or in a sub-folder for owner/repo/path@ref)
  dist/ (bundled JS) or Dockerfile or composite steps
  README.md, LICENSE
release flow:
  git tag -a v1.2.0 -m "v1.2.0" && git push origin v1.2.0      ← exact version
  git tag -fa v1 -m "v1 → v1.2.0" && git push -f origin v1     ← move the MAJOR tag
consumer:
  uses: my-org/build-info-action@v1        → resolves the tag v1 → its commit
  runner downloads the repo at that commit during "Set up job"
```

- **Private/internal action repos:** consumers in other repos need access. In the action repo: **Settings → Actions → General → Access** → "Accessible from repositories in the organization" (or owned by the user).
- **Marketplace:** public repo, `action.yml` at the root, a unique `name`, `branding` (icon + colour), then **Draft a release → Publish this Action to the GitHub Marketplace**.
- **Immutable releases** (GitHub feature): once a release is published, its tag and assets **can't be moved or changed**. During the **March 2026 trivy-action attack**, the one version protected by an immutable release was the one **not** hijacked. Turn it on for actions you publish (**Settings → General → Releases → Enable release immutability**).

> Moving the major tag is the one place where tags are deliberately mutable, which is exactly why **security-sensitive consumers pin full SHAs** (topic 37).

---

## Workflow YAML / commands

### Repository layout for a published composite action

```text
my-org/build-info-action/
├── action.yml
├── README.md          ← inputs, outputs, example usage
├── LICENSE
└── .github/workflows/
    ├── test.yml       ← tests the action itself on every PR
    └── release.yml    ← moves the major tag when a release is published
```

`action.yml` with Marketplace branding:

```yaml
name: CWVJ Build Info
description: Generate build metadata as a file and an output.
author: my-org
branding:
  icon: info            # a Feather icon name
  color: blue
inputs:
  application-name:
    description: Name of the application.
    required: true
outputs:
  build-info-path:
    description: Path to the generated file.
    value: ${{ steps.gen.outputs.path }}
runs:
  using: composite
  steps:
    - id: gen
      shell: bash
      env:
        APP_NAME: ${{ inputs.application-name }}   # via env, never ${{ }} inside run: (injection, topic 37)
      run: |
        mkdir -p output
        printf 'app=%s sha=%s
' "$APP_NAME" "$GITHUB_SHA" > output/build-info.txt
        echo "path=output/build-info.txt" >> "$GITHUB_OUTPUT"
```

### Testing the action in its own repo

```yaml
# .github/workflows/test.yml
name: Test action
on: [pull_request, push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - id: run
        uses: ./                       # the action at the repo root
        with:
          application-name: test-app
      - run: |
          test -f "${{ steps.run.outputs.build-info-path }}"
          grep -q "app=test-app" "${{ steps.run.outputs.build-info-path }}"
```

### Automatically moving the major tag on release

```yaml
# .github/workflows/release.yml
name: Move major tag
on:
  release:
    types: [published]
permissions:
  contents: write
jobs:
  tag:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - env:
          TAG: ${{ github.event.release.tag_name }}     # e.g. v1.2.0
        run: |
          MAJOR="${TAG%%.*}"                              # → v1
          git tag -f "$MAJOR" "$TAG"
          git push -f origin "$MAJOR"
```

**Sample log output:**

```text
Run MAJOR="${TAG%%.*}"
 + 5c1e2d4...a9b8c7d v1 -> v1 (forced update)
```

`${TAG%%.*}` is Bash parameter expansion: "remove everything from the first `.`", so `v1.2.0` → `v1`.

### Consuming it

```yaml
- uses: my-org/build-info-action@v1                                   # auto bug fixes
- uses: my-org/build-info-action@v1.2.0                               # exact tag
- uses: my-org/build-info-action@5c1e2d4f…(40 chars)  # v1.2.0       # immutable (recommended for 3rd-party)
```

### Keeping consumers updated: Dependabot for actions

`.github/dependabot.yml` in the **consuming** repo:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

Dependabot opens PRs that bump `uses:` versions, including SHA pins (it updates the `# v1.2.0` comment too).

---

## Hands-on lab

1. Move your `generate-build-info` composite action (topic 32) to a new public repo with `action.yml` at the root.
2. Add `test.yml` and confirm it passes.
3. Create release `v1.0.0`, run `release.yml`, and confirm the `v1` tag exists.
4. Call it from your practice repo with `@v1`, then with the full SHA.
5. Make a breaking change (rename the input) and release `v2.0.0`. Confirm `@v1` users still get the old behaviour.
6. Add `dependabot.yml` to the practice repo and wait for (or trigger) a version-bump PR.

---

## Where you'll use it

- **Platform teams** publishing internal actions (`my-org/deploy-to-eks@v3`).
- **Open source:** publishing to the Marketplace.
- **Supply-chain hygiene:** SHA pins plus Dependabot in every repo.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Unable to resolve action my-org/x@v1, repository or version not found` | Tag missing, repo private without access, or a typo. | Push the tag. Enable **Access** for private action repos. |
| `Can't find 'action.yml'` for `owner/repo@v1` | `action.yml` isn't at the repo root. | Move it, or reference `owner/repo/subdir@v1`. |
| Users broke after your release | A breaking change under the same major. | Revert. Release as a new MAJOR. |
| `failed to push some refs … (stale info)` moving the tag | Tag protection or a ruleset blocks force-pushing tags. | Allow it for the release workflow, or move the tag manually. |
| Marketplace publish greyed out | The repo isn't public, `action.yml` isn't at the root, or the name is taken. | Fix the requirement shown. |

---

## Quick recap

- Publish = own repo + `action.yml` at the root + semver tags (+ a moving major tag).
- Breaking change = new MAJOR. Consumers choose `@v1` (auto fixes), `@v1.2.0`, or a **SHA** (immutable).
- Test the action in its own repo. Automate major-tag moves on release.
- Private action repos need **Access** settings. Marketplace needs a public repo + branding.
- Use Dependabot (`github-actions` ecosystem) to keep consumers updated. Enable immutable releases.

**Interview questions**

1. **Q: How do popular actions let users write `@v4` but still get fixes?**
   A: They publish `v4.x.y` releases and force-move the `v4` tag to the newest one.
2. **Q: Why might a security team forbid `@v4` and require SHAs?**
   A: Tags are mutable, so a compromised maintainer could repoint them. A full SHA can't change.
3. **Q: How do you keep SHA-pinned actions up to date without manual work?**
   A: Dependabot with `package-ecosystem: github-actions`.
