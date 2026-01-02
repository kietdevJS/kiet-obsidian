# Kubernetes Visual Learning Guide

Visual representations of key Kubernetes concepts to aid understanding.

## The Deployment Lifecycle

```mermaid
graph TD
    A["Developer<br/>kubectl apply deployment.yaml"] 
    
    B["Kubernetes API Server<br/>Processes request"]
    
    C["Deployment Controller<br/>Creates ReplicaSet"]
    
    D["ReplicaSet Controller<br/>Ensures replica count"]
    
    E1["Pod 1<br/>Creating"]
    E2["Pod 2<br/>Creating"]
    E3["Pod 3<br/>Creating"]
    
    F["Kubernetes Scheduler<br/>Assigns to Nodes"]
    
    G1["Node 1<br/>Pod 1 Running"]
    G2["Node 2<br/>Pod 2 Running"]
    G3["Node 3<br/>Pod 3 Running"]
    
    A --> B --> C --> D --> E1 & E2 & E3
    E1 & E2 & E3 --> F
    F --> G1 & G2 & G3
    
    style A fill:#fff,stroke:#333
    style B fill:#f0f0f0,stroke:#333
    style C fill:#f0f0f0,stroke:#333
    style D fill:#f0f0f0,stroke:#333
    style E1 fill:#bbf,stroke:#333
    style E2 fill:#bbf,stroke:#333
    style E3 fill:#bbf,stroke:#333
    style F fill:#f0f0f0,stroke:#333
    style G1 fill:#9f9,stroke:#333
    style G2 fill:#9f9,stroke:#333
    style G3 fill:#9f9,stroke:#333
```

## Pod Failure and Self-healing

```mermaid
graph LR
    A["Deployment<br/>Replicas: 3<br/>Desired: 3"] 
    
    B["Pod 1 Running<br/>10.0.1.5"]
    C["Pod 2 Running<br/>10.0.1.9"]
    D["Pod 3 Running<br/>10.0.1.12"]
    
    E["Pod 2 Failed<br/>Crashed"]
    
    F["Deployment Detects<br/>Current: 2 / Desired: 3"]
    
    G["New Pod 2<br/>Creating..."]
    
    H["New Pod 2 Running<br/>10.0.1.15"]
    
    A -->|Monitors| B & C & D
    D -->|Failure| E
    E -->|Detected| F
    F -->|Creates| G
    G -->|Running| H
    
    style A fill:#ff9,stroke:#333,stroke-width:2px
    style B fill:#9f9,stroke:#333
    style C fill:#9f9,stroke:#333
    style D fill:#9f9,stroke:#333
    style E fill:#f99,stroke:#333
    style F fill:#f0f0f0,stroke:#333
    style G fill:#fbb,stroke:#333
    style H fill:#9f9,stroke:#333
```

## Rolling Update Process

```mermaid
graph TD
    subgraph Initial["Initial State: v1.0"]
    I1["Pod 1<br/>v1.0 Running"]
    I2["Pod 2<br/>v1.0 Running"]
    I3["Pod 3<br/>v1.0 Running"]
    end
    
    subgraph Step1["Step 1: Create new Pod"]
    S1a["Pod 1<br/>v1.1 Starting"]
    S1b["Pod 2<br/>v1.0 Running"]
    S1c["Pod 3<br/>v1.0 Running"]
    end
    
    subgraph Step2["Step 2: Terminate old Pod"]
    S2a["Pod 1<br/>v1.1 Running"]
    S2b["Pod 2<br/>v1.0 Running"]
    S2c["Pod 3<br/>v1.0 Running"]
    end
    
    subgraph Final["Final State: v1.1"]
    F1["Pod 1<br/>v1.1 Running"]
    F2["Pod 2<br/>v1.1 Running"]
    F3["Pod 3<br/>v1.1 Running"]
    end
    
    Initial -->|Wait for health| Step1 -->|Terminate| Step2 -->|Repeat| Final
    
    style Initial fill:#9f9,stroke:#333,stroke-width:2px
    style Final fill:#9f9,stroke:#333,stroke-width:2px
    style Step1 fill:#ffb,stroke:#333
    style Step2 fill:#ffb,stroke:#333
```

## Recreate Strategy

```mermaid
graph TD
    subgraph Initial["Initial State: v1.0"]
    I1["Pod 1 v1.0 Running"]
    I2["Pod 2 v1.0 Running"]
    I3["Pod 3 v1.0 Running"]
    end
    
    subgraph Termination["Terminate All"]
    T["All Pods Stopping<br/>Service Unavailable"]
    end
    
    subgraph Final["Final State: v1.1"]
    F1["Pod 1 v1.1 Running"]
    F2["Pod 2 v1.1 Running"]
    F3["Pod 3 v1.1 Running"]
    end
    
    Initial -->|Stop all| Termination -->|Start new| Final
    
    style Initial fill:#9f9,stroke:#333,stroke-width:2px
    style Termination fill:#f99,stroke:#333,stroke-width:2px
    style Final fill:#9f9,stroke:#333,stroke-width:2px
```

## Blue-Green Deployment

```mermaid
graph TD
    subgraph Phase1["Phase 1: Before Deployment"]
    LB1["Load Balancer"]
    subgraph Blue["Blue (v1.0)<br/>Active"]
    B1["Pod 1"]
    B2["Pod 2"]
    B3["Pod 3"]
    end
    LB1 -->|All traffic| Blue
    end
    
    subgraph Phase2["Phase 2: Parallel Deployment"]
    LB2["Load Balancer"]
    Blue2["Blue (v1.0)<br/>Still Active"]
    subgraph Green["Green (v1.1)<br/>Testing"]
    G1["Pod 1"]
    G2["Pod 2"]
    G3["Pod 3"]
    end
    LB2 -->|All traffic| Blue2
    Green -->|Verification| Green
    end
    
    subgraph Phase3["Phase 3: After Cutover"]
    LB3["Load Balancer"]
    Blue3["Blue (v1.0)<br/>Ready for rollback"]
    subgraph Green3["Green (v1.1)<br/>Active"]
    G1_3["Pod 1"]
    G2_3["Pod 2"]
    G3_3["Pod 3"]
    end
    LB3 -->|All traffic| Green3
    end
    
    Phase1 -->|Deploy new version| Phase2 -->|Switch traffic| Phase3
    
    style Blue fill:#9f9,stroke:#333,stroke-width:2px
    style Blue2 fill:#9f9,stroke:#333,stroke-width:1px
    style Green fill:#ffb,stroke:#333,stroke-width:2px
    style Green3 fill:#9f9,stroke:#333,stroke-width:2px
    style Blue3 fill:#f99,stroke:#333,stroke-width:1px
```

## Canary Deployment

```mermaid
graph LR
    T0["v1.0: 100%<br/>v1.1: 0%<br/>Only v1.0"]
    T1["v1.0: 90%<br/>v1.1: 10%<br/>10% new"]
    T2["v1.0: 70%<br/>v1.1: 30%<br/>30% new"]
    T3["v1.0: 50%<br/>v1.1: 50%<br/>50% new"]
    T4["v1.0: 20%<br/>v1.1: 80%<br/>80% new"]
    T5["v1.0: 0%<br/>v1.1: 100%<br/>Only v1.1"]
    
    T0 -->|Monitor<br/>OK| T1 -->|Monitor<br/>OK| T2 -->|Monitor<br/>OK| T3 -->|Monitor<br/>OK| T4 -->|Monitor<br/>OK| T5
    
    style T0 fill:#f99,stroke:#333
    style T5 fill:#9f9,stroke:#333
    style T1 fill:#ffb,stroke:#333
    style T2 fill:#ffb,stroke:#333
    style T3 fill:#ffa,stroke:#333
    style T4 fill:#ffb,stroke:#333
```

## Deployment Strategy Comparison

## Service Load Balancing

```mermaid
graph TB
    Client["Client Requests<br/>Service DNS:<br/>nginx.default.svc"]
    
    Service["Service<br/>ClusterIP: 10.0.0.1<br/>Port: 80"]
    
    L1["Load Balancer<br/>Round-robin"]
    
    P1["Pod 1<br/>nginx<br/>10.0.1.5:80"]
    P2["Pod 2<br/>nginx<br/>10.0.1.9:80"]
    P3["Pod 3<br/>nginx<br/>10.0.1.12:80"]
    
    Client -->|Request| Service
    Service -->|Discovers Pods| L1
    L1 -->|Route 1/3| P1
    L1 -->|Route 1/3| P2
    L1 -->|Route 1/3| P3
    
    P1 -->|Response| Service
    P2 -->|Response| Service
    P3 -->|Response| Service
    Service -->|Response| Client
    
    style Service fill:#9f9,stroke:#333,stroke-width:2px
    style P1 fill:#bbf,stroke:#333
    style P2 fill:#bbf,stroke:#333
    style P3 fill:#bbf,stroke:#333
```

## Scaling In Action

```mermaid
graph LR
    T1["Time: T1<br/>Load: 10k req/s"]
    
    R1["Replicas: 3<br/>Pod 1, 2, 3"]
    
    T2["Time: T2<br/>Load: 50k req/s<br/>CPU: 85%"]
    
    Scale["HPA Triggers<br/>Scale Up"]
    
    R2["Replicas: 10<br/>Pod 1-10"]
    
    T3["Time: T3<br/>Load: 10k req/s<br/>CPU: 25%"]
    
    ScaleDown["HPA Triggers<br/>Scale Down"]
    
    R3["Replicas: 3<br/>Pod 1-3"]
    
    T1 -->|Normal| R1
    R1 -->|Load Increase| T2
    T2 -->|High CPU| Scale
    Scale -->|Add 7 Pods| R2
    R2 -->|Load Decrease| T3
    T3 -->|Low CPU| ScaleDown
    ScaleDown -->|Remove Pods| R3
    
    style R1 fill:#bbf,stroke:#333,stroke-width:2px
    style R2 fill:#ff9,stroke:#333,stroke-width:2px
    style R3 fill:#9f9,stroke:#333,stroke-width:2px
```

## Labels and Selectors Connection

```mermaid
graph TD
    subgraph Deployment["Deployment"]
    Dep["Deployment: nginx<br/>Selector: app=nginx"]
    end
    
    subgraph Pods["Pods"]
    P1["Pod 1<br/>Labels: app=nginx<br/>env=prod<br/>MATCH"]
    P2["Pod 2<br/>Labels: app=nginx<br/>env=prod<br/>MATCH"]
    P3["Pod 3<br/>Labels: app=db<br/>env=prod<br/>NO MATCH"]
    end
    
    subgraph Service["Service"]
    Svc["Service: nginx<br/>Selector: app=nginx"]
    end
    
    Dep -->|Selector matches| P1 & P2
    Dep -->|Ignored| P3
    Svc -->|Selector matches| P1 & P2
    Svc -->|Ignored| P3
    
    P1 -->|Managed by| Dep
    P2 -->|Managed by| Dep
    P1 -->|Exposed by| Svc
    P2 -->|Exposed by| Svc
    
    style Dep fill:#f9f,stroke:#333,stroke-width:2px
    style Svc fill:#9f9,stroke:#333,stroke-width:2px
    style P1 fill:#9f9,stroke:#333
    style P2 fill:#9f9,stroke:#333
```

## Namespace Isolation

```mermaid
graph TB
    Cluster["Kubernetes Cluster"]
    
    subgraph NS1["Namespace: production"]
    Dep1["Deployment: api"]
    Pod1["Pods: api-1, api-2, api-3"]
    Svc1["Service: api-service"]
    end
    
    subgraph NS2["Namespace: staging"]
    Dep2["Deployment: api"]
    Pod2["Pods: api-1, api-2"]
    Svc2["Service: api-service"]
    end
    
    subgraph NS3["Namespace: development"]
    Dep3["Deployment: api"]
    Pod3["Pod: api-1"]
    Svc3["Service: api-service"]
    end
    
    Cluster --> NS1 & NS2 & NS3
    NS1 --> Dep1 --> Pod1 --> Svc1
    NS2 --> Dep2 --> Pod2 --> Svc2
    NS3 --> Dep3 --> Pod3 --> Svc3
    
    style Cluster fill:#f0f0f0,stroke:#333,stroke-width:3px
    style NS1 fill:#9f9,stroke:#333,stroke-width:2px
    style NS2 fill:#ffb,stroke:#333,stroke-width:2px
    style NS3 fill:#bbf,stroke:#333,stroke-width:2px
```

## Scheduler Decision Making

```mermaid
graph TD
    Pod["New Pod<br/>Requests:<br/>CPU: 2<br/>RAM: 4Gi"]
    
    Scheduler["Kubernetes Scheduler"]
    
    Filter["Filter Phase<br/>Can this node run the Pod?"]
    
    N1["Node 1<br/>Free: CPU 2, RAM 8Gi<br/>PASS"]
    N2["Node 2<br/>Free: CPU 1, RAM 6Gi<br/>FAIL<br/>Insufficient CPU"]
    N3["Node 3<br/>Free: CPU 3, RAM 2Gi<br/>FAIL<br/>Insufficient RAM"]
    
    Score["Score Phase<br/>Which node is best?"]
    
    S1["Node 1 Score: 85<br/>Good fit"]
    
    Select["Select Best Node"]
    
    Assign["Pod Assigned<br/>to Node 1"]
    
    Pod --> Scheduler
    Scheduler --> Filter
    Filter --> N1 & N2 & N3
    N1 --> Score
    Score --> S1 --> Select --> Assign
    
    style Pod fill:#bbf,stroke:#333,stroke-width:2px
    style Scheduler fill:#f0f0f0,stroke:#333,stroke-width:2px
    style N1 fill:#9f9,stroke:#333
    style N2 fill:#f99,stroke:#333
    style N3 fill:#f99,stroke:#333
    style Assign fill:#9f9,stroke:#333,stroke-width:2px
```

## Rollback Scenario

```mermaid
graph TD
    V1["Deployment<br/>v1.0<br/>Stable"]
    
    Deploy["Deploy<br/>v1.1"]
    
    V2["v1.1<br/>Running"]
    
    Issue["Critical Bug<br/>Application Error"]
    
    Detect["Monitoring Detects<br/>Error Rate High"]
    
    Rollback["Execute Rollback<br/>kubectl rollout undo"]
    
    V1Back["v1.0<br/>Restored<br/>Stable"]
    
    V1 -->|Production| Deploy
    Deploy -->|v1.1 Running| V2
    V2 -->|Issue Occurs| Issue
    Issue -->|Alert| Detect
    Detect -->|Manual/Automatic| Rollback
    Rollback -->|Previous Version| V1Back
    
    style V1 fill:#9f9,stroke:#333,stroke-width:2px
    style V2 fill:#f99,stroke:#333,stroke-width:2px
    style Issue fill:#f66,stroke:#333,stroke-width:2px
    style V1Back fill:#9f9,stroke:#333,stroke-width:2px
```

---

## Study Tips

- **Trace the flows**: Start from top (user action) and follow down to Pods
- **Identify bottlenecks**: What could fail at each step?
- **Think about edge cases**: What happens when resources run out?
- **Practice**: Deploy real Deployments and watch these processes happen

---

See also:
- [[Core Concepts Hub]] - Text explanation
- [[Kubernetes Hierarchy Analogy]] - Conceptual analogy
- [[Kubernetes Takeaways Summary]] - Summary of all takeaways
