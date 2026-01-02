# Rolling Updates

A **Rolling Update** is a strategy for deploying a new version of an application with zero downtime by gradually replacing old Pods with new ones.

## How Rolling Updates Work

Instead of stopping all Pods and starting new ones (which causes downtime), a rolling update:
1. Creates a new Pod running the new version
2. Waits for it to be healthy
3. Removes an old Pod
4. Repeats until all Pods are updated

```mermaid
timeline
    title Rolling Update from v1 to v2
    
    section Initial State
    Pod 1 v1 : active
    Pod 2 v1 : active
    Pod 3 v1 : active
    
    section Step 1
    Pod 1 v2 : active
    Pod 2 v1 : active
    Pod 3 v1 : active
    
    section Step 2
    Pod 1 v2 : active
    Pod 2 v2 : active
    Pod 3 v1 : active
    
    section Final State
    Pod 1 v2 : active
    Pod 2 v2 : active
    Pod 3 v2 : active
```

## Benefits

- **Zero Downtime**: Service never stops accepting requests
- **Gradual Rollout**: Issues can be detected early before all Pods are updated
- **Automatic Health Checks**: Only healthy new Pods receive traffic

## Related Concepts

- [[Deployments]] - The resource that manages rolling updates
- [[Rollbacks]] - Quickly reverting to a previous version
- [[Self-healing]] - Ensuring the correct number of Pods during updates
