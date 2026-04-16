# Lesson 6.4 — Using Kyverno, cosign, and trivy to validate SBOM and vulnerability attestation

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Demonstrate how to use Kyverno, Cosign, and Trivy together to validate Software Bill of Materials (SBOM) and vulnerability attestation for container images in a Kubernetes cluster.

## Prerequisites

- Kubernetes 1.30 cluster running
- Helm installed
- Cosign installed
- Trivy installed
- Docker Hub credentials

## Steps

### Step 1: Install Kyverno Using Helm

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace \
  --set admissionController.replicas=1 \
  --set backgroundController.replicas=1 \
  --set cleanupController.replicas=1 \
  --set reportsController.replicas=1
```

### Step 2: Generate a Cosign Key Pair

```bash
cosign generate-key-pair
```

Log in to Docker Hub using Cosign:

```bash
cosign login -u <your-dockerhub-username> -p <your-docker-pat> docker.io
```

### Step 3: Create a Kubernetes Secret for the Public Key

Store the Cosign public key in a Kubernetes Secret:

```bash
kubectl create secret generic cosign-public-key \
  --from-file=cosign.pub=cosign.pub \
  -n kyverno
```

Add a Docker registry secret to the `kyverno` namespace:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<your-dockerhub-username> \
  --docker-password=<your-docker-pat> \
  --docker-email=chrijack@cisco.com \
  --namespace=kyverno
```

Add the same registry secret to the `default` namespace:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<your-dockerhub-username> \
  --docker-password=<your-docker-pat> \
  --docker-email=chrijack@cisco.com \
  --namespace=default
```

Update the Kyverno admission controller to use the image pull secret:

```bash
kubectl edit deploy -n kyverno kyverno-admission-controller
```

Add the following under the `containers` section:

```yaml
        - --imagePullSecrets=regcred
```

### Step 4: Scan the Container Image with Trivy and Generate an SBOM

Use Trivy to scan your container image for vulnerabilities and generate an SBOM (Software Bill of Materials):

```bash
trivy image --format cosign-vuln --output scan.json docker.io/<your-dockerhub-username>/cks@sha256:271c1e875b433379a3b95becd3a95b241a1803a5f2a9862663e95a01a3e15c28
```

### Step 5: Create an Attestation with Cosign

Sign the image:

```bash
cosign sign --key cosign.key docker.io/<your-dockerhub-username>/cks@sha256:271c1e875b433379a3b95becd3a95b241a1803a5f2a9862663e95a01a3e15c28
```

Create the attestation:

```bash
cosign attest --key cosign.key --predicate scan.json --type vuln docker.io/<your-dockerhub-username>/cks@sha256:271c1e875b433379a3b95becd3a95b241a1803a5f2a9862663e95a01a3e15c28
```

### Step 6: Verify the Attestation

Verify the attestation using the Cosign public key:

```bash
cosign --key cosign.pub verify-attestation --type vuln <your-dockerhub-username>/cks@sha256:271c1e875b433379a3b95becd3a95b241a1803a5f2a9862663e95a01a3e15c28 | more
```

Decode the attestation payload:

```bash
cosign verify-attestation --key cosign.pub --type vuln <your-dockerhub-username>/cks@sha256:271c1e875b433379a3b95becd3a95b241a1803a5f2a9862663e95a01a3e15c28 | jq -r .payload | base64 -d | jq .
```

### Step 7: Define and Apply a Kyverno Policy

Create a Kyverno policy file (`vuln-attestation.yaml`) to enforce image attestation verification:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-vulnerabilities
spec:
  validationFailureAction: Enforce
  background: false
  webhookTimeoutSeconds: 30
  failurePolicy: Fail
  rules:
    - name: checking-vulnerability-scan-not-older-than-one-hour
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - imageReferences:
        - "*"
        attestations:
        - type: https://cosign.sigstore.dev/attestation/vuln/v1
          conditions:
          - all:
            - key: "{{ time_since('','{{ metadata.scanFinishedOn }}', '') }}"
              operator: LessThanOrEquals
              value: "24h"
          attestors:
          - count: 1
            entries:
            - keys:
                publicKeys: "k8s://kyverno/cosign-public-key"
```

Apply the policy:

```bash
kubectl apply -f vuln-attestation.yaml
```

### Step 8: Test the Policy

Deploy a Pod using the image with the SBOM attestation:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: app-signed
  name: app-signed
spec:
  containers:
  - image: <your-dockerhub-username>/cks@sha256:271c1e875b433379a3b95becd3a95b241a1803a5f2a9862663e95a01a3e15c28
    name: app-signed
  imagePullSecrets:
  - name: regcred
EOF
```

Attempt to deploy an unsigned image to verify the policy blocks it:

```bash
kubectl run app-unsigned --image=nginx
```

## Verification

- Confirm Kyverno pods are running: `kubectl get pods -n kyverno`
- Verify the Cosign keys were created: `ls -la cosign.*`
- Check secrets were created: `kubectl get secrets -n kyverno`
- Confirm the signed pod deployed successfully
- Confirm the unsigned pod deployment was blocked by the Kyverno policy

## Cleanup

```bash
kubectl delete pod app-signed
kubectl delete secret cosign-public-key -n kyverno
kubectl delete secret regcred -n kyverno
kubectl delete secret regcred -n default
kubectl delete policy check-vulnerabilities
helm uninstall kyverno -n kyverno
kubectl delete namespace kyverno
```
