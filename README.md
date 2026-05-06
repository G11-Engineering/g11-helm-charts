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

## 💻 Local Demo On Ubuntu With kind

This repository now includes a local demo path that is separate from the production and staging ArgoCD applications:

- `app-of-apps-kind.yaml`
- `argo-apps-kind/g11-kind-postgres.yaml`
- `argo-apps-kind/g11-kind.yaml`
- `g11/values-kind.yaml`
- `postgres/values-kind.yaml`
- `nonhelmhelpers/kind-config.yaml`
- `nonhelmhelpers/istio-profile.yml`

### What The Local Demo Needs

For the local demo, you do **not** need cert-manager or Sealed Secrets because the `values-kind.yaml` files disable those production-specific parts.

You **do** need these controllers and components in the cluster:

- Istio using `nonhelmhelpers/istio-profile.yml`
- Argo CD
- Argo Rollouts
- Crunchy Data PostgreSQL Operator
- Metrics Server

### 1. Install The Required Tools On Ubuntu

Install base packages and Docker:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl git gnupg jq

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker "$USER"
newgrp docker
docker version
```

Install `kubectl`:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
kubectl version --client
```

Install `kind`:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

Install `helm`:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Install `istioctl`:

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.28.1 sh -
sudo install -m 0755 istio-1.28.1/bin/istioctl /usr/local/bin/istioctl
istioctl version
```

Install the Argo CD CLI if you want it:

```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 0555 argocd /usr/local/bin/argocd
argocd version --client
```

### 2. Create The kind Cluster

Use the included kind config so the Istio ingress NodePorts from `nonhelmhelpers/istio-profile.yml` are reachable on the host:

```bash
kind create cluster --name g11 --config nonhelmhelpers/kind-config.yaml
kubectl cluster-info --context kind-g11
kubectl config use-context kind-g11
```

If you need to recreate it:

```bash
kind delete cluster --name g11
```

### 3. Install Istio Using The Repo Profile

This repository expects Istio, and for local work you should use the included profile file because it enables the ingress gateway, fixed NodePorts, Istio CNI, and the settings this repo was built around:

```bash
istioctl install -f nonhelmhelpers/istio-profile.yml -y
kubectl get pods -n istio-system
```

### 4. Create And Label The Namespaces Before Installing Argo CD Or The Apps

The `g11` chart enforces strict mTLS, and the rollout health checks depend on Istio-aware traffic inside the mesh. Create the namespaces first and label them for sidecar injection:

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace argocd istio-injection=enabled --overwrite

kubectl create namespace g11-kind --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace g11-kind istio-injection=enabled --overwrite
```

### 5. Install The Cluster Dependencies

Install Argo CD with Helm instead of the raw `install.yaml` manifest. This avoids CRD annotation size issues on newer clusters and keeps the install path aligned with CRD best practices:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd \
  -n argocd \
  --set configs.params.server.insecure=true
kubectl rollout status deployment/argocd-server -n argocd --timeout=180s
```

Install Argo Rollouts into the same namespace so the rollout controller is also part of the mesh:

```bash
helm upgrade --install argo-rollouts argo/argo-rollouts -n argocd
kubectl rollout status deployment/argo-rollouts -n argocd
```

Install Metrics Server:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
kubectl rollout status deployment/metrics-server -n kube-system
```

Install the Crunchy Data PostgreSQL Operator. This chart uses the `PostgresCluster` CRD, so the operator must be installed before Argo CD syncs the `postgres` chart:

```bash
kubectl create namespace postgres-operator --dry-run=client -o yaml | kubectl apply -f -
kubectl apply --server-side -k "github.com/CrunchyData/postgres-operator/config/default?ref=v6.0.1"
kubectl rollout status deployment/pgo -n postgres-operator --timeout=180s
```

### 6. Prepare The Git Source That Argo CD Will Sync From

Argo CD does not sync from your uncommitted working tree. Commit and push the local demo files to a branch in a Git repository Argo CD can read.

If you are not using this repository's `main` branch directly, update:

- `app-of-apps-kind.yaml`
- `argo-apps-kind/g11-kind-postgres.yaml`
- `argo-apps-kind/g11-kind.yaml`

Set `repoURL` and `targetRevision` to your fork and branch.

### 7. Set Up Argo CD Against The Cluster

You can do this either by applying the parent application YAML directly or by creating the parent application in the Argo CD UI.

#### Option A: Apply The Parent Application YAML

```bash
kubectl apply -f app-of-apps-kind.yaml
kubectl get applications -n argocd
```

Expected applications:

- `app-of-apps-kind`
- `g11-kind-postgres`
- `g11-kind`

#### Option B: Create The Parent Application Through The Argo CD UI

Start a local port-forward and get the initial admin password:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Then:

1. Open `https://localhost:8080`.
2. Log in with username `admin` and the password from the command above.
3. Go to **Applications** and click **NEW APP**.
4. Set **Application Name** to `app-of-apps-kind`.
5. Set **Project Name** to `default`.
6. Set **Sync Policy** to `Automatic` if you want the demo to reconcile itself.
7. Set **Repository URL** to the Git repo that contains your committed local demo files.
8. Set **Revision** to the branch you pushed.
9. Set **Path** to `argo-apps-kind`.
10. Set **Cluster URL** to `https://kubernetes.default.svc`.
11. Set **Namespace** to `argocd`.
12. Create the application and sync it.

After that, Argo CD should create the child applications `g11-kind-postgres` and `g11-kind`.

### 8. Verify The Local Demo

Check that the applications synced and that the workloads came up:

```bash
kubectl get applications -n argocd
kubectl get pods -n argocd
kubectl get pods -n g11-kind
kubectl get postgrescluster -n g11-kind
kubectl get rollout -n g11-kind
```

For local access:

```bash
# Argo CD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Initial Argo CD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# Frontend directly
kubectl port-forward svc/frontend -n g11-kind 3000:3000
```

Open `https://localhost:8080` in your browser and log in with username `admin` and the password from the command above.

Because `nonhelmhelpers/kind-config.yaml` maps the Istio gateway NodePorts to the host, you can also test the gateway directly on:

- `http://g11.127.0.0.1.nip.io`

Use `g11.127.0.0.1.nip.io` for the demo hostname. Istio 1.28 rejects `localhost` as a Gateway/VirtualService host because it is not an FQDN, so `g11/values-kind.yaml` uses a loopback `nip.io` name instead.

### Local Demo Notes

- The local Postgres application uses Helm release name `hippo-postgres` so that the generated database secret names match `postgres.clusterName`.
- `g11/values-kind.yaml` disables sealed secrets and TLS/ACME on purpose for local work.
- If you want the local demo to use real sealed secrets, install Sealed Secrets into the cluster and reseal the secrets against that cluster's public certificate.
- If OAuth callbacks need to work end-to-end, use a hostname and callback configuration that matches your identity provider setup instead of the placeholder demo secrets.
- Argo CD syncs from Git, not from your working tree. The `kind` app-of-apps files must exist in the remote branch referenced by `repoURL` and `targetRevision` before the bootstrap application can work.

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

