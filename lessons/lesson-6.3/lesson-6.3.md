# Lesson 6.3 — Scanning with Trivy Operator

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Deploy and use the Trivy Operator to scan a Kubernetes cluster for container image vulnerabilities and manifest configuration issues.

## Prerequisites

- Kubernetes 1.30 cluster is running
- Helm is installed and configured
- Trivy CLI is installed locally

## Steps

### Step 1: Add Trivy Operator Helm Repository

```bash
helm repo add aqua https://aquasecurity.github.io/helm-charts/
helm repo update
```

### Step 2: Install Trivy Operator

```bash
helm install trivy-operator aqua/trivy-operator --namespace trivy-system --create-namespace
```

Verify the installation:

```bash
helm list -n trivy-system
```

### Step 3: Verify Trivy Operator Installation

Check that the Trivy Operator pods are running:

```bash
kubectl get pods -n trivy-system
```

### Step 4: Deploy a Sample NGINX Pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-sample
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.19.10
EOF
```

### Step 5: Check Vulnerability Reports

Wait a few moments for the Trivy Operator to scan the pod, then view the vulnerability reports:

```bash
kubectl get vulnerabilityreports -n default
```

To see detailed vulnerability information:

```bash
kubectl describe vulnerabilityreports <vulnerability-resource-name> -n default
```

Replace `<vulnerability-resource-name>` with the actual name from the previous command.

### Step 6: Review Configuration Audit Reports

View all configuration audit reports across the cluster:

```bash
kubectl get configauditreports -A
```

To see details of a specific configuration issue:

```bash
kubectl describe configauditreports <config-audit-report-name> -n <namespace>
```

### Step 7: Deploy a Redis Pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: redis-sample
  namespace: default
spec:
  containers:
  - name: redis
    image: redis:6.0.9
EOF
```

### Step 8: Review Reports for New Pod

View vulnerability and configuration reports for all pods:

```bash
kubectl get vulnerabilityreports -n default
kubectl describe vulnerabilityreports <vulnerability-resource-name> -n default

kubectl get configauditreports -A
kubectl describe configauditreports <config-audit-report-name> -n default
```

## Verification

To verify the Trivy Operator is working correctly, use the Trivy CLI to scan the Kubernetes cluster:

```bash
trivy k8s --include-namespaces kube-system --report summary
trivy k8s --include-namespaces kube-system --report all
```

## Cleanup

Remove all resources created during the demonstration:

```bash
kubectl delete pod nginx-sample -n default
kubectl delete pod redis-sample -n default
helm uninstall trivy-operator -n trivy-system
kubectl delete namespace trivy-system
```
