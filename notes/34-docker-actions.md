# 34 · Docker Container Actions (and Which Action Type to Choose)

> **Part F: Custom actions** · Previous: [33 · JavaScript actions](33-javascript-actions.md) · Next: [35 · Publishing and versioning actions](35-publishing-and-versioning-actions.md) · [Back to INDEX](INDEX.md)
>
> **Video:** [GitHub JavaScript & Docker Actions Explained | Hands-on Demos](https://www.youtube.com/watch?v=sjSmSzMJL5s) (Demo 2)
>
> **Code:** [`code/34-docker-actions/`](code/34-docker-actions/)

---

## What it is

A **Docker action** packages the automation logic **and its whole execution environment** (operating system, runtime, tools, libraries and your code) into a **Docker image**. GitHub builds or pulls that image on the runner, starts a container, and runs its **`ENTRYPOINT`**.

```yaml
runs:
  using: docker
  image: Dockerfile          # build from this file (or: docker://alpine:3.22 to pull)
  args:
    - ${{ inputs.name }}
```

Unlike JavaScript actions (which rely on the runner's Node.js), Docker actions bring **everything** with them, so they're **language-agnostic**: Bash, Python, Java, Go, Node.js, Ruby, PHP, Rust, C++, anything the image contains.

> **Mental model:**
>
> ```text
> Reuse Existing Workflow Steps → Composite Action
>
> Write New Automation Logic → JavaScript Action
>
> Package the Entire Execution Environment → Docker Action
> ```

---

## Why it exists

Some automation needs a **custom runtime, special software, OS utilities, or a language other than JavaScript**, which are awkward or undesirable to install on the runner every time. Package it once in an image and it runs **the same way on every runner**:

```text
Ubuntu → Python → Terraform → kubectl → Helm → Custom Application
```

```text
Alpine Linux → Java → Maven → Trivy → Custom Application
```

> **Recommendation:** choose a Docker action **only** when you need a **custom execution environment, specialized dependencies, OS utilities, or a non-JavaScript language** that composite or JavaScript actions can't conveniently provide.

> **Production insight:** instead of installing dependencies on every run, they're baked into the image once and reused. That gives **consistent, portable, reproducible** automation.

> **Performance consideration:** Docker actions **start more slowly** than JavaScript actions, because GitHub may need to **build** or **pull** the image first.

> **Platform limitation:** Docker actions run **only on Linux runners** (the container runtime is Linux-based), not on Windows or macOS runners. Self-hosted Linux runners must have Docker installed and configured.

> **Key takeaway:** Docker actions give the **most flexibility** but should usually be the **last choice**, used only when composite or JavaScript actions can't meet the runtime, dependency or OS needs.

**Jenkins comparison:** like `agent { docker { image '…' } }` scoped to **one step** instead of a whole stage.

---

## Analogy

A **food truck** vs cooking in someone else's kitchen:

- A JavaScript action cooks in the **runner's kitchen**, using the stove that's already there (Node.js).
- A Docker action **drives up its own food truck**: its own stove, fridge and spices (OS, runtime, tools). It parks in the runner's car park, plugs into the runner's power (Docker on the VM), and cooks the same dish identically everywhere. Setting up the truck takes longer (image build or pull).
- The runner still owns the car park. The truck **doesn't replace the runner**.

---

## How it works inside

```text
step: uses: ./.github/actions/docker-system-info   with: name: Cloud With VarJosh
  "Set up job": for image: Dockerfile → docker build -t <hash> .github/actions/docker-system-info
                for image: docker://… → docker pull
  step runs:
    docker run --rm \
      --workdir /github/workspace \
      -v <runner workspace>:/github/workspace \           ← your checked-out repo
      -v <runner temp>/_github_home:/github/home \
      -v <runner temp>/_runner_file_commands:/github/file_commands \  ← GITHUB_OUTPUT etc.
      -e INPUT_NAME="Cloud With VarJosh" -e GITHUB_* -e RUNNER_* … \
      <image> "Cloud With VarJosh"                         ← args → $1, $2 …
  container exits → exit code = step result → container removed
```

Consequences:

- The repo is mounted at **`/github/workspace`**, so the action can read and modify repo files.
- Inputs arrive **both** as `INPUT_<NAME>` environment variables **and** (if you list them) as `args`.
- To set outputs, write to `$GITHUB_OUTPUT` **inside the container**. It's mounted, so it works.
- The container runs as **root** by default. Files it creates in the workspace are root-owned (this can bite later steps on self-hosted runners).

> **Important:** a Docker action **doesn't replace the runner**. GitHub still provisions the VM from `runs-on`. The **runner orchestrates** (builds the image, starts the container), and the **container executes** the action's logic.

---

## Workflow YAML / commands

### Demo 2: a Docker action

#### Step 1: Repository

Repo and SSH as in [topic 02](02-git-and-github-recap.md). Workflows are discovered and evaluated when pushed.

#### Step 2: Structure

```text
Demo-02
├── .github
│   ├── actions
│   │   └── docker-system-info
│   │       ├── action.yml
│   │       ├── Dockerfile
│   │       └── entrypoint.sh
│   └── workflows
│       └── docker-action-demo.yaml
```

- Each sub-folder of `.github/actions` is **one local action** with its own `action.yml`.
- Docker actions also have a **Dockerfile** that defines the image.
- Every Docker action defines a **Docker `ENTRYPOINT`**, its **starting point** and **automation logic**. Here it's `entrypoint.sh`. In production it could be `python app.py`, `java -jar app.jar`, a Go binary (`./my-application`), a Node.js app, or anything else.

> **Note:** production Docker actions may include more scripts, Python programs, config and docs. However big it is, GitHub always **builds the image, starts a container, and runs the entry point**.

#### Step 3: `Dockerfile`

```dockerfile
FROM alpine:3.22

COPY entrypoint.sh /entrypoint.sh

RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

- **`FROM alpine:3.22`**: the base image. **Alpine Linux** is a tiny distribution popular for images.
- **`COPY entrypoint.sh /entrypoint.sh`**: copy the script with the automation logic into the image.
- **`RUN chmod +x /entrypoint.sh`**: runs **at build time**. It makes the script executable.
- **`ENTRYPOINT ["/entrypoint.sh"]`**: the command run **whenever a container starts** from this image.

```text
Build Docker Image → Start Container → Execute ENTRYPOINT
```

> **Observation:** JavaScript actions run directly on the runner's Node.js. Docker actions first **build an image**, then **start a container**, then run the **ENTRYPOINT**.

#### Step 4: `entrypoint.sh`

```bash
#!/bin/sh

NAME="$1"

echo "========================================"
echo "GitHub Docker Action"
echo "========================================"

echo
echo "Hello ${NAME}!"
echo

echo "Container Information"
echo "----------------------------------------"
echo "OS Information"
cat /etc/os-release
echo
echo "Current User     : $(whoami)"
echo "Working Directory: $(pwd)"
```

- **`#!/bin/sh`** (the **shebang**): run with the **Bourne shell `sh`** (Alpine has no Bash by default).
- **`NAME="$1"`**: `$1` is the **first command-line argument**. GitHub passes the caller's value there (via `args` in `action.yml`):

  ```text
  Calling Workflow (with.name) → action.yml (args) → entrypoint.sh ($1) → NAME
  ```

- The rest prints information **from inside the container** (`/etc/os-release`, user, working directory), which proves it runs in its own isolated environment, not on the runner directly.

> **Note:** the script is deliberately simple. In production, the entry point might run Python, Java, Node.js or Go apps, or a complex shell script that orchestrates many tools: provisioning cloud infrastructure, building and publishing artifacts, running security scans, and so on.
>
> **Key takeaway:** the goal is to understand **how a Docker action runs inside a container** and **how it gets inputs** from the caller.

#### Step 5: `action.yml`

[`action.yml`](code/34-docker-actions/.github/actions/docker-system-info/action.yml):

```yaml
name: Docker System Info
description: Demonstrates a simple Docker Action

inputs:
  name:
    description: Name to display
    required: true

runs:
  using: docker
  image: Dockerfile

  args:
    - ${{ inputs.name }}
```

- **`using: docker`**: this is a Docker action.
- **`image: Dockerfile`**: build an image from this file **on the runner**, then start a container from it. (Alternative: `image: docker://ghcr.io/org/tool:1.2` to **pull** a prebuilt image, which starts faster.)

  ```text
  GitHub Workflow → GitHub-hosted Runner (runs-on, from workflow yaml) → Build Docker Image (Dockerfile) → Start Docker Container → Execute Docker ENTRYPOINT
  ```

- **`args:`**: command-line arguments for the container. `${{ inputs.name }}` becomes **`$1`** in `entrypoint.sh`.

  ```text
  Calling Workflow (with.name) → Action Input → Docker Container → entrypoint.sh ($1)
  ```

> **Observation:** for **all three** action types, inputs come from the caller. Only **how the action consumes them** differs (`${{ inputs.x }}` in composite, `core.getInput` in JavaScript, `$1` / `$INPUT_X` in Docker).

#### Step 6: The calling workflow

[`.github/workflows/docker-action-demo.yml`](code/34-docker-actions/.github/workflows/docker-action-demo.yml):

```yaml
name: Docker Action Demo

on:
  workflow_dispatch:

jobs:
  demo-job:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout Repository
        uses: actions/checkout@v7

      - name: Execute Docker Action
        uses: ./.github/actions/docker-system-info

        with:
          name: Cloud With VarJosh
```

- `uses:` points at the local Docker action, and `with:` supplies inputs.
- **No `npm install`**: Docker actions carry everything in the image.
- GitHub automatically does `Build Docker Image → Start Container → Execute ENTRYPOINT`, and passes the `with:` value as a command-line argument.

  ```text
  Calling Workflow (with.name) → Docker Action → entrypoint.sh ($1) → Workflow Logs
  ```

> **Observation:** to the caller, a Docker action is just another `uses:`. GitHub handles building, starting and running.

#### Steps 7–8: Commit, push, run

```bash
git add .
git commit -m "Demo: Add Docker Action demo"
git push origin main
```

GitHub discovers the workflow under **Actions**. It's `workflow_dispatch`, so start it manually: **Docker Action Demo → Run workflow**. GitHub builds the image, starts a container, and runs `entrypoint.sh`.

#### Step 9: Observe (sample log output)

```text
Set up job
  Build container for action use: '/home/runner/work/repo/repo/./.github/actions/docker-system-info/Dockerfile'.
  /usr/bin/docker build -t 2a7c3b:9f… -f "…/Dockerfile" "…/docker-system-info"
  #5 [1/3] FROM docker.io/library/alpine:3.22
  …
Execute Docker Action
  /usr/bin/docker run --name 2a7c3b… --workdir /github/workspace --rm -e "INPUT_NAME" … -v "/home/runner/work/repo/repo":"/github/workspace" 2a7c3b:9f… "Cloud With VarJosh"
  ========================================
  GitHub Docker Action
  ========================================

  Hello Cloud With VarJosh!

  Container Information
  ----------------------------------------
  OS Information
  NAME="Alpine Linux"
  ID=alpine
  VERSION_ID=3.22.5
  PRETTY_NAME="Alpine Linux v3.22"
  HOME_URL="https://alpinelinux.org/"
  BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"

  Current User     : root
  Working Directory: /github/workspace
```

- The image was **built in "Set up job"**, before any step ran.
- `docker run … -v …:/github/workspace`: the runner's workspace is mounted in.
- `Working Directory: /github/workspace`: GitHub mounts the checked-out repo here, so the action can read and modify repo files from inside the container.
- `Current User: root`: containers run as root by default.

> **Observation:** composite, JavaScript and Docker actions are all consumed with the same `uses:` syntax. The Docker action runs in **its own container environment** (OS, runtime, tools, code), and full control over the environment is its key advantage.

---

### Which custom action should you use?

Choose the **simplest type that meets the need**. Most organizations progress **composite → JavaScript → Docker** only as extra capability becomes necessary.

```text
Composite Action  → Reuse Existing GitHub Actions, Shell Commands & Scripts

JavaScript Action → Write New Automation Logic using the Node.js Runtime

Docker Action     → Package Your Own Runtime, Tools & Dependencies
```

| Action type | What you write | What executes |
|---|---|---|
| **Composite action** | Workflow steps (actions, shell commands, scripts) | Existing steps run directly on the **runner** |
| **JavaScript action** | JavaScript or TypeScript | JavaScript runs on the runner's **Node.js runtime** |
| **Docker action** | Any executable (Shell, Python, Java, Go, Node.js, …) | The executable runs **inside a Docker container** created on the runner |

| Requirement | Recommended action |
|---|---|
| Reuse existing actions, shell commands or scripts | **Composite** |
| Standardize common CI/CD steps across workflows or repos | **Composite** |
| Implement custom programming logic | **JavaScript** |
| Complex data processing or advanced automation logic | **JavaScript** |
| Use the Actions Toolkit or GitHub APIs | **JavaScript** |
| Package custom runtimes, tools or specialized dependencies | **Docker** |
| Use languages other than Node.js | **Docker** |
| Require a consistent, isolated execution environment | **Docker** |

```text
Need to reuse existing workflow steps?
        ↓
Composite Action

Need to write new automation logic?
        ↓
JavaScript Action

Need your own runtime, tools or execution environment?
        ↓
Docker Action
```

> **Rule of thumb:** **start with composite** (the simplest to write and maintain). Move to **JavaScript** for custom logic that can't be cleanly built from existing steps. Choose **Docker** only for a **custom runtime, specialized tools or environment** the hosted runner can't provide.

| 🆕 Extra comparison | Composite | JavaScript | Docker |
|---|---|---|---|
| Runs on Windows/macOS runners | ✓ | ✓ | ✗ (Linux only) |
| Start-up speed | fast | fast | slower (build/pull) |
| Needs `shell:` on `run` steps | yes | n/a | n/a |
| Has pre/post hooks | no | yes (`pre:`, `post:`) | yes (`pre-entrypoint`, `post-entrypoint`) |

---

## Hands-on lab

1. Build the demo and match the log. Find the `docker build` and `docker run -v …:/github/workspace` lines.
2. In `entrypoint.sh` add `ls -la` and `echo "INPUT_NAME env: $INPUT_NAME"` to show the workspace mount and that inputs also come as env vars.
3. Write an output from the container: `echo "os=$(. /etc/os-release; echo $ID)" >> "$GITHUB_OUTPUT"`, give the step an `id`, and print it in the next step.
4. Run the job on `windows-latest` and read the Linux-only error.
5. Push the image to GHCR and switch to `image: docker://ghcr.io/<you>/system-info:1.0`. Compare the start-up time.

---

## Where you'll use it

- Tools that need a **specific OS/toolchain** (Terraform + kubectl + Helm + Python together).
- **Non-JavaScript** custom logic (a Go or Python internal CLI).
- Many security scanners are distributed this way (container-based actions).
- Reading the log to understand why a third-party Docker action is slow (image build time).

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Container action is only supported on Linux` | Used on Windows or macOS. | Use an `ubuntu-*` runner. |
| `exec /entrypoint.sh: permission denied` | Script not executable. | `RUN chmod +x`, or `git update-index --chmod=+x entrypoint.sh`. |
| `exec /entrypoint.sh: no such file or directory` even though it exists | Windows line endings (CRLF) or a missing interpreter (`#!/bin/bash` on Alpine). | Convert to LF. Use `#!/bin/sh`, or install bash. |
| Slow step every run | Image rebuilt each time. | Publish a prebuilt image and use `image: docker://…`. |
| Later steps get `Permission denied` on files | The container created root-owned files in the workspace. | `chown` inside the action, or run as a non-root user. |
| Input value is empty | Not listed in `args`, and the script reads `$1`. | Add it to `args`, or read `$INPUT_NAME`. |

---

## Quick recap

- Docker action = `runs.using: docker` + `image:` (Dockerfile or `docker://`) + optional `args`. The container runs the `ENTRYPOINT`.
- The runner still exists. It builds or pulls and runs the container, with the repo mounted at `/github/workspace`.
- Inputs → `args` (`$1`) and `INPUT_*` env vars. Outputs → `$GITHUB_OUTPUT` (mounted).
- Linux only, slower start, most flexible. **Choose composite → JavaScript → Docker** in that order.

**Interview questions**

1. **Q: Why are Docker actions Linux-only?**
   A: They run as Linux containers via the runner's Docker engine. Hosted Windows and macOS runners don't provide that.
2. **Q: How does a Docker action access the repository files?**
   A: GitHub mounts the runner's workspace into the container at `/github/workspace` (the default working directory).
3. **Q: Composite, JavaScript or Docker for "run our internal Go CLI with a specific Terraform version"?**
   A: Docker (or a composite action that installs the tools). Docker bakes in the exact runtime and dependencies.

---

### References

- GitHub: [Actions docs](https://docs.github.com/actions) · [Composite action](https://docs.github.com/actions/sharing-automations/creating-actions/creating-a-composite-action) · [JavaScript action](https://docs.github.com/actions/sharing-automations/creating-actions/creating-a-javascript-action) · [Docker action](https://docs.github.com/actions/sharing-automations/creating-actions/creating-a-docker-container-action) · [Actions Toolkit](https://github.com/actions/toolkit)
- Docker: [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) · [Docker overview](https://docs.docker.com/get-started/docker-overview/)
