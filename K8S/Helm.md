# Helm

**Helm** is a package manager for [[Kubernetes]] that streamlines the installation and management of Kubernetes applications. It works similarly to `apt` for Ubuntu or `yum` for RedHat.

## What is Helm?

Helm packages Kubernetes resources into reusable bundles called "charts". Instead of managing multiple YAML files separately, a Helm chart packages everything together.

## Core Concepts

### Charts
A chart is a collection of files that describe a related set of Kubernetes resources. A single chart might include:
- [[Deployments]]
- [[Services]]
- ConfigMaps
- Ingress resources
- Persistent volumes

### Repositories
Helm charts are stored in repositories, similar to package registries:
```bash
# Add a repository
helm repo add nginx-ingress https://kubernetes.github.io/ingress-nginx

# Update repository index
helm repo update

# Search for charts
helm repo search nginx
```

### Releases
When you install a chart, Helm creates a "release" - an instance of that chart running in your cluster.

## Installation

### Download Helm Binary

```bash
# Download Helm v3.16.2 for Linux AMD64
wget https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz

# Extract archive
tar -zxvf helm-v3.16.2-linux-amd64.tar.gz

# Move to system path
sudo mv linux-amd64/helm /usr/local/bin/helm

# Verify installation
helm version
```

## Basic Usage

### Install a Chart

```bash
# Install from repository
helm install my-release stable/nginx-ingress

# Install from local chart
helm install my-release ./my-chart

# Install with custom values file
helm install my-release ./my-chart -f custom-values.yaml

# Install in specific namespace
helm install my-release ./my-chart -n ingress-nginx
```

### Manage Releases

```bash
# List all releases
helm list

# List releases in specific namespace
helm list -n ingress-nginx

# Get release status
helm status my-release

# Upgrade a release
helm upgrade my-release ./my-chart

# Rollback to previous version
helm rollback my-release

# Uninstall a release
helm uninstall my-release
```

## Customizing Charts with values.yaml

Every Helm chart includes a `values.yaml` file with default configuration values. You can override these values:

### Method 1: Edit values.yaml

```bash
# Download chart to local directory
helm pull nginx-ingress/ingress-nginx --untar

# Edit values.yaml
vim ingress-nginx/values.yaml

# Install with modified values
helm install ingress-nginx ./ingress-nginx -n ingress-nginx
```

### Method 2: Override via Command Line

```bash
# Override single value
helm install my-release ./chart --set replicaCount=3

# Override multiple values
helm install my-release ./chart \
  --set replicaCount=3 \
  --set service.type=NodePort
```

### Method 3: Custom Values File

```bash
# Create custom-values.yaml
cat > custom-values.yaml <<EOF
replicaCount: 3
service:
  type: NodePort
  nodePort: 30080
EOF

# Install with custom values
helm install my-release ./chart -f custom-values.yaml
```

## Advantages

**Package Management**
- Bundle related resources together
- Version control for entire applications
- Easy distribution and sharing

**Templating**
- Reduce YAML repetition
- Parameterize configurations
- Environment-specific deployments

**Release Management**
- Track deployment history
- Easy rollbacks
- Upgrade orchestration

**Community Charts**
- Pre-built charts for common applications
- Battle-tested configurations
- Active maintenance

## Disadvantages

**Learning Curve**
- Another tool to learn
- Template syntax complexity
- Chart structure understanding needed

**Debugging Complexity**
- Templates can obscure actual YAML
- Error messages may be cryptic
- Harder to troubleshoot

**Version Management**
- Chart versions vs application versions
- Breaking changes in chart updates
- Compatibility issues

## When to Use Helm

Use Helm when:
- Deploying complex applications with many resources
- Managing multiple environments (dev, staging, prod)
- Need repeatable, version-controlled deployments
- Installing third-party applications
- Team needs standardized deployment process

Don't use Helm when:
- Learning Kubernetes basics (understand YAML first)
- Very simple applications (single Pod/Service)
- Need full control over every YAML detail
- Team lacks Helm expertise

## Real-world Example: Ingress Controller

Installing Nginx Ingress Controller with Helm:

```bash
# Add repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Pull chart locally for customization
helm pull ingress-nginx/ingress-nginx --untar

# Edit values.yaml for on-premise setup
# Change service type from LoadBalancer to NodePort
# Set specific ports instead of random assignment

# Install with custom configuration
helm install ingress-nginx ./ingress-nginx \
  -n ingress-nginx \
  --create-namespace \
  -f values.yaml

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

See [[On-Premise Ingress Setup]] for complete implementation.

## Helm vs kubectl

| Aspect | kubectl | Helm |
|--------|---------|------|
| **Approach** | Imperative/Declarative | Package-based |
| **Resources** | Individual YAML files | Bundled charts |
| **Versioning** | Manual tracking | Built-in releases |
| **Rollback** | Manual reapply | `helm rollback` |
| **Templating** | None | Go templates |
| **Complexity** | Simple | More complex |
| **Best For** | Simple apps, learning | Complex apps, production |

## Chart Structure

A typical Helm chart directory:

```
my-chart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration
├── charts/             # Dependent charts
├── templates/          # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # Template helpers
│   └── NOTES.txt       # Post-install notes
└── README.md          # Chart documentation
```

## Related Concepts

- [[Management Tools]] - Tools for managing Kubernetes
- [[On-Premise Ingress Setup]] - Using Helm for Ingress deployment
- [[YAML Best Practices]] - Clean YAML configuration
- [[K8S Namespace]] - Organizing Helm releases
- [[Deployments]] - Resources managed by Helm charts
