# Lesson 4.2 — Implementing Pod-to-Pod Encryption with Cilium

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Set up and verify pod-to-pod encryption using Cilium with WireGuard in a Kubernetes cluster. This demonstrates how to secure all inter-pod communications within the cluster using high-performance encryption.

## Prerequisites

- A Kubernetes cluster with at least two nodes
- `kubectl` configured to access the cluster
- `cilium` CLI tool installed

## Steps

### Step 1: Install Cilium with Encryption

Install Cilium with WireGuard encryption enabled:

```bash
cilium install --version 1.16.1 \
--set encryption.enabled=true \
--set encryption.type=wireguard
```

This installs Cilium with encryption enabled, using WireGuard to secure all pod-to-pod communications within the cluster.

### Step 2: Verify Cilium Installation and Encryption

Verify that all Cilium pods are running and that encryption is enabled:

```bash
kubectl get pods -n kube-system -l k8s-app=cilium
cilium status --wait
```

The first command checks that all Cilium pods are running successfully. The second command waits for Cilium to be fully operational and confirms the deployment status.

### Step 3: Create a Namespace for Traffic Generation

Create an isolated namespace for the traffic generation resources:

```bash
kubectl create namespace traffic-gen
```

### Step 4: Create a Deployment for Traffic Generation

Create a deployment that generates traffic between pods:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traffic-generator
  namespace: traffic-gen
spec:
  replicas: 2
  selector:
    matchLabels:
      app: traffic-gen
  template:
    metadata:
      labels:
        app: traffic-gen
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      - name: traffic-generator
        image: busybox
        command: ["/bin/sh"]
        args: ["-c", "while true; do wget -q -O- http://traffic-service; sleep 1; done"]
EOF
```

This deployment creates two replicas in the `traffic-gen` namespace. Each pod runs:
- **Nginx container:** Listens on port 80 to serve HTTP requests
- **Busybox traffic generator:** Continuously sends HTTP requests to the Nginx service (`traffic-service`)

### Step 5: Create a Service to Expose Nginx Pods

Create a service to expose the Nginx containers:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: traffic-service
  namespace: traffic-gen
spec:
  selector:
    app: traffic-gen
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF
```

This service exposes the Nginx containers on port 80, allowing traffic-generator containers to send HTTP requests to them.

### Step 6: Verify Pod Distribution Across Nodes

Check the distribution of pods across nodes to ensure traffic flows between nodes:

```bash
kubectl get pods -n traffic-gen -o wide
```

You should see each pod running on a different node, which is essential for testing pod-to-pod encryption across node boundaries.

### Step 7: Verify Encryption is Enabled

Confirm that Cilium encryption is active:

```bash
kubectl -n kube-system exec -ti ds/cilium -- cilium status | grep Encryption
```

This verifies that WireGuard encryption is enabled and functioning.

### Step 8: Analyze Traffic Patterns Using tcpdump

Examine the encrypted traffic on the WireGuard interface:

```bash
kubectl -n kube-system exec -ti ds/cilium -- bash

cilium-dbg debuginfo --output json | jq .encryption

apt-get update
apt-get -y install tcpdump

tcpdump -n -i cilium_wg0
```

This enables detailed inspection of encrypted traffic flows. You can observe and analyze the actual network packets on the WireGuard interface (`cilium_wg0`) to confirm that traffic between pods is encrypted.

## Verification

- All Cilium pods are running in the `kube-system` namespace
- Cilium status shows encryption as enabled
- The `traffic-generator` deployment has two replicas running on different nodes
- Traffic is flowing through the encrypted WireGuard interface
- tcpdump output shows encrypted packets on `cilium_wg0`

## Cleanup

Remove the resources created during this lesson:

```bash
kubectl delete namespace traffic-gen
```
