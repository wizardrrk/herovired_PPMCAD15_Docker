## Real-World Docker Issues
---

## Real-World Issues While Working With Docker

These are **actual problems** that developers and DevOps engineers run into daily when building, running, and managing Docker containers. The kind of things that break your builds, waste hours of debugging, and show up at 2 AM.

---

### 1. "It Works on My Machine, But Not in the Container"

The most classic Docker headache. Your app runs fine locally but fails inside the container.

**Common causes:**

```
Scenario: Node.js app works locally, crashes in container
─────────────────────────────────────────────────────────

  Cause 1: Different base image OS
    - You develop on Ubuntu, container uses Alpine
    - Native packages (bcrypt, sharp, etc.) compiled for wrong OS
    - Fix: Run npm rebuild inside the container, or use the same OS

  Cause 2: Missing system dependencies
    - Your local machine has libpng, ffmpeg, etc. installed
    - The container image doesn't have them
    - Fix: Add RUN apk add --no-cache <package> in Dockerfile

  Cause 3: Different Node/Python/Java version
    - Local: Node 20, Container: Node 18
    - Fix: Match versions explicitly in FROM directive
```

```bash
# Debugging: Get into the container and test manually
docker run -it --entrypoint /bin/sh myapp:latest
# Now you can run commands inside and see what's different
```

---

### 2. Docker Build Fails - Caching Issues

You change one line in your code and Docker rebuilds **everything** from scratch, taking 10+ minutes.

```
BAD Dockerfile (cache busts on every code change):
────────────────────────────────────────────────
  FROM node:18-alpine
  WORKDIR /app
  COPY . .                     ← Everything copied, cache busted
  RUN npm install              ← Reinstalls ALL dependencies every time
  CMD ["node", "server.js"]

GOOD Dockerfile (cache-friendly layer ordering):
────────────────────────────────────────────────
  FROM node:18-alpine
  WORKDIR /app
  COPY package.json package-lock.json ./   ← Dependencies file first
  RUN npm ci                                ← Cached if package.json unchanged
  COPY . .                                  ← Only source code changes
  CMD ["node", "server.js"]
```

**Another caching issue:**

# Issue Description:

Docker's build cache works by comparing the FROM instruction as a string. 
- If your Dockerfile says FROM python:3.12 and Docker already has a cached layer for that exact line, it reuses it without checking - if the upstream python:3.12 tag now points to a newer image. 
- So you update your base image (or the upstream maintainer publishes a new one), but your build keeps using the old cached version.

```bash
# You updated your base image but Docker uses the cached old version
docker build -t myapp .    # Still using stale cached layer

# Fix: Force fresh build
docker build --no-cache -t myapp .

# Or pull latest base image first
docker pull node:18-alpine
docker build -t myapp .
```

---

### 3. Container Runs But App Not Accessible (Port Issues)

You start the container, no errors in logs, but you can't reach the app from your browser.

```
Scenario: curl http://localhost:3000 → Connection refused
─────────────────────────────────────────────────────────

  Check 1: Did you publish the port?
    docker run myapp              ← Port NOT exposed to host
    docker run -p 3000:3000 myapp ← Port mapped correctly

  Check 2: Is your app binding to 0.0.0.0 or 127.0.0.1?
    App listening on 127.0.0.1:3000 → Only accessible inside container ❌
    App listening on 0.0.0.0:3000   → Accessible from outside ✅

  Check 3: Is the port mapping correct?
    docker run -p 8080:3000 myapp
    → Access via localhost:8080 (NOT 3000)
    → Left side = host port, Right side = container port
```

```bash
# Debug: Check if app is actually listening inside the container
docker exec -it <container> sh
netstat -tulpn     # or ss -tulpn
curl localhost:3000/health
```

**Common Flask/Django mistake:**

```python
# WRONG - binds to localhost only (unreachable from outside container)
app.run(host='127.0.0.1', port=5000)

# CORRECT - binds to all interfaces
app.run(host='0.0.0.0', port=5000)
```

---

### 4. Container Keeps Restarting / Exiting Immediately

You run the container, it starts, then immediately exits with no obvious error.

```
Debugging steps:
─────────────────────────────────────────────────────────

  Step 1: Check exit code
    docker inspect --format='{{.State.ExitCode}}' <container>
    
    Exit 0   → Process finished normally (not a crash, app just completed)
    Exit 1   → Application error (check logs)
    Exit 137  → Killed by OOM (out of memory) or SIGKILL
    Exit 139  → Segmentation fault
    Exit 143  → Graceful termination (SIGTERM)

  Step 2: Check logs
    docker logs <container>
    docker logs --tail 50 <container>

  Step 3: Run interactively to see what happens
    docker run -it myapp /bin/sh
```

**Common causes:**

```
Cause 1: CMD runs a script that finishes and exits
  → Container is supposed to be long-lived but CMD exits
  → Fix: Make sure CMD runs a long-lived process (e.g., web server)

Cause 2: Missing config file or environment variable
  → App starts, can't find DB_URL, crashes
  → Fix: Pass env vars: docker run -e DB_URL=... myapp

Cause 3: OOM killed (exit code 137)
  → Container used more memory than allowed
  → Fix: Increase memory limit: docker run --memory=512m myapp
  → Or fix the memory leak in your application
```

---

### 5. Docker Daemon Disk Full

One of the most common production surprises. Docker silently fills up your disk with old images, containers, volumes, and build cache.

```
Symptoms:
─────────────────────────────────────────────────────────
  - docker build fails with "no space left on device"
  - Containers won't start
  - Host machine becomes unresponsive
```

```bash
# Check what's eating disk space
docker system df

# Output example:
# TYPE            TOTAL   ACTIVE   SIZE      RECLAIMABLE
# Images          45      3        12.8GB    11.2GB (87%)
# Containers      12      2        850MB     780MB (91%)
# Build Cache     -       -        5.4GB     5.4GB
# Volumes         8       2        3.2GB     2.8GB (87%)

# Clean up unused resources
docker system prune              # Remove stopped containers + dangling images
docker system prune -a           # Also remove all unused images
docker system prune -a --volumes # Nuclear option: remove everything unused

# Clean up specifically
docker container prune    # Remove stopped containers
docker image prune -a     # Remove unused images
docker volume prune       # Remove unused volumes
docker builder prune      # Remove build cache
```

**Logs filling disk (silent killer):**

```bash
# Docker logs have NO size limit by default
# A crash-looping container can write GBs of logs

# Check log file size
du -sh /var/lib/docker/containers/*/*-json.log

# Fix: Set log limits globally in /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
# Restart Docker daemon after this change
sudo systemctl restart docker
```

---

### 6. Image Size Too Large

Your Docker image is 2GB+ and takes forever to build, push, and pull.

```
Common bloat causes:
─────────────────────────────────────────────────────────

  Cause 1: Using a fat base image
    FROM ubuntu:latest        → ~78MB base
    FROM node:18              → ~350MB base (includes full Debian)
    FROM node:18-alpine       → ~50MB base ✅

  Cause 2: Dev dependencies in production image
    RUN npm install           → Installs devDependencies too
    RUN npm ci --only=production  → Production deps only ✅

  Cause 3: Build tools left in final image
    → Use multi-stage builds to separate build and runtime

  Cause 4: Leftover cache and temp files
    RUN apt-get update && apt-get install -y curl
    → Leaves cache behind
    RUN apt-get update && apt-get install -y curl \
        && rm -rf /var/lib/apt/lists/*  ← Clean up ✅
```

```bash
# Check what's making your image large
docker history myapp:latest

# Compare image sizes
docker images | grep myapp
```

**Multi-stage build to reduce size:**

```dockerfile
# Stage 1: Build (has all build tools)
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Runtime (only what's needed to run)
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]

# Result: Image drops from ~800MB to ~150MB
```

---

### 7. Permission Denied Errors

Container fails with "Permission denied" when reading/writing files, especially with mounted volumes.

```
Scenario: App can't write to a mounted volume
─────────────────────────────────────────────────────────

  docker run -v $(pwd)/data:/app/data myapp
  → Error: EACCES: permission denied, open '/app/data/output.json'

  Why: Host directory is owned by your user (UID 1000)
       Container runs as root (UID 0) or a different user
       → UID mismatch = permission denied
```

```bash
# Fix 1: Run container with your user's UID
docker run -u $(id -u):$(id -g) -v $(pwd)/data:/app/data myapp

# Fix 2: Set correct permissions in Dockerfile
RUN chown -R appuser:appgroup /app/data
USER appuser

# Fix 3: Match UIDs between host and container
# In Dockerfile, create user with same UID as host user
RUN adduser -u 1000 -D appuser
USER appuser
```

---

### 8. Secrets Accidentally Baked Into Images

Sensitive credentials end up permanently stored in the Docker image layers.

```
BAD - Secret is in the image history forever:
─────────────────────────────────────────────────────────

  ENV DATABASE_PASSWORD=supersecret123     ← Visible in image layers
  COPY .env /app/.env                      ← .env baked into image
  RUN echo "password" > /app/config.txt    ← Even if you delete it later,
  RUN rm /app/config.txt                      the earlier layer still has it
```

```bash
# Anyone can see secrets in image layers
docker history myapp:latest
docker inspect myapp:latest
```

```
SAFE approaches:
─────────────────────────────────────────────────────────

  1. Pass secrets at runtime (never at build time)
     docker run -e DATABASE_PASSWORD=secret myapp

  2. Use .dockerignore to prevent .env from being copied
     echo ".env" >> .dockerignore

  3. Use Docker BuildKit secrets for build-time secrets
     docker build --secret id=mysecret,src=secret.txt .

  4. Use multi-stage builds - secrets in builder stage
     don't carry over to the final image
```

---

### 9. "Cannot Kill Container" / Zombie Processes

Container gets stuck and even `docker stop` or `docker rm -f` doesn't work.

```
Scenario: docker stop hangs for 30 seconds then force-kills
─────────────────────────────────────────────────────────

  Why: Your app doesn't handle SIGTERM signal
  → docker stop sends SIGTERM → app ignores it → waits 10s timeout
  → docker sends SIGKILL (force kill)

  Impact: No graceful shutdown, in-flight requests are dropped,
          DB connections not closed properly
```

```dockerfile
# Fix 1: Use tini or dumb-init as PID 1 (handles signals properly)
FROM node:18-alpine
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]

# Fix 2: Handle SIGTERM in your application code
# Node.js example:
# process.on('SIGTERM', () => {
#   console.log('Shutting down gracefully...');
#   server.close(() => process.exit(0));
# });
```

```bash
# Check for zombie/orphan processes inside container
docker top <container>

# Nuclear option if container is truly stuck
docker rm -f <container>

# If even that fails
sudo kill -9 $(docker inspect --format='{{.State.Pid}}' <container>)
```

---

### Quick Reference: Common Docker Issues & Fixes

```
Issue                           │ Likely Cause                      │ Quick Fix
────────────────────────────────┼───────────────────────────────────┼──────────────────────────
App works locally, fails in     │ Missing deps / wrong OS / version │ Match env in Dockerfile
  container                     │                                   │
Build is slow / cache busts     │ Bad layer ordering in Dockerfile  │ COPY deps first, code last
Container runs but port not     │ App on 127.0.0.1 / port not      │ Bind 0.0.0.0, use -p flag
  reachable                     │   published                       │
Container exits immediately     │ CMD finishes / missing env var /  │ Check logs + exit code
                                │   OOM killed                      │
Disk full                       │ Old images, volumes, log files    │ docker system prune -a
Image too large                 │ Fat base, dev deps, no multi-     │ Alpine + multi-stage build
                                │   stage                           │
Compose containers can't        │ Using localhost instead of        │ Use service name as host
  talk to each other            │   service name                    │
Permission denied on volumes    │ UID mismatch between host and     │ Run with -u or match UIDs
                                │   container                       │
Secrets in image                │ ENV / COPY .env in Dockerfile     │ Pass at runtime with -e
docker stop hangs               │ App doesn't handle SIGTERM        │ Use tini + handle signals
```

---
