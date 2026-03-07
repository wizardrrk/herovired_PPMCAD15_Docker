# Class Discussion - Session 6
## CI/CD Pipelines, Build Artifacts & Artifact Management

---

## The Problem: How Software Used to Be Delivered

Traditionally, getting code from a developer's machine to production was a **manual, error-prone process**:

```
Traditional Deployment Flow:
────────────────────────────────────────────────────────────────────

Developer writes code (e.g., Java + pom.xml)
    ↓
Runs build commands locally:
    mvn test
    mvn package        (compile + create executable)
    mvn deploy
    ↓
A JAR / WAR file is generated on the developer's machine
    ↓
File is shared manually:
    - NFS file share
    - Email the package
    - Send the APK via Teams / Slack
    ↓
Ops person receives the file
    ↓
Manually places it in the correct folder on the production server
    ↓
Manually restarts the services

────────────────────────────────────────────────────────────────────
```

### What's Wrong With This?

- **No consistency** — builds depend on the developer's local machine setup
- **No traceability** — which version of the code was built? Who built it?
- **Error-prone** — wrong file in the wrong folder, forgot to restart, wrong version deployed
- **No quality gates** — no automated tests, scans, or validations before deployment
- **Slow and manual** — every release requires human intervention at every step

---

## The Solution: CI/CD Pipelines

Instead of building and deploying manually, we automate the entire process using a **CI/CD pipeline**.

### The Core Idea

> Why don't we have a place where we can: pull the code base → run a standard set of commands to create a high-quality executable → push that to a central repository — and then have another pipeline that pulls the executable and deploys it to production automatically?

**That "place" is a CI/CD tool like Jenkins.**

---

## CI — Continuous Integration

CI is the automated process of **building, testing, and packaging** your code every time a change is pushed.

```
CI Pipeline Flow:
────────────────────────────────────────────────────────────────────

  Code Push (Git)
      ↓
  ┌─────────────────────────────────────────────────────────┐
  │                    CI PIPELINE                          │
  │                                                         │
  │  1. Code Pull        ← Pull latest code from repo      │
  │      ↓                                                  │
  │  2. Lint             ← Check code formatting & style    │
  │      ↓                                                  │
  │  3. Test             ← Unit tests + Integration tests   │
  │      ↓                                                  │
  │  4. Scan             ← Code quality + Code coverage     │
  │      ↓                                                  │
  │  5. Build Artifact   ← Create the deployable package    │
  │      ↓                                                  │
  │  6. Push to Registry ← Store in a central repository    │
  └─────────────────────────────────────────────────────────┘
```

### What Happens at Each Stage

```
Stage              │ What It Does                           │ Tools
───────────────────┼────────────────────────────────────────┼──────────────────────
Code Pull          │ Fetches latest code from Git           │ Git, GitHub, GitLab
Lint               │ Checks code style and formatting       │ ESLint, Pylint, Checkstyle
Test               │ Runs unit + integration tests          │ JUnit, pytest, Jest
Scan               │ Code quality, coverage, vulnerabilities│ SonarQube, Trivy, Snyk
Build Artifact     │ Creates the deployable package         │ Maven, npm, Docker
Push to Registry   │ Stores the artifact for deployment     │ Nexus, ECR, DockerHub
```

---

## Two Types of Artifacts

The final output of a CI pipeline is an **artifact** — the deployable package. There are broadly two types depending on whether your app is containerized or not.

### Type 1: Containerized Applications

If your app runs inside containers, the artifact is a **Docker image**.

```
Containerized Artifact:
────────────────────────────────────────────────────────────

Source Code → Lint → Test → Scan → Dockerfile → docker build → Docker Image
                                                                    ↓
                                                        Push to Docker Registry
                                                                    ↓
                                                ┌──────────────────────────────┐
                                                │  Docker Hub (public/private) │
                                                │  AWS ECR                     │
                                                │  Azure ACR                   │
                                                │  Google Artifact Registry    │
                                                │  GitHub Container Registry   │
                                                └──────────────────────────────┘
```

The image contains **everything** — code, dependencies, runtime, OS libraries. It is the single deployable unit.

### Type 2: Non-Containerized Applications

If your app is a traditional service (Java, Node.js, Python) that runs directly on a server without Docker, the artifact is an **executable package**.

```
Non-Containerized Artifact:
────────────────────────────────────────────────────────────

  Java App    →  mvn package   →  app-service-1.10.jar
  Node.js App →  npm build     →  dist.zip
  Python App  →  setup.py      →  app-1.0.0.tar.gz
  Go App      →  go build      →  binary executable
                                        ↓
                              Push to Artifact Repository
                                        ↓
                              ┌──────────────────────────────┐
                              │  JFrog Artifactory           │
                              │  Sonatype Nexus              │
                              │  AWS CodeArtifact            │
                              │  Azure Artifacts             │
                              │  GitHub Packages             │
                              └──────────────────────────────┘
```

These artifacts must be **versioned properly** (e.g., `app-service-1.10.jar`, `app-service-1.11.jar`) so you always know what version is deployed and can rollback if needed.

---

### Artifact Comparison

```
┌──────────────────────┬───────────────────────────┬───────────────────────────┐
│                      │ Containerized             │ Non-Containerized         │
├──────────────────────┼───────────────────────────┼───────────────────────────┤
│ Artifact type        │ Docker Image              │ JAR, WAR, ZIP, binary     │
│ Contains             │ Code + deps + OS + runtime│ Code + deps only          │
│ Runs on              │ Any Docker host           │ Specific server setup     │
│ Registry             │ DockerHub, ECR, ACR       │ Nexus, JFrog Artifactory  │
│ Versioning           │ Image tags (v1.0, v1.1)   │ File name (app-1.0.jar)   │
│ Portability          │ High (runs anywhere)      │ Lower (needs matching env)│
│ Environment included │ Yes                       │ No                        │
└──────────────────────┴───────────────────────────┴───────────────────────────┘
```

---

## The Complete CI/CD Picture

CI handles building and packaging. **CD (Continuous Deployment/Delivery)** handles getting that artifact to production.

```
End-to-End CI/CD Pipeline:
════════════════════════════════════════════════════════════════════

  CI (Continuous Integration)                CD (Continuous Deployment)
  ┌─────────────────────────────────┐       ┌─────────────────────────────┐
  │                                 │       │                             │
  │  Code Pull                      │       │  Pull artifact from         │
  │      ↓                          │       │  registry                   │
  │  Lint + Test + Scan             │       │      ↓                      │
  │      ↓                          │       │  Deploy to target env       │
  │  Build Artifact                 │       │  (place in correct folder / │
  │      ↓                          │       │   update container)         │
  │  Push to Registry               │       │      ↓                      │
  │                                 │  ──→  │  Restart services           │
  │  Artifact:                      │       │      ↓                      │
  │  • Docker Image → Docker Reg.   │       │  Health checks              │
  │  • JAR/WAR     → Nexus/JFrog    │       │      ↓                      │
  │                                 │       │  Live in Production         │
  └─────────────────────────────────┘       └─────────────────────────────┘

  Tool: Jenkins, GitHub Actions,             Tool: Jenkins, GitHub Actions,   
        GitLab CI, CircleCI                        GitLab CI, Spinnaker, ArgoCD

════════════════════════════════════════════════════════════════════
```

---
