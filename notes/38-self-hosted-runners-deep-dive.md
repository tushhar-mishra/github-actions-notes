# 38 · Self-Hosted Runners Deep Dive (Install, Labels, Groups, Ephemeral, ARC)

> **Part G: Security and production** · Previous: [37 · Security hardening](37-security-hardening.md) · Next: [39 · Debugging and troubleshooting](39-debugging-and-troubleshooting.md) · [Back to INDEX](INDEX.md)
>
> 🆕 New topic. Topic 07 explained **what** self-hosted runners are and **when** to choose them (from lectures 01–02). This topic shows **how** to run them in production.

---

## What it is

A **self-hosted runner** is a machine **you** manage (VM, bare metal, container, or Kubernetes pod) running GitHub's open-source **runner application** (`actions/runner`). It connects **outbound** to GitHub, waits for jobs whose `runs-on` labels match, and runs them.

Key vocabulary:

| Term | Meaning |
|---|---|
| **Registration** | Linking a runner to a **repo**, **organization** or **enterprise** with a short-lived registration token. |
| **Labels** | Tags like `self-hosted`, `linux`, `x64`, `gpu`, `prod-vpc` that jobs target with `runs-on`. |
| **Runner group** | An org/enterprise-level collection of runners with an **access policy** (which repos and workflows may use them). |
| **Ephemeral runner** | Accepts **one job**, then deregisters, so it's a fresh machine every time. |
| **ARC** (Actions Runner Controller) | A Kubernetes operator that creates runner **pods on demand** and deletes them after each job (auto-scaling, ephemeral). |

---

## Why it exists

(From topic 07.) Hosted runners can't reach your **private network**, can't use your **GPUs or licensed software**, may not meet **compliance** rules, and can get expensive at scale. Self-hosted solves that, but **you** take on patching, scaling, monitoring and, above all, **security**.

**Jenkins comparison:** this is exactly a **Jenkins agent**. `config.sh` ≈ adding a node, labels ≈ node labels, ARC ≈ the **Kubernetes plugin** spinning up agent pods per build.

---

## Analogy

**Hiring your own delivery drivers** instead of using a courier company:

- **Registration** = giving a driver a company ID badge.
- **Labels** = their licence types ("van", "refrigerated", "hazmat").
- **Runner groups** = which shops each driver is allowed to deliver for.
- **Persistent runner** = the same van every day (it can collect rubbish, or worse, a stowaway).
- **Ephemeral runner** = a **fresh rental van for every delivery**, returned afterwards.
- **ARC** = a dispatch office that rents exactly as many vans as there are parcels right now.

---

## How it works inside

```text
1. ./config.sh --url https://github.com/ORG/REPO --token <registration-token> --labels gpu,prod-vpc
      → registers; stores credentials in .runner / .credentials (keep these private)
2. ./run.sh (or as a service)
      → long-polls GitHub over HTTPS 443 (OUTBOUND only; no inbound ports needed)
3. job queued with runs-on: [self-hosted, linux, gpu]
      → GitHub offers it to an IDLE runner having ALL those labels and allowed by its group
4. runner downloads the job, runs steps in _work/<repo>/<repo>, streams logs
5. persistent: stays registered, workspace mostly reused (left-over files, Docker images, caches!)
   ephemeral (--ephemeral): deregisters after one job; your automation creates a new one
6. runner app auto-updates itself (must stay within the supported version window)
```

Required outbound access: `github.com`, `api.github.com`, `*.actions.githubusercontent.com`, plus artifact/cache storage hosts (see "Self-hosted runner communication" in the docs).

**Security model:** a job can run **any code** from the workflow (and from a PR, if the workflow runs on PRs). On a **persistent** runner that code can leave backdoors, read other jobs' files, or steal credentials on the machine. Hence:

- ❌ **Never attach self-hosted runners to public repositories** (anyone can open a PR).
- ✅ Prefer **ephemeral** runners (one job, then destroy).
- ✅ Use **runner groups** to limit which repos or workflows can use privileged runners (e.g. the prod-VPC runner).
- ✅ Run the runner as a **non-root** user, give the host only the cloud permissions it needs, and isolate its network.

---

## Workflow YAML / commands

### 1. Install a runner on a Linux VM (repo-level)

**Repository → Settings → Actions → Runners → New self-hosted runner → Linux x64** shows commands like:

```bash
# as a dedicated non-root user, e.g. "gha"
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v<VERSION>/actions-runner-linux-x64-<VERSION>.tar.gz
tar xzf actions-runner-linux-x64.tar.gz

./config.sh --url https://github.com/<OWNER>/<REPO> \
            --token <REGISTRATION_TOKEN> \
            --name vm-runner-01 \
            --labels prod-vpc,docker \
            --unattended

sudo ./svc.sh install gha      # install as a systemd service running as user "gha"
sudo ./svc.sh start
sudo ./svc.sh status
```

**Sample output:**

```text
√ Connected to GitHub
# Runner Registration
Enter the name of the runner group to add this runner to: [press Enter for Default]
√ Runner successfully added
√ Runner connection is good
# Runner settings
√ Settings Saved.

● actions.runner.<owner>-<repo>.vm-runner-01.service - GitHub Actions Runner
     Active: active (running)
     … Listening for Jobs
```

(Use the exact version, URL and token GitHub shows you. Registration tokens expire after about 1 hour.)

### 2. Target it from a workflow

```yaml
jobs:
  deploy-internal:
    runs-on: [self-hosted, linux, prod-vpc]   # ALL labels must match
    steps:
      - uses: actions/checkout@v7
      - run: curl -fsS http://internal-api.corp.local/health   # reachable only from inside the VPC
```

Or target a runner **group** (org runners):

```yaml
runs-on:
  group: prod-runners
  labels: [linux, x64]
```

### 3. Ephemeral runner

```bash
./config.sh --url https://github.com/<OWNER>/<REPO> --token <TOKEN> --ephemeral --unattended
./run.sh        # takes ONE job, then the runner deregisters and the process exits
```

Pair it with automation (an autoscaling group, a script, or ARC) that creates a fresh machine for every job.

### 4. ARC: runners as Kubernetes pods (recap your Kubernetes notes)

**Kubernetes recap:** a **pod** is the smallest runnable unit (one or more containers). An **operator/controller** watches custom resources and creates pods to match. **Helm** installs packaged Kubernetes apps ("charts").

```bash
# 1. Install the controller (once per cluster)
helm install arc \
  --namespace arc-systems --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

# 2. Create a runner scale set for your org or repo (authenticate with a GitHub App or PAT)
helm install arc-runner-set \
  --namespace arc-runners --create-namespace \
  --set githubConfigUrl="https://github.com/<ORG>" \
  --set githubConfigSecret.github_token="<PAT>" \
  --set minRunners=0 --set maxRunners=10 \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set

kubectl get pods -n arc-runners -w
```

```yaml
jobs:
  build:
    runs-on: arc-runner-set          # the Helm release name becomes the runs-on label
    steps:
      - run: echo "running in a pod"
```

**Sample `kubectl` output while a job runs:**

```text
NAME                                  READY   STATUS    AGE
arc-runner-set-abc12-runner-x7k2p     1/1     Running   8s     ← created for this job
arc-runner-set-abc12-runner-x7k2p     0/1     Completed 1m12s  ← deleted afterwards
```

ARC scales from `minRunners` (can be 0, costing nothing when idle) to `maxRunners`, and every runner is **ephemeral**. For production use a **GitHub App** (not a PAT) for authentication, and plan how jobs get Docker (`containerMode: dind` or `kubernetes` mode).

### 5. Operations checklist

| Area | Do this |
|---|---|
| Updates | Keep the runner app current (it self-updates, so don't block it). Patch the OS image regularly. |
| Disk | Clean `_work`, Docker images and caches (`docker system prune` in a cron job), or use ephemeral runners. |
| Monitoring | Alert on runners **Offline** and on queue time (jobs waiting for a runner). |
| Secrets on the host | None. Use OIDC/instance roles with least privilege. |
| Node runtime | Runner v2.327.1+ for `node24` actions (topic 33). ARM32 is unsupported for Node 24. |

---

## Hands-on lab

1. On a spare Linux VM (or WSL), register a **repo-level** runner with label `lab` and run it with `./run.sh`.
2. Run a workflow with `runs-on: [self-hosted, lab]` and watch the job appear in the runner's terminal (`Running job: …`).
3. Stop the runner (Ctrl+C) and trigger again. The job sits **Queued** ("Waiting for a runner to pick up this job…"). That's the classic "self-hosted runner offline" incident.
4. Re-register with `--ephemeral`, run one job, and watch it deregister.
5. (Optional, with a kind/minikube cluster) Install ARC with `minRunners=0`, run a job, and watch a pod appear and disappear.
6. Remove the runner properly: `./config.sh remove --token <REMOVAL_TOKEN>`.

---

## Where you'll use it

- **Private network deploys:** a runner inside the VPC reaching internal databases or Kubernetes APIs (an alternative to the capstone's public EKS endpoint).
- **GPU / ML pipelines** and licensed tools.
- **Cost control at scale:** ARC on spot nodes.
- **Compliance:** builds that must stay on company hardware.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Job stuck **"Waiting for a runner to pick up this job…"** | No **online, idle** runner has **all** the labels, the runner group doesn't allow this repo or workflow, or all runners are busy. | Check **Settings → Actions → Runners** (Idle/Offline/Active). Fix labels or group access. Add capacity. |
| Runner shows **Offline** | Service stopped, host down, network or proxy blocks outbound 443, or credentials removed. | `sudo ./svc.sh status`, then `journalctl -u actions.runner.*`. Check `_diag/Runner_*.log`. |
| `Runner version vX is deprecated and cannot receive messages` | The runner app is too old (auto-update was blocked). | Update the runner. Allow self-update. |
| `docker: permission denied while trying to connect to the Docker daemon socket` | The runner user isn't in the `docker` group. | `sudo usermod -aG docker gha` and restart the service. |
| A job sees files from an earlier job | Persistent runner, dirty workspace. | Ephemeral runners, or clean-up steps. |
| `No space left on device` | Old images and workspaces piling up. | Scheduled prune, or ephemeral runners. |
| Node 24 actions fail on an old/ARM32 host | Unsupported runtime. | Upgrade the runner. Move off ARM32. |

---

## Quick recap

- Install: download the runner → `config.sh --url --token --labels` → `svc.sh install/start`. Outbound HTTPS only.
- `runs-on: [self-hosted, ...labels]` must match **all** labels. **Runner groups** control which repos may use them.
- **Ephemeral** (`--ephemeral`, or ARC) = one job per fresh runner, the secure default.
- **ARC** = Kubernetes controller + runner scale sets, scaling 0 → N, `runs-on: <scale-set-name>`.
- **Never on public repos.** Least privilege on the host, keep the runner updated, monitor Offline status and queue time.

**Interview questions**

1. **Q: A job is stuck in "Queued" on self-hosted runners. What do you check?**
   A: That runners are online and idle, that labels match all `runs-on` entries, that the runner group allows the repo and workflow, and the runner logs and network connectivity.
2. **Q: Why prefer ephemeral runners?**
   A: Each job gets a clean machine, so there's no leftover state, malware or credentials from earlier jobs.
3. **Q: What is ARC?**
   A: Actions Runner Controller: a Kubernetes operator that runs GitHub runners as ephemeral pods and autoscales them based on queued jobs.
