# Lesson 20.3 — Using gVisor

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective
Demonstrate the use of gVisor for container isolation in Kubernetes 1.30.

## Prerequisites
- gVisor is installed and configured (as done in Sub-lesson 20.2).
- RuntimeClass for gVisor is already created and validated (as done in Sub-lesson 20.2).

## Steps

### Step 1: Deploy a Pod Using gVisor RuntimeClass

Create and deploy the RuntimeClass for gVisor:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
EOF
```

Create and deploy a pod using the gVisor runtime:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-gvisor
spec:
  runtimeClassName: gvisor
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
EOF
```

Verify the pod deployment:

```bash
kubectl get pod nginx-gvisor -o wide
```

**Expected Output:**
The pod should be listed as `Running` with the `RuntimeClassName` set to `gvisor`.

### Step 2: Test gVisor Pod Isolation

Exec into the gVisor pod:

```bash
kubectl exec -it nginx-gvisor -- /bin/sh
```

Check the kernel version inside the container:

```bash
uname -r
```

Compare with the host kernel version:

```bash
uname -r
```

**Expected Output:**
The kernel version should reflect gVisor's user-space kernel handling of system calls, which may differ from the host kernel.

## Verification

Confirm that:
- The pod is running with the gVisor runtime class
- The pod can execute commands and display kernel information
- Container isolation is functioning through gVisor

## Cleanup

Remove the test pod:

```bash
kubectl delete pod nginx-gvisor
```

Optionally, remove the gVisor RuntimeClass:

```bash
kubectl delete runtimeclass gvisor
```
