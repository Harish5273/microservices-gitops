# Troubleshooting and Solutions – Kubernetes Microservices Deployment

## Project Overview

During the deployment of a microservices application using Kubernetes, Argo CD, Traefik, and Amazon ECR, we encountered several issues.

This document explains the problems, root causes, troubleshooting steps, and solutions implemented.

### Environment

| Component | Technology |
|---|---|
| Cloud Provider | AWS |
| Compute | EC2 – Ubuntu |
| Container Registry | Amazon ECR |
| Container Orchestration | Kubernetes (K3s) |
| GitOps | Argo CD |
| Ingress Controller | Traefik |
| Container Runtime | containerd |
| Application Namespace | microservices |
| AWS Region | us-east-1 |

### Application Services

1. Frontend
2. User Service
3. Product Service
4. Order Service
5. Notification Service

---

# Issue 1: Argo CD Sync Failed Due to NGINX Admission Webhook

### Problem

Argo CD application showed:

```text
SYNC STATUS: OutOfSync
HEALTH STATUS: Healthy
```

During synchronization, Argo CD reported:

```text
failed calling webhook
"validate.nginx.ingress.kubernetes.io"

service "ingress-nginx-controller-admission" not found
```

### Root Cause

The cluster had both NGINX and Traefik Ingress configurations.

The NGINX ValidatingWebhookConfiguration was still registered, but its associated admission Service was missing.

The webhook had:

```yaml
failurePolicy: Fail
```

This meant Kubernetes rejected Ingress creation or updates when the webhook could not be reached.

### Troubleshooting

Checked the registered webhook configurations:

```bash
kubectl get validatingwebhookconfigurations
```

Inspected the NGINX admission webhook:

```bash
kubectl describe validatingwebhookconfiguration ingress-nginx-admission
```

Confirmed that it referenced:

```text
Service:
  Name: ingress-nginx-controller-admission
  Namespace: ingress-nginx
  Port: 443
```

### Solution

Since the deployment was moving to Traefik and the NGINX admission webhook was no longer required, removed the stale webhook configuration:

```bash
kubectl delete validatingwebhookconfiguration ingress-nginx-admission
```

### Verification

```bash
kubectl get validatingwebhookconfigurations
```

**Result:** The stale NGINX webhook no longer blocked Kubernetes Ingress updates.

Note: This removed the admission webhook configuration, not necessarily the NGINX controller itself.

---

# Issue 2: Ingress Resources Were Not Explicitly Assigned to Traefik

### Problem

The cluster had two IngressClasses:

```bash
kubectl get ingressclass
```

Output:

```text
NAME       CONTROLLER
nginx      k8s.io/ingress-nginx
traefik    traefik.io/ingress-controller
```

Some Ingress resources showed:

```text
CLASS: <none>
```

Only the frontend Ingress explicitly used Traefik.

### Root Cause

The Kubernetes Ingress manifests for the backend services did not explicitly specify an IngressClass.

Although Traefik was the default IngressClass, explicit configuration was preferable to avoid ambiguity between multiple controllers.

### Solution

Updated the Ingress manifests in the GitHub GitOps repository.

Repository:

https://github.com/Harish5273/microservices-gitops

Added this field under `spec` in all five Ingress manifests:

```yaml
spec:
  ingressClassName: traefik
```

Preserved the existing routing rules, paths, and backend Services.

Committed and pushed the changes to GitHub. Argo CD automatically detected the updated manifests because automated synchronization was enabled.

### Verification

```bash
kubectl get ingress -n microservices
```

Expected result:

```text
NAME                           CLASS
frontend-ingress               traefik
notification-service-ingress   traefik
order-service-ingress          traefik
product-service-ingress        traefik
user-service-ingress           traefik
```

**Result:** All five Ingress resources were successfully assigned to Traefik.

---

# Issue 3: Argo CD Showed Synced but Progressing

### Problem

After fixing the Ingress issue, Argo CD showed:

```text
SYNC STATUS: Synced
HEALTH STATUS: Progressing
```

### Root Cause Investigation

Inspected the Argo CD Application:

```bash
kubectl get applications microservices -n argocd -o yaml
```

Found the health message:

```text
Caused by apps/Deployment:microservices/user-service,
apps/Deployment:microservices/frontend
```

Initially, the frontend and user-service Deployments were identified as the resources causing the Progressing status.

Further investigation showed that all five microservices had an image-pulling problem.

### Important Concept

- Synced means the Kubernetes resources match the desired Git configuration.
- Healthy means the deployed resources are functioning according to their health checks.

A successful GitOps synchronization does not guarantee that application containers have started successfully.

---

# Issue 4: All Five Pods Were in ImagePullBackOff

### Problem

Checked application pods:

```bash
kubectl get pods -n microservices
```

Output:

```text
NAME                     READY   STATUS
frontend                 0/1     ImagePullBackOff
user-service             0/1     ImagePullBackOff
product-service          0/1     ImagePullBackOff
order-service            0/1     ImagePullBackOff
notification-service     0/1     ImagePullBackOff
```

### Root Cause Investigation

Inspected the frontend pod:

```bash
kubectl describe pod <frontend-pod-name> -n microservices
```

The Events section showed:

```text
Failed to pull image

pull access denied, repository does not exist
or may require authorization

authorization failed: no basic auth credentials
```

This identified the actual problem.

The application images were stored in private Amazon ECR repositories, but Kubernetes did not have registry authentication credentials.

### Step 1: Verify AWS Identity

```bash
aws sts get-caller-identity
```

Confirmed the AWS identity:

```text
Account: 591424882000
Arn: arn:aws:iam::591424882000:user/microserive-app
```

The EC2 shell was using an IAM user rather than an EC2 IAM role.

### Step 2: Verify the ECR Image

```bash
aws ecr describe-images \
  --repository-name frontend \
  --image-ids imageTag=latest \
  --region us-east-1
```

Confirmed that the frontend image existed in ECR.

### Step 3: Verify All Five Repositories

```bash
for repo in frontend user-service product-service order-service notification-service
do
  echo "Checking $repo..."

  aws ecr describe-images \
    --repository-name "$repo" \
    --image-ids imageTag=latest \
    --region us-east-1 \
    --query 'imageDetails[0].imageTags' \
    --output text
done
```

Confirmed that all five repositories contained the `latest` image tag.

### Step 4: Verify ECR Authentication

```bash
aws ecr get-login-password --region us-east-1 > /dev/null

echo $?
```

Output:

```text
0
```

This confirmed that the AWS CLI could successfully generate an ECR authentication token.

### Conclusion

The repositories and images existed, and AWS CLI authentication worked.

The missing configuration was Kubernetes image-pull authentication.

---

# Issue 5: Kubernetes Had No ECR Image Pull Secret

### Problem

Inspected the default ServiceAccount:

```bash
kubectl get serviceaccount default -n microservices -o yaml
```

There was no `imagePullSecrets` configuration.

Checked the namespace Secrets:

```bash
kubectl get secrets -n microservices
```

Output:

```text
No resources found in microservices namespace.
```

### Root Cause

Kubernetes did not have a Docker registry Secret containing the credentials required to authenticate with private ECR.

### Solution

#### Step 1: Generate an ECR Authentication Token

```bash
ECR_TOKEN=$(aws ecr get-login-password --region us-east-1)
```

This stores the token in a temporary shell variable.

#### Step 2: Create a Kubernetes Docker Registry Secret

```bash
kubectl create secret docker-registry ecr-registry \
  --docker-server=591424882000.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$ECR_TOKEN" \
  --namespace=microservices
```

Clear the temporary shell variable:

```bash
unset ECR_TOKEN
```

Note: An initial attempt using `--docker-password-stdin` failed because the installed kubectl version did not support that flag. We used `--docker-password` with a shell variable instead.

#### Step 3: Attach the Secret to the Default ServiceAccount

```bash
kubectl patch serviceaccount default \
  -n microservices \
  -p '{"imagePullSecrets":[{"name":"ecr-registry"}]}'
```

Why?

All five Deployments used the default ServiceAccount.

Attaching the image pull secret to this ServiceAccount allowed pods using it to reference the ECR credentials automatically.

#### Step 4: Verify the Secret

```bash
kubectl get secret ecr-registry -n microservices
```

Expected type:

```text
kubernetes.io/dockerconfigjson
```

Verify the ServiceAccount:

```bash
kubectl get serviceaccount default -n microservices -o yaml
```

Expected configuration:

```yaml
imagePullSecrets:
- name: ecr-registry
```

**Result:** Kubernetes now had the registry credentials required to pull private ECR images.

---

# Issue 6: Existing Pods Needed to Be Recreated

### Problem

After configuring the image pull secret, the existing pods had already entered `ImagePullBackOff`.

### Solution

Restarted all application Deployments:

```bash
kubectl rollout restart deployment -n microservices
```

This triggered new pods to be created using the updated ServiceAccount configuration.

### Verification

Monitored pod status:

```bash
kubectl get pods -n microservices -w
```

Initially, the new pods appeared as:

```text
0/1 Running
```

After the containers started and readiness probes passed:

```text
1/1 Running
```

All five services successfully pulled their images and became Ready.

The old pods entered `Terminating` during the rollout and were removed.

**Result:** The ECR image-pull authentication problem was resolved.

---

# Final Verification Commands

Use these commands to verify the complete deployment.

### 1. Check Application Pods

```bash
kubectl get pods -n microservices
```

Expected: All five pods should show `1/1 Running`.

### 2. Check Deployments

```bash
kubectl get deployments -n microservices
```

Expected: All five Deployments should show their desired replicas as ready and available.

### 3. Check Services

```bash
kubectl get svc -n microservices
```

### 4. Check Ingress Resources

```bash
kubectl get ingress -n microservices
```

Expected: All five Ingress resources should use the Traefik class.

### 5. Check Argo CD Application

```bash
kubectl get applications -n argocd
```

Expected:

```text
NAME           SYNC STATUS   HEALTH STATUS
microservices  Synced        Healthy
```

### 6. Check Traefik

```bash
kubectl get pods -n kube-system | grep traefik
```

Expected: Traefik controller should be Running and Ready.

---

# Important Lessons Learned

| Problem | What I learned |
|---|---|
| Admission webhook failure | Kubernetes admission webhooks can block resource creation and updates. |
| Multiple IngressClasses | Explicit `ingressClassName` avoids ambiguity between Ingress controllers. |
| Synced but Progressing | GitOps synchronization and application health are different concepts. |
| ImagePullBackOff | Always inspect pod Events to identify the actual image-pull failure. |
| Private ECR authentication | AWS CLI authentication does not automatically authenticate Kubernetes container pulls. |
| Image pull secrets | Kubernetes needs registry credentials to pull images from private registries. |
| ServiceAccounts | Image pull secrets can be attached to ServiceAccounts and reused by their pods. |
| Rollout restart | Recreating pods allows them to use updated ServiceAccount configuration. |

---

# Pending Improvement: Automatic ECR Credential Renewal

The current solution uses an ECR authorization token stored in a Kubernetes Secret.

**Important:** ECR authorization tokens expire after 12 hours.

Existing running containers continue running, but a future pod replacement may fail to pull an image after the token expires.

For a more reliable setup, the next improvement is to:

1. Replace the long-lived IAM user credentials with an EC2 IAM role.
2. Grant the role the required ECR pull permissions.
3. Configure automatic ECR credential renewal or a suitable ECR credential-provider integration for K3s/containerd.
4. Test image pulls after token renewal.

Do not consider automatic renewal configured yet; the current implementation is a working temporary solution.

---

# Final Outcome

The following issues were identified and resolved:

- Removed a stale NGINX admission webhook that blocked Ingress updates.
- Explicitly configured all five Ingress resources to use Traefik.
- Identified why Argo CD showed Synced but Progressing.
- Diagnosed private ECR authentication failure using pod Events.
- Verified that all five ECR repositories and image tags existed.
- Created an ECR Docker registry Secret in Kubernetes.
- Attached the Secret to the default ServiceAccount.
- Restarted Deployments and successfully pulled all five images.

**Final result:** All five microservices successfully pulled their private ECR images and reached `1/1 Running`. The GitOps synchronization was successful. Final Argo CD Healthy status should be confirmed with the verification commands above.
