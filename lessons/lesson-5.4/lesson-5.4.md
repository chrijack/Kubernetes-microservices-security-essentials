# Lesson 5.4 — Policy Enforcement: ImagePolicyWebhook

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective
Configure an ImagePolicyWebhook admission controller in Kubernetes 1.30 to reject pods using images with the `:latest` tag and enforce stricter image policies.

## Prerequisites
- Kubernetes 1.30 cluster running (local or remote)
- Administrative privileges in the cluster
- Webhook admission controller enabled on the API server

## Steps

### Step 1: Set Up Host DNS

Ensure the ImagePolicyWebhook can be resolved by the Kubernetes cluster by updating the `/etc/hosts` file on all nodes.

**Update `/etc/hosts` File:**
```bash
sudo nano /etc/hosts
127.0.0.1 image-bouncer-webhook
```
This makes the service resolvable within the Kubernetes cluster as `image-policy-webhook.default.svc`.

---

### Step 2: Generate Certificates

Generate self-signed certificates for securing communication between the API server and the webhook backend.

**Generate the Certificate and Key:**
```bash
openssl genrsa -out webhook-server.key 2048

openssl req -new -key webhook-server.key \
  -subj "/CN=system:node:image-bouncer-webhook.default.pod.cluster.local/O=system:nodes" \
  -addext "subjectAltName=DNS:image-bouncer-webhook,DNS:image-bouncer-webhook.default.svc,DNS:image-bouncer-webhook.default.svc.cluster.local" \
  -out webhook-server.csr
```

**Check the CSR (Optional):**
```bash
openssl req -in webhook-server.csr -text -noout
```

**Encode CSR to Base64 and Export as Environment Variable:**
```bash
export SIGNING_REQUEST=$(cat webhook-server.csr | base64 | tr -d "\n")
```

**Create a CertificateSigningRequest Manifest:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: image-bouncer-webhook.default
spec:
  request: $SIGNING_REQUEST
  signerName: kubernetes.io/kubelet-serving
  expirationSeconds: 864000  # ten days
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```

**Approve and Retrieve the Signed Certificate:**
```bash
kubectl get csr
kubectl certificate approve image-bouncer-webhook.default
kubectl get csr image-bouncer-webhook.default -o=jsonpath={.status.certificate} | base64 --decode > webhook-server.crt
```

**Place the Certificate in the `/etc/kubernetes/pki` Directory:**
```bash
sudo cp ~/webhook-server.crt /etc/kubernetes/pki/
```

---

### Step 3: Create TLS Secret in Kubernetes

The webhook backend needs to access the TLS certificate, so store the certificate and key as a secret in Kubernetes.

**Create a TLS Secret:**
```bash
kubectl create secret tls tls-image-bouncer-webhook --cert=webhook-server.crt --key=webhook-server.key
```

---

### Step 4: Create ImagePolicyWebhook Configuration Files

**Create the Admission Configuration:**
```bash
sudo nano /etc/kubernetes/pki/admission_configuration.yaml
```
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/pki/admission_kube_config.yaml
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false
```

**Create the Admission KubeConfig:**
```bash
sudo nano /etc/kubernetes/pki/admission_kube_config.yaml
```
```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/webhook-server.crt
    server: https://image-bouncer-webhook:31323/image_policy
  name: bouncer_webhook
contexts:
- context:
    cluster: bouncer_webhook
    user: api-server
  name: bouncer_validator
current-context: bouncer_validator
preferences: {}
users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/pki/apiserver.crt
    client-key: /etc/kubernetes/pki/apiserver.key
```

---

### Step 5: Set Up the ImagePolicyWebhook in Kubernetes

**Create the Webhook Service:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  labels:
    app: image-bouncer-webhook
  name: image-bouncer-webhook
spec:
  type: NodePort
  ports:
  - name: https
    port: 443
    targetPort: 1323
    protocol: "TCP"
    nodePort: 31323
  selector:
    app: image-bouncer-webhook
EOF
```

**Deploy the Webhook Backend:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: image-bouncer-webhook
spec:
  selector:
    matchLabels:
      app: image-bouncer-webhook
  template:
    metadata:
      labels:
        app: image-bouncer-webhook
    spec:
      containers:
      - name: image-bouncer-webhook
        imagePullPolicy: Always
        image: "kainlite/kube-image-bouncer:latest"
        args:
        - "--cert=/etc/admission-controller/tls/tls.crt"
        - "--key=/etc/admission-controller/tls/tls.key"
        - "--debug"
        - "--registry-whitelist=docker.io,registry.k8s.io"
        volumeMounts:
        - name: tls
          mountPath: /etc/admission-controller/tls
      volumes:
      - name: tls
        secret:
          secretName: tls-image-bouncer-webhook
EOF
```

**Edit the API Server Manifest:**
```bash
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add the following lines:
```yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
- --admission-control-config-file=/etc/kubernetes/pki/admission_configuration.yaml
```

**Restart the API Server:**
The `kube-apiserver` Pod will automatically restart after the changes.

---

### Step 6: Test the ImagePolicyWebhook

**Deploy a Test Pod with a Forbidden Image:**
```bash
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
        image: nginx
EOF
```

**Observe the Failure:**
The webhook should reject the pod creation.
```bash
kubectl describe replicaset nginx-latest
```

**Expected Error Message:**
```
Error creating: pods "nginx-latest-xxxxx" is forbidden: image policy webhook backend denied one or more images: Images using latest tag are not allowed
```

> **Tip:** To see logs from the webhook backend demonstrating how image policies are enforced, use:
> ```bash
> kubectl logs -f deployment/image-bouncer-webhook
> ```

**Fix the Image Tag and Redeploy:**
```bash
kubectl delete rs nginx-latest
```

Edit the deployment to use a specific version of the `nginx` image (e.g., `nginx:1.19`).
```bash
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
        image: nginx:1.19
EOF
```

**Check Pod Creation:**
```bash
kubectl describe replicaset nginx-latest
kubectl get pods | grep nginx-latest
```

---

## Verification

Verify that the ImagePolicyWebhook is working correctly by:
1. Confirming that pods with `:latest` tags are rejected
2. Confirming that pods with specific version tags are accepted
3. Checking webhook deployment and pod status in the cluster

---

## Cleanup

**Delete the Resources:**
```bash
kubectl delete -f ./nginx-latest.yml
kubectl delete ValidatingWebhookConfiguration image-policy-webhook
kubectl delete deployment image-policy-webhook
kubectl delete service image-policy-webhook
kubectl delete secret image-policy-webhook-secret
```
