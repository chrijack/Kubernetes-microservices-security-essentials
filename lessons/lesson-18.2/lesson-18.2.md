# Lesson 18.2 — Configure Security Contexts

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Understand and implement Kubernetes security contexts to enforce container sandboxing and runtime security. This includes running containers as non-root users, managing privilege escalation, controlling capabilities, and validating security policies.

## Prerequisites

- Kubernetes cluster running
- `kubectl` configured and connected to the cluster
- `capsh` utility available for decoding Linux capabilities

## Steps

### Step 1: Running Containers as Non-Root Users

Deploy an insecure Pod running as root:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
spec:
  containers:
  - name: web-container
    image: nginx
EOF
```

Deploy a secure Pod running as non-root user:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  containers:
  - name: web-container
    image: nginxinc/nginx-unprivileged
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
EOF
```

Compare the user IDs:

```bash
echo "Insecure Pod (running as root):"
kubectl exec insecure-pod -- id
echo "\nSecure Pod (running as non-root):"
kubectl exec secure-pod -- id
```

### Step 2: Pod-Level vs. Container-Level Security Contexts

Deploy a Pod with a pod-level security context:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-level-security-context
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
EOF
```

Deploy a Pod with a container-level security context overriding the pod-level context:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: container-level-security-context
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 2000
EOF
```

Verify the effective user and group IDs:

```bash
echo "Checking user and group IDs in pod-level-security-context pod:"
kubectl exec pod-level-security-context -- id

echo "\nChecking user and group IDs in container-level-security-context pod:"
kubectl exec container-level-security-context -- id
```

### Step 3: Preventing Privilege Escalation

Deploy a Pod with privilege escalation prevented:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-escalation-pod
spec:
  containers:
  - name: alpine
    image: alpine:latest
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
      allowPrivilegeEscalation: false
EOF
```

Deploy a Pod with privilege escalation allowed:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: escalation-allowed-pod
spec:
  containers:
  - name: alpine
    image: alpine:latest
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
      allowPrivilegeEscalation: true
EOF
```

Verify NoNewPrivs setting in both pods:

```bash
echo "Checking NoNewPrivs in no-escalation-pod:"
kubectl exec no-escalation-pod -- cat /proc/1/status | grep NoNewPrivs

echo "\nChecking NoNewPrivs in escalation-allowed-pod:"
kubectl exec escalation-allowed-pod -- cat /proc/1/status | grep NoNewPrivs
```

### Step 4: Avoiding Privileged Containers

Deploy an unprivileged Pod:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: unprivileged-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      privileged: false
EOF
```

Deploy a privileged Pod for comparison:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      privileged: true
EOF
```

Compare and decode effective capabilities:

```bash
echo "Effective capabilities in unprivileged pod:"
UNPRIVILEGED_CAPEFF=$(kubectl exec unprivileged-pod -- cat /proc/1/status | grep CapEff | awk '{print $2}')
echo "CapEff: $UNPRIVILEGED_CAPEFF"
echo "Decoded capabilities:"
capsh --decode=$UNPRIVILEGED_CAPEFF

echo "\nEffective capabilities in privileged pod:"
PRIVILEGED_CAPEFF=$(kubectl exec privileged-pod -- cat /proc/1/status | grep CapEff | awk '{print $2}')
echo "CapEff: $PRIVILEGED_CAPEFF"
echo "Decoded capabilities:"
capsh --decode=$PRIVILEGED_CAPEFF
```

Compare ability to change system time:

```bash
echo "Attempting to change system time in unprivileged pod (should fail):"
kubectl exec unprivileged-pod -- date -s "12:00:00"

echo "\nAttempting to change system time in privileged pod (should succeed):"
kubectl exec privileged-pod -- date -s "12:00:00"
```

### Step 5: Dropping Unnecessary Capabilities

Deploy a Pod with default capabilities:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: default-capabilities-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
EOF
```

Deploy a Pod with dropped capabilities:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dropped-capabilities-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      capabilities:
        drop:
        - NET_RAW
        - SYS_ADMIN
EOF
```

Compare and decode effective capabilities:

```bash
echo "Effective capabilities in default-capabilities-pod:"
DEFAULT_CAPEFF=$(kubectl exec default-capabilities-pod -- cat /proc/1/status | grep CapEff | awk '{print $2}')
echo "CapEff: $DEFAULT_CAPEFF"
echo "Decoded capabilities:"
capsh --decode=$DEFAULT_CAPEFF

echo "\nEffective capabilities in dropped-capabilities-pod:"
DROPPED_CAPEFF=$(kubectl exec dropped-capabilities-pod -- cat /proc/1/status | grep CapEff | awk '{print $2}')
echo "CapEff: $DROPPED_CAPEFF"
echo "Decoded capabilities:"
capsh --decode=$DROPPED_CAPEFF
```

Compare network operations:

```bash
echo "Attempting to ping in default-capabilities-pod (should succeed):"
kubectl exec default-capabilities-pod -- ping -c 1 8.8.8.8

echo "\nAttempting to ping in dropped-capabilities-pod (should fail):"
kubectl exec dropped-capabilities-pod -- ping -c 1 8.8.8.8
```

### Step 6: Using runAsNonRoot

Deploy a Pod with `runAsNonRoot` at the pod level:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: run-as-non-root-pod
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
EOF
```

Attempt to deploy a Pod with `runAsNonRoot` set but with a root user (this should fail):

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: run-as-non-root-failure
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 0
EOF
```

Observe the failure:

```bash
kubectl get pods run-as-non-root-failure
kubectl describe pod run-as-non-root-failure | grep -i "Error"
```

## Verification

The demonstrations show various security context configurations and their effects:

- **User isolation**: Non-root containers run with restricted user IDs (1000, 2000, etc.) instead of root (UID 0)
- **Privilege escalation**: The `NoNewPrivs` flag prevents containers from gaining additional privileges
- **Capabilities**: Privileged containers have full Linux capabilities, while unprivileged containers have a minimal set
- **Network restrictions**: Dropping `NET_RAW` capability prevents raw socket operations like ping
- **Security enforcement**: The `runAsNonRoot: true` setting prevents deployment of containers attempting to run as root

## Cleanup

Remove all created Pods:

```bash
kubectl delete pod insecure-pod secure-pod pod-level-security-context container-level-security-context no-escalation-pod escalation-allowed-pod unprivileged-pod privileged-pod default-capabilities-pod dropped-capabilities-pod run-as-non-root-pod run-as-non-root-failure
```
