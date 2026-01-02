# AIRHART Backend Overview

> A comprehensive guide to understanding the AIRHART backend architecture, data flow, and event-driven design.

## What is AIRHART?

AIRHART is a **message-based and event-driven architecture** with a centralized data hub and decentralized business logic. It's designed for airport operational data management with high availability, real-time processing, and extensibility.

## Core Principles

### 🔄 Event-Driven Architecture
- Asynchronous message-based communication
- Publish-subscribe patterns
- Idempotent message processing
- Priority-based message queuing

### 🎯 Reactive Microservices
- Independent components with dedicated data stores
- Isolated deployment and release cycles
- Horizontal scalability
- Fault-tolerant by design

### 🔌 Extensibility
- Dynamic data model configuration
- Plugin system for custom logic
- Adapter framework for external integrations
- Business rules engine

## System Architecture

```
External Systems (Adapters)
         ↓
    Kafka Topics
         ↓
   ┌─────────────┐
   │    AODB     │ ← Core operational data processing
   │  Pipeline   │
   └─────────────┘
         ↓
    Kafka Topics (Output)
         ↓
   ┌─────────────┐
   │ QueryStream │ ← Real-time data distribution
   │  Component  │
   └─────────────┘
         ↓
    Clients (UI, APIs)
```

## Main Components

- **[[AODB Component]]** - Airport Operational Database, the heart of data processing
- **[[QueryStream Component]]** - Real-time data query and streaming service
- **[[Master Data Component]]** - Reference data management
- **[[Adapter System]]** - External system integrations

## Data Flow

1. **[[Inbound Message Flow]]** - External systems → Adapters → AODB
2. **[[AODB Processing Pipeline]]** - Enrichment, matching, business rules
3. **[[Outbound Message Flow]]** - AODB → QueryStream → Clients

## Key Concepts

### Messages & Topics
- **[[Kafka Messaging]]** - Message broker infrastructure
- **[[Message Types]]** - Adapter messages, DTDs, documents
- **[[Topic Structure]]** - Partitioning and consumer groups

### Data Processing
- **[[Entity Model]]** - Flexible schema system
- **[[Business Rules]]** - Configurable processing logic
- **[[Enrichment Pipeline]]** - Data enhancement flow

### Event-Driven Patterns
- **[[Event Sourcing]]** - Message history tracking
- **[[CQRS Pattern]]** - Command-query separation
- **[[Eventual Consistency]]** - Distributed data consistency

## Quick Navigation

### By Role
- **Developer** → Start with [[AODB Component]] and [[Pipeline Architecture]]
- **Architect** → Review [[Architecture Overview]] and [[Design Principles]]
- **Ops** → Check [[Deployment Model]] and [[Monitoring]]

### By Topic
- **Message Flow** → [[Inbound Message Flow]] → [[AODB Processing Pipeline]] → [[Outbound Message Flow]]
- **Components** → [[AODB Component]] | [[QueryStream Component]] | [[Master Data Component]]
- **Integration** → [[Adapter System]] | [[Kafka Messaging]] | [[API Gateway]]

## References

- [Confluence: Logical Architecture](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/1278051)
- [Confluence: Software Architecture](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/950559)
- Codebase: `SA.Component.AODB`, `SA.Component.QueryStream`

---

**Tags:** #airhart #architecture #backend #overview
**Created:** 2025-12-15
**Last Updated:** 2025-12-15
