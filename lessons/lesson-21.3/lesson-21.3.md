# Lesson 21.3 — Implementing and Verifying mTLS with Cilium

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Configure mTLS (mutual TLS) using Cilium and SPIRE for secure mutual authentication between pods in a Kubernetes cluster, then verify that identities and authentication are working correctly.

## Prerequisites

- Kubernetes cluster running
- kubectl configured to access the cluster
- Cilium CLI installed

## Steps

### Step 1: Enable mTLS with Cilium

Install Cilium with encryption and mutual authentication enabled:

```bash
cilium install \
    --set encryption.enabled=true \
    --set encryption.type=wireguard \
    --set authentication.mutual.spire.enabled=true \
    --set authentication.mutual.spire.install.enabled=true \
    --set authentication.mutual.spire.install.server.dataStorage.enabled=false
```

This command installs Cilium with WireGuard encryption and enables mutual authentication with SPIRE, which issues and verifies identity certificates for mTLS between pods.

### Step 2: Deploy Client and Server Pods

Deploy a service, echo server deployment, and test client pod:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  labels:
    app: echo
  name: echo
spec:
  ports:
  - port: 8080
    name: high
    protocol: TCP
    targetPort: 8080
  selector:
    app: echo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: echo
  name: echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - image: kicbase/echo-server:1.0
        name: echo
        ports:
        - containerPort: 8080
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: pod-worker
  name: pod-worker
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot:latest
    command: ["sleep", "infinite"]
EOF
```

This creates an `echo` service and deployment running an echo server, along with a `pod-worker` pod for testing network connectivity.

### Step 3: Apply Initial Cilium Network Policy

Define a Cilium network policy allowing specific traffic without mTLS enforcement:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: no-mutual-auth-echo
spec:
  endpointSelector:
    matchLabels:
      app: echo
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: pod-worker
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/headers"
EOF
```

This policy allows ingress traffic from `pod-worker` to the echo service on port 8080 for the `/headers` path.

### Step 4: Verify Pod Deployment

Check that pods and services are deployed correctly:

```bash
kubectl get svc echo
kubectl get pod pod-worker
```

### Step 5: Test Initial Network Policy

Test network connectivity from `pod-worker` to the echo service:

```bash
echo "Testing allowed path:"
kubectl exec -it pod-worker -- curl -s -o /dev/null -w "%{http_code}" http://echo:8080/headers

echo "Testing denied path:"
kubectl exec -it pod-worker -- curl http://echo:8080/headers-1
```

The first request to `/headers` should succeed, while the second request to `/headers-1` should be denied by the policy.

### Step 6: Verify SPIRE Health

Check the status of SPIRE components managing certificate identities:

```bash
kubectl get all -n cilium-spire
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- /opt/spire/bin/spire-server healthcheck
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- /opt/spire/bin/spire-server agent list
```

### Step 7: Verify SPIFFE Identities

List and verify SPIFFE identities issued by SPIRE:

```bash
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- /opt/spire/bin/spire-server entry show -parentID spiffe://spiffe.cilium/ns/cilium-spire/sa/spire-agent

IDENTITY_ID=$(kubectl get cep -l app=echo -o=jsonpath='{.items[0].status.identity.id}')
echo "Echo pod identity: $IDENTITY_ID"
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- /opt/spire/bin/spire-server entry show -spiffeID spiffe://spiffe.cilium/identity/$IDENTITY_ID
```

These commands verify that the echo pod has been assigned a SPIFFE identity and show the registered entries in SPIRE.

### Step 8: Enforce and Verify Mutual Authentication

Apply the mutual authentication policy and test it:

```bash
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.16.0/examples/kubernetes/servicemesh/cnp-with-mutual-auth.yaml

cilium config set debug true

echo "Testing allowed path:"
kubectl exec -it pod-worker -- curl -s -o /dev/null -w "%{http_code}" http://echo:8080/headers

echo "Testing denied path:"
kubectl exec -it pod-worker -- curl http://echo:8080/headers-1
```

### Step 9: Check Cilium Logs for Authentication Details

Enable debug mode and review logs to verify mTLS validation:

```bash
kubectl -n kube-system -c cilium-agent logs $(kubectl -n kube-system get pods -l k8s-app=cilium -o jsonpath='{.items[0].metadata.name}') --timestamps=true | grep "Policy is requiring authentication\|Validating Server SNI\|Validated certificate\|Successfully authenticated"
```

## Verification

Confirm mTLS is working by:
- Verifying SPIRE components are healthy and running in the `cilium-spire` namespace
- Confirming that pods have been issued SPIFFE identities
- Testing that traffic allowed by policy succeeds and traffic denied by policy is blocked
- Reviewing Cilium agent logs for successful authentication events

## Cleanup

To remove all resources created during this lesson:

```bash
kubectl delete pod pod-worker
kubectl delete deployment echo
kubectl delete svc echo
kubectl delete ciliumnnetworkpolicies no-mutual-auth-echo mutual-auth-echo
cilium uninstall
```
