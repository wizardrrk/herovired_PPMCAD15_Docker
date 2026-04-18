# Kubernetes Troubleshooting — Interview Q&A

> Real scenario-based questions and how to approach them step by step.

---

## Q1: Three WordPress Pods Are Not Running Properly

**Scenario:** Three WordPress pods are deployed but not running correctly. Check the resources and fix it. **Do not increase or decrease the resource limits.**

---

### Troubleshooting Steps

**Step 1 — Check the Deployment**

```bash
kubectl get deploy -n <namespace>
```

**Step 2 — Describe the Deployment**

```bash
kubectl describe deploy <deployment-name> -n <namespace>
```

**What we found:** The resource **requests were greater than limits**, which is invalid. Kubernetes cannot schedule a pod where the minimum guaranteed resources (requests) exceed the maximum allowed resources (limits).

**The Fix:** Tune the **requests** so they are always **less than or equal to** limits — without changing the limit values.

---

### Understanding Requests vs Limits

Take this example from a `deploy.yaml`:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 512Mi    # Problem! Request (512Mi) > Limit (256Mi)
  limits:
    cpu: 500m
    memory: 256Mi
```

| Field        | What It Means                                                                 |
|--------------|-------------------------------------------------------------------------------|
| **Requests** | The **minimum** resources Kubernetes **guarantees** to the pod. The scheduler uses this to find a node with enough capacity. |
| **Limits**   | The **maximum** resources a pod is **allowed to use**. If it exceeds this, it gets throttled (CPU) or killed/OOMKilled (memory). |

**Golden Rule:** `requests` ≤ `limits` — always.

In this case, memory request (512Mi) is **greater** than memory limit (256Mi), which is invalid. Fix it by lowering the memory request to something ≤ 256Mi (e.g., 128Mi or 256Mi).

---

**Step 3 — Check Pod Status**

```bash
kubectl get pods -n <namespace>

# or check across all namespaces:
kubectl get pods -A    # short form of --all-namespaces
```

**Step 4 — Describe the Pod for Events & Errors**

```bash
kubectl describe pod <pod-name> -n <namespace>
```

**Step 5 — Check Persistent Volumes (if applicable)**

If the pod uses a PersistentVolume, verify the PV and PVC are bound and healthy:

```bash
kubectl get pv
kubectl get pvc -n <namespace>
```

**Step 6 — Check Pod Logs**

```bash
kubectl logs <pod-name> -n <namespace>
```

---


## Q2: App Is Throwing HTTP Errors (502, 503, 504, 404)

**Scenario:** Your application running on Kubernetes starts giving errors like 502, 504, 503, 404. How do you troubleshoot?

---

### Approach: Trace the Request Path Top-Down

First, understand the full architecture — how traffic flows from the user to your app:

```
User → Route53 (DNS) → ALB (Load Balancer) → EKS Cluster → Service → Pods
```

Troubleshoot **layer by layer**, from the outside in.

---

### Layer 1 — DNS (Route53)

- Is the DNS record pointing to the correct ALB?
- Is the hosted zone configured properly?
- Has the record propagated? (`dig` or `nslookup` to verify)

### Layer 2 — Load Balancer (ALB)

- Check ALB **metrics** in CloudWatch (5xx errors, healthy host count, target response time).
- Are the **target groups** healthy?
- Is the **listener** and **routing rules** configured correctly?
- Are the **security groups** allowing traffic on the correct ports?

### Layer 3 — Kubernetes Level

**Check pod status:**

```bash
kubectl get pods -n <namespace>
```

**Describe the pod for events and state:**

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### Layer 4 — Application Level

**Check application logs:**

```bash
kubectl logs <pod-name> -n <namespace>
```

**Exec into the pod to investigate live:**

```bash
kubectl exec -it <pod-name> -n <namespace> -- bash
```

**Inside the pod, check:**

1. **Environment variables** — Are all required env vars set correctly?
   ```bash
   env | grep -i <keyword>
   ```
2. **Additional log files** — Some apps write logs to files (e.g., `/var/log/app/`), not just stdout.
3. **Database connectivity** — Can the pod reach the database?
   ```bash
   # Example for MySQL
   mysql -h <db-host> -u <user> -p

   # Example for PostgreSQL
   psql -h <db-host> -U <user> -d <dbname>

   # Generic connectivity check
   curl -v telnet://<db-host>:<port>
   ```

---

### Quick Cheat Sheet: What Each Error Usually Means

| Error | Common Cause                                                     |
|-------|------------------------------------------------------------------|
| **404** | Wrong path, missing ingress rule, or app route not configured. |
| **502** | Pod is crashing or not responding; ALB can't get a valid response. |
| **503** | No healthy targets; pods are down or service is misconfigured.  |
| **504** | Timeout; app is too slow to respond, or network/security group is blocking. |

---

*Approach: Always trace the traffic path from DNS → LB → Kubernetes → App. Fix the first broken layer you find.*