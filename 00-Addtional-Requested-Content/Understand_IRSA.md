# IRSA - IAM Roles for Service Accounts

> Guide to understanding IRSA in AWS EKS.

---

## What Problem Does IRSA Solve?

Imagine your app is running inside a Kubernetes pod on AWS EKS, and it needs to talk to an AWS service like S3 or DynamoDB.

**Without IRSA**, you'd have to do one of these (all bad):

- Hardcode AWS access keys inside your app - **huge security risk**.
- Attach a broad IAM role to the entire EC2 node - **every pod on that node gets the same permissions**, even pods that don't need them.

**With IRSA**, each pod gets **only the exact permissions it needs** - nothing more, nothing less.

---

## What is IRSA?

**IRSA = IAM Roles for Service Accounts**

It's a way to link:

- A **Kubernetes Service Account** (identity inside your cluster)
- To an **AWS IAM Role** (permissions inside AWS)

So when your pod runs using that Service Account, it **automatically** gets temporary AWS credentials scoped to that specific IAM Role.

---

## The Key Players

| Component              | What It Does                                                |
|------------------------|-------------------------------------------------------------|
| **Kubernetes Service Account** | An identity assigned to a pod inside the cluster.     |
| **AWS IAM Role**       | Defines what AWS resources can be accessed (S3, DynamoDB…). |
| **OIDC Provider**      | The "trust bridge" that lets AWS verify tokens from your EKS cluster. |
| **STS (Security Token Service)** | Issues temporary credentials to the pod.         |

---

## How It Works (Step by Step)

```
Pod starts up
    │
    ▼
Pod has a Kubernetes Service Account attached
    │
    ▼
EKS injects a JWT token into the pod (automatically)
    │
    ▼
Pod calls AWS STS: "Hey, here's my token, give me credentials"
    │
    ▼
STS checks with the OIDC Provider: "Is this token legit?"
    │
    ▼
OIDC says: "Yes, this token is from EKS cluster X, service account Y"
    │
    ▼
STS issues temporary AWS credentials to the pod
    │
    ▼
Pod uses those credentials to access AWS services (S3, DynamoDB, etc.)
```

---

## How to Set It Up (High-Level Steps)

### Step 1: Enable OIDC Provider for Your EKS Cluster

1. Go to **AWS Console → IAM → Identity Providers → Add Provider**
2. Select **OpenID Connect**
3. Get your OIDC provider URL from:
   **EKS Console → Your Cluster → Details → OpenID Connect provider URL**
4. Set **Audience** to `sts.amazonaws.com`
5. Verify thumbprint and create the provider

This creates the "trust bridge" between your cluster and AWS IAM.

### Step 2: Create an IAM Role with a Trust Policy

The trust policy says: *"Only this specific Service Account from this specific cluster can assume this role."*

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID:sub": "system:serviceaccount:NAMESPACE:SERVICE_ACCOUNT_NAME",
          "oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID:sub": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

### Step 3: Attach Permissions to the IAM Role

Attach whatever AWS policy your app needs (e.g., `AmazonS3ReadOnlyAccess`).

### Step 4: Annotate the Kubernetes Service Account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/my-irsa-role
```

### Step 5: Use the Service Account in Your Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app-sa   # ← This is the magic line
  containers:
    - name: app
      image: my-app-image
```

That's it. The pod will automatically get AWS credentials.

---

## Why IRSA is Great

| Benefit               | Explanation                                         |
|------------------------|-----------------------------------------------------|
| **Least Privilege**    | Each pod gets only the permissions it needs.         |
| **No Hardcoded Keys**  | No access keys sitting in your code or env vars.     |
| **Temporary Credentials** | Credentials auto-expire and auto-rotate.          |
| **Pod-Level Isolation**| Different pods on the same node can have different roles. |
| **AWS Native**         | Works seamlessly with all AWS SDKs - zero code changes. |

---

## Quick Mental Model

```
Kubernetes World          Trust Bridge          AWS World
┌──────────────────┐      ┌──────────┐      ┌──────────────┐
│  Service Account │────> │   OIDC   │─────>│   IAM Role   │
│  (who you are)   │      │ Provider │      │ (what you    │
│                  │      │ (verify) │      │  can do)     │
└──────────────────┘      └──────────┘      └──────────────┘
```

---

*That's IRSA in a nutshell. Your pods get AWS powers - safely, temporarily, and with zero hardcoded secrets.*