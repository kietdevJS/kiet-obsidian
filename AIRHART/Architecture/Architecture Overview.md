# Architecture Overview

> Core architectural principles and design patterns of the AIRHART system

## Architectural Principles

### 1. Message-Based Communication

**Core Concept:** Asynchronous messaging enables loose coupling and scalability

```
Component A --[message]--> Kafka Topic --[message]--> Component B
```

**Benefits:**
- Components can scale independently
- Fault tolerance through retry mechanisms
- Non-blocking operations
- Event history preserved in topics

**Key Patterns:**
- Publish-Subscribe for one-to-many communication
- Point-to-Point for specific integrations
- Priority queues for operational vs. batch messages

### 2. Event-Driven Design

**Event Flow:**
```
External Event → Adapter → Topic → AODB → Processing → Output Topic → QueryStream
```

**Characteristics:**
- **Idempotent Processing** - Same message produces same result
- **Eventual Consistency** - Data eventually consistent across system
- **Event Sourcing** - All changes captured as events
- **CQRS** - Command (write) and Query (read) separation

### 3. Reactive Microservices

**Component Independence:**
- Each component has its own:
  - Data store (PostgreSQL, document DB)
  - Execution environment (Kubernetes pods)
  - Release pipeline
  - Scaling strategy

**Benefits:**
- Independent deployment
- Technology flexibility
- Failure isolation
- Optimized scaling per component

### 4. Multi-Tenancy

```
┌─────────────────────────────────┐
│   Tenant A (Airport CPH)     │
│  ┌─────┐ ┌─────┐ ┌─────┐       │
│  │AODB │ │ QS │ │ MD  │       │
│  └─────┘ └─────┘ └─────┘       │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│   Tenant B (Airport ISX)        │
│  ┌─────┐ ┌─────┐ ┌─────┐        │
│  │AODB │ │ QS  │ │ MD  │        │
│  └─────┘ └─────┘ └─────┘        │
└─────────────────────────────────┘
```

**Isolation:**
- Separate environments per tenant
- Independent configuration
- Isolated data stores
- Custom business rules per tenant

## System Layers

### Infrastructure Layer
- **Kubernetes** - Container orchestration
- **Kafka** - Message broker
- **PostgreSQL** - Relational data store
- **Azure** - Cloud infrastructure

### Component Layer
- **[[AODB Component]]** - Operational data processing
- **[[QueryStream Component]]** - Data distribution
- **[[Master Data Component]]** - Reference data
- **[[Adapter Components]]** - External integrations

### Application Layer
- **AODB Pipeline** - Message processing pipeline
- **QueryStream Subscription Manager** - Client connections
- **QueryStream Database Manager** - Data persistence
- **API Gateway** - REST endpoints

### Integration Layer
- **Inbound Adapters** - External system → AIRHART
- **Outbound Adapters** - AIRHART → External system
- **Plugins** - Custom business logic

## Design Patterns

### 1. Pipeline Pattern (AODB)
```
Input → Mapping → Matching → Enrichment → Business Rules → Persistence → Output
```

See: [[AODB Processing Pipeline]]

### 2. Feed Pattern (QueryStream)
```
Source Data → Compile Feed → Store Feed → Query/Subscribe → Stream to Clients
```

See: [[QueryStream Feeds]]

### 3. Adapter Pattern
```
External System → Protocol Adapter → Message Transformation → Kafka Topic
```

See: [[Adapter System]]

### 4. Plugin Pattern
```
Core Pipeline → Plugin Hook → Custom Logic → Continue Pipeline
```

See: [[Plugin Architecture]]

## Data Flow Architecture

### Inbound Flow
```
External System
    ↓ (HTTP/File/API)
Adapter Component
    ↓ (Validate & Transform)
Kafka Interface Topic
    ↓ (Consume)
AODB Adapter Consumer
    ↓ (Map & Match)
AODB Internal Topic
    ↓ (Process)
AODB Pipeline
```

See: [[Inbound Message Flow]]

### Processing Flow
```
AODB Pipeline Input
    ↓
1. Deserialize Message
    ↓
2. Load Entity from Database
    ↓
3. Apply Business Rules
    ↓
4. Enrich with Master Data
    ↓
5. Persist Changes
    ↓
6. Publish Document Updates
```

See: [[AODB Processing Pipeline]]

### Outbound Flow
```
AODB Output Topic
    ↓
QueryStream Database Manager
    ↓ (Compile & Store)
QueryStream Feed Database
    ↓ (Match Subscriptions)
QueryStream Subscription Manager
    ↓ (WebSocket/REST)
Clients (UI, External Systems)
```

See: [[Outbound Message Flow]]

## Scalability Strategy

### Horizontal Scaling
- **AODB Pipeline**: Scale pods to match Kafka partition count
- **QueryStream Database Manager**: Scale with Kafka lag
- **QueryStream Subscription Manager**: Scale with client connections

### Partitioning
- **Topic Partitions**: 50 partitions for parallel processing
- **Message Keys**: Entity ID for consistent routing
- **Consumer Groups**: Multiple consumers per partition

### Performance Optimization
- **Priority Queues**: Operational window messages first
- **Batch Processing**: Off-peak bulk operations
- **Caching**: QueryStream feed cache for fast access

## Resilience & Fault Tolerance

### Retry Mechanisms
- **Transient Errors**: Retry with exponential backoff
- **Concurrency Conflicts**: Automatic retry
- **Database Failures**: Retry forever (for critical paths)

See: [[Resilience Patterns]]

### Error Handling
- **Dead Letter Topics**: Failed messages for investigation
- **Circuit Breakers**: Prevent cascading failures
- **Bulkheads**: Isolate failure domains

### High Availability
- **Multi-Zone Deployment**: Cross-availability zones
- **Database Replication**: Geo-replicated data stores
- **Health Checks**: Kubernetes liveness/readiness probes

## Extensibility Model

### Configuration-Based
- **[[Dynamic Data Model]]** - Add entities and attributes via configuration
- **[[Business Rules]]** - Configurable processing logic
- **[[Schema Updates]]** - Runtime schema evolution

### Code-Based
- **[[Plugin System]]** - Custom C# plugins for complex logic
- **[[Adapter Development]]** - Custom adapters for integrations
- **[[Actions]]** - Configurable actions triggered by events

## Security Architecture

### Authentication & Authorization
- **IDP (Identity Provider)** - Centralized authentication
- **JWT Tokens** - Stateless authentication
- **Role-Based Access** - Permission system
- **Scope-Based Security** - Field-level security

### Network Security
- **TLS Encryption** - All API communication
- **Private Networks** - Internal component communication
- **API Gateway** - Single entry point
- **IP Whitelisting** - Adapter access control

## Monitoring & Observability

### Metrics
- **Message Throughput** - Messages per second
- **Processing Latency** - Time from receive to persist
- **Error Rates** - Failed messages by type
- **Resource Usage** - CPU, memory, disk

### Logging
- **Structured Logging** - JSON formatted logs
- **Correlation IDs** - Trace messages across components
- **Log Aggregation** - Centralized log storage
- **Alert Rules** - Automated incident detection

### Tracing
- **Distributed Tracing** - OpenTelemetry integration
- **Message Tracking** - End-to-end message flow
- **Performance Profiling** - Bottleneck identification

## Related Notes

- [[AODB Component]] - Core processing engine
- [[QueryStream Component]] - Data distribution
- [[Kafka Messaging]] - Message infrastructure
- [[Deployment Model]] - Kubernetes architecture
- [[Design Principles]] - Detailed design patterns

---

**Tags:** #architecture #design-patterns #scalability #event-driven
**Created:** 2025-12-15
**References:** 
- [Confluence: O0500 Software Architecture](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/950559)
- [Confluence: Logical Architecture](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/1278051)
