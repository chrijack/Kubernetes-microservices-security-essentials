# Lesson 18.3 — Pod Security Admission

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Demonstrate Pod Security Admission (PSA) enforcement by creating a namespace with specific security standards, attempting to deploy an insecure pod (which will be rejected), and then deploying a compliant secure pod.

## Prerequisites

- Kubernetes cluster with Pod Security Admission enabled
- `kubectl` access to the cluster

## Steps

### Step 1: Create a Namespace with Specific Security Standards

Create a namespace with Pod Security Admission labels that enforce the restricted security standard, audit against baseline, and warn about privileged configurations.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: privileged
EOF
```

Expected output:
```
namespace/secure-namespace created
```

### Step 2: Attempt to Deploy an Insecure Pod

Try to deploy a pod with privileged mode enabled. This deployment will fail because it violates the enforced "restricted" security standard.

```bash
cat <<EOF | kubectl apply -f - -n secure-namespace
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
spec:
  containers:
  - name: web-container
    image: nginx
    securityContext:
      privileged: true
EOF
```

Expected output:
```
Error from server (Forbidden): error when creating "STDIN": pods "insecure-pod" is forbidden: violates PodSecurity "restricted:latest": privileged (container "web-container" must not set securityContext.privileged=true)
```

### Step 3: Deploy a Secure Pod in the Same Namespace

Deploy a pod that complies with the "restricted" security standard. This includes:
- Running as a non-root user
- Dropping all Linux capabilities
- Disallowing privilege escalation
- Using RuntimeDefault seccomp profile

```bash
cat <<EOF | kubectl apply -f - -n secure-namespace
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: web-container
    image: nginxinc/nginx-unprivileged
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      seccompProfile:
        type: RuntimeDefault
EOF
```

Expected output:
```
pod/secure-pod created
```

## Verification

The secure pod will be created successfully because it complies with all security policies defined in the `secure-namespace`. The insecure pod was rejected, demonstrating that Pod Security Admission is actively enforcing the configured security standards.

## Cleanup

Remove all resources created during the demonstration:

```bash
kubectl delete pod insecure-pod secure-pod -n secure-namespace
kubectl delete namespace secure-namespace
```
