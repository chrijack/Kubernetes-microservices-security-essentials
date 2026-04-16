# Lesson 20.2 — Sandboxed Pods: Setup and Verification

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Set up and verify the functionality of gVisor and Kata Containers in Kubernetes 1.30.

## Prerequisites

- Kubernetes 1.30 cluster running
- Root or sudo access on worker nodes
- Virtualization support enabled (KVM)

## Steps

### Step 1: Check Virtualization Support

Install the `cpu-checker` package to verify KVM capability:

```bash
sudo apt-get update && sudo apt-get install -y cpu-checker
```

Check if virtualization is enabled and KVM is supported:

```bash
kvm-ok
```

Expected output should confirm that "KVM acceleration can be used."

Check if KVM modules are loaded:

```bash
lsmod | grep kvm
```

You should see `kvm_intel` or `kvm_amd` in the output, indicating that KVM modules are loaded.

### Step 2: Install and Configure gVisor

Download and install gVisor runtime:

```bash
( set -e; ARCH=$(uname -m); URL=https://storage.googleapis.com/gvisor/releases/release/latest/${ARCH}; wget ${URL}/runsc ${URL}/runsc.sha512 ${URL}/containerd-shim-runsc-v1 ${URL}/containerd-shim-runsc-v1.sha512; sha512sum -c runsc.sha512 -c containerd-shim-runsc-v1.sha512; rm -f *.sha512; chmod a+rx runsc containerd-shim-runsc-v1; sudo mv runsc containerd-shim-runsc-v1 /usr/local/bin; )
```

Validate the gVisor installation:

```bash
runsc --version
containerd-shim-runsc-v1 -v
```

Configure containerd to use gVisor runtime:

```bash
cat <<EOF | sudo tee /etc/containerd/config.toml
version = 2
[plugins."io.containerd.runtime.v1.linux"]
  shim_debug = true
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"
EOF
```

Restart containerd:

```bash
sudo systemctl restart containerd
```

### Step 3: Create RuntimeClass for gVisor

Apply the RuntimeClass configuration for gVisor:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
EOF
```

### Step 4: Deploy Sample Pod with gVisor

Create and deploy a pod using the gVisor RuntimeClass:

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
EOF
```

Verify the pod is running:

```bash
kubectl get pod nginx-gvisor -o wide
```

### Step 5: Install and Configure Kata Containers

Apply the RBAC configuration for Kata Containers:

```bash
kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-rbac/base/kata-rbac.yaml
```

Deploy Kata Containers:

```bash
kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-deploy/base/kata-deploy.yaml
```

Wait for Kata Containers deployment to complete:

```bash
kubectl -n kube-system wait --timeout=10m --for=condition=Ready -l name=kata-deploy pod
```

Verify Kata Containers installation by checking the deployed pods:

```bash
kubectl -n kube-system get pods -l name=kata-deploy
```

### Step 6: Apply RuntimeClasses for Kata Containers

Apply the Kata RuntimeClass configurations:

```bash
kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/runtimeclasses/kata-runtimeClasses.yaml
```

### Step 7: Deploy Sample Workload with Kata QEMU

Deploy a test workload using the Kata QEMU runtime:

```bash
kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/examples/test-deploy-kata-qemu.yaml
```

## Verification

Verify RuntimeClasses are registered:

```bash
kubectl get runtimeclass
```

You should see RuntimeClasses for `gvisor`, `kata-qemu`, and `kata-fc` listed.

Check that all nodes support the new runtimes:

```bash
kubectl get nodes --show-labels
```

Verify that Kata deployment pods completed successfully:

```bash
kubectl -n kube-system logs kata-deploy
```

## Cleanup

Remove sample pods and deployments:

```bash
kubectl delete pod nginx-gvisor
kubectl delete -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/examples/test-deploy-kata-qemu.yaml
```

To remove Kata Containers entirely:

```bash
kubectl delete -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-deploy/base/kata-deploy.yaml
kubectl delete -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-rbac/base/kata-rbac.yaml
```

To remove gVisor, uninstall from the system:

```bash
sudo apt-get remove -y runsc
sudo rm -f /usr/local/bin/runsc /usr/local/bin/containerd-shim-runsc-v1
```
