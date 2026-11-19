# 48 · Capstone 06 · Containerize and Scan the Application (Stage 8)

> **Part H: Capstone** · Previous: [47 · Code Analysis and Quality Gates with SonarQube (Stage 7)](47-capstone-05-sonarqube-analysis.md) · Next: [49 · Publish and Promote Images to Amazon ECR with OIDC (Stage 9)](49-capstone-07-publish-images-to-ecr.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

**Stage 8** writes production **Dockerfiles** (a non-root Java 21 runtime image for the backend, a non-root Nginx Alpine image serving the Vite `dist/` for the frontend), passes the **exact validated build output** (JAR / `dist/`) to new **image-build-and-scan jobs** via **artifacts**, builds the images, scans them with **Trivy image** scans, fixes the vulnerabilities found, and exports each scanned image as a `.tar` artifact for the push jobs.

## Why it exists

A container image is what actually runs in production, so it needs its own scan (OS packages, packaged JARs), not just a source scan. Passing **the same artifact** between jobs means the image contains exactly what was tested, and **saving the scanned image** means exactly that image is pushed later.

## Analogy

After inspecting the parts (source scan, tests), you **seal them into a shipping container** (image), **X-ray the whole container** (image scan), and **put a tamper seal on it** (save the exact image as a `.tar` artifact) so the forklift (push job) moves exactly this one.

## How it works inside

```text
backend-job ── upload-artifact (JAR) ──► backend-image-build-scan-job (needs: backend-job)
                                           download-artifact → read version → tag → docker/build-push-action (load: true, no push)
                                           trivy image (HIGH,CRITICAL, exit-code 1)
                                           (Stage 9 adds: docker save → .tar → upload-artifact, outputs: image-tag)
frontend-job ── upload-artifact (dist/) ─► frontend-image-build-scan-job (same pattern, Nginx image)
```

| Dockerfile practice | Why |
|---|---|
| Non-root `USER` (UID 10001 / nginx user) | A container breakout gets fewer privileges |
| `COPY --chown=…` instead of a separate `RUN chown` | One fewer layer, correct ownership immediately |
| `EXPOSE` = documentation only | Ports are published by Docker `-p` or Kubernetes Services |
| `ENTRYPOINT ["java","-jar",…]` / Nginx in the foreground | The container's single main process |
| `apk upgrade` in the Nginx image | Patch OS packages flagged by the image scan |
| Nginx `pid /tmp/nginx.pid` + writable cache dirs | Needed when running as non-root |

Topics used: artifacts (24), job outputs (21), `needs` (06), the Docker recap (16), Trivy gates (44).

## Workflow YAML / commands: build it step by step

### Stage 8: Package the validated artifacts into images, then scan them

**The rule for this stage: build once, then package what was validated.** The JAR and `dist/` that passed tests and SonarQube are carried to new jobs as **artifacts**. Nothing is rebuilt, because a rebuild costs time and could produce something different from what was tested.

```text
backend-job:  build + test + Sonar → upload-artifact backend-jar
backend-image-build-scan-job (needs backend-job):
    download JAR → verify → read version (pom.xml) → tag = <version>-<sha7> → build (load) → verify → Trivy image scan
frontend-job: build + test + Sonar → upload-artifact frontend-dist
frontend-image-build-scan-job (needs frontend-job):
    download dist → verify → read version (package.json) → tag → build (load) → verify → Trivy image scan
```

**Why separate jobs?** Each job has one responsibility (application validation vs image validation), with a clear failure boundary. It also leaves room for different permissions later (the ECR push in Stage 9) without touching the build job.

#### Backend Dockerfile (`cwvj-3tier-mega-project/backend/Dockerfile`)

```dockerfile
# Use a Java 21 runtime image based on Ubuntu 24.04 LTS.
FROM eclipse-temurin:21-jre-noble

# Add metadata to identify and describe the container image.
LABEL org.opencontainers.image.title="CWVJ User Management API" \
      org.opencontainers.image.description="Spring Boot backend for the Cloud With VarJosh GitHub Actions Mega Project" \
      org.opencontainers.image.vendor="Cloud With VarJosh"

# Create a dedicated non-root system group and user for running the application.
RUN groupadd --system --gid 10001 appuser \
    && useradd --system \
       --uid 10001 \
       --gid 10001 \
       --no-create-home \
       --shell /usr/sbin/nologin \
       appuser

# Set /app as the working directory inside the container.
WORKDIR /app

# Copy the backend JAR into the image and assign ownership to UID 10001 and GID 10001.
COPY --chown=10001:10001 user-management-api.jar app.jar

# Run the application as the non-root user and group.
USER 10001:10001

# Document the port on which the backend application listens.
EXPOSE 8080

# Start the Spring Boot application when the container launches.
ENTRYPOINT ["java", "-jar", "app.jar"]
```

| Choice | Why |
|---|---|
| `21-jre-noble` (a **JRE**, not a JDK) | The JAR is already compiled, so no `javac` is needed at runtime. That means a smaller image with fewer tools for an attacker. |
| OCI `LABEL`s | Standard metadata that tools understand. It doesn't change runtime behaviour. |
| A system user with fixed UID/GID 10001, no home, `nologin` shell | **Least privilege**: a compromised app isn't root. |
| `WORKDIR /app` | Creates `/app` if it's missing, and relative paths resolve there. |
| `COPY --chown=10001:10001` | Correct ownership in the same layer, so no extra `RUN chown`. Numeric IDs don't need name lookup. |
| `USER 10001:10001` | Selects an existing user. It doesn't create one. |
| `EXPOSE 8080` | **Documentation only.** Ports are published by `docker -p` or a Kubernetes Service. |
| Exec-form `ENTRYPOINT` | Java becomes **PID 1** and receives `SIGTERM` directly, so it shuts down gracefully. Runtime args are appended, and a different program needs `--entrypoint`. |

> **Beyond this project:** enterprises often use **hardened internal base images** or **distroless** images (no shell, no package manager). They add multi-stage builds, base images pinned by **digest**, a read-only root filesystem, dropped capabilities, regular rescans, SBOMs and signing (topic 40).

#### Backend: upload the JAR (add after the SonarQube step in `backend-job`)

```yaml
      - name: Upload backend JAR
        uses: actions/upload-artifact@v7
        with:
          name: backend-jar
          path: cwvj-3tier-mega-project/backend/target/user-management-api.jar
          if-no-files-found: error
```

The artifact's **name** (`backend-jar`) is its handle for the download. It doesn't rename the file. `if-no-files-found: error` fails loudly instead of silently uploading nothing. Because it's placed **after** SonarQube, only a fully validated JAR is shared (topic 24).

#### Backend: the image job (add after `backend-job`)

```yaml
  backend-image-build-scan-job:
    name: Build and Scan Backend Container Image
    runs-on: ubuntu-26.04
    needs: backend-job

    steps:
      # Checkout the repository so the Dockerfile and pom.xml are available.
      - name: Checkout source code
        uses: actions/checkout@v7

      # Download the exact JAR produced and validated by the backend job.
      - name: Download backend JAR
        uses: actions/download-artifact@v8
        with:
          name: backend-jar
          path: cwvj-3tier-mega-project/backend

      # Verify that the downloaded JAR is available.
      - name: Verify backend JAR
        working-directory: cwvj-3tier-mega-project/backend
        run: ls -lh user-management-api.jar

      # Set up Java to run the Maven Help Plugin.
      - name: Set up Java
        uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: '21'

      # Read the application version from pom.xml.
      - name: Read application version
        id: app-version
        working-directory: cwvj-3tier-mega-project/backend
        run: |
          VERSION=$(mvn --batch-mode help:evaluate \
            -Dexpression=project.version \
            -q \
            -DforceStdout)

          echo "version=$VERSION" >> "$GITHUB_OUTPUT"

      # Generate an image tag using the application version and short commit SHA.
      - name: Generate image tag
        id: image-tag
        env:
          APP_VERSION: ${{ steps.app-version.outputs.version }}
          GITHUB_SHA: ${{ github.sha }}
        run: |
          SHORT_SHA="${GITHUB_SHA::7}"
          IMAGE_TAG="${APP_VERSION}-${SHORT_SHA}"

          echo "image_tag=$IMAGE_TAG" >> "$GITHUB_OUTPUT"

      # Build the image using the exact JAR validated by the backend job.
      - name: Build backend container image
        uses: docker/build-push-action@v7
        with:
          context: cwvj-3tier-mega-project/backend
          file: cwvj-3tier-mega-project/backend/Dockerfile
          tags: cwvj-mega-backend:${{ steps.image-tag.outputs.image_tag }}
          load: true

      # Display the locally built image for verification.
      - name: Verify backend container image
        run: docker images cwvj-mega-backend

      # Scan the container image for HIGH and CRITICAL vulnerabilities.
      - name: Scan backend container image with Trivy
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: cwvj-mega-backend:${{ steps.image-tag.outputs.image_tag }}
          severity: HIGH,CRITICAL
          exit-code: '1'
```

| Step | Point |
|---|---|
| Download to `backend/` | A fresh runner has no JAR. The artifact puts it exactly where the Dockerfile's `COPY` expects it. |
| `ls -lh` check | A missing file fails **here**, with a clear message, not deep inside `docker build`. |
| `setup-java` | Maven needs a JVM. Java 25 is the default on Ubuntu 26.04, so this selects the preinstalled 21 from the tool cache. **Nothing is rebuilt.** |
| `mvn help:evaluate -Dexpression=project.version -q -DforceStdout` | Prints just the version (`0.0.1-SNAPSHOT`) → a step output (topic 21). |
| Tag `${APP_VERSION}-${GITHUB_SHA::7}` | For example `0.0.1-SNAPSHOT-a1b2c3d`: it ties the image to the release **and** the commit. Never `latest`. |
| `load: true` | Load the image into the runner's local Docker so it can be inspected and scanned. **No push yet.** |
| Trivy `image-ref`, `HIGH,CRITICAL`, `exit-code: '1'` | The **gate**: an image that was merely built doesn't move forward until it passes. |

```bash
git add . && git commit -m "ci(backend): build and scan container image" && git push origin feature/initial-ci
```

**Result: the image scan fails ❌, which is expected.**

```text
Total: 8 (HIGH: 5, CRITICAL: 3)

Target                                                    Type   Vulnerabilities
cwvj-mega-backend:0.0.1-SNAPSHOT-8bc7f8f (ubuntu 24.04)  ubuntu        0
app/app.jar                                               jar           8
```

The OS layer is clean, but the **JAR** carries vulnerable dependencies. `COPY` just places the JAR; it fixes nothing.

**Why did the Stage 5 filesystem scan say 0 for `pom.xml`?** Trivy analyzes a `pom.xml` (it tries to resolve the dependency graph, with [documented limits](https://trivy.dev/docs/latest/guide/coverage/language/java/) on scopes, optional dependencies and versions it can't determine) differently from a packaged JAR (it inspects what's really inside). Here many versions come from the **Spring Boot parent POM** and from transitive dependencies, so they aren't written in the child POM. The logs don't prove which limit caused this case. The lesson is: **scan the artifact you'll ship**, and treat the filesystem scan as an early, complementary check, not proof.

*Optional:* reproduce the finding before packaging by adding this right after the Maven build (then remove it, because it fails the job):

```yaml
      - name: Run Trivy backend JAR scan
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          scan-type: rootfs
          scan-ref: cwvj-3tier-mega-project/backend/target
          scanners: vuln
          severity: HIGH,CRITICAL
          exit-code: '1'
```

**Fix the dependencies, not the gate.** The findings are in Jackson and Tomcat, which Spring Boot manages. Override the managed versions in `backend/pom.xml` (no new direct dependencies):

```xml
<properties>
    <java.version>21</java.version>
    <jackson-bom.version>3.2.3</jackson-bom.version>
    <tomcat.version>11.0.25</tomcat.version>
</properties>
```

```bash
git add cwvj-3tier-mega-project/backend/pom.xml
git commit -m "fix(backend): update vulnerable Jackson and Tomcat versions"
git push origin feature/initial-ci
```

Maven resolves the new versions, so you get a new validated JAR, a new image and a passing scan ✅. The gate (`HIGH,CRITICAL`, `exit-code: '1'`) stays exactly as strict.

#### Frontend Dockerfile (`cwvj-3tier-mega-project/frontend/Dockerfile`)

```dockerfile
# Use the official Nginx image based on Alpine 3.24 (slim variant).
FROM nginx:1.31.6-alpine3.24-slim

# Upgrade installed Alpine packages to their latest available versions.
RUN apk upgrade --no-cache

# Remove the default Nginx server configuration.
RUN rm /etc/nginx/conf.d/default.conf

# Copy our application-specific Nginx configuration into the image.
COPY nginx.conf /etc/nginx/nginx.conf

# Copy the validated Vite production build into Nginx's static content directory.
COPY dist /usr/share/nginx/html

# Assign the required ownership to the Nginx cache directory and PID file.
RUN chown -R nginx:nginx /var/cache/nginx \
    && touch /tmp/nginx.pid \
    && chown nginx:nginx /tmp/nginx.pid

# Run Nginx as a non-root user.
USER nginx

# Document the port on which Nginx listens inside the container.
EXPOSE 8080

# Start Nginx in the foreground when the container launches.
CMD ["nginx", "-g", "daemon off;"]
```

| Choice | Why |
|---|---|
| An Nginx base, no Node.js | The app is already static files, so only a web server is needed at runtime. |
| `apk upgrade --no-cache` | Pulls in the patched OS packages the image scan flagged. It upgrades every package and can't fix CVEs that have no patched version. |
| Remove `default.conf` and replace `nginx.conf` | The custom config listens on **8080**, serves `/usr/share/nginx/html`, **proxies `/api/` to the `cwvj-backend` Service on 8080** (same origin, so no CORS), uses `try_files … /index.html` for React routes, and sets `pid /tmp/nginx.pid`. |
| `COPY dist` | The **validated** Vite build. It's read-only for Nginx, so no ownership change is needed. |
| `chown` on the cache dir plus `/tmp/nginx.pid` | Running as non-root needs writable temp and PID paths. Adding `USER` alone isn't enough. |
| `USER nginx`, port 8080 | The base image already has an `nginx` user. Ports below 1024 need root, hence 8080. |
| `CMD ["nginx","-g","daemon off;"]` | Keeps Nginx in the **foreground** as the main process. The base image's `ENTRYPOINT` (`/docker-entrypoint.sh`) runs its hooks and then this `CMD`. |

#### Frontend: upload `dist/` (add after the SonarQube step in `frontend-job`)

```yaml
      - name: Upload frontend build
        uses: actions/upload-artifact@v7
        with:
          name: frontend-dist
          path: cwvj-3tier-mega-project/frontend/dist
          if-no-files-found: error
```

#### Frontend: the image job (add after `frontend-job`)

```yaml
  frontend-image-build-scan-job:
    name: Build and Scan Frontend Container Image
    runs-on: ubuntu-26.04
    needs: frontend-job

    steps:
      # Checkout the repository so the Dockerfile and package.json are available.
      - name: Checkout source code
        uses: actions/checkout@v7

      # Download the exact frontend build produced and validated by the frontend job.
      - name: Download frontend build
        uses: actions/download-artifact@v8
        with:
          name: frontend-dist
          path: cwvj-3tier-mega-project/frontend/dist

      # Verify that the frontend build artifact is available.
      - name: Verify frontend build
        working-directory: cwvj-3tier-mega-project/frontend
        run: ls -lah dist

      # Read the application version from package.json.
      - name: Read application version
        id: app-version
        working-directory: cwvj-3tier-mega-project/frontend
        run: |
          VERSION=$(node -p "require('./package.json').version")

          echo "version=$VERSION" >> "$GITHUB_OUTPUT"

      # Generate a readable image tag using the application version and short commit SHA.
      - name: Generate image tag
        id: image-tag
        env:
          APP_VERSION: ${{ steps.app-version.outputs.version }}
          GITHUB_SHA: ${{ github.sha }}
        run: |
          SHORT_SHA="${GITHUB_SHA::7}"
          IMAGE_TAG="${APP_VERSION}-${SHORT_SHA}"

          echo "image_tag=$IMAGE_TAG" >> "$GITHUB_OUTPUT"

      # Build the frontend container image using the validated frontend build artifact.
      - name: Build frontend container image
        uses: docker/build-push-action@v7
        with:
          context: cwvj-3tier-mega-project/frontend
          file: cwvj-3tier-mega-project/frontend/Dockerfile
          tags: cwvj-mega-frontend:${{ steps.image-tag.outputs.image_tag }}
          load: true

      # Display the locally built container image for verification.
      - name: Verify frontend container image
        run: docker images cwvj-mega-frontend

      # Scan the container image for HIGH and CRITICAL vulnerabilities.
      - name: Scan frontend container image with Trivy
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: cwvj-mega-frontend:${{ steps.image-tag.outputs.image_tag }}
          severity: HIGH,CRITICAL
          exit-code: '1'
```

It mirrors the backend job, with one difference: the version comes from `node -p "require('./package.json').version"` (`0.1.0`), since Node is preinstalled on the runner. That gives tags like `0.1.0-a1b2c3d`.

```bash
git add . && git commit -m "ci: build and scan frontend container image" && git push origin feature/initial-ci
```

**Expected:** the frontend image passes ✅, because `apk upgrade` already removed the OS findings. That's an expectation, not a guarantee: the gate stays strict, and a new CVE tomorrow fails the job, as it should.

**Two scans, two jobs:** the **filesystem** scan catches problems early in source and manifests. The **image** scan checks what you'll actually ship (OS packages plus packaged dependencies). Keep both. Next, Stage 9 publishes these **exact scanned images** to ECR without rebuilding them.

---

## Hands-on lab

1. Backend: upload the JAR, add the image job, and watch the **image** scan find vulnerabilities the FS scan missed (see "Why did the Stage 5 filesystem scan say 0"), fix them at the **dependency** level, and re-run green.
2. Frontend: upload `dist/`, add the image job, then watch the Nginx image scan pass.
3. After Stage 9, download the `.tar` artifact and `docker load -i` it locally, then `docker run` it.
4. Verify that `docker inspect` shows a non-root `User`.

## Where you'll use it

- Container hardening reviews (non-root, minimal base, patched OS packages).
- "Scan the image you ship" in every regulated pipeline.

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Image scan fails, FS scan passed | The image contains OS packages or bundled JARs the FS scan didn't see. | Upgrade the base image/packages or dependencies. Keep the gate strict. |
| `COPY failed: file not found` | The artifact wasn't downloaded into the build context. | Check the `download-artifact` `path`. |
| Nginx container crashes as non-root | PID or cache paths not writable. | `pid /tmp/nginx.pid`, chown the cache dirs (as in the original). |
| Push job can't find the image | Image not saved/uploaded, or a wrong artifact name. | `docker save` + `if-no-files-found: error`. Match the names. |
| `No space left on device` | Big images on a 14 GB runner. | Prune between steps. Smaller base images. |

## Quick recap

- Pass the validated JAR/`dist/` as artifacts, build the images in separate jobs, Trivy **image** scan as a gate.
- Hardened Dockerfiles: non-root, `--chown`, upgraded packages, `EXPOSE` as documentation only.
- Save the **exact scanned image** as a `.tar` artifact so the push job publishes precisely it.

**Interview questions**

1. **Q: Why scan both the repo and the image?**
   A: They see different things. The image includes OS packages and the packaged artifacts. The repo scan sees source, manifests and secrets.
2. **Q: Why move the image to the push job as a `.tar` artifact instead of rebuilding?**
   A: Rebuilding could produce a different image than the one that was scanned. Moving the saved image guarantees the scanned bits are the ones published.
3. **Q: Name three Dockerfile hardening practices used here.**
   A: A non-root user, ownership set at copy time, and patched/minimal base packages (plus a single foreground process).
