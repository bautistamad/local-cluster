# Local Kubernetes Cluster with GitOps

A production-ready local Kubernetes development environment using **Kind**, **ArgoCD**, and **Crossplane** with **LocalStack** for AWS infrastructure simulation.

## 🚀 Features

- **GitOps-ready**: All infrastructure as code, managed by ArgoCD
- **Local Kubernetes**: 3-node Kind cluster (1 control-plane + 2 workers)
- **Infrastructure as Code**: Crossplane with LocalStack for AWS resource provisioning
- **Automated Sync**: Changes to Git automatically deploy to the cluster
- **Multi-service AWS Support**: S3, EC2, RDS, Lambda, DynamoDB, SQS, SNS

## 📚 Tech Stack

| Component | Purpose |
|-----------|---------|
| **Kind** | Local Kubernetes cluster |
| **ArgoCD** | GitOps continuous deployment |
| **Crossplane** | Kubernetes-native infrastructure provisioning |
| **LocalStack** | Local AWS cloud emulation |
| **Kustomize** | Kubernetes manifest management |

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│                    GitHub Repo                       │
│  (Single Source of Truth - GitOps)                  │
└────────────────┬────────────────────────────────────┘
                 │
                 │ Monitors & Syncs
                 ▼
┌─────────────────────────────────────────────────────┐
│              ArgoCD (in cluster)                     │
│  - infrastructure App → infrastructure/              │
│  - crossplane App → Helm chart                      │
└────────────────┬────────────────────────────────────┘
                 │
                 │ Applies
                 ▼
┌─────────────────────────────────────────────────────┐
│         Kind Kubernetes Cluster (Local)              │
│                                                      │
│  ┌──────────────┐  ┌─────────────────────────┐     │
│  │  Crossplane  │  │  AWS Providers          │     │
│  │  Controller  ├─►│  (S3, EC2, RDS, etc.)   │     │
│  └──────────────┘  └───────────┬─────────────┘     │
│                                 │                    │
└─────────────────────────────────┼────────────────────┘
                                  │
                                  │ Provisions
                                  ▼
                        ┌──────────────────┐
                        │   LocalStack     │
                        │  (AWS Emulator)  │
                        │   Port: 4510     │
                        └──────────────────┘
```

## 📋 Prerequisites

Install the following tools:

```bash
# macOS
brew install kind kubectl argocd docker

# Verify installations
kind version
kubectl version --client
argocd version --client
docker --version
```

## 🎯 Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/bautistamad/local-cluster.git
cd local-cluster
```

### 2. Start LocalStack

```bash
docker run -d \
  --name localstack \
  -p 4510:4566 \
  -e SERVICES=s3,ec2,rds,lambda,dynamodb,sqs,sns \
  localstack/localstack
```

### 3. Create Kind Cluster

```bash
kind create cluster --config cluster/kind-config.yaml --name kind
```

This creates a 3-node cluster (1 control-plane + 2 workers).

### 4. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### 5. Bootstrap GitOps

```bash
# Apply ArgoCD Applications
kubectl apply -f bootstrap/main.yaml
kubectl apply -f bootstrap/crossplane.yaml
```

ArgoCD will automatically:
- Install Crossplane via Helm
- Deploy AWS providers (S3, EC2, RDS, Lambda, DynamoDB, SQS, SNS)
- Configure LocalStack connection
- Sync all infrastructure from Git

### 6. Verify Installation

```bash
# Check ArgoCD Applications
kubectl get applications -n argocd

# Check Crossplane Providers (takes 5-10 minutes to download)
kubectl get providers

# Expected output (when ready):
# NAME                          INSTALLED   HEALTHY   AGE
# provider-aws-s3               True        True      5m
# provider-aws-ec2              True        True      5m
# provider-aws-rds              True        True      5m
# provider-aws-lambda           True        True      5m
# provider-aws-dynamodb         True        True      5m
# provider-aws-sqs              True        True      5m
# provider-aws-sns              True        True      5m
```

### 7. Access ArgoCD UI

```bash
# Port forward ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# Open https://localhost:8080
# Username: admin
# Password: <from command above>
```

## 📁 Repository Structure

```
local-cluster/
├── cluster/                    # Local Kind configuration
│   └── kind-config.yaml        # Cluster topology (not synced to cluster)
│
├── bootstrap/                  # ArgoCD Applications (apply manually once)
│   ├── main.yaml              # Infrastructure Application
│   └── crossplane.yaml        # Crossplane Helm Application
│
├── infrastructure/             # Core cluster infrastructure (GitOps managed)
│   ├── namespaces/            # Namespace definitions
│   ├── crossplane/
│   │   ├── providers/         # AWS providers (S3, EC2, RDS, etc.)
│   │   └── providerconfigs/   # LocalStack configuration
│   └── kustomization.yaml
│
├── apps/                       # User applications (GitOps managed)
│   └── kustomization.yaml
│
├── CLAUDE.md                   # AI assistant guidance
└── README.md                   # This file
```

## 🔄 GitOps Workflow

### Making Changes

1. **Edit manifests** in `infrastructure/` or `apps/`
2. **Commit and push** to `main` branch
3. **ArgoCD syncs automatically** (within 3 minutes)
4. **Verify**: `kubectl get applications -n argocd`

### Adding New Infrastructure

```bash
# 1. Create manifest in infrastructure/
cat > infrastructure/my-component/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
# ...
EOF

# 2. Update infrastructure/kustomization.yaml
# Add: - my-component/

# 3. Commit and push
git add infrastructure/
git commit -m "feat: Add my-component"
git push origin main

# 4. ArgoCD syncs automatically
```

## 💡 Usage Examples

### Create an S3 Bucket in LocalStack

```bash
cat <<EOF | kubectl apply -f -
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: my-test-bucket
spec:
  forProvider:
    region: us-east-1
  providerConfigRef:
    name: localstack
EOF

# Verify in Kubernetes
kubectl get buckets

# Verify in LocalStack
aws --endpoint-url=http://localhost:4510 s3 ls
```

### Create a DynamoDB Table

```bash
cat <<EOF | kubectl apply -f -
apiVersion: dynamodb.aws.upbound.io/v1beta1
kind: Table
metadata:
  name: my-table
spec:
  forProvider:
    region: us-east-1
    attribute:
    - name: id
      type: S
    hashKey: id
    billingMode: PAY_PER_REQUEST
  providerConfigRef:
    name: localstack
EOF

# Verify
kubectl get tables
```

### Create an SQS Queue

```bash
cat <<EOF | kubectl apply -f -
apiVersion: sqs.aws.upbound.io/v1beta1
kind: Queue
metadata:
  name: my-queue
spec:
  forProvider:
    region: us-east-1
  providerConfigRef:
    name: localstack
EOF

# Verify
kubectl get queues
```

## 🛠️ Common Commands

### Cluster Management

```bash
# Delete cluster
kind delete cluster --name kind

# Recreate cluster
kind create cluster --config cluster/kind-config.yaml --name kind

# Get cluster info
kubectl cluster-info
kubectl get nodes
```

### ArgoCD Management

```bash
# List applications
argocd app list

# Get application details
argocd app get infrastructure

# Sync application manually
argocd app sync infrastructure

# Force refresh
argocd app sync infrastructure --force
```

### Crossplane Management

```bash
# List all providers
kubectl get providers

# List all managed resources
kubectl get managed

# Describe a provider
kubectl describe provider provider-aws-s3

# Check ProviderConfig
kubectl get providerconfigs
```

### LocalStack Management

```bash
# Check LocalStack health
curl http://localhost:4510/_localstack/health

# List S3 buckets
aws --endpoint-url=http://localhost:4510 s3 ls

# List DynamoDB tables
aws --endpoint-url=http://localhost:4510 dynamodb list-tables

# List SQS queues
aws --endpoint-url=http://localhost:4510 sqs list-queues
```

## 🐛 Troubleshooting

### ArgoCD Application Stuck in "OutOfSync"

```bash
# Check application status
kubectl get application infrastructure -n argocd -o yaml

# Force sync
argocd app sync infrastructure --force

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-application-controller
```

### Crossplane Provider Not Healthy

```bash
# Check provider status
kubectl get providers

# Check provider pods
kubectl get pods -n crossplane-system

# Describe provider for errors
kubectl describe provider provider-aws-s3

# Check pod logs
kubectl logs -n crossplane-system <provider-pod-name>
```

### LocalStack Connection Issues

```bash
# Verify LocalStack is running
docker ps | grep localstack

# Check LocalStack logs
docker logs localstack

# Test connectivity from cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://host.docker.internal:4510/_localstack/health
```

### Provider Images Taking Too Long

Providers are large images (150-200MB each). First-time download can take 5-10 minutes on slow connections.

```bash
# Watch pods downloading
kubectl get pods -n crossplane-system -w

# Check image pull progress
kubectl describe pod <provider-pod-name> -n crossplane-system
```

## 🧹 Cleanup

```bash
# Delete the Kind cluster
kind delete cluster --name kind

# Stop LocalStack
docker stop localstack
docker rm localstack
```

## 📖 References

- [Kind Documentation](https://kind.sigs.k8s.io/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Crossplane Documentation](https://docs.crossplane.io/)
- [LocalStack Documentation](https://docs.localstack.cloud/)
- [Upbound AWS Providers](https://marketplace.upbound.io/providers/upbound)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'feat: Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📝 License

This project is for educational and development purposes.

---

**Made with ❤️ for local Kubernetes development**
