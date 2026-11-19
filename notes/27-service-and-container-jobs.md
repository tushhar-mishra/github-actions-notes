# 27 · Service Containers and Container Jobs

> **Part D: Moving data around** · Previous: [26 · Matrix strategy](26-matrix-strategy.md) · Next: [28 · Environments](28-environments.md) · [Back to INDEX](INDEX.md)
>
> 🆕 New topic. Topic 07 mentioned that "steps can run directly on the VM or inside containers on that VM". This topic shows how.

---

## What it is

Two features that use Docker containers **around** your job:

1. **Service containers** (`services:`): extra containers that run **next to** your job for its whole lifetime, usually a **database, cache or message queue** your tests need (PostgreSQL, Redis, RabbitMQ).
2. **Container jobs** (`container:`): run **all the job's steps inside** a container image you choose, instead of directly on the runner VM.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container: python:3.13-slim      # steps run inside this image
    services:
      postgres:                      # a database for the tests
        image: postgres:17
```

Both need a **Linux** runner with Docker (GitHub-hosted Ubuntu runners have it).

---

## Why it exists

- **Integration tests need real dependencies.** Mocking a database hides bugs. A throwaway real PostgreSQL per job is easy and isolated.
- **Reproducible toolchains.** If your team builds inside `maven:3.9-eclipse-temurin-21`, running CI in that **same image** removes "works on my machine" differences from the runner's pre-installed tools.
- **Clean-up is automatic.** GitHub starts and destroys the containers for you.

**Jenkins comparison:** `agent { docker { image 'python:3.13' } }` ≈ `container:`. The `docker.image('postgres').withRun { … }` sidecar pattern ≈ `services:`.

---

## Analogy

Your job is a **cook** in a rented kitchen (the runner):

- **Service containers** = **appliances delivered for the shift**: a fridge (database), a microwave (Redis). They're plugged in before the cook starts and taken away after.
- **Container job** = the cook works **inside a portable kitchen pod** with exactly the knives and spices the recipe needs, rather than using whatever happens to be in the rented kitchen.

---

## How it works inside

```text
"Set up job"
  1. docker network create github_network_<id>         (a private network)
  2. for each service: docker pull + docker run on that network, with its "name" as hostname
  3. wait for each service's health check (if options: --health-cmd is set)
  4. if container: is set → start the job container on the same network,
     mounting the workspace (/__w) and tool folders
steps run
  • container job: each run: step is "docker exec" inside the job container
       → reach a service by its NAME:  postgres:5432
  • no container (steps on the VM):
       → reach a service via ports: mapping on localhost:  localhost:5432
"Stop containers" (automatic, at the end) → containers and network removed
```

The rule people forget:

| Steps run… | Hostname for the service | Need `ports:`? |
|---|---|---|
| **in a container job** | the service **label** (`postgres`) | No |
| **directly on the runner** | `localhost` | **Yes**, map it: `ports: ['5432:5432']` |

---

## Workflow YAML / commands

### A. Steps on the runner + a PostgreSQL service

```yaml
name: Integration Tests
on: push

jobs:
  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: test-only-password     # throwaway test DB, not a real secret
          POSTGRES_DB: appdb
        ports:
          - 5432:5432                                # runner:container
        options: >-
          --health-cmd "pg_isready -U app"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    steps:
      - name: Query the database (service is already healthy)
        run: |
          sudo apt-get install -y postgresql-client >/dev/null
          PGPASSWORD=test-only-password psql -h localhost -U app -d appdb -c "select version();"
```

| Line | Meaning |
|---|---|
| `services.postgres` | A service whose **label** (and hostname inside container jobs) is `postgres`. |
| `env:` | Configures the Postgres image (user, password, DB name). |
| `ports: - 5432:5432` | Publishes it on the runner's `localhost:5432` for steps on the VM. |
| `options: --health-cmd …` | Extra `docker run` flags. The health check makes GitHub **wait until Postgres accepts connections** before running the steps. |
| `>-` | YAML folded block without a trailing newline: joins the lines into one string (topic 03). |

**Sample log output:**

```text
Initialize containers
  /usr/bin/docker network create --label … github_network_8c1f…
  /usr/bin/docker pull postgres:17
  /usr/bin/docker create --name 3a1b…_postgres17 … --health-cmd "pg_isready -U app" … -p 5432:5432 postgres:17
  postgres service is starting, waiting 2 seconds before checking health...
  postgres service is healthy.
Run sudo apt-get install …
                                   version
 PostgreSQL 17.2 (Debian 17.2-1.pgdg120+1) on x86_64-pc-linux-gnu …
Stop containers
  Stop and remove container: 3a1b…_postgres17
  Remove container network: github_network_8c1f…
```

### B. Container job + Redis service (hostname = label)

```yaml
jobs:
  test-in-container:
    runs-on: ubuntu-latest
    container:
      image: python:3.13-slim
      env:
        REDIS_HOST: redis                 # the service label is the hostname
    services:
      redis:
        image: redis:7
        options: --health-cmd "redis-cli ping" --health-interval 5s --health-retries 5
    steps:
      - uses: actions/checkout@v7
      - run: pip install redis
      - run: python -c "import redis,os; r=redis.Redis(host=os.environ['REDIS_HOST']); r.set('k','v'); print(r.get('k'))"
```

**Sample log output:**

```text
Run python -c "import redis …"
b'v'
```

No `ports:` were needed, because the job container and Redis are on the same Docker network.

### C. Private images and credentials

```yaml
container:
  image: ghcr.io/my-org/build-image:1.4.0
  credentials:
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}   # needs permissions: packages: read (topic 18)
```

---

## Hands-on lab

1. Run example A and confirm the `select version();` output.
2. Remove `ports:` from A and watch `psql` fail with `Connection refused`. Then switch the job to `container: ubuntu:24.04` and use host `postgres`. It works without ports.
3. Remove the `--health-cmd` options and add an immediate connection step. Sometimes it fails because Postgres isn't ready yet. That's why health checks matter.
4. Add a matrix over `image: [postgres:16, postgres:17]` (`image: ${{ matrix.pg }}`) to test two database versions.

---

## Where you'll use it

- **Backend integration tests** against a real database (the capstone's Spring Boot backend uses PostgreSQL in production. A `services: postgres` job is the standard way to test it in CI).
- **Caches and queues:** Redis, RabbitMQ, Kafka in tests.
- **Pinned toolchains:** build inside an internal "golden" build image.
- **Database version matrix** (topic 26).

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Container operations are only supported on Linux runners` | `services:`/`container:` on Windows or macOS. | Use `ubuntu-*` runners. |
| `connection refused` to `localhost:5432` | No `ports:` mapping (steps on the VM), or the service isn't ready yet. | Add `ports:` and a health check. |
| `could not translate host name "postgres"` | Steps on the VM can't use the label as a hostname. | Use `localhost` (or run as a container job). |
| `Service container postgres failed` / unhealthy | Bad env (Postgres needs `POSTGRES_PASSWORD`), or the image crashed. | Check the "Initialize containers" log, or `docker logs` it in a debug step. |
| `Error response from daemon: pull access denied` | A private image without `credentials:`. | Add credentials and a token with read access. |
| `actions/checkout` or node actions fail inside an Alpine/minimal container | The image lacks Git or glibc for JavaScript actions. | Use a Debian-based image, or install `git`. |

---

## Quick recap

- `services:` = sidecar containers (DB, cache) for the job's lifetime, started healthy before the steps and removed after.
- `container:` = run all steps inside the chosen image.
- Steps on the VM → `localhost` + `ports:`. Container job → hostname = service label, no ports.
- Use health checks (`options: --health-cmd`). Linux runners only.

**Interview questions**

1. **Q: How do you give integration tests a real PostgreSQL in GitHub Actions?**
   A: A `services: postgres:` container with env, a health check, and either `ports: 5432:5432` (steps on the runner) or the hostname `postgres` (container job).
2. **Q: Why can't my step reach `postgres:5432`?**
   A: The steps run on the VM, not in a container, so use `localhost` with a port mapping.
3. **Q: When would you use `container:` for a job?**
   A: To run steps in a pinned toolchain image for reproducibility, independent of the runner's pre-installed software.
