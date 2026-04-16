# Lesson 5.3 — Enforcing Software Supply Chain Security with Validating Admission Policy

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Enforce Software Supply Chain security by configuring a ValidatingWebhookConfiguration that validates container image sources. This policy ensures only images from trusted registries are allowed in the cluster.

## Prerequisites

- Webhook configuration from Lesson 5.2 is already in place and functional
- The webhook validation server is configured with registry whitelisting (e.g., `--registry-whitelist=docker.io,registry.k8s.io`)
- Administrative privileges in the Kubernetes cluster

## Steps

### Step 1: Create a Validation Policy

Create a ValidatingWebhookConfiguration that enforces registry whitelisting for all new pod creation requests.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: image-policy-webhook
webhooks:
- name: image-bouncer-webhook.com
  clientConfig:
    service:
      name: image-bouncer-webhook
      namespace: default
      path: "/validate"
    caBundle: $(cat /etc/kubernetes/pki/webhook-server.crt | base64 | tr -d "\n")
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  failurePolicy: Fail
  admissionReviewVersions: ["v1"]
EOF
```

### Step 2: Test Rejection with Untrusted Registry

Deploy a pod using an image from an untrusted registry to verify the webhook rejects it.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: untrusted-image-test
  labels:
    tier: untrusted-image
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: untrusted-image
  template:
    metadata:
      labels:
        tier: untrusted-image
    spec:
      containers:
      - name: untrusted-image
        image: badregistry.io/malicious-image:latest
EOF
```

Observe the rejection:

```bash
kubectl describe replicaset untrusted-image-test
```

Expected error message:
```
Error creating: pods "untrusted-image-test-xxxxx" is forbidden: Image 'badregistry.io/malicious-image:latest' is not allowed by the registry whitelist.
```

### Step 3: Deploy with Trusted Registry Image

Clean up the failed deployment and redeploy using an image from a trusted registry.

Delete the failed ReplicaSet:

```bash
kubectl delete rs untrusted-image-test
```

Deploy a pod using a trusted registry image:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: trusted-image-test
  labels:
    tier: trusted-image
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: trusted-image
  template:
    metadata:
      labels:
        tier: trusted-image
    spec:
      containers:
      - name: trusted-image
        image: registry.k8s.io/nginx:1.19
EOF
```

Verify successful pod creation:

```bash
kubectl describe replicaset trusted-image-test
kubectl get pods | grep trusted-image-test
```

The pod should be created successfully with the trusted image.

## Verification

To verify the webhook is functioning correctly:

View webhook logs to see registry whitelist enforcement in action:

```bash
kubectl logs -f deployment/image-bouncer-webhook
```

## Cleanup

Remove all demonstration resources:

```bash
kubectl delete rs trusted-image-test
kubectl delete ValidatingWebhookConfiguration image-policy-webhook
```

To demonstrate behavior when the webhook backend is unavailable, you can adjust the `failurePolicy` to `Ignore` in the ValidatingWebhookConfiguration before cleanup.
