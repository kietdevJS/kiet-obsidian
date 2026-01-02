# Pods

A **Pod** (as in a pod of whales or pea pod) is the smallest and simplest deployable unit in Kubernetes. It represents one or more containers grouped together that share resources and a network namespace.

![[Screenshot 2025-12-08 at 21.51.10.png]]

## Fundamental Characteristics

### Network Identity
- Every Pod is assigned its own unique internal IP address within the cluster (see [[Pod Network Identity]])
- All containers in a Pod share this single network interface
- Containers within a Pod can communicate via `localhost`

### Shared Resources
Containers in the same Pod share:
- Network namespace (IP address, ports, hostnames)
- Storage volumes
- Configuration (environment variables, secrets)

### Single Responsibility Principle
Although a Pod *can* contain multiple containers, it's a **best practice to run only one container per Pod** (see [[Container Best Practice]])
- Simplifies debugging
- Enables independent scaling
- Reduces dependencies

## Pod Lifecycle

Pods are ephemeral - they are created and destroyed regularly:
- Normal: Updated via [[Deployments]]
- Failures: Replaced by [[Self-healing]] mechanisms
- When replaced: Assigned a new [[Pod Network Identity|IP address]]

This ephemeral nature is why [[Services]] exist as a stable abstraction layer.

## Basic Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: car-serv
  namespace: car-serv  # target namespace
  labels:
    app: car-service   # for use with selectors
spec:
  containers:
    - name: car-serv
      image: elroydevops/car-serv
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
```

## Production Consideration

In real-world scenarios, Pods are **almost never run independently**. Instead, you use [[Deployments]] which:
- Manage [[Scaling|scaling]]
- Implement [[Self-healing]]
- Enable [[Rolling Updates]] and [[Rollbacks]]

## Related Concepts

- [[Pod Network Identity]] - IP addressing for Pods
- [[Services]] - Stable endpoints for accessing Pods
- [[Deployments]] - Management layer for Pods
- [[Container Best Practice]] - One container per Pod
- [[Labels and Selectors]] - How to identify Pods
- [[Kubernetes Networking]] - How Pods communicate
