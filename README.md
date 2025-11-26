# GitOps Kubernetes Infrastructure with FluxCD

This repository implements a comprehensive GitOps approach for managing Kubernetes clusters using FluxCD. It provides a structured, scalable solution for deploying and managing infrastructure applications, observability tools, and cluster configurations across multiple environments.

## 🏗️ Project Overview

This project demonstrates modern Kubernetes cluster management using:

- **GitOps Principles**: All cluster state is declaratively managed through Git
- **FluxCD**: Continuous delivery and reconciliation of cluster resources
- **Kustomize**: Configuration management and environment-specific customizations
- **Helm**: Package management for complex applications
- **SOPS**: Secret encryption and management
- **Multi-cluster Support**: Organized structure for multiple cluster types (k3d, kind, AWS, Azure, GCP, etc.)

## 📁 Repository Structure

```text
├── install/               # FluxCD bootstrap configuration
├── components/            # Shared kustomize components (common-repo GitRepository)
├── common/                # Shared infrastructure applications
│   ├── cert-manager/      # Certificate management
│   ├── dbms/              # Database management systems
│   ├── external-secrets/  # External secret management
│   ├── istio-system/      # Service mesh
│   ├── keda/              # Event-driven autoscaling
│   ├── kube-system/       # Core Kubernetes system apps
│   ├── kube-tools/        # Kubernetes tooling (Kyverno, NFD)
│   └── observability/     # Observability platform (Grafana, Loki, etc.)
└── README.md              # This file
```

## 🎯 Key Features

### Application Architecture

Each application follows a consistent pattern:

```text
app-name/
├── app/
│   ├── kustomization.yaml      # App resources configuration
│   ├── helmrepository.yaml     # Helm repository definition
│   ├── helmrelease.yaml        # Application deployment
│   └── [other-resources].yaml  # ConfigMaps, Secrets, etc.
└── install.yaml                # FluxCD Kustomization
```

### Infrastructure Components

#### Core Infrastructure (`common/`)

- **cert-manager**: Automatic TLS certificate management
- **external-secrets**: Kubernetes External Secrets Operator

#### Kubernetes System (`common/kube-system/`)

- **metrics-server**: Resource metrics collection
- **reloader**: Automatic pod restarts on ConfigMap/Secret changes

#### Kubernetes Tools (`common/kube-tools/`)

- **kyverno**: Policy engine for Kubernetes
- **node-feature-discovery**: Hardware feature detection

#### Observability Platform (`common/observability/`)

- **grafana**: Visualization and dashboards
- **loki**: Log aggregation
- **alloy**: Telemetry collection
- **kube-prometheus-stack**: Prometheus, Grafana, AlertManager
- **tempo**: Distributed tracing
- **node-exporter**: Hardware and OS metrics
- **blackbox-exporter**: External service probing
- **gatus**: Health status dashboard

#### Event-Driven Autoscaling (`common/keda/`)

- **keda**: Kubernetes Event-driven Autoscaling

#### Service Mesh (`common/istio-system/`)

- **istio**: Service mesh for traffic management and security

#### Database Systems (`common/dbms/`)

- **cloudnative-pg**: PostgreSQL operator
- **dragonfly-operator**: Redis-compatible in-memory store

## 🚀 Quick Start

### Prerequisites

- A running Kubernetes cluster
- `kubectl` installed and configured
- `flux` CLI installed
- Git repository access
- SSH key pair for authentication
- `age` key for SOPS encryption (optional)

### 1. Bootstrap FluxCD

Apply the FluxCD bootstrap configuration (folder is `install/` in this repo):

```bash
kubectl apply --kustomize install
```

This installs FluxCD components in the `flux-system` namespace.

### 2. Configure Git Authentication

Create a Kubernetes secret for Git repository access:

```bash
kubectl create secret generic flux-system \
    --namespace=flux-system \
    --from-file=identity=<PRIVATE_KEY_PATH> \
    --from-file=identity.pub=<PUBLIC_KEY_PATH> \
    --from-literal=known_hosts="$(ssh-keyscan -p <PORT|22> <GIT_REPO_HOST> 2>/dev/null | grep -v '^#')"
```

Or use the Flux CLI for GitHub App authentication:

```bash
flux create secret githubapp flux-system \
  --namespace=flux-system \
  --app-id=${FLUX_GITHUB_APP_ID} \
  --app-installation-id=${FLUX_GITHUB_APP_INSTALLATION_ID} \
  --private-key-file=github-app-key.pem
```

### 3. Create SOPS Age Secret (Optional)

If you are using SOPS for secret encryption:

```bash
cat <YOUR_AGE_KEY_FILE> | kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=/dev/stdin
```

### 4. Deploy Cluster Configuration

Choose your cluster type and apply the configuration:

```bash
# For k3d local development
kubectl apply --kustomize clusters/k3d

# For production AWS EKS
kubectl apply --kustomize clusters/aws

# For production Azure AKS
kubectl apply --kustomize clusters/azure
```

### 5. Verify Deployment

Check FluxCD reconciliation status:

```bash
flux get kustomizations
flux get helmreleases
```

## 🔧 Configuration Management

### Environment Variables

Cluster-specific variables are managed in:

- `clusters/<cluster-type>/vars/cluster-settings.yaml`
- `clusters/<cluster-type>/vars/cluster-secrets.yaml` (SOPS encrypted)

### Adding New Applications

1. **Create Application Structure**:

   ```bash
   mkdir -p common/<category>/new-app/app
   ```

2. **Add Repository Definition**:

   ```yaml
   # app/helmrepository.yaml
   apiVersion: source.toolkit.fluxcd.io/v1
   kind: HelmRepository
   metadata:
     name: new-app
     namespace: flux-system
   spec:
     interval: 1h
     url: https://charts.example.com/
   ```

3. **Create HelmRelease**:

   ```yaml
   # app/helmrelease.yaml
   apiVersion: helm.toolkit.fluxcd.io/v2
   kind: HelmRelease
   metadata:
     name: new-app
   spec:
     interval: 30m
     chart:
       spec:
         chart: new-app
         sourceRef:
           kind: HelmRepository
           name: new-app
           namespace: flux-system
   ```

4. **Create Kustomization**:

   ```yaml
   # app/kustomization.yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - helmrepository.yaml
     - helmrelease.yaml
   ```

5. **Create Install Configuration**:

   ```yaml
   # install.yaml
   apiVersion: kustomize.toolkit.fluxcd.io/v1
   kind: Kustomization
   metadata:
     name: new-app
     namespace: flux-system
   spec:
     targetNamespace: new-app
     path: ./common/<category>/new-app/app
     sourceRef:
       kind: GitRepository
       name: common
       namespace: flux-system
     prune: true
     wait: true
     interval: 30m
     timeout: 5m
   ```

6. **Update Parent Kustomization**:

   Add to the category's `kustomization.yaml`:

   ```yaml
   resources:
     - new-app/install.yaml
   ```

## �� Secret Management

### SOPS Integration

Secrets are encrypted using SOPS with age keys:

```bash
# Encrypt a secret
sops --encrypt --age <AGE_PUBLIC_KEY> secret.yaml > secret.sops.yaml

# Edit encrypted secret
sops secret.sops.yaml

# Decrypt for viewing
sops --decrypt secret.sops.yaml
```

### External Secrets

The External Secrets Operator integrates with:

- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Google Secret Manager

## 🏢 Multi-Cluster Management

### Cluster Types Supported

- **k3d**: Local development with k3d
- **kind**: Local development with kind
- **aws**: Amazon EKS production clusters
- **azure**: Azure AKS production clusters
- **gcp**: Google GKE production clusters
- **k0s**: Bare metal k0s clusters
- **metal**: Physical server deployments

### Cluster-Specific Customizations

Each cluster can override common configurations using Kustomize:

```yaml
# clusters/aws/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../common
patchesStrategicMerge:
  - aws-specific-overrides.yaml
```

## 📊 Monitoring and Observability

### Access Dashboards

- **Grafana**: `https://grafana.<your-domain>`
- **Prometheus**: `https://prometheus.<your-domain>`
- **AlertManager**: `https://alertmanager.<your-domain>`
- **Gatus**: `https://status.<your-domain>`

### Key Metrics Monitored

- Cluster resource utilization
- Application health and performance
- Network traffic and security
- Storage and database metrics
- Custom business metrics

## 🔄 GitOps Workflow

1. **Make Changes**: Update configurations in Git
2. **Commit & Push**: Push changes to the repository
3. **FluxCD Sync**: FluxCD automatically detects and applies changes
4. **Verification**: Monitor deployment status through FluxCD
5. **Rollback**: Use Git history for quick rollbacks if needed

## 🚨 Troubleshooting

### Common Issues

**FluxCD not syncing:**

```bash
flux reconcile kustomization flux-system
flux logs --follow
```

**Application deployment failed:**

```bash
flux get helmreleases
kubectl describe helmrelease <app-name>
```

**Secret decryption issues:**

```bash
kubectl logs -n flux-system deployment/kustomize-controller
```

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make changes following the established patterns
4. Test in a local k3d cluster
5. Submit a pull request

## 📚 References

- [FluxCD Documentation](https://fluxcd.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
- [SOPS Documentation](https://github.com/mozilla/sops)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
