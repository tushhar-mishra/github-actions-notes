# 23 · Workflow Commands, Environment Files and Job Summaries

> **Part D: Moving data around** · Previous: [22 · Reusable workflows](22-reusable-workflows.md) · Next: [24 · Artifacts](24-artifacts.md) · [Back to INDEX](INDEX.md)
>
> This topic collects **all** the ways a step can talk back to GitHub, including the ones the capstone and the JavaScript action (topic 33) use.

---

## What it is

A running step can **send instructions to the runner** in two ways:

1. **Environment files**: special files whose paths GitHub gives you in variables. You **append lines** to them.

   | File variable | What appending to it does |
   |---|---|
   | `$GITHUB_ENV` | Sets environment variables for **later steps** in this job |
   | `$GITHUB_OUTPUT` | Sets **outputs** of this step (topic 21) |
   | `$GITHUB_PATH` | Adds a folder to `PATH` for **later steps** |
   | `$GITHUB_STEP_SUMMARY` | Adds Markdown to the job's **summary page** |
   | `$GITHUB_STATE` | Saves state for an action's `post:` step (topic 33) |

2. **Workflow commands**: specially formatted lines **printed to the log** (`::command parameters::message`):

   | Command | Effect |
   |---|---|
   | `::error file=app.py,line=10::Message` | Red **annotation** (also shown on the PR's Files tab) |
   | `::warning::Message` / `::notice::Message` | Yellow / blue annotation |
   | `::group::Title` … `::endgroup::` | Collapsible section in the log |
   | `::add-mask::value` | Hide a value as `***` from now on |
   | `::debug::Message` | Shown only when debug logging is on (topic 39) |
   | `::stop-commands::TOKEN` … `::TOKEN::` | Temporarily turn off command parsing (for printing untrusted text) |

> ⚠️ **Deprecated and disabled:** `::set-output`, `::save-state`, `::set-env` and `::add-path`. `set-env` and `add-path` were disabled in **2020** after a security flaw (CVE-2020-15228: any log line, even from untrusted input, could set variables). `set-output` and `save-state` were deprecated in **October 2022**; their removal was postponed, so today they still run with a deprecation warning. Use the environment files instead. Old blog posts still show these. Don't copy them.

---

## Why it exists

- Steps are **separate shell processes** (topic 04), so `export` doesn't carry over. You need a sanctioned way to pass values → `GITHUB_ENV` / `GITHUB_OUTPUT`.
- You install a tool into `~/tools/bin` and later steps must find it → `GITHUB_PATH`.
- Reviewers want "**3 tests failed in `auth.py` line 42**" shown **on the PR**, not buried in 2,000 log lines → annotations.
- Managers want a **one-page result** (coverage, image tag, scan counts) → job summaries.

**Jenkins comparison:** `env.X = ...` in Groovy ≈ `GITHUB_ENV`. The Warnings Next Generation plugin ≈ annotations. A build description or an HTML Publisher report ≈ `GITHUB_STEP_SUMMARY`.

---

## Analogy

The runner is a **stage manager**, and each step is an **actor**:

- Writing in the **shared notebook** (environment files): "Next actor: the prop is in drawer 3" (`GITHUB_ENV`), "Add the costume room to the map" (`GITHUB_PATH`), "Add this to tonight's programme" (`GITHUB_STEP_SUMMARY`).
- **Shouting cues** during the show (workflow commands in the log): "WARNING: the curtain is stuck!" (`::warning::`), "Don't repeat the password aloud" (`::add-mask::`).

---

## How it works inside

```text
before each step, the runner creates fresh empty files and exports their paths:
    GITHUB_ENV=/home/runner/work/_temp/_runner_file_commands/set_env_<uuid>
    GITHUB_OUTPUT=…/set_output_<uuid>   GITHUB_PATH=…/add_path_<uuid>   GITHUB_STEP_SUMMARY=…/step_summary_<uuid>
step runs → your commands append lines
after the step ends, the runner READS the files:
    GITHUB_ENV    → merged into the environment of every LATER step in this job
    GITHUB_OUTPUT → stored as steps.<id>.outputs.*
    GITHUB_PATH   → prepended to PATH for later steps
    STEP_SUMMARY  → uploaded and rendered as Markdown on the run's summary page (max 1 MiB per step)

log lines are scanned live: a line starting with "::" that matches a known command is executed
```

Key consequences:

- A value written to `$GITHUB_ENV` is **not** visible in the **same** step, only in later ones.
- Environment files are **per job**. A new job starts empty (use outputs to cross jobs).
- Because commands are parsed from **logs**, printing attacker-controlled text (a PR title, a commit message) could trigger them. Wrap such output with `::stop-commands::`.

---

## Workflow YAML / commands

```yaml
name: Workflow Commands Demo
on: workflow_dispatch

jobs:
  demo:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - name: Environment files
        id: meta
        run: |
          # 1. env var for later steps
          echo "IMAGE_TAG=v${GITHUB_RUN_NUMBER}-${GITHUB_SHA::7}" >> "$GITHUB_ENV"

          # 2. step output
          echo "build_time=$(date -u +%FT%TZ)" >> "$GITHUB_OUTPUT"

          # 3. multi-line env var (delimiter syntax)
          {
            echo "CHANGELOG<<EOF"
            git log -3 --oneline
            echo "EOF"
          } >> "$GITHUB_ENV"

          # 4. add a tool folder to PATH for later steps
          mkdir -p "$HOME/tools/bin"
          printf '#!/bin/sh\necho hello-from-mytool\n' > "$HOME/tools/bin/mytool"
          chmod +x "$HOME/tools/bin/mytool"
          echo "$HOME/tools/bin" >> "$GITHUB_PATH"

          echo "IMAGE_TAG in this same step: '${IMAGE_TAG:-<not set yet>}'"

      - name: Use them
        run: |
          echo "IMAGE_TAG=$IMAGE_TAG"
          echo "built at ${{ steps.meta.outputs.build_time }}"
          echo "$CHANGELOG"
          mytool

      - name: Annotations, groups, masking
        run: |
          echo "::group::Install dependencies"
          echo "pip install … (lots of lines hidden in a collapsible group)"
          echo "::endgroup::"

          echo "::notice title=Image::Built image tag $IMAGE_TAG"
          echo "::warning file=app.py,line=1::Debug flag left enabled"

          TOKEN="abc123-generated-at-runtime"
          echo "::add-mask::$TOKEN"
          echo "token is $TOKEN"

      - name: Job summary
        run: |
          {
            echo "## Build result 🚀"
            echo ""
            echo "| Item | Value |"
            echo "|---|---|"
            echo "| Image tag | \`$IMAGE_TAG\` |"
            echo "| Commit | \`${GITHUB_SHA::7}\` |"
            echo "| Triggered by | @$GITHUB_ACTOR |"
          } >> "$GITHUB_STEP_SUMMARY"

      - name: Print untrusted text safely
        env:
          TITLE: "::warning::I am a fake command from a PR title"
        run: |
          STOP=$(uuidgen)
          echo "::stop-commands::$STOP"
          echo "$TITLE"               # printed literally, NOT executed
          echo "::$STOP::"
```

**Sample log output:**

```text
✓ Environment files
    IMAGE_TAG in this same step: '<not set yet>'       ← GITHUB_ENV applies to LATER steps
✓ Use them
    IMAGE_TAG=v7-3f9a1c2
    built at 2026-11-19T10:12:44Z
    3f9a1c2 feat: add commands demo
    a1b2c3d docs: readme
    9e8f7a6 ci: first workflow
    hello-from-mytool
✓ Annotations, groups, masking
    ▶ Install dependencies                              ← collapsed group
    Notice: Built image tag v7-3f9a1c2
    Warning: Debug flag left enabled
    token is ***
✓ Job summary
✓ Print untrusted text safely
    ::warning::I am a fake command from a PR title      ← shown as plain text
```

On the run page:

- **Annotations** box: 1 notice, 1 warning (the warning links to `app.py` line 1).
- **Summary** tab: the Markdown table renders under the job name.

### Where the capstone and other topics use these

- `$GITHUB_ENV` for the dynamic image tag (topic 19).
- `$GITHUB_OUTPUT` for step and job outputs (topics 21–22), e.g. the capstone's `image-tag`.
- `$GITHUB_STEP_SUMMARY` from JavaScript with `core.summary` (topic 33's workflow-summary action).
- `::add-mask::` for runtime tokens like `aws ecr get-login-password`.

---

## Hands-on lab

1. Run the demo and compare it with the sample. Open the **Summary** page and the **Annotations** box.
2. Remove the `uuidgen` / `stop-commands` lines and re-run. The fake title now becomes a real **warning annotation**. That's the injection risk.
3. Write a 3-line value to `$GITHUB_OUTPUT` **without** the delimiter syntax and read the `Invalid format` error.
4. Add `- run: echo "::set-output name=x::y"` and read the **deprecation warning** in the annotations (the command still works today; compare with `::set-env`, which is rejected).

---

## Where you'll use it

- **Build metadata** to later steps (tags, versions, timestamps).
- **Tool installs** (`GITHUB_PATH`) in custom setup steps.
- **PR feedback:** linters and test reporters emit `::error file=…,line=…::` so problems appear inline on the diff.
- **Release dashboards:** `GITHUB_STEP_SUMMARY` with coverage, scan results and deployed URLs.
- **Security:** `::add-mask::` for derived secrets, `::stop-commands::` for untrusted text.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Variable empty in the same step | `GITHUB_ENV` applies to **later** steps. | Use a shell variable now, the env file for later. |
| Variable empty in another job | Env files are per job. | Use job outputs (topic 21). |
| `Unable to process file command 'env' successfully` / `Invalid format` | A multi-line value, or a stray newline in `key=value`. | Use the `NAME<<EOF … EOF` syntax. Make sure the delimiter doesn't appear inside the value. |
| `The 'set-output' command is deprecated and will be disabled soon` / it has no effect | Old syntax. | `>> "$GITHUB_OUTPUT"`. |
| `The 'add-path' command is disabled` | Old syntax. | `>> "$GITHUB_PATH"`. |
| Tool still "not found" after `GITHUB_PATH` | Same step, or a wrong folder. | Use it in the next step. `ls` the folder. |
| Summary missing | Wrote to `GITHUB_STEP_SUMMARY` with `>` (overwrite) in several steps, or it's over 1 MiB. | Use `>>`. Keep summaries concise. |
| Random annotations or "commands" appear | Untrusted text was printed. | Wrap it in `::stop-commands::`, and pass untrusted values via `env:` (topic 37). |

---

## Quick recap

- Environment files: `GITHUB_ENV` (later steps' env), `GITHUB_OUTPUT` (step outputs), `GITHUB_PATH` (PATH), `GITHUB_STEP_SUMMARY` (Markdown summary), `GITHUB_STATE` (action post steps).
- Workflow commands: `::error/warning/notice::`, `::group::`, `::add-mask::`, `::debug::`, `::stop-commands::`.
- `set-env` and `add-path` are **disabled**; `set-output` and `save-state` are **deprecated** (still work with a warning).
- Multi-line values need the `<<DELIMITER` syntax.

**Interview questions**

1. **Q: How do you make a tool you installed available to later steps?**
   A: `echo "/path/to/bin" >> "$GITHUB_PATH"`.
2. **Q: Why were `set-env` and `add-path` disabled?**
   A: They were parsed from log output, so untrusted text in logs could set environment variables or PATH (a code-injection risk). File-based commands replaced them.
3. **Q: How do you show a result table on the workflow run page?**
   A: Append Markdown to `$GITHUB_STEP_SUMMARY`.
