# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GitOps repository for managing a local Kubernetes cluster using Kind, ArgoCD, and Crossplane with LocalStack.

**Stack:**
- **Kind**: Local Kubernetes cluster (3 nodes: 1 control-plane, 2 workers)
- **ArgoCD**: GitOps continuous deployment
- **Crossplane**: Infrastructure as Code with LocalStack provider
- **LocalStack**: Local AWS cloud emulation

## Repository Structure

```
local-cluster/
├── cluster/                    # Local Kind configuration (not synced to cluster)
│   └── kind-config.yaml        # Kind cluster topology definition
│
├── bootstrap/                  # ArgoCD Applications (apply manually)
│   ├── main.yaml              # Infrastructure Application
│   └── crossplane.yaml        # Crossplane Helm Application
│
├── infrastructure/             # Core cluster infrastructure (managed by ArgoCD)
│   ├── namespaces/            # Namespace definitions
│   ├── crossplane/            # Crossplane providers & configs
│   │   ├── providers/         # Crossplane providers (AWS, etc)
│   │   └── providerconfigs/   # Provider configurations (LocalStack)
│   └── kustomization.yaml
│
└── apps/                       # User applications (managed by ArgoCD)
    └── kustomization.yaml
```

## Common Commands

### Cluster Management

```bash
# Create the Kind cluster
kind create cluster --config cluster/kind-config.yaml --name kind

# Delete the cluster
kind delete cluster --name kind

# Get cluster info
kubectl cluster-info --context kind-kind
kubectl get nodes
```

### ArgoCD Management

```bash
# Apply bootstrap Applications (do this ONCE after ArgoCD installation)
kubectl apply -f bootstrap/main.yaml
kubectl apply -f bootstrap/crossplane.yaml

# Check ArgoCD Applications
kubectl get applications -n argocd
argocd app list
argocd app get infrastructure
argocd app get crossplane

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Username: admin
# Password: kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

### Crossplane & LocalStack

```bash
# Check Crossplane installation
kubectl get providers
kubectl get providerconfigs

# Check LocalStack health
curl http://localhost:4566/_localstack/health

# Create a test S3 bucket with Crossplane
kubectl apply -f - <<EOF
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: test-bucket
spec:
  forProvider:
    region: us-east-1
  providerConfigRef:
    name: localstack
EOF

# Check managed resources
kubectl get managed
kubectl get buckets
```

## GitOps Workflow

1. **Make changes** to manifests in `infrastructure/` or `apps/`
2. **Commit and push** to main branch
3. **ArgoCD syncs automatically** (automated sync enabled)
4. **Verify deployment**: `argocd app get <app-name>`

### Adding New Infrastructure

1. Create manifests in `infrastructure/<component>/`
2. Update `infrastructure/kustomization.yaml` to include new component
3. Commit and push - ArgoCD applies automatically

### Adding New Applications

1. Create manifests in `apps/<app-name>/`
2. Update `apps/kustomization.yaml` to include new app
3. Commit and push - ArgoCD applies automatically

## Architecture Notes

**Separation of Concerns:**
- `cluster/` - Local configuration (ignored by ArgoCD)
- `bootstrap/` - ArgoCD Application definitions (applied manually once)
- `infrastructure/` - Core cluster services (namespaces, Crossplane, etc)
- `apps/` - User applications

**ArgoCD Applications:**
- `infrastructure` - Monitors `infrastructure/` directory via Kustomize
- `crossplane` - Installs Crossplane via Helm chart directly

**Crossplane with LocalStack:**
- Provider: `provider-aws-s3` (Upbound)
- ProviderConfig: `localstack` (points to `http://host.docker.internal:4566`)
- Credentials: Fake credentials in Secret `aws-creds`

## Initial Setup (from scratch)

```bash
# 1. Create Kind cluster
kind create cluster --config cluster/kind-config.yaml --name kind

# 2. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# 4. Bootstrap GitOps
kubectl apply -f bootstrap/main.yaml
kubectl apply -f bootstrap/crossplane.yaml

# 5. Start LocalStack (in separate terminal)
docker run -d -p 4566:4566 localstack/localstack

# 6. Verify everything is synced
argocd app list
kubectl get providers
```
