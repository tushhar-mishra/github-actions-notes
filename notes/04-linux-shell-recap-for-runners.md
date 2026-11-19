# 04 · Linux Shell Recap for Runners

> **Part A: Foundations** · Previous: [03 · YAML](03-yaml-for-github-actions.md) · Next: [05 · What is GitHub Actions?](05-what-is-github-actions.md) · [Back to INDEX](INDEX.md)
>
> 🆕 New topic. This is a short recap from your Linux notes of exactly what every `run:` step depends on.

---

## What it is

A **runner** is the machine that runs your workflow (topic 07). Most runners are **Linux** (Ubuntu). A `run:` step is just **shell commands** that GitHub types into a terminal on that machine for you.

- A **shell** is the program that reads your commands and runs them. On Linux runners the default is **Bash** (Bourne Again SHell).
- An **environment variable** is a named value the shell and programs can read, like `HOME=/home/runner`.
- An **exit code** is a number every command returns when it finishes. **`0` = success. Anything else = failure.**

> **The single most important rule in this course:** GitHub Actions decides a step passed or failed **only** by looking at the exit code of its shell. Exit 0 → ✅. Non-zero → ❌, and the job stops.

---

## Why it exists

You will write things like `pip install`, `docker build`, `kubectl apply`. To debug them you must know:

- why a step failed (exit codes)
- why a variable is empty (environment variables and scope)
- why a command was "not found" (`PATH`)
- why an error in the middle of a pipe was ignored (`pipefail`)

---

## Analogy

The shell is a **cook following a recipe card** (your `run:` block). After each instruction, the cook either nods (exit code 0) or shouts "it burned!" (non-zero). GitHub is the **head chef**. The moment the cook shouts, the head chef stops that recipe and marks the dish as failed.

---

## How it works inside

When a `run:` step starts on a Linux runner:

1. GitHub writes your `run:` text into a **temporary script file** (in a temp folder on the runner).
2. It starts Bash with `bash -e <that file>`. If you set `shell: bash` explicitly, it uses `bash --noprofile --norc -eo pipefail <file>` instead.
   - `-e` = **exit on the first command that fails**.
   - `-o pipefail` = a pipeline `a | b` fails if **any** part fails, not just the last one.
3. Bash runs the commands in order.
4. When the script ends, Bash's exit code becomes the **step result**.
5. Everything the commands print (**stdout** = normal output, **stderr** = error output) streams into the step's log.

Each `run:` step is a **new shell process**. A variable you `export` in step 1 is **gone** in step 2. (Topic 23 shows the official way to pass values between steps: `$GITHUB_ENV`.)

---

## Workflow YAML / commands

### Exit codes

```bash
ls /tmp ; echo "exit code: $?"          # $? = exit code of the last command
ls /nope ; echo "exit code: $?"
```

```text
exit code: 0
ls: cannot access '/nope': No such file or directory
exit code: 2
```

### Environment variables

```bash
export APP_ENV=dev        # create a variable that child programs can see
echo "$APP_ENV"           # read it → dev
echo "$HOME"              # a built-in one → /home/runner on hosted Linux runners
env | sort | head         # list all variables
```

### PATH: why "command not found" happens

`PATH` is a list of folders the shell searches for programs.

```bash
echo "$PATH"            # /usr/local/bin:/usr/bin:/bin:...
which python3           # shows which file runs when you type python3
```

If a tool is installed somewhere that isn't in `PATH`, you get `command not found`. (In topic 23 you'll add folders to `PATH` for later steps with `$GITHUB_PATH`.)

### Chaining and pipes

```bash
cd app && make          # run make only if cd succeeded
make || echo "failed"   # run echo only if make failed
cat log.txt | grep ERROR   # pipe: output of cat becomes input of grep
```

### Safe script header (good for long scripts in workflows)

```bash
set -euo pipefail
# -e  stop on first error
# -u  treat unset variables as errors (catches typos like $APP_EVN)
# -o pipefail  a failing command anywhere in a pipe fails the pipe
```

### The commands you'll see most on runners

| Command | Used for |
|---|---|
| `sudo apt-get update && sudo apt-get install -y jq` | Install packages. Hosted runners allow `sudo` with no password. |
| `curl -fsSL <url>` | Download things. `-f` fails on HTTP errors, which matters for exit codes. |
| `chmod +x script.sh` | Make a script executable. |
| `mkdir -p a/b/c` | Create nested folders. |
| `cat`, `grep`, `ls -la`, `pwd` | Inspect files while debugging. |
| `docker ...`, `kubectl ...` | Recapped in topics 19 and 49+. |

### A step that shows all of this

```yaml
- name: Shell basics on a runner
  run: |
    set -euo pipefail
    echo "I am $(whoami) on $(uname -s)"
    echo "Working directory: $(pwd)"
    export GREETING=hello
    echo "$GREETING"
    ls does-not-exist        # this fails → step fails here
    echo "never printed"
```

**Sample log output:**

```text
Run set -euo pipefail
I am runner on Linux
Working directory: /home/runner/work/cwvj-gha-practice/cwvj-gha-practice
hello
ls: cannot access 'does-not-exist': No such file or directory
Error: Process completed with exit code 2.
```

- `runner`: the Linux user that runs jobs on GitHub-hosted runners.
- `/home/runner/work/<repo>/<repo>`: the **workspace**, where your code is checked out. It's also in the variable `$GITHUB_WORKSPACE`.
- `Error: Process completed with exit code 2.`: GitHub saw a non-zero exit code and failed the step. `never printed` didn't run because of `-e`.

---

## Hands-on lab

On any Linux machine (or WSL, Windows Subsystem for Linux):

1. Run `false; echo $?` and `true; echo $?`. Which number does each print?
2. Run `false | true; echo $?`, then `set -o pipefail; false | true; echo $?`. Explain the difference.
3. Save the "step that shows all of this" script as `test.sh`, run `bash -e test.sh`, and compare the output to the sample log.

---

## Where you'll use it

- **Every `run:` step** you write.
- **Debugging failures.** The log line `Process completed with exit code N` is your first clue.
- **Self-hosted runners** (topic 38), where *you* maintain the Linux machine: users, packages, `PATH`, disk space.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Error: Process completed with exit code 1.` | Some command in the step returned non-zero. | Scroll up in the log to the first error line. |
| `command not found` | The tool isn't installed, or it isn't in `PATH`. | Install it in an earlier step (or use a `setup-*` action, topic 15). Check `which`. |
| Variable is empty in the next step | Each step is a new shell, so `export` doesn't carry over. | Use `$GITHUB_ENV` (topic 23). |
| A step passed even though `grep` found nothing important | You ended with `|| true`, or a pipe hid the failure. | Add `set -o pipefail`. Don't hide errors with `|| true` by accident. |
| `Permission denied` running `./script.sh` | The file isn't executable. | `chmod +x script.sh`, or run `bash script.sh`. |

---

## Quick recap

- A `run:` step = a temporary Bash script. **Exit code 0 = pass**, anything else = fail.
- Default: `bash -e`. With `shell: bash`: `bash --noprofile --norc -eo pipefail`.
- Each step is a **new shell**, so exported variables don't carry over.
- `PATH` decides which programs can be found.
- Your code lives in `$GITHUB_WORKSPACE` (`/home/runner/work/<repo>/<repo>`).

**Interview questions**

1. **Q: How does GitHub Actions decide that a `run:` step failed?**
   A: From the shell's exit code. Non-zero means failure.
2. **Q: Why might a failing command in a pipe not fail the step?**
   A: Without `pipefail`, a pipe's exit code is only the last command's. Use `set -o pipefail`, or `shell: bash`, which enables it.
3. **Q: I `export`ed a variable in step 1. Why is it empty in step 2?**
   A: Every step runs in a new shell process. Use `echo "NAME=value" >> "$GITHUB_ENV"`.
