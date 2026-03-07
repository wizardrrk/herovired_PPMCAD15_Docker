# Class Discussion - Session 3
## Dockerfile ARG & ENV Instructions + Image Tagging Best Practices

---

## The `ARG` Instruction

### What is ARG?

`ARG` defines **build-time variables** that users can pass when building a Docker image. These variables are only available **during the build process** and do NOT persist in the running container.

```
ARG Lifecycle:
┌──────────────────────────────────────────────────┐
│  docker build  →  Available  →  Container runs   │
│                   (build time)   → GONE ❌       │
└──────────────────────────────────────────────────┘
```

### Syntax

```dockerfile
ARG <name>
ARG <name>=<default_value>
```

### Passing ARG Values During Build

```bash
docker build --build-arg <ARG_NAME>=<value> .
```

### Example: Using ARG to Set a Dynamic Base Image Version

```dockerfile
ARG NODE_VERSION=18

FROM node:${NODE_VERSION}

WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "app.js"]
```

```bash
# Uses default (node:18)
docker build -t myapp:latest .

# Override to use node 20
docker build --build-arg NODE_VERSION=20 -t myapp:latest .
```

> **Note:** ARGs used **before** `FROM` are only available in the `FROM` instruction itself. To use them after `FROM`, you must redeclare them.

```dockerfile
ARG APP_VERSION=1.0

FROM node:18

# Must redeclare after FROM to use it here
ARG APP_VERSION
RUN echo "Building version: $APP_VERSION"
```

### Example: Using ARG for Conditional Dependencies

```dockerfile
FROM python:3.11-slim

ARG ENVIRONMENT=production

WORKDIR /app
COPY requirements.txt .

# Install dev dependencies only if ENVIRONMENT is "development"
RUN if [ "$ENVIRONMENT" = "development" ]; then \
      pip install -r requirements-dev.txt; \
    else \
      pip install -r requirements.txt; \
    fi

COPY . .
CMD ["python", "app.py"]
```

```bash
# Production build (default)
docker build -t myapp:prod .

# Development build with extra dev tools
docker build --build-arg ENVIRONMENT=development -t myapp:dev .
```

---

## The `ENV` Instruction

### What is ENV?

`ENV` sets **environment variables** that persist both during the build **and** inside the running container. These are available to the application at runtime.

```
ENV Lifecycle:
┌──────────────────────────────────────────────────┐
│  docker build  →  Available  →  Container runs   │
│                   (build time)   → Still there ✅ │
└──────────────────────────────────────────────────┘
```

### Syntax

```dockerfile
ENV <key>=<value>
ENV <key1>=<value1> <key2>=<value2>
```

### Example 1: Setting Application Configuration

```dockerfile
FROM node:18

ENV NODE_ENV=production
ENV PORT=3000

WORKDIR /app
COPY . .
RUN npm install

# The app can read process.env.PORT and process.env.NODE_ENV
CMD ["node", "server.js"]
```

### Example 2: Setting Database Connection Details

```dockerfile
FROM python:3.11-slim

ENV DB_HOST=localhost
ENV DB_PORT=5432
ENV DB_NAME=myappdb
ENV DB_USER=admin

WORKDIR /app
COPY . .
RUN pip install -r requirements.txt

CMD ["python", "app.py"]
```

### Example 3: Overriding ENV at Runtime

Even though `ENV` is baked into the image, you can **override** it when running the container:

```bash
# Override DB_HOST and DB_NAME at runtime
docker run -e DB_HOST=prod-db.example.com -e DB_NAME=proddb myapp:latest
```

### Example 4: Using ENV with Java Applications

```dockerfile
FROM eclipse-temurin:17-jre

ENV JAVA_OPTS="-Xms256m -Xmx512m"
ENV APP_PROFILE=production

WORKDIR /app
COPY target/myapp.jar .

CMD java $JAVA_OPTS -Dspring.profiles.active=$APP_PROFILE -jar myapp.jar
```

---

## ARG vs ENV — Quick Comparison

```
┌────────────┬───────────────────────┬───────────────────────┐
│            │        ARG            │        ENV            │
├────────────┼───────────────────────┼───────────────────────┤
│ Available  │ Build time only       │ Build time + Runtime  │
│ Set via    │ --build-arg           │ Dockerfile / -e flag  │
│ Persists   │ No                    │ Yes (in image layers) │
│ Use case   │ Build configuration   │ App configuration     │
└────────────┴───────────────────────┴───────────────────────┘
```

### Using ARG and ENV Together

A common pattern is to pass a value at build time using `ARG` and then persist it using `ENV`:

```dockerfile
FROM node:18

ARG APP_VERSION
ENV APP_VERSION=${APP_VERSION}

# Now APP_VERSION is available at both build AND run time
RUN echo "Building version: $APP_VERSION"
CMD ["node", "app.js"]
```

```bash
docker build --build-arg APP_VERSION=2.1.0 -t myapp:2.1.0 .
```

---

## Real-World Reference: Selenoid

**Selenoid** ([hub.docker.com/r/aerokube/selenoid](https://hub.docker.com/r/aerokube/selenoid)) is a Docker-based Selenium hub alternative that launches browser containers on demand. It is a good example of a project that uses Docker images with proper tagging and versioning.

```bash
# Pull a specific version
docker pull aerokube/selenoid:1.11.3

# Run with configuration
docker run -d --name selenoid \
  -p 4444:4444 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(pwd)/config:/etc/selenoid:ro \
  aerokube/selenoid:1.11.3
```

> Selenoid demonstrates good versioning — each release gets a **unique, immutable tag**. This is exactly the practice you should follow for your own images.

---

## Image Tagging Best Practices

### The Problem: Reusing Tags

Reusing the same tag for different builds is a **bad practice** that leads to confusion, broken rollbacks, and unreliable deployments.

### Scenario: What NOT To Do

```
Timeline of a Bad Tagging Practice:
────────────────────────────────────────────────────────────────────

Day 1: New analytics page feature is ready
       ↓
       Build image → Tag: v2.0.3
       ↓
       Push to registry → Deploy to production ✅

Day 3: Bug discovered in the analytics page
       ↓
       Dev team fixes the code
       ↓
       Build image → Reuse same tag: v2.0.3  ❌
       ↓
       Force push to registry (overwrites old v2.0.3)
       ↓
       Deploy to production

────────────────────────────────────────────────────────────────────
```

### Why Is This Bad?

**1. No Traceability**
- You can't tell which version of the code `v2.0.3` actually contains
- Was it the original release? Or the patched one?
- Debugging and auditing become nearly impossible

**2. Broken Rollbacks**
- If the fix introduces a new bug, you can't rollback to the *original* `v2.0.3`
- That image has been overwritten and is gone forever
- You've lost a known working state

**3. Cache and Pull Confusion**
- Other environments may still have the **old** `v2.0.3` cached locally
- Running `docker pull` may or may not give the updated image depending on cache
- Different servers could be running different code under the same tag

**4. Breaks Team Trust**
- Team members assume a tag represents a **specific, unchanging** build
- Silently changing what a tag points to violates this assumption

### The Correct Approach: Always Create a New Tag

```
Correct Tagging Practice:
────────────────────────────────────────────────────────────────────

Day 1: Feature ready
       ↓
       Build image → Tag: v2.0.3
       ↓
       Deploy to production ✅

Day 3: Bug found → Code fixed
       ↓
       Build image → Tag: v2.0.4  ✅ (NEW tag)
       ↓
       Push to registry → Deploy to production

Need to rollback? → Just redeploy v2.0.3 (it still exists!) ✅

────────────────────────────────────────────────────────────────────
```

### Golden Rule

> **Every `docker build` for an application should produce a new, unique tag.**
> 
> Tags should be treated as **immutable** — once pushed, never overwritten.

### Common Tagging Strategies

```
Strategy                │  Example
────────────────────────┼─────────────────────
Semantic Versioning     │  v2.0.3, v2.0.4
Git Commit SHA          │  myapp:a1b2c3d
Build Number (CI/CD)    │  myapp:build-1547
Date-based              │  myapp:2025-02-21
Combined                │  myapp:v2.0.4-a1b2c3d
```

---