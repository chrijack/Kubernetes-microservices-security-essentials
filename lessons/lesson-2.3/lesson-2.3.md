# Lesson 2.3 — Using Secrets in Pods

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Learn how to use secrets in Kubernetes pods through multiple mounting strategies including environment variables and volume mounts.

## Prerequisites

- A Kubernetes cluster with kubectl configured
- An existing secret named `my-secret` with keys `username` and `password`

## Steps

### Step 1: Using Secrets as Environment Variables

Create a pod that mounts specific secret keys as environment variables:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: my-container
    image: nginx
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: username
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: password
EOF
```

Verify the pod and environment variables:

```bash
kubectl get pod secret-env-pod
kubectl logs secret-env-pod
kubectl exec -it secret-env-pod -- sh
env
```

### Step 2: Using Secrets as Volumes

Create a pod that mounts secrets as a filesystem volume:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: my-container
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: "/etc/secret-path"
  volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
EOF
```

Verify the mounted secrets:

```bash
kubectl exec -it secret-volume-pod -- ls /etc/secret-path
kubectl exec -it secret-volume-pod -- cat /etc/secret-path/username
kubectl exec -it secret-volume-pod -- cat /etc/secret-path/password
```

### Step 3: Using All Secret Keys as Environment Variables

Create a pod that mounts all keys from a secret as environment variables:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod-all
spec:
  containers:
  - name: my-container
    image: nginx
    envFrom:
    - secretRef:
        name: my-secret
EOF
```

### Step 4: Using Subpath with Secret Volumes

Create a pod that mounts a single key from a secret to a specific file path:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-single-file-pod
spec:
  containers:
  - name: my-container
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: "/etc/secret-file"
      subPath: username
  volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
EOF
```

## Verification

- All pods should reach `Running` state: `kubectl get pods`
- Environment variables should be visible in pods created with `env` or `envFrom`
- Volume-mounted secrets should be accessible as files at their mount paths
- Subpath mounts should show only the specified key at the mount location

## Cleanup

Remove all pods created during this lesson:

```bash
kubectl delete pod secret-env-pod secret-volume-pod secret-env-pod-all secret-single-file-pod
```
