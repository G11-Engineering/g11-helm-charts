# G11 Helm Charts

Kubernetes deployment manifests for the G11 microservices platform using GitOps principles with ArgoCD.

> **Note**: This repository is designed to be deployed by ArgoCD, which is configured and provisioned by the [G11-Engineering/g11-iac-sys-adm](https://github.com/G11-Engineering/g11-iac-sys-adm) Terraform repository. The infrastructure repository sets up the entire Kubernetes cluster with all required system components.

## 🏗️ Architecture Overview

### Application Layer (This Repository)
- **Service Mesh**: Istio 1.28.1 for traffic management and security
- **GitOps**: ArgoCD with App-of-Apps pattern for declarative deployments
- **Secrets Management**: Sealed Secrets with cluster-wide scope
- **Database**: PostgreSQL with Crunchy Data Operator
- **TLS/Certificates**: cert-manager with Let's Encrypt ACME
- **Autoscaling**: Horizontal Pod Autoscaler (HPA) for frontend service
- **Progressive Delivery**: Argo Rollouts with canary deployments

### Infrastructure Layer (Provisioned by Terraform)

The [g11-iac-sys-adm](https://github.com/G11-Engineering/g11-iac-sys-adm) repository provisions:

#### Core System Components
- **kube-system**: Core Kubernetes components, AWS EFS CSI Driver, Metrics Server, Sealed Secrets controller
- **istio-system**: Istio v1.28.1 service mesh (control plane, CNI, ingress gateway with NodePort)
- **argocd**: ArgoCD for GitOps continuous deployment
- **cluster-autoscaler**: Cluster Autoscaler for pod-aware node scaling
- **cert-manager**: Certificate management with Let's Encrypt integration
- **postgres-operator**: Crunchy Data PostgreSQL Operator for database management

#### Application Namespaces (G11 Blog)
- **g11-prod**: Production environment
  - Istio sidecar injection enabled
  - PostgreSQL instance (`hippo`) pre-deployed with EFS storage
  - Ready for G11 Blog production workloads

- **g11-stg**: Staging environment
  - Istio sidecar injection enabled
  - PostgreSQL instance (`hippo`) pre-deployed with EFS storage
  - Ready for G11 Blog staging workloads

#### Additional Components
- **Argo Rollouts**: Progressive delivery controller with kubectl plugin
- **Helm**: v3.x for package management
- **Calico**: v3.31.0 for pod networking (CNI)

## 📁 Repository Structure

```
.
├── argo-cd/              # ArgoCD server + Istio Gateway + cert-manager ClusterIssuers
├── argo-apps/            # ArgoCD Application manifests (managed by App-of-Apps)
│   ├── g11-prod.yaml     # Production application definition
│   └── g11-stg.yaml      # Staging application definition
├── app-of-apps.yaml      # Bootstrap application (manages all apps)
├── g11/                  # Main application Helm chart
│   ├── templates/
│   │   ├── microservice-*.yaml  # Argo Rollouts for each service
│   │   ├── gateway.yaml         # Istio Gateway with TLS
│   │   ├── virtualservice.yaml  # Traffic routing + HTTPS redirect
│   │   ├── certificate.yaml     # Let's Encrypt TLS certificates
│   │   └── secrets.yaml         # Sealed Secrets for credentials
│   └── values.yaml
├── postgres/             # PostgreSQL Helm chart (Crunchy Data PGO)
└── SEALED-SECRETS.md     # Guide for encrypting secrets
```

## 🚀 Quick Start

### Using Terraform Infrastructure (Recommended)

If you're using the [g11-iac-sys-adm](https://github.com/G11-Engineering/g11-iac-sys-adm) Terraform repository, **all infrastructure is already provisioned**. Skip to [GitOps Deployment](#-gitops-deployment-app-of-apps-pattern).

### Manual Setup Requirements

For manual cluster setup, ensure the following are installed and configured:

**Required Components:**
- Kubernetes cluster (1.24+) with `kubectl` configured
- Istio v1.28.1 service mesh with ingress gateway
- cert-manager with Let's Encrypt ClusterIssuers
- Sealed Secrets controller
- Crunchy Data PostgreSQL Operator
- ArgoCD with Argo Rollouts
- Metrics Server (for HPA)

**Required Namespaces:**
- `g11-prod` and `g11-stg` (with Istio injection enabled)
- PostgreSQL clusters deployed in both namespaces

**CLI Tools:**
- `helm` (v3.x)
- `kubectl`
- `kubeseal` (v0.24.0) - [Installation Guide](SEALED-SECRETS.md)
- `argocd` CLI (optional)

---

## 🔄 GitOps Deployment (App-of-Apps Pattern)

> **Recommended Approach**: Use GitOps with ArgoCD for automated deployments.

### What is App-of-Apps?

The App-of-Apps pattern uses a single "parent" ArgoCD Application that manages multiple "child" Applications. This provides:

- ✅ Single source of truth for all applications
- ✅ Declarative application management
- ✅ Automatic synchronization from Git
- ✅ Environment consistency (prod and staging)

### Prerequisites

Ensure you have:
- ArgoCD installed and accessible
- This repository added to ArgoCD (or public GitHub access)
- Sealed secrets already encrypted in `g11/values.yaml`

### Deploy Using App-of-Apps

```bash
# Apply the bootstrap application
kubectl apply -f app-of-apps.yaml

# This creates the app-of-apps Application in ArgoCD, which then:
# 1. Reads argo-apps/ directory
# 2. Creates Applications for g11-prod and g11-stg
# 3. Each application deploys the g11 Helm chart to its namespace
```

### Verify Deployment

```bash
# Check ArgoCD Applications
kubectl get applications -n argocd

# Expected output:
# NAME           SYNC STATUS   HEALTH STATUS
# app-of-apps    Synced        Healthy
# g11-prod       Synced        Healthy
# g11-stg        Synced        Healthy

# Check pods in production
kubectl get pods -n g11-prod

# Check pods in staging
kubectl get pods -n g11-stg
```

### Sync Applications Manually

```bash
# Sync the parent app (triggers sync of all child apps)
argocd app sync app-of-apps

# Or sync individual applications
argocd app sync g11-prod
argocd app sync g11-stg
```

## 🔐 Sealed Secrets

This repository uses **Sealed Secrets with cluster-wide scope** to manage sensitive credentials securely in Git.

- ✅ **GitOps-friendly**: Encrypted secrets can be committed to Git
- ✅ **Cluster-wide scope**: Same encrypted value works in all namespaces
- ✅ **Secure**: Only the cluster can decrypt using its private key

### Encrypting Secrets

```bash
# Encrypt a value (works in all namespaces)
echo -n "YOUR_SECRET_VALUE" | kubeseal --raw --scope=cluster-wide
```

Update `g11/values.yaml` with the encrypted output:
```yaml
secrets:
  useSealed: true
  jwtSecret: AgBx7R3kj8...  # ← Paste encrypted value here
```

See [SEALED-SECRETS.md](SEALED-SECRETS.md) for detailed guide.

## 🌐 Accessing Services

### Port Forwarding (Development)

```bash
# ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Frontend (Production)
kubectl port-forward svc/frontend -n g11-prod 8080:80

# Frontend (Staging)
kubectl port-forward svc/frontend -n g11-stg 8180:80
```

### Via Istio Gateway (Production)

- **Production**: `https://yourdomain.com`
- **Staging**: `https://stg.yourdomain.com`
- **ArgoCD**: `https://argocd.yourdomain.com`

## 📊 Monitoring & Operations

### Horizontal Pod Autoscaler

The frontend service autoscales between 2-5 pods based on CPU (>70%) and memory (>80%) usage.

```bash
# View HPA status
kubectl get hpa -n g11-prod

# Watch resource usage
kubectl top pods -n g11-prod -l app=frontend
```

### Certificate Management

```bash
# Check certificate status
kubectl get certificate -n istio-system

# View certificate details
kubectl describe certificate argocd-tls-prod -n istio-system
```

## 🔧 Configuration

Key configuration options in `argo-apps/*.yaml`:

| Parameter | Description | Default |
| --- | --- | --- |
| `gateway.hostname` | Domain name for the service | `"*"` |
| `gateway.tls.enabled` | Enable HTTPS/TLS | `true` |
| `gateway.tls.acme.useStaging` | Use Let's Encrypt staging | `true` |
| `secrets.useSealed` | Use Sealed Secrets | `true` |

## 🛠️ Troubleshooting

### ArgoCD Applications

```bash
argocd app get g11-prod
argocd app logs g11-prod
```

### Certificates

```bash
kubectl describe certificate -n istio-system
kubectl logs -n cert-manager -l app=cert-manager
kubectl get challenges -n istio-system
```

### Sealed Secrets

```bash
kubectl logs -n kube-system -l name=sealed-secrets-controller
kubectl get sealedsecret -n g11-prod
```

### PostgreSQL

```bash
kubectl get postgrescluster -n g11-prod
kubectl logs -n postgres-operator -l app.kubernetes.io/name=pgo
```

## 📚 Resources

### Related Repositories

- [G11 Infrastructure (Terraform)](https://github.com/G11-Engineering/g11-iac-sys-adm) - Provisions the entire Kubernetes cluster and all system components

### Documentation

- [Istio Documentation](https://istio.io/latest/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Sealed Secrets Guide](https://github.com/bitnami-labs/sealed-secrets)
- [Crunchy Data PGO](https://access.crunchydata.com/documentation/postgres-operator/latest/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)

## 🤝 Contributing

1. Make changes in a feature branch
2. Test changes in staging environment
3. Update encrypted secrets using `kubeseal` if needed
4. Commit and push to trigger ArgoCD sync
5. Verify deployment in ArgoCD UI

> **Note**: For infrastructure changes (nodes, system components, operators), see the [g11-iac-sys-adm](https://github.com/G11-Engineering/g11-iac-sys-adm) repository.

## 📝 License

MIT License

