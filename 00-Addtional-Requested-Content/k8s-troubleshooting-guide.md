
## The Kubernetes Troubleshooting Playbook

### Introduction: A Systematic Approach

When faced with a Kubernetes issue, avoid random guessing. Follow a systematic approach, starting from the application and working your way out through the layers of abstraction.

**The Troubleshooting Funnel:** **Pod -> Service/Ingress -> Network Policy -> Node -> Control Plane -> External Integrations (Cloud, Storage, etc.)**

The first and most important commands for any issue are:
*   `kubectl get <resource> <name>`: To see the object's desired state.
*   `kubectl describe <resource> <name>`: To see the object's status, configuration, and, most importantly, its **Events**. The `Events` section is your best friend.
*   `kubectl logs <pod-name>`: To see what the application itself is doing.

---

### Category 1: Pod & Application Lifecycle Issues (The "Day 1" Problems)

These are the most frequent issues, often occurring right after a deployment.

#### Use Case 1: The "CrashLoopBackOff" Nightmare
*   **Symptom:** `kubectl get pods` shows a pod's status as `CrashLoopBackOff`. The pod starts, crashes, and Kubernetes restarts it in a continuous loop.
*   **Environment:** Both Self-Managed & EKS.
*   **Troubleshooting:**
    1.  **Describe the Pod:** This reveals the container's exit code.
        ```bash
        kubectl describe pod <pod-name>
        ```
        Look at the `Last State` section for the `Exit Code`:
        *   `Exit Code: 1`: Generic application error. **The logs are your answer.**
        *   `Exit Code: 137`: `OOMKilled`. The container exceeded its memory limit.
        *   `Exit Code: 0`: The application finished successfully. This is an error for a long-running service.
    2.  **Check Previous Logs:** The pod is crashing, so you need the logs from the *last* terminated instance.
        ```bash
        kubectl logs <pod-name> --previous
        ```
*   **Common Causes & Solutions:**
    *   **Application Error:** A bug, a failed database connection, or a missing config file. **Solution:** Fix the application code or its configuration based on the log output.
    *   **Misconfigured Probes:** A failing `livenessProbe` will cause Kubernetes to kill the pod. **Solution:** Check the probe's endpoint, port, and timings (`initialDelaySeconds`) in the pod spec.
    *   **Out of Memory:** **Solution:** Increase the `resources.limits.memory` in your Deployment YAML. Also, profile your application to understand its memory needs.
    *   **Process Exits:** The container's startup command is not a long-running process. **Solution:** For services, ensure your `CMD` or `ENTRYPOINT` starts a web server or other persistent process. For one-off tasks, use a Kubernetes `Job` instead of a `Deployment`.

---

#### Use Case 2: The "ImagePullBackOff" or "ErrImagePull" Blocker
*   **Symptom:** A pod is stuck in `Pending`, and its status is `ImagePullBackOff` or `ErrImagePull`.
*   **Environment:** Both Self-Managed & EKS.
*   **Troubleshooting:**
    1.  **Describe the Pod:** The `Events` section will show the exact reason for the failure.
        ```bash
        kubectl describe pod <pod-name>
        ```
*   **Common Causes & Solutions:**
    *   **Typo in Image Name/Tag:** The `image` field in the YAML is incorrect. **Solution:** Correct the typo and re-apply the manifest.
    *   **Authentication to Private Registry Failed:** This is the most common cause for non-public images. You are pulling from a private registry (Docker Hub, ECR, etc.) without providing credentials.
        *   **Event Message:** `UNAUTHORIZED: authentication required`.
        *   **Solution:**
            1.  Create a Docker registry secret:
                ```bash
                kubectl create secret docker-registry my-registry-secret \
                  --docker-server=<your-registry-server> \
                  --docker-username=<your-username> \
                  --docker-password=<your-password> \
                  --docker-email=<your-email>
                ```
            2.  Add this secret to your Deployment's Pod spec or link it to the pod's `serviceAccount`.
                ```yaml
                spec:
                  containers:
                  - name: my-private-app
                    image: myprivate.registry.io/my-app:1.0
                  imagePullSecrets:
                  - name: my-registry-secret
                ```
    *   **EKS Specific (ECR):** The worker node's IAM Instance Profile role is missing permissions to pull from ECR. **Solution:** Ensure the `AmazonEC2ContainerRegistryReadOnly` IAM policy (or equivalent permissions) is attached to the node role.

---

#### Use Case 3: Pod Stuck in `Pending` (Resource Exhaustion)
*   **Symptom:** A new pod is stuck with a `Pending` status and never starts.
*   **Environment:** Both Self-Managed & EKS.
*   **Troubleshooting:**
    1.  **Describe the Pod:** Check the `Events` at the bottom.
        ```bash
        kubectl describe pod <pod-name>
        ```
        You will see a `FailedScheduling` event with a message like: `0/3 nodes are available: 3 node(s) didn't have enough cpu/memory` or `3 node(s) had taints that the pod didn't tolerate`.
*   **Common Causes & Solutions:**
    *   **Insufficient Cluster Resources:** The cluster simply doesn't have enough unallocated CPU or memory on any node to meet the pod's `resources.requests`.
        *   **Solution (EKS/Cloud):** If you have a **Cluster Autoscaler** configured, this should trigger it to add a new node. If not, you need to manually add nodes to your Node Group / ASG.
        *   **Solution (Self-Managed):** Manually provision a new worker node and add it to the cluster.
    *   **Mismatched Taints/Tolerations:** The nodes have taints (e.g., dedicated to a specific workload), and your pod does not have the corresponding toleration. **Solution:** Add the required `tolerations` to your pod spec or remove the taint from the node if it's incorrect.

---

#### Use Case 4: Pod Stuck in `Pending` (Volume Issues)
*   **Symptom:** A pod that requires persistent storage is stuck in a `Pending` state.
*   **Environment:** Both Self-Managed & EKS.
*   **Troubleshooting:**
    1.  **Describe the Pod:** The `Events` section is key.
        ```bash
        kubectl describe pod <pod-name>
        ```
        The event message will be something like: `pod has unbound immediate PersistentVolumeClaims`. This tells you the problem isn't the pod itself, but its storage request.
    2.  **Check the PersistentVolumeClaim (PVC):**
        ```bash
        kubectl get pvc <pvc-name-from-pod-spec>
        ```
        You'll see its status is also `Pending`.
    3.  **Describe the PVC:** Now find out why the PVC is pending.
        ```bash
        kubectl describe pvc <pvc-name>
        ```
        The events will give the final clue, like `failed to provision volume with StorageClass "aws-ebs": rpc error...` or `no persistent volumes available for this claim and no storage class is set`.
*   **Common Causes & Solutions:**
    *   **Invalid `StorageClass` Name:** The `storageClassName` in the PVC spec doesn't exist. **Solution:** Correct the name (`kubectl get sc` to see available classes).
    *   **Dynamic Provisioning Fails:** The underlying storage provisioner is failing. See **Use Case 10** for a detailed breakdown.
    *   **No Available `PersistentVolume` (for static provisioning):** You are not using a `StorageClass` and are expecting to bind to a pre-created `PersistentVolume` (PV), but no available PV meets the PVC's size and access mode requirements. **Solution:** Create a suitable PV or switch to dynamic provisioning with a `StorageClass`.

---

### Category 2: Node & Cluster Level Issues

#### Use Case 5: Worker Node Fails to Join the Cluster (Self-Managed)
*   **Symptom:** You've provisioned a new VM, installed `kubelet`, but it never appears in `kubectl get nodes`.
*   **Environment:** Self-Managed (kubeadm, kops, etc.).
*   **Troubleshooting:**
    1.  **SSH into the Failing Node.** The problem is on the node itself.
    2.  **Check Kubelet Logs:** This is the most critical step.
        ```bash
        sudo journalctl -u kubelet -f
        ```
    3.  **Check Network Connectivity:** Can the worker node reach the API server?
        ```bash
        # From the worker node
        curl -k https://<your-api-server-ip>:<port>/version
        ```
        If this times out, you have a networking issue (firewall, security group, routing).
*   **Common Causes & Solutions:**
    *   **Network Failure:** A firewall is blocking the port (usually 6443) between the worker and the control plane. **Solution:** Open the required ports.
    *   **Invalid Bootstrap Token:** When using `kubeadm join`, the token might be expired or incorrect. **Solution:** Generate a new token on the control plane (`kubeadm token create`) and re-run the join command.
    *   **Certificate Mismatch/TLS Handshake Error:** The logs show a TLS error. This means the kubelet isn't trusting the API server's certificate, or the CA hash provided during the join is wrong. **Solution:** Re-run the `kubeadm join` command with the correct `--discovery-token-ca-cert-hash`.

---

#### Use Case 6: EKS Worker Node Fails to Join the Cluster
*   **Symptom:** An EC2 instance launches in your EKS Node Group, but it never appears as `Ready` in `kubectl get nodes`.
*   **Environment:** EKS.
*   **Troubleshooting:**
    1.  **Check Security Groups:** The worker node SG must allow outbound traffic on TCP port 443 to the EKS control plane SG. The control plane SG must allow this inbound traffic.
    2.  **Check VPC Subnet Tags:** The subnets where nodes are launched must be tagged correctly for EKS to use them. The key tag is `kubernetes.io/cluster/<cluster-name>` with a value of `owned` or `shared`.
    3.  **Check Kubelet Logs on the Node:** Use SSM Session Manager or SSH to connect to the EC2 instance and run `sudo journalctl -u kubelet -f`.

---

### Category 3: Storage & Cloud Integration Issues

#### Use Case 7: Dynamic Volume Provisioning Fails (EBS/StorageClass)
*   **Symptom:** A `PersistentVolumeClaim` (PVC) is stuck in the `Pending` state indefinitely.
*   **Environment:** EKS / Cloud with a CSI driver.
*   **Troubleshooting:**
    1.  **Describe the PVC:** This is the starting point.
        ```bash
        kubectl describe pvc my-app-pvc
        ```
        The `Events` section will show an error like `ProvisioningFailed`.
    2.  **Check the Provisioner Logs:** The error message in the PVC event often hints at the cause, but the real details are in the logs of the storage provisioner pod. For EKS using the EBS CSI driver, this is the `ebs-csi-controller` pod.
        ```bash
        kubectl logs -n kube-system -l app=ebs-csi-controller
        ```
*   **Common Causes & Solutions:**
    *   **Incorrect `StorageClass` Parameters:** The `StorageClass` YAML might have invalid parameters (e.g., incorrect `type: gp3` vs `gp2`, or invalid encryption settings). **Solution:** Correct the `StorageClass` definition.
    *   **IAM Permissions Failure:** The CSI controller's Service Account (using IRSA) lacks the IAM permissions to create/modify EBS volumes (`ec2:CreateVolume`, `ec2:AttachVolume`, `ec2:CreateTags`, etc.). **Solution:** Verify the IAM Role attached to the `ebs-csi-controller-sa` service account has the required permissions. The AWS managed policy `AmazonEBSCSIDriverPolicy` is usually sufficient.
    *   **AZ Mismatch:** The pod is scheduled in an Availability Zone where the requested EBS volume type is not available, or it's trying to attach a volume that exists in a different AZ (for pre-provisioned volumes).

---

#### Use Case 8: AWS Load Balancer Controller Fails to Create an ALB
*   **Symptom:** You create an `Ingress` object with the correct annotations for the ALB controller, but no ALB is created in your AWS account, or the Ingress address remains empty.
*   **Environment:** EKS.
*   **Troubleshooting:**
    1.  **Check the Controller Logs:** This is the most important step. The controller will tell you exactly why it's failing.
        ```bash
        # The controller deployment is named 'aws-load-balancer-controller'
        kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
        ```
    2.  **Describe the Ingress:**
        ```bash
        kubectl describe ingress my-ingress
        ```
        Look for events from the controller.
*   **Common Causes & Solutions:**
    *   **Missing Subnet Tags:** The controller needs to know which subnets to use for the ALB. Your public subnets must be tagged with `kubernetes.io/role/elb: "1"`, and private subnets with `kubernetes.io/role/internal-elb: "1"`.
    *   **IAM Permissions Failure:** The controller's Service Account (using IRSA) is missing the extensive IAM permissions needed to manage ALBs, Target Groups, Security Groups, etc. **Solution:** Ensure the IAM role has the correct policy. AWS provides a recommended policy JSON you can download and attach.
    *   **Conflicting Ingresses:** Two Ingress objects are trying to claim the same listener port on the same ALB (this is controlled by IngressGroups). The logs will show a conflict.
    *   **Security Group Issues:** The controller might not have permission to modify the node security groups to allow traffic from the ALB.

---

#### Use Case 9: Pod gets "Access Denied" from AWS (IRSA Failure)
*   **Symptom:** Your application pod starts but fails with "Access Denied" when trying to call an AWS service like S3 or DynamoDB.
*   **Environment:** EKS.
*   **Troubleshooting:** This requires a meticulous checklist.
    1.  **Service Account Annotation:** `kubectl get sa my-app-sa -o yaml`. Verify the `eks.amazonaws.com/role-arn` annotation is correct.
    2.  **Pod Spec:** Verify the pod is using the correct `serviceAccountName`.
    3.  **IAM Role Trust Policy:** This is the most common failure point. Go to the IAM Role in the AWS console. The trust policy **must** perfectly match the OIDC provider URL and the `system:serviceaccount:namespace:name`. One typo will cause it to fail.
    4.  **IAM Role Permissions:** Does the role's *permissions policy* actually grant access to the resource (e.g., `s3:GetObject` on `arn:aws:s3:::my-bucket/*`)?
    5.  **Injected Variables:** `kubectl exec -it <pod-name> -- env | grep AWS`. Verify `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` are present. If not, the EKS Pod Identity Webhook is not working.

---

### Category 4: Operational & Observability Issues

#### Use Case 10: Unable to Retrieve Complete Application Logs
*   **Symptom:** You run `kubectl logs` but the error you're looking for is gone because the pod has restarted multiple times, or the logs have rotated. You cannot search or aggregate logs across multiple pods.
*   **Environment:** Both Self-Managed & EKS.
*   **Troubleshooting (This is an architectural problem, not a single command fix):**
    *   `kubectl logs` is a debugging tool, not a production logging solution. It only fetches logs currently stored by the container runtime on the node, which is limited in size and ephemeral.
*   **Solution: Implement a Centralized Logging Architecture:**
    1.  **Deploy a Logging Agent:** Deploy a logging agent like **Fluentd** or **Fluent Bit** as a `DaemonSet`. This runs one agent pod on every node in the cluster.
    2.  **Configure the Agent:** The agent is configured to automatically discover running containers on its node and scrape their log files (usually from `/var/log/pods/...`). It enriches these logs with Kubernetes metadata (pod name, namespace, labels).
    3.  **Forward to a Backend:** The agent forwards these structured logs to a centralized, searchable backend.
        *   **Self-Hosted:** Elasticsearch/OpenSearch with Kibana (the ELK/EFK Stack), or Loki with Grafana.
        *   **Cloud-Native:** Amazon CloudWatch Logs, Google Cloud Logging, etc.
    4.  **Sidecar Pattern:** For applications that don't write to `stdout`/`stderr` but to files, you can use a "sidecar" container. This is a second container in the same Pod that reads the log file from a shared volume and streams it to its own `stdout`, where the node-level agent can then pick it up.