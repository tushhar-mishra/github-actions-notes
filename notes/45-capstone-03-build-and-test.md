# 45 · Capstone 03 · Build and Validate the Backend and Frontend (Stage 6)

> **Part H: Capstone** · Previous: [44 · Trivy Filesystem Security Scanning (Stage 5)](44-capstone-02-trivy-filesystem-scan.md) · Next: [46 · AWS Infrastructure Labs: EKS, Networking, RDS, SonarQube](46-capstone-04-aws-infrastructure-labs.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

**Stage 6** adds two parallel jobs: **`backend-job`** (Java 21, Maven `clean verify` = compile + unit tests with JUnit 5/Mockito + JaCoCo coverage + JAR) and **`frontend-job`** (Node.js, `npm ci`, Vitest tests with coverage, `npm run build` → Vite production `dist/`), each with **dependency caching** through the setup actions.

## Why it exists

CI must prove the app **compiles, passes tests and produces its production build** before any quality analysis, containerization or publishing. Caching keeps those jobs fast.

## Analogy

Two kitchens cooking **at the same time**: the backend kitchen bakes the main dish (JAR) and the frontend kitchen prepares dessert (static files). Each keeps a **pantry** of ingredients from last time (Maven/npm caches), so they don't go shopping on every order.

## How it works inside

| Concept | In this stage | Topic |
|---|---|---|
| Parallel jobs | `backend-job` and `frontend-job` (and Trivy) run at the same time | 06 |
| `defaults.run.working-directory` | Each job runs in its own sub-folder of the monorepo | 14 |
| `actions/setup-java@v6` (`distribution: temurin`, `java-version: '21'`, `cache: maven`, `cache-dependency-path: …/pom.xml`) | Installs Java 21 (the 26.04 image defaults to Java 25) and caches `~/.m2/repository` keyed on that `pom.xml` | 15, 25 |
| `actions/setup-node@v6` (`cache: npm`, `cache-dependency-path: …/package-lock.json`) | Installs Node and caches `~/.npm` | 25 |
| `npm ci` | Installs **exactly** what `package-lock.json` says (reproducible). Fails if the lock file and package.json disagree | 04 |
| Maven lifecycle | `verify` runs validate → compile → test → package → verify, with plugins (Surefire, JaCoCo) doing the work | 01 |
| Cache hit ≠ skip | Maven/npm still run. They just don't download again | 25 |

## Workflow YAML / commands: build it step by step

### Stage 6: Backend and frontend build-and-test jobs

```text
push → Trivy Repository Scan → (Backend Build and Test ∥ Frontend Build and Test)
```

Both jobs say `needs: trivy-scan-job`, so a failed scan **skips** them. Once the scan passes they run in parallel, and one failing doesn't cancel the other.

#### Backend job (add after `trivy-scan-job`)

```yaml
  backend-job:
    name: Backend Build and Test
    runs-on: ubuntu-26.04
    needs: trivy-scan-job

    defaults:
      run:
        working-directory: cwvj-3tier-mega-project/backend

    steps:
      - name: Checkout source code
        uses: actions/checkout@v7

      - name: Set up Java
        uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: '21'
          cache: maven
          cache-dependency-path: cwvj-3tier-mega-project/backend/pom.xml

      - name: Verify Java version
        run: java --version

      - name: Build, test, and package backend
        run: mvn --batch-mode clean verify

      - name: Inspect backend build outputs
        run: |
          echo "=== Maven Build Outputs ==="
          ls -lah target/

          echo "=== Unit Test Reports ==="
          ls -lah target/surefire-reports/

          echo "=== JaCoCo Coverage Reports ==="
          ls -lah target/site/jacoco/
```

| Line | What it does in this project |
|---|---|
| `runs-on: ubuntu-26.04` | A pinned image, so a moving `-latest` label can't change your tools overnight (topic 07). |
| `defaults.run.working-directory` | A monorepo: every `run:` step starts in `backend/`. It affects **`run` only**, not `uses` steps (topic 14). |
| `setup-java` `temurin` / `'21'` | An explicit JDK, so the build doesn't depend on whatever the runner has preinstalled. |
| `cache: maven` + `cache-dependency-path` | Caches `~/.m2/repository` with a key from the hash of **this** `pom.xml`. It's restored at the start and saved at the end of the job (topic 25). |
| `mvn --batch-mode clean verify` | Non-interactive. Runs the lifecycle phases in order (compile → test → package → verify). A failed compile or test returns non-zero, so the job fails. |

**What `verify` runs here** (all configured in `pom.xml`):

| Plugin | Phase | Output |
|---|---|---|
| Compiler | compile | classes |
| Surefire (JUnit 5 + Mockito, `UserServiceTest` with a mocked `UserRepository`) | test | `target/surefire-reports/` |
| Spring Boot Maven plugin | package | the executable `target/*.jar` (used for the Docker image in Stage 8) |
| JaCoCo | verify | `target/site/jacoco/jacoco.xml` (used by SonarQube in Stage 7), plus `index.html` and `jacoco.csv` |

> There are no integration tests (no Failsafe plugin). Coverage comes **only** from what the unit tests run, and `verify` doesn't produce coverage on its own: the JaCoCo plugin goals in `pom.xml` do.

#### Frontend job

```yaml
  frontend-job:
    name: Frontend Build and Test
    runs-on: ubuntu-26.04
    needs: trivy-scan-job

    defaults:
      run:
        working-directory: cwvj-3tier-mega-project/frontend

    steps:
      - name: Checkout source code
        uses: actions/checkout@v7

      - name: Set up Node.js
        uses: actions/setup-node@v6
        with:
          node-version: '22'
          cache: npm
          cache-dependency-path: cwvj-3tier-mega-project/frontend/package-lock.json

      - name: Verify Node.js and npm versions
        run: |
          node --version
          npm --version

      - name: Install frontend dependencies
        run: npm ci

      - name: Build frontend application
        run: npm run build

      - name: Run frontend tests and generate coverage
        run: npm run test:coverage

      - name: Inspect frontend build and test outputs
        run: |
          echo "=== Frontend Build Outputs ==="
          ls -lah dist/

          echo "=== Frontend Coverage Reports ==="
          ls -lah coverage/
          ls -lah coverage/lcov.info
```

| Line | Why |
|---|---|
| Its own checkout | Jobs don't share files. Data moves only through artifacts, caches or outputs. |
| `cache: npm` + lockfile path | Caches **`~/.npm`** (not `node_modules`), with the key from the hash of the lockfile. `setup-node` only looks in the repo root by default, hence the explicit path. |
| `npm ci` | A clean, exact install from `package-lock.json`. It deletes `node_modules`, **fails** if the lockfile and `package.json` disagree, and never edits them. Commit both files. |
| `npm run build` | Vite bundles the app into `dist/` (copied into the Nginx image in Stage 8). |
| `npm run test:coverage` | Vitest runs the tests and writes `coverage/lcov.info`. SonarQube only **imports** this report; it never runs tests. |

#### Push and check

```bash
git add . && git commit -m "ci: add backend and frontend build jobs" && git push
```

- **First run:** `Set up Java` shows `maven cache is not found` and `Set up Node.js` shows `npm cache is not found`. Dependencies download, and the caches are saved at the end of the job.
- **Second run** (push a comment-only change to `ci.yaml`):

```text
Cache hit for: setup-java-Linux-x64-maven-<hash>
Cache restored successfully
Cache hit for: node-cache-Linux-x64-npm-<hash>
Cache restored successfully
```

The keys depend only on `pom.xml` and `package-lock.json`, so a workflow comment doesn't change them. A cache hit **doesn't skip** the build or the tests; it only skips downloads.

---

## Hands-on lab

1. Add both jobs and confirm both jobs pass, running **in parallel** (check the workflow graph).
2. Push a comment-only workflow change and find **`Cache hit for: setup-java-…-maven-…`** / **`Cache restored successfully`** (and the npm equivalent) in the logs. Compare durations with the first run.
3. Break a unit test and see `backend-job` fail with Surefire's test report in the log.

## Where you'll use it

- Every Java/Node monorepo CI.
- Choosing `npm ci` over `npm install` in pipelines. Explicit `cache-dependency-path` in monorepos.

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `npm ci` fails: `package.json and package-lock.json … are not in sync` | The lock file is outdated. | Run `npm install` locally and commit the lock file. |
| Cache never hits | Wrong `cache-dependency-path`, or the key changed. | Check the path. Print the setup action's cache logs. |
| `java: invalid target release: 21` / wrong Java | Default Java 25 used. | `setup-java` with `java-version: '21'`. |
| Tests pass locally, fail in CI | Time zone, locale or ordering assumptions. | Make tests deterministic. Print the env. |
| Coverage report missing for SonarQube later | Coverage not generated (JaCoCo/Vitest config). | Check the `target/site/jacoco/` and `coverage/` outputs (Stage 7 relies on them). |

## Quick recap

- Parallel backend (Maven `clean verify`) and frontend (`npm ci`, test with coverage, `npm run build`) jobs.
- `setup-java` / `setup-node` with built-in caching keyed on `pom.xml` / `package-lock.json`.
- `working-directory` defaults for the monorepo. A cache hit speeds things up but still runs the build.

**Interview questions**

1. **Q: Why `npm ci` instead of `npm install` in CI?**
   A: It installs exactly the locked versions, never rewrites the lock file, and fails on mismatch, which makes builds reproducible.
2. **Q: How does `setup-java`'s `cache: maven` work?**
   A: It restores and saves `~/.m2/repository` with a key from the hash of the `pom.xml` files (or `cache-dependency-path`).
3. **Q: What does `mvn verify` run?**
   A: The lifecycle up to `verify`: compile, unit tests (Surefire), package, then verification (for example integration tests and coverage checks).
