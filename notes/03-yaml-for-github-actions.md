# 03 · YAML for GitHub Actions

> **Part A: Foundations** · Previous: [02 · Git and GitHub recap](02-git-and-github-recap.md) · Next: [04 · Linux shell recap for runners](04-linux-shell-recap-for-runners.md) · [Back to INDEX](INDEX.md)
>
> Extra material: [YAML video](https://youtu.be/LT3U9fkOVDQ), [YAML notes](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2009). This topic covers exactly the YAML you need for workflows.

---

## What it is

**YAML** stands for **"YAML Ain't Markup Language"**. It's a recursive joke name. YAML is a simple text format for writing **structured data** (settings, lists, nested groups) in a way humans can read.

Every GitHub Actions workflow is a `.yml` or `.yaml` file. Both extensions work.

YAML only has three building blocks:

1. **Key–value pairs (maps):** `name: CI`
2. **Lists (sequences):** lines starting with `- `
3. **Scalars:** single values like text, numbers, `true` or `false`

**Indentation** (spaces at the start of a line) shows what belongs to what.

---

## Why it exists

Configuration used to be written in XML (lots of `<tags>`) or JSON (lots of `{}` and `""`). Both are hard to read and edit by hand. YAML says the same thing with far fewer symbols. That's why GitHub Actions, Kubernetes, Docker Compose, Ansible, Helm, Argo CD, GitLab CI/CD and Azure Pipelines all use it.

**Jenkins comparison:** a Jenkinsfile is written in **Groovy** (a programming language), so it can contain loops and real code. A GitHub Actions workflow is **YAML**, which is just data. Logic like conditions and loops comes from GitHub's own **expressions** (topic 11) and **matrix** (topic 26), not from YAML.

---

## Analogy

YAML is like a **shopping list with sub-lists**:

```text
Groceries:
  Fruit:
    - apples
    - bananas
  Dairy:
    - milk
```

The **indent** shows that apples belong under Fruit, and Fruit belongs under Groceries. Change the indent and you change the meaning, just like moving "milk" under "Fruit" would be wrong.

---

## How it works inside

When you push a workflow file, GitHub **parses** it (reads the text and turns it into a data tree) **before** running anything.

```text
your .yml text ──► YAML parser ──► data tree ──► GitHub checks the tree against
                                                  the workflow schema (allowed keys)
                                                        │
                                   ┌────────────────────┴────────────────────┐
                                   ▼                                         ▼
                             valid → workflow can run               invalid → run fails at once:
                                                                    "Invalid workflow file"
```

This means a YAML mistake **never reaches a runner**. The run fails instantly with an error that points to a line number.

---

## Workflow YAML / commands

### 1. Maps (key: value)

```yaml
name: My First Workflow     # key "name", value "My First Workflow"
```

- A **space after the colon is required**. `name:CI` is not a key–value pair.

### 2. Nested maps (indentation)

```yaml
jobs:                       # "jobs" holds a map
  build:                    # 2 spaces in: "build" belongs to jobs
    runs-on: ubuntu-latest  # 4 spaces in: belongs to build
```

- Use **spaces only. Tabs are not allowed** in YAML.
- Use the same number of spaces at each level. 2 is the convention.

### 3. Lists (sequences)

```yaml
steps:
  - name: Step 1            # "- " starts a list item. This item is a map with 2 keys
    run: echo "one"
  - name: Step 2            # a second item
    run: echo "two"
```

- Each `-` is **one item**. `name` and `run` line up under each other, so they're keys of the **same** item.

Short "flow" style lists and maps (JSON-like) also work:

```yaml
branches: [main, develop]          # list on one line
with: { python-version: "3.13" }   # map on one line
```

### 4. Scalars and quoting

```yaml
retention: 7          # number
enabled: true         # boolean
version: "3.10"       # string. Quotes are needed! Unquoted 3.10 becomes the number 3.1
cron: "*/5 * * * *"   # must quote: a value starting with * has a special meaning in YAML
message: "Deploy: done"   # quote values that contain ": "
```

### 5. Multi-line strings (you'll use these in `run:` all the time)

```yaml
run: |                 # | = "literal block": keep the line breaks exactly
  echo "line one"
  echo "line two"

description: >         # > = "folded block": join lines with spaces into one line
  This is one long
  sentence.
```

With `|` each line becomes a separate shell command, which is what you want for scripts.

### 6. Comments

```yaml
# Anything after # is ignored
on: push   # comments can go at the end of a line too
```

### 7. The `on:` gotcha

In an old YAML version (YAML 1.1), `on`, `yes`, `no`, `off` meant `true` or `false`. GitHub's parser handles `on:` correctly in workflows, but generic YAML tools may show it as `true:`. Don't let that confuse you. It's the same file.

### A complete example, annotated

```yaml
name: YAML Demo                 # map key at the top level
on:                             # map: which events start the workflow
  push:
    branches: [main]            # flow-style list
jobs:                           # map of jobs
  hello:                        # job id (you choose it)
    runs-on: ubuntu-latest
    steps:                      # list of steps
      - name: Multi-line script
        run: |                  # literal block → two commands
          echo "Hello"
          echo "Version is ${{ '3.10' }}"
```

**Sample log output:**

```text
Run echo "Hello"
Hello
Version is 3.10
```

`${{ ... }}` is a GitHub **expression**, which GitHub evaluates before running the step. It's explained in topic 11. Here it just prints a string.

---

## Hands-on lab

1. Install a YAML checker locally, or paste into any online YAML validator.
2. Write the "complete example" above, then **break it on purpose** in three ways:
   - replace two spaces with a **tab**
   - remove the space after `runs-on:`
   - change `version: "3.10"` to `version: 3.10` and see what value you get
3. Later (topic 08) you'll push a broken file and see GitHub's real "Invalid workflow file" error.

---

## Where you'll use it

- **Every** GitHub Actions file: workflows (`.github/workflows/*.yml`) and action metadata (`action.yml`, topic 32).
- **Kubernetes manifests** in the capstone: same YAML rules.
- **Code review.** Most "my workflow doesn't work" problems turn out to be indentation.

---

## When it breaks

| Error message (what GitHub shows) | Meaning | Fix |
|---|---|---|
| `Invalid workflow file: .github/workflows/ci.yml#L12` · `You have an error in your yaml syntax on line 12` | The YAML itself can't be parsed. Usually a tab, a missing space after `:`, or broken indentation. | Look at the line **and the line above it**. |
| `Unexpected value 'step'` | The YAML is valid but uses a key GitHub doesn't know (typo: `step` instead of `steps`). | Fix the key name. |
| `Required property is missing: runs-on` | Indentation moved `runs-on` out of the job. | Align it under the job id. |
| Python version `3.1` installed instead of `3.10` | Unquoted number. | Quote version strings: `"3.10"`. |
| Script runs as one weird line | You used `>` instead of `|`. | Use `|` for `run:` scripts. |

Tip: [`actionlint`](https://github.com/rhysd/actionlint) checks workflow YAML **and** GitHub-specific rules before you push. Covered in topic 39.

---

## Quick recap

- YAML = YAML Ain't Markup Language: maps, lists, scalars, and indentation with **spaces only**.
- `- ` starts a list item. `key: value` needs a space after the colon.
- Quote versions (`"3.10"`), cron strings, and anything with `: ` or a leading `*`.
- Use `|` for multi-line scripts.
- YAML errors fail the run instantly, before any runner starts.

**Interview questions**

1. **Q: Why might `python-version: 3.10` install Python 3.1?**
   A: Unquoted, YAML reads 3.10 as the number 3.1. Quote it: `"3.10"`.
2. **Q: What's the difference between `|` and `>`?**
   A: `|` keeps the line breaks (good for scripts). `>` folds the lines into one.
3. **Q: Can a YAML error consume runner minutes?**
   A: No. GitHub validates the file before it queues any job.
