# Lesson 22.5 — Policy Enforcement: Validating Admission Policy

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Configure a Kubernetes Validating Admission Policy to enforce container image validation, requiring that image tags must not be "latest" and only whitelisted registries are allowed. This builds on Lesson 22.4, replacing the ImagePolicyWebhook mechanism with the newer Validating Admission Policy approach in Kubernetes 1.30+.

## Prerequisites

- Kubernetes 1.30 cluster running (local or remote)
- Administrative privileges in the cluster
- Existing webhook service from Lesson 22.4, including certificate and TLS setup

## Steps

### Step 1: Define the ValidatingAdmissionPolicy

Create a ValidatingAdmissionPolicy resource that enforces container image validation rules.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: image-tag-policy
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE"]
  validations:
    - expression: "object.spec.containers.all(c, !c.image.endsWith(':latest'))"
      message: "Images with 'latest' tag are not allowed."
    - expression: "object.spec.containers.all(c, ['docker.io', 'registry.k8s.io'].exists(r, c.image.startsWith(r)))"
      message: "Only images from docker.io and registry.k8s.io are allowed."
EOF
```

This policy enforces two rules:
- Rejects pods where any container image uses the "latest" tag
- Allows only images from the specified whitelisted registries: `docker.io` and `registry.k8s.io`

### Step 2: Create PolicyBinding to Enforce the Policy

Bind the policy to all pod creation requests across the cluster.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: image-tag-policy-binding
spec:
  policyName: image-tag-policy
  matchResources:
    namespaceSelector: {}
    objectSelector: {}
  validationActions: ["Deny"]
EOF
```

This binding ensures the policy applies cluster-wide to all pod creation requests with a Deny action.

### Step 3: Test the ValidatingAdmissionPolicy with Forbidden Image

Deploy a pod with a forbidden image using the "latest" tag to verify the policy blocks it.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-latest
  labels:
    tier: nginx-latest
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: nginx-latest
  template:
    metadata:
      labels:
        tier: nginx-latest
    spec:
      containers:
      - name: nginx-latest
        image: nginx:latest
EOF
```

Check the status and observe the rejection:

```bash
kubectl describe replicaset nginx-latest
```

Expected error message:
```
Error creating: pods "nginx-latest-xxxxx" is forbidden: Images with 'latest' tag are not allowed.
```

### Step 4: Deploy with Allowed Image and Registry

Delete the forbidden ReplicaSet and deploy a pod with a whitelisted image and specific tag.

```bash
kubectl delete rs nginx-latest
```

Deploy the allowed image:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-allowed
  labels:
    tier: nginx-allowed
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: nginx-allowed
  template:
    metadata:
      labels:
        tier: nginx-allowed
    spec:
      containers:
      - name: nginx-allowed
        image: docker.io/nginx:1.19
EOF
```

Verify successful pod creation:

```bash
kubectl get pods | grep nginx-allowed
kubectl describe replicaset nginx-allowed
```

## Verification

The policy has been successfully configured when:
- Pods with "latest" tags are rejected with the policy violation message
- Pods with images from whitelisted registries and specific version tags are created successfully
- The ValidatingAdmissionPolicy and ValidatingAdmissionPolicyBinding are active in the cluster

## Cleanup

Remove the ValidatingAdmissionPolicy and its binding:

```bash
kubectl delete ValidatingAdmissionPolicy image-tag-policy
kubectl delete ValidatingAdmissionPolicyBinding image-tag-policy-binding
```
