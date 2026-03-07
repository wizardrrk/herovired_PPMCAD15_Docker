# Class Discussion - Session 1
---

## Traditional Application Deployment

### How We Used to Deploy Applications

**For a typical Java/Python/Node.js application:**

1. **Get a hosting machine**
   - Physical server
   - Virtual Machine (e.g., EC2 on AWS, Azure VM, GCP Compute)

2. **Setup the environment**
   - Copy application artifacts
   - Install dependencies
   - Configure runtime environment

3. **Run the application**
   ```bash
   # Java
   java -jar application.jar
   
   # Python
   python app.py
   
   # Node.js
   node index.js
   ```

---

### Enterprise Deployment Pipeline

In a typical enterprise setup, code moves through multiple environments:

```
Developer Workstation
        ↓
Development Environment
        ↓
QA Environment
        ↓
UAT/Staging Environment
        ↓
Production Environment
```

Each environment requires:
- Application code
- Runtime (Java, Python, Node.js)
- Dependencies (libraries, packages)
- Configuration
- Operating system libraries

---

## The "Works on My Machine" Problem

### Common Scenarios

**Scenario 1: UAT Failure**
- Code breaks in UAT environment
- Developer says: "It works perfectly on my machine!"
- Team spends hours debugging

**Scenario 2: Production Issues**
- Code works fine in UAT
- Breaks in production
- Emergency fixes required

### Root Causes

**1. Different Artifacts**
- Build artifacts may differ between environments
- Compilation settings might vary
- Different build tools or versions

**2. Dependency Mismatch**
- Libraries not installed properly
- Different versions across environments
- Missing dependencies

**3. Runtime Version Differences**
```
Developer Machine: Python 3.9
Development:       Python 3.9
QA:                Python 3.8  ← Mismatch!
UAT:               Python 3.9
Production:        Python 3.7  ← Major issue!
```

**4. Operating System Differences**
- Developer: macOS or Windows
- Development: Ubuntu 20.04
- Production: CentOS 7
- Different OS libraries and behaviors

**5. Configuration Drift**
- Environment variables differ
- Configuration files not synced
- Secrets management inconsistencies

---

## Enter Docker: The Solution

### The Docker Promise

Docker said: **"We'll give you a framework to package everything together!"**

### What Gets Packaged in a Docker Image?

```
┌─────────────────────────────────────────┐
│  Docker Image                           │
├─────────────────────────────────────────┤
│  Your Application Code                  │
├─────────────────────────────────────────┤
│  All Dependencies (libraries, packages) │
├─────────────────────────────────────────┤
│  Language Runtime (Python, Java, Node)  │
├─────────────────────────────────────────┤
│  Operating System Libraries             │
├─────────────────────────────────────────┤
│  3rd Party Tightly Coupled Packages     │
└─────────────────────────────────────────┘
```

### The Result

**One image → Run anywhere**

- Same image in development
- Same image in QA
- Same image in UAT
- Same image in production

**Guaranteed consistency across all environments!**

---

## History of Docker

### Timeline

**2013:** Docker released as open-source project

**2014:** Docker becomes the standard for containerization

**Kubernetes Integration:**
- Kubernetes adopted Docker as its Container Runtime Interface (CRI)
- Versions 1.0 through 1.23 used Docker by default

**2022: Kubernetes 1.24 - Major Shift**
- Kubernetes announced Docker deprecation
- Default CRI changed to **containerd**
- Organizations given option to choose CRI:
  - containerd (default)
  - CRI-O
  - Docker (if needed)

### Why the Change?

Kubernetes needed a **lightweight, purpose-built** container runtime:
- Docker was feature-rich but heavyweight
- containerd extracted from Docker, optimized for Kubernetes
- Better performance, smaller footprint

---

## Container Ecosystem

### Understanding the Terminology

**Container:**
- The **technology** or concept
- Standardized way to package and run applications

**Container Products/Runtimes:**
- **Docker** (most popular)
- **containerd** (Kubernetes default)
- **CRI-O** (OpenShift, Kubernetes-native)
- **Podman** (Red Hat's daemonless alternative)

### Analogy: Just like version control:

Git (VCS technology) and GitHub, GitLab, Bitbucket are the products for running git

---

## Current Market Usage

### Image Building (Build Time)

**~95% of the tech world uses Docker** for building images

Why?
- Mature ecosystem
- Excellent documentation
- Developer-friendly
- Industry standard

### Container Runtime (Production)

**Different runtimes dominate different platforms:**

| Platform | Primary Runtime | Market Share |
|----------|----------------|--------------|
| **Kubernetes** | containerd | ~70% |
| **AWS ECS** | Docker | Majority |
| **Azure Container Instances** | Docker | Majority |
| **OpenShift** | CRI-O | Default |
| **Standalone** | Docker | Common |

---

## Container Orchestration

### Why Orchestration?

Running containers in production requires:
- Scaling (horizontal/vertical)
- Load balancing
- Service discovery
- Health monitoring
- Rolling updates
- Self-healing

### Major Orchestration Tools

**1. Kubernetes** (Most Popular)
- Industry standard
- Cloud-agnostic
- Massive ecosystem
- Complex but powerful

**2. AWS ECS (Elastic Container Service)**
- AWS-native
- Simpler than Kubernetes
- Tight AWS integration

**3. Docker Swarm**
- Native Docker orchestration
- Simpler than Kubernetes
- Less feature-rich

**4. Azure Container Instances**
- Serverless containers
- Pay-per-second
- Easy deployment

---

## Virtual Machines vs Containers

### Virtual Machine Architecture

**Example Setup:**

```
Physical Server
├── Hardware: 16 Core CPU, 64 GB RAM, 100 GB Storage
├── Host OS: Windows Server
├── Hypervisor: VMware / Hyper-V / VirtualBox
└── Virtual Machines:
    ├── VM1: Linux Ubuntu (4 cores, 16 GB RAM)
    ├── VM2: Windows Server (4 cores, 16 GB RAM)
    ├── VM3: CentOS (4 cores, 16 GB RAM)
    └── VM4: Debian (4 cores, 16 GB RAM)
```

### How VMs Work

**Hardware-Level Virtualization:**
- Hypervisor virtualizes physical hardware
- Each VM gets virtual CPU, RAM, storage, network
- **VMs share hardware** (CPU, memory, storage, network)
- **VMs DO NOT share OS** (each has own OS)

**Key Point:** VM OS is **independent** of host OS
- Windows host can run Linux VMs
- Linux host can run Windows VMs
- macOS can run both

---

## Docker on Windows

### The Paradox

**Question:** How can Linux containers run on Windows?

**Common confusion:**
- Windows laptop
- Install Docker Desktop
- Run Linux containers
- But we said "Linux containers can't run on Windows OS!"

### The Answer: Two Approaches

#### **Approach 1: Hyper-V (Legacy)**

```
Windows 10/11
    ↓
Hyper-V (Hypervisor)
    ↓
Lightweight Linux VM
    ↓
Docker Engine
    ↓
Linux Containers
```

**How it works:**
- Hyper-V creates a lightweight Linux VM
- Docker Desktop considers this VM as "host OS"
- Containers run inside this Linux VM

#### **Approach 2: WSL 2 (Recommended) **

**WSL = Windows Subsystem for Linux**

```
Windows 10/11
    ↓
WSL 2 (Linux kernel integration)
    ↓
Linux distribution (Ubuntu, Debian, etc.)
    ↓
Docker Engine
    ↓
Linux Containers
```

**What is WSL?**
- Linux kernel running natively in Windows
- Direct system call translation
- Better performance than Hyper-V
- Lower resource overhead
- Tight integration with Windows

**Important:** Docker Desktop treats WSL/Hyper-V VM as the host OS for Linux Containers, **not Windows itself**.

---

## Docker Registry

### What is a Docker Registry?

A **repository for Docker images** - like GitHub for code, but for container images.

### Popular Docker Registries

#### **Public Registries**

| Registry | Provider | URL | Notes |
|----------|----------|-----|-------|
| **Docker Hub** | Docker Inc. | hub.docker.com | Default, largest public registry |
| **GitHub Container Registry** | GitHub | ghcr.io | Integrated with GitHub |
| **GitLab Container Registry** | GitLab | registry.gitlab.com | Integrated with GitLab |
| **Quay.io** | Red Hat | quay.io | Enterprise-focused |

#### **Cloud Provider Registries**

| Registry | Provider | Abbreviation |
|----------|----------|--------------|
| **Elastic Container Registry** | AWS | ECR |
| **Azure Container Registry** | Microsoft | ACR |
| **Google Container Registry** | Google Cloud | GCR |

#### **Self-Hosted Private Registries**

| Product | Use Case |
|---------|----------|
| **Sonatype Nexus** | Enterprise artifact management |
| **JFrog Artifactory** | Universal artifact repository |
| **Harbor** | Cloud-native registry with security |
| **Docker Registry** | Lightweight open-source option |

### Registry Workflow

```
Developer Machine
    ↓ (docker build)
Local Image
    ↓ (docker push)
Docker Registry (Docker Hub, ECR, etc.)
    ↓ (docker pull)
Production Server
    ↓ (docker run)
Running Container
```

### Checking Image Pricing

**Question: How to know if Docker images are free or paid?**

**Docker Hub Tiers:**

1. **Free Images**
   - Official images (nginx, python, node, etc.)
   - Community images
   - Personal public repositories

2. **Paid/Subscription**
   - Docker Pro/Team/Business subscriptions
   - Private repositories (limits on free tier)
   - Rate limiting on anonymous pulls

**Check before using:**
```bash
# View image details on Docker Hub
docker search nginx

# Check official tag
docker pull nginx:alpine  # Official images are free
```

**Pro Tip:** Always use official images when available - they're:
- Free
- Well-maintained
- Security-scanned
- Documented

---

## Build vs Runtime

### Two Distinct Phases

```
BUILD TIME                      RUNTIME
-----------                    ---------
Create Image                   Run Container
    ↓                              ↓
docker build                   docker run
    ↓                              ↓
Dockerfile                     Image
    ↓                              ↓
Stored in Registry             Executing Container
```

### Build Time: Creating an Image

**What you do:**
1. Write a Dockerfile
2. Build the image
3. Push to registry

**Example Dockerfile:**
```dockerfile
# Base image
FROM ubuntu:22.04

# Add application code
COPY app.jar /app/

# Install dependencies
RUN apt-get update && apt-get install -y openjdk-17-jre

# Build application (if needed)
RUN mvn package

# Define how to run
CMD ["java", "-jar", "/app/app.jar"]
```

**Build command:**
```bash
docker build -f Dockerfile -t myapp:1.0 .
```

**Result:** Image created and ready to run

---

### Runtime: Running a Container

**What you do:**
1. Pull image from registry (if not local)
2. Run container from image
3. Access running application

**Example:**
```bash
# Run container from image
docker run -d \
  -p 8080:80 \
  --name my-app \
  myapp:1.0
```

**Result:** Container running, application accessible

---

### Port Conflict

```bash
# First container - works fine
docker run -d -p 8080:80 --name my-nginx nginx:alpine

# Second container - ERROR!
docker run -d -p 8080:80 --name my-apache httpd
# Error: port 8080 already occupied
```

**Why it fails:**
- Port 8080 on host already mapped to my-nginx
- Can't map same host port to two containers

**Solution - Use different port:**
```bash
# Use different host port
docker run -d -p 8181:80 --name my-apache httpd

# Now both work:
# nginx:  http://localhost:8080
# apache: http://localhost:8181
```

---


## Security & Vulnerability Scanning

### Why Scan Docker Images?

**Risks:**
- Base images may have vulnerabilities
- Dependencies with known CVEs
- Outdated packages
- Malicious code (rare, but possible)

### Popular Scanning Tools

| Tool | Provider | Features |
|------|----------|----------|
| **Trivy** | Aqua Security | Open-source, comprehensive, fast |
| **Snyk** | Snyk | Developer-first, integrates with CI/CD |
| **Tenable** | Tenable | Enterprise vulnerability management |
| **Rapid7** | Rapid7 | Application security scanning |
| **Docker Scan** | Docker | Built into Docker CLI |
| **Clair** | Quay | Open-source, API-driven |

---

## Container Runtime Alternatives

### Docker Alternatives

**Podman** (Red Hat)
- Daemonless architecture
- Rootless containers (better security)
- Docker CLI compatible
- No central daemon (more secure)

```bash
# Podman commands are identical to Docker
podman run -d -p 8080:80 nginx
podman ps
podman build -t myapp .
```

**Key Differences:**
- Docker: Client-server architecture (daemon)
- Podman: Fork-exec model (no daemon)

---
