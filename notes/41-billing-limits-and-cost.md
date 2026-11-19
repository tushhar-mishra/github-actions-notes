# 41 · Billing, Limits and Cost Optimization

> **Part G: Security and production** · Previous: [40 · Supply chain](40-supply-chain-attestations.md) · Next: [42 · Capstone 00: overview](42-capstone-00-overview.md) · [Back to INDEX](INDEX.md)
>
> 🆕 New topic. Lecture 01 said "billing depends on GitHub subscription plans and workflow usage", and lectures 04/08/09/13 repeatedly said "reduce unnecessary runs to cut costs". This topic makes that concrete. **Prices and quotas change: always check [GitHub's billing docs](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions) for current numbers.**

---

## What it is

GitHub Actions usage is measured in:

- **Minutes** on **GitHub-hosted runners**, counted **per job**, rounded up to the whole minute, and **multiplied by OS** for paid usage (Linux is the cheapest, Windows costs more, macOS the most).
- **Storage** for **artifacts** and **caches** (and GitHub Packages).

General rules (2026):

| | Public repos | Private repos |
|---|---|---|
| Standard GitHub-hosted runners | **Free** | Free **monthly quota** of minutes and storage per plan (Free < Pro/Team < Enterprise), then pay per minute |
| Larger runners | Billed | Billed |
| Self-hosted runners | Free to GitHub (you pay for your machines) | Free to GitHub (you pay for your machines)* |

\* In Dec 2025 GitHub announced a $0.002/minute "platform charge" for self-hosted runner usage from March 2026, then **postponed it** after feedback. As of late 2026 it hasn't taken effect, but GitHub said it's re-evaluating, so check the billing docs. The same announcement **reduced GitHub-hosted runner prices** from 1 Jan 2026.

Plus **limits** that apply whatever you pay:

| Limit | Value (typical) |
|---|---|
| Job execution time (hosted) | **6 hours** (default `timeout-minutes: 360`) |
| Workflow run time | 35 days (including waiting) |
| Jobs in a matrix | **256** per workflow run |
| Concurrent jobs | Depends on plan (e.g. tens for Free/Pro, hundreds for Enterprise). Fewer for macOS |
| Job outputs | 1 MB per job, 50 MB per run |
| Cache storage | 10 GB per repo by default (more purchasable), evicted after 7 days unused |
| Artifact retention | 1–90 days per upload. Repo setting up to 400 days (private) |
| Workflow queue | If too many runs are queued, new ones are rejected |
| API rate limit for `GITHUB_TOKEN` | 1,000 requests/hour per repo (higher on Enterprise Cloud) |

---

## Why it exists

A matrix of `3 OS × 3 versions` on every push to every PR, plus 5 GB of artifacts kept for 90 days, adds up fast. Finance teams notice, and so do developers waiting in queues. Cost optimization is also **speed** optimization: fewer and shorter jobs mean faster feedback.

**Jenkins comparison:** in Jenkins you pay for the servers 24/7, idle or not. In GitHub Actions you pay per minute used (hosted), or you run self-hosted/ARC to control cost at scale.

---

## Analogy

A **taxi meter**: every job starts a meter (rounded up to the minute). A Windows taxi costs more per minute than a Linux taxi, and a macOS taxi more still. Parking (storage) is charged by the day. Ways to save: take fewer trips (filters, concurrency cancel), shorter trips (caching), cheaper cars (Linux, arm64), and don't leave luggage parked for 90 days (retention).

---

## How it works inside

```text
job starts on a hosted runner → billable minutes begin
job ends → minutes = ceil(duration)  ×  OS multiplier (paid usage)
  e.g. 3 jobs of 61 s each = 3 × 2 = 6 billable minutes (rounding is per job!)
storage = artifacts + caches (+ packages), measured as GB-hours across the month
included quota consumed first → then the spending limit/budget → over budget → jobs blocked
```

Two things follow from per-job rounding:

1. **Many tiny jobs cost more** than one combined job: 10 jobs of 5 seconds = 10 minutes.
2. **Skipped jobs cost nothing**: job-level `if:` (topic 13) and filtered-out workflows (topic 10) are free.

Where to look: **Settings → Billing and plans → Usage** (personal or org), the repo **Actions → Usage metrics/Performance metrics**, and each run's **Billable time** (run summary → ⋯ → View billable time / Usage).

---

## Workflow YAML / commands

### A cost-aware CI workflow

```yaml
name: Cost-aware CI
on:
  pull_request:
    paths-ignore: ["docs/**", "**/*.md"]        # 1. skip docs-only changes (not on required workflows! topic 10)
  push:
    branches: [main]

concurrency:                                    # 2. cancel superseded PR runs
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  test:
    runs-on: ubuntu-26.04                       # 3. Linux (cheapest); consider ubuntu-26.04-arm
    timeout-minutes: 15                         # 4. cap runaway jobs
    strategy:
      matrix:
        python: ${{ github.event_name == 'push' && fromJSON('["3.12","3.13","3.14"]') || fromJSON('["3.13"]') }}
        # 5. full matrix only on main; one version on PRs
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v6
        with:
          python-version: ${{ matrix.python }}
          cache: pip                            # 6. dependency cache (topic 25)
      - run: pip install -r requirements.txt && pytest
      - uses: actions/upload-artifact@v7
        if: failure()                           # 7. only keep reports when they're useful
        with:
          name: pytest-report-${{ matrix.python }}
          path: reports/
          retention-days: 3                     # 8. short retention
```

| # | Saving |
|---|---|
| 1 | No minutes for docs-only PRs |
| 2 | Old runs cancelled when a developer pushes again |
| 3 | Linux is the cheapest per minute. arm64 is often cheaper still |
| 4 | Stops a hung test burning 6 hours |
| 5 | 1 job per PR instead of 3. The full matrix only on `main` |
| 6 | Faster installs = fewer minutes |
| 7–8 | Less artifact storage |

The matrix expression uses the `cond && a || b` pattern (GitHub's "ternary"): if the event is `push`, use the 3-version list, otherwise the 1-version list.

### Check what a run cost

```bash
gh api repos/OWNER/REPO/actions/runs/<run-id>/timing --jq '.billable'
```

```json
{ "UBUNTU": { "total_ms": 184000, "jobs": 3 } }
```

### Clean up storage

```bash
gh cache list --limit 50
gh cache delete --all                                  # clear all caches for the repo
# delete artifacts older than X days via the API or a scheduled workflow
```

Also set **Settings → Actions → General → Artifact and log retention** to a sensible default (for example 14 days).

---

## Hands-on lab

1. Open your practice repo's **Actions → (a run) → ⋯ → View usage/billable time** for a run from topic 26 (matrix) vs topic 08. Compare.
2. Use the `timing` API call above on three runs.
3. Add `concurrency` with `cancel-in-progress` to a PR workflow, push twice quickly, and see the first run cancelled (minutes saved).
4. Set repo artifact retention to 7 days and delete old caches with `gh cache delete`.
5. Calculate: a matrix of 3 OS × 3 versions, each job 4 min 10 s, on 40 PR pushes a day. How many minutes per day? (9 jobs × 5 min (rounded up) × 40 = 1,800, before OS multipliers.)

---

## Where you'll use it

- **Budget reviews** for org-wide CI.
- **Pipeline design:** which jobs run on PR vs `main` vs nightly.
- **Self-hosted/ARC decisions:** at high, steady volume, own runners can be cheaper (topic 38).
- **Interviews:** "how would you reduce CI cost and time?"

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Jobs fail to start: "spending limit reached" / included minutes exhausted | Private-repo quota used up, budget hit. | Raise the budget, optimize, or self-host. |
| Unexpected big bill | macOS/Windows matrices, larger runners, long timeouts, frequent schedules (`*/5`). | Check usage by workflow and runner type. Trim. |
| Storage quota warnings | Large artifacts, long retention, many caches. | Shorter retention, delete old ones. |
| `Exceeded the maximum number of 256 jobs in a matrix` | Matrix too big. | Exclude combinations, or split. |
| Many runs stay **Queued** | The plan's concurrent-job limit was reached. | Concurrency groups, fewer parallel jobs, larger plan, or self-hosted. |

---

## Quick recap

- Hosted minutes are **per job, rounded up**, with an OS multiplier for paid usage. Public repos on standard runners are free.
- Storage = artifacts + caches. Use retention and clean-up.
- Biggest savers: **filters, concurrency cancel, job-level `if`, caching, Linux/arm64, smaller PR matrices, timeouts, short retention**.
- Know the limits: 6 h per job, 256 matrix jobs, 10 GB cache, 1 MB outputs, plan concurrency.

**Interview questions**

1. **Q: Name four ways to reduce GitHub Actions costs.**
   A: Path/branch filters, concurrency with cancel-in-progress, dependency caching, a smaller matrix on PRs (full matrix on main/nightly). Also Linux runners, timeouts, short artifact retention.
2. **Q: Are self-hosted runners free?**
   A: GitHub doesn't charge minutes for them, but you pay for the infrastructure and the operational work.
3. **Q: Why can 10 tiny jobs cost more than 1 job?**
   A: Billing rounds up per job, so each sub-minute job costs a full minute.
