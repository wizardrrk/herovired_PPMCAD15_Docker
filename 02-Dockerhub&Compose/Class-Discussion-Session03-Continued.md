# Class Discussion - Session 3 (Continued)
## Monolith vs Microservices & Running Multi-Container Applications

---

## Hands-On Task

> **For @all:** Modify your application and Dockerfile to make it a **long-lived container**.
> Currently, it runs as a short-lived container (exits after execution).
>
> Reference: [Session 2 Hands-On Labs — Lab 1](https://github.com/aryanm12/PPMCAD15/blob/main/02-Dockerhub%26Compose/Session2-Hands-On-Labs.md#lab-1-docker-hub-operations)

**Hint:** A long-lived container is one that keeps running (e.g., a web server listening on a port) rather than executing a script and exiting immediately.

---

## Monolith vs Microservices

### Monolithic Architecture

All components of the application run as a **single unit** on a **single machine**.

```
Monolith Application (Single Machine)
┌─────────────────────────────────────┐
│                                     │
│   Frontend + Backend + Database     │
│   + Auth + Payments + Notifications │
│                                     │
│         All in ONE process          │
│                                     │
└─────────────────────────────────────┘
```

- Everything is packaged, deployed, and scaled **together**
- Simple to develop initially, but becomes complex over time
- A failure in one component can bring down the **entire** application

### Microservices Architecture

Different components run as **independent services** on separate machines or containers.

```
Microservices (Multiple Containers / Machines)
┌────────────┐  ┌────────────┐  ┌────────────┐
│  Frontend  │  │  Backend   │  │  Database  │
│  Container │  │  Container │  │  Container │
└────────────┘  └────────────┘  └────────────┘

┌────────────┐  ┌────────────┐  ┌────────────┐
│    Auth    │  │  Payments  │  │   Redis    │
│  Container │  │  Container │  │  Container │
└────────────┘  └────────────┘  └────────────┘
```

- Each component is developed, deployed, and scaled **independently**
- Failure in one service doesn't necessarily crash the entire app
- More complex to manage, but offers better scalability and flexibility

---

## Inter-Service Communication

In real-world applications, multiple components work together. The key question is: **how do these different components talk to each other?**

### Typical Communication Flow

```
┌──────────┐        ┌──────────┐        ┌──────────┐
│ Frontend │ ----→  │ Backend  │ ----→  │ Database │
└──────────┘        └──────────┘        └──────────┘
                         │
                         ├----→  ┌──────────┐
                         │       │  Redis   │
                         │       │ (Cache)  │
                         │       └──────────┘
                         │
                         └----→  ┌──────────┐
                                 │  Other   │
                                 │ Services │
                                 └──────────┘
```

### E-Commerce Example

A typical e-commerce application has multiple services communicating in a chain:

```
┌──────────┐      ┌───────────┐      ┌─────────┐      ┌──────────┐
│ Frontend │ --→  │ Checkout  │ --→  │  Order  │ --→  │ Payments │
│          │      │  Service  │      │ Service │      │ Service  │
└──────────┘      └───────────┘      └─────────┘      └──────────┘
```

Each of these services could be running as a **separate container**, and they need a way to discover and communicate with each other.

---

## Running Multi-Container Applications

If we need to run a full-scale app using containers, where multiple services need to work together - how do we do it?

There are **two main approaches**:

---

### 1. Docker Compose

**Purpose:** Running multi-container applications on a **single machine**.

```
Docker Compose — Single Machine Setup
┌─────────────────────────────────────────────┐
│             Single Server / VM              │
│                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │   App   │  │  Redis  │  │   DB    │      │
│  │Container│  │Container│  │Container│      │
│  └─────────┘  └─────────┘  └─────────┘      │
│        │            │            │          │
│        └────────────┴────────────┘          │
│              Docker Network                 │
│                                             │
└─────────────────────────────────────────────┘
```

**Key Characteristics:**

- Designed for **non-production** environments (local dev, testing, staging)
- Runs all containers on a **single machine** (server / VM / laptop)
- Does **NOT** support creating a cluster across multiple machines
- Provides a framework for containers to communicate via shared networks
- Lets you define volumes, networks, and dependencies in a single YAML file

### 2. Container Orchestration Tools

**Purpose:** Running and managing containers at scale across **multiple machines**.

```
Container Orchestration - Multi-Machine Cluster
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Machine 1  │  │   Machine 2  │  │   Machine 3  │
│ ┌────┐┌────┐ │  │ ┌────┐┌────┐ │  │ ┌────┐┌────┐ │
│ │ App││Redis││  │ │ DB ││ App│ │  │ │ App││ DB │ │
│ └────┘└────┘ │  │ └────┘└────┘ │  │ └────┘└────┘ │
└──────────────┘  └──────────────┘  └──────────────┘
        │                │                 │
        └────────────────┴─────────────────┘
                  Cluster Network
```

**Common Orchestration Tools:**

```
Tool                │  Usage Level
────────────────────┼──────────────────────────
Kubernetes (K8s)    │  Industry standard, most widely used
Amazon ECS          │  AWS-native container orchestration
OpenShift           │  Enterprise Kubernetes (Red Hat)
Docker Swarm        │  Rarely used today
```

**Key Characteristics:**

- Designed for **production** environments
- Distributes containers across a **cluster of multiple machines**
- Provides auto-scaling, self-healing, load balancing, and rolling updates
- Handles service discovery — containers can find and talk to each other across machines

---

## Docker Compose vs Orchestration — Quick Comparison

```
┌──────────────────┬─────────────────────┬──────────────────────────┐
│                  │  Docker Compose     │  Orchestration (K8s etc) │
├──────────────────┼─────────────────────┼──────────────────────────┤
│ Runs on          │ Single machine      │ Cluster of machines      │
│ Use case         │ Dev / Testing       │ Production               │
│ Scaling          │ Limited             │ Auto-scaling             │
│ Self-healing     │ No                  │ Yes                      │
│ Load balancing   │ No                  │ Yes                      │
│ Complexity       │ Simple (YAML file)  │ Complex                  │
│ Networking       │ Single host network │ Cross-host networking    │
└──────────────────┴─────────────────────┴──────────────────────────┘
```

---