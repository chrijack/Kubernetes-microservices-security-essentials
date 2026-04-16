# Lesson 1.1 — Resource Isolation with ResourceQuota and LimitRange

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Learn how to implement resource isolation in Kubernetes using ResourceQuota and LimitRange to control CPU and memory consumption across namespaces and individual containers.

## Prerequisites

- Kubernetes cluster with kubectl access
- Default namespace available

## Steps

### Step 1: Create a ResourceQuota

Apply the ResourceQuota to establish hard limits on total resource consumption in the namespace:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: default
spec:
  hard:
    requests.cpu: "1"                   # Total CPU requests across all pods
    limits.cpu: "2"                     # Total CPU limits across all pods
    requests.memory: 1Gi                # Total memory requests across all pods
    limits.memory: 2Gi                  # Total memory limits across all pods
    pods: "10"                          # Maximum number of pods that can be created
    requests.storage: "100Gi"           # Total storage requests across all PVCs
EOF
```

Expected output:
```
resourcequota/compute-resources created
```

### Step 2: Create a LimitRange

Apply the LimitRange to set per-container defaults and constraints:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: default
spec:
  limits:
  - default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: 500m                # Maximum CPU limit per container
      memory: 512Mi            # Maximum memory limit per container
    min:
      cpu: 50m                 # Minimum CPU request per container
      memory: 64Mi             # Minimum memory request per container
    maxLimitRequestRatio:
      cpu: 2                   # Maximum ratio of limit to request for CPU
      memory: 4                # Maximum ratio of limit to request for memory
    type: Container
EOF
```

Expected output:
```
limitrange/limit-range created
```

### Step 3: Create a Pod with Resource Requests and Limits

Create a pod that explicitly specifies its own resource requests and limits:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo-pod
spec:
  containers:
  - name: demo-container
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
EOF
```

Expected output:
```
pod/resource-demo-pod created
```

### Step 4: Verify Resource Quota Usage

Check how the ResourceQuota is being consumed by the pods in the namespace:

```bash
kubectl describe resourcequota compute-resources
```

Expected output (values may vary):
```
Name:            compute-resources
Namespace:       default
Resource         Used  Hard
--------         ----  ----
limits.cpu       500m  2
limits.memory    128Mi 2Gi
pods             1     10
requests.cpu     250m  1
requests.memory  64Mi  1Gi
```

Verify the pod's actual resource allocation:

```bash
kubectl describe pod resource-demo-pod
```

Expected output (partial):
```
...
Containers:
  demo-container:
    ...
    Limits:
      cpu:     500m
      memory:  128Mi
    Requests:
      cpu:     250m
      memory:  64Mi
...
```

### Step 5: Demonstrate ResourceQuota Enforcement

Attempt to create a pod that exceeds the namespace quota:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: large-pod
spec:
  containers:
  - name: large-container
    image: nginx
    resources:
      requests:
        memory: "1.5Gi"
        cpu: "1.5"
      limits:
        memory: "2Gi"
        cpu: "2"
EOF
```

Expected output:
```
Error from server (Forbidden): error when creating "STDIN": pods "large-pod" is forbidden: exceeded quota: compute-resources, requested: limits.cpu=2,limits.memory=2Gi,requests.cpu=1500m,requests.memory=1536Mi, used: limits.cpu=500m,limits.memory=128Mi,requests.cpu=250m,requests.memory=64Mi, limited: limits.cpu=2,limits.memory=2Gi,requests.cpu=1,requests.memory=1Gi
```

### Step 6: Demonstrate LimitRange Defaults

Create a pod without specifying resources to see LimitRange defaults applied automatically:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: default-pod
spec:
  containers:
  - name: default-container
    image: nginx
EOF
```

Expected output:
```
pod/default-pod created
```

Verify that the LimitRange defaults were automatically applied to the pod:

```bash
kubectl describe pod default-pod
```

Expected output (partial):
```
...
Containers:
  default-container:
    ...
    Limits:
      cpu:     200m
      memory:  256Mi
    Requests:
      cpu:     100m
      memory:  128Mi
...
```

## Verification

- ResourceQuota prevents pods from consuming more resources than allocated to the namespace
- LimitRange automatically applies default resources to pods that don't specify them
- Both mechanisms enforce minimum and maximum resource constraints

## Cleanup

Remove all demo resources:

```bash
kubectl delete resourcequota compute-resources
kubectl delete limitrange limit-range
kubectl delete pod resource-demo-pod default-pod
```

Expected output:
```
resourcequota "compute-resources" deleted
limitrange "limit-range" deleted
pod "resource-demo-pod" deleted
pod "default-pod" deleted
```
