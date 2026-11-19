# 47 · Capstone 05 · Code Analysis and Quality Gates with SonarQube (Stage 7)

> **Part H: Capstone** · Previous: [46 · AWS Infrastructure Labs: EKS, Networking, RDS, SonarQube](46-capstone-04-aws-infrastructure-labs.md) · Next: [48 · Containerize and Scan the Application (Stage 8)](48-capstone-06-containerize-and-scan.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

**Stage 7** adds **SonarQube analysis** to both CI jobs: Maven's Sonar plugin for the backend, and `SonarSource/sonarqube-scan-action@v8.3.0` for the frontend. It imports test **coverage** reports and waits for the **Quality Gate** result, so a failing gate **fails CI**. It then deliberately changes thresholds to show the effect.

## Why it exists

Tests prove behaviour. SonarQube measures **code quality and security** (bugs, vulnerabilities, code smells, coverage, duplication) and enforces team standards automatically. "Quality gate failed" stops low-quality code before it's containerized.

## Analogy

A **building inspector**: the builders (tests) say the house works. The inspector checks it against the **building code** (Quality Gate: coverage ≥ X%, no critical issues). If the inspection fails, no occupancy certificate (no image build).

## How it works inside

```text
backend-job:  mvn verify (JaCoCo XML) → mvn sonar:sonar -Dsonar.qualitygate.wait=true …
frontend-job: vitest --coverage (lcov) → sonarqube-scan-action (sonar.qualitygate.wait=true)
   │ uses SONAR_HOST_URL (vars) + SONAR_TOKEN (secrets)
   ▼
SonarQube server analyses, applies the project's Quality Gate
   pass → step ✅ → later image jobs (needs) can run
   fail → step ❌ → job ❌ → dependent image/push jobs skipped
```

- **Maven vs Node.js:** the backend uses the Maven scanner plugin (it knows the classpath and compiled classes). The frontend uses the generic scanner action.
- SonarQube **doesn't run tests**. It **imports** coverage that JaCoCo/Vitest produced in the same job.
- The token is a **secret**. The server URL can be a **variable** (topic 17).

## Workflow YAML / commands: build it step by step

### Stage 7: SonarQube analysis with a blocking Quality Gate

Each stack produces coverage with its **own** tools, and SonarQube is the shared place that judges it:

| | Backend (Java) | Frontend (React) |
|---|---|---|
| Tests and coverage (Stage 6) | Maven + JaCoCo → `target/site/jacoco/jacoco.xml` | Vitest (V8) → `coverage/lcov.info` |
| Scanner | **SonarScanner for Maven** (`sonar-maven-plugin`, SonarSource's recommended way for Maven) | **`SonarSource/sonarqube-scan-action`** (downloads and runs the SonarScanner CLI) |
| Runs in | the **same job**, right after the build, so the report is already on disk | same |

```text
runner: scanner analyzes code + imports the coverage report ──upload──► SonarQube server (EC2)
                                                                       stores results in RDS
                                                                       evaluates the Quality Gate
runner: -Dsonar.qualitygate.wait=true polls ◄── PASSED / FAILED ───────┘
         FAILED → scanner exits non-zero → step ✗ → job ✗ → workflow ✗
```

Without `qualitygate.wait=true`, the step would succeed as soon as the upload finished, and the gate would be just a report nobody reads. With it, quality becomes a **real CI control**.

#### Backend: add a step after "Inspect backend build outputs"

Rename the job's **display name** to `Backend Build, Test and Quality Analysis` (keep the job ID and `needs`).

```yaml
      - name: Analyze backend code with SonarQube
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
        run: |
          mvn --batch-mode org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
            -Dsonar.projectKey=cwvj-mega-backend \
            -Dsonar.qualitygate.wait=true
```

- The token is a **secret** and the URL is a **variable**, because the URL isn't sensitive (topic 17). The scanner reads the standard env names `SONAR_TOKEN` and `SONAR_HOST_URL` automatically, so the names must be exactly these.
- Maven is preinstalled on hosted runners ([runner-images](https://github.com/actions/runner-images)). It downloads the plugin into the cached `~/.m2`, so there's no separate scanner to install.
- `sonar.projectKey` must match the SonarQube project from topic 46.

#### Frontend: add a step after "Inspect frontend build and test outputs"

Rename the job's display name to `Frontend Build, Test and Quality Analysis`.

```yaml
      - name: Analyze frontend code with SonarQube
        uses: SonarSource/sonarqube-scan-action@v8.3.0
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
        with:
          projectBaseDir: cwvj-3tier-mega-project/frontend
          args: >
            -Dsonar.projectKey=cwvj-mega-frontend
            -Dsonar.sources=src
            -Dsonar.tests=src
            -Dsonar.test.inclusions=**/*.test.jsx
            -Dsonar.exclusions=**/*.test.jsx,src/test-setup.js
            -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
            -Dsonar.qualitygate.wait=true
```

| Argument | Why |
|---|---|
| `projectBaseDir` | All the relative paths below resolve from `frontend/`. |
| `sources=src`, `tests=src` | Source and tests live in the same folder… |
| `test.inclusions=**/*.test.jsx` | …so mark the `*.test.jsx` files as **tests**… |
| `exclusions=**/*.test.jsx,src/test-setup.js` | …and keep them (and the Vitest setup file) out of the **source** scope. They're reclassified, not ignored. |
| `javascript.lcov.reportPaths` | The Stage 6 LCOV report. SonarQube imports it and never runs the tests. |

```bash
git add . && git commit -m "ci: add sonarqube code coverage analysis" && git push origin feature/initial-ci
```

#### Watch the gate fail, then change the policy

**Run 1** (80% gates): Trivy ✅. Backend tests ✅ → Quality Gate ❌. Frontend tests ✅ → Quality Gate ❌. **Passing tests don't mean the pipeline passes.** The gate is an extra policy check on top of the tests.

**Lower the thresholds for the demo** (in SonarQube, edit the **Coverage · Overall Code** condition of each gate):

| Gate | Before | After |
|---|---|---|
| `cwvj-backend-qg` | fails below 80% | fails below **60%** |
| `cwvj-frontend-qg` | fails below 80% | fails below **40%** |

The policy lives in SonarQube, not Git, so there's nothing to commit. Use **Actions → CWVJ CI | Build, Scan and Publish → Run workflow → `feature/initial-ci`**.

**Run 2:** all three jobs ✅. **The code didn't change; the policy did.** That's the power of a gate, and the reason to guard its settings: never lower real thresholds just to turn a pipeline green.

---

## Hands-on lab

1. Add both analysis steps and confirm both analyses show up in the SonarQube UI with coverage numbers.
2. Change the gate thresholds, run again, and watch CI **pass/fail with the gate**.
3. Find the line `QUALITY GATE STATUS: PASSED` (or `FAILED`) in the logs.
4. Remove the coverage path setting and see coverage drop to 0%, which proves SonarQube only imports reports.

## Where you'll use it

- Enterprise CI quality gates (SonarQube/SonarCloud), and audit evidence of code-quality checks.

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Not authorized` / 401 | Wrong or expired `SONAR_TOKEN`. | Regenerate the token and update the secret. |
| Timeout connecting to SonarQube | Network (port 9000, SG). | Topic 46 network notes. |
| Coverage 0% | Report path wrong, or tests ran without coverage. | Check `sonar.coverage.jacoco.xmlReportPaths` / `sonar.javascript.lcov.reportPaths` and that the files exist. |
| `QUALITY GATE STATUS: FAILED` | Real quality issues or thresholds. | Fix the code or tests. **Don't lower thresholds to make it pass** (the original's demo lowering was for illustration only). |

## Quick recap

- SonarQube analysis in both jobs, with coverage imported from JaCoCo/Vitest.
- `qualitygate.wait=true`, so the gate result decides the job result.
- URL as a variable, token as a secret. Don't weaken gates just to pass.

**Interview questions**

1. **Q: Does SonarQube run your tests?**
   A: No. It imports coverage reports generated by your test tools in the same job.
2. **Q: How does a Quality Gate stop a pipeline?**
   A: The scanner waits for the gate result and exits non-zero on failure, so the job fails and dependent jobs are skipped.
3. **Q: Why use the Maven plugin for Java instead of the generic scanner?**
   A: It understands the Maven project (modules, classpath, compiled classes) and configures the analysis automatically.
