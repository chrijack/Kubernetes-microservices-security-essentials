# Lesson 19.4 — Secrets Encryption at Rest

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Demonstrate how to implement and verify encryption for Kubernetes secrets at rest using the AES-CBC encryption provider.

## Prerequisites

- Kubernetes 1.30 cluster with etcd access
- `etcd-client` tools installed
- Root or sudo access to cluster control plane nodes
- Understanding of Kubernetes secrets

## Steps

### Step 1: Install ETCD Client and View Unencrypted Secrets

Install the etcd client tools:
```bash
sudo apt-get update && sudo apt-get install etcd-client
```

Create an unencrypted secret:
```bash
kubectl create secret generic unencrypted-secret --from-literal=mykey=mydata
```

Retrieve the secret from etcd to verify it is stored in plain text:
```bash
sudo ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/unencrypted-secret | hexdump -C
```

Observe that the secret data is visible in plain text.

### Step 2: Configure Encryption

Kubernetes supports several encryption providers:
- **identity** — no encryption
- **aescbc** — AES-CBC with PKCS#7 padding
- **aesgcm** — AES-GCM with random nonce
- **secretbox** — XSalsa20 and Poly1305
- **kms v2** — key management service (beta)

Note: kms v1 is deprecated and should not be used.

Generate an encryption key:
```bash
head -c 32 /dev/urandom | base64
```

Create the encryption configuration file:
```bash
cat <<EOF > encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: $(head -c 32 /dev/urandom | base64)
    - identity: {}
EOF
```

Move the configuration file to the appropriate directory:
```bash
sudo mkdir -p /etc/kubernetes/enc/
sudo mv encryption-config.yaml /etc/kubernetes/enc/
```

Update the Kubernetes API server configuration to enable encryption:
```bash
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add the following lines to the spec:
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
    volumeMounts:
    - mountPath: /etc/kubernetes/enc
      name: enc
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/enc
      type: DirectoryOrCreate
    name: enc
```

Restart the kubelet to apply changes:
```bash
sudo systemctl restart kubelet
```

### Step 3: Verify Encryption

Create a new secret after enabling encryption:
```bash
kubectl create secret generic test-secret --from-literal=mykey=mydata
```

Retrieve the new secret from etcd to verify it is encrypted:
```bash
sudo ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-secret | hexdump -C
```

Observe that the secret data is now encrypted and not visible in plain text.

Verify that the secret is still accessible through kubectl (transparent decryption):
```bash
kubectl get secret test-secret -n default -o yaml
```

Check the previously created unencrypted secret to confirm it remains unencrypted in etcd:
```bash
sudo ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/unencrypted-secret | hexdump -C
```

Note that Kubernetes transparently decrypts the secret for authorized users.

To encrypt all existing secrets, trigger a replacement:
```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Validate that previously unencrypted secrets are now encrypted in etcd:
```bash
sudo ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/unencrypted-secret | hexdump -C
```

## Verification

- Unencrypted secrets show plaintext data when queried from etcd
- Encrypted secrets show binary/encrypted data when queried from etcd
- `kubectl` still returns readable secret values (transparent decryption)
- All secrets can be re-encrypted by triggering a replacement after configuration changes

## Cleanup

Remove test secrets:
```bash
kubectl delete secret unencrypted-secret test-secret
```

(Optional) Disable encryption by removing the encryption provider configuration from kube-apiserver.yaml and restarting kubelet.
