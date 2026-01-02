# Kubernetes Learning Index

Complete index of all Kubernetes concepts with recommended learning path.

## 🚀 Quick Start

New to Kubernetes? Start here:

1. **[[Core Concepts Hub]]** - 5 min overview of all concepts
2. **[[Kubernetes Hierarchy Analogy]]** - 3 min intuitive understanding
3. **[[Visual Learning Guide]]** - Visual representations of key processes
4. **[[Kubernetes Takeaways Summary]]** - Detailed summary of all takeaways

---

## 📚 Complete Reference

### Fundamental Units

| Concept | What | Why | Start Here |
|---------|------|-----|-----------|
| **Pods** | Smallest deployable unit | Container abstraction | [[Pods]] |
| **Pod Network Identity** | Unique IP per Pod | Enable networking | [[Pod Network Identity]] |
| **Services** | Stable endpoints | Handle ephemeral Pod IPs | [[Services]] |

### Management & Orchestration

| Concept | What | Why | Start Here |
|---------|------|-----|-----------|
| **Deployments** | Production Pod management | Add scaling, self-healing, versioning | [[Deployments]] |
| **Deployment Strategies** | Ways to update applications | Different trade-offs for different needs | [[Deployment Strategies]] |
| **Scaling** | Increase/decrease Pods | Handle varying load | [[Scaling]] |
| **Self-healing** | Automatic Pod recovery | Handle failures without intervention | [[Self-healing]] |
| **Rolling Updates** | Gradual version deployment | Zero-downtime updates (default strategy) | [[Rolling Updates]] |
| **Rollbacks** | Quick version reversion | Handle failed deployments | [[Rollbacks]] |

### Organization & Discovery

| Concept | What | Why | Start Here |
|---------|------|-----|-----------|
| **Labels & Selectors** | Metadata identification | Organize and connect resources | [[Labels and Selectors]] |
| **Namespaces** | Logical resource groups | Isolate projects/environments | [[K8S Namespace]] |
| **Networking** | Pod communication | Enable inter-Pod traffic | [[Kubernetes Networking]] |

### Operation & Infrastructure

| Concept | What | Why | Start Here |
|---------|------|-----|-----------|
| **Scheduler** | Pod to Node assignment | Optimal resource utilization | [[Kubernetes Scheduler]] |
| **Management Tools** | CLI (kubectl) & UI (Rancher) | Interact with cluster | [[Management Tools]] |
| **Container Best Practice** | One container per Pod | Simplify and scale independently | [[Container Best Practice]] |
| **YAML Best Practices** | Configuration patterns | Clean, reusable manifests | [[YAML Best Practices]] |

---

## 🎓 Recommended Learning Paths

### Path 1: Conceptual Understanding (30 minutes)
1. [[Kubernetes Hierarchy Analogy]] - Understand the bakery metaphor
2. [[Core Concepts Hub]] - See how concepts connect
3. [[Visual Learning Guide]] - Visual reinforcement
4. [[Kubernetes Takeaways Summary]] - Full summary

### Path 2: Hands-on Learning (2 hours)
1. [[Pods]] - Understand container units
2. [[Deployments]] - Learn production management
3. [[Deployment Strategies]] - Understand different update approaches
4. [[Services]] - Enable networking
5. [[Labels and Selectors]] - Organize resources
6. [[Management Tools]] - Practice CLI commands
7. [[YAML Best Practices]] - Write clean configs

### Path 3: Deep Dive (4 hours)
1. Start with Path 1 (30 min)
2. Read all documentation in Fundamental Units section
3. Read all documentation in Management & Orchestration section
4. Read all documentation in Organization & Discovery section
5. Read Operation & Infrastructure section
6. Review [[Core Concepts Hub]] for integration

### Path 4: Operations Focus (2 hours)
1. [[Deployments]] - How to manage applications
2. [[Kubernetes Scheduler]] - How placement works
3. [[Management Tools]] - CLI commands
4. [[YAML Best Practices]] - Configuration patterns
5. [[Scaling]] - Handle load changes
6. [[Self-healing]] - Automatic recovery
7. [[Rolling Updates]] & [[Rollbacks]] - Safe deployments

---

## 🔗 Concept Relationships

```
                          DEPLOYMENTS
                         /    |     \
                        /     |      \
                   Scaling  Self-heal Rolling Updates
                       \      |      /
                        \     |     /
                         \    |    /
                           PODS
                           /   \
                          /     \
                    Network ID  Services
                          \     /
                           \   /
                        Networking
                            
        Labels & Selectors (throughout)
        Namespaces (organize all)
        Scheduler (places Pods)
        Management Tools (operate all)
```

---

## Common Questions and Answers

| Question | Answer | Read |
|----------|--------|------|
| What's the smallest thing I can deploy? | Pods | [[Pods]] |
| Can Pods have multiple containers? | Yes, but don't | [[Container Best Practice]] |
| Why does my Pod IP keep changing? | Pods are ephemeral | [[Pod Network Identity]] |
| How do I expose Pods to users? | Use Services | [[Services]] |
| How do I scale my application? | Deployments with replicas | [[Deployments]], [[Scaling]] |
| What happens if a Pod crashes? | Self-healing creates a new one | [[Self-healing]] |
| How do I update my application? | Rolling Updates | [[Rolling Updates]] |
| How do I undo a bad deployment? | Rollbacks | [[Rollbacks]] |
| How does Kubernetes decide where to run my Pod? | The Scheduler | [[Kubernetes Scheduler]] |
| How do I organize my cluster? | Labels, Selectors, Namespaces | [[Labels and Selectors]], [[K8S Namespace]] |
| What CLI commands should I know? | kubectl apply, get, exec | [[Management Tools]] |
| What should I include in my YAML files? | Clean specs, useful labels/annotations | [[YAML Best Practices]] |

---

## 📖 Documentation by Category

### Conceptual (Big Picture)
- [[Kubernetes Hierarchy Analogy]] - Intuitive understanding
- [[Core Concepts Hub]] - How concepts connect
- [[Kubernetes Takeaways Summary]] - Comprehensive summary

### Visual
- [[Visual Learning Guide]] - Diagrams and flows

### Core Technology
- [[Pods]] - Container units
- [[Deployments]] - Pod management
- [[Services]] - Network abstraction
- [[Kubernetes Networking]] - Communication

### Features
- [[Scaling]] - Capacity management
- [[Self-healing]] - Automatic recovery
- [[Rolling Updates]] - Safe deployments
- [[Rollbacks]] - Version reversion

### Organization
- [[Labels and Selectors]] - Resource identification
- [[K8S Namespace]] - Logical grouping
- [[Container Best Practice]] - Design patterns

### Operation
- [[Kubernetes Scheduler]] - Pod placement
- [[Management Tools]] - kubectl and Rancher
- [[YAML Best Practices]] - Configuration

### Reference
- [[K8S Architecture]] - System components
- [[What is K8S]] - Why Kubernetes exists
- [[When to use K8S]] - Use case guidelines

---

## 🔍 Search by Use Case

**"I want to..."**

### Deploy and manage applications
See: [[Deployments]], [[Labels and Selectors]], [[YAML Best Practices]]

### Handle growing traffic
See: [[Scaling]], [[Kubernetes Scheduler]], [[Kubernetes Networking]]

### Deploy safely without downtime
See: [[Rolling Updates]], [[Rollbacks]], [[Services]]

### Organize my cluster
See: [[K8S Namespace]], [[Labels and Selectors]], [[Container Best Practice]]

### Debug issues
See: [[Management Tools]], [[Self-healing]], [[Kubernetes Scheduler]]

### Understand how Kubernetes works
See: [[Kubernetes Hierarchy Analogy]], [[Core Concepts Hub]], [[Visual Learning Guide]]

### Write better YAML files
See: [[YAML Best Practices]], [[Labels and Selectors]]

### Set up multiple environments
See: [[K8S Namespace]], [[YAML Best Practices]]

---

## 📊 Concept Dependency Graph

```
Kubernetes
├── Clusters & Nodes
│   └── Kubernetes Scheduler
├── Containers & Images
│   └── Pods
│       ├── Pod Network Identity
│       ├── Container Best Practice
│       └── Deployments
│           ├── Scaling
│           ├── Self-healing
│           ├── Rolling Updates
│           └── Rollbacks
├── Networking
│   ├── Services
│   ├── Kubernetes Networking
│   └── Labels & Selectors
├── Organization
│   ├── Labels & Selectors
│   └── Namespaces
└── Operations
    ├── Management Tools
    └── YAML Best Practices
```

---

## Learning Checklist

- [ ] Understand what a Pod is ([[Pods]])
- [ ] Know why we use Deployments ([[Deployments]])
- [ ] Grasp the three capabilities of Deployments:
  - [ ] Scaling ([[Scaling]])
  - [ ] Self-healing ([[Self-healing]])
  - [ ] Version management ([[Rolling Updates]], [[Rollbacks]])
- [ ] Understand Pod network identity ([[Pod Network Identity]])
- [ ] Know why Services are needed ([[Services]])
- [ ] Learn Labels and Selectors ([[Labels and Selectors]])
- [ ] Understand namespaces ([[K8S Namespace]])
- [ ] Know basic kubectl commands ([[Management Tools]])
- [ ] Understand how the Scheduler works ([[Kubernetes Scheduler]])
- [ ] Know YAML best practices ([[YAML Best Practices]])
- [ ] See full big picture ([[Core Concepts Hub]])

---

## 🚀 Next Steps After Learning

1. **Set up a local Kubernetes cluster** (Docker Desktop, Minikube)
2. **Create and deploy a simple application** using the patterns learned
3. **Practice scaling** and observe Pod behavior
4. **Cause a Pod to fail** and watch self-healing
5. **Deploy a new version** using rolling updates
6. **Experiment with Services** and DNS resolution
7. **Organize with Namespaces** and Labels
8. **Write YAML files** following best practices
9. **Use Rancher or Dashboard** for visual management
10. **Read production docs** for enterprise patterns

---

**Total Learning Time**: 2-4 hours for solid understanding
**Hands-on Time**: 1-2 hours to solidify concepts

Good luck! 🎉
