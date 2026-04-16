# Lesson 2.1 — Understanding Kubernetes Secrets

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Understand the purpose, types, and security considerations of Kubernetes secrets.

## Prerequisites

- Kubernetes cluster access
- kubectl CLI configured
- Basic understanding of Kubernetes objects

## Steps

### Step 1: Introduction to Secrets

**What are Kubernetes secrets?**
Secrets are Kubernetes objects designed to store and manage sensitive information. They provide a mechanism to store passwords, OAuth tokens, SSH keys, and similar data.

**Why secrets are needed:**
- Centralized management of sensitive data
- Prevent hardcoding sensitive information in pods or container images
- Secrets can be injected as environment variables or mounted as volumes
- Enable fine-grained access control through RBAC

**Security considerations:**
- Secrets are stored in etcd and are only base64 encoded by default (not encrypted)
- Importance of encrypting secrets at rest in etcd
- Use network policies to restrict access to etcd
- Apply RBAC to control access to secrets

### Step 2: Types of Secrets

**Secret types and their purposes:**

- **Opaque**: General-purpose secret storage (default type)
- **kubernetes.io/service-account-token**: Token for service accounts
- **kubernetes.io/dockercfg** & **kubernetes.io/dockerconfigjson**: Docker registry credentials
- **kubernetes.io/ssh-auth**: SSH authentication
- **kubernetes.io/tls**: TLS public/private key pair and certificate
- **kubernetes.io/basic-auth**: Basic authentication (username and password)
- **kubernetes.io/bootstraptoken**: Bootstrap process for new Kubernetes nodes

**Demonstrate creating different types of secrets:**

Opaque secret:
```bash
kubectl create secret generic my-secret --from-literal=username=admin --from-literal=password=t0p-Secret
```

TLS secret:
```bash
kubectl create secret tls my-tls-secret --cert=path/to/tls.cert --key=path/to/tls.key
```

Docker registry secret:
```bash
kubectl create secret docker-registry my-registry-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

## Verification

Verify that secrets have been created:
```bash
kubectl get secrets
kubectl describe secret <secret-name>
```

## Cleanup

Remove created secrets:
```bash
kubectl delete secret my-secret my-tls-secret my-registry-secret
```
