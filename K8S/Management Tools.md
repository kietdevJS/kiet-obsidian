# Kubernetes Management Tools

Managing a Kubernetes cluster involves interacting with the API server to deploy, inspect, and manage resources. This is typically done via CLI tools or graphical interfaces.

## CLI Tools

### kubectl

`kubectl` (often aliased as `k`) is the standard command-line tool for communicating with the Kubernetes control plane.

#### Key Commands
-   **`kubectl apply -f <file.yaml>`**: The declarative way to manage applications. It creates or updates resources defined in the YAML file.
-   **`kubectl get pods`** (or `k get po`): Lists the status of Pods in the current namespace.
-   **`kubectl exec -it <pod_name> -- /bin/bash`**: Allows you to enter a container's environment for debugging.

#### YAML Optimization
When exporting YAML configurations for reuse (e.g., using `kubectl get pod <name> -o yaml`), it is best practice to remove auto-generated fields to keep the code clean and portable.

See [[YAML Best Practices]] for details on removing auto-generated fields.

### Helm

[[Helm]] is a package manager for Kubernetes that simplifies deployment and management of applications.

**Key Features**:
- Package Kubernetes resources as charts
- Version control for deployments
- Templating for reusable configurations
- Easy installation of third-party applications

**Common Commands**:
```bash
# Add repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Install chart
helm install my-release ./my-chart

# List releases
helm list

# Upgrade release
helm upgrade my-release ./my-chart

# Rollback
helm rollback my-release
```

See [[Helm]] for complete usage guide.

## Graphical UI: Rancher

**Rancher** is a popular open-source platform for managing Kubernetes clusters.

-   **Ease of Use**: Allows users to manage clusters without deep CLI knowledge.
-   **Capabilities**:
    -   Scaling applications via UI.
    -   Viewing container logs.
    -   Editing YAML configurations directly in the browser.
    -   Managing access control and security.

## The Scheduler

While not a direct user tool, the **Scheduler** is a critical component of the control plane that works in the background.
-   **Role**: It watches for newly created Pods that have no Node assigned.
-   **Decision Making**: It selects a Node for the Pod to run on based on resource availability (CPU/RAM) and constraints.

See also: [[K8S Architecture]], [[Helm]], [[YAML Best Practices]]
