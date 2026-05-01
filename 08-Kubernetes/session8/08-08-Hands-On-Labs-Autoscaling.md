# Session 8-08: Autoscaling & Resource Management - Hands-On Labs

**Goal**: Learn to deploy and manage autoscaling in Kubernetes, from metrics collection to multi-layer scaling strategies.

---

## Lab 1: Install Metrics Server and Verify Monitoring

**Objective**: Deploy Metrics Server and test `kubectl top` command.

### Steps:

1. **Check if Metrics Server is installed**:
   ```bash
   kubectl get deployment metrics-server -n kube-system
   ```
   - If it exists, skip to step 3.
   - If not found, proceed to step 2.

2. **Install Metrics Server** (for self-managed clusters):
   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   ```

3. **Wait for Metrics Server to be ready**:
   ```bash
   kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=300s
   ```

4. **Verify metrics collection** (wait ~30 seconds for first data):
   ```bash
   kubectl top nodes
   kubectl top pods --all-namespaces
   ```


5. **Troubleshooting** (if no metrics appear):
   - Check Metrics Server logs: `kubectl logs -n kube-system deployment/metrics-server`
   - Common issues:
     - Kubelet API not accessible (firewall/certificate issues)
     - Metrics Server pod not running: `kubectl describe pod -n kube-system -l k8s-app=metrics-server`

---

## Lab 2: Create and Configure HPA

**Objective**: Set up Horizontal Pod Autoscaler targeting CPU utilization.

### Manifest: `app-with-hpa.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"

---
apiVersion: v1
kind: Service
metadata:
  name: hpa-demo
spec:
  selector:
    app: hpa-demo
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50  # Scale when average CPU > 50%
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

### Steps:

1. **Deploy the application**:
   ```bash
   kubectl apply -f app-with-hpa.yaml
   ```

2. **Verify HPA creation**:
   ```bash
   kubectl get hpa
   kubectl describe hpa hpa-demo
   ```

3. **View HPA status repeatedly**:
   ```bash
   kubectl get hpa hpa-demo --watch
   ```

4. **Examine HPA events**:
   ```bash
   kubectl describe hpa hpa-demo
   kubectl get events --sort-by='.lastTimestamp' | grep hpa-demo
   ```

---

## Lab 3: Generate Load and Watch HPA Scale

**Objective**: Create CPU load and observe pods scale up in real-time.

### Steps:

1. **Open three terminals**:

   **Terminal 1 - Watch HPA status**:
   ```bash
   kubectl get hpa hpa-demo --watch
   ```

   **Terminal 2 - Watch pod count**:
   ```bash
   kubectl get pods -l app=hpa-demo --watch
   ```

2. **Generate load** (Terminal 3):
   ```bash
   # Get the service IP or LoadBalancer endpoint
   SERVICE_IP=$(kubectl get svc hpa-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

   # Generate continuous HTTP requests (using kubectl run)
   kubectl run -it --rm load-generator --image=busybox /bin/sh

   # Inside the load-generator pod:
   while true; do wget -q -O- http://hpa-demo; done
   ```

3. **Observe scaling**:
   - Within 1-2 minutes, metrics should show increasing CPU usage
   - HPA should calculate desired replicas and scale up
   - Watch Terminal 2 for new pods appearing
   - Terminal 1 shows replica count increasing

4. **Monitor HPA decision calculation**:
   ```bash
   kubectl describe hpa hpa-demo
   ```
   Look for `Current Metrics` showing actual CPU utilization percentage.

---

## Further Reading

- [VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Cluster Autoscaler Documentation](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
