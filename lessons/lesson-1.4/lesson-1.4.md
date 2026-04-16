# Lesson 1.4 — OPA Gatekeeper and ValidatingAdmissionPolicy

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Implement custom security policies in Kubernetes using both OPA Gatekeeper and ValidatingAdmissionPolicy to enforce label requirements on Pod resources.

## Prerequisites

- Kubernetes cluster with API server capable of running admission controllers
- kubectl configured with access to the cluster
- ValidatingAdmissionPolicy feature gate enabled (Kubernetes 1.26+)

## Steps

### Step 1: Install OPA Gatekeeper

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.16.3/deploy/gatekeeper.yaml
```

### Step 2: Verify Gatekeeper Installation

```bash
kubectl get pods -n gatekeeper-system
```

### Step 3: Create an OPA Constraint Template

Define a constraint template that requires an "environment" label on Pods:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredenvironmentlabel
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredEnvironmentLabel
      validation:
        openAPIV3Schema:
          properties:
            validEnvironments:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredenvironmentlabel

        violation[{"msg": msg}] {
          not input.review.object.metadata.labels.environment
          msg := "Pod must have an 'environment' label"
        }

        violation[{"msg": msg}] {
          value := input.review.object.metadata.labels.environment
          valid_environments := input.parameters.validEnvironments
          not valid_environment(value, valid_environments)
          msg := sprintf("'environment' label must be one of %v, but is '%v'", [valid_environments, value])
        }

        valid_environment(value, valid_environments) {
          valid_environments[_] == value
        }
EOF
```

### Step 4: Create an OPA Constraint

Apply the constraint template to enforce valid environment labels:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredEnvironmentLabel
metadata:
  name: pod-must-have-valid-environment
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    validEnvironments: ["production", "staging", "development"]
EOF
```

### Step 5: Create a ValidatingAdmissionPolicy

Define a policy using the built-in ValidatingAdmissionPolicy feature:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-environment-label
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
  validations:
    - expression: "object.metadata.labels.environment != null"
      message: "Pod must have an 'environment' label"
EOF
```

### Step 6: Create a ValidatingAdmissionPolicyBinding

Bind the policy to relevant namespaces, excluding system namespaces:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-environment-label-binding
spec:
  policyName: require-environment-label
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: NotIn
        values: ["kube-system", "gatekeeper-system"]
EOF
```

## Verification

### Test 1: Attempt to Create a Pod Without the Required Label

This should fail:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-no-label
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
EOF
```

### Test 2: Attempt to Create a Pod With an Invalid Environment Value

This should fail (environment must be production, staging, or development):

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-invalid-env
  labels:
    environment: test
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
EOF
```

### Test 3: Create a Compliant Pod

This should succeed:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-valid
  labels:
    environment: production
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
EOF
```

### Verify Policy Enforcement

Check that only compliant Pods were created:

```bash
kubectl get pods --show-labels
```

## Cleanup

Remove all test resources and policies:

```bash
# Delete the test pods
kubectl delete pod nginx-pod-no-label nginx-pod-invalid-env nginx-pod-valid

# Delete the OPA Gatekeeper resources
kubectl delete constrainttemplate k8srequiredenvironmentlabel
kubectl delete k8srequiredenvironmentlabel pod-must-have-valid-environment

# Delete the ValidatingAdmissionPolicy and Binding
kubectl delete validatingadmissionpolicy require-environment-label
kubectl delete validatingadmissionpolicybinding require-environment-label-binding

# Uninstall OPA Gatekeeper
kubectl delete -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.16.3/deploy/gatekeeper.yaml

# Wait for Gatekeeper namespace to be deleted
kubectl wait --for=delete namespace/gatekeeper-system --timeout=300s

# Verify cleanup
echo "Verifying cleanup..."
kubectl get pods --all-namespaces | grep gatekeeper
kubectl get constrainttemplates
kubectl get k8srequiredenvironmentlabels
kubectl get validatingadmissionpolicies
kubectl get validatingadmissionpolicybindings
```
