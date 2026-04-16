# Lesson 20.4 — Using Kata Containers

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Demonstrate the use of Kata Containers for enhanced security and isolation in Kubernetes 1.30.

## Prerequisites

- Kata Containers are installed and configured (as done in Sub-lesson 20.2).
- RuntimeClasses for Kata Containers are already created and validated (as done in Sub-lesson 20.2).

```bash
kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/runtimeclasses/kata-runtimeClasses.yaml
```

Verify the RuntimeClass configuration:

```bash
kubectl describe runtimeclass kata-qemu
```

## Steps

### Step 1: Deploy a Pod Using Kata Containers (kata-qemu)

Deploy a sample Nginx workload using Kata Containers:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-kata-qemu
spec:
  runtimeClassName: kata-qemu
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
EOF
```

Verify the pod deployment:

```bash
kubectl get pod nginx-kata-qemu -o wide
```

The pod should be listed as `Running` with the `RuntimeClassName` set to `kata-qemu`.

### Step 2: Test Kata Containers Pod Isolation

Access the pod and verify virtualization:

```bash
kubectl exec -it nginx-kata-qemu -- /bin/sh
```

Check for hypervisor presence:

```bash
dmesg | grep -i hypervisor
```

The output should indicate that the pod is running inside a virtualized environment.

Check the kernel version:

```bash
uname -r
```

The kernel version should differ from the host's kernel, confirming isolation within a lightweight VM.

### Step 3: Performance Comparison

Run a simple benchmark in both pod types:

```bash
kubectl exec nginx-gvisor -- dd if=/dev/zero of=/dev/null count=1000000
kubectl exec nginx-kata-qemu -- dd if=/dev/zero of=/dev/null count=1000000
```

Compare the performance characteristics between the two isolation mechanisms.

## Verification

- Pod is running with `RuntimeClassName` set to `kata-qemu`
- Hypervisor presence is detected within the pod
- Kernel version differs from the host system
- Pod file system is minimal and isolated with no access to host file systems

## Cleanup

Remove the test pod:

```bash
kubectl delete pod nginx-kata-qemu
```
