# Session 8-01: Kubernetes Foundations
## Minikube Setup, First Pod & kubectl Essentials

---

## Prerequisites

> **Before this session:** You should be comfortable with Docker (building images, running containers) and basic Linux commands from earlier sessions.

### Required Tools

| Tool | Version | Install |
|------|---------|---------|
| Docker | Latest | https://github.com/aryanm12/PPMCAD15/blob/main/01-Docker%26Containers/01-docker-installation-guide.md |
| kubectl | Latest | See Step 1 below |
| Minikube | Latest | See Step 1 below |
| Git | Latest | `brew install git` / `sudo apt install git` |

### Hardware Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB free | 8 GB free |
| Disk | 20 GB free | 40 GB free |

---

## ════════════════════════════════════════════════════════
## Lab 1 - Install Minikube & kubectl
## ════════════════════════════════════════════════════════

**Objective:** Install Minikube and kubectl on your local machine. Verify the cluster starts successfully and you can communicate with it.

### What You'll Learn
- Install kubectl (the Kubernetes CLI)
- Install Minikube (local K8s cluster)
- Start your first Kubernetes cluster
- Verify cluster health

---

### Step 1: Install kubectl

kubectl is the command-line tool you'll use for ALL Kubernetes interactions. Think of it as your remote control for the cluster.

**macOS:**
```bash
# Using Homebrew
brew install kubectl

# Verify
kubectl version --client
```

**Linux (Ubuntu/Debian):**
```bash
# Download latest stable
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
```

**Windows (PowerShell as Administrator):**
```powershell
# Using Chocolatey
choco install kubernetes-cli

# Or download directly
curl.exe -LO "https://dl.k8s.io/release/v1.31.0/bin/windows/amd64/kubectl.exe"
# Move kubectl.exe to a directory in your PATH

# Verify
kubectl version --client
```

### Step 2: Install Minikube

Minikube creates a single-node Kubernetes cluster on your local machine using Docker as the driver.

**macOS:**
```bash
brew install minikube
```

**Linux:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

**Windows (PowerShell as Administrator):**
```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
choco install minikube

# Or download installer from: https://minikube.sigs.k8s.io/docs/start/
```

### Step 3: Start Your First Cluster

```bash
# Start Minikube with Docker driver
minikube start --driver=docker

# If Minikube is running on Windows with HyperV use below command
# Enable Hyper-V:
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
Restart your computer
minikube start --driver=hyperv --container-runtime=containerd

# You should see output like:
# Done! kubectl is now configured to use "minikube" cluster
```

> **What just happened?** Minikube created a Docker container running a complete Kubernetes cluster inside it. This container has the Control Plane (API Server, etcd, Scheduler, Controller Manager) AND a worker node - all in one.

### Step 4: Verify Everything Works

```bash
# Check Minikube status
minikube status
# Expected: host: Running, kubelet: Running, apiserver: Running

# Check cluster info
kubectl cluster-info
# Expected: Kubernetes control plane is running at https://...

# Check the node
kubectl get nodes
# Expected: minikube   Ready   control-plane   ...

# Check system pods (these run Kubernetes itself)
kubectl get pods -n kube-system
# You should see: coredns, etcd, kube-apiserver, kube-controller-manager,
#                 kube-proxy, kube-scheduler, storage-provisioner
```

### Step 5: Explore the Dashboard

```bash
# Open the Kubernetes web dashboard
minikube dashboard
# This opens a browser with a visual overview of your cluster
```

> **Tip:** The dashboard is great for beginners to visualize what's happening. In production, teams use CLI + monitoring tools instead.

### ✅ Lab 1 Success Criteria

- kubectl and Minikube installed successfully
- `minikube status` shows Running
- `kubectl get nodes` shows minikube node in Ready state
- Dashboard opens in browser

---

## ════════════════════════════════════════════════════════
## Lab 2 - Understanding kubeconfig
## ════════════════════════════════════════════════════════

**Objective:** Understand how kubectl knows which cluster to talk to. Explore the kubeconfig file that Minikube created for you.

### What You'll Learn
- What kubeconfig is and where it lives
- Read and understand clusters, users, and contexts
- Switch between contexts (preview for when you have multiple clusters)

---

### Step 1: Locate and Read Your kubeconfig

```bash
# The kubeconfig file location
cat ~/.kube/config
```

or 

```powershell
cat C:\Users\<username>\.kube\config
```
You'll see three main sections. Let's break them down:

**clusters** - Where to connect:
```yaml
clusters:
- cluster:
    certificate-authority: /path/to/ca.crt
    server: https://192.168.49.2:8443    # <-- Minikube API server address
  name: minikube
```

**users** - How to authenticate:
```yaml
users:
- name: minikube
  user:
    client-certificate: /path/to/client.crt
    client-key: /path/to/client.key
```

**contexts** - Combines cluster + user:
```yaml
contexts:
- context:
    cluster: minikube
    namespace: default
    user: minikube
  name: minikube
current-context: minikube     # <-- This is what kubectl uses RIGHT NOW
```

### Step 2: Use kubectl config Commands

```bash
# See all configured contexts
kubectl config get-contexts
# The asterisk (*) shows your active context

# See current context
kubectl config current-context
# Expected: minikube

# View the full merged config
kubectl config view

# See just the clusters
kubectl config get-clusters
```

### Step 3: Understand the Flow

```
You type: kubectl get pods
         ↓
kubectl reads ~/.kube/config
         ↓
Finds current-context: minikube
         ↓
Looks up context "minikube" → cluster: minikube, user: minikube
         ↓
Connects to server: https://192.168.49.2:8443
Uses client certificate for authentication
         ↓
API Server returns the list of pods
```

> **Why this matters:** When you later work with EKS (AWS managed Kubernetes) or self managed Kubernetes Cluster, new contexts will get added to this same file. You'll switch between your local Minikube and cloud EKS cluster using `kubectl config use-context`.

### ✅ Lab 2 Success Criteria

- Can read and explain the three sections of kubeconfig (clusters, users, contexts)
- `kubectl config get-contexts` shows your Minikube context with `*`
- Understand that `current-context` determines which cluster kubectl talks to

---

## ════════════════════════════════════════════════════════
## Lab 3 - Your First Pod
## ════════════════════════════════════════════════════════

**Objective:** Create your first Kubernetes Pod both imperatively (command line) and declaratively (YAML manifest). Understand the difference between the two approaches.

### What You'll Learn
- Create pods using `kubectl run` (imperative)
- Create pods using YAML manifests (declarative)
- Inspect, describe, and troubleshoot pods
- View pod logs and exec into containers
- Delete pods

---

### Step 1: Create a Pod Imperatively

The imperative approach is quick for testing - you tell Kubernetes *exactly what to do*:

```bash
# Create a simple nginx pod
kubectl run my-nginx --image=nginx:latest

# Check it's running
kubectl get pods
# Expected: my-nginx   1/1   Running   0   ...

# Get more details
kubectl get pods -o wide
# Shows IP address, node, and container image
```

### Step 2: Inspect the Pod

```bash
# Describe gives you EVERYTHING about the pod
kubectl describe pod my-nginx
# Look for:
#   - Status: Running
#   - IP: (the pod's internal IP)
#   - Events: (pull image, create container, start container)

# Check pod logs
kubectl logs my-nginx
# Shows nginx access/error logs

# Follow logs in real-time (like tail -f)
kubectl logs my-nginx -f
# Press Ctrl+C to stop following
```

### Step 3: Exec Into the Pod

Just like `docker exec`, you can get a shell inside a running pod:

```bash
# Get a shell inside the nginx container
kubectl exec -it my-nginx -- /bin/bash

# Inside the container, try:
hostname
cat /etc/nginx/nginx.conf
curl localhost:80
exit
```

> **Key Connection:** This is the same concept as `docker exec -it <container> /bin/bash` from our Docker sessions. The difference is kubectl routes through the API Server → kubelet → container runtime, instead of talking directly to Docker.

### Step 4: Create a Pod Declaratively (YAML)

The declarative approach is what you'll use in production - you describe the *desired state* and K8s figures out how to get there.

Create the file `demo-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-k8s
  labels:
    app: hello
    environment: lab
spec:
  containers:
  - name: hello-container
    image: busybox
    command: ['sh', '-c', 'echo Hello from Kubernetes! && sleep 3600']
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "250m"
```

**Let's break down the YAML - the A-K-M-S pattern:**

```
A → apiVersion: v1          # Which API group (v1 for core resources like Pods)
K → kind: Pod               # What type of resource
M → metadata:               # Identity - name, labels, namespace
      name: hello-k8s
      labels:
        app: hello
S → spec:                   # Desired state - what containers to run
      containers:
      - name: hello-container
        image: busybox
```

Apply it:

```bash
# Create the pod from YAML
kubectl apply -f demo-pod.yaml

# Check it
kubectl get pods
# Expected: hello-k8s   1/1   Running   0   ...

# View the logs
kubectl logs hello-k8s
# Expected: Hello from Kubernetes!

# Check the labels
kubectl get pods --show-labels
```

### Step 5: Understand Imperative vs Declarative

```bash
# Imperative: "Create this pod right now"
kubectl run my-pod --image=nginx

# Declarative: "Make sure this pod exists as described"
kubectl apply -f demo-pod.yaml

# Declarative is idempotent - run it again:
kubectl apply -f demo-pod.yaml
# Output: pod/hello-k8s unchanged  (nothing to change!)
```

> **Best Practice:** Always use declarative (YAML + `kubectl apply`) in real projects. Imperative is only for quick tests and debugging.

### Step 6: Clean Up

```bash
# Delete the imperative pod
kubectl delete pod my-nginx

# Delete the declarative pod
kubectl delete -f demo-pod.yaml

# Verify
kubectl get pods
# Expected: No resources found in default namespace.
```

### ✅ Lab 3 Success Criteria

- Created pods both imperatively and declaratively
- Used `kubectl describe`, `kubectl logs`, and `kubectl exec`
- Understand the A-K-M-S YAML structure
- Know the difference between imperative and declarative approaches

---

## ════════════════════════════════════════════════════════
## Lab 4 - Multi-Container Pods & Pod Lifecycle
## ════════════════════════════════════════════════════════

**Objective:** Create a multi-container pod to understand the sidecar pattern. Observe pod lifecycle states and understand restart policies.

### What You'll Learn
- Create a multi-container (sidecar) pod
- Understand shared networking in a pod (localhost communication)
- Observe pod lifecycle: Pending → Running → Succeeded/Failed
- Understand restart policies

---

### Step 1: Create a Multi-Container Pod

In this pod, one container writes logs and another reads them. They share storage via an `emptyDir` volume.

Create `sidecar-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
  labels:
    app: sidecar-demo
spec:
  containers:
  # Main application container - writes logs
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do echo "$(date) - App is running" >> /var/log/app.log; sleep 5; done']
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log

  # Sidecar container - reads and displays logs
  - name: log-reader
    image: busybox
    command: ['sh', '-c', 'tail -f /var/log/app.log']
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log

  volumes:
  - name: shared-logs
    emptyDir: {}
```

```bash
# Create the pod
kubectl apply -f sidecar-pod.yaml

# Check both containers are running
kubectl get pods sidecar-demo
# Expected: sidecar-demo   2/2   Running   0   ...
# The 2/2 means both containers are running!

# View logs from the sidecar (log-reader)
kubectl logs sidecar-demo -c log-reader
# Expected: timestamps with "App is running"

# View logs from the main app
kubectl logs sidecar-demo -c app
# No stdout output (it writes to a file)
```

### Step 2: Verify Shared Networking

Both containers in the same pod share the network namespace (same IP, same localhost):

```bash
# Exec into the app container
kubectl exec -it sidecar-demo -c app -- sh

# Check the pod's IP
hostname -i

# Exit and exec into the log-reader container
exit
kubectl exec -it sidecar-demo -c log-reader -- sh

# Check the IP again - it's the SAME
hostname -i
# Both containers see the same IP address!

exit
```

### Step 3: Observe Pod Lifecycle

```bash
# Create a pod that completes (exits after running)
kubectl run lifecycle-demo --image=busybox --restart=Never -- sh -c "echo 'Job done!' && sleep 10 && echo 'Exiting'"

# Watch the pod lifecycle
kubectl get pods lifecycle-demo -w
# You'll see: Pending → Running → Completed

# Press Ctrl+C to stop watching

# Check the status
kubectl get pods lifecycle-demo
# Expected: lifecycle-demo   0/1   Completed   0   ...

# Read its output
kubectl logs lifecycle-demo
```

### Step 4: See a Pod Fail

```bash
# Create a pod with a bad image (will fail to pull)
kubectl run bad-pod --image=nginx:nonexistent-tag

# Watch what happens
kubectl get pods bad-pod -w
# You'll see: ErrImagePull or ImagePullBackOff

# Describe to see the events
kubectl describe pod bad-pod
# Look at Events section - you'll see the pull failure details

# Clean up
kubectl delete pod bad-pod lifecycle-demo
kubectl delete -f sidecar-pod.yaml
```

### ✅ Lab 4 Success Criteria

- Multi-container pod running with 2/2 containers
- Verified shared networking (same IP) between containers
- Observed pod lifecycle: Pending → Running → Completed
- Saw ImagePullBackOff error and used describe to debug it

---

## ════════════════════════════════════════════════════════
## Lab 5 - Exploring the Cluster
## ════════════════════════════════════════════════════════

**Objective:** Use kubectl to explore your Kubernetes cluster components. Understand what runs in kube-system and how the control plane works.

### What You'll Learn
- Explore kube-system namespace (where K8s itself runs)
- Understand what each system component does
- Use different kubectl output formats
- Learn useful kubectl shortcuts

---

### Step 1: Explore System Components

```bash
# List ALL namespaces
kubectl get namespaces
# Expected: default, kube-node-lease, kube-public, kube-system

# Look at the system pods - these ARE Kubernetes
kubectl get pods -n kube-system
# You should see:
#   coredns-*                ← DNS for service discovery
#   etcd-minikube            ← Database storing all cluster state
#   kube-apiserver-*         ← The API gateway (everything talks to this)
#   kube-controller-manager  ← Watches and maintains desired state
#   kube-proxy-*             ← Network rules for Service routing
#   kube-scheduler-*         ← Decides which node runs which pod
#   storage-provisioner      ← Minikube's storage handler
```

### Step 2: Different Output Formats

```bash
# Default table output
kubectl get pods -n kube-system

# Wide output - shows node and IP
kubectl get pods -n kube-system -o wide

# YAML output - see the full resource definition
kubectl get pod -n kube-system etcd-minikube -o yaml

# JSON output (useful for scripting with jq)
kubectl get pods -n kube-system -o json | jq '.items[].metadata.name'

# Custom columns
kubectl get pods -n kube-system -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP

# Just the names
kubectl get pods -n kube-system -o name
```

### Step 3: Inspect the Node

```bash
# Get node details
kubectl describe node minikube
```

### Step 4: Useful kubectl Shortcuts

```bash
# Short names (aliases) - these all work:
kubectl get po          # pods
kubectl get svc         # services
kubectl get deploy      # deployments
kubectl get ns          # namespaces
kubectl get no          # nodes

# Get ALL resources in the current namespace
kubectl get all

# Get ALL resources across ALL namespaces
kubectl get all -A

# Watch for changes in real-time
kubectl get pods -w

# Show API resources and their short names
kubectl api-resources | head -20
```

### Step 5: Create and Use a Custom Namespace

```bash
# Create a namespace for our labs
kubectl create namespace k8s-labs

# List namespaces
kubectl get namespaces

# Create a pod in our new namespace
kubectl run lab-nginx --image=nginx -n k8s-labs

# List pods - default namespace (empty)
kubectl get pods

# List pods in our namespace
kubectl get pods -n k8s-labs

# Set k8s-labs as the default namespace for convenience
kubectl config set-context --current --namespace=k8s-labs

# Now kubectl commands default to k8s-labs
kubectl get pods
# Expected: lab-nginx is shown!

# Switch back to default namespace
kubectl config set-context --current --namespace=default

# Clean up
kubectl delete namespace k8s-labs
```

### ✅ Lab 5 Success Criteria

- Can list and explain the main kube-system components
- Used multiple output formats (-o wide, -o yaml, -o json)
- Inspected node resources with `kubectl describe node`
- Created a namespace, deployed a pod into it, and cleaned up

---