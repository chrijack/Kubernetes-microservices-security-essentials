# Lesson 22.2 — ImagePolicyWebhook Policy Enforcement

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Configure and demonstrate an ImagePolicyWebhook admission controller in Kubernetes 1.30 to enforce image policies and prevent the deployment of pods using disallowed images.

## Prerequisites

- Kubernetes 1.30 cluster running (local or remote)
- Administrative privileges in the cluster
- Webhook admission controller enabled on the API server

## Steps

### Step 1: Set Up Host DNS

Ensure the ImagePolicyWebhook can be resolved by the Kubernetes cluster by updating the `/etc/hosts` file on all nodes.

```bash
sudo nano /etc/hosts
127.0.0.1 image-bouncer-webhook
```

This makes the service resolvable within the Kubernetes cluster as `image-bouncer-webhook.default.svc`.

### Step 2: Generate Certificates

Generate self-signed certificates for securing communication between the API server and the webhook backend.

1. **Generate the Certificate and Key:**

```bash
openssl genrsa -out webhook-server.key 2048

openssl req -new -key webhook-server.key \
  -subj "/CN=system:node:image-bouncer-webhook.default.pod.cluster.local/O=system:nodes" \
  -addext "subjectAltName=DNS:image-bouncer-webhook,DNS:image-bouncer-webhook.default.svc,DNS:image-bouncer-webhook.default.svc.cluster.local" \
  -out webhook-server.csr
```

2. **Verify the CSR (optional):**

```bash
openssl req -in webhook-server.csr -text -noout
```

3. **Encode the CSR and Create CertificateSigningRequest:**

```bash
export SIGNING_REQUEST=$(cat webhook-server.csr | base64 | tr -d "\n")

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

4. **Approve and Retrieve the Signed Certificate:**

```bash
kubectl get csr
kubectl certificate approve image-bouncer-webhook.default
kubectl get csr image-bouncer-webhook.default -o=jsonpath={.status.certificate} | base64 --decode > webhook-server.crt
```

5. **Place the Certificate in the `/etc/kubernetes/pki` Directory:**

```bash
sudo cp ~/webhook-server.crt /etc/kubernetes/pki/
```

### Step 3: Create TLS Secret in Kubernetes

Store the certificate and key as a secret in Kubernetes for use by the webhook backend.

```bash
kubectl create secret tls tls-image-bouncer-webhook --cert=webhook-server.crt --key=webhook-server.key
```

### Step 4: Create ImagePolicyWebhook Configuration Files

1. **Create the Admission Configuration:**

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

2. **Create the Admission KubeConfig:**

```bash
sudo nano /etc/kubernetes/pki/admission_kube_config.yaml
```

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/webhook-server.crt
    server: https://image-bouncer-webhook:30080/image_policy
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

### Step 5: Set Up the ImagePolicyWebhook in Kubernetes

1. **Create the Webhook Service:**

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
    nodePort: 30080
  selector:
    app: image-bouncer-webhook
EOF
```

2. **Deploy the Webhook Backend:**

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

3. **Edit the API Server Manifest to Enable ImagePolicyWebhook:**

```bash
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add the following lines to the `spec.containers[0].command` section:

```yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
- --admission-control-config-file=/etc/kubernetes/pki/admission_configuration.yaml
```

4. **Verify the API Server Restart:**

The `kube-apiserver` Pod will automatically restart after the changes.

### Step 6: Test the ImagePolicyWebhook

1. **Deploy a Test Pod with a Disallowed Image (latest tag):**

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

2. **Observe the Failure:**

The webhook should reject the pod creation because it uses the `latest` tag.

```bash
kubectl describe replicaset nginx-latest
```

Expected error message:
```
Error creating: pods "nginx-latest-xxxxx" is forbidden: image policy webhook backend denied one or more images: Images using latest tag are not allowed
```

3. **Delete the Failed ReplicaSet:**

```bash
kubectl delete rs nginx-latest
```

4. **Redeploy with a Specific Image Tag:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-versioned
  labels:
    tier: nginx-versioned
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: nginx-versioned
  template:
    metadata:
      labels:
        tier: nginx-versioned
    spec:
      containers:
      - name: nginx-versioned
        image: nginx:1.19
EOF
```

5. **Verify Successful Pod Creation:**

```bash
kubectl describe replicaset nginx-versioned
kubectl get pods | grep nginx-versioned
```

## Verification

- Pod creation with the `latest` tag is rejected by the ImagePolicyWebhook
- Pod creation with specific image versions (e.g., `nginx:1.19`) succeeds
- Webhook logs show policy enforcement decisions

To view webhook logs:

```bash
kubectl logs -f deployment/image-bouncer-webhook
```

## Cleanup

Remove all resources created during this demonstration:

```bash
kubectl delete replicaset nginx-versioned
kubectl delete deployment image-bouncer-webhook
kubectl delete service image-bouncer-webhook
kubectl delete secret tls-image-bouncer-webhook
```

Remove configuration files from the control plane node:

```bash
sudo rm /etc/kubernetes/pki/admission_configuration.yaml
sudo rm /etc/kubernetes/pki/admission_kube_config.yaml
sudo rm /etc/kubernetes/pki/webhook-server.crt
```

Edit the API server manifest to remove the ImagePolicyWebhook plugin:

```bash
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

Remove these lines:

```yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
- --admission-control-config-file=/etc/kubernetes/pki/admission_configuration.yaml
```

The API server will automatically restart with the updated configuration.
