# YAML Configuration Best Practices

When working with Kubernetes manifests, following best practices ensures clean, portable, and reusable configurations.

## Exporting YAML from Running Resources

When you export YAML from existing resources (e.g., `kubectl get pod <name> -o yaml`), Kubernetes includes auto-generated fields that are **not** useful for deployment files.

### Fields to Remove

These fields are automatically generated and should be deleted from exported configs:

```yaml
# REMOVE THESE FIELDS
metadata:
  uid: "12345-abcde-67890"                    # Unique identifier (auto-generated)
  resourceVersion: "12345"                     # Internal version tracking
  creationTimestamp: "2025-12-29T10:30:00Z"   # Auto-timestamp
  generation: 2                                # Internal counter
  managedFields: [...]                         # Internal field tracking

status:                                        # Runtime state (read-only)
  phase: Running
  conditions: [...]
  containerStatuses: [...]
```

### Clean YAML Template

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
  labels:
    app: my-app
    environment: production
  annotations:
    description: "This is my application pod"
spec:
  containers:
  - name: my-container
    image: my-image:1.0
    ports:
    - containerPort: 8080
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```

**What to keep:**
- `apiVersion` - API version
- `kind` - Resource type
- `metadata.name` - Resource name
- `metadata.namespace` - Namespace
- `metadata.labels` - Organization labels
- `metadata.annotations` - Custom metadata
- `spec` - Desired state specification

## Organization Principles

### Structure for Readability

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: my-app
spec:
  # Configuration grouped logically
  database:
    host: postgres.default
    port: 5432
  cache:
    ttl: 3600
```

### Use Labels for Organization

See [[Labels and Selectors]] for:
- Labeling resources consistently
- Using selectors to identify related resources
- Organizing by application, environment, team

```yaml
metadata:
  labels:
    app: nginx                    # Application name
    version: v1                   # Version
    environment: production       # Environment
    team: backend                 # Owning team
```

## Reusability Tips

### 1. Use Namespaces for Project Isolation

Instead of deploying everything to `default`, create dedicated namespaces:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-project
  labels:
    project: my-project
```

Then deploy resources to this namespace:

```yaml
metadata:
  namespace: my-project
```

### 2. Template Variables

Use tools like Kustomize or Helm to avoid duplicating similar configs:

```yaml
# Not recommended - duplicated configs
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 3
  template:
    spec:
      containers:
      - image: myapp:v1

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 3
  template:
    spec:
      containers:
      - image: myapp:v2
```

### 3. Resource Requests and Limits

Always specify resource requests for proper [[Kubernetes Scheduler|scheduling]]:

```yaml
spec:
  containers:
  - name: my-app
    resources:
      requests:           # Guaranteed minimum
        cpu: "100m"
        memory: "128Mi"
      limits:             # Maximum allowed
        cpu: "500m"
        memory: "512Mi"
```

## Version Control Best Practices

1. **Commit YAML files** to your repository
2. **Remove sensitive data** (passwords, tokens) - use Secrets instead
3. **Keep configurations DRY** - avoid duplication
4. **Document non-obvious settings** with annotations

```yaml
metadata:
  annotations:
    description: "Main web server for production"
    oncall: "ops-team@company.com"
    runbook: "https://wiki.company.com/runbooks/web-server"
```

## Editing Helm values.yaml Files

When installing Kubernetes applications with [[Helm]], you often need to customize the chart's default configuration by editing the `values.yaml` file. This is especially critical for on-premise deployments.

### Why Edit values.yaml?

Helm charts are designed for generic use cases. The `values.yaml` file contains all configurable parameters that you can override to match your specific environment.

**Common scenarios requiring customization:**
- Changing Service type from LoadBalancer to NodePort (on-premise)
- Setting fixed NodePort values instead of random assignments
- Customizing resource limits and requests
- Configuring persistent storage paths
- Enabling/disabling features
- Setting custom image registries

### How to Get values.yaml

```bash
# Method 1: Pull chart and extract values.yaml
helm pull <repository>/<chart> --untar
cd <chart>/
vim values.yaml

# Method 2: View default values without downloading
helm show values <repository>/<chart> > values.yaml
```

### Example: Ingress Controller for On-Premise

For [[On-Premise Ingress Setup]], the default Nginx Ingress Controller chart uses `type: LoadBalancer`, which doesn't work on-premise (no cloud provider).

**Critical changes needed:**

```yaml
# Default values.yaml (won't work on-premise)
controller:
  service:
    type: LoadBalancer
    # ports assigned randomly

# Custom values.yaml for on-premise
controller:
  service:
    type: NodePort           # Change from LoadBalancer
    nodePorts:
      http: 30080           # Fix port instead of random
      https: 30443          # Fix port instead of random
```

### Testing Before Installation

Always validate your customizations before installing:

```bash
# Dry run to see what would be created
helm install <release> <chart> -f custom-values.yaml --dry-run

# Generate YAML to inspect resources
helm template <release> <chart> -f custom-values.yaml > output.yaml
```

### Best Practices for values.yaml Editing

1. **Create separate custom file** - Don't edit original `values.yaml`
   ```bash
   cp values.yaml custom-values.yaml
   # Edit custom-values.yaml
   helm install myapp . -f custom-values.yaml
   ```

2. **Only include changed values** - No need to copy entire file
   ```yaml
   # minimal-custom.yaml (only overrides)
   controller:
     service:
       type: NodePort
   ```

3. **Comment your changes** - Explain why you overrode defaults
   ```yaml
   controller:
     service:
       type: NodePort  # On-premise cluster has no cloud LoadBalancer
   ```

4. **Version control your custom values** - Track what you changed
   ```bash
   git add custom-values.yaml
   git commit -m "Configure Ingress for on-premise deployment"
   ```

5. **Document environment-specific values** - Separate dev/staging/prod configs
   ```
   values-dev.yaml
   values-staging.yaml
   values-prod.yaml
   ```

### When to Edit vs Use --set

**Edit values.yaml when:**
- Making multiple changes
- Changes are complex (nested values)
- Configuration should be version-controlled
- Setting up production environments

**Use `--set` flag when:**
- Quick one-off changes
- Testing different values
- Scripting installations
```bash
helm install myapp chart/ --set service.type=NodePort --set service.nodePort=30080
```

## Application to Key Concepts

- Use [[Labels and Selectors]] to organize resources
- Apply to [[Deployments]] for consistent application configuration
- Apply to [[Services]] to expose correctly configured Pods
- Apply to [[K8S Namespace]] for project isolation

## Related Concepts

- [[Deployments]] - Common resource type to configure
- [[Services]] - Expose Pods with clean configs
- [[Labels and Selectors]] - Organization mechanism
- [[Management Tools]] - `kubectl apply` uses these configs
- [[Helm]] - Package manager using values.yaml for configuration
- [[On-Premise Ingress Setup]] - Real example of values.yaml customization
