# Kubernetes Resource Management and Security using Kyverno and ArgoCD

## Project Overview
This project demonstrates implementing resource management policies in Kubernetes using Kyverno and ArgoCD. We focus on enforcing resource requests and limits to ensure proper resource allocation and cluster stability.

## What is Kyverno?
Kyverno is a policy engine designed specifically for Kubernetes. It operates as an admission controller, allowing you to:
- Validate: Define and enforce rules for resource configurations
- Mutate: Automatically modify resource configurations
- Generate: Create additional resources as needed
- Verify: Check resource configurations against security standards

### Why Kyverno?
- **Native Kubernetes Integration**: Uses Kubernetes resources for policy definitions
- **No New Language Required**: Policies are written in YAML, just like Kubernetes resources
- **Easy to Manage**: Policies can be managed using the same tools you use for other Kubernetes resources
- **Real-time Enforcement**: Works as an admission controller to enforce policies at resource creation

## Admission Controllers in Kubernetes
Admission controllers are plugins that intercept requests to the Kubernetes API server before object persistence. They can:
- **Validate** requests to ensure they meet specific criteria
- **Modify** requests to inject additional configurations
- **Reject** requests that violate policies

## Resource Requests and Limits
In Kubernetes, resource requests and limits are crucial for proper cluster management:

- **Resource Requests**: The minimum amount of resources guaranteed to a container
  - Helps Kubernetes schedule pods efficiently
  - Ensures pods get the resources they need

- **Resource Limits**: The maximum amount of resources a container can use
  - Prevents resource hogging
  - Protects cluster stability

## Project Implementation

### Technologies Used
- **Kubernetes**: Container orchestration platform
- **Kyverno**: Policy enforcement
- **ArgoCD**: GitOps continuous delivery
- **Amazon EKS**: Managed Kubernetes service

### Implementation Steps

1. **Install ArgoCD**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. **Install Kyverno**
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

3. **Deploy Resource Policy**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-requests-limits
spec:
  validationFailureAction: enforce
  rules:
  - name: validate-resources
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "CPU and memory resource requests and limits are required."
      pattern:
        spec:
          containers:
          - resources:
              requests:
                memory: "?*"
                cpu: "?*"
              limits:
                memory: "?*"
                cpu: "?*"
```

### What This Policy Does
- Enforces resource requests and limits for all containers
- Blocks deployment of pods without proper resource specifications
- Ensures better resource management and cluster stability

## Usage Example

Deploy a compliant pod:
```bash
kubectl run test-pod --image=nginx --restart=Never \
  --requests='cpu=100m,memory=128Mi' \
  --limits='cpu=200m,memory=256Mi'
```

This will succeed because it includes required resource specifications.

Non-compliant pod (will be blocked):
```bash
kubectl run test-pod --image=nginx
```

This will fail because it lacks resource specifications.

