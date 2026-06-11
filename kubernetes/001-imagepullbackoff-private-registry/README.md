# Incident 001: Kubernetes ImagePullBackOff due to Private GitLab Registry Authentication Failure

## Objective

This lab reproduces a real-world Kubernetes deployment failure where a workload cannot pull an image from a private GitLab Container Registry.

The goal is to demonstrate a production-style DevOps/SRE troubleshooting process for `ImagePullBackOff`, including detection, root cause analysis, resolution, validation, and preventive controls.

This scenario is relevant because private registry authentication failures are common in Kubernetes environments using GitLab Container Registry, Harbor, Docker Registry, AWS ECR, Azure ACR, or other private image repositories.

---

## Environment

| Component | Details |
|---|---|
| Kubernetes cluster | kubeadm homelab |
| Runtime | containerd |
| Registry | GitLab Container Registry |
| Registry endpoint | `10.10.1.101:5050` |
| Namespace | `devops-lab` |
| Application | `demo-private-registry-app` |
| Failure type | Missing or invalid `imagePullSecret` |
| Expected pod state | `ImagePullBackOff` / `ErrImagePull` |

---

## Skills Demonstrated

- Kubernetes workload troubleshooting
- Private container registry authentication
- GitLab Container Registry usage
- Kubernetes `imagePullSecrets`
- Namespace-level ServiceAccount configuration
- Deployment rollout recovery
- Event and log analysis
- Incident documentation
- Runbook creation
- Preventive control design

---

## Failure Simulated

A Kubernetes Deployment references an image stored in a private GitLab Container Registry:

```text
10.10.1.101:5050/root/demo-private-registry-app:latest
```

However, the `devops-lab` namespace does not contain a valid image pull secret.

Because Kubernetes does not have valid credentials to authenticate to the private registry, the pod cannot pull the image and enters:

```text
ImagePullBackOff
ErrImagePull
```

Common event messages may include:

```text
Failed to pull image
pull access denied
authentication required
no basic auth credentials
Back-off pulling image
```

---

## Architecture

```text
Developer / GitLab CI
        |
        v
GitLab Container Registry
10.10.1.101:5050
        |
        | image pull requires authentication
        v
Kubernetes Worker Node
containerd runtime
        |
        v
Pod: demo-private-registry-app
```

Failure point:

```text
Kubernetes pod -> private registry authentication -> failed image pull
```

---

## Repository Structure

```text
real-world-devops-sre-incidents/
├── README.md
├── kubernetes/
│   └── 001-imagepullbackoff-private-registry/
│       └── README.md
├── manifests/
│   └── kubernetes/
│       └── demo-private-registry-deployment.yaml
├── runbooks/
│   └── imagepullbackoff-private-registry.md
├── incident-reports/
│   └── 001-imagepullbackoff-private-registry.md
└── evidence/
    ├── logs/
    │   ├── 001-namespace-created.txt
    │   ├── 001-workload-status-before.txt
    │   ├── 001-kubectl-describe-pod-before.txt
    │   ├── 001-kubectl-events-before.txt
    │   ├── 001-workload-status-after.txt
    │   ├── 001-kubectl-describe-pod-after.txt
    │   └── 001-rollout-status-after.txt
    └── screenshots/
```

---

## Step 1: Create Namespace

```bash
kubectl create namespace devops-lab
```

Verify:

```bash
kubectl get namespace devops-lab
```

Save evidence:

```bash
kubectl get namespace devops-lab > evidence/logs/001-namespace-created.txt
```

---

## Step 2: Create Broken Deployment

Create this manifest:

```bash
nano manifests/kubernetes/demo-private-registry-deployment.yaml
```

Paste:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-private-registry-app
  namespace: devops-lab
  labels:
    app: demo-private-registry-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-private-registry-app
  template:
    metadata:
      labels:
        app: demo-private-registry-app
    spec:
      containers:
        - name: demo-private-registry-app
          image: 10.10.1.101:5050/root/demo-private-registry-app:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
```

Apply it:

```bash
kubectl apply -f manifests/kubernetes/demo-private-registry-deployment.yaml
```

Check pod status:

```bash
kubectl get pods -n devops-lab
```

Expected result:

```text
NAME                                         READY   STATUS             RESTARTS   AGE
demo-private-registry-app-xxxxxxxxxx-xxxxx   0/1     ImagePullBackOff   0          1m
```

---

## Step 3: Capture Failure Evidence

Set pod variable:

```bash
POD=$(kubectl get pods -n devops-lab -l app=demo-private-registry-app -o jsonpath='{.items[0].metadata.name}')
```

Capture workload status:

```bash
kubectl get deploy,pod -n devops-lab -o wide | tee evidence/logs/001-workload-status-before.txt
```

Capture pod description:

```bash
kubectl describe pod "$POD" -n devops-lab | tee evidence/logs/001-kubectl-describe-pod-before.txt
```

Capture namespace events:

```bash
kubectl get events -n devops-lab --sort-by=.lastTimestamp | tee evidence/logs/001-kubectl-events-before.txt
```

---

## Step 4: Troubleshooting Process

### 4.1 Check pod state

```bash
kubectl get pods -n devops-lab
```

Look for:

```text
ImagePullBackOff
ErrImagePull
```

### 4.2 Describe the pod

```bash
kubectl describe pod "$POD" -n devops-lab
```

Check the `Events` section for image pull errors.

### 4.3 Verify image name

```bash
kubectl get deployment demo-private-registry-app   -n devops-lab   -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
```

Expected:

```text
10.10.1.101:5050/root/demo-private-registry-app:latest
```

### 4.4 Check secrets in namespace

```bash
kubectl get secrets -n devops-lab
```

If no registry secret exists, Kubernetes has no credentials to authenticate to the private registry.

### 4.5 Check whether Deployment has imagePullSecrets

```bash
kubectl get deployment demo-private-registry-app -n devops-lab -o yaml | grep -A5 imagePullSecrets
```

If no output appears, the Deployment does not reference a registry secret.

### 4.6 Check ServiceAccount imagePullSecrets

```bash
kubectl get serviceaccount default -n devops-lab -o yaml
```

If `imagePullSecrets` is missing, the default ServiceAccount also does not provide registry credentials.

---

## Root Cause

The workload used an image from a private GitLab Container Registry, but the Kubernetes namespace did not contain or reference a valid registry authentication secret.

The pod failed because Kubernetes could not authenticate to:

```text
10.10.1.101:5050
```

Root cause category:

```text
Deployment configuration / registry authentication
```

---

## Step 5: Create GitLab Registry Secret

Use a GitLab Deploy Token or another registry credential with minimum required access.

Recommended permission:

```text
read_registry
```

Set variables locally:

```bash
REGISTRY_SERVER="10.10.1.101:5050"
REGISTRY_USER="<your-gitlab-deploy-token-username>"
REGISTRY_PASSWORD="<your-gitlab-deploy-token-password>"
REGISTRY_EMAIL="myserv@gmail.com"
```

Create the Kubernetes registry secret:

```bash
kubectl create secret docker-registry gitlab-regcred   --docker-server="$REGISTRY_SERVER"   --docker-username="$REGISTRY_USER"   --docker-password="$REGISTRY_PASSWORD"   --docker-email="$REGISTRY_EMAIL"   -n devops-lab
```

Verify secret exists:

```bash
kubectl get secret gitlab-regcred -n devops-lab
```

Expected:

```text
NAME             TYPE                             DATA   AGE
gitlab-regcred   kubernetes.io/dockerconfigjson   1      10s
```

Important:

```text
Do not commit real secret values, token values, or full secret YAML into GitHub.
```

---

## Step 6: Patch Deployment with imagePullSecret

Patch the Deployment:

```bash
kubectl patch deployment demo-private-registry-app   -n devops-lab   -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"gitlab-regcred"}]}}}}'
```

Restart rollout:

```bash
kubectl rollout restart deployment demo-private-registry-app -n devops-lab
```

Watch rollout:

```bash
kubectl rollout status deployment demo-private-registry-app -n devops-lab
```

Check pod:

```bash
kubectl get pods -n devops-lab -o wide
```

Expected result:

```text
NAME                                         READY   STATUS    RESTARTS   AGE
demo-private-registry-app-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
```

---

## Step 7: Capture Recovery Evidence

Refresh pod variable:

```bash
POD=$(kubectl get pods -n devops-lab -l app=demo-private-registry-app -o jsonpath='{.items[0].metadata.name}')
```

Capture workload status:

```bash
kubectl get deploy,pod -n devops-lab -o wide | tee evidence/logs/001-workload-status-after.txt
```

Capture pod description:

```bash
kubectl describe pod "$POD" -n devops-lab | tee evidence/logs/001-kubectl-describe-pod-after.txt
```

Capture rollout status:

```bash
kubectl rollout status deployment demo-private-registry-app -n devops-lab | tee evidence/logs/001-rollout-status-after.txt
```

---

## Alternative Production Pattern: Patch Default ServiceAccount

If most workloads in the namespace pull from the same private registry, add the secret to the default ServiceAccount.

```bash
kubectl patch serviceaccount default   -n devops-lab   -p '{"imagePullSecrets":[{"name":"gitlab-regcred"}]}'
```

Verify:

```bash
kubectl get serviceaccount default -n devops-lab -o yaml
```

This allows pods using the default ServiceAccount to automatically use the registry secret.

---

## Validation Checklist

| Check | Command | Expected Result |
|---|---|---|
| Namespace exists | `kubectl get ns devops-lab` | Namespace exists |
| Secret exists | `kubectl get secret gitlab-regcred -n devops-lab` | Secret exists |
| Deployment patched | `kubectl get deploy demo-private-registry-app -n devops-lab -o yaml` | `imagePullSecrets` exists |
| Pod running | `kubectl get pods -n devops-lab` | Pod is `Running` |
| Rollout successful | `kubectl rollout status deployment demo-private-registry-app -n devops-lab` | Successfully rolled out |
| No pull error | `kubectl describe pod "$POD" -n devops-lab` | No new pull failure events |

---

## Preventive Controls

1. Use GitLab Deploy Tokens with least privilege.
2. Use `read_registry` permission for pull-only workloads.
3. Avoid personal access tokens for automated workloads.
4. Store registry credentials as Kubernetes Secrets.
5. Reference registry secrets through ServiceAccounts for namespace-wide consistency.
6. Avoid using `latest` image tags in production.
7. Use immutable tags such as commit SHA or semantic versions.
8. Add CI/CD validation to confirm image existence before deployment.
9. Add deployment alerts for failed rollout or pod image pull errors.
10. Document private registry onboarding in a runbook.
11. Rotate registry tokens periodically.
12. Never commit real credentials or secret manifests to GitHub.

---

## Example Production-Ready Manifest After Fix

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-private-registry-app
  namespace: devops-lab
  labels:
    app: demo-private-registry-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-private-registry-app
  template:
    metadata:
      labels:
        app: demo-private-registry-app
    spec:
      imagePullSecrets:
        - name: gitlab-regcred
      containers:
        - name: demo-private-registry-app
          image: 10.10.1.101:5050/root/demo-private-registry-app:v1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 300m
              memory: 256Mi
```

---

## Commands Summary

```bash
kubectl create namespace devops-lab

kubectl apply -f manifests/kubernetes/demo-private-registry-deployment.yaml

kubectl get pods -n devops-lab

POD=$(kubectl get pods -n devops-lab -l app=demo-private-registry-app -o jsonpath='{.items[0].metadata.name}')

kubectl describe pod "$POD" -n devops-lab

kubectl get events -n devops-lab --sort-by=.lastTimestamp

REGISTRY_SERVER="10.10.1.101:5050"
REGISTRY_USER="<your-gitlab-deploy-token-username>"
REGISTRY_PASSWORD="<your-gitlab-deploy-token-password>"
REGISTRY_EMAIL="myserv@gmail.com"

kubectl create secret docker-registry gitlab-regcred   --docker-server="$REGISTRY_SERVER"   --docker-username="$REGISTRY_USER"   --docker-password="$REGISTRY_PASSWORD"   --docker-email="$REGISTRY_EMAIL"   -n devops-lab

kubectl patch deployment demo-private-registry-app   -n devops-lab   -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"gitlab-regcred"}]}}}}'

kubectl rollout restart deployment demo-private-registry-app -n devops-lab

kubectl rollout status deployment demo-private-registry-app -n devops-lab

kubectl get pods -n devops-lab -o wide
```

---

## Evidence to Commit

Commit these files:

```text
evidence/logs/001-namespace-created.txt
evidence/logs/001-workload-status-before.txt
evidence/logs/001-kubectl-describe-pod-before.txt
evidence/logs/001-kubectl-events-before.txt
evidence/logs/001-workload-status-after.txt
evidence/logs/001-kubectl-describe-pod-after.txt
evidence/logs/001-rollout-status-after.txt
```

Do not commit:

```text
Real GitLab token
Real Docker config JSON
Full Kubernetes Secret YAML with encoded credentials
Screenshots showing secret values
```

---

## Lessons Learned

Private registry authentication is a critical deployment dependency.

A Kubernetes manifest can be syntactically correct and the application image can exist, but the workload will still fail if the cluster does not have valid registry credentials.

The best long-term fix is to standardize private registry access at namespace or platform level using ServiceAccount `imagePullSecrets`, least-privilege deploy tokens, CI/CD validation, and clear incident runbooks.

---

## Status

| Item | Status |
|---|---|
| Failure reproduced | Pending |
| Evidence captured | Pending |
| Root cause documented | Completed |
| Fix implemented | Pending |
| Validation completed | Pending |
| Preventive actions documented | Completed |
