# Lesson 19.2 — Creating and Using Secrets

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Learn how to create and use secrets in Kubernetes using both imperative and declarative approaches, and understand immutable secrets and StringData features.

## Prerequisites

- Kubernetes cluster running
- kubectl configured to access the cluster
- Basic understanding of Kubernetes resources

## Steps

### Step 1: Create a Secret (Imperative Approach)

Create secret files and use kubectl to generate a secret from them:

```bash
echo -n 'user' > username.txt
echo -n 'password' > password.txt
```

```bash
kubectl create secret generic my-secret --from-file=username.txt --from-file=password.txt
```

### Step 2: Create a Secret (Declarative Approach)

Base64 encode your secret values:

```bash
echo -n 'user' | base64
echo -n 'password' | base64
```

Create and apply a YAML manifest:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: dXNlcg==
  password: cGFzc3dvcmQ=
EOF
```

### Step 3: Verify the Secret

Check the created secret:

```bash
kubectl get secrets my-secret -o yaml
```

### Step 4: Create an Immutable Secret

Immutable secrets prevent accidental updates to secret data. This feature was introduced in Kubernetes 1.21.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: immutable-secret
type: Opaque
data:
  username: YWRtaW4=
  password: dDBwLVNlY3JldA==
immutable: true
EOF
```

### Step 5: Use StringData for Plain Text Values

StringData allows you to provide plain text values that are automatically base64 encoded, as opposed to the `data` field which requires pre-encoded values.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: string-data-secret
type: Opaque
stringData:
  config.yaml: |
    apiUrl: "https://my-api.com"
    username: "user"
    password: "pass"
EOF
```

## Verification

Verify all created secrets:

```bash
kubectl get secrets
kubectl get secrets my-secret -o yaml
kubectl get secrets immutable-secret -o yaml
kubectl get secrets string-data-secret -o yaml
```

## Cleanup

Delete the secrets:

```bash
kubectl delete secret my-secret immutable-secret string-data-secret
rm -f username.txt password.txt
```
