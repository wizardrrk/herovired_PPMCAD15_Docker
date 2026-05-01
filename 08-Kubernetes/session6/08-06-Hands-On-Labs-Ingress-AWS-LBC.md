# Session 8-06: Ingress with AWS Load Balancer Controller - Hands-On Labs

## Overview
In this session, you'll deploy AWS Load Balancer Controller, create sample applications, and implement path-based routing using Ingress. You'll verify that AWS creates an ALB for your Ingress and that traffic correctly routes to your applications.

**Prerequisites:**
- EKS cluster running (from Session 2)
- kubectl configured to access your cluster
- Helm 3+ installed
- AWS CLI configured with cluster access
- The Terraform from Session 2 already configured IRSA role for AWS LBC

---

## Pre-Req: 

Helm 3 Installation: https://helm.sh/docs/intro/install/

## Lab 1: Install AWS Load Balancer Controller

### Goal
Install AWS Load Balancer Controller via Helm using the IRSA service account.

### Step 1 - Create the IAM Policy

1. Go to **AWS Console → IAM → Policies → Create Policy**
2. Switch to **JSON** view and paste the official policy from:
   ```
   https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
   ```
3. Name it `AWSLoadBalancerControllerIAMPolicy`
4. Review and create the policy

---

### Step 2 - Create the OIDC Identity Provider for Your EKS Cluster

1. Go to **AWS Console → IAM → Identity Providers → Add Provider**
2. Select **OpenID Connect**
3. Get your OIDC provider URL from:
   **EKS Console → Your Cluster → Details → OpenID Connect provider URL**
4. Set **Audience** to `sts.amazonaws.com`
5. Verify thumbprint and create the provider

---

### Step 3 - Create the IAM Role

1. Go to **AWS Console → IAM → Roles → Create Role**
2. Select **Web Identity** as the trusted entity type
3. Select the OIDC Provider you just created
4. Set **Audience** to `sts.amazonaws.com`
5. In the trust relationship, update the `StringEquals` condition as follows - replace `${OIDC_PROVIDER}` with the OIDC URL copied from the EKS details page (without the `https://` prefix):

   ```json
   "StringEquals": {
     "${OIDC_PROVIDER}:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller",
     "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
   }
   ```

6. Attach the `AWSLoadBalancerControllerIAMPolicy` created in Step 1
7. Name the role - e.g., `AWSLoadBalancerControllerRole`

---

### Step 4 - Add the EKS Helm Repository

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

**Expected output:**
```
"eks" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
Successfully got an update from the "eks" chart repository
```

---

### Step 5 - Install the AWS Load Balancer Controller

Connect to your EKS cluster
```bash
aws eks update-kubeconfig --region <aws-region> --name <your-cluster-name>
```

Make sure to update the command with actual values from your EKS cluster

```bash
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set serviceAccount.annotations.eks\.amazonaws\.com/role-arn="<arn-of-iam-role-from-step-3>" \
  --set region=<aws-region> \
  --set vpcId="<vpc-id-of-eks-cluster>" \
  --set enableWaf="false" \
  --set enableWafv2="false"
```

working command for mac:

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=EKS-B15 \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set-string serviceAccount.annotations.eks\\.amazonaws\\.com\\/role-arn=arn:aws:iam::233245302554:role/AWSLBCROLE \
  --set region=ap-south-1 \
  --set vpcId=vpc-067db2a4438ac151a

---

### Step 6 - Verify the Installation

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get pods -n kube-system | grep aws-load-balancer-controller
```

**Expected output:**
```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           2m

aws-load-balancer-controller-789abc123-def45   1/1   Running   0   2m
aws-load-balancer-controller-789abc123-ghi67   1/1   Running   0   2m
```

Check logs to confirm the controller is healthy and watching for Ingress resources:
```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

---

## Lab 2: Deploy Two Sample Applications

### Goal
Create two simple web applications with custom HTML pages and ClusterIP services so we can route to them via Ingress.

### 2.1 Deploy App 1

Create file `app1-deployment.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app1-config
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body>
        <h1>Hello, World, I am serving from app1!</h1>
    </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: nginx
        volumeMounts:
        - name: app1-volume
          mountPath: /usr/share/nginx/html/app1
        ports:
        - containerPort: 80
      volumes:
      - name: app1-volume
        configMap:
          name: app1-config
---
apiVersion: v1
kind: Service
metadata:
  name: app1-service
spec:
  type: ClusterIP
  selector:
    app: app1
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Deploy:
```bash
kubectl apply -f app1-deployment.yaml
```

Verify:
```bash
kubectl get pods -l app=app1
kubectl get svc app1-service
```

### 2.2 Deploy App 2

Create file `app2-deployment.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app2-config
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body>
        <h1>Hello, World, I am serving from app2!</h1>
    </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: nginx
        volumeMounts:
        - name: app2-volume
          mountPath: /usr/share/nginx/html/app2
        ports:
        - containerPort: 80
      volumes:
      - name: app2-volume
        configMap:
          name: app2-config
---
apiVersion: v1
kind: Service
metadata:
  name: app2-service
spec:
  type: ClusterIP
  selector:
    app: app2
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Deploy:
```bash
kubectl apply -f app2-deployment.yaml
```

Verify:
```bash
kubectl get pods -l app=app2
kubectl get svc app2-service
```

### 2.3 Test Internal Connectivity

Verify services are reachable from within the cluster:
```bash
kubectl run test-pod --image=curlimages/curl:latest --rm -it --restart=Never -- sh -c "curl http://app1-service/app1/"
kubectl run test-pod --image=curlimages/curl:latest --rm -it --restart=Never -- sh -c "curl http://app2-service/app2/"
```

Both should return the custom HTML page defined in each ConfigMap.

---

## Lab 3: Create Path-Based Ingress

### Goal
Create an Ingress resource that routes traffic to different services based on URL path using the AWS Load Balancer Controller.

### 3.1 Create Ingress Manifest

Create file `ingress-alb.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress-alb
  namespace: default
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

### 3.2 Deploy Ingress

```bash
kubectl apply -f ingress-alb.yaml
```

Check status:
```bash
kubectl get ingress
```

**Initial output (ALB creation in progress):**
```
NAME               CLASS   HOSTS   ADDRESS   PORTS   AGE
demo-ingress-alb   alb     *                 80      10s
```

Wait for the ADDRESS field to populate - this takes 2–3 minutes while the ALB is being provisioned:
```bash
kubectl get ingress -w
```

**Once ready:**
```
NAME               CLASS   HOSTS   ADDRESS                                                   PORTS   AGE
demo-ingress-alb   alb     *       k8s-default-demoingr-xyz123.us-east-1.elb.amazonaws.com   80      3m
```

### 3.3 Retrieve ALB DNS Name

```bash
ALB_DNS=$(kubectl get ingress demo-ingress-alb -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo $ALB_DNS
```

Save this DNS name - you'll use it to test routing.

---

## Lab 4: Verify ALB Creation in AWS Console

### Goal
Confirm AWS LBC created an ALB for your Ingress and verify target group configuration.

### 4.1 Check ALB in AWS Console

1. Go to **EC2 Dashboard → Load Balancers**
2. Look for an ALB with a name similar to `k8s-default-demoingr-xyz123`
3. Confirm the DNS name matches what `kubectl` showed

### 4.2 Inspect Target Groups

1. Click the ALB name
2. Go to the **Target Groups** tab
3. You should see 2 target groups:
   - One for `app1-service` (path: `/app1`)
   - One for `app2-service` (path: `/app2`)
4. Click into each target group
5. Under the **Targets** tab, verify:
   - Targets are **Pod IPs** (not EC2 IPs - because we used `target-type: ip`)
   - Each target group shows 2 targets (matching your 2 replicas)
   - Status is **Healthy**

### 4.3 Inspect Listener Rules

1. Go back to the ALB
2. Click the **Listeners** tab → click **HTTP:80**
3. You should see routing rules:
   - Path: `/app1` → Target Group: `app1-service`
   - Path: `/app2` → Target Group: `app2-service`
4. Make sure that the Security Group associated with this ALB has inbound rule allowing Port 80 from everywhere

---

## Lab 5: Test Traffic Routing

### Goal
Verify that traffic correctly routes to the right service based on path.

### 5.1 Test /app1 Path

```bash
ALB_DNS=$(kubectl get ingress demo-ingress-alb -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$ALB_DNS/app1/
```

**Expected output:**
```html
<!DOCTYPE html>
<html>
<body>
    <h1>Hello, World, I am serving from app1!</h1>
</body>
</html>
```

### 5.2 Test /app2 Path

```bash
curl http://$ALB_DNS/app2/
```

**Expected output:**
```html
<!DOCTYPE html>
<html>
<body>
    <h1>Hello, World, I am serving from app2!</h1>
</body>
</html>
```

### 5.3 Test Non-Existent Path

```bash
curl http://$ALB_DNS/admin
```

**Expected output:**
HTTP 404 - no routing rule is configured for this path.

---

## Lab 6: Host-Based Routing (Optional/Self-Practice)

### Goal
If you have a domain, implement host-based routing.

### 6.1 Create Host-Based Ingress

Create file `ingress-host-based.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-host-ingress
  namespace: default
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
spec:
  ingressClassName: alb
  rules:
  - host: app1.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

### 6.2 Deploy and Test

```bash
kubectl apply -f ingress-host-based.yaml

# Get ALB DNS
ALB_DNS=$(kubectl get ingress demo-host-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test with Host header (if you don't own the domain)
curl -H "Host: app1.yourdomain.com" http://$ALB_DNS/
curl -H "Host: app2.yourdomain.com" http://$ALB_DNS/
```

If you own the domain, update your DNS records:
```
app1.yourdomain.com    CNAME    <ALB_DNS>
app2.yourdomain.com    CNAME    <ALB_DNS>
```

Then test directly:
```bash
curl http://app1.yourdomain.com/
curl http://app2.yourdomain.com/
```

---

## Lab 7: Clean Up

### Goal
Remove all resources created in this session.

### 7.1 Delete Ingress Resources

```bash
kubectl delete ingress demo-ingress-alb
# If you did Lab 6:
kubectl delete ingress demo-host-ingress
```

Verify the ALB is deleted in AWS Console (takes ~1–2 minutes):
```bash
# Check AWS Console → EC2 → Load Balancers
# The ALB should disappear
```

### 7.2 Delete Applications

```bash
kubectl delete -f app1-deployment.yaml
kubectl delete -f app2-deployment.yaml
```

This deletes the Deployments, Services, and ConfigMaps for both apps.

### 7.3 (Optional) Uninstall AWS LBC

```bash
helm uninstall aws-load-balancer-controller -n kube-system
kubectl delete serviceaccount aws-load-balancer-controller -n kube-system
```

> Keep AWS LBC installed if you'll use it for future sessions.

### 7.4 Verify Cleanup

```bash
kubectl get ingress
kubectl get deployment
kubectl get svc
kubectl get configmap
```

Should show no resources (except the default `kubernetes` service and `kube-root-ca.crt` configmap).

---

## Troubleshooting

### Issue: Ingress ADDRESS stays `<pending>`

**Cause:** AWS LBC isn't running or has permission issues.

**Fix:**
```bash
# Check AWS LBC pods
kubectl get pods -n kube-system | grep aws-load-balancer

# Check logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Check IRSA annotation on the service account
kubectl describe sa aws-load-balancer-controller -n kube-system
```

### Issue: ALB targets show "Unhealthy"

**Cause:** Pods aren't responding to health checks on the expected path.

**Fix:**
```bash
# Check pod logs
kubectl logs -l app=app1

# Check pod is running
kubectl get pods -l app=app1

# Test the path from inside the pod
kubectl exec -it <pod-name> -- curl localhost:80/app1/
```

> Note: The ALB health check will probe `/app1` on the pod. Make sure the ConfigMap content is mounted correctly and the path resolves.

### Issue: curl returns 404 or Connection Refused

**Cause:** Target group not yet healthy, or the path in your curl doesn't match the Ingress rule.

**Fix:**
1. Wait 30–60 seconds for health checks to pass after ALB creation
2. Verify the path in your `curl` command matches exactly what's in the Ingress (e.g., `/app1/` vs `/app1`)
3. Check ALB listener rules in the AWS Console