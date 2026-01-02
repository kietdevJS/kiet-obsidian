# Deployments

A **Deployment** provides declarative updates for [[Pods]] and ReplicaSets. It is the standard way to manage stateless applications in Kubernetes.

## Why use Deployments?

In actual projects, Pods are almost never run independently because they don't heal themselves. Deployments provide the management layer that standalone Pods lack.

### Core Capabilities

1.  **Scaling**:
    -   You can increase or decrease the number of Pod replicas to handle traffic load.
    -   *Example*: "I need 5 instances of my web server."

2.  **Self-healing**:
    -   The Deployment controller continuously monitors the state.
    -   If a Pod fails, crashes, or is deleted, the Deployment automatically creates a new one to maintain the desired state.

3.  **Version Management (Rolling Updates)**:
    -   Deployments allow you to update the running application (e.g., v1 to v2) with zero downtime.
    -   It incrementally replaces old Pods with new ones.
    -   **Rollbacks**: If a new version has errors, you can quickly revert to the previous stable version.

## Replica Strategy

By defining a `replicas` count, the Deployment ensures a specific number of identical Pods are running simultaneously.

```mermaid
graph TD
    D[Deployment] -->|Manages| RS[ReplicaSet]
    RS -->|Maintains 3 Replicas| P1[Pod 1]
    RS -->|Maintains 3 Replicas| P2[Pod 2]
    RS -->|Maintains 3 Replicas| P3[Pod 3]
```

## Configuration Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

## Complete Deployment Lifecycle

```mermaid
graph TD
    Deploy["Deploy Deployment<br/>replicas: 3"] -->|Creates| RS["ReplicaSet"]
    RS -->|Manages| P1["Pod 1"]
    RS -->|Manages| P2["Pod 2"]
    RS -->|Manages| P3["Pod 3"]
    
    P2 -->|Crashes| Dead["Failed"]
    RS -->|Self-healing| P2New["Pod 2 Replacement"]
    
    NewImg["New Image<br/>v1.15 to v1.16"] -->|Triggers| Strategy["Choose Strategy"]
    Strategy -->|Rolling Update| Update["Gradually replace"]
    Strategy -->|Recreate| Recreate["Stop all, start new"]
    Strategy -->|Blue-Green| BlueGreen["Run both, switch traffic"]
    Strategy -->|Canary| Canary["Gradual traffic shift"]
    
    Update -->|Gradually| P1New["Pod 1 v1.16"]
    
    Issue["Critical Bug"] -->|Triggers| Rollback["Rollback to v1.15"]
    Rollback -->|Reverts| P1Old["Pod 1 v1.15"]
```

For detailed information on all deployment strategies, see [[Deployment Strategies]].

## The Deployment Abstraction Stack

```mermaid
graph TB
    D["Deployment<br/>What you create"]
    RS["ReplicaSet<br/>Manages replica count"]
    P["Pods<br/>Actually run containers"]
    
    D -->|Creates| RS
    RS -->|Creates| P
```

## Management Best Practices

1. **Always use Deployments** in production, not standalone Pods
2. **Use [[Labels and Selectors]]** for organizing and managing workloads
3. **Define resource requests and limits** for proper [[Kubernetes Scheduler|scheduling]]
4. **Use [[Services]]** to expose Pods, not Pod IPs directly

## Related Concepts

- [[Labels and Selectors]] - How Deployments find Pods
- [[Scaling]] - Managing replica count
- [[Self-healing]] - Automatic Pod replacement
- [[Rolling Updates]] - Safe version updates
- [[Rollbacks]] - Reverting failed deployments
- [[Kubernetes Scheduler]] - Places Pods on nodes
- [[Pods]] - The units managed by Deployments
