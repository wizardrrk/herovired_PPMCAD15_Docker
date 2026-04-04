# Session 8-04: Core Workloads - Hands-On Labs

## Prerequisites
- Kubernetes cluster running (kind, minikube, or managed cluster)
- `kubectl` configured to access cluster
- Text editor for YAML files
- Previous sessions completed (pods, basic kubectl commands)

---

## Lab 1: Create a ReplicaSet & Observe Self-Healing

### Objective
Understand how ReplicaSets maintain desired pod count through label selection.

### Step-by-Step

1. Create a ReplicaSet manifest file `lab1-replicaset.yaml`:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.24-alpine
        ports:
        - containerPort: 80
```

2. Apply the manifest:
```bash
kubectl apply -f lab1-replicaset.yaml
```

**Expected output:**
```
replicaset.apps/web-rs created
```

3. Verify pods were created:
```bash
kubectl get pods -L tier
```

**Expected output:**
```
NAME          READY   STATUS    RESTARTS   AGE     TIER
web-rs-abc12  1/1     Running   0          10s     frontend
web-rs-def45  1/1     Running   0          10s     frontend
web-rs-ghi78  1/1     Running   0          10s     frontend
```

4. Check ReplicaSet status:
```bash
kubectl get replicaset web-rs
```

**Expected output:**
```
NAME     DESIRED   CURRENT   READY   AGE
web-rs   3         3         3       20s
```

### Self-Healing Test

5. Delete one pod:
```bash
kubectl delete pod web-rs-abc12
```

6. Immediately check pods:
```bash
kubectl get pods -L tier
```

**Expected behavior:** ReplicaSet immediately creates a new pod to maintain 3 replicas. You should see a new pod (e.g., `web-rs-xyz99`) in Creating/Running state.

7. Watch reconciliation in real-time:
```bash
kubectl get pods -L tier --watch
```

Press Ctrl+C to stop watching.

### Success Criteria
- [ ] 3 pods running with `tier=frontend` label
- [ ] ReplicaSet shows DESIRED=3, CURRENT=3, READY=3
- [ ] After deleting a pod, new pod created within seconds
- [ ] Pod names are automatically generated (hash suffix)

---

## Lab 2: Create a Deployment (First Steps)

### Objective
Deploy an application using Deployment instead of ReplicaSet directly.

### Step-by-Step

1. Create deployment manifest `lab2-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.24-alpine
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 5
```

2. Apply the deployment:
```bash
kubectl apply -f lab2-deployment.yaml
```

**Expected output:**
```
deployment.apps/web-app created
```

3. Check deployment status:
```bash
kubectl get deployment web-app
```

**Expected output:**
```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
web-app   3/3     3            3           30s
```

4. List underlying ReplicaSets:
```bash
kubectl get replicaset -l app=web
```

**Expected output:**
```
NAME                 DESIRED   CURRENT   READY   AGE
web-app-xyz1a2b3c   3         3         3       30s
```

5. List pods:
```bash
kubectl get pods -l app=web
```

**Expected output:**
```
NAME                      READY   STATUS    RESTARTS   AGE
web-app-xyz1a2b3c-abc12   1/1     Running   0          30s
web-app-xyz1a2b3c-def45   1/1     Running   0          30s
web-app-xyz1a2b3c-ghi78   1/1     Running   0          30s
```

### Success Criteria
- [ ] Deployment shows READY=3/3
- [ ] Exactly 1 ReplicaSet exists (the active one)
- [ ] All 3 pods show Running status
- [ ] Pod names include both ReplicaSet hash and pod hash

---

## Lab 3: Rolling Updates (Image Update)

### Objective
Perform a rolling update and observe how Kubernetes gradually replaces pods.

### Setup

Use the deployment from Lab 2 (if not running, create it again):
```bash
kubectl apply -f lab2-deployment.yaml
```

### Step-by-Step

1. Watch the rollout in a separate terminal (leave running):
```bash
kubectl rollout status deployment/web-app --watch
```

2. In another terminal, update the image:
```bash
kubectl set image deployment/web-app nginx=nginx:1.25-alpine
```

**Expected output:**
```
deployment.apps/web-app image updated
```

3. Watch the status terminal — you should see:
```
Waiting for deployment "web-app" to rollout.
Waiting for deployment "web-app" to rollout.
deployment "web-app" successfully rolled out
```

4. Check ReplicaSets during update (quickly):
```bash
kubectl get replicaset -l app=web -o wide
```

**Expected output (if caught mid-update):**
```
NAME                   DESIRED   CURRENT   READY   AGE
web-app-old-hash       1         1         1       5m      (old RS scaling down)
web-app-new-hash       2         2         2       20s     (new RS scaling up)
```

5. After update completes, verify new image:
```bash
kubectl get pods -l app=web -o jsonpath='{.items[0].spec.containers[0].image}'
```

**Expected output:**
```
nginx:1.25-alpine
```

6. Check rollout history:
```bash
kubectl rollout history deployment/web-app
```

**Expected output:**
```
deployment.apps/web-app
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

### Success Criteria
- [ ] Rollout status shows "successfully rolled out"
- [ ] All pods now run `nginx:1.25-alpine`
- [ ] History shows 2 revisions
- [ ] Old ReplicaSet scaled to 0, new ReplicaSet scaled to 3
- [ ] No pods in `Terminating` or `CrashLoopBackOff` state

---

## Lab 4: Rollback a Deployment

### Objective
Demonstrate reverting to a previous deployment version.

### Setup

Ensure you have the deployment with 2 revisions from Lab 3. If not, run Labs 2-3 first.

### Step-by-Step

1. View current deployment:
```bash
kubectl get deployment web-app -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Expected output:**
```
nginx:1.25-alpine
```

2. View revision history:
```bash
kubectl rollout history deployment/web-app
```

**Expected output:**
```
deployment.apps/web-app
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

3. Get details of revision 1:
```bash
kubectl rollout history deployment/web-app --revision=1
```

**Expected output:**
```
deployment.apps/web-app with revision #1
Pod Template:
  Labels:	app=web
  Containers:
   nginx:
    Image:      nginx:1.24-alpine
```

4. Rollback to revision 1:
```bash
kubectl rollout undo deployment/web-app --to-revision=1
```

**Expected output:**
```
deployment.apps/web-app rolled back
```

5. Watch rollout:
```bash
kubectl rollout status deployment/web-app
```

6. Verify the image reverted:
```bash
kubectl get pods -l app=web -o jsonpath='{.items[0].spec.containers[0].image}'
```

**Expected output:**
```
nginx:1.24-alpine
```

7. Check revision history again:
```bash
kubectl rollout history deployment/web-app
```

**Expected output:**
```
deployment.apps/web-app
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

Note: Old revision 1 is now revision 3 (history rotates, keeping last 10 by default).

### Success Criteria
- [ ] Rollback command completes successfully
- [ ] All pods revert to `nginx:1.24-alpine`
- [ ] Rollout status shows "successfully rolled out"
- [ ] Revision history shows new entry

---

## Lab 5: Scaling Replicas

### Objective
Practice manual scaling up and down using kubectl.

### Step-by-Step

1. Current state:
```bash
kubectl get deployment web-app
```

**Expected output:**
```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
web-app   3/3     3            3           10m
```

2. Scale up to 5 replicas:
```bash
kubectl scale deployment/web-app --replicas=5
```

**Expected output:**
```
deployment.apps/web-app scaled
```

3. Watch pods being created:
```bash
kubectl get pods -l app=web --watch
```

You should see 2 new pods in Creating state, then Running. Press Ctrl+C when done.

4. Verify scaling:
```bash
kubectl get deployment web-app
```

**Expected output:**
```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
web-app   5/5     5            5           10m
```

5. Check ReplicaSet:
```bash
kubectl get replicaset -l app=web
```

Should show DESIRED=5, CURRENT=5, READY=5 for the active ReplicaSet.

6. Now scale down to 2 replicas:
```bash
kubectl scale deployment/web-app --replicas=2
```

7. Watch pods being terminated:
```bash
kubectl get pods -l app=web --watch
```

You should see 3 pods in Terminating state. Press Ctrl+C when done.

8. Final verification:
```bash
kubectl get deployment web-app
kubectl get pods -l app=web
```

Should show READY=2/2 and only 2 pods.

### Success Criteria
- [ ] Scale up to 5 replicas succeeds
- [ ] Scale down to 2 replicas succeeds
- [ ] Deployment READY field matches replica count
- [ ] Pod distribution changes smoothly (no hanging states)

---

## Lab 6: Deployment Strategies (RollingUpdate vs Recreate)

### Objective
Compare the two main deployment strategies and observe their behavior.

### Part A: RollingUpdate (Default)

1. Create deployment with explicit RollingUpdate strategy `lab6a-rolling.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-rolling
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      strategy: rolling
  template:
    metadata:
      labels:
        strategy: rolling
    spec:
      containers:
      - name: nginx
        image: nginx:1.24-alpine
```

2. Apply and verify:
```bash
kubectl apply -f lab6a-rolling.yaml
kubectl get deployment web-rolling
```

3. Open separate terminal to watch:
```bash
kubectl get pods -l strategy=rolling --watch
```

4. Update image (in another terminal):
```bash
kubectl set image deployment/web-rolling nginx=nginx:1.25-alpine
```

5. Observe the rolling update:
- Old and new pods coexist temporarily
- New pods start before old pods terminate
- At any point, at least 2 pods available (3 - maxUnavailable=1)
- Maximum 4 pods exist (3 + maxSurge=1)

Press Ctrl+C to stop watching.

### Part B: Recreate Strategy

1. Create deployment with Recreate strategy `lab6b-recreate.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-recreate
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      strategy: recreate
  template:
    metadata:
      labels:
        strategy: recreate
    spec:
      containers:
      - name: nginx
        image: nginx:1.24-alpine
```

2. Apply and verify:
```bash
kubectl apply -f lab6b-recreate.yaml
kubectl get deployment web-recreate
```

3. Open separate terminal to watch:
```bash
kubectl get pods -l strategy=recreate --watch
```

4. Update image:
```bash
kubectl set image deployment/web-recreate nginx=nginx:1.25-alpine
```

5. Observe the Recreate behavior:
- All old pods terminate first (go to Terminating state)
- Brief moment with 0 pods running (downtime!)
- After all old pods gone, new pods created
- New pods start from scratch

Press Ctrl+C to stop watching.

### Comparison Table

Fill in during observation:

| Aspect | RollingUpdate | Recreate |
|--------|---------------|----------|
| Downtime | None | Brief |
| Simultaneous versions | Yes | No |
| Max pod count | 3 + maxSurge | 3 |
| Rollback speed | Slow (must roll up new) | Fast (keep new RS ready) |
| Use case | Production | Testing/Dev |

### Success Criteria
- [ ] RollingUpdate shows overlapping old/new pods
- [ ] Recreate shows all pods deleted before new ones created
- [ ] RollingUpdate achieves zero downtime
- [ ] Both strategies complete successfully

---

## Lab 7: Resource Requests & Limits

### Objective
Configure pod resource constraints and observe scheduling behavior.

### Part A: Deployment with Resource Requests

1. Create deployment with requests `lab7a-requests.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-requests
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-requests
  template:
    metadata:
      labels:
        app: web-requests
    spec:
      containers:
      - name: nginx
        image: nginx:1.24-alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

2. Apply deployment:
```bash
kubectl apply -f lab7a-requests.yaml
```

3. Check pod QoS class:
```bash
kubectl get pods -l app=web-requests -o jsonpath='{.items[0].status.qosClass}'
```

**Expected output:**
```
Burstable
```

(Requests < Limits, so Burstable QoS)

4. Verify resource allocation:
```bash
kubectl describe pod -l app=web-requests | grep -A 5 "Requests"
```

**Expected output:**
```
Requests:
  cpu:      100m
  memory:   128Mi
Limits:
  cpu:      500m
  memory:   256Mi
```

### Part B: Guaranteed QoS (requests == limits)

1. Create deployment with guaranteed resources `lab7b-guaranteed.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-guaranteed
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-guaranteed
  template:
    metadata:
      labels:
        app: web-guaranteed
    spec:
      containers:
      - name: nginx
        image: nginx:1.24-alpine
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 250m
            memory: 256Mi
```

2. Apply deployment:
```bash
kubectl apply -f lab7b-guaranteed.yaml
```

3. Check QoS class:
```bash
kubectl get pods -l app=web-guaranteed -o jsonpath='{.items[0].status.qosClass}'
```

**Expected output:**
```
Guaranteed
```

### Part C: Node Resource Status

1. View node resource allocation:
```bash
kubectl describe node <node-name>
```

Look for:
```
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                900m          2100m
  memory             512Mi         1024Mi
```

2. Check available resources:
```bash
kubectl top nodes
kubectl top pods
```

(Note: top commands require metrics-server, may not be available in all clusters)

### Part D: BestEffort QoS (no requests/limits)

1. Create deployment without resources `lab7c-besteffort.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-besteffort
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-besteffort
  template:
    metadata:
      labels:
        app: web-besteffort
    spec:
      containers:
      - name: nginx
        image: nginx:1.24-alpine
```

2. Apply:
```bash
kubectl apply -f lab7c-besteffort.yaml
```

3. Check QoS:
```bash
kubectl get pods -l app=web-besteffort -o jsonpath='{.items[0].status.qosClass}'
```

**Expected output:**
```
BestEffort
```

### Success Criteria
- [ ] Burstable pod shows requests < limits
- [ ] Guaranteed pod shows requests == limits
- [ ] BestEffort pod shows no requests/limits
- [ ] Node describes shows allocated resources
- [ ] QoS class correctly assigned to each pod
- [ ] All pods successfully scheduled and running

---