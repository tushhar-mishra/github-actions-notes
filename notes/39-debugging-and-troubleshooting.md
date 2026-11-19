# 39 · Debugging and Troubleshooting Workflows

> **Part G: Security and production** · Previous: [38 · Self-hosted runners](38-self-hosted-runners-deep-dive.md) · Next: [40 · Supply chain: SBOM, provenance and attestations](40-supply-chain-attestations.md) · [Back to INDEX](INDEX.md)
>
> 🆕 New topic. Every topic so far has a "When it breaks" table. This one teaches the **method and the tools**, and collects the most common errors in one catalogue.

---

## What it is

A repeatable way to answer **"why did my workflow do that?"**, using:

- the **trace** from topic 08 (event → match → queue → runner → steps → results) to locate **where** it broke
- **logs**, **debug logging**, **re-runs** and **annotations**
- **local tools**: `actionlint` (lint), `act` (run locally), the `gh` CLI
- **context dumps** (`toJSON`)

---

## Why it exists

CI failures are expensive: blocked merges, broken deploys, people waiting. Most of them are a handful of repeating causes (filters, permissions, secrets, YAML, caches, runners). A method turns a 2-hour guessing session into a 5-minute check.

**Jenkins comparison:** Jenkins "Replay" ≈ re-run jobs, and console log + `set -x` ≈ step debug logging. `act` gives you something Jenkins never had: running the pipeline locally.

---

## Analogy

A **car mechanic's diagnostic flow**: first ask **where** the problem is (engine won't start = trigger, stalls at a junction = a step, runs but is slow = cache or runner), then plug in the **diagnostic computer** (debug logs), **re-run the test drive** (re-run jobs), and use the **workshop simulator** (`act`) before touching the real car.

---

## How it works inside

### Step 1: Locate the failing stage

```text
No run at all?            → stage ③ matching: on:, filters, file location, default branch (topics 09–10, 31)
Run shows "Invalid workflow file"? → YAML/schema (topic 03)
Run exists, job "Queued" forever?  → runner matching / concurrency / plan (topics 07, 29, 38)
Job skipped?              → if: / needs: (topic 13)
Step failed?              → read the FIRST error line in that step's log (topic 04)
Passed but wrong result?  → contexts/expressions evaluated differently than you thought (topic 11)
```

### Step 2: Turn on debug logging

| Switch | How | Effect |
|---|---|---|
| **Re-run with debug** | Run page → **Re-run jobs → ☑ Enable debug logging** | One re-run with both of the below |
| `ACTIONS_STEP_DEBUG=true` | Repository **secret or variable** | Prints `##[debug]` lines: expression evaluation, action internals, `core.debug()` output |
| `ACTIONS_RUNNER_DEBUG=true` | Repository **secret or variable** | Runner diagnostic logs (job setup, networking), downloadable with the run logs |

### Step 3: Inspect, reproduce, fix

- **Dump contexts** to see the real values:

  ```yaml
  - run: |
      echo '${{ toJSON(github.event) }}' | head -c 4000
      echo '${{ toJSON(needs) }}'
      echo '${{ toJSON(steps) }}'
  ```

  (Use `env:` for untrusted events, topic 37.)
- **Shell tracing:** add `set -x` to see each command as Bash runs it.
- **Re-run failed jobs** only (keeps successful jobs, saves minutes).
- **Lint locally** with `actionlint` before pushing.
- **Run locally** with `act` for fast iteration on steps.

---

## Workflow YAML / commands

### actionlint: catch errors before you push

```bash
# install (one of)
brew install actionlint            # macOS
go install github.com/rhysd/actionlint/cmd/actionlint@latest
# run in the repo root
actionlint
```

**Sample output:**

```text
.github/workflows/ci.yml:12:9: unexpected key "step" for "job" section. expected one of "runs-on", "steps", … [syntax-check]
   |
12 |         step:
   |         ^~~~~
.github/workflows/ci.yml:20:24: property "pyhton-version" is not defined in object type {python-version: string} [expression]
.github/workflows/ci.yml:25:14: "github.event.pull_request.title" is potentially untrusted. avoid using it directly in inline scripts. instead, pass it through an environment variable [expression]
```

It checks YAML, workflow keys, `${{ }}` expressions and types, `needs` references, matrix values, shell scripts (via `shellcheck` if installed) and injection risks.

### act: run workflows on your laptop (uses Docker)

```bash
brew install act                         # or download from github.com/nektos/act
act -l                                   # list jobs
act push -j test                         # run job "test" for a push event
act workflow_dispatch -W .github/workflows/ci.yml --input environment=qa
act -s GITHUB_TOKEN="$(gh auth token)"   # provide secrets (-s NAME=value or --secret-file)
```

**Sample output:**

```text
[CI/test] 🚀  Start image=catthehacker/ubuntu:act-latest
[CI/test]   ✅  Success - Main actions/checkout@v7
[CI/test] ⭐ Run Main npm test
| > app@1.0.0 test
| > jest
| Tests: 12 passed, 12 total
[CI/test]   ✅  Success - Main npm test
[CI/test] 🏁  Job succeeded
```

Limits: `act` approximates hosted runners with Docker images. Services, caches, OIDC, environments and some contexts behave differently, so always confirm on GitHub.

### gh CLI: investigate runs from the terminal

```bash
gh run list --workflow ci.yml --limit 5
gh run view <run-id> --log-failed          # only the failed steps' logs
gh run rerun <run-id> --failed --debug     # re-run failed jobs with debug logging
gh run watch                               # live status
gh run download <run-id> -n test-report    # fetch an artifact
gh workflow view ci.yml                    # triggers and recent runs
```

### The error catalogue (what you'll actually see)

| Error / symptom | Usual cause | Fix | Topic |
|---|---|---|---|
| No run appears | Wrong folder, `on:` or filters don't match, Actions disabled, file not on the default branch for `schedule`/`workflow_run`/dispatch | Check the path, filters and default branch | 08, 10, 31 |
| `Invalid workflow file … line N` | YAML indentation, tabs, `: ` in an unquoted string, unknown key | `actionlint` | 03 |
| `Unrecognized named-value: 'X'` | Shell var used in `${{ }}`, or a context not available at that level | Use the right context | 11, 13 |
| Job stuck "Waiting for a runner…" | Label mismatch, runner offline, group access, plan limits | Runner settings page | 07, 38 |
| Job/run "Pending" or "Cancelled: higher priority waiting request" | Concurrency group | Expected, or adjust the group/queue | 29 |
| `Error: Process completed with exit code 1.` | A command failed | First error line above it, `set -x` | 04 |
| **403 `Resource not accessible by integration`** | `GITHUB_TOKEN` lacks a permission, or a fork PR | `permissions:` | 18 |
| Secret empty / `Username and password required` | Wrong name or scope, missing `environment:`, **fork PR** | Check the scope | 17, 28 |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | OIDC trust policy `sub`/`aud` mismatch | Print the claims | 36 |
| `Cache not found` every time | Key changes every run, `hashFiles` matched nothing, branch scope, failed jobs don't save | Print the key | 25 |
| `Conflict: an artifact with this name already exists` | Artifacts are immutable (v4+) | Unique names | 24 |
| `Unable to resolve action …` | Wrong tag, private repo, not on the allowed list | Releases page, Access settings | 15, 35 |
| `node20 … no longer supported` | Old action runtime | Upgrade the action | 33 |
| `No space left on device` | 14 GB runner disk | Prune, larger runner | 07 |
| Required check "Expected — Waiting for status" | The workflow was filtered out for that PR | Remove path filters on required workflows | 10, 30 |
| `GH013: Repository rule violations` | Pushed to a protected branch | Open a PR | 30 |
| Works locally, fails in CI | Different OS image, tool versions, env vars, missing `CI=true` behaviour | Pin the image and versions, print `env` | 07, 11 |

---

## Hands-on lab

1. Create a workflow with three deliberate bugs: a tab character, `step:` instead of `steps:`, and `${{ inputs.envirnoment }}`. Find all three with `actionlint` **before** pushing.
2. Make a step fail, then **Re-run jobs → Enable debug logging**, and find the `##[debug]Evaluating condition for step` lines.
3. Set the repo variable `ACTIONS_STEP_DEBUG=true` and compare a normal run's log size.
4. Run your topic 16 workflow locally with `act push`.
5. Use `gh run view --log-failed` on a failed run, then `gh run rerun --failed`.

---

## Where you'll use it

- **On-call for CI/CD:** the fastest path from a red X to the root cause.
- **Pre-commit hooks:** run `actionlint` locally or in CI on workflow changes.
- **Teaching teammates:** the error catalogue as a team wiki page.
- **Interviews:** "your workflow didn't trigger, how do you debug it?" Use the stage-by-stage answer.

---

## When it breaks

(This whole topic is the "when it breaks" guide. Two meta-problems:)

| Symptom | Meaning | Fix |
|---|---|---|
| Debug logging shows nothing new | Set as an **environment** secret, or the name is misspelled. | Repository-level `ACTIONS_STEP_DEBUG` = `true`, or use "Enable debug logging" on re-run. |
| `act` result differs from GitHub | Different image, missing services or OIDC. | Treat `act` as a fast pre-check, and confirm on GitHub. |

---

## Quick recap

- First **locate the stage** (no run / invalid file / queued / skipped / step failed / wrong result).
- **Re-run with debug logging**, or `ACTIONS_STEP_DEBUG` / `ACTIONS_RUNNER_DEBUG`.
- **Dump contexts** with `toJSON`, and `set -x` for shell.
- **actionlint** before pushing, **act** for local runs, **gh run** for logs and re-runs.
- Use the error catalogue. Most failures are filters, permissions, secrets, YAML, caches or runners.

**Interview questions**

1. **Q: How do you get more detailed logs for a failing GitHub Actions job?**
   A: Re-run with "Enable debug logging", or set `ACTIONS_STEP_DEBUG` (and `ACTIONS_RUNNER_DEBUG`) to `true`.
2. **Q: How can you catch workflow syntax and expression errors before pushing?**
   A: `actionlint` locally or in a pre-commit hook.
3. **Q: A workflow with `on: push: branches: [main]` didn't run on your feature branch push. Bug?**
   A: No. The branch filter excluded it. Check `on:` and filters first.
