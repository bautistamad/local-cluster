# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Kubernetes IN Docker (kind) project for managing local Kubernetes clusters. Currently contains a basic cluster configuration with one control-plane node and two worker nodes.

## Kind Cluster Configuration

**File:** `cluster.yaml`
- Defines a 3-node Kind cluster (1 control-plane, 2 workers)
- Uses Kind API version `kind.x-k8s.io/v1alpha4`

**Common Commands:**
```bash
# Create the cluster
kind create cluster --config cluster.yaml --name kind

# Delete the cluster
kind delete cluster --name kind

# Get cluster info
kind get clusters
kubectl cluster-info --context kind-kind

# Access cluster
kubectl config use-context kind-kind

# Check nodes
kubectl get nodes
```

## Expected Project Structure

As this project grows, consider this structure:

```
kind/
├── cluster.yaml          # Cluster node configuration
├── manifests/            # Kubernetes manifests (deployments, services, etc)
│   ├── namespaces/
│   ├── core/
│   └── apps/
├── helm/                 # Helm charts (if using Helm)
├── scripts/              # Automation scripts
│   ├── setup.sh          # Initial cluster setup
│   ├── deploy.sh         # Application deployment
│   └── cleanup.sh
└── docs/                 # Documentation
    └── ARCHITECTURE.md
```

## Development Workflow

When expanding this project:

1. **Add manifests** to `manifests/` for Kubernetes resources
2. **Create setup scripts** in `scripts/` if automating cluster initialization
3. **Use `kubectl apply -f`** to deploy manifests
4. **Document architecture** if adding complex deployments or GitOps patterns

## Notes for Future Work

- If using Helm, add `helm/` directory with chart structures
- If adding GitOps (ArgoCD, Flux), document the workflow in `docs/`
- If adding application code (controllers, operators), create separate directories with appropriate build/test tooling
- Keep `cluster.yaml` as the source of truth for cluster topology
