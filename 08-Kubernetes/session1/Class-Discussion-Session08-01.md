# Class Discussion - Session 8
## Kubernetes Foundations & Container Orchestration

---

## Why Container Orchestration? - The Real Problem

### What Happens When Docker Isn't Enough

In earlier sessions, we learned Docker gives us portable, consistent containers. But in production, you don't run just one container - you run **dozens to hundreds** across multiple servers.

```
The "Docker in Production" Problem:
────────────────────────────────────────────────────────────────────

  Developer's laptop (Docker works great here)
  ┌─────────────────────────────────────┐
  │  docker run -d -p 8080:8080 myapp   │  ← Simple. One command.
  └─────────────────────────────────────┘

  Production (multiple servers, many containers)
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │  Server 1   │  │  Server 2   │  │  Server 3   │
  │  App x 3    │  │  App x 3    │  │  App x 3    │
  │  Redis x 1  │  │  Worker x 2 │  │  DB x 1     │
  └─────────────┘  └─────────────┘  └─────────────┘

  Who decides which container goes where?
  What if Server 2 crashes at 3 AM?
  How do you scale from 3 → 10 app containers?
  How does the frontend find the backend?
```

### The Questions Docker Can't Answer Alone

```
Scenario                            │  Docker Alone          │  With Kubernetes
────────────────────────────────────┼────────────────────────┼──────────────────────────
Container crashes at 2 AM           │  Nobody notices         │  Auto-restarts in seconds
Traffic spikes 10x                  │  Manual SSH + docker run│  HPA scales pods automatically
Server runs out of memory           │  Containers get killed  │  Scheduler places pods elsewhere
New version needs to deploy         │  Stop old, start new    │  Rolling update, zero downtime
Need 5 instances of the same app    │  Run docker 5 times     │  replicas: 5 in YAML
Frontend needs to find backend      │  Hardcode IP addresses  │  DNS-based service discovery
```
---

### How a Deployment Request Flows Through the Cluster

```
Step-by-step: What happens when you run "kubectl apply -f deployment.yaml"

1. kubectl sends the request to the API Server
     ↓
2. API Server validates the YAML and stores it in etcd
     ↓
3. Scheduler notices "there's a pod that needs a home"
   Checks: which node has enough CPU/memory?
   Decision: "Place it on Worker 2"
     ↓
4. kubelet on Worker 2 receives the instruction
   Pulls the container image (if not cached)
   Starts the container via the container runtime (containerd)
     ↓
5. kube-proxy updates network rules so the pod is reachable
     ↓
6. Controller Manager continuously watches:
   "Desired state: 3 replicas. Current state: 3 running. All good."
   If a pod dies: "Current state: 2. Need to create 1 more."
```


---

## Pods - The Atomic Unit of Kubernetes

### What is a Pod?

A Pod is **not** a container. A Pod is a **wrapper around one or more containers** that share:

```
┌──────────────────────────────────────────┐
│                   POD                    │
│                                          │
│  Shared Network Namespace:               │
│  ┌──────────┐   ┌──────────┐             │
│  │Container │   │Container │             │
│  │  (App)   │←→ │ (Sidecar)│             │
│  └──────────┘   └──────────┘             │
│       ↕              ↕                   │
│  Communicate via localhost               │
│  Same IP address: 10.0.0.15              │
│                                          │
│  Shared Storage:                         │
│  ┌──────────────────────────────┐        │
│  │      Shared Volume           │        │
│  │   (both containers can R/W)  │        │
│  └──────────────────────────────┘        │
└──────────────────────────────────────────┘
```

### Pod Lifecycle States

```
State              │  What It Means
───────────────────┼──────────────────────────────────────────────
Pending            │  Pod accepted but not yet scheduled/running
                   │  (waiting for node, pulling image, etc.)
Running            │  At least one container is running
Succeeded          │  All containers exited with code 0 (Job completed)
Failed             │  All containers terminated, at least one failed
Unknown            │  Cannot determine state (usually node communication issue)

Common Error States:
───────────────────────────────────────────────────────────────────
ImagePullBackOff   │  Can't pull the container image (wrong name/tag, no auth)
CrashLoopBackOff   │  Container starts, crashes, restarts, crashes again
OOMKilled          │  Container exceeded memory limits, killed by kernel
CreateContainerError│  Container runtime can't create the container
```

---

## YAML Manifests - The A-K-M-S Pattern

### Every Kubernetes Resource Follows the Same Structure

```yaml
apiVersion: v1              # A - Which API version
kind: Pod                   # K - What type of resource
metadata:                   # M - Identity (name, labels, namespace)
  name: my-app
  labels:
    app: web
spec:                       # S - Desired state (what you want to happen)
  containers:
  - name: web
    image: nginx:1.25
```

### Common API Versions

```
apiVersion │ Resources
───────────┼──────────────────────────────
v1         │ Pod, Service, ConfigMap, Secret, Namespace, PersistentVolume
apps/v1    │ Deployment, StatefulSet, DaemonSet, ReplicaSet
batch/v1   │ Job, CronJob
networking │ Ingress, NetworkPolicy
```

### Labels - How Kubernetes Organizes Everything

Labels are key-value pairs attached to resources. They're how K8s connects things:

```yaml
# Pod with labels
metadata:
  labels:
    app: frontend          # What application is this?
    environment: production # Which environment?
    version: v2.1          # Which version?
    team: platform         # Who owns it?

# A Service selects pods by labels
spec:
  selector:
    app: frontend          # "Route traffic to all pods with app=frontend"
```

---

## kubectl

### Command Structure

```
kubectl  <action>  <resource>  <name>  <flags>

Examples:
kubectl  get       pods                           # List all pods
kubectl  get       pods       my-app              # Get specific pod
kubectl  get       pods       -n kube-system      # In a specific namespace
kubectl  describe  pod        my-app              # Full details
kubectl  delete    pod        my-app              # Remove it
kubectl  apply     -f         deployment.yaml     # Create/update from YAML
kubectl  logs      my-app                         # View stdout/stderr
kubectl  exec      -it        my-app -- /bin/bash # Shell into container
```

### Most Used Commands - Cheat Sheet

```
Viewing Resources:
  kubectl get pods                    # List pods in current namespace
  kubectl get pods -A                 # List pods in ALL namespaces
  kubectl get pods -o wide            # Show IP, node, etc.
  kubectl get all                     # Show pods, services, deployments, etc.
  kubectl describe pod <name>         # Full details + events
  kubectl top pods                    # CPU/memory usage

Creating / Updating:
  kubectl apply -f file.yaml          # Declarative create/update
  kubectl run <name> --image=<img>    # Quick imperative create
  kubectl create namespace <name>     # Create a namespace

Debugging:
  kubectl logs <pod>                  # View container output
  kubectl logs <pod> -f               # Follow logs (live tail)
  kubectl logs <pod> -c <container>   # Logs from specific container
  kubectl logs <pod> --previous       # Logs from last crash
  kubectl exec -it <pod> -- sh        # Shell into container
  kubectl describe pod <pod>          # Check Events section

Deleting:
  kubectl delete -f file.yaml         # Delete what the YAML defined
  kubectl delete pod <name>           # Delete specific pod
  kubectl delete pods --all           # Delete all pods in namespace
```

---

## kubeconfig - How kubectl Finds Your Cluster

```
Location: ~/.kube/config

Structure:
┌─────────────────────────────────────────────────────────┐
│                      kubeconfig                         │
│                                                         │
│  clusters:                                              │
│    - minikube: https://192.168.49.2:8443                │
│    - eks-prod: https://ABC123.eks.amazonaws.com         │
│                                                         │
│  users:                                                 │
│    - minikube: client certificate                       │
│    - eks-admin: aws-iam-authenticator                   │
│                                                         │
│  contexts: (cluster + user + namespace)                 │
│    - local:    minikube cluster + minikube user         │
│    - prod:     eks-prod cluster + eks-admin user        │
│                                                         │
│  current-context: local  ← kubectl uses THIS one        │
└─────────────────────────────────────────────────────────┘

Switching clusters:
  kubectl config use-context prod     # Now talking to EKS
  kubectl config use-context local    # Back to Minikube
```

---

## Minikube - Local Development Cluster

### What Minikube Actually Creates

```
Your Computer
┌──────────────────────────────────────────┐
│                                          │
│  ┌────────────────────────────────────┐  │
│  │        Docker Container            │  │
│  │     (the Minikube "node")          │  │
│  │                                    │  │
│  │  Control Plane Components:         │  │
│  │    - API Server                    │  │
│  │    - etcd                          │  │
│  │    - Scheduler                     │  │
│  │    - Controller Manager            │  │
│  │                                    │  │
│  │  Worker Components:                │  │
│  │    - kubelet                       │  │
│  │    - kube-proxy                    │  │
│  │    - Container Runtime             │  │
│  │                                    │  │
│  │  Your Pods:                        │  │
│  │    - nginx, flask-app, etc.        │  │
│  └────────────────────────────────────┘  │
│                                          │
│  Docker Engine (runs the Minikube VM)    │
└──────────────────────────────────────────┘

Note: In production, the control plane and workers are on SEPARATE machines.
Minikube puts everything in ONE container for simplicity.
```

### Minikube vs Production Kubernetes

```
                     │  Minikube              │  Production (EKS, kubeadm)
─────────────────────┼────────────────────────┼──────────────────────────────
Nodes                │  1 (single node)       │  Multiple (3+ workers typical)
Control Plane        │  Same node as workloads│  Dedicated machines (HA)
Networking           │  Simplified            │  Full CNI plugin (VPC CNI, Cilium)
Storage              │  Local provisioner     │  EBS, EFS, cloud volumes
Load Balancing       │  minikube tunnel       │  Cloud ALB/NLB
Cost                 │  Free (local machine)  │  Cloud compute costs
Purpose              │  Learning & dev        │  Real workloads
```

---